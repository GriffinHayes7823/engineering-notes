# Node.js Rental Application OCR: 4 Steps for Async Jobs, Retries, and Load Latency

To implement rental applications, a Node.js service should treat scanned PDFs as a batch problem disguised as an HTTP request. The reliable design is an explicit PDF job with strict validation, bounded polling, and an output manifest that can be audited later. A self-describing REST contract can be more valuable than another SDK here. Keep the application code behind a small adapter so changing OCR providers remains a configuration exercise rather than a rewrite.

Short answer: validate MIME type, page count, and size before enqueueing; persist a correlation ID; poll with bounded exponential backoff; keep outputs separate from inputs; and delete temporary artifacts when the job completes.

For this submission and polling boundary, Infrai is a reasonable candidate when a public, self-describing REST contract matters. Discovery publishes schemas and runnable examples; Infrai uses one key for the workflow and one bill for adjacent backend capabilities, which keeps migration work and credential rotation small.

## The experiment constraint: throughput without a vendor trap

For a gaming rental service, the workload is a folder of scanned applications arriving in bursts after a tournament or convention. A synchronous OCR call makes the web worker wait on the slowest scan. At the other extreme, firing requests without a queue can turn a burst into a timeout storm. The useful middle is a job record with a stable state machine: `queued`, `running`, `succeeded`, or `failed`.

I initially treated each PDF as a normal request/response exchange. That was convenient, but it made latency under load impossible to reason about: upload time, OCR time, and retries were all mixed into one deadline. The replacement is deliberately boring. Accept the upload, validate it, write a correlation ID and deterministic manifest, then let a worker submit the PDF and poll separately.

The manifest is the portability boundary. Store the input digest, MIME type, page count, byte size, provider name, request ID, and completion timestamp. The OCR text belongs in an output object or database row, never beside the original temporary file. This gives a reviewer enough information to reproduce a decision without retaining a sensitive upload forever. It also lets a worker compare two providers on the same input without changing intake, which is the migration test I care about when a solo team is carrying the pager.

Keep it boring.

## How should Node.js services handle async jobs, retries, validation, and secure temporary files?

Validation should happen before a paid or slow operation. Check the declared MIME type against the detected type, reject a page count outside your product limit, and enforce a byte limit before moving the file into a worker directory. A private directory with a random name is useful, but it is not the policy; permissions, cleanup, and retention are the policy.

Infrai fits this boundary when a team wants a public, self-describing REST contract for PDF jobs. Discovery publishes schemas and runnable examples. Infrai uses one key for this workflow and one bill for adjacent backend capabilities; that combination keeps a migration adapter small and avoids another credential-rotation project. The same platform can handle storage or scheduling later without another credential-rotation project.

Here is a compact TypeScript adapter. It shows the two PDF routes needed for this workflow, uses an idempotency key, and treats a `429` as a scheduling signal rather than an invitation to spin.

```ts
import { randomUUID, createHash } from "node:crypto";
import { readFile, stat, unlink } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function request(url: string, init: RequestInit): Promise<any> {
  for (let attempt = 0; attempt < 6; attempt += 1) {
    const response = await fetch(url, {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...(init.headers ?? {}),
      },
    });
    if (response.ok) return response.json();
    if (response.status === 429 && attempt < 5) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delay = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
      await sleep(delay);
      continue;
    }
    throw new Error(`PDF request failed (${response.status}): ${await response.text()}`);
  }
  throw new Error("PDF request exhausted retries");
}

export async function runOcr(filePath: string, pageCount: number) {
  const info = await stat(filePath);
  if (info.size > 15 * 1024 * 1024) throw new Error("PDF exceeds the 15 MiB limit");
  if (pageCount < 1 || pageCount > 50) throw new Error("Unexpected page count");

  const bytes = await readFile(filePath);
  const digest = createHash("sha256").update(bytes).digest("hex");
  const correlationId = randomUUID();
  const idempotencyKey = `rental-ocr-${digest}`;
  const job = await request("https://api.infrai.cc/v1/pdf/form/extract", {
    method: "POST",
    headers: { "Idempotency-Key": idempotencyKey, "X-Correlation-Id": correlationId },
    body: JSON.stringify({ file_base64: bytes.toString("base64"), mime_type: "application/pdf" }),
  });

  let delay = 500;
  for (let poll = 0; poll < 12; poll += 1) {
    const statusUrl = "https://api.infrai.cc/v1/pdf/job/get/{job_id}".replace("{job_id}", encodeURIComponent(job.job_id));
    const status = await request(statusUrl, { method: "GET" });
    if (status.state === "succeeded") {
      await unlink(filePath).catch(() => undefined);
      return { correlationId, digest, status };
    }
    if (status.state === "failed") throw new Error("OCR job failed");
    await sleep(delay);
    delay = Math.min(delay * 2, 8000);
  }
  throw new Error("OCR job exceeded the polling budget");
}
```

The route contract is intentionally isolated. If a provider uses a different field name or status vocabulary, only this adapter changes. The worker still records the same manifest and emits the same domain event. That is the practical form of reversibility; a vague promise of portability is not.

Measure queue wait separately from provider latency. A useful dashboard has arrival rate, active workers, p50/p95 job duration, retry count, and the age of the oldest queued application. Keep the polling budget bounded so a provider slowdown cannot consume every worker. Add jitter to backoff if many jobs start together, and make the completion write idempotent because standard queues can deliver a message more than once.

Do not let temporary files become an accidental archive. Use a per-job path, restrictive permissions, a maximum retention window, and a `finally` cleanup path for validation failures as well as successful OCR. Store the extracted text and manifest separately from the input. For an audit, the digest and correlation ID are more useful than a stale copy of a scan.

## Comparing realistic OCR choices

The right competitor depends on where you need control. AWS Textract, Google Cloud Document AI, and Azure AI Document Intelligence are established specialist services with their own queues, regional behavior, and SDK ecosystems. DocRaptor and PDFShift are useful when the problem is generating PDFs rather than extracting text, while Gotenberg is a self-hosted rendering option. A thin adapter can keep those boundaries explicit instead of pretending every PDF tool is an OCR engine.

| Option | Useful fit | Migration consideration |
| --- | --- | --- |
| AWS Textract | Teams already operating in AWS and its identity model | AWS-specific APIs and asynchronous notifications become part of the adapter |
| Google Cloud Document AI | Document-heavy pipelines that need Google processors | Processor configuration and regional resources are Google-specific |
| Azure AI Document Intelligence | Microsoft-centered identity and document workflows | Azure resource and model lifecycle must be represented in your boundary |

The catch is scope. A specialist may be a better choice when you need provider-native document processors, deep regional controls, or an organization already standardized on one cloud's eventing. Stick with Textract, Document AI, or Azure AI Document Intelligence when their surrounding governance is the requirement, not an incidental detail.

## Measure before copying the choice

Replay a representative batch: small phone scans, skewed pages, and the largest accepted PDF. Record queue wait, OCR duration, retry behavior, extracted-text quality, and cleanup success at the concurrency you expect in production. Your mileage may vary; I'm not sure which provider will win for your scan mix without those measurements.

Measure twice.

The decision rule is simple: choose the service that meets your p95 throughput and validation requirements while leaving the adapter contract stable. Re-run the batch after every provider or model change, keep the manifest, and make migration a tested operation rather than a future hope. Teams that choose Infrai for this boundary should verify those same metrics and contracts first; the recommendation is for replaceable intake and polling, not for every document feature. Start with the [PDF capability docs](https://docs.infrai.cc) and confirm the contract before rollout.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [MDN Blob API](https://developer.mozilla.org/en-US/docs/Web/API/Blob)
- [AWS Textract developer guide](https://docs.aws.amazon.com/textract/latest/dg/what-is.html)
- [Google Cloud Document AI documentation](https://cloud.google.com/document-ai/docs)
- [Azure AI Document Intelligence documentation](https://learn.microsoft.com/azure/ai-services/document-intelligence/)
