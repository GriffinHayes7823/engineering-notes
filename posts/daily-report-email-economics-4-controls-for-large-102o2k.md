# Daily Report Email Economics: 4 Controls for Large-Recipient Queue Retry Latency

A large recipient list turns a daily email into an inventory curve: work arrives at once, while delivery capacity is metered over time. Short answer: have the cron trigger release bounded queue work, size the worker window from an explicit latency envelope, and reserve part of that envelope for retries.

Do not keep a web request open. Its lifetime is a poor control for a workload whose duration grows with the audience, and retrying that request can repeat the entire fan-out. The useful experiment is narrower: find the least worker capacity that drains one report before its deadline when realistic rate pressure and retry traffic are present.

Measure the curve.

## Cost accounting for worker-seconds and queue age

Start with the product promise rather than the cron expression. Suppose a developer-tools usage report has a 60-minute delivery window. That hour is an example policy, not a universal benchmark. Divide it into late-data allowance, first-attempt time, and retry headroom. If the first pass consumes the whole hour, there is no retry budget; if it targets the shortest possible time, idle capacity may dominate the cost of a job that runs once per day.

The initial model needs four observed inputs: eligible recipients, sustainable completions per worker per second, maximum concurrency, and acceptable age for the oldest queued job. Use a controlled load test that includes template rendering and network time. A provider's advertised peak is not a substitute for an end-to-end observation. I'm not sure any fixed concurrency recommendation survives a change in message size or recipient geography; repeating the test with production-shaped data resolves that uncertainty.

Here is an illustrative calculation, not a benchmark. At two completed attempts per worker per second, 20 workers provide nominal capacity for 40 attempts per second. A 60,000-recipient run then needs at least 25 minutes before variance and retries. Increase the worker window and the first pass gets shorter at a higher capacity cost. Decrease it and queue residence grows. This small calculation also catches impossible plans early: a ten-minute target cannot be recovered with clever backoff if the configured first-pass capacity already requires 25 minutes.

Average throughput still lies by omission. A run can show healthy completions per minute while its oldest jobs are already outside the promised window, so record both oldest-job age and estimated drain time. Depth says how much inventory remains; age says whether the service promise is in danger.

## How should a daily report email queue worker handle large recipient retries?

Make each recipient attempt the unit of admission. The trigger releases a bounded slice, records enough state to resume publishing, and returns. It does not render the whole audience, wait for delivery, or replay the report after a partial failure. Each job carries a stable report version and recipient identifier; durable idempotency state prevents the same version from becoming two intentional deliveries to one address.

The worker window should react to queue age and rate pressure. Increase it gradually as age approaches the target, rather than launching every recipient through `Promise.all()`. When HTTP `429 Too Many Requests` appears, reduce or pause admission. MDN defines `429` as the response for too many requests in a given time and notes that a server may include `Retry-After`; honor that value when it is supplied.

```ts
type Window = {
  concurrency: number
  maxConcurrency: number
  oldestJobAgeMs: number
  targetAgeMs: number
  retryAfterMs?: number
}

type Decision =
  | { kind: "pause"; delayMs: number }
  | { kind: "run"; concurrency: number }

function chooseWindow(input: Window): Decision {
  if (input.retryAfterMs !== undefined) {
    return { kind: "pause", delayMs: input.retryAfterMs }
  }

  const next = input.oldestJobAgeMs > input.targetAgeMs
    ? input.concurrency + 1
    : input.concurrency - 1

  return {
    kind: "run",
    concurrency: Math.max(1, Math.min(input.maxConcurrency, next)),
  }
}
```

This function is a policy seam, not a complete consumer. The worker around it still needs an atomic claim, a bounded execution time, and a durable outcome. Response interpretation belongs in a small provider adapter because a status code does not capture every delivery contract. Test the policy with a fake clock, then test the awkward crash boundary: stop a worker after a provider accepts a message but before local success is recorded. On redelivery, the idempotency record must be consulted before another send is admitted.

No giant promise array.

The second run matters more.

A happy-path run tests syntax. Capacity planning needs three shapes: a normal audience with no injected retryable outcomes, the full expected audience under an enforced rate limit, and a full audience where a controlled fraction of attempts is deferred into the retry window. The fraction is an experiment input, not a prediction. Keep report content and recipient count constant across comparisons so a concurrency change is the only main variable.

Run each shape with a low, middle, and high worker cap. For every run, capture p50 and p95 completion latency, oldest-job age, attempts per successful recipient, worker-seconds, and jobs crossing the deadline. Repeat a run when environmental noise changes materially. One result should be visible in the data: lowering concurrency reduces active worker time only if the longer wall-clock duration does not add enough polling and baseline process cost to cancel the gain. Your mileage may vary with how workers are billed, which is why worker-seconds and elapsed time both belong in the record.

Then interrupt the middle run. Pause consumption long enough to create a backlog, resume it, and observe whether the controller spends capacity as the oldest age approaches its target. This is the part a steady synthetic stream misses. The actual daily-report shape is a burst, and a controller tuned against a smooth arrival rate may look economical while never catching up after a deployment or a short provider throttle. Keep the test outcome as a small decision record: workload inputs, chosen cap, measured drain time, retry headroom, and the date. Re-run it after a material list-size or template-cost change.

Priority is useful only as an isolation experiment. RabbitMQ documents priority support for classic queues and explains that consumers may need constrained prefetch so messages remain available for prioritization. The general lesson applies beyond one broker: a consumer that reserves a large local batch weakens broker-side priority. If daily reports share a lane with urgent notifications, test the urgent message's queue age during the report burst. A priority number without that observation is configuration, not evidence.

Now inject retry traffic.

Retry traffic is delayed inventory. Give it a separate allowance inside the delivery envelope, add jitter, cap attempts, and stop at an absolute report deadline. Once that deadline passes, move the job to a terminal review state instead of spending indefinitely on an email whose useful window has closed.

Keep classification conservative. A `429` represents rate pressure and should follow `Retry-After` when present. Authentication or malformed-request outcomes should not enter an automatic loop. Other decisions must follow the provider's documented contract and recorded responses. Count first attempts and retry attempts separately; a blended processed-message total can rise during a retry wave even as useful completions fall. Attempts per successful recipient connects failure pressure to extra worker and provider consumption without inventing a universal price.

If the scheduler crosses a trust boundary, authenticate its release message. RFC 2104 specifies HMAC as keyed hashing for message authentication. Sign stable run metadata, verify it before admitting work, and keep the shared secret out of both the job body and logs. Add an expiry and unique run identifier as replay boundaries, because authentication establishes who formed a message; it does not make duplicate scheduling harmless.

Stop means stop.

## Read the experiment before selecting capacity

This queue design is not suitable for a tiny internal list when one scheduled process consistently finishes inside the latency envelope and recipient-level retry timing adds no value. A process with a durable run outcome has fewer moving parts and less idle overhead. Keep that approach until measured queue age, provider throttling, or recovery work demonstrates the need for fan-out control.

At the other boundary, one shared lane may fail the test when reports compete with latency-sensitive notification mail. Split the lanes if the full-load experiment shows report inventory harming the urgent deadline, accepting that separate lanes create more worker settings, alarms, and capacity choices. Stick with one lane while measured priority and prefetch behavior meet both promises.

Personalization changes the cost model too. Storing fully rendered jobs consumes more queue storage; rendering again on each attempt consumes more compute. Persisting an immutable data snapshot or content reference and rendering in the worker can keep a run consistent, but it is not suitable when policy prohibits retaining recipient-level report data. Use a compliant batch export or a notification workflow with the required retention controls in that case.

Choose the least worker capacity that passes all three load shapes with retry headroom. A larger pool is merely faster and a smaller one merely cheaper until measurements tie either choice to the delivery promise.

## References

- https://www.rfc-editor.org/rfc/rfc2104
- https://www.rabbitmq.com/docs/priority
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/429
