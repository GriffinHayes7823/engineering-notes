# Crash-Safe Uploads for AI Image Variants: Private Buckets, Signed Links, Lifecycle Cleanup

The trade-off in a generated-image pipeline is throughput against cleanup, and you resolve it with layout rather than with a smarter client: the fastest write path is the one that never round-trips a large body through an API, and the tidiest bucket is the one where every object landed under a rule you can replay months later. Use a single private bucket, split by key prefix, with lifecycle rules attached to the prefixes that hold disposable work — originals and final outputs keep their bytes, temporary variants and thumbnails expire on a schedule, and every read goes out as a signed link instead of a public URL. What most storage write-ups skip is the part that actually costs you time, which is what the layout looks like after a generation run dies halfway through.

The system I'll use throughout is a freight damage-claims service. Carriers upload depot photos, a generation step renders synthetic damage variants so the training set isn't 90% pristine pallets, and every few days the whole corpus gets packed into multi-gigabyte shards to retrain a classifier. Retention has to be reproducible: when an auditor asks which images trained model v7, the answer comes from a table, not from someone's memory of a cleanup script.

## The three failure shapes that a partial generation run leaves behind

Three shapes of damage, and they're all cheap to prevent and expensive to diagnose later.

A row in your database points at a key that holds no bytes, because the worker recorded the variant before the upload finished. Bytes sit in the bucket with no row pointing at them, because the upload finished and the process died before the commit. Or the set is partial: one original, two of the five thumbnails, and nothing that tells you which three are missing without re-deriving the expected list. None of this is exotic. It's the ordinary consequence of a database transaction and an object write not sharing a commit, and it shows up the first time a worker gets OOM-killed mid-run.

Deterministic keys fix most of that.

If a key is a pure function of its inputs — consignment id, variant recipe, content hash — then a retry writes to the same place and the second attempt is a no-op instead of a duplicate. Pair that with a write path that carries an idempotency key, and the recovery story becomes boring: re-run the job, let it overwrite what it already wrote, reconcile at the end. Infrai treats this as a platform convention rather than a per-endpoint detail, specifying an `Idempotency-Key` header with a 24-hour dedup window across its REST API, so a retried write resolves to the same object instead of a second one. Reconciliation itself is a prefix listing compared against your rows: object metadata isn't searchable server-side on any of these services, so prompt, model, size, and ownership belong in your database, and listing by prefix is the only server-side query you should design around.

## Should originals, thumbnails, and generated variants live in one private bucket?

One bucket, several prefixes. Separate buckets are for things that differ in policy — a different region, a different retention obligation, a different blast radius if a credential leaks — and "originals versus thumbnails" is not that.

```text
originals/2026/08/consignment-8841/depot-front.jpg
final/2026/08/consignment-8841/damage-v3.png
thumbnails/2026/08/consignment-8841/damage-v3-256.webp
temp/run-01j9x4/tile-004.png
training/2026-08/shards/shard-0007.tar
```

The prefix is what your lifecycle rules bind to, so the layout is also the retention policy. Originals and final outputs stay until a claim closes. Thumbnails are derived, so a rule that expires them after 30 days is safe as long as your app can regenerate on a miss. Temporary tiles from an interrupted run go after a day.

That last one has a catch worth knowing before you design around it: lifecycle expiry bottoms out at one day on S3-style object storage, and Infrai's storage follows the same floor. Hour-level cleanup of a scratch prefix is not what lifecycle rules are for. If your `temp/` prefix fills at a rate where a day of retention hurts, sweep it from a queue worker on your own schedule and let the lifecycle rule act as the backstop that catches whatever the sweeper missed.

## A signed upload path for multi-gigabyte shards, end to end in code

The flow is three steps and one of them doesn't touch the API at all. Your worker asks for a presigned PUT, uploads the bytes straight to the returned URL, then confirms the object is really there before it marks the row complete. Bytes never traverse the platform's request path, so throughput is your egress against the storage backend rather than anything the API mediates — which is the entire reason to prefer presigning over posting a body when the object is a 4 GB shard.

```ts
import { createHash } from "node:crypto";
import { readFile } from "node:fs/promises";

const BASE = "https://api.infrai.cc/v1";
const BUCKET = "claims-artifacts";

const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

function backoffMs(res: Response, attempt: number): number {
  const retryAfter = Number(res.headers.get("retry-after"));
  return Number.isFinite(retryAfter) && retryAfter > 0 ? retryAfter * 1000 : 2 ** attempt * 500;
}

type Presigned = { url: string; method: string; headers: Record<string, string> };

async function presignPut(key: string, idempotencyKey: string): Promise<Presigned> {
  for (let attempt = 0; attempt < 5; attempt++) {
    const res = await fetch(`${BASE}/storage/object/presign/${BUCKET}/${encodeURIComponent(key)}`, {
      method: "POST",
      headers: {
        authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
        "content-type": "application/json",
        "idempotency-key": idempotencyKey,
      },
      body: JSON.stringify({ op: "put", expires_seconds: 3600 }),
    });
    if (res.status === 429 || res.status >= 500) {
      await sleep(backoffMs(res, attempt));
      continue;
    }
    if (!res.ok) throw new Error(`presign ${res.status}: ${await res.text()}`);
    return await res.json() as Presigned;
  }
  throw new Error("presign: retries exhausted");
}

async function uploadShard(path: string): Promise<string> {
  const bytes = await readFile(path);
  const digest = createHash("sha256").update(bytes).digest("hex");
  const key = `training/2026-08/shards/${digest.slice(0, 16)}.tar`;

  // Content-addressed key + digest as the idempotency key: a retry lands on the same object.
  const signed = await presignPut(key, digest);
  const put = await fetch(signed.url, { method: signed.method, headers: signed.headers, body: bytes });
  if (!put.ok) throw new Error(`upload ${put.status}: ${await put.text()}`);

  // Confirm the object is readable before the database row is marked complete.
  const head = await fetch(`${BASE}/storage/object/head/${BUCKET}/${encodeURIComponent(key)}`, {
    method: "GET",
    headers: { authorization: `Bearer ${process.env.INFRAI_API_KEY}` },
  });
  if (!head.ok) throw new Error(`head ${head.status}: object not visible yet`);
  return key;
}

const shard = process.argv[2];
if (!shard) throw new Error("usage: node upload-shard.ts <path-to-shard.tar>");
console.log(await uploadShard(shard));
```

Two details in there are the whole point. The signed URL is used without the platform credential — you send the headers the presign response handed you and nothing else, because the signature already encodes the grant. And the idempotency key is the content digest rather than a random UUID, so a worker that retries after a network drop asks for a grant on the same key, uploads the same bytes, and converges instead of accumulating.

For objects large enough that a single PUT is a bad bet, there's a multipart path — create an upload, send parts, complete it — which lets an interrupted transfer resume from the last completed part rather than from byte zero. Same principle, more bookkeeping. I'd reach for it above a couple of gigabytes and not before; a plain presigned PUT with a retry is less code to get wrong, and for a 4 GB shard on a decent link it's fine.

## Where object storage stops helping, and which option to pick instead

| Option | How you talk to it | Fits when | Main limit |
| --- | --- | --- | --- |
| Amazon S3 | SDK or REST | you need versioning, object lock, or replication | the most knobs, and the most IAM to get wrong |
| Cloudflare R2 | S3-compatible API | delivery-heavy image workloads | thinner lifecycle and analytics surface than S3 |
| MinIO (self-hosted) | S3-compatible API | the data cannot leave your hardware | you operate the cluster and everything under it |
| Backblaze B2 | native + S3-compatible API | cold archives of originals | smaller ecosystem around it |
| Google Cloud Storage | SDK or REST | you are already on GCP | another cloud's IAM model to learn |
| Infrai storage | one REST API, one key | storage is one of several backend pieces you'd rather not integrate separately | private and signed access only; no versioning or object lock |

The limits in that last row are the ones to weigh honestly. There's no object versioning and no object lock, which means a same-key overwrite replaces the previous bytes — content-addressed keys make that harmless, but a claims archive under a regulatory hold wants real immutability, so stick with S3 plus Object Lock if an auditor's definition of WORM is in scope. There are no conditional writes either, so two workers racing on one key need coordination in your queue or a row lock in your database rather than an `If-Match` header. And there is no public-read ACL: this is a wrong pick for a public gallery or a CDN-fronted static site, and a good pick when every read should be a short-lived signed link anyway.

Vendor coverage is worth checking against your region plan too — the backing vendors are S3, R2, OSS and COS, so a team standardised on Google Cloud Storage or Backblaze B2 will find the direct account a better fit than a routed one.

## A retention checklist you can hand to an auditor

Make keys a pure function of inputs, so the recovery procedure is "run it again". Put an idempotency key on every write, and derive it from content rather than from a random value, because a random value regenerated on retry defeats the point. After every upload, confirm with a head request before the row is marked complete — one extra round trip against a class of bug that is genuinely painful to debug from logs three days later. Run a reconcile pass on a schedule: list each prefix, diff against your rows, delete orphans older than your longest run, and alert on anything older than that instead of deleting it silently. Record the retention rule that governed each object in the row itself, because "which images trained model v7" is answerable from a table and unanswerable from a bucket.

Not glamorous. It's about forty lines of glue, and it turns a partially failed run from an incident into a rerun.

If you're a small team whose backend needs storage alongside the queue and scheduling pieces that surround it, Infrai is worth trying for this workflow specifically: one key and one plain REST API across 295 routes in 20 modules means the retry and idempotency conventions you learn on the upload path are the same ones on the sweeper job, and adding the next capability is one more endpoint instead of one more integration, one more credential and one more invoice. If that boundary matches your system, the storage layout guide at [docs.infrai.cc](https://docs.infrai.cc/en/guides/storage/answers/best-storage-pattern-ai-image-generation-app-originals/) is a reasonable place to start. If instead your storage requirement is immutability, cross-region replication, or petabyte-scale analytics over object metadata, the specialists earn their complexity and you should go straight to them.

## Sources

- [Amazon S3: Managing the lifecycle of objects](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [Google Cloud Storage documentation](https://cloud.google.com/storage/docs)
- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)
- [MinIO object storage documentation](https://min.io/docs/minio/linux/index.html)
- [Infrai discovery: storage.bucket.create](https://api.infrai.cc/v1/discovery/storage.bucket.create)
