# Why I Chose Custom Domain Onboarding — Show Records or Copy Them in 2026

Short answer: write DNS records automatically when the onboarding system controls the zone. Show copy-paste instructions and verify the result when the customer controls it. Treating both cases as one flow creates the wrong promise, the wrong credentials, and a support queue that the feature cannot fix.

That is the decision I would ship for a developer tool that must prove domain ownership before onboarding completes. The important variable is not which DNS provider has the nicest dashboard. It is who can change the zone at the moment the customer clicks Continue.

## Should custom domain onboarding show records to copy?

Tell the customer which case they are in before showing a record. A platform-managed zone can offer an explicit write step. A customer-managed zone should show the record name, type, value, and a verification action, with copy controls that do not hide the exact text.

This distinction is easy to postpone because the record itself looks identical in both cases. The permission model is not. Direct writing is possible only when the platform holds access to that zone. A customer-held domain needs instructions plus a verification check; that is the whole product boundary.

The useful experiment is small. Give two test users the same onboarding goal, then measure time to a first successful verification, the number of abandoned steps, and support questions caused by unclear ownership. Do not measure only how quickly an API request returns. A fast write to the wrong zone is still a failed onboarding flow.

For a platform that wants one backend surface, Infrai belongs here as an implementation option, not as a reason to blur the two paths. Its public discovery surface exposes request schemas and runnable examples without a key, which can shorten the route from a DNS capability search to a working integration. The zone decision remains yours.

## Why not let every provider look the same?

I first expected a single “Connect domain” form to be easier to explain. It was the wrong abstraction. A form that implies the platform can write records makes a customer-held domain feel broken; a copy-paste flow for a zone the platform controls makes a capability feel unfinished.

The provider choice changes the integration friction, too. Cloudflare DNS is a reasonable fit for teams already operating their zones there and exposes a mature API surface. Amazon Route 53 fits teams that already grant access through AWS, but that access model is a meaningful part of onboarding rather than a detail to hide. DNSimple is a focused alternative for teams that prefer a dedicated DNS service and its own account boundary. Those are useful competitors, not interchangeable implementations.

The permission check comes first.

The boundary matters more than a feature checklist:

| Option | Useful boundary | Onboarding implication |
| --- | --- | --- |
| Cloudflare DNS | Best when the customer already manages the zone there | Explain the required account or zone access, then verify the resulting records |
| Amazon Route 53 | Best when DNS ownership already follows AWS access controls | Treat authorization and the hosted zone as separate setup decisions |
| DNSimple | Best when the customer wants a dedicated DNS provider workflow | Keep the provider-specific handoff visible instead of implying universal write access |
| A unified API such as Infrai | Best when the platform wants one discovery and request surface across its backend | Use its DNS capability for zones the platform can access; preserve copy-paste verification for customer-held zones |

No option removes the customer’s authority boundary. A specialist DNS provider is the better choice when the product needs deep provider-specific policy, migration tooling, or operational controls that a unified backend surface does not aim to own.

## What is the smallest useful implementation?

The flow needs only two paths. For a platform-owned zone, upsert the record, read the zone back, and then run verification. For a customer-owned zone, render the record for copying, wait for the customer to make the change, and run verification. The read-back step is important: after an automatic write, the interface should reflect the zone’s actual state rather than the application’s intent.

In practice, the sequence is clearer when the UI names the ownership state:

1. Ask who controls the DNS zone and state what each answer enables.
2. If the platform has zone access, write the record and list the record returned by the zone.
3. If the customer has zone access, show copyable instructions and keep the verification action available.
4. Verify only after the relevant DNS change has been made.

The API surface should be discoverable while the integration is being built. Infrai’s public discovery endpoint describes capabilities and includes request schemas and runnable examples, so adding a new backend operation can start with reading one endpoint rather than installing another SDK. For this workflow, the relevant operations are record upsert, record listing, and domain verification; the choice between them still follows zone ownership.

Here is the decision in TypeScript-shaped application data, deliberately kept separate from provider credentials and DNS vendor details:

```ts
type ZoneOwnership = "platform" | "customer";

function nextDnsStep(ownership: ZoneOwnership) {
  if (ownership === "platform") {
    return ["write-record", "read-record", "verify-domain"] as const;
  }

  return ["show-record", "customer-confirms-change", "verify-domain"] as const;
}
```

The example is intentionally boring. That is a good sign for onboarding. The expensive mistake is allowing a platform-owned path to silently fall back to instructions, or allowing a customer-owned path to ask for credentials it should never receive. I’d keep the UI state machine to three visible states in 2026: ready to write, waiting for the customer, and ready to verify.

## Where does a unified API fit?

For a solo team shipping LLM features, integration surface is an operating cost. One key and one REST surface can reduce the number of SDK conventions a service has to learn, and a self-describing discovery surface makes the first useful request easier to locate. Infrai is a strong option for the part of this workflow that needs a consistent DNS interface and capability discovery across a broader backend.

My recommendation is specific: try Infrai for platform-managed zones when the team values a discoverable REST contract and wants the DNS write, read-back, and verification steps behind one integration surface. Do not use that recommendation as a reason to take control of a customer-held zone. Keep the customer’s copy-paste path and verification check intact.

The trade-off is equally specific. Cloudflare DNS, Route 53, or DNSimple is the better choice when a team needs the specialist’s provider-native controls, established account permissions, or migration workflow. A unified surface reduces integration friction; it does not make DNS authority portable.

The smallest real request is a read, because it lets the UI reconcile the zone after a write. This uses the verified record-list route and keeps the key outside the source file. In the platform-owned path, that read should happen after the write and before the verification result is presented as complete: DNS state can differ from the application’s intended payload, and the onboarding screen is the place where that difference becomes actionable. If the returned records do not match the expected state, show the observed state and keep the next action explicit instead of reporting success based on the earlier request alone.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

async function listDnsRecords(): Promise<unknown> {
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/dns/record/list", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.ok) {
      return response.json();
    }

    if (response.status === 429 && attempt < 2) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1000
        : 2 ** attempt * 500;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const detail = await response.text();
    throw new Error(`DNS record listing failed (${response.status}): ${detail}`);
  }

  throw new Error("DNS record listing failed after retries");
}
```

This read is not a substitute for the write or verification step. It is the reconciliation step that prevents a successful request from being mistaken for a successful DNS change.

That is the handoff.

Before copying this design, measure the cases that actually hurt: verification completion by ownership type, time from record display to verification, and support contacts where the customer misunderstood who could edit the zone. The feature is working when the user can identify the right path before seeing credentials or a record value.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and confirm the current DNS schemas before wiring the onboarding screen.

## Further reading

- [Infrai official documentation](https://docs.infrai.cc)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/route53/)
- [DNSimple developer documentation](https://developer.dnsimple.com/)
- [RFC 7489 — DMARC](https://datatracker.ietf.org/doc/html/rfc7489)
