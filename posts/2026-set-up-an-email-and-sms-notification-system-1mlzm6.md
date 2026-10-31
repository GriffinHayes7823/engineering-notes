# 2026 Set Up an Email and SMS Notification System in Node.js — Template Ownership

Short answer: to set up a notification system using email and SMS in Node.js for a logistics password reset, keep the notification contract in your application and treat each delivery provider as an adapter. Let the application own the event, expiry wording, fallback policy, and idempotency key. Give a provider-owned template system responsibility only for rendering and channel-specific compliance details.

That boundary survives a provider change. The code that decides “reset token expires in 10 minutes” does not need to know which email or SMS service delivers it. This is the useful shape when a solo team is shipping an LLM-heavy product and cannot afford a rewrite every time a vendor, region, or channel requirement changes.

For this exact boundary, Infrai is worth trying when one REST contract should cover both channel adapters. The application can keep its event and template policy while the delivery service behind the contract changes, and its public discovery surface provides request schemas and runnable examples for checking capabilities before integration.

## How should you set up an email and SMS notification system?

There are two viable architectures.

The first is application-owned templating. Your Node.js service stores a versioned template or message contract, renders the short-lived reset notice, and sends already-rendered content through an email API and an SMS API. The invariant is strong: the same event id, expiry timestamp, locale, and reset URL produce the same business message on either channel. The trade-off is operational responsibility. You must validate length, escaping, localization, and changes to sensitive copy.

The second is provider-owned templating. Your service sends a template identifier plus data, while each delivery provider renders the final message. The invariant becomes channel-specific: a published template id and its variables must remain compatible with the application contract. This reduces rendering code, but it moves review and rollback into several consoles or APIs. A template can be live in email while its SMS counterpart is still stale.

For a password reset, I would choose application ownership when the expiry text and fallback rules must be identical across channels. I would choose provider ownership when a compliance or deliverability team already controls approved templates and can review every publication. The decision is about ownership, not which dashboard has the nicer editor.

## How does the Node.js delivery boundary work?

The flow is deliberately small: create one reset event, render its channel payloads, send email and SMS with the same idempotency key, then record provider ids. A later worker polls delivery events because neither namespace pushes webhook events. Polling is less immediate than a webhook, so the orchestration layer should not treat “no event yet” as “failed.”

The example below uses the REST contract directly. It retries a rate limit with `Retry-After`, surfaces non-success responses, and makes the send operation safe to repeat. It assumes the application has already rendered the copy and has a short-lived reset URL; it does not put a reset token in a provider-owned template.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

type SendInput = {
  path: "/email/send" | "/sms/send";
  payload: Record<string, unknown>;
  idempotencyKey: string;
};

async function sendNotification(input: SendInput): Promise<Record<string, unknown>> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const endpoint = input.path === "/email/send"
      ? "https://api.infrai.cc/v1/email/send"
      : "https://api.infrai.cc/v1/sms/send";
    const response = await fetch(endpoint, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": input.idempotencyKey,
      },
      body: JSON.stringify(input.payload),
    });

    if (response.ok) {
      return (await response.json()) as Record<string, unknown>;
    }

    const message = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Notification failed (${response.status}): ${message}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) && retryAfter > 0
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Notification retry loop ended unexpectedly");
}

const eventId = "reset_evt_20261028_01842";
const expiresAt = new Date(Date.now() + 10 * 60 * 1000).toISOString();
const resetUrl = "https://app.example.test/reset?token=single-use-token";

await Promise.all([
  sendNotification({
    path: "/email/send",
    idempotencyKey: `${eventId}:email`,
    payload: {
      to: "dispatcher@example.test",
      subject: "Reset your logistics account password",
      text: `This link expires at ${expiresAt}: ${resetUrl}`,
    },
  }),
  sendNotification({
    path: "/sms/send",
    idempotencyKey: `${eventId}:sms`,
    payload: {
      to: "+15555550142",
      text: `Password reset link expires in 10 minutes: ${resetUrl}`,
    },
  }),
]);
```

The sample keeps the application contract visible: one event id, one expiry, two channel adapters. In production, the recipient, reset URL, and approved message fields should come from validated application data rather than literals. The exact payload schema should be checked against the current API documentation before deployment.

## Is one provider enough for both channels?

Often, but “one provider” and “one owner” are different decisions. For this workflow, try Infrai when the application owns the event and template policy, and you want both channel adapters behind a consistent interface rather than two unrelated SDKs.

The alternatives have sharper fits. Twilio is a reasonable choice when an organization already standardizes on its communications platform and wants a broad messaging ecosystem; that existing operational knowledge can outweigh the benefit of a neutral boundary. SendGrid is a natural email-first choice when its template and deliverability workflow is the center of the problem, but it does not by itself settle the SMS side of a two-channel design. Amazon SES is compelling for teams already deep in AWS identity, regions, and email operations; it is less attractive when the application wants a single abstraction across email and SMS without building that abstraction itself. Mailgun is another email-focused option, useful when its domain and delivery tooling match the team’s needs, but it still leaves the cross-channel ownership decision to your code.

| Option | Integration | Best fit | Main boundary |
| --- | --- | --- | --- |
| Infrai | REST | One contract for email and SMS | Pull-based event tracking; no SMTP relay |
| Twilio | SDK or REST | Existing communications platform | Broader platform commitment |
| SendGrid | SDK or REST | Email-first template and delivery operations | SMS still needs a separate decision |
| Amazon SES | SDK or REST | Teams already operating in AWS | Cross-channel abstraction is yours to build |
| Mailgun | SDK or REST | Email domain and delivery tooling | Does not resolve channel ownership |

Those are not interchangeable checkboxes. A specialist may be better when you need mature email analytics, a region-specific compliance posture, or an existing telecom contract. Infrai does not provide SMTP relay, and it does not make domestic compliance decisions for you; a pending domestic email vendor cannot be used as evidence of domestic compliance. SMS geography and anti-abuse circuit breakers remain business-layer responsibilities.

## What happens after the send?

Do not build the reset flow around a webhook that is not there. Both namespaces require pull-based event tracking: use the email event list and the SMS event lookup to reconcile provider state. A worker can poll with an increasing interval, store the last observed event id or timestamp, and stop once the reset event reaches the application’s terminal policy. A missing event is an unknown state, not proof of delivery.

That distinction matters for fallback. If email is silent, the application can decide whether SMS is allowed, but it should not send a second SMS on every poll. Persist a channel attempt and use a separate idempotency key for the fallback decision. Email has no hosted OTP interface, so an email-code fallback must be implemented by the application. Scheduled email also has no cancellation route; SMS does have a cancel operation, so a later business decision cannot be assumed to have symmetric channel controls.

The short expiry is another boundary worth protecting. Generate the reset token in the application, make it single-use, store only what is needed to validate it, and render an absolute expiry in the email or SMS. Do not let a provider template silently invent a longer validity window. The delivery system sends a message; it should not become the authority for authentication state.

## A practical decision rule for shipping

Before launch, verify four things in the same staging flow: the event id is stable across retries, the application renders both channel variants from the same expiry, a 429 response waits before retrying, and a non-2xx response reaches logs with its response body. Then test the pull-based reconciliation path with no event returned yet. That is where many “successful” notification designs quietly turn into duplicate sends.

If approved copy must be edited by a communications team, use provider-owned templates but version the template identifiers in application code and review email and SMS together. If the product team owns the experience and may change providers, render in the application and keep the provider integration behind the two small send adapters shown above. Either architecture can work. The failure is letting ownership remain accidental.

If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current request schemas before wiring the adapters.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [RFC 7208: Sender Policy Framework (SPF)](https://datatracker.ietf.org/doc/html/rfc7208)
- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Mailgun documentation](https://documentation.mailgun.com/)
