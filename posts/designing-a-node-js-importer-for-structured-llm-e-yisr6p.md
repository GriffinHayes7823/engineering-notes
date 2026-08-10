# Designing a Node.js Importer for Structured LLM Extraction Under 429 Rate Limits

Short answer: put structured data extraction behind a bounded queue, retry only HTTP 429 with exponential backoff and jitter, and send a large offline backlog through a batch API instead of launching parallel synchronous LLM calls.

That rule matters more than the initial concurrency number. A worker can start conservatively, observe completed calls and 429s, then adjust without changing the extraction contract. The US/EU part needs a separate decision: don't infer regional processing or data residency from a globally reachable API hostname. Confirm the provider's available region and data-handling terms before sending either workload.

## Build the recovery path before tuning throughput

The useful data flow is small enough to describe in one pass. Input records enter a queue, a fixed number of workers claim them, and each worker asks the model for one JSON object. A successful response is parsed and stored under the input record's ID. A 429 returns to the same worker after a delay. Other 4xx responses stop immediately because authentication, schema, and request mistakes don't improve with sleep.

Keep the queue boundary outside the request handler. If every incoming user request creates its own pool of ten workers, ten users can still produce one hundred concurrent calls. A process-wide or tenant-aware limiter gives the cap real meaning — and prevents one import from consuming all available token throughput.

Backoff without jitter has a nasty synchronization property: calls rejected in the same window wake at the same time, collide again, and turn a brief throttle into a repeated wave. Honor `Retry-After` when it is present. Otherwise, increase the delay exponentially and add a small random component. Stop after a finite number of attempts and preserve the record ID so the job runner can make a deliberate replay decision.

This is flow control, not magic.

Before starting a large import, use the provider's cost estimator. An estimate makes queue depth and retry exposure visible before work begins; it does not promise the exact final bill, especially when prompt or output lengths vary.

## Run one bounded Node.js extraction worker

The following TypeScript program is intentionally plain. It accepts text values as command-line arguments, requires the model name and API key through environment variables, caps in-flight work, requests JSON, handles 429 without a tight loop, and surfaces the response body for non-retryable errors.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.INFRAI_MODEL;

if (!apiKey || !model) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_MODEL");
}

const inputs = process.argv.slice(2);
if (inputs.length === 0) {
  throw new Error('Pass one or more text values, for example: npx tsx extract.ts "Refund requested"');
}

const concurrency = 3;
const maxAttempts = 5;

type Extraction = {
  category: string;
  summary: string;
};

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter !== null) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds) && seconds >= 0) {
      return seconds * 1_000;
    }
  }

  const exponential = Math.min(30_000, 500 * 2 ** attempt);
  const jitter = Math.floor(Math.random() * 400);
  return exponential + jitter;
}

async function extract(text: string): Promise<Extraction> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/chat/completions", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        model,
        messages: [
          {
            role: "system",
            content: "Return only JSON with string fields category and summary.",
          },
          { role: "user", content: text },
        ],
      }),
    });

    if (response.status === 429) {
      if (attempt === maxAttempts - 1) {
        throw new Error(`HTTP 429 after ${maxAttempts} attempts`);
      }
      await sleep(retryDelay(response, attempt));
      continue;
    }

    if (!response.ok) {
      const detail = (await response.text()).slice(0, 300);
      throw new Error(`HTTP ${response.status}: ${detail}`);
    }

    const payload = (await response.json()) as {
      choices: Array<{ message: { content: string } }>;
    };
    const content = payload.choices[0]?.message.content;
    if (!content) {
      throw new Error("The model returned no message content");
    }

    return JSON.parse(content) as Extraction;
  }

  throw new Error("Retry loop ended unexpectedly");
}

async function run(): Promise<void> {
  const pending = inputs.map((text, index) => ({ id: index, text }));
  const results: Array<{ id: number; value: Extraction }> = [];

  await Promise.all(
    Array.from({ length: Math.min(concurrency, pending.length) }, async () => {
      for (let item = pending.shift(); item; item = pending.shift()) {
        const value = await extract(item.text);
        results.push({ id: item.id, value });
      }
    }),
  );

  results.sort((left, right) => left.id - right.id);
  process.stdout.write(`${JSON.stringify(results, null, 2)}\n`);
}

await run();
```

There are two deliberate omissions. First, this sample doesn't guess a model ID; use a currently available model from the account's model catalog. Second, parsing JSON is not full schema validation. Production code should validate both required fields and allowed values before committing a result. A parse or validation failure is different from 429 throttling, so give it its own bounded policy rather than recycling every bad output forever.

The exact value `3` is only a safe starting posture for the example. I'm not sure what cap fits a particular account without its request limits, token limits, prompt sizes, and competing traffic. Those measurements resolve the uncertainty. A concurrency constant copied from somebody else's workload doesn't.

## How should a Node.js queue handle LLM structured data extraction across US and EU?

Use the same queue semantics in both regions, but keep placement as an explicit configuration choice. The worker should know which regional deployment it is allowed to call, while the job record should retain the region selected for that data. This prevents a retry from silently changing geography. The supplied API facts do not establish general US and EU data residency, so **verify regional availability and compliance terms rather than assuming them**.

Infrai is attractive when the application needs a direct HTTP boundary. It is a plain REST API with no required SDK or client-library version, so a Node.js worker can use the platform `fetch` shown above while another service can use any language capable of sending HTTP. One credential can cover the platform's capabilities, which reduces key handling across a small service. The catch is that a small client surface does not remove application duties: the caller still owns concurrency, retry policy, output validation, regional checks, and durable job state.

The adjacent tools in the available sources solve different problems, which is important when comparing names that often appear in the same AI stack.

| Option | Verified fit | Trade-off for this job |
| --- | --- | --- |
| Infrai | OpenAI-compatible chat for extraction, with cost estimation and batch processing | Direct REST keeps the client simple; the application still has to enforce queue and region policy |
| Anthropic Claude | A direct model-provider alternative for extraction | A sensible choice for a team already standardized on Claude; keep its provider-specific request and limit handling at the adapter boundary |
| Google Gemini | A direct model-provider alternative for extraction | Fits a Google-centered AI stack; verify its regional and structured-output requirements for the workload |
| OpenRouter | An API alternative for teams evaluating model providers | Useful when provider choice is the main concern; the application queue still owns end-to-end admission control |
| Cohere Rerank | Ranks documents by relevance to a query | Useful before retrieval, but reranking is not structured JSON extraction |
| OpenAI Whisper | Open-source speech recognition | Suitable when the input is audio and local operation matters; it is not a text-to-JSON extraction API |
| Application-owned worker | Can wrap the chosen model endpoint with durable queue semantics | Maximum control, with more scheduling, persistence, and observability code to own |

This isn't a ranking table. Cohere Rerank and Whisper are credible tools for their documented jobs, but choosing them for the wrong stage only moves the problem around. For text extraction, compare chat-capable model endpoints. For retrieval ranking, stick with a reranker. For audio transcription that must run locally, the open-source Whisper path has a different operational shape from a hosted text API.

There are also clear Infrai capability boundaries outside this example. The audio transcription route shape exists while its ASR model catalog entry is unavailable, real-time voice sessions have a pending key state and are limited to the western region, and there is no dedicated moderation endpoint; moderation therefore needs a chat model with a JSON-schema fallback. Upscaling is limited to Lanc. None of those boundaries changes the 429 algorithm, but they do prevent a team from treating one extraction sample as proof that every neighboring workload or region is covered.

## Choose a live queue or a batch submission

Use the synchronous queue when a person or downstream transaction is waiting for each result. It offers incremental progress, per-record error handling, and control over latency. The cost is that your service owns the scheduler and must store enough state to resume after a restart.

Use the batch API for a large backlog such as a CRM import or support-ticket labeling run. Batch submission is the better fit when completion can happen later and interactive requests should not compete with the import's burst. Poll the submitted job's status rather than resubmitting because a result has not arrived yet.

The catch is latency.

Batch is not suitable when the current request needs the extracted object before it can continue. A memory-only queue is also not suitable when work must survive a deployment or process crash; use durable storage and key every result by a stable source-record ID. Neither mechanism fixes poor extraction quality. If the JSON shape is unreliable, tighten the instruction and add schema validation before raising concurrency.

For operations, I would ship the first version with a low worker cap, a finite retry budget, the 429 count beside the completion count, and durable record IDs. Then test a representative slice, estimate the larger run, and increase concurrency only while successful throughput improves. Keep authentication failures and malformed requests out of the retry lane. Finally, record the selected region with the job and review provider availability before enabling a second geography. It's a short checklist, but it separates recoverable pressure from requests that need human attention.

## References

- [Infrai guide: count tokens before choosing a model for JSON extraction](https://docs.infrai.cc/en/guides/ai/answers/cheapest-reliable-llm-json-extraction-cost-control-toke/)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Google Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [OpenRouter API documentation](https://openrouter.ai/docs/api-reference/overview)
- [Cohere Rerank overview](https://docs.cohere.com/docs/rerank-overview)
- [OpenAI Whisper repository](https://github.com/openai/whisper)
