
# Wyrm with NVIDIA NIM

Wyrm's default embeddings run locally and keep everything on your machine. When
retrieval accuracy is worth a hosted call, Wyrm can use NVIDIA NIM for both the
embedding and the reranking legs. This is an explicit opt-in, off by default.

## What it adds

- **Retrieval embeddings** from `nvidia/llama-nemotron-embed-1b-v2` (2048
  dimensions), with the correct query-versus-passage input typing that these
  models expect.
- **Reranking** from `nvidia/llama-nemotron-rerank-1b-v2`, applied after fusion
  to reorder candidates by relevance.

## The measured lift

On a LoCoMo-style retrieval benchmark committed in the repo (`bench/`), moving
from the local baseline to NIM improved recall@1 as follows:

| Configuration | recall@1 |
|---|---|
| Local (`nomic-embed-text`) | 33% |
| NIM embeddings | 47% |
| NIM embeddings plus NIM rerank | 52% |

These are numbers from the committed benchmark, reproducible on your own data
by running the script. They are a guide, not a promise; your corpus will differ.

## Enable it

NIM is OpenAI-compatible, so it uses a standard key and base URL.

```bash
export WYRM_VECTOR_PROVIDER=nim          # select NIM for embeddings
export WYRM_RERANK_PROVIDER=nim          # select NIM for reranking (optional, additive)
export NIM_API_KEY=nvapi-...             # your NVIDIA API key
```

Model and endpoint have working defaults (`nvidia/llama-nemotron-embed-1b-v2`,
the NVIDIA hosted base URL); override with `WYRM_EMBED_MODEL`, `NIM_BASE_URL`,
and `WYRM_RERANK_MODEL` if you run NIM elsewhere, including a self-hosted NIM on
your own network.

Keep the key out of plaintext. Store it in Wyrm's vault and inject it only for
the process that needs it:

```bash
wyrm vault put nim-api-key
wyrm vault exec nim-api-key --as NIM_API_KEY -- wyrm-mcp
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
wyrm_reindex
```

On a fresh store there is nothing to reindex; new writes embed at 2048
dimensions from the start.

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
