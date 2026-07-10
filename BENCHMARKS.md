# Wyrm — Retrieval Benchmarks

Wyrm competes on an axis the cloud memory layers can't stand on: **the number is produced with zero cloud calls and zero generative-LLM calls, and anyone can reproduce it offline.** mem0 / Zep / Supermemory headline end-to-end QA accuracy that depends on an LLM extracting and answering; the number below is the **retrieval floor underneath that** — *did the memory actually return a ground-truth supporting turn?* — measured deterministically.

> **Comparing to mem0 / Zep / others?** See **[COMPARISON.md](./COMPARISON.md)** — a cited, honest side-by-side that keeps retrieval-recall and QA-answer-accuracy in *separate* columns (they measure different things, and every competitor number is vendor-self-reported).

## Metric

Real **LoCoMo** (*Evaluating Very Long-Term Conversational Memory*, snap-research/locomo — 10 multi-session conversations, 1,986 QA). Every dialogue turn is ingested tagged with its `dia_id`; for each question that ships ground-truth `evidence` turns, Wyrm recalls over the question and we score whether a supporting `dia_id` comes back in the top-_k_.

> **Metric = "did Wyrm retrieve a ground-truth supporting turn?"** — pure retrieval, no LLM in the loop. End-to-end answer accuracy sits *on top* of this floor; this is the engine-only number, and it is the part Wyrm makes deterministic.

Questions with no evidence turns (LoCoMo categories 3 & 5 — adversarial / world-knowledge) have nothing to retrieve and are excluded from the retrieval headline.

## Results — real LoCoMo (1,982 evidence QA, k=10)

Verified on **wyrm-mcp 7.3.0**.

| Tier | recall@1 | recall@5 | recall@10 | MRR@10 | Properties |
|------|:--------:|:--------:|:---------:|:------:|------------|
| **no-LLM FTS floor** | 29.5% | **52.4%** | **59.9%** | 0.393 | offline · zero-cloud · zero-LLM · CI-gated · reproducible anywhere |
| vector-only (local) | 28.8% | 58.7% | 69.5% | 0.413 | local Ollama embeddings only |
| **local hybrid** (FTS ⊕ vectors) | 32.8% | **60.3%** | **72.2%** | 0.447 | + local Ollama `nomic-embed-text` — still zero-cloud, advisory (varies by machine) |

The lift is monotonic and honest: **recall@10 climbs 59.9% → 69.5% → 72.2%** as you add local vectors on top of the lexical floor. The **FTS floor is the claim that matters**: it needs no model server, no API key, no network — clone the repo and you get the same numbers. The hybrid tier shows the lift from local dense vectors (the embedding model runs on *your* box; nothing leaves it). Wyrm's production `recall` uses a tuned convex score-blend (cand=50, α=0.7) that nudges hybrid recall@5 to ~62%; the table above reports the simpler RRF fusion the standalone harness ships, so the numbers match the reproduction command exactly.

### no-LLM FTS floor — by category (evidence-bearing only)

| Category | n | recall@1 | recall@5 | recall@10 | MRR@10 |
|----------|--:|:--------:|:--------:|:---------:|:------:|
| single-hop  | 841 | 35.1% | 56.8% | 64.1% | 0.447 |
| temporal    | 321 | 33.0% | 59.2% | 66.7% | 0.443 |
| adversarial | 446 | 29.4% | 55.2% | 61.2% | 0.399 |
| multi-hop   | 282 | 14.2% | 34.8% | 45.4% | 0.232 |
| open-domain |  92 | 13.0% | 28.3% | 37.0% | 0.201 |

## Reproduce it

```sh
cd packages/mcp-server
npm run build                                   # ungated dev build

# 1. The deterministic floor — no Ollama, no network, anyone can run this:
curl -sL -o bench/data/locomo10.json \
  https://raw.githubusercontent.com/snap-research/locomo/main/data/locomo10.json
node bench/locomo-real.mjs bench/data/locomo10.json --k 10

# 2. The local-hybrid lift (needs a local Ollama serving nomic-embed-text):
ollama pull nomic-embed-text
node bench/locomo-hybrid.mjs bench/data/locomo10.json --k 10
```

The committed `bench/longmemeval-fixture.json` ships a small self-contained corpus so `node bench/longmemeval.mjs` runs with **no download at all** — that fixture's no-LLM floor is the CI gate that keeps these numbers from regressing.

## Beyond the retrieval floor — two more benchmarks

Retrieval recall is the floor. Two more, all reproducible, all local (added 2026-06-27).

### Negative learning — the firewall benchmark (the race we own)

No public memory benchmark measures this. `bench/negative-learning.mjs` asks: when an agent is about to **repeat** a recorded mistake, does `wyrm_failure_check` block it — and stay silent on novel actions? Deterministic, no LLM, no cloud.

| probe | result |
|---|---|
| same action re-attempted (the guard's real input) | **recall 100%** (16/16) |
| novel actions | **precision 100%** · specificity 100% (0 false blocks of 16) |
| reworded restatement | recall 12.5% — conjunctive-AND FTS is precision-first *by design* |
| `checkVerdict` latency | p50 ~0.1 ms |

Wyrm blocks literal repeats deterministically with zero false alarms. The reworded tier is intentionally conservative; a stopword filter on the fuzzy probe (so incidental filler words can't break the conjunctive-AND) recovered the filler-broken matches — **6.3% → 12.5% with precision held at 100%** (`tests/firewall-recall.test.ts` guards it). Going higher needs OR/threshold matching that would trade away that precision, so the firewall stays precision-first. Run: `node bench/negative-learning.mjs`.

### Embedding-model sweep

The hybrid number uses local `nomic-embed-text`. Swapping the embedder (same harness, `--model`):

| embedder | hybrid r@5 | hybrid r@10 | MRR | multi-hop r@5 | note |
|---|:--:|:--:|:--:|:--:|---|
| `nomic-embed-text` (default, 274 MB) | 60.3% | 72.2% | .447 | 49.3% | the shipped default |
| `mxbai-embed-large` (669 MB) | **61.1%** | **72.9%** | **.453** | **55.3%** | small net lift + notable multi-hop gain; 2.4× size |
| `bge-m3` | — | — | — | — | errored under Ollama (NaN embeddings) — not benchmarkable here |

`mxbai-embed-large` is a viable switchable upgrade (`--model mxbai-embed-large`), especially for multi-hop; nomic stays the lightweight default.

### Cross-encoder reranker (opt-in) — measured: the biggest top-rank lift here

`WYRM_RERANK_MODEL` engages `src/rerank.ts` (bge-reranker-v2-m3 over a local llama.cpp `/v1/rerank` endpoint) as a final, opt-in leg of `recallHybrid`. Measured on real LoCoMo (`bench/locomo-rerank.mjs`, 755 evidence QA / 4 convs, Q8_0, every query reranked — 0 fallbacks):

| metric | hybrid (fused) | **+ reranker** | Δ |
|---|:--:|:--:|:--:|
| recall@1 | 33.2% | **52.6%** | **+19.4** |
| recall@5 | 61.1% | **73.0%** | **+11.9** |
| recall@10 | 73.2% | **76.6%** | +3.4 |
| MRR@10 | .453 | **.613** | **+.16** |

It lifts the **weak categories** most (recall@5): multi-hop **51.4 → 67.6**, open-domain **30.0 → 43.3**, single-hop 59.8 → 74.9, temporal 76.9 → 86.2. A cross-encoder's strength is precision at the *top* — hence the large recall@1 / MRR gains.

**The tradeoff (why it's OFF by default):** a forward pass per candidate is slow — ~3.6 s/query on CPU at cand=15 (far faster on GPU). It's a quality lever for high-value retrieval, not the hot path; the deterministic hybrid stays the default. Setup + repro:

```sh
# 1. a llama.cpp server with reranking, serving bge-reranker-v2-m3 on :8088
llama-server -m bge-reranker-v2-m3-Q8_0.gguf --reranking --pooling rank --port 8088
# 2. the lift bench (local Ollama for embeddings, llama.cpp for the reranker)
WYRM_RERANK_MODEL=bge-reranker-v2-m3 node bench/locomo-rerank.mjs bench/data/locomo10.json --convs 4
```

## Why this framing, honestly

- This is **retrieval recall, not QA accuracy.** We do not claim to beat the cloud memory layers' end-to-end *answer*-accuracy scores — that's a different, LLM-in-the-loop metric on a different task (see `COMPARISON.md`; note every competitor figure is vendor-self-reported, and circulating "81.6% LongMemEval"-type numbers could not be tied to a verifiable primary source). We claim the **strongest deterministic, zero-cloud retrieval floor**, which is the thing those products don't publish because their pipelines can't run without a cloud LLM.
- The floor is **honest about its weak spots** — multi-hop and open-domain are where pure lexical retrieval struggles; the per-category table shows it rather than hiding behind an average.
- Every number here is **offline and reproducible.** No leaderboard submission, no private eval harness — the dataset is public and the harness is in this repo.
