# Node.js Metrics Dashboard API: Keeping Custom Import KPIs Verifiable Across Rollbacks

Choose a metrics dashboard API only after it can show a scheduled import missing its deadline across a deployment rollback. For a fintech SaaS app, a healthy worker is insufficient evidence: the business KPI is whether the expected import produced a committed, usable result. These custom business KPIs need a completion contract before they need a chart.

TL;DR: Keep the schedule and completion evidence outside the worker deployment. Derive an overdue-import KPI from that evidence, then send the dashboard a small aggregate. Evaluate hosted metrics against this rollback test before comparing chart features or telemetry spending.

The proposed data flow is deliberately small. A scheduler records an expected import slot; an importer commits its result with a completion receipt; a separate evaluator compares expected slots with receipts after their deadlines. A metrics API receives the resulting overdue count and evidence status. The dashboard presents those signals together, while an operator can investigate a particular slot in an access-controlled operational view. These are design requirements for the example, not claims about a hosted service.

## Start with a runnable overdue-import calculation

The following TypeScript example models one scheduled settlement import. Times are synthetic milliseconds on the same clock, and the 20 ms grace period is an example policy, not a fintech standard. In production, load slots and receipts from durable storage and use a consistent time source. The evaluator intentionally accepts no release identifier: rolling application code backward must not change which business obligation it evaluates.

```ts
type Slot = {
  id: string;
  dueAt: number;
  grace: number;
};
type Receipt = {
  slotId: string;
  committedAt: number;
};
type Snapshot = {
  slots: Slot[];
  receipts: Receipt[];
  complete: boolean;
};
type Signal = {
  evidenceAvailable: 0 | 1;
  overdueImports: number | null;
};

function evaluate(snapshot: Snapshot, now: number): Signal {
  if (!snapshot.complete) {
    return { evidenceAvailable: 0, overdueImports: null };
  }

  const completed = new Set(
    snapshot.receipts
      .filter((receipt) => receipt.committedAt <= now)
      .map((receipt) => receipt.slotId),
  );
  const overdueImports = snapshot.slots.filter(
    (slot) => now >= slot.dueAt + slot.grace &&
      !completed.has(slot.id),
  ).length;

  return { evidenceAvailable: 1, overdueImports };
}

const slot: Slot = {
  id: "settlement-import:slot-001",
  dueAt: 100,
  grace: 20,
};
const pending: Snapshot = {
  slots: [slot], receipts: [], complete: true,
};

function expect(actual: Signal, overdue: number | null): void {
  if (actual.overdueImports !== overdue) {
    throw new Error("Unexpected overdue-import count");
  }
}

expect(evaluate(pending, 119), 0);
expect(evaluate(pending, 120), 1);
expect(evaluate({ ...pending, complete: false }, 120), null);

const receipt: Receipt = { slotId: slot.id, committedAt: 125 };
const completed: Snapshot = {
  ...pending, receipts: [receipt, receipt],
};
expect(evaluate(completed, 124), 1);
expect(evaluate(completed, 125), 0);
console.log("Import evidence checks passed");
```

This is an evaluator, with executable boundary checks; it is not a database adapter or a dashboard client. The duplicate receipt checks the intended counting rule: one scheduled obligation becomes complete once. A late result clears the current overdue condition. It does not erase the fact that the slot missed its deadline, which belongs in retained incident evidence.

Keep that distinction.

Before connecting a metrics API, define the snapshot contract. `complete` must mean the read includes the authoritative expected slots and completion receipts for the evaluation scope. A query that returns some rows successfully is not enough. If that guarantee cannot be established, emit unavailable evidence and leave the overdue count unknown. Do not convert the null into zero during export. The API adapter should omit that observation and send the separate evidence signal.

## How should a SaaS app metrics dashboard API verify import results?

Use a committed business outcome as the completion boundary. For this design, the receipt and the imported result must become durable together. Otherwise, a receipt written before the result could suppress the very alert the system exists to produce. If the storage design cannot make them atomic, specify a reconciliation step and withhold completion until it verifies the result. That adds work, but it keeps a chart from asserting success prematurely.

A zero-row result needs an explicit contract. If an upstream settlement source legitimately has no records for a scheduled slot, a verified empty result may satisfy that obligation. An empty response that has not passed validation should not. Row count can help explain an import, but it should not silently decide whether the business obligation was fulfilled.

The opposite trap is a completion counter that rises during retries. Suppose the illustrative slot above runs twice. Two worker successes still represent one fulfilled obligation. Count distinct expected slots with valid receipts; retain attempt details elsewhere. This keeps the dashboard understandable without pretending that retry activity is business output.

Error grouping solves a different part of the investigation. The event-grouping documentation in References describes how events are grouped; it does not establish that a scheduled import produced its result. Treat grouped errors as supporting diagnostic evidence. The absence of an error event must never satisfy an expected slot.

## Make rollback preserve the business obligation

The rollback acceptance test is simple: an unfulfilled slot remains overdue before and after the deployment changes. Its identifier, deadline, grace policy, and completion definition must survive. Persist those values with the obligation instead of recalculating historical deadlines from whatever configuration the current worker happens to load.

Consider a proposed release that changes a settlement import from one output to two required outputs. An older worker may still understand the slot identifier while being unable to satisfy the newer completion contract. That is a rollback compatibility question, and a green process-health chart cannot answer it. Before rollout, either keep the new contract compatible with the rollback target or explicitly prevent that target from processing those obligations. Require complete evidence for the new contract before counting success. For example, if the new obligation requires both settlement records and a validation summary, a receipt covering only the records must leave the obligation pending. Encode that distinction in the persisted completion contract and validate receipts against it before constructing the evaluator's input. Otherwise, a rollback could silently weaken the meaning of success even though the slot identifier stayed stable. This favors a little durable state over an extremely thin telemetry integration. For a solo builder, that state is extra implementation work. It earns its place because it also supplies the evidence needed to decide whether a rollback actually restored imports. Keep the first version narrow: one import class, one completion rule, one overdue signal. Expand after the failure rehearsal works.

Retain the evaluator's availability and freshness separately from the overdue value. If evaluation stops while the last count is zero, the dashboard must be able to distinguish that stale observation from a fresh all-clear. Require an explicit missing-data policy from the alerting system and verify it in the rehearsal; do not infer its behavior from how the chart draws a gap.

## Select the metrics API with one failure rehearsal

For this SaaS KPI, a simple hosted metrics dashboard API should let the adapter submit the aggregate, associate it with a bounded import class and environment, and preserve enough timing information to evaluate freshness. Check the actual ingestion and alert semantics against a fixture before adopting it. A convenient write call alone does not establish how absent observations, delayed delivery, or repeated submissions affect the displayed value.

Use the example's 119, 120, and 125 checkpoints as a small acceptance fixture. Before the deadline, the slot is pending. At the deadline it becomes overdue. A later committed result clears the current condition. Repeat the exercise while rolling the worker back and while making the evidence read unavailable. The latter case must produce an evidence problem, not a clean business KPI. This is a practical selection criterion without ranking providers.

Bound the scope.

Keep slot identifiers and settlement payloads out of metric dimensions; retain them in the operational evidence store. For this design, use a small, declared set of aggregate dimensions and review any proposed addition. That limits the amount of telemetry the application creates and gives a solo operator a smaller vocabulary to debug. The trade-off is less detail directly in the chart, so preserve a dependable path from the alert to the underlying obligations.

Spending still matters. Estimate observation frequency, dimension combinations, and retained history for the intended deployment, then check how a candidate accounts for them. Include the effort of diagnosing a failed rollback in that decision. A modest dashboard with clear missing-data behavior is easier to assess than a broad instrumentation project whose business outcome remains ambiguous.

## Before the first rollout

Record the completion contract and the reason for the grace period. Verify that the expected-slot writer, importer, and evaluator agree on slot identity, and that the chosen rollback target can read the persisted contract. Exercise a retry, a legitimate empty result, a late commit, and an unavailable evidence read. Check that the operator can distinguish each condition without inspecting worker logs first.

Then verify the human response. Assign an owner to an overdue import alert and an evidence-unavailable alert, document how to locate the affected obligations, and preserve enough history to inspect a missed deadline after recovery. Keep alert delivery independent of the release being evaluated where practical.

Ship the narrow version once that rehearsal passes. The decision rule stays the same as the application grows: a rollback is operationally safe only when the expected imports remain visible and their committed outcomes can still be verified.

## References

- Event grouping mechanics and the boundary between grouped diagnostic events and business completion evidence: https://docs.sentry.io/concepts/data-management/event-grouping/
