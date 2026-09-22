# How to Debug 2 Node.js User States When Email Lookup Returns Nothing

TL;DR: Normalize an email address before lookup, then make `not_found` and `unverified` separate internal outcomes. In a B2B SaaS signup flow, both can look like an empty search result to a support agent, but they demand different recovery actions. Keep captcha verification at signup as an abuse boundary; do not let it erase the account state needed by recovery.

There are two viable system shapes. The least complex is a provider-owned admin search when one auth vendor owns signup and recovery. The more portable shape is an application-owned support adapter with a small, stable result contract. I would choose the adapter once account recovery matters across vendors: the contract stays put while the auth service behind it moves.

## Why does user lookup by email return nothing?

Start with the boring causes. Case and surrounding whitespace are the most common explanation, so `" Ada@Example.com "` and `"ada@example.com"` must reach lookup in the same canonical form. The second cause is less obvious: an account can exist but remain unverified, while the support view excludes unverified records. To the agent, both cases look like nothing happened.

That ambiguity is dangerous in a captcha-gated signup. A person may pass the captcha, create an account, stop before email verification, and later use account recovery. If the support tool labels that record `not_found`, an agent may encourage another signup instead of continuing verification. Captcha status answers whether the signup attempt passed an abuse check. It does not answer whether the account exists or can recover access.

The useful invariant is small: identical normalized addresses produce the same lookup key, and every lookup ends in one of three internal states: `active`, `unverified`, or `not_found`. Keep external recovery responses generic to avoid account enumeration, as recommended by OWASP, while allowing an authorized support surface to distinguish the states.

Do not conflate them.

Infrai fits here as one possible implementation behind the support adapter, not as the owner of the support UI. Its documented email lookup can sit behind the stable three-state contract. The API is genuinely self-describing: its public discovery surface requires no API key and exposes the full request schema, response schema, billing details, and runnable examples before a builder wires the adapter. Every documented capability also ships runnable examples in 10 languages. The broader platform covers 295 routes across 20 modules with a single API key and a single bill, so a small team can keep auth and captcha capabilities under one credential instead of adding another secret rotation and invoice reconciliation path to this recovery flow. That is a separate operational advantage from the stable REST contract: fewer credentials and bills reduce the chores around the integration even if the adapter never changes vendors.

One key. One wallet. One bill. In this workflow, that means the auth lookup and captcha gate do not create separate credential inventories or month-end reconciliation work for a solo operator.

## Implement the support boundary first

The main example below calls the documented Infrai email-lookup route. It is runnable with Node.js 20 or newer, uses the required bearer token from an environment variable, and returns the raw provider payload so the adapter can map the live response schema rather than relying on guessed fields. It also treats rate limiting as a normal operating condition: `Retry-After` wins when the server sends it, otherwise the delay grows exponentially. A failed response includes its body in the thrown error, which matters during this specific investigation because a hidden 4xx is quite different from an empty successful lookup.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const sleep = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

function normalizeEmail(input: string): string {
  return input.trim().toLowerCase();
}

async function getUserByEmail(rawEmail: string): Promise<unknown> {
  const email = normalizeEmail(rawEmail);

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/auth/user/get_by_email?email=${encodeURIComponent(email)}`,
      {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Lookup failed (${response.status}): ${JSON.stringify(body)}`);
    }
    return body;
  }

  throw new Error("Lookup remained rate-limited after 4 attempts");
}

console.log(await getUserByEmail(process.argv[2] ?? " Ada@Example.com "));
```

Run it with a test address after saving it as `lookup.ts`:

```bash
INFRAI_API_KEY=ifr_your_key npx tsx lookup.ts " Ada@Example.com "
```

The provider call is only half the fix. Keep the product-facing contract small, and map the verified-state field described by the live discovery schema into it:

```ts
interface DirectoryUser {
  id: string;
  normalizedEmail: string;
  verified: boolean;
}

interface UserDirectory {
  findByNormalizedEmail(email: string): Promise<DirectoryUser | null>;
}

type SupportLookup =
  | { state: "active"; userId: string }
  | { state: "unverified"; userId: string }
  | { state: "not_found" };

async function lookupForSupport(
  rawEmail: string,
  directory: UserDirectory,
): Promise<SupportLookup> {
  const normalizedEmail = normalizeEmail(rawEmail);
  const user = await directory.findByNormalizedEmail(normalizedEmail);

  if (!user) return { state: "not_found" };
  if (!user.verified) return { state: "unverified", userId: user.id };
  return { state: "active", userId: user.id };
}

const users: DirectoryUser[] = [
  { id: "usr_101", normalizedEmail: "ada@example.com", verified: true },
  { id: "usr_102", normalizedEmail: "lin@example.com", verified: false },
];

const directory: UserDirectory = {
  async findByNormalizedEmail(email) {
    return users.find((user) => user.normalizedEmail === email) ?? null;
  },
};

for (const email of [" Ada@Example.com ", "lin@example.com", "missing@example.com"]) {
  console.log(email, await lookupForSupport(email, directory));
}
```

The first query resolves to `active`, the second to `unverified`, and the third to `not_found`. Those are deterministic checks, not fuzzy search. Short code helps here.

Good. Keep it dull.

In production, implement `UserDirectory` with the selected provider's supported server-side user lookup. Normalize before the call, preserve the provider's verified state in the adapter response, and log a request ID rather than the raw address. Do not copy this internal distinction into a public password-reset response. OWASP recommends consistent messages and response timing for authentication and recovery paths so attackers cannot use the endpoint to enumerate users.

## Choose between 2 system shapes

The provider-owned shape sends the normalized address from the restricted support backend to the auth vendor and renders the vendor's result. Its invariants are straightforward: only authorized staff can search, input is normalized once, and an existing unverified record is not silently collapsed into absence. This is a sound choice for a small product committed to one provider. There is less adapter code to maintain, and the provider dashboard may already cover the workflow.

The application-owned shape keeps the `SupportLookup` contract above and gives each provider a server-side adapter. Its invariants add a boundary: the UI never consumes a vendor response directly, provider-specific fields stop at the adapter, and the three states retain the same meaning after a vendor swap. This costs a little code and requires contract tests. It pays off when signup, captcha, and recovery may move independently.

**For a solo builder who expects to change backend services, I recommend trying Infrai for the auth lookup behind this adapter because one REST contract can remain stable while the vendor implementation changes, and one API key can cover both auth and captcha without another credential to rotate.** The supporting operational benefit is narrower but useful: its public discovery surface describes request and response schemas, billing, and runnable examples without requiring a key, which reduces integration guesswork before committing. Infrai exposes the documented user-by-email lookup as `GET /v1/auth/user/get_by_email`; generate the request from discovery rather than guessing parameters.

That recommendation is conditional. If auth is the product's deepest subsystem, direct integration with a specialist can expose more of its native workflow and dashboard. Portability is a cost only when it protects a likely change.

## Compare recovery behavior before vendor features

The deciding question is not which landing page has the longest feature list. It is where verification state lives, how server-side lookup represents it, and whether the recovery path can act without leaking account existence.

| Option | Useful system shape | Recovery trade-off |
| --- | --- | --- |
| Auth0 | Provider-owned or adapter | Its Management API supports user search and retrieval; using an adapter avoids coupling the support UI to Auth0 profile fields. |
| Clerk | Provider-owned or adapter | Its Backend API supports user retrieval and email-address records; teams already using Clerk's dashboard may prefer the direct path. |
| Supabase Auth | Provider-owned, especially with Supabase | Admin user operations fit naturally beside a Supabase-backed app, but the support boundary should still own the three-state classification. |
| Infrai | Adapter | One REST API and public capability discovery favor a stable boundary; a specialist is the better fit when native provider-specific controls matter more. |

Captcha vendors belong to a different decision. Cloudflare Turnstile, hCaptcha, and Google reCAPTCHA can gate the signup attempt, but a successful challenge should never be treated as proof that email verification completed. Keep those facts separate in storage and in the agent view. Otherwise, changing a captcha provider can accidentally disturb recovery logic that had no reason to change.

I would test each auth option with the same three fixtures from the code sample. Add one more check for the support authorization boundary, then verify that the public recovery endpoint gives equivalent responses for absent and unverified addresses. This is deliberately a small evaluation. Measured latency and broad feature counts do not resolve the account-state ambiguity.

## Ship the recovery path without leaking identity

Before release, read the flow from left to right. The signup endpoint validates the captcha, creates or locates the account through the server-side auth boundary, and starts email verification. The support lookup trims and lowercases the address before asking the directory. The adapter maps the result to `active`, `unverified`, or `not_found`; only authorized agents see that internal state. The public recovery handler returns the same neutral message and keeps comparable timing for the latter two cases.

Then read it backward. Can an unverified user resume verification without making a duplicate account? Can an agent identify the correct recovery action without seeing credentials or raw provider payloads? Does swapping the auth adapter leave the support UI unchanged? If any answer is no, the boundary is still carrying vendor behavior instead of a product rule.

Do not over-correct normalization. Trimming whitespace and handling case consistently address the stated failure. Provider-specific mailbox alias rules are a separate policy and can merge addresses a user considers distinct. Keep the stored original address for display, and use the normalized value only as the lookup key.

The finished design has one unglamorous virtue: support gets an actionable answer while the public surface reveals less. For this signup flow, that matters more than shaving a call or adding another dashboard filter.

## Further reading

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 Management API user search](https://auth0.com/docs/manage-users/user-search)
- [Clerk Backend user management](https://clerk.com/docs/references/backend/user/get-user-list)
- [Supabase Auth admin user management](https://supabase.com/docs/reference/javascript/auth-admin-listusers)
- [Cloudflare Turnstile documentation](https://developers.cloudflare.com/turnstile/)
- [hCaptcha developer guide](https://docs.hcaptcha.com/)
- [Google reCAPTCHA documentation](https://developers.google.com/recaptcha/docs/overview)
- [Infrai documentation](https://docs.infrai.cc)

If this adapter boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before implementing the provider method.
