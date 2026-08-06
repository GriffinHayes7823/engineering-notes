# Fallback models behind one key: the chatbot API layer I'd build for a SaaS app

Bottom line: for an in-app SaaS chatbot, put a thin routing layer of your own in front of a single chat-completions request shape, keep the fallback ladder as ordered configuration, and treat one key across OpenAI, Claude and Gemini as a billing convenience rather than a reliability plan. The wiring question — how many credentials, how many invoices — is real but small. The question that will page you at night is what your app does on the second attempt.

I've built this three times. Only the third version is one I'd defend.

Version one imported three vendor SDKs into the same Express handler and switched on a `provider` string. It worked until the streaming shapes drifted: one SDK yielded deltas, one yielded full messages with a diff flag, one wanted an async iterator I had to adapt. Version two collapsed all of that behind a self-hosted proxy, which was better, right up to the evening the proxy container OOM'd and took every assistant turn in the product down with it. Version three is the one described below — a single request shape, an ordered ladder in config, and about fifty lines of my own routing code that I can read at 2am without a debugger.

## Should a SaaS chatbot API route across fallback models behind one key?

For the wiring, yes. For the policy, no — that stays in your code.

The data flow is worth saying in plain words before any of the code makes sense. Your in-app widget posts a message array to your own backend route. That route attaches tenant context and picks the first rung of a ladder it read from environment config. It sends one HTTP request with a messages array and a model string, and gets back either a completion or a status code. On a rejection it decides — from the status, not from a library's opinion — whether to try the next rung or hand the failure to the user. Whatever answers, the bytes stream to the browser as server-sent events, and one row lands in your own table describing which rung served the turn and how long it took.

One credential across several model families genuinely removes work. Key rotation stops being a three-vendor project, and a solo founder's monthly reconciliation goes from a spreadsheet to a line item. What it does not do is decide when a slow provider counts as a failed provider. That decision is product-specific: a support assistant should degrade to a smaller model and keep answering, while a code-generation panel should probably surface the error and stop burning tokens on a weaker model that'll produce something the user rejects anyway.

## The routing layer, in about fifty lines of TypeScript

Everything below is a ladder walk with explicit status handling. No SDK, no vendor types, one request shape per rung.

```ts
// chat-ladder.ts — one wire format, an ordered ladder, no vendor SDKs.
type Rung = { base: string; model: string; key: string };

// CHAT_LADDER="https://a.example|fast-model|KEY_A;https://b.example|strong-model|KEY_B"
const LADDER: Rung[] = (process.env.CHAT_LADDER ?? "")
  .split(";")
  .filter(Boolean)
  .map((entry) => {
    const [base, model, keyEnv] = entry.split("|");
    return { base, model, key: process.env[keyEnv] ?? "" };
  });

export type Attempt = { rung: number; model: string; status: number; ms: number; retryAfter: string | null };

export async function chat(messages: unknown[], trace: Attempt[]): Promise<Response> {
  for (let i = 0; i < LADDER.length; i++) {
    const rung = LADDER[i];
    const started = Date.now();

    const res = await fetch(`${rung.base}/v1/chat/completions`, {
      method: "POST",
      headers: { "content-type": "application/json", authorization: `Bearer ${rung.key}` },
      body: JSON.stringify({ model: rung.model, messages, stream: true }),
    });

    // Every attempt is recorded, including the ones that eventually succeed.
    trace.push({
      rung: i,
      model: rung.model,
      status: res.status,
      ms: Date.now() - started,
      retryAfter: res.headers.get("retry-after"),
    });

    if (res.ok) return res;
    // Throttling and upstream faults earn another rung. A 400 means we sent junk;
    // trying a different model with the same junk just doubles the bill.
    if (res.status !== 429 && res.status < 500) return res;
  }

  return new Response(JSON.stringify({ error: "ladder_exhausted", trace }), {
    status: 503,
    headers: { "content-type": "application/json" },
  });
}
```

The `trace` array is the part I'd fight for in review. It costs nothing, it goes into the same log line as the turn id, and it's the difference between knowing your fallback works and hoping it does.

## What the fallback ladder actually costs you

Here's the number that taught me this. We had a per-minute token cap I'd never read carefully, and it only bit during our 8pm peak. The retry wrapper I'd inherited caught 429s, backed off, and retried up to three times — and because attempt four usually succeeded, the route returned 200. Our error rate sat at 0.4% for eleven days while roughly a fifth of evening turns took over 9 seconds end to end. Support tickets said "the assistant is slow", which I filed under vague feedback. The retry loop wasn't broken; it was doing exactly what it was told, and it was hiding the one signal that mattered. I found it by logging attempt counts, not by reading provider dashboards.

The catch with a ladder is that it converts hard failures into soft ones, and soft failures are harder to see. It also multiplies cost in a way that's invisible per request: two rungs at 3,000 prompt tokens each is 6,000 tokens billed for one visible answer, and a quality-driven fallback (retrying because the first answer was thin) can double that again. Cap the ladder at two rungs unless you have data justifying a third.

| Approach | Swap cost | What you operate | Where failures surface |
| --- | --- | --- | --- |
| One SDK per provider, wired into handlers | High — types and stream shapes differ | Nothing extra | Scattered across handlers |
| Self-hosted gateway proxy | Low — a config change | A proxy, its config, its uptime | Gateway logs, if you ship them |
| Hosted aggregator behind one credential | Low — a model string | Nothing | Vendor dashboard, at vendor granularity |

Aggregation isn't free of trade-offs either. A shared wire format tends to expose the intersection of provider features, so vendor-specific things — extended thinking budgets, prompt caching semantics, structured-output modes — either arrive late or pass through as opaque extra fields. Stick with a direct integration when one of those features is load-bearing for your product, and when a compliance review names a specific processor you can't route around.

## Testing and the three numbers I keep on a dashboard

You can test a ladder without touching a provider. Point `CHAT_LADDER` at a local stub that returns 429 with a `Retry-After` header, assert the trace has two entries, and assert the second one succeeded.

```ts
// ladder.test.ts — the failure path deserves a test more than the happy path does.
const trace: Attempt[] = [];
process.env.CHAT_LADDER = "http://127.0.0.1:8081|stub-a|K1;http://127.0.0.1:8082|stub-b|K2";

const res = await chat([{ role: "user", content: "ping" }], trace);

assert.equal(res.status, 200);
assert.equal(trace.length, 2);
assert.equal(trace[0].status, 429);
```

The three numbers worth a dashboard tile: rung-two share (what fraction of turns needed a fallback), tokens billed per delivered answer, and p95 measured across the whole ladder walk rather than per HTTP call. That third one is where my 9-second problem was hiding — per-call latency looked fine because each individual attempt was fast.

Deployment matters more than it sounds. Because the ladder lives in an environment variable, changing providers is a restart, not a release, which means you can respond to a degraded upstream in about ninety seconds. It also means an unreviewed environment change can quietly reroute every customer conversation, so put the ladder value in the same review path as your code. I'm not sure there's a clean answer here; every team I've asked has landed somewhere different.

## What I check before turning the ladder on

Start with the boring things. Every rung answers the same request shape, and a synthetic prompt through each one is part of the deploy check, because a rung nobody exercises is a rung that has quietly expired. Rate-limit responses are surfaced with their `Retry-After` value intact and never collapsed into a generic 500 — the caller can't back off politely against a number it never received. Attempt counts land in the same structured log line as the turn id and tenant id, so a "the bot is slow" ticket becomes a query instead of an investigation. Token spend is attributed per tenant, since one enthusiastic customer on a fallback-heavy workload will otherwise show up only as a bill you can't explain. And there's a documented answer to the question of what users see when every rung is exhausted, written before that day arrives rather than during it.

None of this needs a platform decision on day one. Your mileage may vary on the ladder depth, but the routing layer is maybe an afternoon of work, and it's the piece that keeps the choice reversible while everything upstream keeps changing.

## References

- HTTP semantics, status codes and retries — https://www.rfc-editor.org/rfc/rfc9110
- Additional HTTP status codes, including 429 Too Many Requests — https://www.rfc-editor.org/rfc/rfc6585
- Retry-After header reference — https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Retry-After
- Server-sent events, WHATWG HTML specification — https://html.spec.whatwg.org/multipage/server-sent-events.html
- LiteLLM, an open-source self-hosted LLM gateway — https://github.com/BerriAI/litellm
- openai/whisper, open-source speech recognition — https://github.com/openai/whisper
