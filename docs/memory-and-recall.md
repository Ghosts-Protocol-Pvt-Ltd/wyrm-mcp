
# Memory and recall

Good recall starts with good capture. This skill covers both sides: how Wyrm
finds things, and how to store things so they are findable.

## How recall works

`wyrm_recall` is hybrid by default. It runs two retrievals and fuses them:

- **Keyword** over an FTS5 full-text index. Exact and fast, strong on names,
  identifiers, and error strings.
- **Semantic** over a vector index. Strong on meaning when the words differ
  from what you stored.

The two result sets are combined with reciprocal-rank fusion, then reordered by
a convex score-blend reranker (weighted toward semantic agreement). You do not
configure this to get a good default; it is on out of the box.

## Which retrieval tool

- **`wyrm_recall`** is the everyday choice. Hybrid, ranked, meaning-aware. Use
  it when you want the best answer to "what do we know about X".
- **`wyrm_search`** is pure FTS5. Use it when you know the exact term and want
  literal matches without semantic drift, or when you want to page through
  results deterministically.
- **`wyrm_context_build`** assembles a context block under a token budget you
  pass. It prioritizes truths, then recent sessions, then artifacts, and
  truncates to fit. Use it when you are handing context to a model and need to
  respect a cap.

## What to store, and as what

The kind you choose decides how it comes back and how long it lives.

- A **truth** is a stable fact that stays true until something changes it. Set
  a time-to-live if it is only good for a while; Wyrm prefixes an aged truth
  with a stale marker rather than silently trusting it.
- A **lesson or artifact** is distilled: what worked, what a failure actually
  was, a pattern worth reusing. These are dense-indexed for semantic recall, so
  write them as you would want to read them later.
- A **quest** is work to track. It is not recall material; it is a task.

Capture signal, not chatter. A running commentary of a session is noise in
recall later. A decision, a resolved bug, a convention, or a reusable pattern is
signal. When in doubt, ask whether a future session would want this back.

## Embeddings

The vector index needs an embedding provider. The default chain auto-detects a
local Ollama model (`nomic-embed-text` on `localhost:11434`) and falls back to
FTS-only if none is present. Local embeddings keep everything on your machine.

If you want higher retrieval accuracy and are willing to send text to a hosted
endpoint, Wyrm can use NVIDIA NIM embeddings and reranking. That path, its
accuracy lift, its dimensions, and its egress contract are covered in
`wyrm-nvidia-nim`. One rule to carry over: embedding dimensions must match
across a store. If you change providers on an existing database, reindex with
`wyrm_reindex`, because a dimension mismatch makes similarity return zero rather
than a wrong-but-plausible score.

## Staleness

Truths can carry a time-to-live. When one ages past it, recall and `wyrm_truth_get`
mark it stale instead of dropping it, so you see that a fact is old and can
decide whether it still holds. Set a time-to-live on anything that is true now
but will not be forever (a current version, an in-flight plan, a temporary
workaround).

## Tuning notes

- If recall misses an obvious keyword match, the item was probably stored as a
  quest or as chatter, not as a truth or artifact. Recall ranks distilled
  memory; re-capture it as the right kind.
- If recall returns semantically close but wrong items, prefer `wyrm_search`
  for that query, or add a truth that states the exact fact so it ranks first.
- Keep truths atomic. One fact per truth recalls better than a paragraph that
  mixes three.
