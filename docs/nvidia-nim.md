
# Wyrm with NVIDIA NIM

Wyrm's default embeddings run locally and keep everything on your machine. When
retrieval accuracy is worth a hosted call, Wyrm can use NVIDIA NIM for both the
embedding and the reranking legs. This is an explicit opt-in, off by default.

## What it adds

- **Retrieval embeddings** from `nvidia/nemotron-3-embed-1b` (2048
  dimensions), with the correct query-versus-passage input typing that these
  models expect.
- **Reranking** from `nvidia/llama-nemotron-rerank-vl-1b-v2`, applied after fusion
  to reorder candidates by relevance.

## The measured lift

Measured in September 2026 on the LoCoMo benchmark committed in the repo
(`bench/nim-retrieval.mjs`, 2 conversations, 301 questions, k=10):

| Configuration | recall@1 | recall@10 | MRR |
|---|---|---|---|
| NIM embeddings (`nvidia/nemotron-3-embed-1b`) | 39.9% | 76.7% | 0.507 |
| NIM embeddings plus NIM rerank (`nvidia/llama-nemotron-rerank-vl-1b-v2`) | 55.5% | 80.1% | 0.630 |

Reranking cost about one extra second per query in that run. The local baseline
was not part of this measurement, so this table makes no claim about the gap
between local and NIM; the older figures that did were measured on models NVIDIA
has since retired, and have been withdrawn. Run the script on your own data: your
corpus will differ.

## Enable it

NIM is OpenAI-compatible, so it uses a standard key and base URL.

```bash
export WYRM_VECTOR_PROVIDER=nim          # select NIM for embeddings
export WYRM_RERANK_PROVIDER=nim          # select NIM for reranking (optional, additive)
export NVIDIA_API_KEY=nvapi-...          # your NVIDIA API key (WYRM_NIM_API_KEY also works)
```

Model and endpoint have working defaults (`nvidia/nemotron-3-embed-1b`, the
NVIDIA hosted base URL); override with `WYRM_NIM_EMBED_MODEL`, `WYRM_NIM_BASE_URL`
and `WYRM_RERANK_MODEL` if you run NIM elsewhere, including a self-hosted NIM on
your own network. Reranking runs only when `WYRM_RERANK=1` is also set.

Keep the key out of plaintext. Store it in Wyrm's vault and inject it only for
the process that needs it:

```bash
wyrm vault put nim-api-key
wyrm vault exec nim-api-key --as NVIDIA_API_KEY -- wyrm-mcp
```

## The dimension rule and reindex

Local `nomic-embed-text` is 768 dimensions; NIM is 2048. A vector store holds
one dimension. Wyrm's cosine similarity is dimension-strict: comparing vectors
of different lengths returns zero, not a wrong-but-plausible score. That is a
guardrail, not a bug, and it means switching providers on an existing store
makes old vectors unrecallable until you re-embed.

So when you turn NIM on for a database that already has memories, reindex:

```bash
# after setting WYRM_VECTOR_PROVIDER=nim
wyrm index rebuild
```

On a fresh store there is nothing to reindex; new writes embed at 2048
dimensions from the start.

The same applies when NVIDIA retires a model. The hosted
`nvidia/llama-nemotron-embed-1b-v2` and `nvidia/llama-nemotron-rerank-1b-v2`
reached end of life on 25 August 2026 and now answer HTTP 410. A vector made with
a retired model can never be compared with a new model's, so upgrading Wyrm is
not enough on its own: run `wyrm index rebuild` afterwards. `wyrm doctor` reports
a retired model, a run of failing embeds, or indexing that has stopped.

## Egress: what leaves your machine, and when

This is the contract, stated plainly.

- **Default (no NIM):** nothing leaves your machine. Embedding and reranking
  are local or FTS-only.
- **NIM enabled:** the text being embedded or reranked is sent to the NIM
  endpoint (`integrate.api.nvidia.com` by default, or your own base URL). That
  is the memory content for writes and the query for recall.

Wyrm does not hide this. The health endpoint and the recall determinism receipt
report the active provider, the embedding dimension, and the egress host, so an
operator can see from the runtime, not from the config file, exactly what path a
given recall took. If the receipt says the egress host is `none`, nothing left
the machine on that call.

## When it is worth it

- **Worth it:** accuracy-critical retrieval, larger corpora where the recall@1
  gap matters, or a fleet where one hosted call per write is acceptable.
- **Probably not:** small local stores, air-gapped or privacy-strict setups, or
  anywhere the local baseline already returns the right memory. The default
  exists because it is the right choice more often than not.

Turn it on deliberately, reindex, and confirm the egress line in the receipt
matches what you intend.
