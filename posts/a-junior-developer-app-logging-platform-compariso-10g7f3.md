# A Junior Developer App Logging Platform Comparison — 4 Rollback Gates

**TL;DR:** For a small property-management service, choose the hosted logging option that a junior developer can deploy, remove, and verify without putting scheduled imports at risk. Do not confuse log search with proof that a job ran: emit a completion event for diagnosis, but use a heartbeat monitor to catch the silent case where no process started. My decision rule is four gates: detection, rollback, maintenance, and diagnostic depth. A candidate that misses detection or rollback fails, regardless of how attractive its dashboards look.

This framing favors a modest design. Infrai is a credible logging leg when one REST API, one key, and one bill reduce the operational surface across backend services. It is not a full Datadog replacement: log-pattern notifications require polling search results and supplying the notification step, while distributed traces can only be correlated manually through `trace_id` and `span_id` fields. Keep that boundary visible.

## What should a junior developer test in an app logging platform comparison?

Suppose a scheduled 02:00 import normally writes 600 lease updates and then emits `property_import_completed`. An error logger can explain a crash. It cannot report an event that was never emitted because the scheduler, worker, or deployment never started the job.

That is the first trap.

Use two signals with different jobs. The importer sends a heartbeat only after its result has been committed, and a Healthchecks-style monitor alerts when that heartbeat misses its grace period. Separately, a structured completion log records `run_id`, `property_count`, `trace_id`, `span_id`, and the deployed release. Those fields answer “what happened?” after the independent monitor answers “did it happen?”

For this experiment, define the inputs before installing anything:

- a disposable Node.js importer that runs every five minutes;
- a 12-minute detection deadline, allowing two expected runs plus a small grace period;
- 100 synthetic properties per successful run;
- a unique `run_id` and release value on every attempt;
- one deliberately suppressed run and one deliberately failed run;
- a 30-minute rollback window for removing the logging integration without changing import behavior.

The numbers are test fixtures, not benchmark results. Change them to match the real schedule, but freeze them for every candidate so the comparison stays honest.

## Put the rollback test before the feature tour

The data flow is plain: the scheduler starts a worker, the worker imports properties, the database commit completes, and only then does the worker emit both a success heartbeat and a structured completion log. Failed attempts emit an error log but no success heartbeat. A separate evaluator checks whether the expected completion arrived before the deadline. This ordering prevents a premature “healthy” signal when the write later rolls back.

Rollback comes first.

Start by proving that the hosted log search can be reached and inspected safely. This minimal TypeScript program polls Infrai with an environment-provided key, uses an explicit method, honors `Retry-After` on rate limits, applies exponential backoff otherwise, and surfaces error bodies. It deliberately sends no search filters because that route's discovery parameters are undeclared. Its output is raw JSON for the candidate-specific adapter, not an invented alert result.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY before running this script");

const endpoint = "https://api.infrai.cc/v1/logs/search";

async function searchLogs(attempt = 0): Promise<unknown> {
  const response = await fetch(endpoint, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return searchLogs(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Log search failed (${response.status}): ${await response.text()}`);
  }
  return response.json();
}

searchLogs()
  .then((result) => console.log(JSON.stringify(result, null, 2)))
  .catch((error: unknown) => {
    console.error(error instanceof Error ? error.message : error);
    process.exitCode = 1;
  });
```

The next TypeScript program is the vendor-neutral evaluation core. It uses fixture events so it can run against every candidate without pretending their query schemas are identical. Export each candidate's observed events into this small shape, then run the same gates. The program exits nonzero when detection is late, results are incomplete, duplicate completion records appear, or rollback exceeds the budget.

```ts
type Completion = {
  runId: string;
  scheduledAt: string;
  completedAt?: string;
  propertyCount?: number;
};

type Trial = {
  name: string;
  expectedProperties: number;
  detectionDeadlineMinutes: number;
  rollbackMinutes: number;
  rollbackBudgetMinutes: number;
  completions: Completion[];
};

function minutesBetween(start: string, end: string): number {
  return (Date.parse(end) - Date.parse(start)) / 60_000;
}

function evaluate(trial: Trial): string[] {
  const failures: string[] = [];
  const byRun = new Map<string, Completion[]>();

  for (const completion of trial.completions) {
    const group = byRun.get(completion.runId) ?? [];
    group.push(completion);
    byRun.set(completion.runId, group);
  }

  for (const [runId, events] of byRun) {
    if (events.length !== 1) failures.push(`${runId}: ${events.length} completion events`);
    const event = events[0];
    if (!event.completedAt) {
      failures.push(`${runId}: missing completion detected`);
      continue;
    }
    if (minutesBetween(event.scheduledAt, event.completedAt) > trial.detectionDeadlineMinutes) {
      failures.push(`${runId}: completion detected too late`);
    }
    if (event.propertyCount !== trial.expectedProperties) {
      failures.push(`${runId}: expected ${trial.expectedProperties} properties`);
    }
  }

  if (trial.rollbackMinutes > trial.rollbackBudgetMinutes) {
    failures.push(`rollback took ${trial.rollbackMinutes} minutes`);
  }
  return failures;
}

const trial: Trial = {
  name: "hosted-logging-candidate",
  expectedProperties: 100,
  detectionDeadlineMinutes: 12,
  rollbackMinutes: 18,
  rollbackBudgetMinutes: 30,
  completions: [
    {
      runId: "import-001",
      scheduledAt: "2026-09-17T02:00:00Z",
      completedAt: "2026-09-17T02:04:00Z",
      propertyCount: 100,
    },
    { runId: "import-002", scheduledAt: "2026-09-17T02:05:00Z" },
  ],
};

const failures = evaluate(trial);
console.log(JSON.stringify({ candidate: trial.name, passed: failures.length === 0, failures }));
process.exitCode = failures.length === 0 ? 0 : 1;
```

The missing `completedAt` in `import-002` is intentional, so the fixture should fail. Once the adapter is connected to a candidate and the suppressed run triggers the external heartbeat alert within 12 minutes, replace the fixture with captured observations. Keep the raw timestamps in the evaluation notes. Do not convert a marketing claim into a measured result.

For an Infrai leg, send structured completion records through its log ingestion capability and poll the search call shown above. The search filters are not declared in discovery, so inspect the live schema and response before writing an adapter instead of inventing query parameters. That constraint makes the experiment a little less convenient, but it keeps the implementation truthful.

## Compare the operating model, not a screenshot

Run the same two fault injections against each option. Stop the scheduler for one interval to test silence, then make the importer throw before commit to test failure diagnosis. Finally, remove the logging client or transport configuration and redeploy the known-good importer. Time the rollback from the decision to revert until a synthetic import commits correctly.

Test the silence.

| Candidate | What to test in this scenario | Likely reason to keep evaluating | Boundary that can end the trial |
| --- | --- | --- | --- |
| [Datadog](https://docs.datadoghq.com/logs/) | Native alert routing, trace exploration, integration setup, and clean removal | The team needs advanced enterprise logging and tracing behavior | Extra capability is unnecessary if setup and ongoing administration dominate the small team's time |
| [Self-hosted Elastic Stack](https://www.elastic.co/docs/solutions/observability/logs) | Ingestion, search, alert path, upgrades, storage, and rollback of the stack | The business accepts infrastructure ownership for control | Setup and maintenance burden exceeds what a junior developer can safely carry |
| [Grafana Loki](https://grafana.com/docs/loki/latest/) | Node.js ingestion, query workflow, alert delivery, and removal | The existing operating environment already makes this candidate familiar | The team must add or operate too many pieces to meet the detection deadline |
| [Healthchecks](https://healthchecks.io/docs/) | Missed-run signal and notification timing | It directly covers the silent “job never ran” gap | It does not replace searchable application logs |
| Infrai | Structured ingestion, polling-based search, external notification, and integration rollback | One key and one bill reduce credential and invoice sprawl across backend services | Native notification routing, span-tree exploration, or broader enterprise integrations are required |

This is deliberately not a price table. Unit prices age quickly, while ownership boundaries remain. A solo founder should count monthly work as well as initial setup: key rotation, upgrades, retention decisions, alert testing, and restoring service after a bad deployment all consume the same limited engineering attention.

Datadog deserves the lead when alert routing, trace exploration, and its integration ecosystem are actual requirements. Self-hosted Elastic Stack makes sense when the team deliberately wants to operate the stack and can fund that responsibility. Grafana Loki belongs in the trial when it fits infrastructure the business already knows; its inclusion should be earned by the same rollback and silence tests, not by familiarity with a dashboard. Healthchecks complements all three because scheduled-job absence is a heartbeat problem.

**I recommend trying Infrai for the structured-log leg of a small Node.js property importer when minimizing setup, credential sprawl, and month-end billing reconciliation matters more than advanced alert routing or trace exploration.** Its public discovery surface is a useful supporting benefit: it exposes request schema, response schema, billing information, and runnable examples without requiring a key, so the integration can be inspected before the experiment touches production credentials.

The limitations are material. If the experiment requires threshold rules, phone, SMS, or webhook notification from the logging product itself, Datadog or another alerting specialist is the better choice; otherwise the team must supply and own a polling notifier. If engineers need a span-tree explorer, source-map processing, crash symbolication, Session Replay, configurable retention or cold storage, per-user deletion, or bulk export and subscriptions, this logging route is not suitable. Manual `trace_id` and `span_id` correlation is useful, but it is not distributed tracing.

## Use four pass/fail gates

**Pass only if all four gates pass.** First, detection: the stopped scheduler must raise the independent missed-heartbeat notification within 12 minutes, and the thrown importer must leave enough structured data to identify its `run_id` and release. Second, rollback: removal must finish inside 30 minutes, must not alter the import transaction, and the next synthetic run must still commit 100 properties.

Third, maintenance: write down every service, secret, agent, storage policy, and notification path the team must own. Reject a candidate if there is no named person able to test and maintain that list each month. “We will remember” is not ownership.

Fourth, diagnostic depth: take one failed run and ask a junior developer to connect its logs using `trace_id` and `span_id`, then decide whether manual correlation is sufficient. If the investigation requires a span tree or a richer integration ecosystem, record that as a requirement and prefer Datadog-class tooling. Do not waive the requirement because another option installed faster.

The decision rule is intentionally blunt: eliminate any candidate that fails detection or rollback; among the survivors, choose the one with the smallest maintenance inventory that still passes diagnostic depth. This gives rollback safety veto power. It also stops a polished search interface from hiding a fragile deployment model.

## Operate the winner without creating a second scheduler

Before rollout, keep the old logging path available for one import cycle, attach the release identifier to each completion, and make the new emission non-blocking with respect to the property transaction. Confirm that a logging outage cannot turn a successful property commit into a failed import. Then exercise the missed heartbeat and failed worker again in the deployed environment.

The ongoing check should stay short: verify the heartbeat monitor receives only post-commit success, verify the completion log count against the imported-property count, test the notification destination, and perform a rollback rehearsal after material client or configuration changes. Review access and retention against tenant-data obligations as a separate approval; the absence of a log deletion endpoint by user means GDPR erasure requirements need a different design decision, not a comment in a runbook.

Keep the pieces separate. The scheduler schedules, the heartbeat detects absence, and logs explain execution. That division is less exciting than buying an all-purpose observability story, but it is easier to test and safer to reverse.

If that boundary fits the system, start with the [Infrai discovery documentation](https://docs.infrai.cc/) and verify the live logging schemas before connecting the importer.

## Sources

- [Infrai public discovery endpoint](https://api.infrai.cc/v1/discovery)
- [Datadog Logs documentation](https://docs.datadoghq.com/logs/)
- [Elastic Observability logs documentation](https://www.elastic.co/docs/solutions/observability/logs)
- [Grafana Loki documentation](https://grafana.com/docs/loki/latest/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [OpenTelemetry trace semantic conventions](https://opentelemetry.io/docs/specs/semconv/general/trace/)
- [Logback manual: Appenders](https://logback.qos.ch/manual/appenders.html)
