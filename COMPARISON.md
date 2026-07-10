# Wyrm vs. the field — an honest comparison

> Companion to [`BENCHMARKS.md`](./BENCHMARKS.md). Last verified **2026-06-27** via a
> multi-source, adversarially-verified literature pass against primary sources.
> Every competitor number here is **vendor-self-reported** and cited to its paper/repo.

## The one rule this document refuses to break

There are **two different evaluation axes**, and they are routinely (and incorrectly)
mixed in marketing tables:

| Axis | Question it answers | Who reports it |
|---|---|---|
| **Retrieval recall@k** | "When the question is asked, did the memory layer *surface the ground-truth supporting evidence*?" | **Wyrm** |
| **QA answer accuracy** | "Did a *separate LLM judge* rate the *generated answer* correct vs a golden answer?" | mem0, Zep, and effectively every other system below |

These are **not comparable numbers.** A retrieval recall@k of 72.2 is *not* better or
worse than a QA-accuracy of 66.9 — they measure different things. This document keeps
them in **separate tables** on purpose. Anyone placing "Wyrm 72.2" next to "Mem0 66.9"
in one ranked column is making a category error.

(Note the naming trap: Zep's headline benchmark is literally called *Deep Memory
**Retrieval*** (DMR), but its score is final-answer correctness judged by an LLM — **not**
recall@k.)

## What *is* apples-to-apples: architecture

This table compares facts that mean the same thing for every system.

| System | Open source | Runs fully local / no cloud | LLM at **write** (ingest) | LLM at **answer** | Headline metric |
|---|---|---|---|---|---|
| **Wyrm** | No (proprietary freeware) | ✅ yes | ❌ **none** (deterministic FTS/vector) | ❌ **none** (retrieval only) | retrieval recall@k |
| Mem0 | Apache-2.0 (+ managed cloud) | self-host possible | ✅ required (LLM extraction + ADD/UPDATE/DELETE/NOOP) | ✅ required | QA accuracy (LLM-judge "J") |
| Zep / Graphiti | Graphiti Apache-2.0 (Zep cloud) | self-host possible | ✅ required (LLM graph construction) | ✅ required | QA accuracy (LLM-judge) |
| SuperMemory · cognee · SuperLocalMemory | varies | varies | varies | varies | *no citable LoCoMo/LongMemEval number found* (see below) |

**The defensible differentiator:** Wyrm reaches competitive *retrieval* with **zero cloud
and zero LLM calls at both read and write.** mem0 and Zep each require an LLM at ingest
*and* at answer time. (Plus Wyrm's structural differentiators: negative learning /
failure memory, decision causality, MCP-native, deterministic — none of which these
benchmarks measure.)

## The published numbers (separated by axis — do **not** cross-rank)

### Wyrm — retrieval recall@k, real LoCoMo, local + zero-LLM

Reproduces the figures in [`BENCHMARKS.md`](./BENCHMARKS.md) (snap-research/LoCoMo, 10
convs / 1,982 evidence-bearing QA, k=10; re-run 2026-06-27, HEAD).

| Tier | recall@1 | recall@5 | recall@10 | MRR | LLM? | Cloud? |
|---|---|---|---|---|---|---|
| FTS floor (deterministic, offline) | 29.5 | 52.4 | 59.9 | .393 | none | none |
| + vectors (nomic-embed-text, local) | 28.8 | 58.7 | 69.5 | .413 | none* | none |
| hybrid (FTS ⊕ vector, RRF cand25) | 32.8 | 60.3 | 72.2 | .447 | none* | none |
| production (convex fusion, cand50 α0.7) | 32.1 | 61.6 | 72.5 | .446 | none* | none |

\* the embedding model is a local Ollama call, not a generative LLM; the FTS floor needs
no model at all and is the CI-gated, fully-deterministic number.

### Competitors — QA answer accuracy (LLM-judge), vendor-self-reported

| System | Benchmark | Metric | Score | Source |
|---|---|---|---|---|
| Mem0 (paper) | LoCoMo | LLM-judge "J" | **66.88%** (mem0-graph 68.44%) | arXiv 2504.19413 |
| ↳ same table, comparators | LoCoMo | LLM-judge "J" | Full-Context 72.90 · Zep 65.99† · LangMem 58.10 · OpenAI-memory 52.90 | arXiv 2504.19413 |
| Mem0 (managed Platform v3 — *different config from the paper*) | LoCoMo | answer acc. | 92.5% (Top-200) / 91.8% (Top-50) | github: mem0ai/memory-benchmarks |
| Mem0 (managed Platform v3) | LongMemEval | answer acc. | 94.4% (Top-200) / 94.8% (Top-50) | github: mem0ai/memory-benchmarks |
| Zep (paper) | LongMemEval | answer acc. (GPT-4o judge) | 71.2% (gpt-4o) / 63.8% (gpt-4o-mini) vs baseline 60.2 / 55.4 | arXiv 2501.13956 |
| Zep (paper) | DMR | answer acc. | 94.8% (gpt-4-turbo) / 98.2% (gpt-4o-mini) vs MemGPT 93.4% | arXiv 2501.13956 |

## Footnotes that matter (read before quoting any number)

- **† The "Zep LoCoMo 65.99%" figure is not Zep's.** Zep's own paper reports **no LoCoMo
  numbers at all** — only DMR and LongMemEval. The 65.99% comes from *Mem0's* table
  (Mem0 scoring Zep). A separate third-party correction (getzep/zep-papers issue #5)
  recalculated a circulating "Zep 84% LoCoMo" claim down to ~58.44% once the otherwise-
  excluded adversarial category is included.
- **Mem0's paper (~67%) and Mem0's repo (~92%) disagree** because they measure different
  systems/configs (the ECAI paper's OSS pipeline vs. the managed Platform v3). Both are
  answer-accuracy, both vendor-self-reported.
- **Everything in the competitor table is vendor-self-reported** — the paper authors carry
  `@mem0.ai` / `@getzep.com` affiliations and the benchmark repos are the vendors' own.
  No independent third-party reproduction was found.
- **LoCoMo is a weak benchmark.** The QA-eval set is only **10 conversations**; the
  **adversarial category is excluded** because ground-truth answers are unavailable; and
  the dialogues are **synthetically LLM-generated** with documented gold-answer quality
  flaws. Treat all LoCoMo numbers — ours included — with that caution.
- **Zep itself calls its DMR gains "marginal"** over both MemGPT and a full-context
  baseline (94.8 vs 94.4), and notes DMR's 60-message conversations fit in a modern
  context window, making DMR a weak test of memory.

## What we looked for and did **not** find

No citable, primary-source LoCoMo or LongMemEval number survived verification for
**SuperMemory**, **cognee**, **Letta/MemGPT** (it appears only as a *baseline* inside
Zep's DMR table at 93.4%), or **SuperLocalMemory**. Figures circulating for these (e.g.
a "SuperMemory 81.6% LongMemEval" or "SuperLocalMemory 74.8% LoCoMo") could not be tied
to a verifiable primary source and are deliberately omitted rather than repeated.

## The honest takeaway

- You **cannot** say "Wyrm beats mem0/Zep" — they report a different metric on a
  partly-different benchmark, with an LLM in the loop that Wyrm doesn't use.
- What is **true and rare**: Wyrm delivers competitive *retrieval* (recall@10 72.2 on
  real LoCoMo) **with no cloud and no LLM at read or write**, where mem0 and Zep both
  require an LLM at ingest and answer time. That — plus negative learning, decision
  causality, and MCP-native local-first operation — is the moat. Not a bigger number.

## Sources

- Mem0 — "Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory",
  arXiv:2504.19413 (ECAI 2025). Benchmark repo: github.com/mem0ai/memory-benchmarks.
- Zep — "Zep: A Temporal Knowledge Graph Architecture for Agent Memory",
  arXiv:2501.13956 (Jan 2025). Blog: blog.getzep.com.
- LoCoMo — snap-research/locomo (the dataset Wyrm's retrieval numbers are measured on).
- Wyrm retrieval numbers + methodology — [`BENCHMARKS.md`](./BENCHMARKS.md); reproduce with
  `node bench/locomo-real.mjs` (FTS floor) and `node bench/locomo-hybrid.mjs` (hybrid).
