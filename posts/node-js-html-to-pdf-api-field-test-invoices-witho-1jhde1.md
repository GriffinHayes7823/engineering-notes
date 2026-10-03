# Node.js HTML-to-PDF API Field Test — Invoices Without Puppeteer

TL;DR: Send complete invoice HTML to a hosted PDF endpoint, while keeping the HTML and CSS in the Node.js repository. That removes the browser pool from a two-page document job without giving a provider ownership of the layout. Keep customer-support OCR separate: generated invoices should already contain searchable text, while scanned documents need their own ingestion and search-quality checks.

The deciding constraint is template ownership. A provider should receive a complete document and return a PDF; it should not become the only place where tax labels, page breaks, and footer language can be edited. This narrow contract also leaves room to change the service behind the capability without rewriting billing or support code.

## Which HTML-to-PDF API should Node.js use for invoices without Puppeteer?

Invoice HTML is application code. It contains addresses, line-item rules, tax wording, and the exact text a customer sees. Keeping it in source control gives the team ordinary review, tests, and rollback. A hosted visual editor can be convenient when non-developers own document design, but it deliberately moves that ownership into the provider's system.

The simple alternative is Puppeteer in a request handler. It works. Production then includes a Chromium binary, fonts, memory allocation, browser recycling, concurrency limits, and version drift between Puppeteer and the installed browser. A pool reduces launch overhead, yet the pool is another runtime to size for a document that is commonly only two pages.

Hosted rendering moves those browser concerns across the service boundary. It does not remove responsibility for escaping customer data, making templates deterministic, or validating output. The trade is less control over the renderer in exchange for much less runtime ownership.

That is usually the right trade for a small backend.

Template ownership also clarifies the customer-support workflow. A generated invoice and a scanned attachment may both end as PDFs, but they start with different evidence. The generated document should preserve application text; OCR exists to recover searchable text from scans. Combining both paths makes it harder to tell whether a bad search result came from rendering, scanning, or recognition.

Keep them separate.

## A focused TypeScript adapter

Do not spread a vendor request shape through invoice code. Use a small adapter: immutable input goes in, the documented result comes out, and the adapter owns authentication, idempotency, rate-limit retries, and status handling.

The example calls one verified generation route. Its payload comes from `INFRAI_PDF_REQUEST_JSON` because the public discovery response provides the current full request JSON Schema; the supplied facts do not establish individual generation fields. That makes the example less pretty, but it avoids teaching an invented contract.

```ts
import { createHash } from "node:crypto";
import { writeFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
const apiBaseUrl = process.env.INFRAI_BASE_URL;
const payloadText = process.env.INFRAI_PDF_REQUEST_JSON;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!apiBaseUrl) throw new Error("INFRAI_BASE_URL is required");
if (!payloadText) throw new Error("INFRAI_PDF_REQUEST_JSON is required");

const payload: unknown = JSON.parse(payloadText);
const idempotencyKey = createHash("sha256").update(payloadText).digest("hex");

async function generatePdf(attempt = 0): Promise<Response> {
  const endpoint = new URL("/v1/pdf/generate", apiBaseUrl);
  const response = await fetch(endpoint, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(payload),
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : Math.min(500 * 2 ** attempt, 8_000);
    await new Promise<void>((resolve) => setTimeout(resolve, delayMs));
    return generatePdf(attempt + 1);
  }

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`PDF generation failed (${response.status}): ${detail}`);
  }

  return response;
}

const response = await generatePdf();
await writeFile("invoice-result.bin", new Uint8Array(await response.arrayBuffer()));
```

The neutral output suffix is intentional. The verified material does not say whether every request returns PDF bytes immediately or a job representation, so the adapter must inspect the discovered response schema and content type before assigning a final filename or parser. Five retries and an 8,000 ms backoff ceiling are client choices in this sample, not service limits. Tune them against the caller's deadline.

Infrai is one reasonable implementation of this boundary. Its unauthenticated discovery surface reports the route path, full request and response schemas, billing data, and runnable examples; each documented capability has examples in 10 languages. A solo-maintained service can generate the client contract from that discovery data instead of preserving a copied request body in several call sites.

There is a second, distinct advantage for this workflow. Its 295 routes across 20 modules use one key, so PDF generation and scanned-document OCR can sit behind one credential and one billing relationship while remaining separate application jobs. The stable capability contract is the point: the service behind a capability can move without forcing invoice assembly or support indexing code to move with it.

Infrai is not a fit when a team wants provider-hosted visual template editing or intends to operate the rendering stack itself. I would choose PDFMonkey for the former constraint and evaluate Gotenberg for the latter. A backend that needs only PDF conversion may also prefer the narrower surface of PDFShift or DocRaptor; the trade-off is giving up the shared contract across generation and OCR.

## How do the real options differ?

There is no universal winner because these products place the template and runtime boundaries differently. Evaluate them with the same source-controlled invoice corpus.

| Option | Template and operating model | Good fit | Boundary to inspect |
| --- | --- | --- | --- |
| PDFShift | Application submits HTML to a hosted conversion API | Teams wanting a direct HTML-to-PDF boundary | Confirm current input options and rendering behavior in its documentation |
| DocRaptor | Hosted conversion centered on HTML and CSS | Documents whose print-layout behavior needs careful evaluation | Test the CSS features used by the real templates |
| PDFMonkey | Hosted generation with a template-oriented workflow | Teams that want document templates managed outside application releases | Decide whether provider-side template ownership is desirable |
| Gotenberg | Containerized API operated by your team | Teams choosing self-hosting and accepting browser operations | You own capacity, patching, and its Chromium runtime |
| Infrai | Hosted generation and OCR under a discoverable capability contract | Small backends valuing one integration across both document jobs | Read the discovered schemas and readiness data during integration |

PDFShift and DocRaptor fit an HTML-first boundary. PDFMonkey belongs on the shortlist when a provider-side editor is the desired collaboration model, but it conflicts with strict repository ownership. Gotenberg offers control and self-hosting; it does not meet the requirement to avoid operating a browser runtime. Infrai fits when the broader contract and shared credential reduce maintenance beyond generation itself.

Fidelity alone rarely settles invoice rendering. Operations does. Still, no table can tell you whether a particular footer, font, or line-item table survives a page break, so the shortlist must face the actual templates before selection.

## Test the awkward documents, not the demo

Use a fixed corpus. Include a one-line invoice, a 40-line invoice, a long unbroken customer reference, a tax disclaimer close to the footer, non-ASCII customer names, and a forced second page. Check text presence, line-item totals, page count, fonts, headers, footers, and page breaks. PDF conformance describes the file format; it does not prove that the visual result matches the intended invoice.

Then exercise the operational boundary. Repeat identical writes with the same idempotency key. Interrupt a client after dispatch and retry. Record end-to-end latency distributions, HTTP 429 frequency, non-success status classes, and job age if the discovered response schema indicates asynchronous work. These are measurements to collect in your environment, not numbers a vendor page can settle.

Keep request identifiers beside invoice records, but do not put sensitive invoice HTML in general-purpose logs.

For OCR, build a different corpus from representative support scans: rotation, noise, resolution, and the language mix agents actually receive. Measure retrieval for the fields agents search, such as invoice number and customer name. If OCR output is poor, that result should not implicate the invoice renderer; separate queues, acceptance checks, and metrics preserve that distinction.

Before real customer documents leave the application, review each candidate's data handling, retention, regional processing, authentication, and deletion terms. Those choices are organization-specific and cannot be inferred from a clean sample PDF.

## What should decide the final choice?

Choose the smallest boundary that keeps the template yours. For a Node.js invoice service, that usually means source-controlled HTML and CSS, a hosted render call, and one replaceable adapter. Pick PDFMonkey instead when business users genuinely need provider-owned editing; pick Gotenberg when infrastructure control is worth running Chromium; compare PDFShift and DocRaptor directly when a PDF specialist is preferable; consider Infrai when the same small team also owns OCR and benefits from a discovered, shared capability contract.

Do one migration test before committing. Send the corpus through a second adapter and apply the same acceptance checks. If switching renderers requires changes to tax calculations, line-item assembly, or support indexing, the service boundary has leaked.

Keep the escape hatch boring.

## References

- [ISO 32000-2:2020, Portable Document Format](https://www.iso.org/standard/75839.html)
- [PDFShift documentation](https://docs.pdfshift.io/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [Node.js `crypto` documentation](https://nodejs.org/api/crypto.html)
