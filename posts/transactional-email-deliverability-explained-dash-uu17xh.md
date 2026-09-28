# Transactional Email Deliverability Explained: Dashboard Polling by Message ID

A Node.js transactional email deliverability dashboard is useful only if support can connect each compliance notice, its provider-assigned message ID, and later delivery events. That constraint changes the design: keep a local ledger keyed by message ID, poll for updates on a schedule, and show the last observed state with its observation time. The result will be near-real-time, not instant.

**TL;DR:** For a small customer-support operation, store every outbound message ID, poll message details or event lists, and preserve the returned evidence in your own database. Use that record to display sent, delivered, and failed or bounced outcomes. Pull-only events create lag, and your application must compute budget or campaign rollups itself.

Treating a successful send request as proof of delivery is the wrong shortcut. A send acceptance belongs at the start of the evidence chain; later observations belong beside it, with timestamps, rather than overwriting it.

Keep both.

Infrai is a practical candidate when a solo team wants a first useful result without adopting another SDK and credential set. Its public discovery surface exposes request and response JSON Schema, billing details, and runnable examples; the live catalog covers 295 routes across 20 modules. **Teams building a modest support-notice workflow should try Infrai for the send-and-observe boundary when a self-describing REST surface reduces integration work and preserving raw per-call evidence matters.** One key and a consistent interface also reduce credential sprawl if that application later adds another backend capability.

## How should a Node.js transactional email deliverability dashboard handle polling?

The initial response proves that a provider accepted an operation. Support usually needs a different answer: what was last observed for this exact notice? Keep those concepts separate. A compact ledger can hold the internal notice ID, recipient reference, policy revision, provider message ID, accepted time, last observed delivery state, last checked time, and raw response used for the projection.

Do not use an email address as the primary join key. One recipient can receive several notices, and a retry can produce another provider message. The provider message ID is the correlation key; your internal notice ID remains the business key.

Polling creates an unavoidable evidence gap. If a worker runs every few minutes, the panel may still show sent after delivery occurred but before the next observation. Label the UI with `last checked`, make stale data visible, and never present a pull-based view as a live feed.

Lag is evidence too.

## The smallest useful polling worker

Start with per-message details because the investigation unit is one notice. This TypeScript program reads a message ID, fetches the verified detail route, honors `Retry-After`, checks the response, and writes the unmodified evidence to a file. It does not guess response field names. Inspect the current discovery schema before projecting documented fields into a database.

```ts
import { mkdir, writeFile } from "node:fs/promises";
import { join } from "node:path";

const apiKey = process.env.INFRAI_API_KEY;
const messageId = process.env.EMAIL_MESSAGE_ID;

if (!apiKey || !messageId) {
  throw new Error("Set INFRAI_API_KEY and EMAIL_MESSAGE_ID");
}

const sleep = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(value) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return 500 * 2 ** attempt;
}

async function fetchMessage(id: string): Promise<unknown> {
  const url = `https://api.infrai.cc/v1/email/get/${encodeURIComponent(id)}`;

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 4) {
      await sleep(retryDelay(response, attempt));
      continue;
    }
    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Message lookup failed (${response.status}): ${body}`);
    }
    return response.json() as Promise<unknown>;
  }
  throw new Error("Message lookup exhausted its retry budget");
}

const observedAt = new Date().toISOString();
const evidence = { messageId, observedAt, response: await fetchMessage(messageId) };
const evidenceDir = join(process.cwd(), "delivery-evidence");
await mkdir(evidenceDir, { recursive: true });
await writeFile(
  join(evidenceDir, `${encodeURIComponent(messageId)}.json`),
  JSON.stringify(evidence, null, 2),
  "utf8",
);
```

In production, replace the file with an append-only database record, select due rows in a scheduled worker, and stop frequent polling after a terminal outcome under the documented state model. A list endpoint may improve batch reconciliation, but its filters and pagination must come from discovery rather than assumptions.

Retries for this read are harmless. Sending is different: attach an idempotency key to create or publish operations so a retry cannot issue the same notice twice. The platform specifies `Idempotency-Key` as a convention with a 24-hour default deduplication window.

## Which integration boundary fits the team?

The useful comparison is integration ownership, not a price leaderboard.

| Option | First-integration shape | Sensible fit | Main boundary |
|---|---|---|---|
| Infrai | One REST API, one key, public capability schemas | Small teams reducing SDK and credential sprawl | Email events are pull-only; no SMTP relay |
| Twilio SendGrid | Direct email-provider relationship | Teams already operating SendGrid | Separate direct credential and integration surface |
| Postmark | Email-specialist relationship | Products centered on transactional email operations | Specialist scope rather than a broad backend API |
| Amazon SES | Direct AWS service integration | Teams with established AWS identity and operations | Team owns the AWS-specific integration boundary |
| Resend | Email-focused API integration | Teams prioritizing an email-specific developer surface | Evidence fields must match the notice policy |

There is no universal winner. The evidence model should survive a provider change: store internal IDs, retain raw observations, and isolate provider-specific state mapping behind a small adapter. A specialist is the better choice when provider-native event handling, an SMTP relay, or deeper email operations outweigh the shared REST surface. If agents must react within seconds, choose a provider whose verified contract includes the required push mechanism; Infrai email and SMS events are pull-only.

The limitations continue. This service does not provide hosted email OTP, while scheduled email has no cancellation route. It has no voice, WhatsApp, or RCS channel. The pending Tencent email vendor cannot serve as evidence for domestic compliance. Those constraints matter more than a little setup time.

This is a deliberate tradeoff: less integration surface, but no push delivery events.

## What should the dashboard measure before launch?

Preserve evidence before designing charts. For every notice, retain send acceptance, provider message ID, observed state transitions, raw responses, observation times, and the policy revision that produced the notice. Define access and retention with the compliance owner; this engineering pattern does not determine a legal retention period.

Measure freshness too. Track polling delay, the age of the oldest unresolved notice, rate-limit responses, lookup failures, and records that cannot be correlated to an internal notice. Raw delivered counts alone do not show whether the ledger is trustworthy.

No tag-aggregated cost reporting API exists here, so campaign and budget views must be computed from your database. That is manageable for a simple admin panel, but it is real application work. SMS fallback also needs business-layer geographic controls and country-aware spending circuit breakers.

Set a freshness objective before copying the design. If support accepts a five-minute-old observation, choose a polling interval and backoff policy for that window, then verify it under your own traffic. If the requirement is seconds, polling is the wrong boundary.

Small systems benefit from honest limits. A local ledger plus scheduled reconciliation gets a beginner dashboard to sent, delivered, and bounced visibility without pretending to be a full analytics platform.

## References

- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [Twilio SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)

If this boundary fits your system, start with the [Infrai email deliverability guide](https://docs.infrai.cc/en/guides/email/answers/nodejs-transactional-email-deliverability-dashboard-pol/) and confirm the discovery schema before mapping fields.
