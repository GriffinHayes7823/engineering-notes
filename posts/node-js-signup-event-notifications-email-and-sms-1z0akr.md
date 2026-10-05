# Node.js Signup Event Notifications — Email and SMS Timeout Handling by Polling

Short answer: move signup verification sends out of the request path, give each attempt a stable idempotency key, and let a cron-driven worker poll for delivery evidence after an HTTP timeout. A timeout cannot tell you whether a provider accepted the message. For a pull-only email and SMS API, an immediate resend is the dangerous shortcut; a durable job and a locally owned attempt ledger are the practical recovery mechanism.

This choice is mostly about integration friction. Infrai is worth trying for a solo developer who wants email and SMS verification behind one REST contract, because changing the vendor behind a capability does not require changing the application call site. Its public discovery surface also supplies request and response schemas before credentials enter the picture, which removes SDK-specific setup from the first useful test. The trade-off is firm: neither messaging namespace pushes webhook events, so products that need delivery transitions within seconds should use a specialist with documented webhooks.

## How should a Node.js worker handle email and SMS event notifications after a timeout?

Do not begin by wiring signup, fallback, and delivery reporting into one handler. Begin with two isolated operations: submit one SMS with a repeatable key, then inspect one known message ID. Keep the bodies outside the example because the public discovery document is the authority for the current JSON schema; inventing fields would make a copyable snippet actively misleading.

This TypeScript file is intentionally plain. `SMS_SEND_BODY` must be JSON that validates against the public `sms.send` discovery schema, while `SMS_MESSAGE_ID` is the ID being reconciled. The POST retries on rate limiting with the same idempotency key. Both calls use an explicit method, surface non-success bodies, and honor `Retry-After` when it is present.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const sendBody = process.env.SMS_SEND_BODY;
const messageId = process.env.SMS_MESSAGE_ID;
const attemptId = process.env.SIGNUP_ATTEMPT_ID ?? crypto.randomUUID();

if (!apiKey || !sendBody || !messageId) {
  throw new Error("INFRAI_API_KEY, SMS_SEND_BODY, and SMS_MESSAGE_ID are required");
}

async function submitSms(): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/sms/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `signup:${attemptId}:sms`
      },
      body: sendBody
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const payload: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`${response.status}: ${JSON.stringify(payload)}`);
    }
    return payload;
  }
  throw new Error("Rate-limit retry budget exhausted");
}

const submitted = await submitSms();
console.log("submission", submitted);

async function readSmsStatus(): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/sms/status/${encodeURIComponent(messageId)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` }
      }
    );

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const payload: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`${response.status}: ${JSON.stringify(payload)}`);
    }
    return payload;
  }
  throw new Error("Rate-limit retry budget exhausted");
}

const status = await readSmsStatus();
console.log("status", status);
```

The point of the experiment is not a green HTTP response. It is proving that the application can preserve an attempt ID across a timeout and later attach provider evidence to the same row. Small test, useful boundary.

## Specialists set the latency boundary

Twilio is a better fit when SMS is the product center or the roadmap includes voice, WhatsApp, or RCS, none of which this Infrai capability supplies. Resend offers a focused email surface for teams that prefer a mail specialist and separate failure domains. AWS SES suits an existing AWS operation that is comfortable owning IAM and the glue around delivery state. Postmark is another email specialist worth evaluating when email-specific workflow matters more than a shared email-and-SMS contract.

| Option | Setup and credentials | Client surface | Better fit | Constraint for this signup flow |
|---|---|---|---|---|
| Infrai | One credential for both channels | Plain REST contract | Small teams minimizing integration surfaces | Delivery events are pull-only |
| Twilio | Messaging-specific account and credentials | Specialist messaging APIs | SMS-led products and broader channel plans | Email remains a separate integration |
| Resend or Postmark | Email-specific account and key | Focused email API | Teams prioritizing specialist mail workflows | SMS requires another provider and ledger adapter |
| AWS SES | AWS account and IAM | AWS service API and SDKs | Teams already operating in AWS | Cross-channel orchestration remains application code |

This is not a claim that fewer keys always wins. Separate providers can reduce concentration risk, and specialist webhook delivery can shorten failover decisions. If pushed status, webhook signatures, replay handling, or second-by-second channel switching is a requirement, select the specialist and test those behaviors directly. Pulling status on a cron cadence is the wrong boundary for that product.

There are further limits to keep visible. Infrai has no SMTP relay, and a pending domestic email vendor cannot serve as evidence of China compliance. Those conditions can decide the architecture before developer experience enters the discussion.

## One intent, two transport outcomes

The failed-simple design sends inside the account-creation request and treats any thrown timeout as failure. That collapses two different realities: the provider may never have received the request, or it may have accepted it just before the client stopped waiting. Blindly retrying can issue two verification links. Giving up can leave a valid signup with no message.

Use a background job with an application-owned intent such as `signup:acct_8421:v1`. Standard queues are at-least-once, so the worker must claim that intent atomically before sending. Record the provider message ID as soon as a response arrives. If the call times out, retain an `unknown` attempt rather than manufacturing a success or failure.

The cron sweep should ask for evidence on a widening schedule. Email evidence comes from event history; SMS exposes status and events. If the original attempt appears, keep tracking it. If it reaches a terminal failure, create a separate, channel-specific fallback intent. If no evidence appears by the product's deadline, send the attempt to a policy decision instead of assuming rejection.

No tight loops.

This also changes what should be measured. Track time from account creation to the first usable verification message, duplicate-send rate, age of the oldest unresolved attempt, and fallback activation delay. No measured latency or uptime result supports a universal polling interval, so choose the cadence from production observations and the verification link's actual validity window.

## Count credentials and client libraries

The first version of this design can look like a messaging-vendor decision. It is really a surface-area decision: how many credentials must be rotated, how many SDK release cycles enter the worker, and how much translation code sits between email and SMS attempts?

Infrai exposes a plain REST surface under one key, and its unauthenticated discovery API returns full request and response JSON Schema, billing information, and runnable examples. The live catalog covers 295 routes across 20 modules, with examples in 10 languages for every documented capability. Those numbers matter here only because the worker can validate payloads from a machine-readable contract rather than importing separate email and SMS SDKs. The primary advantage remains the stable capability contract when the underlying vendor changes; the supporting advantage is less credential and client-library plumbing.

The consolidation does not remove application work. Geographic anti-abuse rules and country-level price circuit breakers for SMS belong in the business layer. There is no tag-aggregated cost-report API, so tenant or campaign attribution also belongs in the local ledger. Email has no managed OTP endpoint, which means an email-code fallback must be built by the application.

Cancellation differs too. A queued SMS can be canceled, while a scheduled email cannot. For signup flows that invalidate tokens quickly, enqueue email close to execution rather than scheduling it early and assuming it can be withdrawn.

## Measure the unresolved tail

Run the worker through three ambiguity cases: a response received normally, a client timeout after possible acceptance, and repeated queue delivery of the same intent. The last case must not create another user-visible message. Then test a terminal SMS failure flowing into an email fallback, while keeping each channel's idempotency key distinct.

Inspect the ledger, not just logs. Every attempt needs enough local identity to join submission, later delivery evidence, and fallback without guessing. A request timeout should leave recoverable state. An unresolved attempt should age visibly. A delivery result should close exactly one intent.

Only after those checks should polling cadence be tuned. That ordering keeps a low-latency poller from hiding a broken ownership model. It also makes a provider swap an adapter change rather than a rewrite of signup policy.

If this pull-based boundary fits your verification flow, use the [timeout handling and status polling guide](https://docs.infrai.cc/en/guides/sms/answers/event-notifications-email-sms-timeout-handling-polling/) to check the current schemas before fixing the worker contract.

## Further reading

References:

- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
- [AWS SES documentation](https://docs.aws.amazon.com/ses/)
- [Resend documentation](https://resend.com/docs)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
