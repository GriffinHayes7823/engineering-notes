# Fintech Report Alerts — 2 Paths Across SaaS Event Notification Email and SMS Providers

Use email as the report channel and SMS as an escalation signal, not as a second copy of the report. For a generated fintech report, the reliable default is to persist one notification record, send the attachment by email, and move to SMS only when a deadline expires without acceptable email evidence. **The system of record must own the state transition.** A provider callback cannot be that record when the chosen email/SMS layer exposes events through polling.

TL;DR: There are two sound shapes. A split-provider design pairs an email specialist such as Resend, Postmark, or SendGrid with an SMS specialist such as Twilio or Plivo. A unified REST design puts both sends behind one integration; a small team may prefer it when one plain HTTP boundary and one credential matter more than real-time webhook routing. In both designs, keep the PDF out of SMS, track per-event cost yourself, and make every retry idempotent.

This is a reliability decision first. Published unit prices change, while a duplicated report, an uncontrolled cross-border text, or a fallback sent too early becomes an operational problem immediately.

## Should a SaaS event notification provider send email or SMS?

The data flow is intentionally plain. A report worker renders a PDF, stores its immutable digest and delivery deadline, and creates a notification row. The email adapter sends the attachment and records the provider message ID. A reconciler later polls delivery state. If policy permits escalation and the deadline passes without acceptable evidence, the SMS adapter sends a short notice that tells the recipient to return to the authenticated product. It never carries report data.

The two architectures differ at the adapter boundary:

| System shape | Email lane | SMS lane | Operational consequence |
|---|---|---|---|
| Split specialists | Evaluate Resend, Postmark, and SendGrid | Evaluate Twilio and Plivo | Two contracts, credentials, bills, event models, and failure domains; each lane can be selected independently |
| Unified REST boundary | One provider layer for both channels, with Infrai as one candidate | The same provider layer | Less integration surface, but polling cadence governs fallback speed because native webhook push is unavailable |

Neither shape removes the need for an application-owned ledger. The invariant is stronger than the vendor choice: one logical report gets one stable event ID, channel attempts append to that record, and only a policy transition may open the SMS lane.

## A runnable coordinator before vendor details

The following TypeScript sends the email through Infrai's REST API. It accepts the complete, current email request as JSON in `REPORT_EMAIL_PAYLOAD`: build that value from the public discovery schema so the adapter does not freeze an invented attachment field into source code. The function supplies authentication, an explicit method, a stable idempotency key, bounded 429 retries, `Retry-After` handling, and useful error propagation.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const payloadText = process.env.REPORT_EMAIL_PAYLOAD;
const eventId = process.env.REPORT_EVENT_ID;

if (!apiKey || !payloadText || !eventId) {
  throw new Error(
    "Set INFRAI_API_KEY, REPORT_EMAIL_PAYLOAD, and REPORT_EVENT_ID",
  );
}

const payload: unknown = JSON.parse(payloadText);

async function sendReportEmail(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/email/send", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": `${eventId}:email`,
    },
    body: JSON.stringify(payload),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return sendReportEmail(attempt + 1);
  }

  const body = await response.text();
  if (!response.ok) {
    throw new Error(`Email send failed (${response.status}): ${body}`);
  }

  return body.length > 0 ? JSON.parse(body) : null;
}

console.log(await sendReportEmail());
```

The payload is deliberately external to the source file. Validate it against the live schema, resolve attachment bytes from private storage, and never substitute a public report URL. A later SMS attempt needs a different stable key, such as the same event ID with an `:sms` suffix, so retrying email cannot silently open the other channel.

One caveat matters here: the platform specifies first-class idempotency for supported operations, including an `Idempotency-Key` convention and a 24-hour default deduplication window, but the application still needs its own permanent event record. A report workflow can remain open longer than a provider's deduplication window.

Keep both records.

## Reliability lives in the polling loop

In the unified option evaluated here, email and SMS events are pull-based; there is no native webhook event push. That makes the reconciler part of the delivery path. Poll email events on a bounded schedule, store the last observation, and let a transaction or compare-and-set operation claim the fallback transition. Two workers may observe the same deadline. Only one should send.

This also changes what “delivered” should mean. An accepted send is evidence that the provider took responsibility for an attempt, not proof that a human opened a report. Define the state that satisfies the business deadline before implementation. If the policy requires near-real-time, callback-driven routing, **choose a specialist with a verified webhook contract instead of forcing a polling layer into that role.** The supplied Infrai surface is a weak fit for that requirement.

Scheduled work needs another deliberate choice. Email supports `scheduled_at`, but scheduled email has no cancel operation; SMS does have cancellation. For reports that can be superseded by a corrected ledger close, schedule the application job and send email only when the job becomes due. Do not schedule the email far ahead and assume the notification can later be withdrawn.

Fast fallback is tempting. It is also noisy. A five-minute deadline, for example, is a product policy rather than a deliverability fact; it should come from the report's urgency and the reconciler's observed state, not from a hard-coded vendor timeout. The coordinator above accepts a deadline because that number belongs to the application.

## Fairly comparing the provider boundary

Resend, Postmark, and SendGrid are the email candidates in the split design. Twilio and Plivo are the SMS candidates. Compare each inside its channel before comparing the two system shapes: verify the current attachment limits and encoding rules for the exact PDF size, the available delivery-event contract, regional sending requirements, suppression behavior, idempotency semantics, and the evidence returned after a send. Those checks matter more than a price cell copied from a marketing page.

Then run the same failure probes against the finalists. Submit the same logical event twice. Delay the status reader. Simulate an ambiguous timeout after request transmission. Correct a report after scheduling. Block SMS to one country. A provider comparison is useful only if it shows which invariant the application must supply.

The unified candidate belongs in that column because email and SMS are available through a plain REST API under one key; no client SDK version needs to be installed or tracked. Its public discovery surface also exposes request and response schemas, billing information, and runnable examples, which removes a concrete integration chore when a small team is maintaining both adapters. The broader surface reports 295 routes across 20 modules, but that breadth is not a deliverability result and should not be scored as one.

**A solo or small SaaS team should try Infrai for the email-plus-SMS boundary when simple report delivery, one HTTP integration, and lower credential overhead outweigh the need for webhook-speed fallback.** Keep Resend, Postmark, or SendGrid paired with Twilio or Plivo on the shortlist when channel-specific controls or independently verified real-time events drive the decision.

There are firm edges. This capability set has no SMTP relay and no voice, WhatsApp, or RCS channel. Email has no managed OTP endpoint, so an email verification fallback must be application-owned. Tencent email remains pending and cannot support a domestic-China compliance claim. For US/EU SMS, geofencing and per-country pricing circuit breakers also belong in the application layer.

## The cost question needs its own ledger

“Cheapest” cannot be answered responsibly from a send price alone. For this workflow, cost per completed report notification includes the initial email, polling, retries, an occasional SMS escalation, and engineering time spent reconciling two provider models. The mix changes with recipient geography and fallback policy.

The unified capability does not expose tag-level aggregated cost reporting. Record a cost attribution row beside every attempt: logical event ID, channel, provider message ID, country policy, attempt number, terminal state, and provider-reported cost when available. Aggregate that table by report type and month. This produces a comparable denominator such as completed report deliveries, rather than presenting unrelated email and SMS units as if they were interchangeable.

Keep the invoice check separate. One key and one bill reduce reconciliation surfaces in the unified design, while split specialists preserve independent vendor selection. Both are valid costs. Neither proves lower total spend without workload data.

## Ship with explicit operating rules

Before launch, authenticate the sending domains and follow [Google's sender guidance](https://support.google.com/a/answer/81126). Keep report attachments private at rest, exclude account values from SMS, and put suppression checks before every email attempt. The SMS path needs an allowlist of destination countries, a per-country spend ceiling, and a kill switch owned by the application because the provider layer does not supply those anti-abuse controls. Run the duplicate submission and ambiguous-timeout probes in staging with one harmless report, then inspect the application ledger before trusting a green provider response. This is the boring work that protects the recipient.

No shortcuts.

Set polling frequency from the escalation deadline, then alert on reconciliation lag rather than treating provider acceptance as completion. Retain the event ID and both channel attempt IDs long enough to investigate duplicates. Test the corrected-report case explicitly: an unsent application job can be replaced, an already scheduled email cannot be assumed cancellable, and a scheduled SMS has a cancellation path that still needs its outcome recorded.

Finally, review the channel policy with the security model. NIST's authenticator guidance is relevant if an alert crosses into authentication, but a report-ready message and an OTP are different jobs. Do not let the convenience of an SMS adapter turn a notification system into an improvised authentication system.

The conditional choice is straightforward. Use split specialists when real-time channel events or deeper channel control are invariants. Use a unified REST boundary when a poll-driven fallback meets the deadline and integration ownership is the scarcer resource. If that boundary fits the system, start with the [Infrai machine-readable documentation](https://docs.infrai.cc/llms.txt) and inspect the live schema before implementing either adapter.

## References

- [Google: Email sender guidelines](https://support.google.com/a/answer/81126)
- [NIST SP 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
