# Wyrm — Retrieval Benchmarks

Wyrm competes on an axis the cloud memory layers can't stand on: **the number is produced with zero cloud calls and zero generative-LLM calls, and anyone can reproduce it offline.** mem0 / Zep / Supermemory headline end-to-end QA accuracy that depends on an LLM extracting and answering; the number below is the **retrieval floor underneath that** — *did the memory actually return a ground-truth supporting turn?* — measured deterministically.

> **Comparing to mem0 / Zep / others?** See **[COMPARISON.md](./COMPARISON.md)** — a cited, honest side-by-side that keeps retrieval-recall and QA-answer-accuracy in *separate* columns (they measure different things, and every competitor number is vendor-self-reported).

## Metric

Real **LoCoMo** (*Evaluating Very Long-Term Conversational Memory*, snap-research/locomo — 10 multi-session conversations, 1,986 QA). Every dialogue turn is ingested tagged with its `dia_id`; for each question that ships ground-truth `evidence` turns, Wyrm recalls over the question and we score whether a supporting `dia_id` comes back in the top-_k_.

> **Metric = "did Wyrm retrieve a ground-truth supporting turn?"** — pure retrieval, no LLM in the loop. End-to-end answer accuracy sits *on top* of this floor; this is the engine-only number, and it is the part Wyrm makes deterministic.

Questions with no evidence turns (LoCoMo categories 3 & 5 — adversarial / world-knowledge) have nothing to retrieve and are excluded from the retrieval headline.

## Results — real LoCoMo (1,982 evidence QA, k=10)

First verified on **wyrm-mcp 7.3.0**; rows added since then self-document their own version and measurement date (see the bundled-tier row below, 8.7.0).

| Tier | recall@1 | recall@5 | recall@10 | MRR@10 | Properties |
|------|:--------:|:--------:|:---------:|:------:|------------|
| **no-LLM FTS floor** | 29.5% | **52.4%** | **59.9%** | 0.393 | offline · zero-cloud · zero-LLM · CI-gated · reproducible anywhere |
| vector-only (local) | 28.8% | 58.7% | 69.5% | 0.413 | local Ollama embeddings only |
| **local hybrid** (FTS ⊕ vectors) | 32.8% | **60.3%** | **72.2%** | 0.447 | + local Ollama `nomic-embed-text` — still zero-cloud, advisory (varies by machine) |
| bundled ONNX int8 (default install) | 32.5% | 60.0% | 72.0% | 0.443 | + bundled `nomic-embed-text-v1.5-int8` (no Ollama, no setup) — within 0.3/0.2 pts of the Ollama hybrid row above; zero-config default as of 8.7.0. Bundled runs in its own vector scope; cross-provider similarity vs Ollama measured mean 0.868 (informational; spaces isolated by model id). |

The lift is monotonic and honest: **recall@10 climbs 59.9% → 69.5% → 72.2%** as you add local vectors on top of the lexical floor. The **FTS floor is the claim that matters**: it needs no model server, no API key, no network — clone the repo and you get the same numbers. The hybrid tier shows the lift from local dense vectors (the embedding model runs on *your* box; nothing leaves it). Wyrm's production `recall` uses a tuned convex score-blend (cand=50, α=0.7) that nudges hybrid recall@5 to ~62%; the table above reports the simpler RRF fusion the standalone harness ships, so the numbers match the reproduction command exactly.

The bundled row swaps only the embedder (int8 ONNX in-process, no Ollama) into the exact same harness — same RRF fusion, same cand=25, same evidence-based scoring — measured with `npm run eval:bundled` (`bench/locomo-hybrid.mjs --provider bundled`) on 2026-08-14: recall lands within the ship gate (±1.5 pts of 60.3/72.2). It does this from its own vector scope, not a shared one: bundled ONNX (int8) and Ollama's GGUF conversion of nomic-embed-text are independently-built, each individually sane, but measured mean cosine 0.868 apart (`bench/parity-bundled.mjs` / `npm run bench:parity`, informational, not gated) — too far apart to mix under one model id without silently degrading recall, which is why the bundled provider is model-scoped as `nomic-embed-text-v1.5-int8` rather than sharing Ollama's `nomic-embed-text` id. Retrieval recall still lands on target because it depends on relative ranking within each backend's own space, not on the two backends occupying the identical vector space.

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

No public memory benchmark measures this. `bench/negative-learning.mjs` asks: when an agent is about to **repeat** a recorded mistake, does `wyrm_failure_check` block it, and stay silent on novel actions? Deterministic, no LLM, no cloud. The figures are those of the committed receipt (`bench/firewall-receipt.json`, regenerated on wyrm-mcp 9.2.1, 2026-09-14):

| probe | result |
|---|---|
| same action re-attempted (the guard's real input) | **recall 100%** (16/16) |
| reworded restatement | **recall 100%** (16/16) |
| novel actions | **precision 100%** · specificity 100% (0 false blocks of 16) |
| confidence separation | 0.86 |
| staleness oracle | passes |
| `checkVerdict` latency, in-process (advisory) | p50 0.29 ms · p95 0.58 ms |

What the latency row is. The bench times `FailurePatterns.checkVerdict()` called in-process with `process.hrtime`: 48 calls in one process (16 same-action, 16 reworded, 16 novel) against a temporary store that is already open and holds the 16 recorded failures. No warm-up pass is discarded. It is the cost of the check itself. It is not what a hook pays per tool call: a fresh `wyrm-guard` process also pays Node start-up and the store open, and `bench/guard-budget.mjs` measures that separately (median 62 to 66 ms over 30 spawns per path against a 202-row store, same machine, 2026-09-21).

Both are measurements on one machine on one date, not guarantees. The committed figure and the re-runs come from the same machine, a 12th-gen Intel Core i5 laptop (i5-12500H, 16 threads, Linux). Five re-runs on 2026-09-21 (wyrm-mcp 9.2.3, Node 22.23, 1-minute load near 3) gave a median between 0.33 and 0.60 ms and a 95th percentile between 0.67 and 1.47 ms. Latency is advisory: no floor gates it, and it is left out of the receipt's `content_sha256` so that a re-run on the same code reproduces the same hash. `scripts/check-claims.mjs` fails the build if the row above stops matching the committed receipt.

Correction, 2026-09-21: until this date the row read a median of 2.7 ms and a 95th percentile of 6.1 ms. That was a single 9.1.2 run on 2026-09-12, and the row was not refreshed when the receipt was regenerated two days later. The 9.1.2 source re-measured on 2026-09-21 gave a median of 0.34 to 0.38 ms, so the old figure described the machine during that one run, not the engine.

The reworded tier used to sit at 12.5% (8.5.3), because the fuzzy probe was a conjunctive-AND FTS kept precision-first by design. Target normalization (canonical command matching, flag order and `=` form folded, positionals kept in order) closed that gap without giving up a single false block. Run: `node bench/negative-learning.mjs`.

The end-to-end version of the same question, a blind agent working real tickets with and without the firewall in front of its actions, is the agent A/B: repeated-failure rate 38.9% (14/36) without the firewall to 8.3% (3/36) with it, 11 blocks, relative reduction 78.6% (bootstrap 95% CI 45.5% to 100%, n=36 per arm, 3 repetitions). Method, per-scenario table and the stated limits: `docs/firewall-receipt.md`.

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
