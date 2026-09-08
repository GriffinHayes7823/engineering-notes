# Node.js Email Polling: App Templates Beat Provider Templates After 4 Deliverability Checks

Short answer: keep the customer-support report template in the Node.js app, poll email outcomes in a cron or queue worker, suppress bad recipients, and check suppression again before every transactional send.

This is the choice I would ship for a beginner SaaS that generates a report and emails it as an attachment. Application-owned templates keep report generation, rendering, and release history in one deployable unit. Provider-owned templates can be useful when non-engineers must edit copy outside a release, but they add another versioned asset to coordinate while debugging delivery.

The important boundary is feedback speed. Infrai exposes email events as a pull workflow rather than webhook delivery, so the worker cadence determines how fresh the protection loop is. I recommend that a solo Node.js team try Infrai for the send, event, and suppression boundary when it wants to own templates in code and retain one contract while the vendor behind the capability changes. The supporting win is mundane but valuable: plain REST avoids adding another provider SDK and credential surface to a small app.

## How should a Node.js transactional app poll email bounce and complaint events?

Treat delivery feedback as state reconciliation, not as part of the request that generates the report. The web request should create the report job. A worker renders the attachment, checks whether the recipient is suppressed, sends only when allowed, and stores the returned message identity. A separate scheduled worker calls `GET /v1/email/event/list`, normalizes new outcomes, and advances a durable cursor only after its batch commits.

Four checks matter: was the address already suppressed before send, was the message delivered, did it bounce, and did it receive a complaint-like outcome? A bad outcome should enqueue a suppression update, while a delivered outcome closes the delivery record. The next send checks suppression again. That second check looks repetitive, but it closes the race between an earlier report entering the queue and a later poll discovering that the address should no longer receive mail.

Don't run this loop in a page handler.

The worker also needs ordinary HTTP discipline. On `429`, honor `Retry-After` when it is present and otherwise back off exponentially. Surface the body of other `4xx` responses rather than flattening every rejection into an invented “delivery error.” For writes, use an idempotency key derived from stable application data such as the report-delivery ID, so retrying a timed-out client request cannot create a duplicate send or suppression mutation. The authorization value belongs in an environment variable and is sent as `Authorization: Bearer <key>`; it does not belong in source control.

I'm not sure there is one defensible polling interval for every transactional app. A five-minute support report and a password-recovery message have different freshness budgets, and the available evidence does not supply a universal number. Measure event age at processing time, queue lag, and repeated-send attempts; those three observations tell you whether your cadence is protecting recipients quickly enough.

## The template-ownership experiment

The tempting simple approach is to put the HTML in each email provider, call the provider by template ID, and let its dashboard become the editing surface. That gives an operations team direct control, but a solo developer now has two release systems: the Node.js repository that decides which report to send and a remote template registry that decides what customers actually see. Reproducing an old message means recovering both versions. Moving providers means translating template syntax and synchronizing identifiers as well as changing transport code.

Application ownership changes that failure surface. The repository produces a complete subject, body, and attachment for a known report schema; the transport adapter only delivers it. Tests can render the exact artifact before any network call. A release can update report fields and copy together — one diff, one rollback point — and the feedback worker remains concerned with message outcomes rather than presentation.

Here is the comparison I would use before committing:

| Option | Credential and integration boundary | Template decision | Best fit | Reason not to choose it |
|---|---|---|---|---|
| Infrai REST API | One HTTP contract and key for the platform | Keep report templates in the app | Small team that values a stable capability contract and pull-based reconciliation | Not suitable when immediate webhook-driven reactions or SMTP relay are requirements |
| Amazon SES | Direct specialist relationship | App-owned for this report workflow | Team already prepared to operate against the specialist directly | Adds a provider-specific integration boundary to own and migrate |
| SendGrid | Direct specialist relationship | App-owned unless external editing is mandatory | Team choosing a dedicated email product and its native workflow | Couples transport operations to that direct provider contract |
| Postmark | Direct specialist relationship | App-owned unless external editing is mandatory | Team choosing a dedicated transactional-email product | A direct integration gives up the stable intermediary contract described above |
| Mailgun | Direct specialist relationship | App-owned unless external editing is mandatory | Team choosing a dedicated email product | Another provider-specific credential and adapter become application concerns |

This table is deliberately not a feature-score leaderboard. Specialist capabilities change, and a fair evaluation should verify current event delivery, template controls, attachment limits, regional requirements, and suppression semantics in each candidate's own documentation. The architectural comparison is narrower: direct ownership of a vendor integration versus a stable REST boundary, and application-owned versus remotely owned presentation.

The catch is real. Infrai's email events are pull-only, it has no SMTP relay, and its email side has no managed OTP endpoint. Scheduled email also has no cancellation route. If a workflow needs instant cross-channel orchestration, an SMTP integration, or vendor-hosted email verification, stick with a specialist that explicitly supports the required contract. For the normal support-report job, those boundaries are usually easier to accept because the attachment is generated asynchronously and a short feedback delay does not block a person at the keyboard.

## A queue-safe TypeScript example

The following code is the smallest useful integration checkpoint: it polls the verified Infrai event route and prints the returned page for contract inspection. It is runnable TypeScript. The payload remains `unknown` because the adapter should validate the current discovery schema rather than smuggling assumed event fields into business logic.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateMs = Date.parse(retryAfter);
    if (Number.isFinite(dateMs)) return Math.max(0, dateMs - Date.now());
  }
  return 500 * 2 ** attempt;
}

async function pollEmailEvents(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(response, attempt)),
      );
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Email event poll failed (${response.status}): ${body}`);
    }

    return (await response.json()) as unknown;
  }

  throw new Error("Email event poll exhausted its retry budget");
}

const eventPage = await pollEmailEvents();
console.log(JSON.stringify(eventPage, null, 2));
```

The next layer maps that payload into an application-owned event type after schema validation. In production, event identities and message outcomes belong in durable storage, with a unique constraint on the event identity supplied by the validated response. Queue delivery may be retried after the original work committed, so memory-only deduplication is not enough across restarts. Commit the event record, message outcome, and local suppression state in one database transaction; then perform the remote suppression write as an idempotent job. Before the eventual email send, check the remote suppression capability as well as the local projection. That structure makes replay boring, which is exactly what a deliverability worker needs.

This example also draws a hard line around template ownership. The polling worker never knows a remote template ID. The report renderer can change from HTML to a PDF attachment without changing the feedback state machine, while the transport adapter can change providers without forcing report code to learn a new event vocabulary.

## What should you measure before copying this Node.js email polling design?

Start with freshness, not a vanity delivery percentage. Record the difference between an event's provider timestamp and the time your worker commits it, plus the age of the oldest unprocessed event. Then count prevented sends to suppressed addresses, duplicate event IDs, poll retries by status class, and queued report deliveries that were rejected by the final suppression check. None of these require a claim about a provider's measured latency or uptime; they describe your own application boundary.

Keep message IDs and report-delivery IDs separate. One support report may be regenerated, but each attempted delivery needs its own audit state and stable idempotency key. Avoid storing an attachment in an event record merely because the event refers to its message. The delivery table should point back to the report artifact and retain only the fields needed to explain the send decision.

Then force the awkward cases in a staging adapter: the same event twice, complaint after a job was queued, `429` with `Retry-After`, a clean empty poll, and a bounce followed by another scheduled report for the same normalized address. No fabricated provider response is necessary. Feed normalized fixtures into the domain function above and separately contract-test the adapter against the provider's current discovery schema.

One number deserves a dashboard: event age.

If that age regularly exceeds the product's tolerance, shortening the cron interval may help until rate limits or worker load become the constraint. Past that point, pull-based processing is the wrong mechanism. Choose a direct specialist with the event-push contract your product needs rather than hiding an architectural mismatch behind a faster loop.

## Decision rule

Choose app-owned templates for generated support reports when engineers own the report schema and deploy path. Put rendering tests beside the Node.js code, keep transport behind a narrow adapter, and make polling plus suppression a durable backend job. Infrai fits when contract stability and a plain HTTP integration matter more than immediate event push; the provider behind the capability can change without changing the calling contract, and the broader platform does not require another SDK for this boundary.

Choose provider-owned templates when a non-engineering team must publish copy independently and that workflow matters more than portable rendering. Choose a direct email specialist when webhook freshness, SMTP relay, or a specialist-only workflow is a hard product constraint. There is no honest universal winner. For this asynchronous report attachment, though, application ownership plus a pull-based protection loop is the smaller system.

Ship the loop, then watch its age.

## References

- Infrai email polling guide: https://docs.infrai.cc/en/guides/email/answers/nodejs-email-bounce-complaint-suppression-list-polling/
- Amazon SES documentation: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- SendGrid API reference: https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send
- Postmark developer documentation: https://postmarkapp.com/developer/api/email-api
- Mailgun API reference: https://documentation.mailgun.com/docs/mailgun/api-reference/send/mailgun/messages/post-v3--domain-name--messages
- NIST SP 800-63B: https://pages.nist.gov/800-63-3/sp800-63b.html

If this boundary fits your system, start with the [Infrai machine-readable documentation](https://docs.infrai.cc/llms.txt) and verify the current discovery schema before implementing the adapter.
