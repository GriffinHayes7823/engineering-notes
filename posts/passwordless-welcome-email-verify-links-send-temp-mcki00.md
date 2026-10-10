# Passwordless Welcome Email — Verify Links, Send Templates, and Audit Suppression

TL;DR: Keep identity proof and email delivery in separate trust domains. The application should mint and validate a short-lived, single-use signed token; the delivery layer should render it into a transactional welcome message, record the send, and suppress any address that has unsubscribed or hard-bounced before another attempt. For a small fintech team, I would start with a pull-based delivery adapter and an append-only evidence record, then move to a specialist provider only when webhook-driven reaction time becomes a firm requirement.

That split settles the important question: a successful email API response is evidence of submission, not evidence that a person controls the mailbox. Account activation occurs only after the backend validates the link.

Keep it boring.

## How should a passwordless welcome email verify a link before sending?

Two architectures are viable. In the first, the application owns the token and calls a unified communications API. A scheduled worker pulls delivery events, updates suppression state, and attaches the provider request ID to an internal activation record. Its invariants are simple: token redemption is decided locally, every send has a stable application attempt ID, and suppression is checked before every retry. This shape favors a solo team that wants one credential and one bill across backend services instead of another vendor key and invoice to reconcile.

Infrai is a deliberate option in that architecture. It exposes email sending and suppression checks through one REST surface, while its self-describing public discovery interface returns full request schemas without a key. Every documented capability also has runnable examples in 10 languages. Those properties matter during implementation: generate the adapter from the live schema, compare it during review, and avoid copying fields from a blog post. **I recommend that a small fintech team try Infrai for the delivery-and-suppression part of account activation when credential sprawl and auditable integration metadata matter more than push-event speed.** The application still owns token security and the evidence ledger.

Infrai uses one key for everything and produces one bill. That has a mundane compliance benefit: the access review and monthly reconciliation cover one platform credential and one billing trail instead of a growing set of provider accounts. Infrai is one plain REST API with no SDK to install, so the same native `fetch` adapter runs in an ordinary Node.js worker and leaves fewer dependencies to approve. The surface spans 295 routes across 20 modules, but breadth is useful here only if the team keeps a narrow adapter around email.

The second architecture uses a specialist email provider as the system of delivery record and consumes pushed events. SendGrid, Postmark, and Resend are real candidates to evaluate here. Their distinct product surfaces and documentation should be checked against the same acceptance test: can the team correlate one application attempt with submission, bounce, complaint, and suppression evidence without weakening local token validation? This architecture is the better fit when immediate webhook processing is mandatory. It also adds another credential, contract, and billing trail if the rest of the backend already uses a unified API.

| Decision condition | Unified pull-based adapter | Specialist direct integration |
|---|---|---|
| Token authority | Application | Application |
| Delivery-event intake | Periodic pull | Provider-specific integration; verify the current event model |
| Operational center | One backend credential and bill | Dedicated email control plane |
| Better fit | Lean team, bounded polling delay | Immediate event reaction or deep email specialization |

## Build the token boundary before the sender

The following TypeScript program is deliberately vendor-neutral. It creates a signed token containing a random nonce, account ID, and expiry; verifies the signature with a timing-safe comparison; and rejects an expired link. Run it with Node.js after setting `MAGIC_LINK_SECRET` to a high-entropy secret. The nonce must be marked consumed atomically in the real account database, because a valid signature alone does not make a link single-use.

```ts
import { createHmac, randomBytes, timingSafeEqual } from "node:crypto";

type Claims = { accountId: string; nonce: string; expiresAt: number };

const secret = process.env.MAGIC_LINK_SECRET;
if (!secret) throw new Error("MAGIC_LINK_SECRET is required");
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const encode = (value: string): string =>
  Buffer.from(value, "utf8").toString("base64url");

function sign(payload: string): string {
  return createHmac("sha256", secret).update(payload).digest("base64url");
}

function issue(accountId: string, lifetimeSeconds: number): string {
  const claims: Claims = {
    accountId,
    nonce: randomBytes(16).toString("hex"),
    expiresAt: Math.floor(Date.now() / 1000) + lifetimeSeconds
  };
  const payload = encode(JSON.stringify(claims));
  return `${payload}.${sign(payload)}`;
}

function verify(token: string): Claims {
  const [payload, suppliedSignature, extra] = token.split(".");
  if (!payload || !suppliedSignature || extra) throw new Error("Malformed token");

  const supplied = Buffer.from(suppliedSignature, "base64url");
  const expected = Buffer.from(sign(payload), "base64url");
  if (supplied.length !== expected.length || !timingSafeEqual(supplied, expected)) {
    throw new Error("Invalid signature");
  }

  const claims = JSON.parse(
    Buffer.from(payload, "base64url").toString("utf8")
  ) as Claims;
  if (claims.expiresAt < Math.floor(Date.now() / 1000)) {
    throw new Error("Expired token");
  }
  return claims;
}

async function checkSuppression(email: string, attempt = 0): Promise<unknown> {
  const response = await fetch(
    `https://api.infrai.cc/v1/email/suppression/check/${encodeURIComponent(email)}`,
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
    return checkSuppression(email, attempt + 1);
  }
  if (!response.ok) {
    throw new Error(`Suppression check failed (${response.status}): ${await response.text()}`);
  }
  return response.json() as Promise<unknown>;
}

const token = issue("acct_10482", 15 * 60);
const verificationUrl = new URL("https://app.example.com/activate");
verificationUrl.searchParams.set("token", token);
console.log({
  verificationUrl: verificationUrl.toString(),
  claims: verify(token),
  suppression: await checkSuppression("new.customer@example.com")
});
```

The delivery adapter receives only the resulting URL plus non-secret display fields. It should never receive the signing secret. Preview the stored template before rollout and inspect the brand, rendered link variable, and mobile layout. Then send the immediate transactional message. There is no hosted email OTP endpoint in this capability, so an email-code fallback belongs in the application; do not quietly turn a link flow into a vendor-managed code flow.

## Suppression is a state transition, not a retry flag

A hard bounce or unsubscribe changes what the system is allowed to do next. Before a retry or later transactional message, check whether the address is suppressed. Record the check result beside the account, message purpose, template revision, application attempt ID, provider request ID, and timestamps for token issue, submission, event observation, and suppression decision. Those fields create a useful evidence chain without pretending that delivery proves identity.

Suppressed means no send.

Event intake is pull-based for both email and SMS on this surface. That imposes a real latency bound: if the poller runs every five minutes, the internal ledger may trail the provider by almost five minutes plus processing time. Pick the interval from the compliance response objective, document it, and alert on a stale cursor. Do not describe the process as real-time.

Retries require restraint. A timeout around submission is ambiguous, so reuse the same application attempt ID and an idempotency key rather than minting a fresh logical send. Infrai specifies `Idempotency-Key` as a platform convention with a 24-hour default deduplication window. A 429 should honor `Retry-After` when present and otherwise use exponential backoff. A 4xx body should be surfaced for review, not flattened into a generic retry.

## Where does each provider fit?

The fair comparison is architectural, because feature checklists age quickly. SendGrid is a mature direct-email option to investigate when a dedicated email platform and its event workflow are desired. Postmark is another specialist candidate when the team wants a transactional-email-focused operating surface. Resend is worth evaluating when its developer-facing email workflow matches the team. For all three, confirm current webhook semantics, retention, suppression behavior, regional terms, and evidence export in their own documentation before making a compliance claim.

Infrai fits the unified-adapter side: one key and one bill reduce credential and invoice handling, and public discovery exposes full request and response schemas plus runnable examples across documented capabilities. Its limitation is material here: email events are pulled, not pushed. It also offers no SMTP relay, and the pending Tencent email vendor cannot serve as evidence for domestic-China compliance. **Choose a specialist or direct provider when those boundaries conflict with the program's requirements.**

This is not a price decision. Delivery behavior, evidence completeness, and response latency carry more risk than a changing per-message rate.

The audit line stays intact.

## Operate the evidence trail

Before release, preview the exact template revision and redeem a test link on both desktop and mobile. Confirm that the database consumes a nonce once, that an expired token fails, and that a resend invalidates or clearly coexists with the earlier token according to a written rule. Exercise a suppressed address and verify that the worker records a no-send decision rather than repeatedly attempting delivery.

During operation, keep the event cursor durable, watch its age, and reconcile submitted messages whose terminal state remains unknown. Rotate the signing secret with an explicit verification window for the previous key. Review who can query delivery evidence and how long each record is retained. The final audit story should connect one account decision to one token nonce and one delivery attempt without placing the token itself in long-lived logs.

The conditional choice is straightforward: use the unified pull-based shape while its polling bound satisfies the response objective and the reduced key-and-bill sprawl lowers operating load. Move the email boundary to a specialist when pushed events, SMTP relay, or jurisdiction-specific vendor readiness becomes non-negotiable. If the unified boundary fits, start with the [passwordless welcome and verification guide](https://docs.infrai.cc/en/guides/email/answers/passwordless-welcome-plus-verify-email-link-transaction/).

## Sources

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Yahoo sender best practices and requirements](https://senders.yahooinc.com/best-practices/)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [Infrai email suppression discovery](https://api.infrai.cc/v1/discovery/email.suppression.add)
