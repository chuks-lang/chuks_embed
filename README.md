# chuks_embed

High-throughput embedding engine with true parallel processing. Built on Chuks native `spawn` + channels for real CPU parallelism, on a 16-core machine, 16 API calls happen simultaneously.

## Providers

| Provider   | Models                                                                  | Auto-Detected By         |
| ---------- | ----------------------------------------------------------------------- | ------------------------ |
| **OpenAI** | text-embedding-3-small, text-embedding-3-large, text-embedding-ada-002  | `text-embedding-` prefix |
| **Cohere** | embed-english-v3.0, embed-multilingual-v3.0, embed-english-light-v3.0   | `embed-` prefix          |
| **Ollama** | nomic-embed-text, mxbai-embed-large, all-minilm, snowflake-arctic-embed | Default (local)          |

The provider is **auto-resolved from the model name** — no configuration needed.

## Installation

Add to your `chuks.json`:

```bash
chuks add @chuks/embed
```

## Quick Start

```chuks
import { embed } from "pkg/@chuks/embed"

// Embed a single text
var vec = await embed.single("Hello world", { model: "text-embedding-3-small" })

// Embed a batch
var result = await embed.batch(["cat", "dog", "fish"], {
    model: "text-embedding-3-small"
})
println(result.dimensions)   // 1536
println(result.totalTokens)  // token usage

// Compare meaning
var score = embed.similarity(result.vectors[0], result.vectors[1])
println(score)  // 0.85 — cat and dog are similar
```

## Parallel Batch Embedding

The killer feature. Process massive datasets by splitting into batches and running `workers` API calls simultaneously:

```chuks
import { embed } from "pkg/@chuks/embed"

var documents: []string = loadDocuments()  // 100,000 documents

var result = await embed.batchParallel(documents, {
    model:     "text-embedding-3-small",
    batchSize: 100,     // 100 texts per API call
    workers:   16,      // 16 concurrent API calls
})

println(result.vectors.length())  // 100000
println(result.batchCount)        // 1000
println(result.duration)          // milliseconds elapsed
println(result.totalTokens)       // total tokens used
```

### How It Works

```
100,000 documents ÷ 100 per batch = 1,000 batches
1,000 batches ÷ 16 workers = 63 rounds of 16 parallel API calls

Each round:
  → spawn 16 goroutines (real OS threads)
  → 16 API calls fire simultaneously
  → channel.receive collects ordered results
  → flatten into allVectors array
```

### Progress Callback

```chuks
var result = await embed.batchParallel(documents, {
    model:      "text-embedding-3-small",
    batchSize:  100,
    workers:    16,
    onProgress: function(completed: int, total: int): void {
        println("Progress: " + toString(completed) + "/" + toString(total))
    }
})
```

## API Reference

### `embed.single(text, config)` → `Task<[]any>`

Embed one text string. Returns the raw embedding vector.

### `embed.batch(texts, config)` → `Task<EmbeddingResult>`

Embed an array of texts in a single API call.

**Returns:**

- `.vectors` — array of embedding vectors
- `.model` — resolved model name
- `.dimensions` — vector dimensionality
- `.totalTokens` — tokens consumed

### `embed.batchParallel(texts, config)` → `Task<BatchResult>`

Embed texts with parallel worker processing.

**Returns:**

- `.vectors` — all embedding vectors (ordered)
- `.model` — resolved model name
- `.dimensions` — vector dimensionality
- `.totalTokens` — total tokens consumed across all batches
- `.batchCount` — number of API calls made
- `.duration` — wall-clock time in milliseconds

### `embed.similarity(a, b)` → `any`

Cosine similarity between two vectors. Returns -1 to 1 (1 = identical meaning).

### `embed.dotProduct(a, b)` → `any`

Dot product of two vectors. For pre-normalized vectors, this equals cosine similarity but is faster.

### `embed.normalize(vector)` → `any`

Normalize a vector to unit length. Pre-normalizing enables using `dotProduct` instead of `similarity`.

### `embed.dimensions(model)` → `int`

Look up known dimension count for a model. Returns 0 for unknown models. Supports prefixed names like `"openai/text-embedding-3-small"`.

### `embed.search(query, documents, config)` → `Task<[]any>`

Convenience method: embeds the query and all documents, returns matches ranked by similarity.

**Config extras:** `topK` (default: 10)

**Returns:** Array of `{ index, score }` sorted by score descending.

## Config Options

| Option       | Type     | Default             | Description                                              |
| ------------ | -------- | ------------------- | -------------------------------------------------------- |
| `model`      | string   | _required_          | Model name (auto-resolves provider)                      |
| `provider`   | string   | auto                | Force provider: `"openai"`, `"cohere"`, `"ollama"`       |
| `apiKey`     | string   | from .env           | API key (auto-loads `OPENAI_API_KEY` / `COHERE_API_KEY`) |
| `baseUrl`    | string   | auto                | Custom API endpoint                                      |
| `batchSize`  | int      | 100                 | Texts per API call                                       |
| `workers`    | int      | 8                   | Concurrent API calls for batchParallel                   |
| `dimensions` | int      | null                | Override output dimensions (OpenAI only)                 |
| `inputType`  | string   | `"search_document"` | Cohere input type                                        |
| `onProgress` | function | null                | Progress callback `(completed, total)`                   |
| `topK`       | int      | 10                  | Results to return from search                            |

## Provider Prefix Syntax

Force a specific provider by prefixing the model name:

```chuks
// Explicit OpenAI
{ model: "openai/text-embedding-3-small" }

// Explicit Cohere
{ model: "cohere/embed-english-v3.0" }

// Explicit Ollama (local)
{ model: "ollama/nomic-embed-text" }
```

## API Key Resolution

Keys are resolved in order:

1. `apiKey` in config object
2. `.env` file via `dotenv.load()`:
   - OpenAI: `OPENAI_API_KEY`
   - Cohere: `COHERE_API_KEY` or `CO_API_KEY`
   - Ollama: no key needed

## Vector Math

```chuks
import { embed } from "pkg/@chuks/embed"

var a = await embed.single("king", { model: "text-embedding-3-small" })
var b = await embed.single("queen", { model: "text-embedding-3-small" })
var c = await embed.single("bicycle", { model: "text-embedding-3-small" })

// Cosine similarity
println(embed.similarity(a, b))  // ~0.85 (semantically close)
println(embed.similarity(a, c))  // ~0.25 (semantically distant)

// Pre-normalize for faster repeated comparisons
var normA = embed.normalize(a)
var normB = embed.normalize(b)
println(embed.dotProduct(normA, normB))  // same as similarity but faster

// Dimension lookup
println(embed.dimensions("text-embedding-3-small"))   // 1536
println(embed.dimensions("nomic-embed-text"))          // 768
```

## Semantic Search

```chuks
import { embed } from "pkg/@chuks/embed"

var docs: []string = [
    "The cat sat on the mat",
    "Dogs are loyal companions",
    "Python is a programming language",
    "Machine learning transforms data",
    "Cats and dogs are popular pets"
]

var matches = await embed.search("animals as pets", docs, {
    model: "text-embedding-3-small",
    topK: 3
})

var i: int = 0
while (i < matches.length()) {
    println(toString(matches[i].index) + ": " + docs[matches[i].index] + " (score: " + toString(matches[i].score) + ")")
    i = i + 1
}
// Output:
// 4: Cats and dogs are popular pets (score: 0.91)
// 1: Dogs are loyal companions (score: 0.82)
// 0: The cat sat on the mat (score: 0.74)
```

## Practical Recommendations

| Use Case              | Model                   | Dimensions | Speed          | Cost            |
| --------------------- | ----------------------- | ---------- | -------------- | --------------- |
| **Production RAG**    | text-embedding-3-small  | 1536       | Fast           | $0.02/1M tokens |
| **Maximum quality**   | text-embedding-3-large  | 3072       | Medium         | $0.13/1M tokens |
| **Multilingual**      | embed-multilingual-v3.0 | 1024       | Fast           | Cohere pricing  |
| **Local/Free**        | nomic-embed-text        | 768        | Depends on GPU | Free            |
| **Lightweight local** | all-minilm              | 384        | Very fast      | Free            |
