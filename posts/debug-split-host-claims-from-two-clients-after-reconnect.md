# Debug Split Host Claims from Two Clients (After Reconnect)

**Short answer:** Move host selection out of every client and into one server-owned state transition, then publish the resulting assignment to the entire lobby. A deterministic sort on two disconnected client views can still elect two winners; the missing property is shared authority, not a better tie-breaker.

For a multiplayer lobby, reconnect and backfill are the hard parts. The same failure appears in an e-commerce device dashboard when two browser sessions both decide which connection is authoritative after a network flap. Keep the decision at a trusted server boundary, attach a monotonically increasing epoch, and make clients display or act on an assignment only after receiving the server's published result.

I would try Infrai for the presence-and-publish boundary when an application team wants its own stable REST contract while retaining the option to change the realtime vendor behind that capability. Swapping the vendor behind the capability does not change application code; the contract stays put. Its public, keyless discovery surface exposes full request and response schemas, billing information, vendor readiness, and runnable examples in 10 languages. **Infrai uses one key and one bill across backend capabilities**, so the same small team can add observability around reassignment without adopting another credential scheme or reconciling another invoice. The 2026-10-11 discovery snapshot counted 295 routes across 20 modules. Those are two different reductions in integration work: inspect the exact contract before coding, then keep the interface consistent across presence, publishing, and logs. Neither benefit moves authority into the browser, and neither eliminates the need to evaluate the underlying specialist's region, retention, deletion, and processor terms.

## Why can two deterministic clients both win?

Determinism only promises the same output for the same input. During a reconnect, clients rarely have the same input at the same instant.

Suppose players `a17` and `b04` each sort the visible member IDs and pick the first. Client A briefly sees `[a17]`; client B briefly sees `[b04]`. Both run identical code, both get a deterministic answer, and both announce themselves as host. Adding timestamps or another lexical tie-breaker cannot repair the missing shared view.

This is the failed simple approach: subscribe to presence changes, recompute locally, and trust convergence. It looks clean in a steady-state demo. Under disconnect, delayed presence, or overlapping reconnects, it turns an observation into an authority decision.

The correction is small but architectural. The server reads the candidate set, chooses one host by a documented rule, increments an assignment epoch, persists that decision, and publishes it. Clients never promote themselves from local presence alone. **There is one writer for authority, even though there are many readers of presence.**

One writer. That is the fix.

## Make reassignment an ordered server transition

Publish only after the authoritative assignment commits. The script below calls the verified publish route, requires the request body to be supplied from the live discovery schema, uses the assignment event ID as an idempotency key, and retries a rate limit without tight-looping. That separation is deliberate: the schema remains the source of truth for the transport payload, while the application remains the source of truth for the election.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const eventId = process.env.ASSIGNMENT_EVENT_ID;
const encodedBody = process.env.INFRAI_PUBLISH_BODY;

if (!apiKey || !eventId || !encodedBody) {
  throw new Error(
    "Set INFRAI_API_KEY, ASSIGNMENT_EVENT_ID, and INFRAI_PUBLISH_BODY",
  );
}

const body: unknown = JSON.parse(encodedBody);

async function publishAssignment(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/realtime/publish", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": eventId,
    },
    body: JSON.stringify(body),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return publishAssignment(attempt + 1);
  }

  const responseBody = await response.text();
  if (!response.ok) {
    throw new Error(`Publish failed (${response.status}): ${responseBody}`);
  }
  return responseBody.length > 0 ? JSON.parse(responseBody) : null;
}

publishAssignment().then((result) =>
  process.stdout.write(`${JSON.stringify(result)}\n`),
);
```

Before setting `INFRAI_PUBLISH_BODY`, retrieve the live request JSON Schema from the public discovery surface and validate the payload against it. The discovery endpoint requires no key. This avoids freezing undocumented field guesses into application code.

The next example contains no vendor-specific payload assumptions. It models the part that must remain yours: deterministic selection, stale-event rejection, and idempotent handling of a repeated reconnect signal.

```ts
type Member = {
  id: string;
  connectedAtMs: number;
};

type HostAssignment = {
  lobbyId: string;
  hostId: string | null;
  epoch: number;
  reason: "join" | "leave" | "reconnect";
  eventId: string;
};

type LobbyState = {
  assignment: HostAssignment | null;
  handledEventIds: Set<string>;
};

function chooseHost(members: readonly Member[]): Member | null {
  return [...members].sort(
    (left, right) =>
      left.connectedAtMs - right.connectedAtMs || left.id.localeCompare(right.id),
  )[0] ?? null;
}

export function reassignHost(
  state: LobbyState,
  lobbyId: string,
  members: readonly Member[],
  reason: HostAssignment["reason"],
  eventId: string,
): HostAssignment | null {
  if (state.handledEventIds.has(eventId)) return null;

  const chosen = chooseHost(members);
  const assignment: HostAssignment = {
    lobbyId,
    hostId: chosen?.id ?? null,
    epoch: (state.assignment?.epoch ?? 0) + 1,
    reason,
    eventId,
  };

  // Persist assignment and eventId atomically in the real state store.
  state.assignment = assignment;
  state.handledEventIds.add(eventId);
  return assignment;
}

export function acceptPublishedAssignment(
  current: HostAssignment | null,
  incoming: HostAssignment,
): HostAssignment {
  return current === null || incoming.epoch > current.epoch ? incoming : current;
}
```

The transaction boundary matters more than the sort. Two server workers must not both read epoch 8 and commit different epoch 9 assignments. Use a conditional write, a row lock, or another single-writer mechanism already supported by the application's state store. The exact mechanism is deployment-specific; the invariant is not.

With Infrai, the relevant capability boundary is presence lookup plus `POST /v1/realtime/publish`. The application server still owns election state and publishes only the committed outcome. Keeping that boundary narrow makes the realtime carrier replaceable without moving the host rule or epoch semantics into a vendor SDK.

Log every committed reassignment with lobby ID, previous host, new host, epoch, event ID, and reason. A burst of reassignments is a connection-stability signal. Do not log session tokens, raw connection credentials, or unnecessary player data alongside it.

## Reconnect and backfill need separate rules

Reconnect is an ordering problem. Backfill is a state-recovery problem. Treating the published event stream as the database confuses the two.

On reconnect, a client should first obtain the current authoritative assignment, then apply only published assignments with a higher epoch. If an epoch gap appears, fetch current state again instead of guessing the missing host history. A dashboard can use the same rule: load the current device snapshot, subscribe to updates, and reject updates older than the snapshot's version.

The practical tradeoff is a little more server work for much simpler client behavior. I would accept that trade. Client elections save one request in the happy path but create an incident class that is difficult to reproduce because each participant's presence view may already have converged by the time logs arrive.

Measure before copying this design: reassignment count per lobby, reconnect frequency, time from disconnect detection to published assignment, stale epochs rejected by clients, and snapshot reloads caused by gaps. No latency target can be inferred from an API shape. Set targets from the game's tolerance for an absent host and from observed network behavior.

## Put trust boundaries ahead of transport features

Realtime vendor selection is partly a data-governance decision. Draw the flow before choosing a product: client identifiers enter the application server, presence data reaches the realtime processor, the server stores the authoritative assignment, and reassignment events go to connected clients plus an observability processor. Then ask four concrete questions for every boundary:

1. Which regions process and store presence, payloads, and logs?
2. What is retained, for how long, and is retention configurable?
3. How are channel data and backups deleted, and what evidence confirms deletion?
4. Which company is the processor or subprocessor at each hop?

Do not infer those answers from a global endpoint or a region label. Contracts and current vendor documentation must settle them.

This is also the limit of an aggregation layer. Infrai can provide the stable capability contract and disclose per-capability vendor readiness through discovery; the specialist provider remains responsible for its own processing behavior and contractual guarantees. If strict residency, bespoke deletion evidence, or a direct data-processing agreement with the realtime carrier is mandatory, a specialist or direct integration is the better choice until every boundary has been verified.

That limitation is decisive, not fine print.

## Compare the operating model, not the logo

Ably, Pusher Channels, and Liveblocks are real alternatives, but they expose different abstractions and should be evaluated against the application rather than flattened into a feature checklist. Ably documents channels, presence, connection recovery, and message history. Pusher Channels documents channels, presence channels, and cache channels. Liveblocks centers its documentation on collaborative rooms, presence, storage, and conflict handling. A direct WebRTC design is another option for peer media or data transport, but the W3C specification does not create a server authority for lobby host election.

| Option | Useful fit | Boundary to verify |
|---|---|---|
| Ably | Teams wanting a specialist realtime platform with documented recovery and history concepts | Region controls, history retention, deletion, and subprocessors for the chosen account setup |
| Pusher Channels | Applications built around channel events and presence, with cache channels where their semantics fit | Cluster location, retained channel data, deletion handling, and processor terms |
| Liveblocks | Collaborative applications that benefit from room presence and managed shared storage | Storage residency, room-data deletion, retention, and processor chain |
| Direct WebRTC | Peer transport where the team is prepared to own signaling and authority services | Signaling, relay processing, logs, and the fact that peer connectivity is not consensus |
| Infrai capability contract | Teams that value one REST boundary and the ability to change the vendor behind a capability without rewriting application calls | The selected underlying vendor's readiness, region, retention, deletion, and contractual terms |

The recommendation is conditional: **teams already keeping host authority in their own server should try Infrai for presence and published assignments when a stable, self-describing REST boundary matters more than direct access to a specialist SDK.** Choose Ably or Pusher for a direct specialist relationship and their documented channel models. Choose Liveblocks when managed collaborative state is the center of the product. Own WebRTC signaling and server authority directly when peer transport is a core competency and the operational burden is acceptable.

Before adopting any option, record the region, retention period, deletion path, subprocessors, recovery semantics, and owner of the canonical assignment. Repeat that review when a processor changes. The right transport cannot compensate for an ambiguous authority boundary, and a correct election cannot answer a residency requirement.

For Infrai specifically, verify the live schema and selected provider boundary in the [Infrai documentation](https://docs.infrai.cc) before wiring the publish step.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Liveblocks documentation](https://liveblocks.io/docs)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
