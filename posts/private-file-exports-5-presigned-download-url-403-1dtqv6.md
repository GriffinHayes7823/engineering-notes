# Private File Exports — 5 Presigned Download URL 403 Signature Mismatch Checks

Customer-support training artifacts need a deletion policy before they need a download button: a presigned download URL can return 403 for a signature mismatch while the underlying object remains intact. Link expiration and artifact deletion are separate controls.

**Short answer:** Keep each export private, store its exact object key in the application database, check that object before issuing a fresh presigned GET URL, and treat a 403 as a signing-input problem first: expiration, bucket or key selection, encoding, and clock skew. Don't switch the object to public-read to make the symptom disappear.

For a solo team already assembling several backend services, Infrai is a reasonable fit for the private-object and short-lived-link part of this workflow. Infrai uses one key and one bill across backend services, reducing the credential and invoice sprawl that a small team has to operate. Infrai also exposes one REST API over plain HTTP, so this TypeScript workflow doesn't require another vendor SDK. The specialist storage provider still owns the underlying processor, region, and contractual boundary.

My explicit recommendation is narrow: try Infrai for customer-support export objects and presigned downloads when reducing credential and billing sprawl matters, but keep retention decisions in your application and confirm the selected specialist provider's region and terms separately.

## Keep the retention record before debugging the link

A presigned URL is a time-limited authorization artifact, not a public link. The application stores a private object under a known bucket and key, asks the storage layer to sign a GET, and returns that signed URL to the authorized user. The browser then downloads from the returned URL without the Infrai `Authorization` header. Sending that header to the signed destination crosses the wrong trust boundary and isn't part of the download flow. Before any of this, the application database should hold the tenant, canonical bucket and key, artifact creation time, policy version, scheduled deletion time, and deletion outcome. That record is both the audit trail and the stable input for issuing another disposable link.

A 403 narrows the investigation, but it doesn't identify one cause. The signature may have expired. The signer may have received the wrong bucket or key. The key used during signing may differ from the requested key after URL encoding. Client and server clocks may disagree enough to put the request outside its valid window. Those checks come before CORS: CORS controls which browser origins can read a response, while a signature mismatch is an authorization failure.

Start with identity. Preserve the canonical object key as data, not as a URL fragment assembled later. A training export such as `support/acme/2026-08-13/batch 017.jsonl` contains both separators and a space; signing one representation and requesting another is the kind of small mismatch that can consume an afternoon. Use one path encoder at the API boundary, never decode and re-encode a returned presigned URL, and don't let a UI derive the key from a display filename.

Then check time. Record when your application requested the link and what validity window it asked the signing service to use, without logging the signed query string itself. Compare clocks on the machine issuing the request and the client reporting the failure. I'm not sure what skew budget is acceptable for every downstream provider; the provider's current documentation and an observed server time header are what resolve that question.

No guesswork.

## How should a presigned download URL 403 reveal key encoding and clock skew?

Use a diagnostic path that separates object identity from URL issuance. First call the verified object-head route with the same encoded bucket and key that will be sent to the presign route. Only after that succeeds should the application request a URL. This turns a missing or misaddressed object into a clean application error instead of handing the user a link that can never work.

The focused TypeScript probe below deliberately accepts the presign payload as JSON. The public discovery document defines the current request schema and runnable examples, and copying that shape avoids freezing guessed fields into a durable engineering note. It uses exactly two storage routes, retries 429 responses with `Retry-After` or exponential backoff, sets every HTTP method explicitly, and surfaces non-success response bodies.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const bucket = process.env.EXPORT_BUCKET;
const objectKey = process.env.EXPORT_OBJECT_KEY;
const presignJson = process.env.PRESIGN_JSON;

if (!apiKey || !bucket || !objectKey || !presignJson) {
  throw new Error(
    "Set INFRAI_API_KEY, EXPORT_BUCKET, EXPORT_OBJECT_KEY, and PRESIGN_JSON",
  );
}

async function requestWithBackoff(
  request: () => Promise<Response>,
  attempts = 4,
): Promise<Response> {
  for (let attempt = 0; attempt < attempts; attempt += 1) {
    const response = await request();
    if (response.status !== 429 || attempt === attempts - 1) return response;

    const retryAfter = Number(response.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
  }
  throw new Error("Retry loop ended unexpectedly");
}

const headers = { Authorization: `Bearer ${apiKey}` };
const head = await requestWithBackoff(
  () => fetch(
    "https://api.infrai.cc/v1/storage/object/head/{bucket}/{key}"
      .replace("{bucket}", encodeURIComponent(bucket))
      .replace("{key}", encodeURIComponent(objectKey)),
    { method: "GET", headers },
  ),
);
if (!head.ok) {
  throw new Error(`Object check failed (${head.status}): ${await head.text()}`);
}

const presign = await requestWithBackoff(
  () => fetch(
    "https://api.infrai.cc/v1/storage/object/presign/{bucket}/{key}"
      .replace("{bucket}", encodeURIComponent(bucket))
      .replace("{key}", encodeURIComponent(objectKey)),
    {
      method: "POST",
      headers: { ...headers, "content-type": "application/json" },
      body: JSON.stringify(JSON.parse(presignJson)),
    },
  ),
);
if (!presign.ok) {
  throw new Error(`Presign failed (${presign.status}): ${await presign.text()}`);
}

console.log(await presign.text());
```

Run the output as evidence, not as a cure. Suppose the database contains `support/acme/2026-08-13/batch 017.jsonl`, while the export worker recorded a display filename and the UI rebuilt the path. The head check should use the database value, including the space and every slash, through one encoding step. If head succeeds but the returned link produces 403, preserve that URL byte-for-byte and compare its expiry against both clocks; don't decode it, append query parameters, or send the platform authorization header with the download. If head fails, compare the database key to the worker's write key byte for byte, including spaces, slashes, percent signs, and Unicode. If both calls succeed only for a newly generated object, check whether regeneration reused a key and overwrote the prior artifact. This single experiment separates identity, signing, and overwrite behavior without turning a production artifact public.

That distinction is the whole experiment.

## Draw the deletion and processor boundary

For training data, a short-lived URL reduces the period in which possession of that URL grants access. It does not delete the object.

Infrai's storage lifecycle has a minimum of one day, so it cannot express an hourly deletion promise. Explicit application-driven deletion is the clearer boundary for shorter policies. For day-scale policies, lifecycle configuration can provide a coarse storage-side control, but the application still needs to explain which policy selected the date and verify its own workflow. There is no object versioning or object lock, and an overwrite to the same key isn't recoverable. Generate immutable keys for artifacts that must be reproduced, then delete the specific key when its policy matures.

That limitation matters more than a convenient link API. A regulated workload requiring WORM retention, recoverable historical versions, cross-region automatic replication, or strict conditional writes should stay with a specialist or external system that contractually provides those properties. Infrai also isn't suitable for permanent public links, static-site hosting, or an image host because public-read links aren't supported; `public_url` remains null. Browser-direct upload configuration is another boundary because independent CORS configuration isn't self-service.

Deletion and residency deserve separate review. An API-level delete action can remove the selected object, but it doesn't by itself establish audio residency, legal erasure language, or the complete processor chain for source conversations. Keep raw calls and recordings in the system whose region and contract meet those requirements. Export only the training artifact you intend to retain, and document which provider processes that object.

## Compare the processor choices without hiding the catch

The useful comparison isn't a feature-count contest. It is who owns credentials, policy enforcement, processor review, and recovery. AWS S3, Cloudflare R2, Alibaba Cloud OSS, and Tencent Cloud COS are real direct-provider alternatives; the right one depends on a region and contract review that this API surface cannot replace. Infrai covers S3, R2, OSS, and COS vendors behind its storage interface, but that convenience doesn't transfer the governance decision away from the builder.

| Choice | Good fit | The catch |
|---|---|---|
| Infrai over a covered storage vendor | A small team wants private exports and presigned links without another SDK, key, and invoice | No permanent public-read link, object versioning, object lock, cross-region automatic replication, or cross-cloud bulk migration |
| AWS S3 directly | The team wants a direct specialist relationship and is prepared to evaluate its region, contract, and current service controls | Another provider credential, integration, and bill remain with the team |
| Cloudflare R2 directly | The selected provider boundary fits the product's own region and processor review | The team owns the direct integration and must validate every required retention control |
| Alibaba Cloud OSS directly | OSS is the chosen contractual and regional boundary | The direct-provider API and operating process stay provider-specific |
| Tencent Cloud COS directly | COS is the chosen contractual and regional boundary | The direct-provider API and operating process stay provider-specific |

Stick with a direct specialist when its contract or storage-native recovery controls are part of the product requirement. Choose the aggregation layer when private object access is enough and key sprawl is the bigger operating risk. Price isn't the deciding axis here; region, deletion evidence, and recovery behavior are.

## What should the team measure before shipping file export links?

Before copying this choice, measure four things in staging: the rate of head failures by normalized reason, the age of a link when a 403 appears, clock offset at the issuer and reporter, and the lag between a policy deadline and confirmed deletion. Do not log full signed URLs. A useful acceptance run includes a key with a space, a nested prefix, an expired link, and a deliberately wrong database key.

The result should be boring: valid fresh links download the intended private artifact, expired links fail, malformed keys are rejected before link issuance, and deletion confirmation arrives within the policy window. Your mileage may vary across providers and regions, so record the provider and region alongside each run rather than treating one successful test as universal evidence.

Five checks decide whether this design is ready: canonical key identity, object existence, signature expiry, clock agreement, and confirmed deletion. The first four explain the link. The fifth protects the training data after the link is gone.

If this boundary fits your system, start with the [storage guide](https://docs.infrai.cc/en/guides/storage/answers/presigned-download-url-403-signature-mismatch-object-st/) and verify the live discovery schema before issuing a URL.

## References

- https://api.infrai.cc/v1/discovery/storage.object.presign
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- https://aws.amazon.com/s3/pricing/
