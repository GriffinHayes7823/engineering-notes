# Product-Image Export: Large ZIP Multipart Upload and Signed Download Links

Short answer: build the ZIP in a worker, write it as a multipart object, and issue a short-lived signed download link only after the storage commit succeeds. For an e-commerce catalog, the harder problem is tenant isolation and cleanup, not the ZIP filename.

I would evaluate this with large-file throughput as the primary axis. The baseline is one long upload from a worker. The candidate is a bounded multipart pipeline with durable part receipts. Before shipping, measure archive generation, upload throughput, retransmitted bytes, and time from final part to a usable link. A faster upload that leaves ambiguous ownership is not a successful export.

## How can a Node.js worker make a large ZIP export safe for tenant-scoped downloads?

Give every export an immutable object key before the worker starts, for example `exports/tenant-42/export-0187/products.zip`. Never use `exports/tenant-42/latest.zip` as the coordination mechanism. The database record owns the tenant ID, export ID, object key, state, and upload session ID; object storage owns bytes, not authorization decisions.

The state transition should be boring: `queued` -> `uploading` -> `committed` -> `ready`. A worker may retry a part, but it must not publish a signed GET URL while the upload is still assembling. The API checks that the requester belongs to the tenant, then signs the exact key with an expiry appropriate to the download. The URL is a capability, so treat it like one.

That separation also handles duplicate clicks. Two requests can create two export records without two workers racing over one mutable key. If the product wants deduplication, compare a canonical export specification in the database and reuse a record that is already `ready`; do not infer identity from a storage listing.

One key. One owner.

## A small worker contract for the multipart handoff

The adapter below keeps provider-specific request details out of the worker. It assumes the adapter implements the storage service's documented multipart operations and returns the provider's part receipt. The worker controls ordering, state, and publication.

```ts
type PartReceipt = { partNumber: number; etag: string };

type ObjectStore = {
  startMultipart(input: { bucket: string; key: string }): Promise<{ uploadId: string }>;
  putPart(input: {
    uploadId: string;
    partNumber: number;
    body: Uint8Array;
  }): Promise<PartReceipt>;
  finishMultipart(input: {
    uploadId: string;
    parts: PartReceipt[];
  }): Promise<void>;
  abortMultipart(uploadId: string): Promise<void>;
  signDownload(input: {
    bucket: string;
    key: string;
    expiresInSeconds: number;
  }): Promise<string>;
};

type ExportRow = {
  id: string;
  tenantId: string;
  bucket: string;
  key: string;
  uploadId?: string;
  status: "queued" | "uploading" | "committed" | "ready" | "failed";
};

type ExportStore = {
  save(row: ExportRow): Promise<void>;
};

export async function publishProductImages(
  objects: ObjectStore,
  rows: ExportStore,
  row: ExportRow,
  chunks: AsyncIterable<Uint8Array>,
): Promise<string> {
  const session = await objects.startMultipart({
    bucket: row.bucket,
    key: row.key,
  });
  row.uploadId = session.uploadId;
  row.status = "uploading";
  await rows.save(row);

  const parts: PartReceipt[] = [];
  try {
    let partNumber = 1;
    for await (const body of chunks) {
      parts.push(await objects.putPart({
        uploadId: session.uploadId,
        partNumber,
        body,
      }));
      partNumber += 1;
    }
    await objects.finishMultipart({ uploadId: session.uploadId, parts });
    row.status = "committed";
    await rows.save(row);
  } catch (error) {
    await objects.abortMultipart(session.uploadId);
    row.status = "failed";
    await rows.save(row);
    throw error;
  }

  row.status = "ready";
  await rows.save(row);
  return objects.signDownload({
    bucket: row.bucket,
    key: row.key,
    expiresInSeconds: 900,
  });
}
```

The chunk producer needs backpressure. It should not read every product image into memory before the first part is sent. Bound the number of in-flight parts, record each receipt durably, and make retries distinguish a missing receipt from a completed part. A worker restart should be able to resume from the stored session and receipts, or intentionally abort and restart; silently creating a second archive is how throughput turns into storage waste. In a tenant-scoped catalog, the failure sequence is easy to miss: the worker reads a page of image keys, compresses several parts, loses its process after part 8, and the retry starts a new session while the database still points at the old one. The final ZIP may look fine, yet the old fragments remain billable and the job's audit trail points at the wrong transfer. Persisting the session before the first part and making the terminal state conditional on the same export ID closes that gap.

One subtle choice deserves a test: the ZIP stream must be finalized before `finishMultipart` runs. The final directory is part of the ZIP, so completing storage after the producer closes is a correctness condition, not a performance detail.

## Where does the simple upload approach fail?

A single request is a reasonable baseline for small exports. It has less state and fewer provider calls. It becomes painful when a transient connection loss forces the worker to retransmit the complete archive, especially while several tenants are exporting at once.

The comparison is about operational ownership, not a vendor scoreboard.

| Approach | Good fit | Main limit |
| --- | --- | --- |
| One long object upload | Small archives and low restart cost | A failed transfer repeats all bytes |
| Multipart object upload | Large archives where part retries improve throughput | Requires receipts, completion, and cleanup |
| Direct browser upload | User-selected files entering the system | CORS, authorization, and client retry state |
| Native provider integration | Replication, retention locks, or provider-specific controls | More SDK and credential surface to own |

Multipart has its own cost. It adds session state, ordered receipts, abort handling, and a reconciler for abandoned sessions. A very small export may not justify that machinery. The catch is that multipart is also not a queue: it does not limit concurrent ZIP generation, and it does not answer whether a user is allowed to download the result.

Set lifecycle cleanup for incomplete uploads as a backstop, while still aborting on a terminal worker failure. Lifecycle behavior varies by service and policy, so verify the actual rule in the chosen provider's documentation. The application should also delete expired export objects according to retention requirements; an expiring signed link does not delete the underlying object.

## What should the export runbook measure before changing storage?

Track p50 and p95 archive-build time separately from part-upload time. Add part size, active workers, retry count, HTTP 429 count, retransmitted bytes, abandoned session age, completion latency, signed-link expiry failures, and bytes retained after the export retention window. These measurements reveal whether the bottleneck is image reads, ZIP compression, network throughput, or a concurrency limit.

I would run the same tenant-sized fixture through a single-upload baseline and a multipart candidate, then test a worker termination between two parts and immediately before completion. The expected result is either a resumable session with the recorded receipts or a clearly failed export with cleanup; there must be no downloadable partial ZIP. I'm not sure which part size will win for your catalog, because image dimensions and compression ratio change the shape of the stream. Your mileage may vary.

For a solo team, keep the decision rule explicit: use the simpler upload while restart cost is acceptable; move to multipart when large-file throughput and retransmission cost dominate; keep a direct provider integration when you need native replication, retention locks, or public delivery controls that the adapter does not expose. That is the trade-off I would ship against.

## References

- [AWS S3 object lifecycle management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [Google Cloud Storage documentation](https://cloud.google.com/storage/docs)
