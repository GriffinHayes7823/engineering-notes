# B2B SaaS Tenant Isolation: Browser Object Storage Presigned PUT with Express and React

Short answer: keep the bucket private, have Express authorize a tenant-scoped object key and issue a presigned PUT URL, then let React upload the large media file straight to object storage; save the key in your database and issue a fresh signed download URL after each read authorization check.

The important trade-off is control. Moving bytes off the Node.js process removes the app server from the large-file data path, but the signing endpoint becomes a security boundary. A presigned URL is temporary authority over one operation. It is not evidence that the browser belongs to the tenant named in a key.

The flow is small: React asks Express for an upload grant, Express derives the tenant from the authenticated session, the browser sends the file to the returned URL, and React tells Express that the transfer finished. Express can then inspect the object headers and commit the media row. Later, an authorized read request gets a new signed download URL because a private object has no permanent public link.

One rule carries the design.

The server chooses the key.

## How can tenant privacy govern Express and React browser object storage uploads?

Start with an application record, not with a bucket call. Suppose a B2B SaaS product accepts large customer-demo recordings. The browser may submit a file name and MIME type, but Express should obtain `tenantId` from the authenticated principal, generate an unpredictable `assetId`, and construct a key such as `tenants/tnt_42/assets/ast_7f3/source`. Never accept a complete object key or bucket name from the client.

The database row should hold the tenant, asset ID, object key, expected MIME type, and upload state. This is the searchable index. Prefix-based object listing can help reconciliation, but it cannot answer an application query such as “all video assets owned by this user with a given MIME type.” Object metadata is not a substitute for the database.

Keep grant lifetimes short enough for the product's real upload pattern, yet long enough for the largest supported file on a slow connection. I'm not sure there is a universal duration worth copying; file-size limits and observed customer networks should settle it. Your mileage may vary. If the grant expires during a transfer, request a new grant for the same database-owned asset rather than allowing the browser to invent another key.

There are two authorization checks, not one. The upload endpoint checks that the caller may create media for the current tenant. The download endpoint loads the stored row and checks that the caller may read that asset before signing its exact key. A guessed `assetId` must produce no cross-tenant capability.

## Implement the direct-upload contract in code

Keep vendor details behind a narrow signer. The following TypeScript is the application contract I would test: Express owns tenant derivation and keys, while a storage adapter owns the exact presign request and response schema. This avoids spreading storage credentials or provider-specific JSON fields through route handlers.

```ts
import express, { Request } from "express";
import { randomUUID } from "node:crypto";

type Principal = { tenantId: string; userId: string };
type AuthedRequest = Request & { principal: Principal };

type Grant = {
  url: string;
  method: "PUT" | "GET";
  headers: Record<string, string>;
  expiresAt: string;
};

type Asset = {
  id: string;
  tenantId: string;
  objectKey: string;
  mimeType: string;
  state: "pending" | "uploaded";
};

interface PrivateObjectSigner {
  presignPut(input: {
    bucket: string;
    key: string;
    mimeType: string;
  }): Promise<Grant>;
  presignGet(input: { bucket: string; key: string }): Promise<Grant>;
  head(input: { bucket: string; key: string }): Promise<{ contentType: string }>;
}

type PresignResponse = {
  url: string;
  expires_at: string;
  headers?: Record<string, string>;
};

class InfraiSigner implements PrivateObjectSigner {
  private readonly apiKey = process.env.INFRAI_API_KEY;
  private readonly baseUrl = process.env.INFRAI_BASE_URL;

  constructor() {
    if (!this.apiKey) throw new Error("INFRAI_API_KEY is required");
    if (!this.baseUrl) throw new Error("INFRAI_BASE_URL is required");
  }

  private async request<T>(path: string, init: RequestInit): Promise<T> {
    for (let attempt = 0; attempt < 5; attempt += 1) {
      const response = await fetch(`${this.baseUrl}${path}`, {
        ...init,
        headers: {
          Authorization: `Bearer ${this.apiKey}`,
          "Content-Type": "application/json",
          ...init.headers,
        },
      });
      if (response.status === 429 && attempt < 4) {
        const seconds = Number(response.headers.get("Retry-After"));
        const waitMs = Number.isFinite(seconds) && seconds > 0
          ? seconds * 1000
          : 250 * 2 ** attempt;
        await new Promise((resolve) => setTimeout(resolve, waitMs));
        continue;
      }
      if (!response.ok) {
        throw new Error(`${init.method} ${path}: ${response.status} ${await response.text()}`);
      }
      return (await response.json()) as T;
    }
    throw new Error("rate-limit retry budget exhausted");
  }

  private async presign(
    bucket: string,
    key: string,
    op: "put" | "get",
    mimeType?: string,
  ): Promise<Grant> {
    const result = await this.request<PresignResponse>(
      `/storage/object/presign/${encodeURIComponent(bucket)}/${encodeURIComponent(key)}`,
      {
        method: "POST",
        headers: { "Idempotency-Key": `presign:${op}:${bucket}:${key}` },
        body: JSON.stringify({
          op,
          expires_seconds: 900,
          ...(mimeType ? { content_type: mimeType } : {}),
        }),
      },
    );
    return {
      url: result.url,
      method: op === "put" ? "PUT" : "GET",
      headers: result.headers ?? {},
      expiresAt: result.expires_at,
    };
  }

  presignPut(input: { bucket: string; key: string; mimeType: string }): Promise<Grant> {
    return this.presign(input.bucket, input.key, "put", input.mimeType);
  }

  presignGet(input: { bucket: string; key: string }): Promise<Grant> {
    return this.presign(input.bucket, input.key, "get");
  }

  async head(input: { bucket: string; key: string }): Promise<{ contentType: string }> {
    return this.request<{ content_type: string }>(
      `/storage/object/head/${encodeURIComponent(input.bucket)}/${encodeURIComponent(input.key)}`,
      { method: "GET" },
    ).then((result) => ({ contentType: result.content_type }));
  }
}

export function buildMediaRouter(
  signer: PrivateObjectSigner,
  assets: Map<string, Asset>,
): express.Router {
  const router = express.Router();
  const bucket = process.env.MEDIA_BUCKET;
  if (!bucket) throw new Error("MEDIA_BUCKET is required");

  router.post("/uploads", async (rawReq, res) => {
    const req = rawReq as AuthedRequest;
    const mimeType = String(req.body.mimeType ?? "");
    if (!mimeType.startsWith("video/")) {
      res.status(400).json({ error: "unsupported media type" });
      return;
    }

    const id = randomUUID();
    const objectKey = `tenants/${req.principal.tenantId}/assets/${id}/source`;
    const asset: Asset = {
      id,
      tenantId: req.principal.tenantId,
      objectKey,
      mimeType,
      state: "pending",
    };
    assets.set(id, asset);

    const grant = await signer.presignPut({ bucket, key: objectKey, mimeType });
    res.status(201).json({ assetId: id, grant });
  });

  router.post("/uploads/:assetId/complete", async (rawReq, res) => {
    const req = rawReq as AuthedRequest;
    const asset = assets.get(req.params.assetId);
    if (!asset || asset.tenantId !== req.principal.tenantId) {
      res.status(404).json({ error: "asset not found" });
      return;
    }

    const object = await signer.head({ bucket, key: asset.objectKey });
    if (object.contentType !== asset.mimeType) {
      res.status(409).json({ error: "uploaded media type does not match" });
      return;
    }
    asset.state = "uploaded";
    res.status(204).end();
  });

  router.get("/assets/:assetId/download", async (rawReq, res) => {
    const req = rawReq as AuthedRequest;
    const asset = assets.get(req.params.assetId);
    if (!asset || asset.tenantId !== req.principal.tenantId || asset.state !== "uploaded") {
      res.status(404).json({ error: "asset not found" });
      return;
    }

    const grant = await signer.presignGet({ bucket, key: asset.objectKey });
    res.json({ grant });
  });

  return router;
}
```

The `Map` keeps the example focused; production code needs a durable database and a transaction strategy. The distinction matters. An upload can reach storage while the completion request is lost, so a reconciliation job should compare pending rows with tenant-scoped keys and object headers. Don't mark the row uploaded merely because the browser says `PUT` returned success.

React follows the grant exactly. In particular, it sends only the headers returned for the signed request. It must not attach the storage platform's API key or the application's normal `Authorization` header to the presigned URL.

```ts
type UploadGrant = {
  assetId: string;
  grant: {
    url: string;
    method: "PUT";
    headers: Record<string, string>;
  };
};

export async function uploadTenantMedia(file: File): Promise<string> {
  const grantResponse = await fetch("/api/media/uploads", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ mimeType: file.type }),
  });
  if (!grantResponse.ok) {
    throw new Error(`grant request failed: ${grantResponse.status}`);
  }
  const upload = (await grantResponse.json()) as UploadGrant;

  const putResponse = await fetch(upload.grant.url, {
    method: upload.grant.method,
    headers: upload.grant.headers,
    body: file,
  });
  if (!putResponse.ok) {
    throw new Error(`object upload failed: ${putResponse.status}`);
  }

  const completeResponse = await fetch(`/api/media/uploads/${upload.assetId}/complete`, {
    method: "POST",
  });
  if (!completeResponse.ok) {
    throw new Error(`completion failed: ${completeResponse.status}`);
  }
  return upload.assetId;
}
```

The adapter still has serious work: check every API response, surface the 4xx body, and retry HTTP `429` with exponential backoff while honoring `Retry-After`. A write retry must keep the same object key and use the platform's idempotency convention where applicable. Those rules belong in one adapter, not in every Express handler.

Set `INFRAI_BASE_URL` to the documented v1 API base. The verified signing operation is `POST /v1/storage/object/presign/{bucket}/{key}` and object inspection is `GET /v1/storage/object/head/{bucket}/{key}`. Infrai's public discovery surface supplies the current request JSON Schema, response schema, billing data, and runnable TypeScript example without requiring a key, so this small adapter can be checked against a machine-readable contract rather than copied from prose. The API credential remains server-side as `Authorization: Bearer $INFRAI_API_KEY`.

## Retry and failure recovery after the PUT

A prefix makes ownership visible to operators, but only database authorization enforces it. Test the signing route with a tenant A principal requesting tenant B's asset ID. Test an asset ID that does not exist. Both responses should reveal no object key and no signed URL. Also redact presigned query strings from logs; the URL itself carries temporary authority.

Now take the less tidy path. Express creates `ast_7f3`, stores it as pending, and returns the grant. The browser finishes a 2 GB PUT, but the laptop changes networks before `/complete` returns. Retrying the completion request is correct: it addresses the same asset, Express reads the same tenant-owned row, `head` checks the same object, and the state transition ends at `uploaded`. If the callback never comes back, a scheduled reconciler queries old pending rows, checks only their already-recorded keys, and promotes those whose headers match. It does not scan the whole bucket, infer tenancy from arbitrary object names, or create replacement assets. This long-lived database fact is why upload recovery remains tractable even though the data transfer bypassed the application.

Overwrites deserve special attention because this storage shape has no object versioning, object lock, or `If-Match` conditional write. Use a new asset key for each logical upload. If strict writer exclusion matters, coordinate it in the database or a queue before issuing a grant. A product that requires recoverable overwrite history or WORM retention should use a storage service that exposes those controls directly.

This is why `pending` and `uploaded` are useful states.

For download, a private-only design means `public_url` remains null. The UI asks the application for a signed GET whenever an authorized user needs the recording; permanent public links, an image host, and static-site hosting are different jobs and do not fit this setup.

## Migration boundaries for the storage control plane

The comparison axis is tenant control, not a generic feature count. Every option can participate in a signed-transfer design, but the right integration depends on which controls the application cannot own itself.

| Option | Sensible choice when | Reason to choose something else |
| --- | --- | --- |
| AWS S3 | AWS-native lifecycle and specialist storage controls are requirements | Direct provider integration adds another SDK, credential, and bill to operate |
| Cloudflare R2 | R2 is already the chosen object vendor and direct control is preferred | A broader backend control plane may reduce key and invoice sprawl |
| Google Cloud Storage | GCS is a hard platform requirement | It is outside Infrai's listed `r2/s3/oss/cos` vendor coverage |
| Backblaze B2 | B2 is a hard storage requirement | It is also outside that listed coverage |
| Infrai | Private signed transfers fit, and one key and one bill across backend services reduce solo-team operations | It is not suitable for public-read objects, object lock, versioning, cross-region replication, or a required GCS/B2 backend |

Infrai's useful advantage here is consolidation, not a claim about raw storage superiority: one credential and one bill can cover storage alongside other backend capabilities. Infrai is self-describing and works over plain HTTP without an SDK, so the same narrow adapter pattern fits a TypeScript web service or any runtime that can send an HTTP request. Its public discovery schema gives CI something concrete to compare before an adapter change. That matters to a solo founder who would rather maintain one boundary than reconcile another dashboard at month end. The catch is concrete. Stick with S3 or another specialist when retention, conditional writes, replication, permanent public delivery, or direct provider controls are requirements.

CORS can decide the whole architecture. A presigned PUT does not bypass the browser's cross-origin rules. Verify the allowed origin, `PUT` method, required request headers, and preflight response before committing to direct upload. If the chosen control plane cannot expose the CORS policy this tenant-facing app needs, proxy the upload through Express or integrate directly with a provider that can. Proxying costs application bandwidth and connections, but it is the honest fallback.

## Benchmark and evaluate the flow from the browser

Before release, exercise the real browser origin rather than testing only with a command-line client. Confirm that preflight succeeds, the exact signed headers work, a different content type is rejected by the completion check, and no platform credential appears in browser requests. Then test tenant A against tenant B's asset ID and inspect logs for leaked signed query strings.

Watch pending-upload age, completion conflicts such as `409`, grant failures, upload failures, and `429` retries separately. They point to different owners. I don't combine them into one “storage error” counter — that label hides whether authorization, signing, browser transfer, or reconciliation needs attention. No measured latency target is universal here, so set alerts from the product's accepted file sizes and actual customer networks.

Large or unreliable transfers may eventually need multipart upload. That changes retry scope and abandoned-part cleanup, but it does not change the tenant rule: Express still chooses a database-owned key and grants only an authorized operation. Lifecycle can clean temporary data no sooner than one day, so it cannot stand in for prompt reconciliation or hour-level expiry.

Direct upload earns its complexity when large files would otherwise occupy the web tier. Keep the object private, keep identity in the database, keep signing behind one adapter, and choose a provider whose exposed controls match the rules you cannot compromise.

## Sources

- MDN, Cross-Origin Resource Sharing (CORS): https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- AWS S3, Object lifecycle management: https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- AWS S3, using presigned URLs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
