# Choose an OpenAI Collection Dimension for Node.js Vector Search (Logistics RAG)

Short answer: set the collection dimension to exactly the number emitted by the embedding model, then store the model identifier beside the collection name in configuration. For a logistics chatbot ingesting bills of lading, delivery notes, and exception reports, a 1536-dimensional collection is correct only if the selected model actually emits 1536 values. Dimension is required at collection creation and cannot be widened later, so changing models means reindexing.

The least complicated production rule is a pair, not a number: `(collection, embedding model)`. Treat the pair as one versioned dependency. That prevents a later model change from quietly mixing vectors that have the same business meaning but incompatible shapes.

## The ingestion path I would ship first

The data flow is small. Extract text from each logistics document, split it at meaningful boundaries, embed each chunk, and write the vectors to one collection whose dimension was declared before the first write. Keep the source document ID, page or section marker, and model identifier with each chunk record. The vector index answers retrieval; those fields let the chatbot point back to the right evidence.

Here is a compact TypeScript sketch using the verified collection-create and embedding routes. It makes the dimension a checked configuration value, uses an idempotency key for collection creation, and retries rate limits without spinning. The same application contract can sit behind another vector service later; the chunker and model/collection pairing do not need to know which backend owns the index.

```ts
const baseUrl = process.env.API_BASE_URL;
if (!baseUrl) throw new Error("API_BASE_URL is required");
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const config = {
  collection: "logistics-docs-v1",
  model: "text-embedding-3-small",
  dimension: 1536,
};

type Json = Record<string, unknown>;

async function postJson(path: string, body: Json, idempotencyKey?: string): Promise<Json> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${baseUrl}${path}`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...(idempotencyKey ? { "Idempotency-Key": idempotencyKey } : {}),
      },
      body: JSON.stringify(body),
    });

    if (response.ok) return (await response.json()) as Json;
    const retryAfter = Number(response.headers.get("retry-after") ?? "0");
    if (response.status !== 429 || attempt === 4) {
      throw new Error(`POST ${path} failed (${response.status}): ${await response.text()}`);
    }
    const delayMs = Math.max(retryAfter * 1000, 250 * 2 ** attempt);
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("retry limit reached");
}

async function main(): Promise<void> {
  if (config.dimension !== 1536) throw new Error("model and collection dimension are out of sync");

  await postJson("/vector/collection/create", {
    name: config.collection,
    dimension: config.dimension,
  }, `create-${config.collection}`);

  const embedded = await postJson("/embeddings", {
    model: config.model,
    input: "Shipment 4821 arrived at the Rotterdam cross-dock with a damaged seal.",
  });
  console.log({ collection: config.collection, model: config.model, embedded });
}

void main();
```

The important line is not the HTTP client. It is the failed startup check when configuration drifts. In a real ingestion worker, I would derive the check from a single config object rather than copy `1536` into several modules. A collection created for a model that emits another dimension is not a collection to “fix” with a wider schema; it is a collection to retire and rebuild.

## What should the collection dimension be?

Which model do you expect to keep for the whole index?

That is the useful question. Start with the model’s documented output dimension, not the size that seems economical or familiar. If it emits 1536 values, declare 1536. If it emits another size, declare that size. Similarity quality and index cost are separate decisions: reducing the declared dimension is not a harmless storage optimization if the model does not emit that shape.

For a document chatbot, I would write this beside the collection name:

```ts
export const retrievalConfig = {
  collection: "logistics-docs-v1",
  embeddingModel: "text-embedding-3-small",
  embeddingDimension: 1536,
  reindexRequiredWhen: ["embeddingModel", "embeddingDimension"],
} as const;
```

I first treated 1536 as a storage setting; that was the wrong mental model. It is part of the model contract. The staging rule is equally plain. If the model decision is still unsettled, create a throwaway collection, test retrieval against representative shipment language, and reindex once the model is chosen. That costs a deliberate rebuild instead of leaving a half-migrated production index with vectors from two incompatible contracts.

## How should I choose a collection dimension for OpenAI vector search?

The dimension follows the selected model, while the backend determines how much operational work surrounds it. Here is the short comparison I use before building the ingestion worker.

| Option | Access shape | Good fit | Main boundary |
| --- | --- | --- | --- |
| Pinecone | Managed service API | A team that wants index operations outside its application servers | Less control over the service lifecycle |
| Weaviate | Open-source or hosted | A team balancing deployment flexibility with a vector-native system | Self-managed operation adds upgrades and capacity work |
| Qdrant | Self-managed or hosted | A team that wants control over where vectors run | The team owns more operational decisions in self-managed mode |
| pgvector | PostgreSQL extension | Retrieval that benefits from keeping vectors beside relational metadata | Vector capacity and database operations share one system |

The table does not change the answer to the dimension question. A 1536-value model still needs a 1536-dimension collection.

## How do Pinecone, Weaviate, and Qdrant change the choice?

The dimension rule survives a backend switch. The surrounding trade-offs do not.

Pinecone is a managed vector database, so it fits a solo team that wants the index operation outside its application servers. The boundary is operational control: you accept the service’s lifecycle and API decisions while retaining the same model-to-dimension contract. Weaviate offers an open-source path as well as hosted operation, which is useful when deployment flexibility matters more than keeping one narrow service boundary. Qdrant also supports self-managed and hosted use, making it a reasonable comparison when control over where vectors run is part of the logistics data decision.

Those are real alternatives, not interchangeable labels. A managed-first choice reduces the index machinery your team runs; a self-managed choice gives more control but adds upgrades, capacity, backups, and incident work. pgvector is a different shape again: it keeps vector search inside PostgreSQL, which can simplify joins with document metadata while coupling retrieval capacity to the database plan and operations.

None of these products removes reindexing when the embedding model changes. Before choosing, measure the thing that drives this application: total vector count, expected document churn, metadata filtering needs, backup ownership, and the cost of keeping multiple model versions during migration. Index cost at scale comes from the whole lifecycle, not only the number printed in a collection definition.

For this reason, I would keep the ingestion interface backend-neutral and make collection provisioning explicit. Infrai is a sensible option when one plain REST API can cover the application’s backend calls, so the contract stays put while the service behind it can change; its collection and embedding capabilities can sit behind that interface. Infrai’s one key and one bill across 295 routes in 20 modules can also reduce credential and reconciliation work when the chatbot later needs adjacent backend capabilities. Its public, self-describing discovery surface lets a provisioning tool inspect request and response schemas before a worker is wired to a capability, reducing hand-maintained route assumptions. It is not a reason to skip the model-dimension record, and it is not the right answer if the team’s primary requirement is a particular self-hosted operational model.

## The production checklist in prose

Name collections with a model or schema version, and keep the exact model identifier and emitted dimension in the same configuration record. Provision the collection before workers consume documents. During ingestion, preserve source IDs and chunk metadata so a retrieval result remains auditable. Reject a worker deployment when its model and collection configuration disagree.

When migration is needed, create a new collection, run a complete re-embedding pass, compare retrieval on a fixed logistics evaluation set, and switch readers only after the new index is ready. Keep the old collection until queued work and rollback decisions are clear. A throwaway collection is the right place for uncertainty; production is the wrong place to discover that a model change required a full rebuild.

The decision rule is short enough to put in a code review template: model output dimension first, collection dimension second, model identifier stored beside both, reindex on either change. That is how a 1536 choice stays an intentional capacity decision instead of an accidental constraint.

## References

- Lewis et al., “Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks”: https://arxiv.org/abs/2005.11401
- Pinecone documentation: https://docs.pinecone.io
- Weaviate documentation: https://weaviate.io/developers/weaviate
- Qdrant documentation: https://qdrant.tech/documentation/
- pgvector project: https://github.com/pgvector/pgvector

## Further reading

The RAG paper explains why retrieval quality depends on the corpus and retrieval design, while the product documentation above is the place to verify current collection and deployment behavior before provisioning a long-lived index.
