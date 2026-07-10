
# Getting started with Wyrm

Wyrm is a local-first memory server for AI agents, spoken over MCP. It keeps
your project's ground truths, lessons, open work, and dead-ends in a structured
SQLite database on your own machine. No cloud account and no separate LLM are
required to run it. An agent connected to Wyrm can recall what was decided last
week instead of re-deriving it, and can be stopped from repeating an approach
that already failed.

This skill gets you from zero to a working loop. The deeper mechanics live in
sibling skills: `wyrm-memory-and-recall` (how recall is scored and what to
store), `wyrm-nvidia-nim` (higher-accuracy retrieval with NVIDIA NIM), and
`wyrm-negative-learning` (the failure firewall).

## Install

```bash
npm install -g wyrm-mcp
```

The install prints a one-time first-run note. The binary set includes
`wyrm-mcp` (the MCP server), `wyrm` (the CLI), and `wyrm-setup` (the client
configurator).

## Wire it into your client

```bash
wyrm-setup
```

`wyrm-setup` detects your installed AI clients and writes the correct MCP
server entry for each. It supports Claude, Cursor, Copilot, Windsurf, and
Codex. Restart the client after setup so it picks up the new server.

To confirm the client sees Wyrm, ask it to call `wyrm_capabilities`. That
returns the live tool inventory and runtime, which is the honest signal that
the connection works. A tool list that does not include `wyrm_*` entries means
the client did not load the server; re-run `wyrm-setup` and restart.

## The loop

Wyrm rewards a simple rhythm. Do these without being asked once the habit sets
in.

1. **Prime at the start.** Call `wyrm_session_prime` before planning. It loads
   the project's ground truths, open quests, validated patterns, and known
   dead-ends into context in one call.
2. **Recall before you re-derive.** Call `wyrm_recall` (or `wyrm_context_build`
   for a token-budgeted assembly) when you are about to reconstruct something.
   The answer may already be stored.
3. **Check before you retry.** Call `wyrm_failure_check` before proposing a fix
   you have tried before. Wyrm remembers what failed.
4. **Capture as you go.** When the work produces durable knowledge, store it:
   stable facts with `wyrm_truth_set`, lessons and decisions with
   `wyrm_remember`, tasks with `wyrm_quest_add`. Use `wyrm_capture` to record
   several at once. Capture signal, not chatter.

## The four kinds of memory

Choosing the right kind is most of good memory hygiene.

- **Truth** (`wyrm_truth_set`) is a stable fact that should stay true until
  something changes it: an architecture decision, a convention, a credential
  location. Truths support a time-to-live and are flagged stale when they age
  out.
- **Lesson or artifact** (`wyrm_remember`, `wyrm_distill`) is distilled
  knowledge: what worked, what a bug turned out to be, a reusable pattern.
- **Quest** (`wyrm_quest_add`) is a unit of work to track across sessions.
- **Session** is the running record of a working session, assembled for you by
  prime and rehydrate.

## Where your data lives

Everything is in a single SQLite file at `~/.wyrm/wyrm.db` (override with
`--db`). It runs in WAL mode and checkpoints on shutdown. Because it is one
file, backup is a copy, and `wyrm_sync_export` produces an encrypted snapshot
you can move between machines. Nothing in the default configuration leaves your
machine.

## Verify

```bash
wyrm stats
```

`wyrm stats` shows counts across truths, artifacts, quests, and sessions, which
confirms writes are landing. From inside a client, `wyrm_capabilities` is the
equivalent check plus the full tool surface.

## Next

Once the loop is a habit, read `wyrm-memory-and-recall` to store things so they
come back, and `wyrm-nvidia-nim` if you want the higher-accuracy retrieval path.
