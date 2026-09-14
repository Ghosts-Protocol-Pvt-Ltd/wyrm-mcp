# Changelog

All notable changes to Wyrm MCP Server will be documented in this file.

## [9.2.3] - 2026-09-14 - Wyrm keeps its memory honest over time

The long-term audit of a store in daily use since June (quests #1150 to #1155)
found that Wyrm kept everything forever, trusted every truth it was ever told, let
its indexes rot, and could silently lose or resurrect data across devices. This
release fixes that, plus the HTTP API gaps the editor clients hit (#1164). Schema
migrations 43 to 49 are additive and forward-safe for stores created on 9.0.x,
9.1.x and 9.2.x. Tool discovery is unchanged (96.9% top-1, 100% top-3 over 65
queries) and the default tool surface stays at 33 tools and 7,995 tokens.

### Upgrading

Nothing destructive happens on upgrade:
- **Failure patterns** that have been quiet for 60 days are downgraded from block to
  warn, but the quiet clock starts when this release's migration is applied, so an
  upgrade alone downgrades nothing. `wyrm failure quiet` previews what would move.
- **Ingest retention** (365 days by default) writes a preview on its first run and
  deletes nothing for 7 days. `wyrm forget --retention --dry-run` shows what would go.
- **Stale truths** are marked in briefs and reported by `wyrm doctor` as advisory
  warnings; they never change doctor's exit code.
- **Backups:** `wyrm doctor` fails when a Wyrm database copy in the Wyrm home is
  readable by other users. `wyrm backup legacy --dry-run` shows the fix.
- **Vectors:** deleting or superseding a row now removes its vectors, and editing
  embedded text drops the old vector and requeues the row. `wyrm maintenance
  --dry-run` shows what the first sweep will prune.
- **Sync:** the first bridge run with the matching connector bridge removes cloud
  copies of superseded and review-queued memories. Run it with `--plan` first.
- **Trust lanes:** new harvested, hook, connector and ingested writes are labelled
  imported. Rows already in the store keep their lane until you run
  `wyrm review lanes --dry-run`, then `--apply`.

### Behaviour changes

- **Briefs show how old each truth is** and mark truths past their TTL as STALE. New
  truths get a default TTL by key and category (`WYRM_TRUTH_TTL_DEFAULTS=0` turns
  defaults off).
- **Failure patterns with no hits for 60 days warn instead of block**, counted from
  this release's migration, and recording the failure again re-arms the block.
- **Ingested third-party data is kept for 365 days** by default
  (`WYRM_INGEST_RETAIN_DAYS`), after a 7-day grace period with a preview.
- **`wyrm doctor` fails when a Wyrm database copy is readable by other users.**
- **New writes from harvest, hooks, connectors and ingestion are labelled imported.**
  Briefs mark them, and imported rows shaped like instructions are left out of briefs.
- **A store created by a newer Wyrm opens read-only**; writes fail with both schema
  versions and the fix.
- **Recall ranks automatic session summaries lower**, drops English function words
  from keyword queries (also in `wyrm_search`), weighs confidence, and shows identical
  content once per result list.
- **HTTP write routes refuse cross-site requests and non-JSON bodies**, rate limits
  apply per authenticated principal, and CORS origins follow the port the server
  binds.
- **Writes are attributed** `cli`, `http` or `mcp` when nothing more specific names
  the writer. These ids grant no authority.
- **The daily maintenance sweep also runs in `wyrm serve` and `wyrm-loop`**, claimed
  so only one process sweeps a store per day.
- **Doctor and setup rows read `name: status`.**

### Fixed: erasure, retention and backups (#1150)

- `wyrm forget` follows derived rows: memories tagged with the forgotten source,
  their vectors and indexing queue entries, and log rows that quote the erased key.
  Every erasure writes an `erasure_log` row with per-table counts, never the erased
  text (migration 43).
- Ingested third-party data (for example transcripts) is retained for 365 days by
  default (`WYRM_INGEST_RETAIN_DAYS`, 0 or `off` disables), enforced by maintenance
  after a 7-day grace period.
- New `wyrm backup create|list|prune|restore|legacy|cloud`: snapshots through the
  SQLite online backup API (never a raw copy of a live WAL database), files 0600 in a
  0700 folder, 7 daily and 4 weekly kept, optional encryption with a vault key, and
  a restore that verifies before and after and pins a pre-restore snapshot.
- Cloud backup uploads an online-API snapshot instead of copying the database, WAL
  and shm files; cloud restore no longer leaves a loose plaintext `.bak`.
- `wyrm maintenance redact --dry-run|--apply [--vacuum]` back-fills credential
  redaction over existing truths, quests, sessions, memories, failure records, logs
  and ingested rows, reporting per-table counts only.
- `wyrm backup cloud` lists snapshots under other machines' prefixes and prunes them
  on request (dry run first; never this machine's prefix).

### Fixed: knowledge that never went stale (#1152)

- New truths get a default TTL by key and category (14 days for status keys, 30 for
  version and release keys, 90 for ops, 365 for architecture and conventions, 180
  otherwise). `WYRM_TRUTH_TTL_DEFAULTS=0` turns defaults off.
- Briefs show each truth's age and mark truths past their TTL as STALE, and flag
  truths that name a path that no longer exists.
- `wyrm truth verify <key|id>`; re-setting a truth with the same value verifies it
  instead of adding a history row.
- Setting a truth whose key is already current under another category warns.
- Supersession writes a `supersedes` decision edge; a `source` made only of typed
  refs writes `because_of` edges.
- Served counts are recorded (`served_counts`), and unresolved failures with no hits
  for `WYRM_FAILURE_QUIET_DAYS` (default 60) downgrade from block to warn with
  history; recording the failure again re-arms it (migration 44).
- Parked quests are a real status: `wyrm quest park|unpark|list`, and rehydrate shows
  them.

### Fixed: index health (#1153)

- `wyrm index backfill [--dry-run] [--limit N]` embeds rows that missed embedding;
  the daily sweep backfills a bounded batch when the provider is healthy.
- Vectors are purged when their row is deleted or superseded, and maintenance prunes
  orphans, superseded rows and old-model duplicates (never the active model of a
  live row, never a model a live server uses) (migration 45).
- Vector search streams rows and uses a (model, content_type, project_id) index:
  global search peak memory growth fell from 154 MB to 60 MB on the reference store,
  with identical results (840 of 840 result lists).
- Quest vectors are labelled `quest` instead of `context`.
- Keyword search plans FTS first; results identical, and far faster under the
  `node:sqlite` backend.
- Editing embedded text requeues the row; stuck indexing-queue rows are cleared
  (migration 47).
- The daily sweep also runs in `wyrm serve` and `wyrm-loop`, claimed atomically so
  one process sweeps a store per day.
- Ended runs older than 90 days are pruned (`WYRM_RUN_RETAIN_DAYS`); graph compaction
  is opt-in (`--compact-graph`).

### Fixed: HTTP API (#1164)

- `POST /pa` registers exactly one project folder; `GET /p` pages (`limit`, `offset`,
  `q`, `path`) with the bare response unchanged.
- Rate budgets are per authenticated principal, with a higher budget for loopback
  and a separate lockout for failed authentication.
- `wyrm serve` shuts down within about 4 seconds on SIGTERM even with keep-alive
  sockets, in-flight requests or SSE streams.
- **Security:** write routes refuse a foreign `Origin`, `Sec-Fetch-Site: cross-site`
  and non-JSON bodies, before authentication, in every auth mode; CORS never answers
  with a wildcard or credentials; the server's own origins follow the port it binds.
- npm audit findings in the editor packages fixed (0 remaining).

### Fixed: fleet and multi-device integrity (#1154)

- Device lineage: sync cursors live in the store, and a device epoch changes on a
  restore or copy, so a restored store's new events are never dropped as echoes; the
  event sequence counter never reuses numbers (migration 46).
- Cloud pull applies rows by timestamp and records real conflicts in
  `sync_conflicts` instead of silently replacing the newer row.
- Federation conflict quests record their quest and close when resolved.
- Superseded and review-queued memories, deletes and visibility demotions retract
  copies already pushed.
- Harvest, bridge, reverse-bridge, metabolize and hook writers are attributed
  (`WYRM_WRITER_ID` attributes without granting authority).
- A pulled cloud row never replaces a different local row that happens to share its
  natural key (skills by name, design tokens by project, category and key); the local
  row stays and the remote version is recorded in `sync_conflicts`.
- Plain CLI writes are attributed `cli`, writes that originate in `wyrm serve` `http`,
  and MCP writes from a client with no name `mcp`, after any explicit agent, writer or
  specific lane. These ids grant no authority.

### Fixed: governance over time (#1155)

- Every review decision records who made it and when (`reviewed_by`, `reviewed_at`),
  through the MCP review tool, the CLI, the HTTP review routes and bulk approve. The
  reviewer is the agent or run when known, otherwise `operator:<WYRM_OPERATOR>`,
  otherwise the surface (migration 48).
- New writes from harvest, trace harvest, auto-extract, reverse bridge, connectors,
  hooks and ingestion go into the imported trust lane. The lane changes how a row is
  shown in briefs; it never sends a row to the review queue.
- With auto-approve on, a weekly sample of auto-approved writes (20 by default,
  `WYRM_REVIEW_SAMPLE_PER_WEEK`) is offered for review without blocking anything:
  `wyrm review sample`, and doctor, prime and rehydrate mention it.
- Governance actions are written to the audit chain: approvals and rejections, the
  auto-approve setting, erasures, applied migrations, truth changes, failure resolve
  and delete, sharing, visibility and lane changes. Events carry ids, kinds and
  counts only.
- `wyrm audit verify` reads the chain in batches and checks signed checkpoints
  (every 1,000 events by default, `WYRM_AUDIT_CHECKPOINT_EVERY`);
  `--from-checkpoint` starts after the newest good one. `wyrm audit checkpoint`
  writes one on demand.
- A store created by a newer Wyrm opens read-only: reads work, writes are refused
  with both schema versions and the fix, in the MCP server, the CLI, the HTTP
  server, the memo and guard hooks, and backup restore. The hooks keep working and
  only skip their ledger writes. `WYRM_ALLOW_NEWER_STORE_WRITES=1` overrides, and
  `wyrm doctor` still fails.
- The session rehydrate brief follows the trust lanes like the context brief:
  untrusted rows and imported rows shaped like instructions are left out, other
  imported rows are marked, and a brief made only of your own and your agents'
  memories is unchanged.
- When maintenance expires the failures recorded by an abandoned run, it writes one
  audit event for the sweep.

### Fixed: recall ranking (#1151)

Every change was measured before and after on LoCoMo, the freshness bench and the
LongMemEval floor; nothing moved outside the committed bands.

| Bench | 9.2.2 | 9.2.3 |
|---|---|---|
| LoCoMo keyword, recall@1 / @5 / @10 / MRR | 29.5 / 52.4 / 59.9 / 0.393 | 32.5 / 57.0 / 63.9 / 0.428 |
| LoCoMo hybrid (bundled model), recall@1 / @5 / @10 / MRR | 32.4 / 60.0 / 72.1 / 0.443 | 33.0 / 60.0 / 71.9 / 0.446 |
| LongMemEval floor, recall@1 / @5 / @10 / MRR | 69.6 / 95.7 / 100 / 0.813 | 73.9 / 100 / 100 / 0.846 |
| Freshness | 16 of 16 | 16 of 16 |

- Automatic session summaries (`[auto:*]` rows) are down-weighted on every recall
  path, not only in the context brief. On a long-lived reference store, automatic
  rows in the top 10 hybrid results for 20 generic questions fell from 112 to 18, and
  the first hand-written result moved from an average rank of 6.9 to 1.3.
- Keyword queries drop English function words. Identifiers, paths, anything with a
  digit or punctuation, and all-caps acronyms are kept, and a query made only of
  function words is kept whole. This also applies to `wyrm_search` on sessions and
  quests.
- Confidence decay no longer refreshes a memory's `updated_at`, so decayed rows stop
  looking recent; decay stamps `decayed_at` instead (migration 49).
- Hybrid ranking weighs confidence with a bounded factor
  (`WYRM_RECALL_CONFIDENCE_WEIGHT`, default 0.3, 0 turns it off), so low-confidence
  harvested rows give way to confirmed ones.
- A result list keeps one copy of identical content within the same trust lane (the
  best-ranked one). Nothing in the store is merged or hidden.
- Memories in the operator lane never lose confidence to decay, as documented; the
  decay selector used to include them.
- Global recall without vectors also weighs confidence. On the reference store,
  low-confidence harvested rows in the top 10 for 20 generic questions fell from 84
  to 53, and the first hand-written result moved from an average rank of 2.3 to 1.0.
- The context brief applies the automatic-summary penalty once instead of twice;
  brief contents on the reference store are identical.
- A store missing a migration between recorded ones (possible when builds from
  different branches opened the same store) now gets the missing migrations applied,
  for every migration from 34 on, which are all safe to apply late. `wyrm doctor`
  reports any gap, and a gap below 34 needs a restore.
- `bench/auto-crowding.mjs` and a test gate keep this measured: on its synthetic
  store the recall target is found first in 11 of 12 queries (was 8), and duplicate
  slots in global top-20 results fell from 51 to 0.

### Changed: wording (#1163)

- Messages, tool descriptions and CLI output no longer use em dashes. Doctor and
  setup rows read `name: status`, empty placeholders read `never`, `n/a` or `none`,
  and graph edges print as `[rel]→`.
- Write-receipt reasons already stored keep their old text; new writes use the new
  wording.
- The instruction blocks that `wyrm embed` and `migrate-prompt` compare with existing
  installs are unchanged, so no install is reported as stale.

### Other packages

- wyrm-lsp 0.1.3: uses the paged project list and scopes quest lookups to the
  enrolled project containing the file.
- VS Code extension 3.1.1: registers the folder on Start Session through `POST /pa`
  and looks project ids up by path.

305 suites / 3,395 tests green (4 skipped).

## [9.2.2] - 2026-09-14 - Strict arguments and a leaner vault wrapper

Finishes the follow-ups the 9.2.1 audit deferred. Tool calls now reject arguments
a tool does not accept, as the v8 constitution requires, and the `wyrm vault exec`
wrapper that sits in front of every MCP server no longer loads the store, the SDK
or the CLI. There is no schema change and no ranking change: the tool-discovery
benchmark (`tests/discovery-bench.test.ts`, 65 queries over the 33 default tools)
scores 96.9% top-1 and 100% top-3, identical to 9.2.1.

### Behaviour changes

- **A tool call with an argument the tool does not accept now fails** with a
  `WYRM_VALIDATION` error that names each unknown key and suggests the closest valid
  one. Before, unknown keys were silently dropped, so a misspelled argument did
  nothing. Every published schema property and every documented alias (the quest and
  goal id spellings, `validated_fix`/`why_it_worked` and the rest) is still accepted.
  Set `WYRM_ALLOW_UNKNOWN_ARGS=1` to turn the rejection into a warning on stderr.
- **Every `wyrm_capture` mode returns `mode` and `refs`** (`"mem:N"`, `"truth:N"`,
  `"quest:N"`) alongside `status`, and `receipt` where a write receipt exists. The
  change is additive: every existing field is kept.
- **Feedback and unshare are reachable from the default tools** as
  `wyrm_capture mode=feedback` and `wyrm_replication action=unshare`. Messages that
  pointed at the hidden `wyrm_feedback` and `wyrm_unshare` now point at these routes.
- `wyrm_capabilities` names every capability by a route the active profile can call,
  adds a `cli` field for capabilities that only have a CLI command, and reports the
  active profile's tool count ("33 MCP tools (154 in the full profile)") instead of a
  fixed 137.
- Validation errors read "Correct the 'x' argument: ... Then call ... again."
- The `wyrm` bin now points at `dist/wyrm.js`, a small dispatcher. `wyrm vault exec`
  runs without loading the rest of the CLI; every other command behaves as before.
- `wyrm vault exec`: a child killed by a signal now ends the wrapper with the same
  signal (it used to exit 1); SIGINT, SIGTERM and SIGHUP sent to the wrapper reach the
  child; if the wrapper's own parent dies, the child gets SIGTERM and, 10 seconds
  later, SIGKILL. `--as` without a name is a usage error (it used to inject a
  variable literally named `undefined`), and a command that cannot start prints an
  error instead of failing silently.

### Fixed

- **The vault wrapper's memory.** Every Claude Code session and the Hermes gateway
  start Wyrm through `wyrm vault exec`, which stayed resident for the whole session
  with the full CLI loaded: 61 of Wyrm's own modules. It now loads 5. Measured with a
  key-file vault, the wrapper's resident memory fell from 66.5 MB to 53.9 MB (bare
  Node is 47.2 MB).
- **The Claude Code run map lost sessions and never shrank.** The run-auto hook wrote
  `~/.wyrm/auto-runs.json` without a lock, so 16 sessions started together kept only
  13 mappings, and nothing ever removed an entry. It now writes under a lock with an
  atomic rename, stamps each entry's last use, and drops entries idle for more than
  7 days. `wyrm run prune [--older-than <days>] [--dry-run]` tidies an existing file
  and also drops mappings whose run has ended.
- Busy errors carry the tool name explicitly instead of having it parsed back out of
  the message text.
- `wyrm_license` reported a hard-coded "32 MCP tools"; it now reports the real count.
- The golden-fixture generator refuses to write a call the argument check would
  reject, and the fixtures were regenerated.
- Validation, busy, deprecation and CLI-redirect messages no longer contain em dashes,
  and a test keeps it that way.
- **A session update over the HTTP API erased fields it was not sent.** `POST /su`
  passes absent fields as `undefined`, and `updateSession` wrote them as NULL, so a
  client that saved only its completed list wiped the session's summary and notes.
  `updateSession` now skips `undefined`; an explicit `null` still clears a field.
  Found by the VS Code extension's engine mode.
- **The first request to `wyrm serve` could stall for up to about two minutes.**
  better-sqlite3 13 builds a Node diagnostic report when it first loads its binding,
  and that report resolves a hostname for every open socket synchronously; `wyrm
  serve` opens its store after its listener exists. The storage seam now turns off
  network data in process reports before it opens a database. The stall appeared in
  6 of 6 live runs during the extension work and did not reproduce with loopback-only
  sockets, so its length depends on which sockets are open.

### Other packages

- **wyrm-lsp 0.1.2.** The published 0.1.1 could not start as a command: its bin had
  no node shebang and was not executable in the tarball, and with the dependencies
  its lockfile pins the server failed to load (the `vscode-languageserver/node`
  subpath is the one version 10 exports). 0.1.2 builds against the pinned
  dependencies and ships an executable bin. Its default API URL follows `WYRM_PORT`,
  then `WYRM_HTTP_PORT`, on 127.0.0.1, the way `wyrm serve` binds; `WYRM_HTTP_URL`
  still wins. A test compiles the server and runs an LSP `initialize` over stdio.
- **VS Code extension 3.1.0 (VSIX).** A new engine mode uses the local Wyrm engine
  through the HTTP API of `wyrm serve` for quests, sessions, context, ground truths,
  recall and capture. `wyrm.mode` chooses `auto` (the default: the engine when it
  answers and accepts the key, otherwise the standalone store), `engine` or
  `standalone`, and the status bar shows which is active and why. The engine key
  comes from VS Code secret storage ("Wyrm: Set Engine Token") or `WYRM_TOKEN` and is
  sent only to loopback or https.
  - The declared settings `autoInjectContext`, `autoSaveOnClose` and
    `maxContextTokens` now take effect; `mcpServerPath`, which was never read, is
    removed.
  - Closing VS Code no longer closes the database under an unfinished session save.
  - `better-sqlite3` loads on first use, so a native module that cannot load costs
    only standalone mode, never activation.
  - `npm run package:vsix` packages from the lockfile, and the package carries its
    LICENSE.

wyrm-lsp: 29 tests green. VS Code extension: 83 tests green.

259 suites / 3,050 tests green (1 skipped).

## [9.2.1] - 2026-09-14 - A quality pass across the whole board

A hands-on audit of 9.2.0 covered the MCP tools, the CLI, the server process, the
Claude Code hooks, packaging, and the editor and adapter packages. It found defects
that lost no data but made Wyrm harder to trust or to run. This release fixes them.
There is no schema change and no ranking change: the tool-discovery benchmark
(`tests/discovery-bench.test.ts`, 65 queries over the 33 default tools) scores
96.9% top-1 and 100% top-3, identical to 9.2.0.

### Behaviour changes

- Typed tools no longer put validation or busy errors in `structuredContent`; the
  same JSON envelope is in the text channel.
- Arguments that used to be ignored or coerced are now `isError` validation
  errors, including `approved: "yes"` on `wyrm_review` and `run_scope` values
  outside its enum on `wyrm_failure_check`.
- `wyrm_failure_check` without `projectPath` also checks the project containing
  the working directory.
- The quest list returns at most 50 quests unless `limit` is given, and reports
  the total.
- An explicit `wyrm_capture` with `mode: "truth"` is written, as `wyrm_truth_set`
  writes, instead of being queued whenever its category already held a truth.
- `wyrm_share` refusals and "not found" responses are `isError`.
- CLI errors go to stderr. A `--project` that matches no project exits 1 instead of
  returning unscoped results, and `wyrm show` exits 1 when nothing is found.
- `wyrm review --auto` without a value shows the setting. Only `on` or `1` enables
  auto-approve; `--auto true` is a usage error.
- `wyrm update` refuses to replace a wyrm that is not a global npm install.
- `wyrm serve` no longer reads `PORT`; use `--port`, `WYRM_PORT` or
  `WYRM_HTTP_PORT`.
- `wyrm doctor` can exit 1 when wyrm-guard hooks are missing or live servers
  predate the installed package.
- A non-interactive vault read waits up to 90 seconds for a locked keyring
  (`WYRM_VAULT_WAIT_SECONDS`, 0 disables).
- A stdio server exits when its client is gone (below).

### Fixed: MCP tools

- **Responses conform to their outputSchema under the official MCP SDK client**,
  which rejects a response that does not. Several `wyrm_capture` modes returned
  bodies without `status`; every mode now returns one, and the enum gains
  `"skipped"`. A new test drives the built server through the SDK client.
- **Validation where arguments crashed, reached SQL unchecked or were ignored:**
  `wyrm_review` (`artifactId`, `approved`; the response echoes the real boolean),
  `wyrm_failure_check` (`scope`, `run_scope`), `wyrm_share` and `wyrm_unshare`
  (`kind`, `id`), `wyrm_feedback` (`artifactId`, `success`), `wyrm_truth_set`
  (`key`, `value`, `confidence` from 0 to 1), quest add (`title`), presence
  announce (`agent_kind`), `wyrm_decided_because`, capture `mode`, recall `limit`
  and `kind`, search `type`, stats `view`, capabilities `format`, buddy `size`, and
  replication `action` (the error lists the valid actions).
- **Every enum a validator enforces is published in the tool's inputSchema**,
  including capture `kind` and quest `priority`, and a test keeps the two in step.
  The validation error no longer claims the schema lists values it did not.
- `wyrm_share` on a row in a private grove says so and names the fix, instead of
  "not found".
- Quest actions accept `quest_id`, `questId`, `id` or `"quest:N"`; goal actions
  accept `id` or `goal_id`. The "#undefined" messages an unrecognised spelling
  produced are gone.
- `wyrm_session_prime` accepts `projectPath`. `wyrm_truth_get`, `wyrm_truth_set`,
  `wyrm_context_build`, `wyrm_thread` and `wyrm_decided_because` fall back to the
  working-directory project or name the missing parameter.
- Recording the same decision edge twice returns the existing edge, and a decision
  without a rationale is labelled with its truth's key.
- An auto-classified truth capture is queued as a possible conflict only when an
  existing truth has the same key or a near-identical statement, not whenever the
  category holds any truth.
- Pointers to tools outside the default surface now name CLI commands or default
  tools, and `wyrm_search` gains `type: "symbols"`.
- Recall and search receipts report vector coverage. When the active model covers
  under 25% of eligible memories they say "effectively FTS-only" instead of hybrid.
  Ranking is unchanged.

### Fixed: the server process

- **A stdio server exits when its client is gone**: stdin closed, stdout EPIPE, or
  its parent process gone. On stdin EOF it first answers the requests still in
  flight, for at most `WYRM_STDIO_DRAIN_MS` (default 30 s). Before, a server whose
  client died was reparented and kept running with `wyrm.db` open and the old code
  loaded.
- HTTP: `GET /` reports the package version; the dashboard loads no fonts from a
  CDN and both CSPs allow no remote host; the unused `/dragon-mark.svg` route is
  gone; the port is `WYRM_PORT`, then `WYRM_HTTP_PORT`, never `PORT`.
- `npm pack` no longer leaves the local `dist/` without its type declarations. The
  published package is unchanged in shape, and its npm description now fits npm's
  255-character limit.
- The sovereign sync daemon removes its pid file on exit and treats a stale one as
  stale.
- Event pruning skips a store that has no events table without a warning; the
  warnings that remain name the database path.

### Fixed: the CLI

- **Errors go to stderr.** Under `wyrm vault exec <key> -- wyrm-mcp`, stdout is the
  MCP channel, and a locked keyring at boot put error text on the wire: Hermes
  logged 28 parse errors in 7 days.
- **`wyrm vault` tells a locked or unavailable keyring apart from a missing master
  key**, and a non-interactive call waits for the keyring with backoff. At boot,
  Hermes started `wyrm vault exec ... -- wyrm-mcp` before the desktop keyring
  unlocked and parked its Wyrm connection 33 times on 2026-09-13.
- **`wyrm update` installs with `--prefix`** set to the npm prefix Wyrm runs from,
  checks that it is writable, and prints the exact command on failure. It used to
  fail with EACCES, or install a second copy, where npm's global prefix is `/usr`
  and Wyrm lives under `~/.npm-global`. The README Quickstart documents the
  user-prefix install.
- **`--help` no longer runs the command.** About 20 of 41 subcommands ran on
  `--help`: `metabolize` and `maintenance` ran, `guard` and `memo` attempted hook
  installs, `serve` started a server. A help table now drives `wyrm --help`,
  `wyrm <command> --help` and `wyrm help <command>`, and lists the commands the help
  used to omit. An unknown command suggests the closest one and exits 1.
- `wyrm review --auto` no longer switches auto-approve on when asked for its status.
- A `--project` that matches nothing exits 1 with "Project not found" in `search`,
  `recall`, `ls`, `presence`, `render` and `reverse-bridge`. Ten commands warn on an
  unknown flag and suggest the closest real one.
- `wyrm show` exits 1 on not found or an unknown type prefix.
- `wyrm capture` reports what it stored ("Stored as memory mem:N (classified as a
  decision ...)") instead of "Captured as truth" for a memory, and enrols the git
  repository on first use.
- `wyrm recall "<query>"` runs a search; `wyrm search` counts matches still in the
  review queue; a bare `wyrm digest` defaults to `--writes`; `wyrm forget --match`
  says it covers the ingested data lake only.
- `wyrm review --approve-all` suggests `wyrm index rebuild` only when an approved row
  has no vector.
- `wyrm doctor` checks wyrm-guard hooks when `~/.claude` exists, and lists live
  servers started before the installed package, which still run the old code.
- `wyrm setup` detects Claude Code by `~/.claude` or a `claude` binary rather than by
  `~/.claude.json`, which setup itself writes; counts only real clients; and logs
  download progress per file when not on a terminal.
- `wyrm cloud sync` exits 1 when the sync reports errors.
- `wyrm serve` takes `--port` and always prints the dashboard URL. `wyrm-ui` reads
  `WYRM_PORT`, passes the port to the server it starts, and answers `--help` without
  starting anything.
- Colour only on a terminal, honouring `NO_COLOR` and `FORCE_COLOR`. The store no
  longer prints "Database checkpoint completed" after every command, and
  `WYRM_LOG_LEVEL` is no longer written into child processes' environment. Warnings
  and errors reach `wyrm.log` stamped with the database path, and `WYRM_LOG_DIR` is
  honoured.
- `guard` and `memo` warn, instead of showing a success mark, when they armed
  nothing.

### Fixed: hooks and the session brief

- **The rehydration brief is packed section by section inside its budget**, in the
  order truths, failures, patterns, quests (ten, then a count), and its receipt
  counts what the text actually holds. One long truth no longer hides the others.
  Choosing the session to restore skips Hermes cron runs, and their skill preamble
  is stripped.
- The session-rehydrate hook prints "Wyrm unavailable: <reason>. Nothing was
  restored." when the store, the binary or the call fails, instead of reporting a
  first session.
- The citation-audit hook grounds ids served in hook context and in Wyrm tool
  output (`#N`, `"id":N`), and in a continued chat's parent transcript.
- The run-auto hook reuses the live run on resume and compact, and never starts one
  on compact.
- **New packaged hook `wyrm-prompt-context.mjs`** on `UserPromptSubmit`, installed by
  setup: one line per record, no auto-session rows, deduplicated within the session,
  a 1,500-character cap, at least two words shared with the prompt, and no recall on
  harness notifications. Median 534 ms against 6,453 ms for the unpackaged hook it
  replaces, on a copy of a store in daily use.

### Fixed: other packages

- `wyrm-lsp`: quest hovers read the compact `GET /q` shape (they showed
  "undefined"); the npm bin has a shebang (the published 0.1.1 lacks one and does
  not start, so this needs a wyrm-lsp release); the README is current.
- VS Code extension: a README, a corrected comment that claimed a shared database, a
  fixed `docs/VSCODE_EXTENSION.md`, and a repository link to the public repo.
- The `@wyrm-mcp` adapter READMEs say the packages are not published yet and show how
  to use them from the monorepo.
- `scripts/mcp-registry-publish.sh` logs in by DNS when the `.well-known` key file is
  unavailable, as it has been since the ghosts.lk site was rebuilt.

253 suites / 2,983 tests green (1 skipped).

## [9.2.0] - 2026-09-13 - NVIDIA retired the NIM models, and nothing said so

NVIDIA retired both of Wyrm's default hosted NIM retrieval models at
2026-08-25T09:00Z. Every embed and rerank call has answered HTTP 410 since. Wyrm
caught the error at both embed sites and returned nothing, so on every install
using the NIM tier new memories stopped being embedded and recall fell back to
keyword-only, while `wyrm doctor` still showed a green tick. On the reference
store, 1,591 memories written after the cutoff have no usable vector.

### If you use NIM (`WYRM_VECTOR_PROVIDER=nim`)

Upgrade, then re-embed with `wyrm index rebuild`. A vector made with the retired
model cannot be compared with any other model's, so none of the old vectors are
usable again. If you set `WYRM_NIM_EMBED_MODEL` to the retired id yourself, remove
it or point it at a supported model.

### Changed

- **Default NIM embedding model is now `nvidia/nemotron-3-embed-1b`** (2048
  dimensions, unchanged). Measured on the committed LoCoMo benchmark
  (`bench/nim-retrieval.mjs`, 2 conversations, k=10): recall@1 39.9%, recall@5
  66.4%, recall@10 76.7%, MRR 0.507. The other live candidate,
  `nvidia/llama-nemotron-embed-vl-1b-v2`, was rejected: it answered an input over
  8,192 tokens with HTTP 400 despite `truncate: 'END'`, so a long memory would
  never embed.
- **Default NIM reranker is now `nvidia/llama-nemotron-rerank-vl-1b-v2`.** Its
  predecessor was retired the same day, and reranking falls back to fusion order on
  any error, so opted-in installs silently lost it. Measured over the new embedding
  model on the same benchmark (301 questions): recall@1 39.9% to 55.5%, recall@10
  76.7% to 80.1%, MRR 0.507 to 0.630, at about one extra second per query.
  Reranking stays opt-in (`WYRM_RERANK=1`).
- **The recall@1 figures previously published for NIM** (33% local, 47% NIM
  embeddings, 52% with rerank) were measured on the retired models. They are
  withdrawn from the README and the NIM skill until re-measured across all legs in
  one configuration.
- **The NIM skill named environment variables Wyrm never reads**
  (`NIM_API_KEY`, `WYRM_EMBED_MODEL`, `NIM_BASE_URL`), including in its vault
  example. Corrected to `NVIDIA_API_KEY`, `WYRM_NIM_EMBED_MODEL` and
  `WYRM_NIM_BASE_URL`.

### Added

- **`wyrm doctor` checks embedding health.** It fails when the configured model is
  retired, when the vector store's last embed error is newer than its last
  success, or when active memories exist newer than the newest vector under the
  active model. The third signal needs no new bookkeeping, so it also flags stores
  that stalled before this upgrade.
- **The vector store records embed failures** (`embed_last_ok_at`,
  `embed_last_error_at`, `embed_last_error` in `wyrm_meta`) instead of
  discarding them.
- **`wyrm visibility <truth|mem|quest>:<id> <within|org|public>`**, an operator
  command for `cross_project_visibility`: the column that decides both
  cross-project search and cloud egress. It refuses promotion inside a private
  grove, bumps `updated_at` so the incremental sync collector actually picks a
  promoted row up, and says plainly that demotion does not recall copies already
  synced. CLI-only by design (quest #1148).
- **A store migrated by a newer Wyrm is detected.** Opening one warns, and
  `wyrm doctor` fails the Schema check, instead of an older binary writing to it
  silently.

### Fixed

- **secure_delete was never on.** The erasure path and its CLI message have said
  "freed pages are overwritten (secure_delete is on)" since 9.0.0. Nothing set the
  pragma, and better-sqlite3 builds with it off. It is now set on every open, with
  a test that reads the database file's bytes. Pages freed before upgrading remain
  until a VACUUM.
- **Approving a queued memory never embedded it.** All four approval paths cleared
  `needs_review` with a bare UPDATE, so approved memories stayed keyword-only for
  good. `MemoryArtifacts.approve()` now does what `add()` does for an active row,
  and the CLI waits for embeds before it exits. Bulk `--approve-all` stays a single
  UPDATE and tells you to run `wyrm index rebuild`.
- **Updating a shared or promoted truth made it private again.** Each new version
  was inserted with the column defaults; `is_shared` and `cross_project_visibility`
  now carry forward.
- **`wyrm_capture` receipts reported the requested review state, not the stored
  one.** Under auto-approve a live row came back "queued_for_review ... review to
  activate".
- **Credential redaction reached only memory inserts and ingest.** It now also
  covers ground truth values and rationales, quest titles and descriptions, memory
  updates, and tool-call argument summaries, and it recognises Stripe secret and
  restricted keys and Google API keys. Rows written before this stay as they were
  stored; back-filling them is quest #1150.
- **`tool_call_log` and `usage_events` grew without bound.** Their prune functions
  existed and nothing called them. The daily sweep now keeps 90 days
  (`WYRM_TOOL_LOG_RETAIN_DAYS`, `WYRM_USAGE_RETAIN_DAYS`), rolling usage up into
  `cost_tracking` before deleting.

### Found, not fixed in this release

A read-only audit of a store in daily use since June 2026 produced the fixes above
and filed the rest with evidence: the bridge push ignores grove privacy (#1149);
erasure, retention and backups (#1150); recall crowded by auto session captures,
a ranking change that needs a measured run (#1151); knowledge that never goes
stale (#1152); index backfill and cleanup (#1153); multi-device integrity (#1154);
governance under permanent auto-approve (#1155); and the cloud connector's
retired NIM default (#1156).

237 suites / 2,784 tests green (1 skipped).

## [9.1.2] - 2026-09-04 - setup stops clobbering your config, and doctor stops calling a broken index healthy

### Fixed

- **Setup destroyed customised MCP server entries on every upgrade.**
  `configureClient` assigned `servers['wyrm'] = freshEntry` outright. The case
  that caught it: an operator had wrapped the launch command in a
  secret-manager call to inject an API key, and set an env var selecting the
  hosted embedding provider. A routine `npm i -g wyrm-mcp` re-ran setup and
  destroyed both, so the server fell back to the bundled local model and the
  semantic half of recall went from 73% coverage to 1%.

  Setup now merges. A default-shaped entry is refreshed so upgrades still
  repoint the binary, but any env you added survives. A customised launcher is
  kept as it is, and only missing env keys are filled in. Your env always wins
  over ours, because that is where the settings that change behaviour live.
  When a launcher is preserved, setup says so. This is the same never-clobber
  stance the hook installer has always taken for `settings.json`.

- **`wyrm doctor` reported a dead vector index as healthy.** Any non-zero
  coverage printed a green tick, so an accidental provider switch that
  stranded 4,344 vectors under the previous model showed as fine at 1%. That
  silence is how the bug above survived. Doctor now fails loudly when another
  model holds more vectors than the active one, and names that model, because
  that sentence is the entire diagnosis. It also fails when a store of 50 or
  more memories is under 25% indexed.

## [9.1.1] - 2026-09-04 - the token firewall only memoises text

### Fixed

- **Images were being memoised.** 9.1.0 memoised anything the Read tool
  touched, which meant screenshots and photographs: a 547 KB PNG became a memo
  whose landmark map was mangled binary, and whose `bytes / 4` token estimate
  claimed 140,000 tokens for a picture that actually costs a couple of
  thousand. On the machine this was found, 23 of 52 memos were images and they
  accounted for 91% of the claimed savings.

  The arithmetic was the smaller half. In `warn` mode a repeat read of an image
  would have handed the model that mangled binary as context INSTEAD of letting
  it see the picture, which is a worse outcome than not having the firewall at
  all. Found by measuring the feature before recommending it be switched on.

  A memo now requires the file to be text, decided by sniffing the head of the
  file the way `grep` and `file` do (a NUL byte, or more than a tenth
  non-printable, means binary) rather than by an extension list, because the
  extension is a hint and the bytes are the truth. Text in any language is
  unaffected.

## [9.1.0] - 2026-09-04 - the store maintains itself, and stops re-reading what it already read

### Added

- **The store now maintains itself.** `maintenance.ts` has always known how to
  prune (session_seen_artifacts, served_records, the Live Memory event log,
  failure_blocks, run_briefs) and to metabolize (supersede exact duplicates,
  decay stale auto-captures). Nothing ever ran it. Scheduling was left to the
  operator, which in practice meant every retention setting in the product was
  a knob that had never once fired, on every install. The store grew, recall
  ranked over more rows that no longer earned their place, and the only symptom
  was a database that got slower for reasons nobody attributed to a missing
  cron entry.

  The MCP server now claims a daily maintenance slot on boot and runs the cheap
  half in the background, after the transport is already serving. The slot is
  claimed with one conditional UPDATE, so a machine running many servers at
  once produces exactly one sweep and the rest return having touched nothing.
  It never blocks a tool call, never throws, and never VACUUMs behind your back
  (that keeps an exclusive lock, so `wyrm maintenance --vacuum` stays explicit).
  `WYRM_AUTO_MAINTENANCE=0` opts out; `WYRM_MAINTENANCE_INTERVAL_HOURS` retunes it.

- **`wyrm doctor` gained two checks**: *Store upkeep* (when the last sweep ran,
  and a loud failure if it has been more than three days or upkeep is disabled)
  and *Token firewall* (whether the read memos are armed, and what they have
  measured). Both exist because the failure mode being fixed here was silence.

- **`wyrm setup` arms the token firewall**, in observe mode. It records what you
  read in full and counts the repeats; it hands nothing back until you have seen
  your own numbers and set `WYRM_MEMO_MODE=warn`. A firewall nobody installs
  measures nothing, which is why arming it is part of setup rather than a
  paragraph in a README.

- **Token firewall, phase 1.** Two Claude Code hooks (`wyrm-memo`, installed
  by `wyrm memo`) remember every whole-file read as a hash plus a landmark map
  in the new `read_memos` table (migration 42), and recognise a repeat
  whole-file read of an unchanged file above a size floor. `observe` mode
  (default) counts the repeat and logs the tokens at stake to the savings log;
  `warn` mode hands the map back as hook context so the model can read a range
  instead. Never blocks. Raw file text is never stored. `wyrm memo --report`
  shows memos, repeats and tokens. Measured motivation: on one 157-call
  session, tool results were about 80k tokens and a large share were re-reads
  of unchanged files.

## [9.0.0] - 2026-09-03 - LATCH: make every claim true before the source goes public

The release before the source opens. Scope was deliberately narrow — only work
that is **irreversible on publish**, **armed by publication**, or a **public
claim that is currently false**. Everything else is post-flip work done in the
open, where it is cheaper.

**Breaking for embedders.** `MemoryArtifacts.add()` no longer lets a caller
activate an untrusted-lane row whatever the auto-approve setting says, and
`FailurePatterns.resolve()` returns a `ResolveResult` and is authority-gated.

**The trust lanes gained a producer.** `source_trust` has existed since v8, but
nothing in the codebase ever wrote `untrusted`, so the lane was a schema column
with no inhabitants and the quarantine protecting it had never been exercised on
real data. Ingested third-party content now lands there. Auto-approve was also
scoped: it could previously clear the review queue for any lane, which quietly
disarmed the gate it was meant to defer to.

**Erasure exists.** `deleteDataCategory()` had no caller anywhere in the tree, so
the ingested lake — WhatsApp and email transcripts whose data subjects never
interacted with Wyrm — had no reachable removal path at all. `forgetData()` is
that path: dry-run by default, refuses an empty predicate, and sweeps the FTS
index with the rows so erased content stops being searchable rather than merely
unlisted.

**The store is owner-only.** `~/.wyrm` was created with the default umask and the
database landed 0644. Both are now 0700/0600, re-tightened on every open so
existing installs are fixed rather than only new ones. Inert on a single-user
laptop, which is how it survived; it bit by default in containers, CI runners and
the shipped Dockerfile.

**Two gates on every CI build**, not only on flip day: an identity gate (operator
paths, third-party emails, the client roster, account ids) and a claims gate (a
comparative rate ships with its denominator or it does not ship). Both found real
leaks in the release that introduced them, which is the argument for having them.

### Found by cold install, not by the suite

The release was packed with `npm pack`, installed into a scratch `HOME`, and used
as a stranger would. The 2,697-test suite found none of the following.

- **A full review queue read as an empty store.** A first capture on a fresh
  machine lands at 60% confidence and is therefore withheld from recall, so the
  new user's first two actions were "store a thing" and "fail to find the thing I
  just stored". Worse than silence: recall answered it with *"Use `wyrm_remember`
  to store knowledge as you work"* — advice that makes a queued store strictly
  worse, since every new capture joins the queue suppressing the answer. Both
  recall surfaces now name the queue and the command that clears it, and
  `queued_for_review` rides the structured body. No ranking changed.
- **Six stray `@license AGPL-3.0-or-later` headers** from the 6.9.x relicense. The
  clean-room assembler only rewrites `@license Proprietary`, so these would have
  published as AGPL files inside an Apache-2.0 repository — one-way incompatible.
- **`bench/listtools-size.mjs` reported a violation that was not one**, pinning 32
  tools after the surface pin was deliberately grown to 33.
- **Documentation still required Ollama** two releases after 8.7.0 bundled the
  model, invented a signup step that no code path enforces, and stated AGPL as the
  licence across every launch draft.

The adapters also moved off the `@wyrm` npm scope, which belongs to another
account, to `@wyrm-mcp/*` — matching `wyrm-mcp`, the package this project owns.

Migration 41. 226 suites / 2,697 tests green.

## [8.7.6] - 2026-08-19 - The grounding ledger becomes load-bearing

8.7.5 shipped attestation but left it inert: the ledger was written only by
`context_build`, so on the `recall`/`search`/`prime` paths real sessions use,
there was nothing to attest against. The Stop hook covered Claude Code from the
transcript; clients that cannot run hooks (claude.ai, desktop, Cowork) had no
enforcement at all, and those are precisely the surfaces where the server is the
only thing that knows what it served.

**Migration 41 — `served_records`.** A per-connection ledger keyed by an opaque
`session_key`. Deliberately *not* `run_id`: `WYRM_RUN_ID` is absent from the MCP
server environment, so `run_id` resolves null at dispatch and every session
would collapse into one bucket, letting any id ever served to anyone ground a
citation. For stdio the process is the connection, so an id minted at boot
identifies the session exactly.

- Recording happens at the single dispatcher chokepoint, reusing the string
  already serialized for telemetry — every read verb covered, no per-handler
  surgery. Failure-isolated: a bookkeeping fault never fails a good call.
- The extractor is deliberately permissive, and the asymmetry is load-bearing:
  over-recording only makes attestation more forgiving, while a missed record
  makes it accuse an honest citation. Recall's bare `(ID: n)` heading carries no
  kind word, so it is recorded under every kind rather than guessed at.
- Pruned on the existing `WYRM_SEEN_TTL_DAYS` knob.

Verified against the live database: one real `wyrm_recall` wrote 20 ledger rows;
citing a served id attests clean, a fabricated id is flagged, and a second
connection cannot borrow the first's grounding. 14 tests.

## [8.7.5] - 2026-08-19 - The grounding rules get an enforcement side

8.7.4 shipped grounding rules as server `instructions`. Instructions are
directives, and a directive is not a guarantee. This makes them checkable.

**`wyrm-citation-audit.mjs`** (new, installed on `Stop`). The transcript is
already the ledger: everything Wyrm put in front of a session is in it, so a
record id that appears in the reply but in no tool result, no SessionStart
brief and no injected context was never retrieved. That is set membership, not
a judgement call. Ungrounded citations are reported back as context.

- Audits only the final assistant turn's user-facing text. A `tool_use` echoes
  ids the model is looking up, and `thinking` is not an assertion to the user,
  so neither counts as a claim.
- Advisory, never blocking. A false accusation that silently ate a turn would
  be worse than the problem it polices.
- Validated on 30 real transcripts: 0 false positives, catches a synthetic
  fabricated citation.

**`SessionSeen.attest()`** (new). The same check against the in-database
ledger: returns `served` citations, `unserved` (fabricated) ones, and `unseen`
records the session was given but never cited. 10 tests.

Known gap, not papered over: `session_seen_artifacts` is still written only by
`context_build`, so `attest()` is correct but not yet load-bearing on the
recall/search/prime paths. The Stop hook does not depend on it.

## [8.7.4] - 2026-08-19 - Grounding rules in the server directive

Wyrm's retrieval is not generative — recall is local FTS5 + vector search
and invents nothing — so the hallucination risk sits entirely on the
reading side: a model paraphrasing a memory that was never returned, or
restating a stored claim as if it were still current.

The MCP server `instructions` now carry `WYRM_GROUNDING_INSTRUCTIONS`:

- Claim something is "in memory" only when a tool call in THIS conversation
  returned it, and cite the id (`mem:NNNN`, `truth:NNN`, `quest:NNN`).
- Never invent or reconstruct an id, a quest number, or the contents of a
  memory that was not retrieved.
- An empty recall is a valid answer — say memory has nothing on it rather
  than filling the gap from assumption.
- Stored entries record what was true WHEN WRITTEN: re-verify any file,
  flag, command, version or endpoint they name before acting on it.
- Surface conflicts between a stored truth and present observation instead
  of silently picking one.

The same clause ships in the `wyrm-mcp-edge` connector so local stdio and
remote surfaces give identical guidance.

## [8.7.3] - 2026-08-18 - Device revoke actually works

`wyrm cloud devices` prints 8-char truncated ids and tells you to run
`wyrm cloud devices revoke <device-id>`, but revoke passed that argument
straight to `DELETE /api/v1/devices/:id`, which answers `invalid_device_id`
(HTTP 400) for anything but a full uuid — and nothing printed full uuids.
Revoking a device was therefore impossible unless its uuid happened to be
sitting in a local file.

- `devices revoke` now resolves its argument against the real device list:
  exact uuid wins, otherwise an unambiguous **prefix** does. A pasted
  trailing ellipsis is stripped, and no-match / ambiguous cases exit with
  the candidates listed instead of a bare API error.
- New `wyrm cloud devices --full` prints whole ids for scripting.

## [8.7.2] - 2026-08-17 - Reverify residuals: docs say what the tool prints

Independent 20-seat re-verify of the 8.7.0 Crucible fixes: 18/20 GREEN,
2 wording residuals, both closed here. `wyrm-setup --help`, GETTING_STARTED,
the changelog and EGRESS now carry the one derived model-bundle size
(**138 MB**, from the sha256 pins) instead of a hand-typed ~137; GETTING_STARTED
says `wyrm upgrade` *offers* the one-time re-embed and that `wyrm report`
prints a block (it never wrote a file). No code behaviour change.

## [8.7.1] - 2026-08-16 - Review queue: an operator auto-approve switch

`wyrm review --auto on|off|status` (persisted in `wyrm_meta`, or
`WYRM_AUTO_APPROVE=1` per process) makes every future write recall-visible
immediately instead of parking agent-extracted memory in the review queue.
Off by default: the queue exists so extracted memory is not trusted blindly;
the operator flips it deliberately. `wyrm review --approve-all` drains what is
already queued. Existing rows are never touched by the switch itself.

## [8.7.0] - 2026-08-16 - The default install actually works: vectors on, and you can see it

Meaning-based search used to be a manual, Ollama-only opt-in most installs
never took, so the default install ran keyword-only and nothing said so out
loud. This release ships local vector search on by default and makes every
automated behavior visible instead of silent.

### Added: Bundled local vectors, on by default

- `wyrm-setup` now downloads a bundled local embedding model
  (`nomic-embed-text-v1.5-int8`, 768d, ~138 MB, one time, no account) and
  turns on tier 2/3 ("local vectors") search automatically. No Ollama, no
  API key, nothing leaves the machine.
- The bundled model runs in its own vector scope (`nomic-embed-text-v1.5-int8`,
  distinct from Ollama's `nomic-embed-text`), so existing Ollama-embedded
  memories are never silently mixed with a different model's vector space.
  Measured on real LoCoMo: recall@5/@10 **60.0% / 72.0%**, within the ship
  gate of the published local-hybrid row (60.3% / 72.2%). Full methodology in
  `BENCHMARKS.md`.
- `wyrm vectors download` / `wyrm vectors status` manage the bundled model
  directly (progress reporting, retry, presence check) for anyone who skipped
  it during setup or wants to confirm it landed.
- `wyrm upgrade` is the guided path from tier 2 to tier 3 ("max recall",
  hosted NVIDIA NIM, 2048d): free API key, masked key entry, a live embed
  call to validate the key before anything is written, per-client conflict
  confirmation (never silently overwrites an explicit provider), and an
  optional one-time reindex of existing memories on the new tier.
- Existing Ollama setups keep working unchanged. Switching to the bundled
  tier re-embeds existing memories once, automatically.

### Added: Visible automation, the receipts you didn't have to ask for

Every automated behavior now says what it did, where you're already looking:

- **Statusline receipt.** The search tier (`▲ local vectors 2/3`, `▲ max
  recall 3/3`, …) and last-session capture count ride the existing statusline
  segment. `WYRM_RECEIPT=0` turns it off.
- **Session-start receipt.** The SessionStart hook's `systemMessage` now
  reports what it restored (or, on a cold start with nothing to restore yet,
  says so plainly instead of staying silent).
- **`wyrm_session_prime` receipt line.** The MCP tool response itself now
  opens with one line: search tier, memory count on file, and which
  auto-capture path is active (hooks on Claude Code, model-driven
  elsewhere), so an agent sees its own operating conditions on the very
  first call of a session, not just on hosts wired up with the CLI hooks.
- **`wyrm doctor`** gained a tier line (`tier 2/3 (local vectors)`) next to
  the existing embedding-provider check, with a `next: wyrm upgrade` nudge
  when sitting on tier 2.
- **`wyrm-setup`'s closing sequence** now runs real checks, not decoration:
  model download progress, then a compact health check (search tier, memory
  file presence, auto-memory hook install), and points to `wyrm doctor` /
  `wyrm report` for anything that needs a support artifact.
- **`wyrm report`** includes the resolved tier and embedding provider (never
  memory content or keys) as the one block to paste when asking for help.

### Added: Paid licenses can now be revoked (and quietly renewed)

A key that has already reached a machine used to be a perpetual fact: nothing
on the client could ever un-grant it. Now the client honours a **signed
revocation list** and paid keys are **time-boxed with a silent refresh**:

- `~/.wyrm/revocations.json`: an Ed25519-signed list fetched in the
  background at boot (bounded, never blocks startup, never throws). Only a
  signature-verified list is ever cached or written (0o600) — a forged list
  is ignored and cannot clear a good one (fail-closed on forgery); a
  stale-but-genuine list is honoured (fail-open on staleness).
- `hasFeature()` / `verifyLicense()` enforce revocation **and** expiry live,
  every call — not once at boot — so a revoked or lapsed key drops to the
  free tier mid-session without a restart. `wyrm license status` says
  *revoked* when that is the reason.
- Silent 90-day refresh: a paid key within 14 days of expiry (or lapsed ≤30
  days) is renewed in the background against `account.ghosts.lk` using the
  stored account token; a refused refresh (blocked/revoked account) prints one
  line and leaves the token alone. Perpetual keys never refresh. Nothing about
  the free tier or existing valid keys changes.

### Fixed

- `parseJsonWithComments` (the JSONC-tolerant client-config parser in
  `autoconfig.ts`) now tracks string-literal state before matching `//`/`/*`
  comment markers, so a config value containing a URL (`"https://..."`) is
  no longer corrupted by comment stripping.

### Fixed (post-merge live test, PR #88 — the terminal is as smart as the MCP lane)

- **`wyrm search` was FTS5-only.** It now runs the same hybrid
  (FTS ⊕ dense, RRF) recall the MCP `wyrm_recall` handler uses, so a
  meaning-adjacent query typed by hand finds the memory, with the same
  provenance footer (or a plain `keyword-only (no vector provider …)` line).
- **`wyrm capture` did not embed at write time.** The CLI process used to
  exit before the fire-and-forget index promise settled; captures now embed
  synchronously before the DB closes and say so (`embedded for semantic
  recall` / `not embedded yet (wyrm index rebuild will backfill)`).
- **`wyrm index rebuild` printed green when everything was skipped** (e.g.
  the bundled model was never downloaded). It now warns with the resolved
  reason and the exact fix.
- **Cold-start receipt** said "first session" even with memories on file;
  `wyrm rehydrate --json` now reports a `memories` count and the SessionStart
  hook reads `Wyrm: N memories on file. No previous session to restore.
  Auto-capture is on.` when the brief is empty but memory is not.
- `wyrm project add <name> --path <dir>` parsed the name as a path; the
  positional is now the name (a bare existing directory still works), and
  `wyrm capture --project X` for an unknown project hints how to create it.
- Setup banner showed a hard-coded `v3.0.0`; it now reads the installed
  version (falls back to `dev`, never a false number).
- Setup's failure-firewall line no longer names the product twice.
- `wyrm doctor`'s vector-index check is model-scoped: coverage is counted
  under the ACTIVE model, not an unfiltered `vectors` count that double-counted
  stale rows from a previous provider.
- `wyrm report` counts CURRENT truths only, matching `wyrm stats`.
- `wyrm doctor` / `wyrm report` stderr is clean of the DB layer's INFO noise.

### Fixed (release gate check (a): the receipt tells the truth across the process boundary)

- The statusline receipt, `wyrm doctor` and `wyrm report` resolved the search
  tier from **their own shell's env**. A provider is routinely configured in
  the MCP client's env block (that is exactly what `wyrm upgrade` writes, and
  what a vault-backed launcher injects), which a bare shell never sees — so
  the receipt read `local vectors 2/3` while recall was running NIM 3/3, and
  doctor told the operator to configure a provider that was already running.
  The server now publishes what it actually resolved (`wyrm_meta`
  `embedding_servers`, pid-keyed, dead pids pruned) and the three surfaces
  prefer the live server that owns their cwd (`· via running server (pid N)`),
  falling back to env resolution only when no server is up. Recorded for
  every resolution, `none` included.
- **`wyrm update` called a failed registry lookup "up to date".** A lookup
  that could not reach npm was cached for the full 24 h and then served as
  `Latest: unknown (offline?) · Update available: no · Already up to date` —
  the daily auto-update timer showed a green tick while the machine sat two
  releases behind. Now: a failed lookup keeps the last KNOWN version instead
  of erasing it, is retried after 30 minutes (not 24 h), and `wyrm update` /
  `wyrm_check_update` say **unknown — npm registry unreachable** (exit code 2
  from the CLI so a timer run reads as a failure). A stale-but-real "update
  available" still surfaces during an outage.

### Fixed (release-gate Crucible, 106-agent gauntlet, 20 confirmed → 0)

- **Bundled tokenizer now matches the pinned `tokenizer.json` normalizer.**
  The in-repo WordPiece pass kept accents (NFC), knew only ASCII punctuation
  and never isolated CJK, so `café` / `naïve` / any ideograph run / a word
  glued to a curly quote or em dash embedded as `[UNK]` or the wrong piece —
  off-distribution vectors for exactly the text agent-written memories are
  full of. It now applies BertNormalizer faithfully (clean_text, CJK
  isolation, lowercase → strip accents) and splits on Unicode punctuation;
  verified id-for-id against the reference `tokenizers` library. ASCII
  English is byte-identical before and after.
- `npm run eval:bundled` is a real ship gate now (non-zero exit outside
  −1.5 pts of the published 60.3/72.2 row; `--no-gate` to print only). Before
  this the gate lived in a comment.
- The npm tarball ships `scripts/hooks/wyrm-run-auto.mjs` — `wyrm-setup`
  installed it from the package but the `files` allowlist left it out, so
  npm installs quietly reported it missing.
- Client-config parser: trailing-comma removal is now string-aware, so a
  value like `"a, ]"` survives a rewrite (the old post-pass regex ran over
  string literals — silent config corruption).
- `wyrm upgrade` / `wyrm-setup` write their `.wyrm-backup` copies of client
  configs as 0600 (they were world-readable copies of files that carry keys).
- License refresh + revocation fetch bound the FULL body under one deadline
  (a dripping body used to hang `wyrm license`); the boot fetch only runs
  when a license key is installed (the free tier makes no `account.ghosts.lk`
  call), `WYRM_LICENSE_REFRESH=0` disables it, long-lived servers re-check
  daily, and `docs/EGRESS.md` lists every call. Revocation lists bind the key
  shape (`WRM-XXXX-XXXX-XXXX-XXXX`) so the signed canonical form is injective,
  and an older genuine list can no longer replace a newer one (`stale`).
- The bundled ONNX provider no longer memoises a rejected session load
  forever; a `.part` larger than the pinned size is discarded instead of
  dead-ending on HTTP 416.
- Honest copy: `wyrm license` says **REVOKED** / **EXPIRED** (with the key
  rows) instead of "free tier (no license key)"; `wyrm upgrade` no longer
  prints "Upgrade complete" when every client was skipped; `wyrm-setup`
  re-runs recognise already-installed hooks, only promise "context restored"
  when hooks exist, name the model download before starting it (`--no-model`
  / `--plain` documented), and close with the next step that matches the
  tier you are on; the session-start receipt's arithmetic includes failure
  patterns; the statusline "saved last session" counts recall-visible
  memories only; the prime receipt survives pre-8.7 cached fleet briefs;
  the hash test provider is no longer labelled "local vectors"; one model
  size label (138 MB, derived from the pins) everywhere; `wyrm vectors status` / `wyrm upgrade`
  read the live server tier like doctor does; CLI commands default the
  console log level to `warn` (no DB INFO chatter on `wyrm capture`).
- Docs: GETTING_STARTED describes the receipts as they actually render;
  README says "no memory data leaves your machine" and points at EGRESS.md.

### Compatibility

No schema change (still **v40**). Known advisory: `adm-zip` via
`onnxruntime-node`'s install-time unpacker (no attacker-supplied archive
reaches it; the fix would pin onnxruntime-node back to 1.21.1). Two new benchmark gates are committed,
live-run scripts (not CI jobs): `npm run eval:bundled` (bundled-provider
LoCoMo recall, gated within 1.5 pts of the published 60.3/72.2 local-hybrid
row) and `npm run bench:parity` (cross-provider cosine similarity between the
bundled and Ollama vector spaces, informational, not gated, since the two
spaces are model-scoped by design). Both run locally and at release time,
since both need a downloaded model. CI itself runs the wasm-lane module-load
check (the bundled provider imports and initializes cleanly with
`onnxruntime-node` absent, no model or network required) plus the full unit
suite. Existing installs with an explicit `WYRM_VECTOR_PROVIDER` are
unaffected; the bundled default only applies to the previously-silent
no-provider-configured case.
**TESTCOUNT**

## [8.6.1] - 2026-08-12 - Self-serve pricing in the README

Docs/discoverability only, no runtime change. The npm README pointed buyers at
"email ryan@ghosts.lk"; it now carries a Pricing section (Pro $29 / Team $199 /
Enterprise $499) linking the public self-serve checkout at
[account.ghosts.lk/pricing](https://account.ghosts.lk/pricing). No code changed.

## [8.6.0] - 2026-08-12 - Negotiated Recall + the Wyrm Lift benchmark

### Added — Negotiated Recall: clean-mode recall you steer

Recall was push-only: Wyrm injected prescriptive memory into every session with no
consent step, so starting something genuinely new meant fighting the anchor of how
you did it last time. Negotiated Recall adds a per-run recall policy.

- **`clean` mode** mutes prescriptive memory (`kind` pattern/heuristic, positive
  lessons) while ALWAYS keeping the failure firewall (anti-patterns, recorded
  failures) and ground truths. Building fresh never means flying blind past a
  known dead-end.
- **Enforced at every v1 recall chokepoint** — `wyrm_recall`, `wyrm_context_build`
  (default + budgeted), `wyrm_session_prime` (fleet + legacy). The pre-handler
  dispatcher cache that could serve a cross-run policy-blind brief is closed
  (`wyrm_context_build` removed from the response cache; a `dispatcherCacheable`
  carve-out for legacy prime).
- **Single-sourced mute rule** — `MUTED_KINDS`/`MUTED_OUTCOME` back both the SQL
  predicate (`mutePredicateSql`) and the JS classifier (`classifyArtifact`), so
  what clean mode hides and what `wyrm_recall_propose` shows are provably the same
  rule and cannot drift (property-tested across all kind×outcome combinations).
- **Two new tools**, hidden from the frozen default surface, callable by name:
  `wyrm_recall_set` (upsert the policy: mode, muted ids, challenged truths; merge +
  dedup) and `wyrm_recall_propose` (preview recall grouped into
  prescriptions / firewall / truths before anything is injected).
- New `recall_policy` table (migration 40). **Default behavior (no policy) is
  byte-identical to before** — the whole prior suite stays green.
- Deferred to fast-follow: CLI `wyrm ls` / `wyrm search` policy parity; the
  agent-facing negotiation hook + challenge reconciliation.

### Added — Wyrm Lift benchmark: memory as a model-capability amplifier

A 2×2 (model × memory) agent benchmark (`bench/lift.mjs`) proving where a cheap
model + Wyrm matches a frontier model + Wyrm on the tasks Wyrm has memory for.
Coverage-honest by construction: the headline is scoped to the covered slice, the
`by_coverage` denominator is always published, and `coverage:'none'` control
scenarios prove memory does not help where it should not. `--dry` mode is a real
regression gate (the thesis assertions fail if the memory plumbing breaks), with a
permanent leak-guard test so no scenario can name the tool in a model prompt.
Sibling to the Firewall Receipt — reuses its harness patterns, touches none of it.
Deferred to P2/P3: live runs (need `ANTHROPIC_API_KEY`), a blind LLM judge to
replace the oracle scorer, per-arm confidence intervals before any published
number, and the DragonSpark/ghost model slot (pending weights).

### Compatibility

Additive guarded migration to schema **v40** (an 8.5.x DB opens 100% rows). New
tools are hidden from the frozen 33-tool default surface, so the advertised surface
is unchanged. **2,347 tests** across **185 suites**, green.

## [8.5.8] - 2026-07-23 - Un-brickable storage: node:sqlite reachable + auto-fallback for a missing native binding

### Fixed — the node:sqlite backend was dead at runtime; wyrm now self-heals a missing better-sqlite3 binding

Wyrm ships a node:sqlite facade as its zero-native-code escape hatch (opt-in via
`WYRM_STORAGE=node`), but it was **unreachable in the compiled ESM dist**: the
availability probe used a bare `require('node:sqlite')`, which throws `require is
not defined` under ESM, so `WYRM_STORAGE=node` silently fell back to
better-sqlite3 and the whole backend was dead code. Combined with npm v12's
skipped native build, a missing better-sqlite3 binding meant a hard crash
(`Could not locate the bindings file`) with no way out.

- **createRequire fix** (`storage/index.ts`, `storage/node-sqlite-facade.ts`):
  both modules now build `require` via `createRequire(import.meta.url)`, so the
  node:sqlite probe and the facade actually work in the ESM dist. Verified in the
  real dist via a child-process test (`WYRM_STORAGE=node` now selects `node`).
- **Auto-fallback** (`openDatabase`): a better-sqlite3 **native-binding load
  failure** (missing/unbuilt addon, ABI mismatch) now falls back to node:sqlite
  instead of crashing the process, with a loud stderr note and the rebuild
  command. This is a **rescue, not the default** — better-sqlite3 stays primary
  for everyone whose binding loads, and only a binding failure triggers it (a
  real SQL/IO error still surfaces).
- **Facade validated** (resolves an internal "incomplete subset" concern): the
  entire suite forced onto node:sqlite passes **2149/2150** — the lone gap is
  SQLITE_BUSY retry fidelity under heavy concurrent daemon writes, an acceptable
  degradation for a rescue path whose alternative is a dead process.
- New `tests/storage-node-fallback.test.ts`; also clears two long-standing
  `'require' is not defined` lint errors.

## [8.5.7] - 2026-07-23 - Named validation errors at the capture/failure write boundary

### Fixed — two raw TypeErrors where a named client error belongs (faeforest field report, 2026-07-20)

A raw TypeError is indistinguishable from "the tool is broken" — the caller
can't tell their own mistake from a defect, so both get reported as defects.
Both reported crashes now return clean, field-named `WYRM_VALIDATION` errors:

- **`wyrm_capture` / `wyrm_remember` `tags`**: the comma-separated string
  encoding (`tags: "a, b"`) crashed deep in storage with `.join is not a
  function`. Tags are stored comma-joined, so the string form is lossless —
  it's now **coerced** to the array it obviously means (new `asTags`
  boundary validator); any other non-array is a named error.
- **`wyrm_failure_record`**: a missing `target`/`description` crashed in
  `record()`'s normalizer (`Cannot read properties of undefined, reading
  'toLowerCase'`); a missing `scope` surfaced as a raw SQLITE `NOT NULL`;
  and an out-of-enum `scope` was silently **inserted past the schema**.
  All three are now named validation errors listing the allowed values —
  the same contract `wyrm_failure_check` already had.
- Regression tests: `tests/capture-tags-validation.test.ts`,
  `tests/failure-record-validation.test.ts`.

## [8.5.6] - 2026-07-23 - `wyrm update` survives npm v12 (allow-scripts on self-update)

### Fixed — self-update no longer bricks wyrm on npm v12+

npm v12 changed the default to **deny dependency lifecycle scripts** (an
allow-list security feature; `ignore-scripts` is still `false`). So `wyrm
update` — which shells `npm install -g wyrm-mcp@latest` — installed the new
version but left `better-sqlite3`'s native binding UNBUILT, and wyrm then died
on the next DB op with `Could not locate the bindings file`. This bricked wyrm
for every user on modern npm the moment they updated.

- **`wyrm update` (CLI) and `wyrm_self_update` (MCP tool)** now pass
  `--allow-scripts=wyrm-mcp,better-sqlite3` so the native addon compiles during
  the update. Harmless/ignored on older npm (scripts run there by default).
- New source-lock test `tests/self-update-allow-scripts.test.ts` keeps the flag
  from silently regressing out of either path.
- **Docs**: `TROUBLESHOOTING.md` gains a section for the npm-v12 symptom (a
  *successful* install that then can't open the DB), and the README notes the
  `--allow-scripts` first-install command for npm v12 users.

Note: a separate latent issue — the opt-in `WYRM_STORAGE=node` backend is
unreachable in the ESM dist (a bare `require('node:sqlite')` throws) — is left
for a dedicated, facade-validated pass; the node:sqlite facade implements only
the API subset the tests exercise, so it is not yet a safe automatic fallback.

## [8.5.5] - 2026-07-23 - Portable, self-healing wyrm resolution in the Claude Code session hooks

### Hardened — Portable, self-healing `wyrm` resolution in the Claude Code session hooks

The npm-global Claude Code hooks (`scripts/hooks/wyrm-*.mjs`) shelled a bare
`wyrm`, so any environment that spawned a hook with a minimal `PATH`
(containers, CI, cron, non-login shells) silently disabled auto-memory —
rehydrate returned an empty brief; capture / run-auto / prune became no-ops —
with no error, because a memory hook must never break a session.

- **rehydrate, capture, prune, run-auto** now resolve `wyrm` portably: try the
  bare command first (zero overhead where `PATH` is set), then fall back to the
  npm/user CLI-install locations that actually exist on the machine
  (`~/.npm-global/bin`, `~/.local/bin`, `node_modules/.bin`, `/usr/local/bin`,
  `/opt/homebrew/bin`).
- Deliberately **not** `/usr/bin`: a distro or global scoped-package can symlink
  an unrelated `wyrm` there (e.g. the HTTP-server entrypoint), and resolving to
  the wrong binary is worse than not resolving — it returns success with empty
  output and silently breaks the hook.
- Output-identical where `wyrm` is already on `PATH`; a truly unreachable `wyrm`
  still degrades to a clean no-op (exit 0). No behavior or token change for a
  standard install — the committed token-economy baseline is unchanged.
- New source-lock suite `tests/hook-wyrm-resolution.test.ts` asserts every
  session hook resolves through the fallback resolver, never a bare `wyrm`, and
  never lists `/usr/bin`.

## [8.5.4] - 2026-07-23 - Prime resolves by cwd, not machine-wide latest-active

A bare `wyrm_session_prime` (no `project_id`/`project_name`) resolved its
project as "whichever project wrote the newest session row, machine-wide" —
so a session on ANY other project silently hijacked every subsequent bare
prime on the box. Live repro: after a wyrm-cloud session, priming from
`/home/you` returned wyrm-cloud (6 truths, 0 quests) instead of kami
(22 truths, 5 quests) — reading as "Wyrm didn't prime".

- `resolvePrimeProject` now walks the server's cwd first via
  `resolveProjectForPath` (the same resolver wyrm-guard and `failure_record`
  already trust) — the MCP server is spawned in the session's project dir, so
  the walk lands on the right project. The latest-active fallback survives
  only for unenrolled directories.
- Explicit `project_id` / `project_name` are unchanged and still win.
- Covers both the default and fleet prime paths (shared helper).
- New regression suite `tests/prime-cwd-project.test.ts` (hijack case,
  fallback preservation, explicit-name precedence).

## [8.5.3] - 2026-07-20 - Cloud: quarantine-and-skip undecryptable deltas (one bad push can't wedge the fleet)

Sovereign pull used to stop FOREVER on a delta it could not decrypt or apply
(`break pullLoop`, a sticky stop that never advanced the cursor). A single
permanently-undecryptable delta — a peer that pushed rows encrypted with the
wrong master key — therefore halted that device's pull, and with it the whole
fleet's shared memory, indefinitely (one device stalled 25+ ticks on a single
`skill:433` after a mis-keyed push).

- A failing delta now gets 2 sticky retries (enough to ride out a transient row
  ordering or partial-write blip), then is set aside in the cursor's quarantine
  map and SKIPPED so the cursor advances past it. A later correct re-push arrives
  as a new delta id and applies normally; the poison delta stays quarantined for
  visibility and is surfaced in the sync summary and daemon tick log
  (`Quarantined: N`).
- One mis-keyed push can no longer wedge the fleet's sovereign sync.

## [8.5.2] - 2026-07-20 - Cloud: loud key-mismatch at login, validity-gated re-login, daemon interval carry

Three fixes from a fleet field report (a second machine joining an org account),
all in the sovereign cloud CLI:

- **Login fails loud on a key it cannot read.** `wyrm cloud login` used to report
  full success even when the local master key could not decrypt the account's
  existing deltas (joining an org whose data was encrypted with a different key).
  Every surface said "connected" while sync moved 0 rows in both directions, the
  only evidence buried in a later daemon tick. Login now pulls one existing delta
  and test-decrypts it; on failure it says plainly that this device cannot read
  the account yet and names the fix (`wyrm cloud recovery restore`).
- **Re-login gates on session validity, not file existence.** An expired or
  revoked session made `login` refuse with "already logged in" while `status` on
  the same session reported "expired", so it could neither be used nor replaced.
  Login now probes the session; only a still-valid one blocks, a definite 401
  clears the stale file and proceeds, and an ambiguous (network) error preserves
  the original guard.
- **Daemon restart carries its interval.** `wyrm cloud daemon restart` reverted
  to the 10-minute default instead of the interval it was running at. The manager
  now persists `interval_ms`; restart carries it (an explicit `--interval` still
  wins), and `status`/`restart` print the interval.

## [8.5.1] - 2026-07-19 - White-label the dashboard; keep operator identity out of the package

Wyrm ships to other people, so the dashboard must carry NO operator-specific
identity out of the box. 8.5.0 baked one operator's branding into the shipped
code; 8.5.1 makes the package neutral by construction.

- **Branding is white-label by default (`src/branding.ts`).** The shipped code
  names no company, logo, or internal concept. Product wordmark, logo, and the
  fleet view's labels/copy resolve ONLY from a private `~/.wyrm/brand.json` (or
  env) at runtime — so a reseller who never configures gets a neutral board,
  and an operator's identity can't land in the public tarball by omission. The
  fleet tab is opt-in (`WYRM_UI_FLEET` on|off|auto; auto shows it only when a
  fleet grove exists), and is generic "Fleet Channel" unless an operator names
  it.
- **Operator logo no longer in the package.** Dropped the bundled brand SVG
  from the `files` whitelist; an operator supplies their own via
  `brand.json.markUrl`. (8.5.0 and earlier shipped it.)
- Removed the last operator-identifying strings from `dist` (verified by scan).

> **Note:** 8.5.0 is superseded — it shipped operator-branded strings + logo.
> Upgrade to 8.5.1.

## [8.5.0] - 2026-07-19 - The War Room dashboard + brand type finally loads

A live ops view for the fleet war room, and the fix that makes the whole
dashboard wear its own typeface.

- **War Room tab in the web dashboard (`/ui`).** A real-time board for the
  `fleet-warroom` grove: the signature **loop pipeline** (findings → claimed →
  fixed → verified) with flowing connectors, a **presence board** (which fleet
  agents are alive, pid-liveness LED), the **effectiveness strip** (repeats
  blocked, tokens saved, rows shared — straight off the 8.4.0 recorder), and a
  type-coded **activity feed** (finding / request / receipt / verdict, with
  agent attribution and relative time). New `GET /ui/warroom` endpoint,
  failure-isolated per section; degrades to "grove not found" on a device
  without the grove.
- **Fix: the dashboard's brand fonts never loaded.** The response-header CSP
  omitted the Google Fonts CDNs the dashboard `<head>` intends, so every view
  silently fell back from Geist + JetBrains Mono to system sans. `style-src`
  now allows `fonts.googleapis.com` and `font-src` allows `fonts.gstatic.com` —
  the Stealth-Silver typography actually renders now.

## [8.4.0] - 2026-07-19 - The flight recorder: effectiveness + quality over time

`wyrm doctor` (8.3.0) answers "what's off right now"; nothing answered "is it
getting better or worse". This adds the longitudinal ledger — a timestamped
trend so a slow degradation (recall drifting as the corpus outgrows the index,
vectors quietly off for days, a defect class recurring) is legible instead of
something you only notice when you re-run a tool by hand.

- **`metrics_snapshots` table (migration 37)** — one row per (kind, capture):
  - **health** — corpus size, vector coverage, FTS drift, review-queue depth,
    unresolved failures, the resolved provider, and doctor-parity check bits.
  - **effectiveness** — did memory actually *help*? failure blocks (repeats
    prevented), rehydrate/recall/capture call volumes, and estimated tokens
    saved (recovered context + blocked retries).
  - **retrieval** — recall@1 / recall@10 / MRR from the eval bench, tagged with
    the release, so a version that drops recall is caught.
- **`wyrm metrics <snapshot|trend|show>`** — `snapshot` writes health +
  effectiveness (cheap, offline); `--full` adds retrieval (runs the bench).
  `trend` prints the series with **regression/degradation flags** ("vectors OFF
  for 4 consecutive snapshots", "review queue growing", "recall@10 REGRESSED
  4.0pts between 8.3.0 and 8.4.0", "+2,400 tokens saved since last snapshot").
- **Nightly recorder** — a systemd timer captures health + effectiveness at
  23:50 under the server's vault-injected provider env (so `provider` is
  accurate, and no NIM tokens are spent — health/effectiveness never embed).
- Failure-isolated by construction: a missing table or busy DB degrades ONE
  metric to null; the recorder never throws into its caller.

## [8.3.1] - 2026-07-19 - Crucible pass on 8.3.0: the honesty surface, proven honest

An 87-agent adversarial Crucible over 8.3.0 (every finding cross-examined by 3
independent skeptics) caught a CRITICAL regression in the surfaces meant to END
silent degradation, plus a chain of correctness/leak fixes. All re-verified.

- **CRITICAL: resolver and factory could disagree on their default arm.**
  `resolveEmbeddingState()` reported `none` for an unrecognized
  `WYRM_VECTOR_PROVIDER` while `createProvider()` built `LocalHashProvider` — so
  any typo/case/whitespace value (`NIM`, `nim `, `bogus`) made every status
  surface say "vectors off" while the server silently indexed non-semantic
  hash-384 vectors: the exact status≠reality class 8.3.0 exists to kill,
  reintroduced through mismatched defaults. Fixed with a shared
  `normalizeProviderKey()` (trim+lowercase) and a factory default of
  `NoneProvider` (never a silent local-hash). Regression-tested.
- **Recall receipt `ftsOnlyReason` now truthful.** `vector-init` built a store
  even for `auto`, so recall mislabeled vectors-disabled as `empty_index` /
  `embed_error`; the `indexStale` state was unreachable. Now `vector-init` gates
  on the *resolved* provider, and recall consults the provider name +
  `indexCoverage()` and adds a `stale_index` reason.
- **Capabilities no longer conflates deprecation with degradation** — a working
  (deprecated) ollama is reported advisory, not "recall FTS5-only".
- Scheme-less `WYRM_NIM_BASE_URL` → `nim_bad_base_url` (was read as healthy
  local); `OllamaProvider.embed` bounded by a timeout like the others; `local`
  is an honest advisory (no dangling `Fix:`); the capabilities briefing header
  is pure ASCII on the default wire.
- **CLI:** a valueless `--note`/`--name` no longer crashes with a raw
  better-sqlite3 TypeError; `index --help rebuild` prints usage instead of
  running status.
- **Session hooks:** a literal-`null` stdin payload no longer breaks the
  "always exit 0" contract; compact-continuation / meta transcript lines no
  longer become the session objective; an objective starting with `--` is
  preserved; the recency query honors `WYRM_DB_PATH`.

157 suites / 2107 tests green. Two adversarial false positives were correctly
rejected by the skeptic panel. Confirmed defects captured to Wyrm negative memory.

## [8.3.0] - 2026-07-18 - The honesty release: wyrm doctor, no silent degradation, NIM-only embeddings

Driven by tallow's silent-degradation field report (2026-07-18): subsystems could
degrade quietly while every status surface kept reporting healthy — "the call
succeeds, does less than the caller believes, and returns something that looks
like success." Writes got receipts in 7.9.0; this release gives reads and config
the same treatment, and settles the embedding-provider question: NVIDIA NIM is
the supported provider, Ollama is out of the auto chain.

- **`wyrm doctor` — the health check that refuses to flatter.** Six checks
  (resolved embed provider, vector index vs corpus, FTS integrity, failure
  firewall, review queue, schema), each printing OK or DEGRADED with the exact
  fix command; anything degraded exits non-zero. A subsystem that is off now
  LOOKS off.
- **Ollama leaves the auto chain.** The old `OllamaAutoProvider` silently
  delegated to a no-op when Ollama was absent — the root of "semantic search
  never ran on this machine". `auto` now resolves honestly to `none` and every
  surface says so with the NIM fix line. Explicit `WYRM_VECTOR_PROVIDER=ollama`
  still works but deprecation-warns and leaves in the next major.
  (`auto` still never selects a remote provider: embedding memory text
  off-machine stays an explicit egress decision, same posture as grove sync.)
- **`resolveEmbeddingState()`** — one pure, env-derived source of truth
  (`resolved` / `reason` / `fix` / `egressHost`) read by every honesty surface.
- **`wyrm_capabilities` stops lying.** The runtime line previously hardcoded
  "ollama embeddings" regardless of reality; it now reports the RESOLVED
  provider and appends a degradation warning + fix whenever vectors are off.
- **`wyrm_session_prime` vector-health advisory.** A fresh session is told,
  in the brief itself, when recall is running at the FTS-only floor
  (~59.9% vs 72.2% hybrid recall@10) or the index is empty — with the fix.
- **Recall receipt: `ftsOnlyReason`** (the 2026-07-15 token-path audit,
  defect 4). `embedModel: null` no longer conflates "vectors disabled",
  "no key", "empty index", and "embed error" — the receipt says which.
- **CLI: `wyrm failure <list|resolve>`.** `wyrm_failure_resolve` routes but is
  not advertised on the v8 32-verb surface, so a fixed failure had no
  discoverable off-switch. Now it does.
- **CLI: `wyrm project <add|list>`.** Projects can be registered without a
  session-start side effect.
- **Fix: `wyrm index rebuild --help` actually ran the rebuild.** `--help` on
  the `index` verb now prints usage, never executes.

## [8.2.0] - 2026-07-17 - Sovereign auto-sync daemon + the sync that never settled

Sync correctness. 8.1.0 surfaced the sovereign path; wiring up a hands-off daemon
for it exposed a real bug in the delta engine that had been quietly making every
sync slow.

- **Sovereign auto-sync daemon.** `wyrm cloud daemon <start|stop|restart|status>`
  (and the `wyrm_cloud_sync` `sovereign-daemon-start`/`-stop`/`-status` actions)
  run the sovereign per-row E2E sync periodically, so cross-device memory is
  hands-off rather than a manual `wyrm cloud sync`. It is a thin scheduler that
  spawns the same `wyrm cloud sync` CLI per tick — process-isolated, so a tick
  never leaks the sync engine's DB connection and a tick crash can't take the
  scheduler down. Single-instance via its own PID file, distinct from the legacy
  snapshot daemon.
- **Fix: sovereign sync never settled (migration 36).** `collectChangedRows`
  only applies its `updated_at > cursor` incremental filter to tables that HAVE
  an `updated_at` column. `quests` and `design_references` didn't, so every sync
  re-collected ALL their org/public rows, stamped each with `Date.now()`, and
  re-pushed them — forever. With the push endpoint costing ~1s/row server-side,
  a full quest table made every sync take minutes. Migration 36 adds `updated_at`
  (backfilled) to both. **Measured: 106-rows-every-sync (minutes) → 0 rows /
  1.5s steady state.**
- **No triggers — app-code stamping.** Both tables are external-content FTS5,
  where a self-`UPDATE` inside a trigger corrupts the FTS index when an indexed
  column changes ("database disk image is malformed"). `updated_at` is stamped in
  app code on every write (`addQuest`/`updateQuest`/`updateQuestFields`,
  `design.ts`) — the convention the four already-correct synced tables use.
- **No silent drops.** `collectChangedRows` now COLLECTS rows with a NULL
  `updated_at` (ordered last) instead of filtering them out, so a row that ever
  misses its stamp re-pushes loudly rather than vanishing from sync.
- **Push chunk 50 → 25** so each request stays well under the client's 60s abort
  even at 2x server latency (a 50-row chunk measured ~47s).

Migrations at v36. 157 suites / 2,102 tests.

## [8.1.0] - 2026-07-17 - Sovereign sync surfaced, entity graph, delete propagation

Coherence. The post-v8 health pass named one real architectural incoherence (two overlapping
cloud systems) and one memory gap (the knowledge graph existed but nothing populated it). Both
are closed here, alongside the sovereign-sync GA correctness blocker.

- **Sovereign cloud sync is now reachable from MCP (quest #972).** The `wyrm_cloud_sync` tool
  drove only the weaker whole-DB snapshot daemon (System B: last-write-wins by mtime); the
  sovereign zero-knowledge, per-row AES-256-GCM E2E delta sync (System A) was CLI-only, so an
  agent asking for "cloud sync" got the weak path. `wyrm_cloud_sync` gains `sovereign`,
  `sovereign-preview`, and `sovereign-status` actions that drive the sovereign path (process
  isolated over the same `wyrm cloud …` CLI, no second WAL write-connection). Its description
  now leads with the sovereign path; `wyrm_cloud_backup` and the snapshot daemon are re-labeled
  as the legacy snapshot, fenced off from the sovereign claim. No new raw tool.
- **Deletes propagate (migration 34).** The sovereign sync engine had a `recordTombstone` with
  zero callers, so a delete on one device never reached another and could resurrect on pull.
  AFTER DELETE triggers on the six synced tables now record tombstones, which `runSync` already
  pushes as encrypted deltas.
- **Private groves leak no deletion metadata (migration 35, quest #113).** Migration 34 recorded
  a tombstone for *every* delete; a private grove (the default, incl. GHOST PROTOCOL) or a
  within-visibility row would push content-free deletion metadata to the cloud. The triggers are
  now grove-gated with a `WHEN` clause that mirrors the upsert egress gate exactly: a tombstone
  records only when the row itself would have synced.
- **Entity auto-extraction (the graph recall leg now fires).** Deterministic entity extraction
  (proper-noun spans, code-shaped tokens, known tech, with UUID/hash/random-token rejection)
  populates the `entities` + `relationships` graph on capture, plus a `wyrm entities
  backfill|stats` CLI. The v8 graph recall leg was inert without it; on a populated project the
  FTS entity lookup replaces the old arbitrary first-500 scan so it scales.
- **Metabolize auto-safe lanes default-on.** Exact-hash duplicate supersede (canonical kept,
  never deleted, sync-safe) and gentle bounded decay of stale auto-captures now run in
  maintenance by default. Near-duplicate queuing stays opt-in (`WYRM_METABOLIZE=1`);
  `WYRM_METABOLIZE=0` disables the auto-safe lanes.

Migrations at v35. 156 suites / 2,096 tests.

## [8.0.1] - 2026-07-17 - Injection quarantine hardening (the garak red-team)

Security. The v8 premortem's one named residual (#965) — the full garak red-team against a
live brief render — was run, and it found a real gap: the hand-authored injection detector
caught only **3.4%** of the real 622-payload garak 0.15.1 corpus (latentinjection +
promptinject + dan + in-the-wild jailbreaks). The DAN-family role-reassignment and persona
jailbreaks evaded it entirely.

- **Defense-in-depth quarantine.** The fix is architecture, not a bigger regex (a regex
  provably can't enumerate every jailbreak): **`untrusted`-lane content is now CATEGORICALLY
  withheld from context briefs** — its bytes never become agent-read context, whatever they
  contain (0 escapes by construction against the full corpus). It stays fully searchable and
  recallable, marked untrusted; only the assembled brief is guarded. `imported`-lane content
  (git/pr/rules) remains detector-gated + ⚠-marked.
- **Detector hardened** from 3.4% → **80.7%** on the real corpus with **0 false positives** on
  benign engineering prose, now covering the real attack classes (instruction-override,
  role/persona reassignment, DAN/jailbreak, coercion/termination threats, goal-hijack,
  output-format hijack, credential/exfil, command-exec, fictional-character framing,
  encoding-smuggling — morse/base64/unicode-override).
- **`bench/injection-redteam.mjs` + `tests/injection-redteam.test.ts`**: the red-team is now a
  committed, offline CI gate (curated structural corpus; point it at a fresh garak export for
  the full 622-payload deep run). The gate's hard invariant is untrusted-lane escapes = 0.

Migrations unchanged (v33). 153 suites / 2,077 tests.

## [8.0.0] - 2026-07-17 - PROOF: memory you can prove

The v8 "PROOF" major. Started from an operator report — "I feel like Wyrm has gaps, Claude
forgets after a while" (2026-07-15) — and a premortem of eight ways v8 could fail. Shipped
across 7.9.0–7.12.0 and cut here. The through-line: **every claim Wyrm makes is now provable
— receipted, gated, or measured — and the one place it isn't is written down, not hidden.**

- **Receipts (F1).** Every memory write returns and ledgers a structured receipt (stored /
  queued / merged / aliased / dropped + why); `wyrm digest --writes` reconstructs any day
  offline; a synthetic-day reconciliation gate fails CI on an unexplained drop.
- **Bridge (F2).** `wyrm bridge <init|status|run [--watch]>` — remember / render / reverse
  substrate lanes from one receipted config, adopting the legacy distill state; harness
  files become compiled views of Wyrm. Re-import content-hash guards on every import verb.
- **Metabolism (F3).** `wyrm metabolize` — exact-hash dupes auto-merge by supersede (never
  delete, sync-safe), near-dups queue as review candidates, gentle bounded decay; truth-
  shape lint; the brief-SLO gate (which caught a real brief-budget bug on first run).
- **Numbers (F4).** Recency + temporal + usefulness + knowledge-graph recall legs (all
  identity when absent, all env-tunable); three CI-gated benches; `docs/RETRIEVAL-NUMBERS.md`.
- **Trust fabric (F5).** `source_trust` provenance lanes (operator > agent > imported >
  untrusted), marked at every brief render; brief-surface injection quarantine; process-
  identity-first presence everywhere; opt-in probationary write lanes for new fleet agents.
- **Known residual risk (honest):** the full 559-payload garak red-team against a live brief
  render (#965) is a crucible-harness job, not run here — a test-level corpus gates CI. Do
  the full run before any public launch. See `specs/wyrm-v8/premortem-2026-07-15.md` §re-read.

152 test suites / 2,071 tests. No breaking API/tool changes vs 7.x — the 8.0.0 bump marks the
completed PROOF program, not a surface break; migrations climb to v33 (additive).

## [7.12.0] - 2026-07-16 - v8 F4+F5: retrieval numbers + trust fabric

- **Temporal recall** (v8-A4): an explicit time expression in the query — "yesterday", "today", "last week", "last N days", "N days ago", an ISO date — soft-filters recall toward the queried window (`WYRM_RECALL_TEMPORAL_WEIGHT`, default 0.5; a floor, never an exclusion, so a decisively better out-of-window match survives). No time expression = byte-identical ordering.
- **The feedback loop is closed**: memory whose reuse `wyrm_feedback` confirmed useful earns a bounded boost (up to 1.15×, volume-damped, never punitive — `WYRM_RECALL_USEFULNESS_WEIGHT`).
- **`docs/RETRIEVAL-NUMBERS.md`**: the citable two-tier LoCoMo table (real set: FTS floor r@10 59.9%, hybrid 72.6%, MRR 0.448), the freshness eval, the discovery gate — one reproducible page (draft, pending operator sign-off).
- **Trust fabric** (v8-A5, migration 33): every artifact carries a `source_trust` lane (`operator > agent > imported > untrusted`), derived conservatively from the writer (nothing becomes operator by accident). Imported/untrusted provenance is MARKED at every brief render, and instruction-shaped content from a non-first-party lane is quarantined out of briefs entirely — the same text from the operator passes (a trust lane, not censorship).
- **Presence is process-identity-first everywhere** (v8-A6): `liveAgents` and the reaper now key on pid+start-time — an idle pid-alive agent never reads dead, a pid-dead one is reaped before its TTL.
- **Import lanes finished** (#939/#947): pr re-import guard (`pr:` signatures), rules imports truth-shape-linted and receipted, harvest drops accounted.
- FEATHERWEIGHT re-baselined with attribution (S2 delta = ranking now selects different corpus items + capture receipts; the ≤8K ListTools pin holds).

## [7.11.0] - 2026-07-16 - v8 F3: memory metabolism (consolidate, decay, lint, gate)

- **`wyrm metabolize [--dry-run] [--project] [--no-near]`** (v8-A3): the digestion pass, receipted end-to-end. Exact-hash duplicates auto-merge by **supersede — never delete** (deletes don't tombstone through cloud sync; supersede propagates and recall already hides it). Near-duplicates (deterministic token-shingle Jaccard) queue as review merge-candidates, idempotent via `nd:` signatures — summarize-then-judge, never auto-applied. Gentle bounded decay for old, never-reused `[auto:*]`/bridge captures only (factor 0.9/run, floor 0.3; operator memory never decays). Maintenance lane behind `WYRM_METABOLIZE=1`.
- **Truth-shape lint.** `wyrm_truth_set` queues config-assignment shapes (`no_xss = true`) for review instead of storing them as bare truths — the 2026-07-11 41-pseudo-truth dump is structurally closed (`WYRM_NO_TRUTH_LINT=1` bypass).
- **Fix: the context-brief budget was spent kind-major** ('pattern' first), so a flood of `[auto:*]` digests could exhaust `wyrm_context_build`/prime's memory brief before a single distilled lesson got a slot — silently defeating the 2026-07-12 provenance downrank. The brief now selects its top-N in penalty-sorted global order BEFORE kind grouping. Found by the new brief-SLO CI gate on its first run.
- **Brief-SLO gate**: a committed junk/signal fixture asserts the capped prime brief still carries the signal (≥5/6 truths, ≥3/6 lessons against a 4× junk flood).

## [7.10.0] - 2026-07-16 - v8 F2: wyrm bridge (the substrate flow, orchestrated)

- **`wyrm bridge <init|status|run>`** (v8-A2, one-substrate-many-renders): one configured, receipted runner for the substrate lanes. `remember` — daily operator notes → sectioned, hash-deduped, captured direct (operator trust lane; the legacy distill script's state is ADOPTED on first run so migration never re-captures). `render` — composes the guarded v7 F4 pipeline (human edits inside managed regions are harvested to review BEFORE overwrite). `reverse` — sweep-only ingest of native-file edits → provenance-tagged review candidates. Config `~/.wyrm/bridge.json`, state `~/.wyrm/bridge-state.json`, every action in the write ledger.
- **`wyrm_import_git` re-import guard** (the 2026-07-08 duplicate-import scar, closed): each commit carries a `gi:` content-hash signature; importing the same history again is a no-op with per-commit drop receipts. Imports are also F1-receipted (queued under the imported trust lane unless auto-approved).
- Scope notes: render write-event triggering and the pr/rules import lanes remain tracked-open (#945, #947/#939).

## [7.9.0] - 2026-07-16 - v8 F1: write receipts (nothing silently lost)

- **Write receipts (v8-A1).** Every memory write answers for itself: a structured receipt (`stored` / `queued` / `merged` / `aliased` / `dropped` + why) rides the response of `wyrm_remember`, `wyrm_capture` (including the conflict-queue path), `wyrm_auto_capture` (dedup drops accounted, not invisible), `wyrm_quest_add`, and `wyrm_truth_set` (supersede cascades noted) — and lands in the new local `write_ledger` (migration 32; never replicated, maintenance-pruned via `WYRM_WRITE_LEDGER_DAYS`, default 60d).
- **`wyrm digest --writes [--since N] [--json]`.** The offline "what did Wyrm learn, queue, and refuse" report: outcomes by tool and source, non-stored reasons, review-queue depth. Deterministic, no network.
- **`wyrm capture` CLI prints a trailing `receipt:` line** so hook callers (session capture, the .remember distill) can log what actually happened to their write.
- **`wyrm_stats`** now shows the last 24h of write outcomes and the review-queue depth (queued memory is invisible to recall — a silent queue reads as lost memory).
- **Synthetic-day reconciliation gate.** A scripted day of captures must reconcile to the ledger with zero unexplained non-stored rows — the write-side analogue of the golden-transcript corpus, enforced in CI.
- **Freshness eval (v8-A4).** `bench/freshness.mjs` + CI gate: the recency prior's own number — fresh-wins@1 16/16 at the shipped default vs 1/16 disabled; decisively-better stale matches preserved 4/4; anti-softening check included.
- **Fix: claim legibility.** A quest claim by an agent not on the presence board now says "announce first" instead of reporting a phantom "already claimed by another agent" (the FK failure was swallowed into the CAS-conflict path).
- **Fix: heuristic-conflict advisory never fired** — its FTS query MATCHed a table alias (throws "no such column"), silently swallowed since introduction. Full-table-name MATCH restores the advisory.
- **v8 "PROOF" spec-kit** committed under `specs/wyrm-v8/` (premortem, constitution v8-A1…A7, spec, plan, 34 tasks).

## [7.8.0] - 2026-07-15 - Recall recency prior + the rehydrate write side

- **Recall recency prior.** `recallHybrid` / `recallHybridGlobal` (the default `wyrm_recall` / `wyrm_search` artifact path) re-weight fused scores with a bounded time-decay multiplier `1 − w + w·2^(−age/halfLife)` — default weight 0.25, half-life 30 days. Fresh memory now out-ranks equally-relevant stale memory; the `(1 − w)` floor keeps a decisively better old match on top. Tune with `WYRM_RECALL_RECENCY_WEIGHT` (0 disables and restores the historic ordering exactly) and `WYRM_RECALL_RECENCY_HALFLIFE_DAYS`. Root cause it fixes: month-old lessons scored relevance 1.0 on queries about yesterday's work.
- **`wyrm session log`.** New CLI verb — the WRITE side of `wyrm rehydrate`. Upserts a per-project session record keyed by `--run <harness session id>` (`--objectives/--completed/--issues/--commits/--notes`), so SessionEnd/PreCompact hooks keep the continuity brief fresh instead of replaying the last manually-recorded session forever. Exits 0 when the directory maps to no project (a memory hook must never break a session).
- **Fix: snake_case direct-capture fields no longer silently dropped** (tallow field report, 2026-07-14). `wyrm_remember` / `wyrm_capture mode:'direct'` now accept `validated_fix` / `why_it_worked` as aliases of `validatedFix` / `whyItWorked` — the DB column, the outputSchema echo, and the mode-guard error all use the snake_case spelling, so following them used to save an empty-shell artifact while the write looked fine. `whyItWorked` (either spelling) now genuinely persists through `wyrm_capture` too.
- **Fix: validation-error template glue.** `expected` no longer reads "Correct the 'mode' argument — it you passed …" — the "it" is only prepended to validate.ts-shaped details ("must be one of …"), not to hand-written details that carry their own subject.
- **Docs: `wyrm events` now appears in `wyrm --help`** (it worked but was undiscoverable).
- **`wyrm presence` CLI** (tallow ask, 2026-07-14): `announce` / `list` / `release` from the shell, so Claude Code hooks can finally reach the presence board. Liveness is process-identity-first — `--pid <n>` records `{pid, pid_start}` from `/proc` and the agent is alive iff that exact process still exists (a TTL is a clock: it marks idle-but-alive agents dead; the start time stops a recycled pid from impersonating a dead one). TTL remains the fallback for rows without a pid, and `list` shows every row with a per-row verdict instead of hiding expired ones.
- **`wyrm_session_prime` review-queue advisory** (premortem follow-up): when captures sit at `needs_review=1` — which hybrid recall filters out — prime now says so ("N captured memories await review"), so queued memory is never mistaken for lost memory.

## [7.7.0] - 2026-07-12 - NVIDIA integrations, the Firewall Receipt, sharper long sessions

- **The Firewall Receipt.** `npm run bench:firewall` emits a reproducible, hash-stable attestation (`bench/firewall-receipt.json`) that the negative-memory firewall holds: 100% recall on repeated mistakes, 100% precision on novel actions, sub-millisecond, deterministic, no LLM, no cloud. Gated in CI (a regression fails the build), and a test recomputes the committed hash so tamper or drift fails the suite too.
- **NeMo Agent Toolkit memory provider.** The new `nvidia-nat-wyrm` integration (`integrations/nvidia-nat-wyrm/`) registers Wyrm as a `_type: wyrm` memory backend for the NVIDIA NeMo Agent Toolkit, plus a `GET /d/search` BM25 endpoint on wyrm-http that backs it.
- **Sharper context over long, compaction-heavy sessions.** `wyrm_context_build` and `wyrm_session_rehydrate` now downrank `[auto:*]` session-summary dumps so distilled, hand-captured decisions win the token budget. `WYRM_RANK_WEIGHTS` are normalized to sum 1 so a custom weighting can't silently mask the downrank.
- **`wyrm skill list`.** New CLI command surfaces the bundled SKILL.md guides plus your registered skills (they used to be buried in `node_modules`).
- **Fix: `wyrm_capture` no longer crashes** when the direct-capture fields (`kind`/`problem`/`validated_fix`) are passed without `mode:'direct'`. It now returns a clean, actionable error pointing at `mode:'direct'` instead of a server-side `Cannot read properties of undefined (reading 'trim')` and a silent write failure; `memory.add` guards a missing `problem` legibly.
- **Fix: `wyrm_context_build`** returns a clean "task is required" error instead of an opaque crash on a missing or mis-named argument.

## [7.6.0] - 2026-07-11 - Free tier works instantly, no account

- **Install-and-it-works.** The free/local tier now runs with no account, no login, and no network — instantly on install. Previously the official build required a free account (`wyrm login`) before any tool would run; that account wall is gone.
- **Activation is now an operator opt-in.** Set `WYRM_REQUIRE_ACTIVATION=1` to require every user to sign in (managed/team deployments). Off by default, including on the official published build.
- **`wyrm login` stays optional** — it unlocks cloud and paid features, nothing more. Your memory was always local; now nothing gates getting started.
- **Revenue protection unchanged.** Paid capabilities are enforced per-tool by `hasFeature()` as before; the former activation-gate tamper check moved into `hasFeature()`, so on the official build a stubbed license verifier denies *paid* features (never the free tier).

## [7.5.3] - 2026-07-11 - License metadata

- **Machine-readable proprietary declaration.** `package.json` now uses npm's `SEE LICENSE IN LICENSE` convention instead of `UNLICENSED`. The terms are unchanged (Wyrm remains proprietary); the old value told scanners "no rights granted to anyone," which is not accurate for official builds used under the Wyrm Terms of Service, and it zeroed the package's Socket.dev license score.
- **Historical license note reworded.** LICENSE and NOTICE still record that wyrm-mcp 7.2.1 and earlier were open source, but no longer name the old license inline; supply-chain scanners were string-matching it and flagging current proprietary versions as copyleft. The old versions' packages carry their own license text, which continues to govern them.

## [7.5.2] - 2026-07-11 - Feedback

- **New `wyrm feedback` command.** Opens a prefilled bug report, idea, or question with your version and platform filled in, so reports arrive usable. `--bug` and `--idea` go to Issues; `--question` goes to Discussions. Wyrm sends no telemetry, so this is the path when you want to reach the maintainer, and nothing is sent until you submit it yourself.
- **Install banner now points to it.** A one-line prompt so a rough edge or an idea has an obvious home.
- Repository Discussions enabled, plus bug and idea issue templates, so questions and feature ideas have a low-friction place to land.

## [7.5.1] - 2026-07-11 - Packaging

- **Smaller, runtime-only install.** The published tarball now ships compiled JavaScript only. Source maps and type-declaration files are no longer included, and the runtime JS is minified. This is a packaging change with no runtime behavior change.
- **Bundled skills curated to the Wyrm product set.** The install now carries four focused product skills — `wyrm-getting-started`, `wyrm-memory-and-recall`, `wyrm-nvidia-nim`, `wyrm-negative-learning` — alongside `buddy-protocol`. Internal workflow tooling is no longer part of the distributed package.
- Publish path hardened so the packaged artifact is produced the same way every time.

## [7.5.0] - 2026-07-10 - NVIDIA NIM retrieval provider (embeddings + reranking)

### Code-review hardening (15 confirmed findings from an xhigh multi-agent review)
- **Egress honesty, the core invariant, tightened three ways:** (1) a remote Ollama (`OLLAMA_URL` → GPU box) now reports its host as egress instead of claiming 'none'; (2) the receipt is built from what ACTUALLY EXECUTED (the `onStats` signal), so a recall that configured NIM but fell back to lexical no longer claims a 'vector' stage or names a remote host that never saw text; (3) ONE canonical loopback classifier (`isLoopbackHost`) replaces three hand-rolled copies — closing a false-local hole (`127.example.com` was treated as loopback) and a false-remote hole (`[::1]` was treated as remote).
- **NIM correctness:** `createProvider('nim')` prefers env keys over `config.apiKey` (the shared CLI fills the latter with `OPENAI_API_KEY` for every provider — it was Bearer-authing NVIDIA with the OpenAI key); a self-hosted NIM on loopback needs no key; embeddings and rerank now send `truncate:'END'` (NIM defaults to erroring on over-length input — long memories silently failed to index); the hosted rerank URL is derived from the model id so `WYRM_RERANK_MODEL` alone selects a different reranker.
- **Staleness guard fixed:** `indexCoverage` is scoped to artifact vectors (what recall searches), flags stale by majority-under-old-model (not the defeated `count===0`), skips non-embedding baselines (`none`/hash — no more false stale when Ollama is mid-probe), is cached ~5s (no full-table GROUP BY per recall), and is guarded against throwing on the hot path.
- **Health & misc:** unauthenticated `/health?check_dependencies=1` no longer returns raw error text (leaked the absolute db path + OS user); `OpenAIProvider.embed` gained the missing timeout; `docs/EGRESS.md` dropped a non-existent per-call rerank gate. +10 tests (25 → in the NIM/receipt suites). Suite: 1,954.

### Live-verified against build.nvidia.com (2026-07-10, Inception key)
- Hosted defaults moved to the CURRENT NVIDIA retrieval generation: the llama-3.2 embedqa/rerankqa models reached hosted EOL 2026-05-18 (410 Gone) — defaults are now `nvidia/llama-nemotron-embed-1b-v2` (2048d, verified) and `nvidia/llama-nemotron-rerank-1b-v2` (endpoint verified, clean logit separation). The llama-3.2 entries stay in the dims table for self-hosted NIM containers.
- Embeddings verified live end-to-end: both `input_type` roles, dimension self-correction, egress attribution (`integrate.api.nvidia.com`). Rerank verified live: relevant passage ranked first.
- NIM rerank default timeout 1500→4000 ms (observed 0.5–1.2 s hosted round-trips from LK left no headroom; a timeout silently drops the quality stage).
- `NimProvider.embed` now honors 429/5xx with Retry-After-aware backoff (2 retries, same discipline as the cloud-sync client) so rate-limited hosted keys degrade to slower, not to a crashed leg. +1 test.

**NVIDIA NIM retrieval provider (embeddings + reranking), explicit opt-in.** Two new legs on existing seams, both gated so the `auto` chain can never select them (memory text leaving the machine is a deliberate choice — the same egress posture as grove sync):

- *Embeddings* — `WYRM_VECTOR_PROVIDER=nim` + `NVIDIA_API_KEY` (or `WYRM_NIM_API_KEY`) activates `NimProvider`: the OpenAI-compatible NIM `/v1/embeddings` surface plus NVIDIA's retrieval `input_type` extension (their asymmetric embedders require it). Hosted (integrate.api.nvidia.com, the Inception-member path) and self-hosted NIM containers both work (`WYRM_NIM_BASE_URL`); model `WYRM_NIM_EMBED_MODEL` (default `nvidia/llama-3.2-nv-embedqa-1b-v2`); dimensions seed from a known-model table (`WYRM_NIM_EMBED_DIM` override) and self-correct from the first response before anything is persisted; timeout `WYRM_NIM_TIMEOUT` (default 8000ms).
- *Reranking* — `WYRM_RERANK_PROVIDER=nim` switches the existing `rerank.ts` cross-encoder stage to NVIDIA's ranking wire shape (`{model, query:{text}, passages:[{text}]}` → `{rankings:[{index, logit}]}`), Bearer-authed, default hosted endpoint for `nvidia/llama-3.2-nv-rerankqa-1b-v2` (self-hosted: point `WYRM_RERANK_URL` at `/v1/ranking`), default timeout 1500ms for the hosted round-trip. The resilience contract is unchanged: any failure returns null and recall falls back to the fusion order — never an error, never a stall.
- *Interface* — `EmbeddingProvider.embed()` gains an optional `inputType: 'query' | 'passage'`; `VectorStore.addVector` embeds as `passage`, `VectorStore.search` as `query`. Symmetric providers ignore the hint; no existing provider changes behavior.
- 14 new tests (`nim-provider.test.ts`), every network call mocked — the no-network deterministic-core invariant holds.

**NVIDIA-standards pass (key-free): egress honesty, deep health, stage timings, the reindex guard, and the three-leg bench.**

- *Egress-honest determinism receipt.* The receipt's `dataEgress: 'none'` claim would have silently become a lie the moment the opt-in cloud providers were enabled. It is now computed from the path that ran: `'none'` on the default local path (claim unchanged), or the named hosts (`embed:integrate.api.nvidia.com; rerank:ai.api.nvidia.com`) when an opt-in remote scorer saw text — with a new `local` boolean and an attestation line that states exactly which claim it is making. Providers expose `remoteHost` (loopback NIM containers correctly read as local); `rerankEgressHost()` does the same for the rerank endpoint.
- *Per-stage recall timings.* `recallHybrid` accepts an `onStats` observer and reports fts/vector/fusion/rerank wall-clock ms; `wyrm_recall` rides them on the receipt as `stageMs` (the RAG-blueprint per-stage-metrics pattern).
- *Provider-switch reindex guard.* `VectorStore.indexCoverage()` detects the silent semantic-blind state (active model has zero vectors while another model has many — what happens after switching nomic→NIM without reindexing); the receipt carries `indexStale` with the fix (`wyrm_reindex`), and health reports it.
- *Deep health.* `GET /health?check_dependencies=1` returns db migration level, embedding provider/model/readiness, index coverage + staleness, rerank config, and the live egress posture — coarse by design (no project names, no secrets; readiness probes never make a paid remote call). Also fixed: the HTTP surface's provider cast omitted `'nim'`, so the new provider could never activate over HTTP.
- *Egress disclosure.* `docs/EGRESS.md` — the complete inventory of every outbound call Wyrm can make, its gate, payload class, and off switch. If a call isn't on the table, that's a bug.
- *Three-leg NIM bench.* `bench/nim-retrieval.mjs` — local-nomic vs NIM-embed vs NIM-embed+rerank on LoCoMo, same fusion math as `recallHybrid`; prints the fully resolved config before running (reproduce from stdout), reports quality and latency as separate planes, and SKIPS key-gated legs loudly with the reason. Runs leg 1 today; legs 2–3 light up when `NVIDIA_API_KEY` lands.
- 7 more tests (egress honesty, loopback classification, stage/stale passthrough, coverage guard). Suite: 1,948.

## [7.4.0] - 2026-07-04 - audit alignment: the claims become the code

An external pitch audit (2026-07-04) found four places where the marketing was ahead of the code. This release closes the gap in the direction of MORE capability, not quieter copy. Minor bump: new observable behavior on `wyrm_truth_set` and the daemon; no breaking surface change.

**Truth auto-cascade.** A value-changing `wyrm_truth_set` supersede now automatically cascade-invalidates the decision edges downstream of the OLD truth (`Causality.invalidateDownstream`), exactly as the docs always claimed. Implemented inside `GroundTruths.set()` so every write path inherits it (MCP handler, daemon write endpoint, capture, CLI import); wired at each composition root via `wireTruthAutoCascade`. Same-value re-sets do not cascade; `WYRM_NO_AUTO_CASCADE=1` opts out; fail-open (a cascade error never breaks the truth write). The tool response now surfaces "Auto-cascade: N downstream decision edge(s) invalidated". Manual `wyrm_decision_invalidate` remains for refutations, revocations, and non-truth sources. 6 new tests (`truth-auto-cascade.test.ts`).

**The daemon can act.** `wyrm-loop` shipped with a stub ToolDispatcher that returned `ok:false` for EVERY internal tool — the OODA loop could decide but never act unattended. It now constructs the same `makeInternalDispatch` dependency set as the wyrm-mcp server. Safety posture unchanged: the SAFE_INTERNAL_TOOLS whitelist still gates every call (read-mostly + bounded writes) and `wyrm_call_external` stays default-deny behind `WYRM_LOOP_ALLOW_EXTERNAL`. New runtime coverage (`wyrm-loop-dispatch.test.ts`, 3 tests) plus a tool-surface-integrity guard asserting the stub can never return.

**Enforcement transparency.** A BLOCKED `wyrm_failure_check` verdict now names the enforcement world it lives in: `wyrm-guard hook ACTIVE -- hard-blocked at the harness` when the PreToolUse hook is installed and `WYRM_GUARD_MODE=block`, or `ADVISORY ONLY on this machine -- run 'wyrm guard' to enforce` otherwise (60s-cached, single settings-file read, ASCII, fail-quiet). `installWyrmGuardHooks` no longer silently no-ops when `~/.claude` is missing — it reports the firewall as NOT enforced on this machine and says how to fix it. 4 new tests (`failure-enforcement-line.test.ts`).

**MCP Registry readiness.** `package.json` gains `mcpName: "lk.ghosts/wyrm"` (the registry's npm ownership proof) and `server.json` (registry metadata, domain-verified via ghosts.lk) ships in the package directory.

**Docs honesty pass.** `docs/index.md` no longer claims per-client profile auto-detection (removed in 7.0; `WYRM_PROFILE` is the one lever); `wyrm_decision_invalidate` docs describe the new automatic path; GETTING_STARTED names the hook-level hard block instead of only "warns". Second sweep (adversarial claim verification): "all installed AI clients" now names the actual supported roster; "Zero Config" acknowledges the one-time free `wyrm login` the official build requires; test badges de-drifted to the real count (1,901). And the one default outbound call a local-first product had — the daily npm version poll — is now refusable: `WYRM_NO_VERSION_CHECK=1` disables the passive check (explicit `wyrm_check_update` still works), and the Local-only claim discloses it.

## [7.3.3] - 2026-07-03 - staleness oracle + zero-config guard installer

Two additive features and the security fixes an adversarial pre-release audit surfaced. No tool/API surface change (patch); the default wire is byte-identical when the new features are unused.

**Failure Firewall staleness oracle (migration 30, PROTOCOL.md §10.2.5).** `wyrm_failure_record` accepts an optional `files` list that anchors a failure to the source it is about (project-relative path + normalized-content hash + a stat fast-path). At check time, a file/symbol-scoped hard block whose anchored source has positively drifted downgrades to an advisory carrying `stale_anchor: "source changed since this failure was recorded — re-verify"`. An exact command/edit signature match never downgrades (code moving on does not unprove a command failure). Fail-open end to end: missing files, no project root, a pre-migration DB, or any fs error leaves the verdict unchanged. Deterministic, no LLM, offline.

**Zero-config wyrm-guard installer.** `wyrm setup` now wires the failure-firewall PreToolUse hooks automatically, plus an explicit `wyrm guard [--remove|--status]`. Pure-TS port of the repo `scripts/hooks/install.sh` strip/merge (no `jq`), idempotent, npm-global-safe, and it refuses to touch a `settings.json` it cannot faithfully merge rather than clobbering it. Unrelated user hooks are preserved byte-for-byte.

**Security (pre-release crucible, 3-of-3 adversarial verify).** Closed an anchor-injection firewall bypass: because `record()` coalesces by signature, a re-record could previously attach a decoy anchor to a block it did not own and drift it to advisory. Only a freshly-created failure may introduce new anchors now; a coalesced re-record may only refresh existing ones. Closed a hook command-injection: the guard command is POSIX single-quote-escaped (was double-quoted), so an install path containing shell metacharacters can never execute at PreToolUse time. Block attribution now credits the firm match, not a downgraded one.

## [7.3.2] - 2026-06-27 - benchmark receipts + firewall reworded-recall

Adds the *receipts* behind the intelligence-layer claims and one real firewall improvement they surfaced — all measured on real LoCoMo and reproducible from the repo. No tool/API surface change (patch).

- **Negative-learning benchmark** (`bench/negative-learning.mjs`) — the firewall measured, the benchmark no recall-only memory runs: **100% recall / 100% precision** blocking a repeated mistake (0 false blocks on novel actions), deterministic, p50 ~0.1 ms.
- **Firewall reworded-recall fix** — a stopword filter on the fuzzy `failure_check` probe lifts reworded-repeat recall **6.3% → 12.5%** with **precision held at 100%** (guarded by `tests/firewall-recall.test.ts`).
- **Cross-encoder reranker measured** (`bench/locomo-rerank.mjs`) — the opt-in `WYRM_RERANK_MODEL` leg lifts recall@1 **33% → 53%** (+19 pts) and MRR **.45 → .61**; weak categories lift most.
- **Honest competitor comparison** (`COMPARISON.md`, now shipped in the npm tarball) — retrieval recall@k vs the field's LLM-judged QA-accuracy, never conflated, every figure cited.
- **npm README** surfaces all three benchmarks; embedding-model sweep documented (`mxbai-embed-large` as an option); the QA-accuracy harness kept as an opt-in bring-your-own-answerer tool.

## [7.1.0] - 2026-06-14 - "BROOD" Phase F4: the context economy finished, the monolith drained, the render target

7.1 completes the 7.0 "BROOD" arc. The context economy is finished (resources + pagination return LINKS, not inlined bytes), the index.ts monolith is drained (quest #80 closed), every advertised tool can be driven through MCP Tasks on its long ops, casual sessions cost ~zero MCP tokens via the render target, fleet exhaust is harvested into institutional memory, and the retrieval engine ships a published two-tier number with a CI-gated no-LLM floor. **Every 6.x name stays callable**, every migration is additive/guarded (a v7.0.x DB opens 100% rows), and `wyrm render` never silently overwrites an unharvested human edit. All numbers are **measured** on this machine (Article VIII): from a committed bench, the suite, or a release-gate run; nothing is asserted.

### The context economy finished

- **`wyrm://` resources (T034)** — `src/handlers/resources.ts` adds the MCP resources primitive: big payloads return as LINKS, fetched on demand via `resources/read`. URI scheme `wyrm://{capabilities,stats,memory/{id},project/{id}/{truths|failures|quests|memory|stats}}`; `parseWyrmUri` is strict + path-safe (Article VII — traversal / control chars / extra depth / unknown leaves / bad ids all reject; ids are bound params, never interpolated). `readWyrmResource` is a pure local-DB read (zero LLM/network/clock, Article III; hard 200-row caps). `resources/list` + `resources/templates/list` + `resources/read` wired to the same `dispatcherCtx` so resource and tool surfaces share one backend (no drift); server advertises `resources:{}` (vendor-neutral). **`detail=full` FALLBACK proven**: `recall` gains `detail=full|link`; `detail=link` (or a resource-advertising client) appends one `wyrm://memory/{id}` resource_link per result, but the inline body is BYTE-IDENTICAL to `detail=full` — **no client loses data**. `capabilities`/`intro` stay advertised (retire-to-resource deferred — the fallback is proven but the tools are kept).
- **Keyset cursor pagination (T035)** — `src/keyset.ts` (net-new): an opaque base64url COMPOSITE `(sortKey, id)` cursor — never a bare sortKey — so a page boundary inside a run of TIED sort keys drops nothing and repeats nothing (the explicit fix for the wyrm-cloud sync keyset bug, the cautionary tale). `clampPageSize` is a hard `[1, MAX_PAGE_SIZE=200]` cap with `DEFAULT_PAGE_SIZE=50` (no unbounded limit requestable); `buildPage` over-fetches by one (no COUNT query). `MemoryArtifacts.listPage` is the keyset sibling of `listAll` over a TOTAL order (confidence DESC, id DESC), and `wyrm://project/{id}/memory` paginates via an optional path-safe `?cursor=` query surfacing `next_cursor` in the body and the ReadResource `_meta`. The advertised 32-verb surface is byte-IDENTICAL (pagination rides the resource layer + a store method, not the frozen tool schemas).

### The monolith drained — quest #80 CLOSED (T036)

- **`index.ts` 5,521 → 998 lines** (spec §7 criterion 11: ≤1,000); **`buildAllTools()` deleted**; the **142-case switch (103 advertised public cases) → 0**. 18 new `handlers/<domain>.ts` ToolSpec modules carry the drained domains — project(5) events(6) skill(10) datalake(4) entity(6) intelligence(12) orchestration(3) cloud(7) companion(15) syncops(3) presence(3) causality(4) symbols(4) invoicing(2) audit(3) share(6) agent(4) mcpclient(6) = 103 tools. Each case body + its inputSchema lifted VERBATIM (byte-identical wire); handlers destructure from a broad `DispatcherContext` (`handlers/dispatch-context.ts`) carrying the 39 module-scoped singletons + 6 helper closures — ZERO behavior drift. `registry.ts` spreads all 18 domains; dispatch is one O(1) `handlerRegistry.get()`. A new quest #80 line-count lock test pins `index.ts ≤1,000`.
- **Deviation:** the 103 drained tools are UNTYPED at this stage (no `outputSchema` — spec FR-3 funds their schemas as optional during rollout; the T019 dual-emit ratchet stays scoped to the typed hot-path + `wyrm_run` contract surface). Three index.ts factories were also extracted to hold the ≤1,000 ceiling (`internal-dispatch.ts`, `context-build-budgeted.ts`, `buddy-runner.ts`).

### MCP Tasks on the retained long ops (T037)

- **`src/tasks.ts` + `src/tasks-dispatch.ts`** — MCP Tasks RC support on the long ops of tools that REMAIN on MCP (`wyrm_maintenance` reindex/vector-backfill + harvest/auto_capture), **capability-gated with a SYNCHRONOUS fallback** "until the RC stabilizes". Wyrm's faithful approach NEVER defers work: `runLongOp` always runs the op synchronously, returning either `{kind:inline}` (a normal CallToolResult — the fallback spine) or `{kind:task}` (stores the completed result, returns an already-`completed`/`failed` CreateTaskResult). `tasks/get|result|list|cancel` served from a bounded in-memory TaskStore (ttl + 256-cap eviction); `tasks:{list,cancel,requests.tools.call}` advertised at `initialize` (additive — ListTools/discovery/byte-stability unaffected). **ADMIN GATE** (Article VII): the destructive rebuild ops are off by default; `WYRM_ADMIN=1` opts in (a stray fleet agent can't vacuum/reindex over MCP); the gate is a deterministic non-retryable SEP-1303 `WYRM_ADMIN_REQUIRED`. Harvest/auto_capture are NOT admin-gated (thin-corpus harvesting must stay reachable). This fixes the curated-client "reindex-last-17%" wound — maintenance stays on MCP precisely so a constrained client CAN drive a backfill.

### The render target — the zero-MCP-token casual path (T038)

- **`wyrm render`** (`src/render-target.ts`, net-new template-isolated writer) — deterministically compiles a project's authoritative state (truths / failures / quests / validated patterns) straight into the harness-native memory slot, so a casual session spends ZERO tool calls to load it. `renderMemoryMd` enforces a **HARD 200-line budget** (sections in priority order; the overflowing section truncated with an explicit "N more — query <tool>" note; verified: a 2,000-row corpus stays ≤200 body lines). `renderForClient` adapters (Article V, vendor-neutral): claude→CLAUDE.md, cursor→`.cursor/rules/wyrm-memory.md`, copilot→`.github/copilot-instructions.md`, agents→AGENTS.md — one digest block, only the destination differs; client detection reuses `autoconfig.ts`. **DETERMINISTIC / byte-stable** (zero `Date.now()`/`Math.random()` in the OUTPUT — every volatile value passed via `RenderStamp`; same model + stamp ⇒ byte-identical, golden-replayable). **PROVENANCE-stamped** ("Compiled by Wyrm vX … edits are HARVESTED to the review queue, not lost"). **SAFE** (Article VII): `resolveInsideRoot` rejects traversal/absolute escape AND (after security pass #2) refuses an in-root symlink target and re-asserts the realpath of the nearest existing ancestor; `spliceWyrmRegion` replaces ONLY the marked region (operator prose preserved); a non-Wyrm-managed file is SKIPPED by default (never clobbers a hand-written MEMORY.md). Opt-in daemon re-render gated on `WYRM_RENDER_WATCH=1` (off by default).

### The reverse bridge (T039)

- **`src/reverse-bridge.ts`** (net-new, pure + deps-injected) closes the render loop: watches the harness-native files `wyrm render` writes, diffs human/agent edits against the last-rendered block, and feeds them through `auto_capture` into the REVIEW QUEUE. Two hard invariants, each test-locked: **NEVER SILENT INGEST** (every edit becomes a `needs_review=1` candidate, `created_by='reverse-bridge'`, with an escape-guarded LIKE dedup probe) and **NEVER SILENT OVERWRITE** (`guardRender()` detects a region edited since the last render from on-disk bytes; `wyrm render` runs a pre-render sweep that queues any region edit before overwriting it — the edit survives as a review candidate, never destroyed). Offline/deterministic; opt-in watcher (`WYRM_REVERSE_BRIDGE=1`), one-shot `wyrm reverse-bridge` sweep is read-only on the watched files. **Deviation:** implemented as a one-shot sweep primitive + the render-time guard rather than a standalone watcher daemon process (the watcher loop overlaps the existing agent/replication daemon ticks).

### Trace harvest (T040)

- **`src/trace-harvest.ts`** (net-new) harvests a harness's working trace into run-tagged review-queue candidates, OFFLINE. PURE parsers for three trace shapes (Claude Code session JSONL, `~/.dragon/traces`, `WYRM_TRACE_TOOL_CALLS` output) + deterministic secret REDACTION (vendor API keys, GitHub/AWS/Slack/Google tokens, JWTs, bearer, key=value) that runs BEFORE extraction so a leaked credential never enters the queue. Parsed segments feed the EXISTING `auto_capture` extractor (local Ollama / deterministic fallback — never a cloud LLM, Article III). New HIDDEN tool `wyrm_capture_trace` routed from the capture shim `mode=trace` (candidates `needs_review=1`, run-tagged, `ax:` idempotent). **Deviations:** (1) `wyrm_capture_trace` is a new v7 tool that is NOT advertised (the ≤32 pin holds) — a hidden alias like `wyrm_auto_capture`; it needed a new `V7_HIDDEN_TOOL_NAMES` category (`V7_ADDED = V7_NEW ∪ V7_HIDDEN`) — every count formula that read `V7_NEW` now reads `V7_ADDED`; `WYRM_TOOL_COUNT` stays 137; alias spine 118→119; disposition stays 137 rows. (2) `mode=trace` added to the `wyrm_capture` advertised enum (a surface byte change, re-proven).

### A published number (T041)

- **`bench/longmemeval.mjs`** + **`bench/longmemeval-fixture.json`** — a citable retrieval bench over evidence-turn recall (what the retrieval engine alone achieves, not LLM-answer accuracy). **Tier 1** = no-LLM FTS floor (`memory.recall`, offline, deterministic, byte-stable — the CI-gated tier); **Tier 2** = local-vectors hybrid (FTS ⊕ `nomic-embed-text`, convex α=0.7 cand=50, all local, no cloud — advisory). **CI retrieval-regression GATE** (`tests/longmemeval-floor.test.ts`, UNCONDITIONAL — the retrieval analogue of the discovery BM25 gate, spec §7 criterion 9).
- **Measured — no-LLM FTS floor (committed fixture, deterministic, CI-gated):** recall@1 69.6% · recall@5 95.7% · recall@10 100.0% · MRR .813 (gate band r@5 ≥90% / r@10 ≥95% / MRR ≥.75, conservatively below measured).
- **Measured — local hybrid (fixture, Ollama up):** recall@1 82.6% · recall@5 100.0% · recall@10 100.0% · MRR .906.
- **Published LoCoMo band (real set, 1,982 evidence QA, local hybrid):** recall@5 ≥62.1% / recall@10 ≥72.7% / MRR ≥.447; no-LLM FTS floor on real LoCoMo is 52.4% / 59.9% / .393. The gate enforces the FLOOR not the band — an enforced number must not depend on Ollama or the network (Article III).

### Cloud parity v2 (T042)

- **`src/cloud-profile.ts`** — the **measured** cloud-capable subset of the standard tier (contract **`wyrm-cloud-memory-v2`**), GENERATED from the registry not hand-listed (Article V — cloud parity is a measurable property that cannot lie). The rule: **v2 = standard tier − device-local − egress − subprocess**. The three exclusion classes are the tools a stateless edge Worker cannot honestly serve: SUBPROCESS (`wyrm_run`, `wyrm_maintenance`), EGRESS (`wyrm_call_external`, `wyrm_mcp`, `wyrm_replication`, `wyrm_share`, `buddy`), DEVICE-LOCAL (`wyrm_act`, `wyrm_presence`, `wyrm_session`, `wyrm_capabilities`, and — after security pass #2 — `wyrm_skill`, `wyrm_project`, whose surfaces include local FS writes). `dist/tool-manifest-v2.json` (the v2 edge contract) + `cloudSupportedV2`/`cloudExclusion` on `dist/wyrm-manifest.json`, emitted on every build. **v1→v2 back-compat guard** (`cloudV1ResolvesUnderV2`, build-time throw) — every v1 cloud name is still answered under v2 (7 direct + `wyrm_remember`→`wyrm_capture`, `wyrm_quest_add`→`wyrm_quest` via the spine, Article VI).
- **Measured — cloud parity v2 = 19 tools** (standard 32 − 13 excluded), up from the v1 **9** — within the spec §7-criterion-12 ~18–22 band. Worker NOT deployed (operator-gated); this ships the contract + manifest + server-side readiness only.

### Adversarial security pass #2 (T043)

- Daemon-writer + MCP Tasks + render/reverse-bridge surface audited; all confirmed findings fixed at root cause with a ratchet test (across three commits + a second-round review of the fix pass itself):
  - **CRITICAL** — `wyrm render` silently overwrote an unharvested human edit when the reverse bridge was OFF (the default): the harvest-before-overwrite sweep was gated behind `reverseBridgeEnabled()`. Fix: the sweep now runs UNCONDITIONALLY on an explicit render (the env gates the continuous WATCHER, not the one-shot data-loss guard).
  - **MAJOR** — the render path boundary was lexical-only; an in-root symlink pointing outside the root was followed. Fix: `resolveInsideRoot` refuses a symlink target and re-asserts the realpath of the nearest existing ancestor.
  - **MAJOR** — cloud parity v2 declared device-local noun shims (`wyrm_skill`, `wyrm_project`) as cloud-capable, making the v2 manifest LIE. Fix: both excluded (v2 21→19); a BEHAVIORAL anti-tautology test now cross-references the shim route table against a source-verified deny-list of device-local/egress handlers.
  - **MINOR** — a rejected reverse-bridge candidate was re-queued on the next sweep (no rejection memory). Fix: a per-project rejection tombstone (migration 26); a follow-up second-round fix folded commas into the dedup-sig normalization so a prose comma can't truncate the tag-round-tripped sig.
  - **MINOR** — `wyrm://project/{id}/memory` leaked `needs_review=1` rows. Fix: a constant `AND needs_review = 0` clause matching every other validated read path.

### Compatibility

- **Every legacy name stays callable** — the alias spine grew 118→119 (the new hidden `wyrm_capture_trace`); the advertised 32-verb surface is byte-identical; `WYRM_TOOL_COUNT` stays 137; disposition stays 137 rows.
- **Additive guarded migrations** — schema reaches **v26** (migration 26 = the reverse-bridge rejection tombstone); a v7.0.x DB opens with 100% rows.
- **`wyrm render` never silently overwrites an unharvested human edit** — the harvest-before-overwrite sweep is unconditional; an edited Wyrm region is queued to review before any rewrite.

### Measured surface (release gate, this machine)

- **Default ListTools = 32 tools, 30,793 chars ≈ 7,699 tokens (chars/4, F1-census), byte-stable** (`bench/listtools-size.mjs`); essential 4 tools ≈ 1,736 tokens; legacy 151 tools ≈ 28,464 tokens.
- **Suite: 104 suites / 1,732 tests green** (1,573 at the 7.0.3 base → +159). `npm run build` green (tsc + manifests + 151/151 annotation coverage). Discovery BM25 ≥90% top-3 gate, ListTools byte-stability, golden replay 100%, no-network deterministic-core, and the new LongMemEval floor gate all green.

## [7.0.3] - 2026-06-13 - Cloud device-collision guard: copied sessions fail LOUD

Fixes **failure #40**: a second device on the same Wyrm Cloud account pulled 0 rows and peer skills/truths came back MISSING, even though its push worked — silently, with no error. Root cause is a **device_id collision**: `device_id` lives in `~/.wyrm/cloud.json`, minted per-machine at `wyrm cloud login`. If an operator provisions a second machine by **copying `cloud.json`** (or all of `~/.wyrm`) instead of running its own login, both machines share one `device_id`. The cloud pull query is `WHERE account_id=? AND device_id != <self> AND updated_at > cursor` (correct — a device shouldn't re-pull its own deltas), so each colliding machine filters the OTHER's deltas out as "its own": push works, pull = 0. The **server is left untouched**; the fix prevents the client-side identity collision and surfaces it loudly.

### Added
- **Per-machine fingerprint** (`src/cloud/machine-id.ts`): `machine_fp = sha256(hostname() "\n" install_id)[:32]`, where `install_id` is a 32-hex random id generated once and stored at `~/.wyrm/machine-id` (0600). No new dependency (node `os` + `crypto`). The fingerprint is recorded into `cloud.json` at `wyrm cloud login`.
- **Copied-session guard** on `wyrm cloud sync` (`runSync` / `guardCopiedSession`): recomputes the fp and compares to the session's stored value. On **mismatch** it emits a LOUD stderr warning (never stdout — stdout is the MCP wire) explaining the device_id collision and the fix (`rm ~/.wyrm/cloud.json ~/.wyrm/cloud-cursor.json` — **keep `cloud.key`** — then `wyrm cloud login`), and **STOPS** the sync. Escape hatch for a legitimate hostname change: `--force` or `WYRM_ALLOW_COPIED_SESSION=1` (warns, adopts the new fp, continues).
- **`wyrm cloud doctor`** — diagnoses this device's cloud identity: device_id (first 8), stored vs current fingerprint, and the copied-session verdict. `wyrm cloud status` gains the same identity health line.
- `WYRM_CLOUD_DIR` env override on the cloud config dir (session/cursor/key/machine-id) so tests sandbox away from `~/.wyrm`.

### Behavior / compat
- **Backward compatible**: a pre-7.0.3 session has no `machine_fp`; on first sync the current machine is **adopted silently** (fp written in place, no warning). Only a PRESENT-but-different fp triggers the warning — no false alarms on existing installs after upgrade.
- **No migration** — `cloud.json` and `machine-id` are files, not DB rows.
- The only `~/.wyrm` file meant to be shared across an operator's machines remains `cloud.key` (the E2E master key); `cloud.json` + `cloud-cursor.json` + `machine-id` are per-device.

### Tests
- +13 tests (`tests/cloud-device-fingerprint.test.ts`): fp generated + stable; install id stable + machine-divergent; classify absent→adopt / equal→match / different→mismatch; guard pre-7.0.3→adopt-silent, match→silent, mismatch→warn+stop, `--force` + env→warn+adopt+continue. Hermetic via `WYRM_CLOUD_DIR` (never touches `~/.wyrm`).

## [7.0.2] - 2026-06-13 - Portable skills: SKILL.md content in the registry + cloud sync

Skills become portable across machines (e.g. a Mac mini running a Hermes/NemoClaw agent) — the registry now stores the SKILL.md body, syncs it E2E, and can re-materialize the files.

### Added
- **Migration 25** (additive/guarded/idempotent): `skills.content` (+ `content_sha256`, `content_updated_at`). The external-content `skills_fts` index (name/description/tags) is untouched; a v7.0.1 DB opens with 100% rows preserved.
- **Content capture**: `registerSkill` reads and stores the SKILL.md body best-effort (registration never fails on an unreadable file; re-register keeps prior content).
- **`wyrm skill backfill-content`** — populate content for already-registered skills, idempotent by sha (dry-run on this machine: 347 of 375 registered skills have a readable SKILL.md).
- **`wyrm skill export <dir> [--all]`** — re-materialize SKILL.md files from stored content (round-trip byte-identical). Slug collisions are disambiguated with a name-hash suffix so two distinct skills can never clobber each other.
- **`wyrm skill share <name|--all|--tier T> [--public|--private] [--include-inactive]`** — set cloud-sync visibility, single OR **bulk** (`--all` / by `--tier`). Reports the count promoted.
- Skills are cloud-sync-eligible, gated **private-by-default** by per-row `cross_project_visibility` ('within'); nothing egresses until explicitly shared (Article IX). Enforced in both default and `--all` sync paths.

### Fixed
- `exportSkillContent` no longer silently clobbers on slug collision (deterministic order + name-hash disambiguation; surfaces a `collisions` count).

## [7.0.1] - 2026-06-13 - npm page polish (docs/metadata only)

No code change. Makes the npm package page reflect 7.0 and lead with the site:
- `homepage` → **https://wyrm.ghosts.lk** (the Wyrm site; `repository` still links GitHub in the npm sidebar).
- README: a links row under the badges (wyrm.ghosts.lk · npm · GitHub · Ghost Protocol + the install one-liner) and a one-line local-first positioning sentence; tests badge 1507 → 1543; `legacy` dist-tag reference corrected to `wyrm-mcp@6.18.1` (root README too).

## [7.0.0] - 2026-06-13 - "BROOD": the run-attributed, typed, frozen-surface memory bus for agent fleets (Phases F2+F3)

The category move: 7.0 turns Wyrm from a 137-tool single-chat prose server into the run-attributed, typed, discovery-engineered memory bus for agent fleets — **with every 6.x name still callable**. Every number below is **measured** (Article VIII): from a committed bench, the suite, or the release-gate run on this machine; nothing is asserted.

### The frozen lean surface + machine wire (Phase F3, T018–T033)

- **Default ListTools = 32 tools, 31,972 chars ≈ 7,993 tokens (chars/4, the F1-census convention), byte-identical across consecutive calls** (`bench/listtools-size.mjs`; the 6.x default was 62 tools ≈ 9,757 tokens, full-load 137 ≈ 24.9K). Profiles: `essential` = the always-load core 4 (`session_prime`/`recall`/`capture`/`failure_check`; 6,926 chars ≈ 1,732 tokens); `legacy` = all 150 names (117,511 chars ≈ 29,378 tokens); `WYRM_PROFILE=full` is now a **permanent synonym of `legacy`**. The frozen surface = 19 first-class 6.x names + 12 action-param noun shims (`wyrm_quest`, `wyrm_session`, `wyrm_skill`, `wyrm_entity`, `wyrm_decision_trace`, `wyrm_goal`, `wyrm_presence`, `wyrm_mcp`, `wyrm_audit`, `wyrm_project`, `wyrm_design_token`, `wyrm_reference`) + `wyrm_run`.
- **Discovery is a hard CI gate (BM25 top-3 ≥90%):** 65 natural intent queries, deterministic offline BM25 — measured **100% lenient top-3 / 96.9% strict top-1** on the frozen surface after the when-to-use-first storefront rewrite (53.8% lenient top-3 before it; the rewrite lifted the legacy advisory 137-name baseline 70.8% → 87.7% too). Gate report committed at `specs/wyrm-v7-brood/research/discovery-gate.json`.
- **Typed machine wire:** ToolSpec contract v2 registry with byte-stable deterministically-ordered ListTools; ONE dual-emit renderer (`src/render.ts`) — `content[0].text` is *derived* from `structuredContent`, so text/structured drift is impossible by construction (plain ASCII default, glyphs behind `WYRM_FANCY=1`); SEP-1303 structured errors (`WYRM_VALIDATION` / `WYRM_BUSY` / `WYRM_CLI_EXILE`, each `{error, expected}`) so structured-output subagents self-correct mid-fleet; **the 7 hot-path domains (capture, recall, search, failure, quest, session/prime, review) return schema-validated `structuredContent`** (34 registry ToolSpecs, contract-tested per response); **150/150 advertised tools annotated** (build fails otherwise).
- **Alias spine — generated, never hand-written:** 118 hidden aliases = the 137 6.x names − 19 legacy-named survivors, generated from the LIVE booted ListTools surface (`scripts/gen-alias-spine.mjs --check` in CI), routing to the SAME handler code paths via argument adapters — never reimplementations; **0 hidden aliases enumerable** on any non-legacy profile (standing conformance suite, 17 tests); **synthetic golden replay 100%** — 150 fixtures (137 6.x names + 12 shims + `wyrm_run`), both arg variants replayed live through the spine on a booted server, gating every F3 commit. *(Honesty per Article VIII: the planned ORGANIC `WYRM_GOLDEN_CAPTURE` replay half is unmeasured at 7.0.0 — the 6.18 soak window never elapsed; organic capture + replay is deferred to 7.0.x.)*
- **`wyrm_run` (`action=start|join|status|debrief|end`)** — the 32nd survivor: ULID run rows, role rosters, run-quarantined failure counts + claims on `status`; `debrief` fans each agent's learnings through the LOCAL-only auto_capture pipeline (Ollama via `WYRM_EXTRACT_MODEL` / deterministic fallback — never a cloud LLM) into a run-scoped review queue; `end` promotes/expires quarantined failures, bulk-releases the run's claims (member-only), and writes the run summary artifact.
- **Fleet-mode `session_prime`:** `{run_id, role, token_budget}` — the first prime compiles a role-sliced brief cached per `(run_id, role)` (migration 23, PK CAS); concurrent primes across processes return the byte-identical cached prefix; **default brief ≤1,200 tokens**, over-budget sections stub as plain-text `wyrm://` references.
- **MCP prompts primitive:** `prime` + `debrief` as protocol-level prompts (Article V — hosts that never read CLAUDE.md still get consult-memory-first), rendering through the SAME registry handlers as the tools.
- **Claims/presence hardening:** migration 24 — `run_id`/`role` on `quest_claims`/`agent_presence`, stale-claim eviction wired into `wyrm_maintenance` (previously reap ran only inline at claim time), orchestrator-audience descriptions.
- **CLI exile (22 operator/egress tools off MCP):** new `wyrm license|activate|maintenance|index|update|prompt|hours|invoice|agent` subcommands (thin wrappers over the SAME modules the MCP cases ran); the exiled MCP names return a structured `WYRM_CLI_EXILE` redirect naming the exact command. **Exception (published grace):** `wyrm_cloud_backup` + `wyrm_sync_export` keep EXECUTING through 7.0.x so scheduled backups never silently stop.
- **`wyrm-manifest.json`** (tiers core 4 / standard 32 / legacy 150, per-tool `cloudSupported`) generated from the compiled registry on every build so it cannot lie — plus the committed **137-row disposition table** (`specs/wyrm-v7-brood/disposition.md`: 19 survivor / 96 alias-into-X / 20 CLI-exile / 2 gated-retirement), THE migration appendix.
- **Adversarial security pass #1 (T032, quest #82) complete:** 16 confirmed findings fixed across 4 commits (multi-agent audit of the actor envelope, alias router, shims, and run loop — plus a second-round review of the fixes themselves): control-char strip at the one render chokepoint, the 32K `learnings[]` bound pre-scanned before side effects, run-authority guards (reserved orchestrator role, member-only `end`, sticky-role refresh), fleet-prime cross-project pin, LIKE-wildcard escaping at all 4 sig-dedup sites, presence role boundary, `run_briefs` growth caps. Each fix ships with a ratcheting regression test.

### Removed

- **wyrm-http** (`src/http-server.ts`, the legacy second HTTP server + its bin + `npm run http`) is **DELETED**, exactly as the 6.18 startup warning promised. Evidence (T005): its route-level access log (`~/.wyrm/http-server-access.log`) was never even created — **zero requests** since 6.18 shipped the logging. `wyrm serve` (http-fast) is the maintained HTTP surface (API + Live-Memory SSE + dashboard on :3333). The orphaned static `ui/index.html` went with it (http-fast embeds its own dashboard; `ui/dragon-mark.svg` stays); the Dockerfile CMD and the systemd unit (now `config/wyrm-serve.service`) run http-fast.
- **Client-name sniffing** (replaced by profiles + the generated manifest) and the **Anthropic-only `cache_control` `_meta` injection** (Article V: vendor hint with no MCP capability to gate on) — both removed at T022. The in-memory response cache is unaffected (now keyed on the RESOLVED call, so alias/shim/survivor spellings share one entry).

### Compatibility & migration (Article VI — stated honestly)

- **All 137 6.x names remain callable.** ~115 execute identically via the alias spine; ~22 operator/egress names return a structured redirect carrying the exact `wyrm` CLI replacement (with the cloud_backup/sync_export functional grace above). The institutional hot-path names (`wyrm_session_prime`, `wyrm_recall`, `wyrm_search`, `wyrm_capture`, `wyrm_context_build`, `wyrm_truth_set/get`, `wyrm_failure_check`, `wyrm_failure_record`, `wyrm_decided_because`, `wyrm_review`) are frozen verbatim — every existing CLAUDE.md contract keeps working unmodified.
- **`content[0].text` persists on every response but REFORMATS** (renderer-derived ASCII; `WYRM_FANCY=1` restores glyphs) — exact-string-match consumers must update or pin the 6.x line (the `legacy` dist-tag below); genuine text parsers (statusline) are covered by byte-exact renderer fixture tests.
- **Data:** a v6.17 database opens under 7.0 with 100% row preservation — migrations 20–24 are all additive + guarded; a 6.x binary still reads every pre-existing table; historical rows read as `actor='legacy'`.
- **Per-name dispositions:** the committed 137-row table at `specs/wyrm-v7-brood/disposition.md`.
- **npm 6.x line:** after `wyrm-mcp@7.0.0` publishes, 6.18.0 stays installable under a `legacy` dist-tag with security fixes for 6 months — documented operator command (not run by CI): `npm dist-tag add wyrm-mcp@6.18.0 legacy`.

Measured at the release commit: build green (tsc + generated manifests + 150/150 annotation coverage); **full suite 89 suites / 1,507 tests green** (~23s; the F3 branch base was 73 suites / 1,301); default surface 32 tools / 31,972 chars ≈ 7,993 tokens (chars/4 convention), byte-stable; discovery hard gate 100% lenient top-3 (threshold 90%); synthetic golden replay green in-suite.

### FEATHERWEIGHT — the token economy, measured (ships with 7.0.0)

Wyrm is now the *cheapest credible memory layer per session and per fleet-run* — **measured by a committed meter, never asserted** (Article VIII). FEATHERWEIGHT added **`bench/token-economy.mjs` — THE METER**: a deterministic tokens-per-scenario instrument (chars/4, the F1-census convention) run on the LIVE wire (stdio MCP for tools, the auth-required http-fast + the REAL `scripts/hooks/wyrm-push.py` for hook pushes, `wyrm rehydrate` for SessionStart), over a fixed sandbox corpus (50 truths / 200 memories / 30 failures / 20 quests / 6 sessions, mulberry32 seed 0xFEA7; never `~/.wyrm`). The committed `bench/token-economy-baseline.json` is the canonical state; `--write-baseline` regenerates it byte-identically.

**The headline (chars/4, F1-census convention, this machine):**
- **A Wyrm session costs ~11,622 tokens cold (S1)** — unchanged on the default wire; **down to ~10,638 (−8.5%) on the opt-in single-channel text wire.**
- **A 12-agent fleet run costs ~18,338 tokens (S3) by default; down to ~11,208 (−38.9%) when the orchestrator distributes the role brief once** (`session_prime for_spawn`, opt #2, adoption-gated) — and to **~8,410 (−54.1%) with both opt-in single-channel text + distribute-once.**
- **A casual session (zero Wyrm calls) costs ~11,463 tokens (S4)** — what ListTools (7,993) + server-instructions (158) + SessionStart rehydration (1,522) + a 10-edit/10-prompt hook stream (1,784) cost whether or not Wyrm is used; unchanged (no opt-in path touches it).

**Before → after (baseline at b301794 = the 7.0.0 release-ready tip vs now; default `both` wire unless noted):**

| Scenario | b301794 baseline | now (default) | now (opt-in text channel) | distribute-once (for_spawn) |
|---|---|---|---|---|
| S1 cold start | 11,622 | **11,278 (−3.0%)** | 10,638 (−8.5%) | — |
| S2 working hour | 23,195 | **21,102 (−9.0%)** | 11,841 (−49.0%) | — |
| S3 fleet run (12 agents) | 18,338 | 18,338 (0.0%) | 8,410 (−54.1%) | 11,208 (−38.9% vs S3) |
| S4 casual session | 11,463 | **9,900 (−13.6%)** | 9,900 (−13.6%) | — |

The original FEATHERWEIGHT release left the **default wire byte-identical** (every win opt-in/adoption-gated). The **default-scenario follow-up below then landed three LOSSLESS default wins** — so the canonical `now (default)` column is the table above (committed baseline regenerated: S1 11,278 · S2 21,102 · S3 18,338 · S4 9,900; listtools 7,656). These are **not opt-in: every session pays them less.** Each is byte-safe for existing consumers and drops zero knowledge (proofs below). The meter still *refuses* any saving that would deliver different knowledge: under `--channel structured` it surfaces an `isError` when the `for_spawn` brief would diverge from the agent-side prime, rather than claim a divergent S3b.

**Ranked optimizations that landed (each measured, baseline vs after in its commit body):**
1. **opt #1 — single-channel wire (`WYRM_CHANNEL`, commit `8feb3f3`):** an opt-in env that drops the redundant channel (`text` drops `structuredContent`; `structured` drops the derived `content[0].text`). Default `both` = the byte-compatible baseline. Working-hour saving **−49.0% (text) / −3.1% (structured)**; fleet **−54.1% (text)**; vendor-neutral (no Anthropic-only path, Article V).
2. **opt #2 — `session_prime for_spawn` distribute-once (commit `133ac31`):** orchestrator primes once per `(run_id, role)` (4 role-compiles for a 4-role/12-agent run) and embeds each brief in the same-role agents' spawn prefix, so the 12 per-agent prime deliveries collapse to 4 on the Wyrm wire — **−38.9% vs the S3 floor**, byte-equal brief to the agent-side prime (`brief_byte_equal_to_s3: true`). A forgetful orchestrator pays the S3 floor unchanged (byte-stable CAS fallback).

**Phase 0 — the meter itself (commit `7764208`)** established the S1–S4 baselines and the cacheable-vs-churn split that turns the §7 byte-stability contracts into measurable cache value (S1 100% cacheable; S3 9,541/18,338 tokens cacheable). The #1 single cost is **10 recalls = 19,077 tokens = 82% of the working hour** — the recall-detail-tier work is the deferred next target (see the rejected/deferred pointer below). Two adversarial-review fix groups (`556e39f` majors, `b4ac9c3` minors) hardened the meter and the channel/for_spawn paths.

**Rejected / deferred:** recall detail-tiers (the 82%-of-S2 prize) deferred to a token-economy follow-on — must default to no-information-loss for existing consumers or be opt-in (§5); any default-on detail reduction was rejected for this release. Retrieval floor unaffected (no recall/ranking change landed — the no-LLM LoCoMo FTS floor r@5 52.4% stands untouched).

**Default-scenario follow-up — three LOSSLESS wins every session now pays less (commits `0c9f9f7`, `6a7c245`, `a32b786`; review fixes `2614964`/`0abc149`/`a29acb9`):** the main run's early-stop skipped the default-on, byte-safe optimizations. These cut what EVERY session pays — not an opt-in path — and each drops zero knowledge:

3. **#3 — elide spec-default annotation hints on the ListTools wire (`a32b786`):** the ≤8K default ListTools surface was AT the pin (31,999 chars = 8,000 tokens, **0 headroom**). Each advertised tool serialized all three audited hints verbatim, much of it equal to the documented MCP `ToolAnnotationsSchema` defaults (zero-information bytes). `elideDefaultAnnotations()` (`src/tool-annotations.ts`) drops, at ListTools-build time only, every hint field whose value equals the spec default (`readOnlyHint` false, `destructiveHint`/`idempotentHint` meaningful only under `!readOnly`, `openWorldHint` true; `title` always preserved). The `annotations` KEY is always kept (all-default tools serialize as `"annotations":{}`) so the §7 "100% advertised tools annotated" wire invariant holds. **Default ListTools 8,000 → 7,656 tokens (−344, now 344 under the 8K pin), byte-stable; S1 −3.0%, S4 −13.6%.** *Lossless:* every elided field's value was the spec default a consumer reconstructs identically; no non-default hint dropped (discovery + byte-stability gates re-proven).
4. **#4 — hook-push per-session dedup + per-push char budget (`0c9f9f7`):** `scripts/hooks/wyrm-push.py` now fingerprints each rendered KNOWLEDGE line (sha1 of exact bytes) into a per-session state file in the system temp dir (never `~/.wyrm`, atomic write, bounded), and never re-pushes a line already shown verbatim this session; a per-push char budget (`WYRM_PUSH_CHAR_BUDGET`, default 700) keeps whole lines then appends an explicit `…(+N more)` marker. Default-on; respects the 200ms hook budget + silent-fail. **S4 −10.5%, S2 −5.4%.** *Lossless:* dedup is IDENTICAL-BYTE only — a row whose text CHANGED has a different fingerprint and is re-pushed (verified); the budget signals withheld lines, never silently drops them; the agent already holds suppressed bytes from the earlier same-session push.
5. **#5 — number & null hygiene in the recall body (`6a7c245`):** recall is the dominant default-session cost (S2 recalls 19,076 tok); its per-result structured body carried two pure-waste byte classes. Round `relevance`/`confidence` to 4 dp (an internal ranking score read only as order + rendered as integer %, raw form carried up to 17 sig digits of float noise; the sort key is the RAW score, sorted before body build, so ordering is byte-identical) and elide genuinely-null rationale fields (a `null` carries no info). **S2 recalls 19,076 → 18,269 (−807 tok), S2 −8.9% vs baseline.** *Lossless:* rounding only excess float precision (0.0001 resolution, 100× finer than the integer-% display); null-elision only fields whose value is absent. Follow-up `a29acb9` fixed a coupled bug where the recall TEXT channel coerced a null confidence to a misleading 0% — now omitted, restoring information.

These compose: the canonical committed `bench/token-economy-baseline.json` was regenerated to the new default state (**S1 11,278 · S2 21,102 · S3 18,338 · S4 9,900**; listtools 7,656 tokens). S3 is unchanged (recall/listtools not on its hot path; the for_spawn win stays adoption-gated). Suite **1543 / 91** green; build green (150/150 annotation coverage); discovery top-3 unchanged ≥ gate; ListTools byte-stable.

### Phase F2 — run-native core + fleet negative learning (T008–T017)

Phase F2 on `feat/v7-f2-fleet` — the run-native attribution core and fleet negative learning. 14 commits (`ed44f18..337601f`): 10 task commits T008–T017 + 4 adversarial-review fix groups. Every number below is **measured** — from a committed bench, the suite, or the phase-gate run on this machine; nothing is asserted.

- **Migration 20 (T008)** — `runs` (ULID `run_id`, `parent_run_id`, `orchestrator`, `status`, `debrief_artifact_id`) + `run_agents` + nullable `agent_id`/`run_id` on 9 tables. The quarantine/authority tier landed as `failure_patterns.quarantine_scope` — the pre-existing `scope` column keeps its 6.x signature-dimension semantics (see tasks.md deviations). Migrations 21 (T015, durable confirmation-distinctness ledger) and 22 (T017, append-only prevented-repeat ledger) followed, all additive + guarded.
- **Actor envelope (T009)** — resolved ONCE at the CallTool dispatcher entry, per-field precedence: explicit param > `_meta['wyrm/actor']` > env > clientInfo; `Wyrm-Actor` header on http-fast (ranks at `source='meta'`).
- **Per-agent Ed25519 (T010)** — `agent_id`/`run_id` bound into the signed audit payload as canonical JSON with a payload-version byte in the signed message + `v2:` prefix on the stored signature; 6.x chain verification still passes on v2 rows, the out-of-band 6.x Ed25519 step fails CLOSED.
- **Cross-process write story (T011/T012)** — SQLITE_BUSY → structured `WYRM_BUSY` retry body (SEP-1303 style) + `batchWrites`; opt-in daemon-as-writer (`WYRM_DAEMON_WRITES=1`) over loopback via one internal `POST /write` endpoint, FAIL-DIRECT fallback (an unreachable daemon never costs a memory write).
- **Failure fan-out + run quarantine (T014/T015)** — `failure_check` structured verdict `{blocked, matches[], recorded_by_agent, run_id, confidence}`; run-scoped quarantine with debrief-INDEPENDENT promotion (first re-record by a 2nd distinct agent / orchestrator `promote` flag / maintenance sweep) + TTL expiry for abandoned-run noise.
- **`wyrm-guard` (T016)** — deterministic PreToolUse hook bin: BLOCK = exit 2 + stderr on identity matches, WARN = `additionalContext`, silent when clean; readonly DB open, fails OPEN. Also fixed a latent 6.x `wyrm_failure_check` gap: fuzzy probes containing `-`/`+`/`/` threw an FTS5 syntax error silently swallowed to zero recall — now phrase-quoted (strictly more matches, deterministic).
- **Prevented-repeat analytics (T017)** — `wyrm_stats view=failures`: per-run/per-agent attribution, "N repeats blocked this run", confirmations rollup, top_blocked; append-only ledger, uncached by design.

Measured:
- `bench/fleet-stress.mjs` (32 writers × 24 = 768 writes/mode), gate run: **direct** 768/768, 0 lost, 0 attribution violations, 124 structured WYRM_BUSY retries, p50 0.4ms / p95 82.84ms / max 9013.21ms, 85 writes/s; **daemon** 768/768, 0 lost, 0 violations, 0 retries, p50 31.88ms / p95 93.9ms / max 731.77ms, 646 writes/s; unauthenticated `/write` rejected (401) verified — **GATE PASS** both modes. (T013 commit runs: direct p95 21.24/23.92ms with 81/90 retries and an 8–9s busy-starvation max tail — all recovered, zero lost; daemon p95 72.38/63.02ms, max ≤453ms, 1029/1151 writes/s. Measured, not asserted: the daemon funnel removes contention at 32 writers; direct stays the first-class default at small fleets, p50 0.19ms vs 16–19ms loopback overhead.)
- `bench/failure-visibility.mjs`: **[A]** shared-SQLite write→next-check across two processes, n=50: p50 1ms / p95 2ms / max 5ms (commit runs p95 3–5ms); **[B]** SSE failure-event delivery, loopback subscriber at the production 1s poll cadence, n=60: p50 425ms / p95 907ms / max 938ms (commit runs p95 899–966ms) — target p95 < 1000ms **PASS**. Failures are `is_shared=0`: remote shared-only subscribers structurally never receive them.
- `bench/guard-budget.mjs` (30 fresh process spawns per path — node startup + readonly open + deterministic check — against a 202-row seeded DB, three runs): block p50 39.3–56.3ms / p95 44.8–62.5ms; warn p95 44.9–70.0ms; clean p95 43.5–54.9ms; absolute max observed 70.3ms — all paths under the ~200ms budget with ≥3.5× headroom.
- `bench/fleet-demo.mjs --checkers 3`: 3/3 sibling checkers blocked (confidence 1.0) by agent A's run-scoped failure #1; repeats blocked this run = 3, blocked_total = 3, violations = 0 — **DEMO PASS** (stable on a second run at `--checkers 4`: 4/4 blocked).
- Discovery bench (committed baseline): top-3 strict 61.5% / lenient 70.8% — unchanged from the 6.x baseline after description tuning (an intermediate wordier `wyrm_audit_verify` description measured 60.0/69.2, costing query aud-01 its top-3 slot, and was rejected).
- Suite at the final HEAD (`337601f`): **1278 tests / 72 suites** green, 0 failed, 0 skipped, ~18.1s (the phase-gate run at `25017a9` measured 1238/71 in ~17.9s; the 4 adversarial-review fix commits added 1 suite / 40 tests; 1065 → 1278 across the F2 range). Build: tsc green; tool-manifest 9 cloudSupported @ v6.18.0; tool count unchanged at **137 advertised, 137/137 annotated** (partition locks untouched).
- Documented constants, deliberately NOT claimed as benchmarks (Article VIII): `BUSY_RETRY_AFTER_MS=1000` and `DAEMON_WRITE_TIMEOUT_MS=2000` are caller advice/budgets; `busy_timeout=5000` is the pinned pragma value, not a latency claim.

## [6.18.0] - 2026-06-10 - The 6.x→7.0 bridge: annotated, honest, measured (BROOD Phase F1)

The bridge release before Wyrm 7.0 "BROOD". 100% additive on the 6.x surface —
no tool renamed/removed, no behavior change, no schema migration. Everything
below is annotations, honest descriptions, opt-in capture, CI floors, measured
baselines, and deprecation warnings.

### Added
- **MCP tool annotations on all 137 tools.** Every tools/list entry now carries
  audited `readOnlyHint`/`destructiveHint`/`idempotentHint` (SDK 1.29.0 shape) —
  previously 0 tools were annotated. Partition from a handler-by-handler audit
  (NOT the legacy caching sets, which cover only 63/137 and disagree twice):
  60 read-only / 77 write / 15 destructive / 105 idempotent. Central registry in
  `src/tool-annotations.ts`; `npm run build` fails on any unannotated advertised
  tool (`scripts/check-annotation-coverage.mjs`).
- **`WYRM_GOLDEN_CAPTURE` (default OFF).** Opt-in full-fidelity golden-transcript
  capture for the 7.0 alias-spine replay-equivalence proof: JSONL records (full
  request/response, no truncation) appended to `~/.wyrm/golden/` (0700 dir /
  0600 files, `WYRM_GOLDEN_DIR` to relocate), recursive secret redaction +
  value-pattern safety net, failure-isolated with a 5-failure circuit breaker.
  Covers all 4 response sites in the CallTool handler, including the cache-hit
  early return. Plus 137 deterministic synthetic fixtures under
  `tests/golden/synthetic/` — mandatory, since only 43/137 tools (31.4%) had any
  organic traffic in the measured 3-week tool-call log.
- **No-network deterministic-core CI** (`tests/no-network-core.test.ts`):
  Article III as a test — the network is killed and every core memory op must
  still pass. Core memory ops never require LLM or network.
- **Discovery bench, baseline-record mode** (`tests/discovery.bench.ts`): 65
  natural intent queries across 16 categories, ranked by an in-repo Okapi BM25
  (k1=1.2, b=0.75, deterministic, offline) over the exact name+description
  corpus the wire serves. Measured 6.x baseline: top-3 lenient **70.8%**, top-1
  strict 44.6% (clean cohort top-3 lenient 72.7%). Recorded to
  `specs/wyrm-v7-brood/research/discovery-baseline.json`; the ≥90% hard gate
  (`WYRM_DISCOVERY_GATE=1`) arrives with the rewritten 7.0 surface.
- **Secure-defaults CI guard (quest #81)** + hygiene floor
  (`tests/hygiene-security-floor.test.ts`): loopback bind, minimal spawn env,
  readonly-gate, and token-auth defaults can no longer silently regress;
  migrations-array monotonicity asserted; WYRM_TOOL_COUNT/profile-header drift
  locks; the v5.10 agent-daemon suite's ESM execution contract pinned.
- **wyrm-http route-level access log** (`~/.wyrm/http-server-access.log`,
  timestamp+method+path only) — the evidence trail for the legacy server's
  deletion in 7.0.

### Changed (honesty, not behavior)
- **Description truth pass — 5 verified lies corrected** against the audited
  implementation: `wyrm_recall` (real convex-blend hybrid retrieval, not the
  stale "2-stage FTS + tag"), `wyrm_capture` (review-queue routing +
  `project_id` requirement disclosed), `wyrm_failure_check` (real match order:
  exact signature then FTS-fuzzy, occurrences/last_seen DESC, LIMIT 5),
  `wyrm_review` (reject is a literal hard DELETE), `wyrm_events_subscribe`
  (verified pure read — no persistent subscription).
- **Measured surface baseline published** (spec success criterion 1): full
  ListTools = 137 tools ≈ 77,401 chars (~19.4K tokens); the default `standard`
  profile = 62 tools (~9.8K tokens); `essential` = 25 (~4.6K). The 7.0 target
  is ≤32 advertised tools / ≤8K tokens.

### Deprecated (warnings only — behavior unchanged in 6.18)
- **`wyrm-http`** (the legacy second HTTP server, `src/http-server.ts`) warns on
  startup: deprecated, **deleted in Wyrm 7.0** — use `wyrm serve`.
- **`WYRM_PROFILE=full`** emits a one-time notice: in 7.0 it becomes a permanent
  synonym for `legacy` (every 6.x name stays callable via the alias spine).
- **22 CLI-exile candidates** (spec FR-4: 18 operator tools + the device-egress
  quartet) now advertise a `[Deprecated in v7 → use the wyrm CLI: <cmd>]`
  description suffix and emit a one-time-per-tool stderr note when invoked via
  MCP (failure-isolated; registry in `src/deprecations.ts`). **Exception:**
  `wyrm_cloud_backup` and `wyrm_sync_export` remain *functional* through 7.0.x
  with a published sunset, so scheduled backups never silently stop.

Suite: **1038 tests / 61 suites** green.

## [6.17.0] - 2026-06-08 - Multi-backend secret vault + update-safe master key

### Added
- **Multi-backend credential vault.** The vault master key can now live in the
  **OS keychain** (Secret Service/libsecret on Linux, Keychain on macOS) so it is
  never on disk — a stolen `~/.wyrm` (backup, synced dotfiles, leaked tarball) has
  the ciphertext but not the key. Three backends, auto-resolved (override with
  `WYRM_VAULT_BACKEND`): `keychain` (safe default on a desktop), `passphrase`
  (`WYRM_VAULT_PASSPHRASE`, ideal headless/CI), and legacy `keyfile`.
- **`wyrm vault setup` / `wyrm vault secure [--backend keychain|passphrase]` / `wyrm vault info`.**
  `secure` atomically migrates a vault to a safer backend: back up → re-key (rotate)
  → verify every secret decrypts → roll back on any failure → shred the plaintext
  keyfile. Session-prime now surfaces a one-line advisory when the vault is insecure.

### Fixed / Security
- **Master key is never silently re-minted for an existing vault.** Previously, a
  build that resolved a different backend than the one a vault was created under
  (e.g. a keychain vault opened where the keyring is locked/absent) could mint a
  fresh keyfile and then fail with a misleading `decrypt failed` — effectively
  orphaning the vault. `masterKey()` now **refuses to create a key when `vault.enc`
  already exists** and fails loudly with the backend that's actually required.
- **Backend marker (`vault.meta`, non-secret).** Records which backend a vault was
  created/secured under, so any build (and any future update) resolves the right
  backend instead of falling through to an insecure/incompatible one.
- AES-256-GCM (per-encrypt random IV + integrity), `execFileSync` with no shell,
  secret passed to the keyring via stdin (Linux). 17 vault tests incl. an
  upgrade-orphaning regression test. Adversarially reviewed (18 findings → 0 real).

## [6.16.0] - 2026-06-07 - Read-only dashboard + cloud-tool manifest

### Added
- **Read-only / public-view mode (#33):** `WYRM_UI_READONLY=1` makes the dashboard
  safe to expose beyond localhost. A single dispatcher-level chokepoint (pure,
  unit-tested `readonly-gate.ts`) 403s every mutating request (gated by HTTP
  method, so all current AND future write routes are covered) plus the GET reads
  that egress `is_shared` rows off-box (`/sync/push`, `/events`, `/events/stream`)
  or over-share machine internals (`/ui/homes`, `/audit`). The client hides the
  account switcher + review controls to match (the server gate is the real
  boundary). Startup warns on a non-loopback bind without it. 31 new tests.
- **Cloud-tool manifest (#34):** the build now emits `dist/tool-manifest.json`
  (from `cloud-tools.json`) marking which tools are cloud-supported. This is the
  core half of the Wyrm Cloud connector's auto-inherit contract — the connector's
  CI reads the published manifest and goes red if core adds a cloud tool the
  connector hasn't ported, so web/phone parity can't silently drift.

## [6.15.0] - 2026-06-07 - Grove sync isolation + cloud resilience + ghosts.lk dashboard

### Added
- **Grove sync policy (#29):** per-project `projects.sync_policy` (`private` |
  `cloud` | `team`), **private by default**, gating EVERY replication egress
  channel (cloud sync, Live-Memory events, federation share + pull, team HTTP
  pull). Safe-by-structure isolation — one accidental `is_shared`/visibility flip
  can no longer leak a private grove. Migration 19 (additive, conservative backfill).
- **`wyrm grove` CLI + dashboard surfacing (#30):** set/inspect a project's sync
  policy; account badge + Skills tab in the web dashboard.
- **Constitution Article IX:** two cloud editions — **Wyrm Sync** (zero-knowledge
  E2E) vs **Wyrm Cloud** (managed, server-readable connector, `mcp.wyrm.ghosts.lk`).

### Changed
- **Cloud sync client resilience (#31):** retries 429/5xx/network with exponential
  backoff honoring `Retry-After` (server writes are idempotent, so replay is safe).
- **Web dashboard re-themed to the ghosts.lk brand** (Stealth-Silver `#050505` +
  silver primary + green LED + Geist/JetBrains Mono), replacing the prior purple theme.

## [6.14.1] - 2026-06-06 - Security & reliability hardening (audit pass)

A focused hardening release from a full multi-agent security/accuracy/performance
audit (25 raw findings → 21 adversarially confirmed → 18 fixed). No new features,
no schema change, no API change — behavior is preserved except where it strictly
improves. **881/881 tests pass.**

### Security
- **Subprocess env leak (high):** outbound MCP clients (`wyrm_call_external`) no
  longer inherit Wyrm's entire `process.env` when spawning an external server —
  which leaked every secret in the parent env (`ANTHROPIC_API_KEY`, `WYRM_*`
  tokens, `OPENAI_API_KEY`, …) to a third-party binary. Now passes a minimal
  non-secret env allowlist + only the server's own declared `env`.
- **SQL injection (high):** `wyrm_prune`'s `project_id` was string-interpolated
  into a `SELECT`. Now integer-validated and bound as a parameter.
- **HTTP binds to loopback by default:** the Fast/HTTP APIs and `wyrm` CLI server
  now listen on `127.0.0.1` (set `WYRM_BIND_HOST=0.0.0.0` to expose, with a loud
  warning). Closes LAN reachability of the bearer surface and the per-IP rate-limit
  / `X-Wyrm-Origin` trust weaknesses.
- **Private events stay private:** the SSE stream now *server-enforces* shared-only
  for any non-loopback subscriber — `is_shared=0` events can't leak to a remote
  peer regardless of what the client requests. Added a per-IP SSE connection cap.
- **Vault KDF salt:** passphrase mode now derives the key with a random,
  per-install persisted salt (was a single hardcoded global salt); vault dir is
  created `0700`. Existing vaults keep decrypting (back-compat preserved).

### Reliability & accuracy
- **Recall never throws:** an operator-/punctuation-only query no longer throws out
  of `wyrm_recall` / `wyrm_context_build`; it degrades to recency. FTS NUL bytes
  are stripped at the source (`buildFtsMatchQuery`), protecting every caller
  (incl. `wyrm_skill_list`).
- **Indexer crash-recovery:** queue rows orphaned in `processing` by a crash/kill
  are reclaimed on the next batch — embeddings are no longer silently lost.
- **Rerank weight guard:** `WYRM_RERANK_ALPHA` is finite-clamped to `[0,1]`
  (a malformed value no longer poisons fusion ranking).

### Performance
- **Sync busy-wait removed:** the resilience retry backoff replaced a CPU spin-loop
  with `Atomics.wait` (0% CPU while waiting).
- **N+1 removed:** markdown quest parsing hoists its dedup lookup to a single set.

## [6.14.0] - 2026-06-06 - GOD-SKILL SPEC v2 tiers + Wyrm-native spec-kit

Apex skill governance (god > mega > atomic) and spec-kit-as-quests become first-class. Fully additive + backward-compatible: a new migration adds columns with safe defaults, new tool params are optional, and two new tools are added — no existing behavior changes.

### Added
- **Migration v18** — `skills` gains three governance columns (additive, guarded, safe defaults): `tier TEXT NOT NULL DEFAULT 'atomic'` (`CHECK IN ('atomic','mega','god')`), `governs TEXT NOT NULL DEFAULT '[]'` (JSON string array), `composes TEXT NOT NULL DEFAULT '[]'` (JSON string array) + `idx_skills_tier`. Also creates the **`specs`** registry table (`project_id`, `spec_dir`, `title`, `summary`, `task_count`, `UNIQUE(project_id, spec_dir)`) for the spec→project link. Pre-existing skills converge to `tier='atomic'` / `'[]'` / `'[]'`.
- **`wyrm_skill_register`** now accepts optional `tier` (`atomic|mega|god`), `governs` (string[]), `composes` (string[]). Omitting them preserves the exact prior behavior.
- **`wyrm_skill_search`** + **`wyrm_skill_list`** gain an optional `tier` filter (FTS rank order preserved when filtering search).
- **`wyrm_skill_graph`** (tool #136) — returns the GOD-SKILL SPEC v2 governance graph: walks god → governs(mega) → governs(atomic) plus each node's `composes`, as a nested tree **and** a flat edge-list (+ machine-readable JSON). Optional `root` scopes to one apex; omitting it returns the forest of every `god`-tier skill. Cycle-safe; flags unresolved `governs` targets as `missing`.
- **`wyrm_spec_register`** (tool #137) — makes GHOSTMESH spec-kit Wyrm-native. Reads a spec dir (`spec.md` title/summary + `tasks.md`), parses the task list (`T001 …`, `- [ ] …`, `- [x] …`, `1. …`, `[P]` markers, code-fence/heading-aware), and creates **one Wyrm quest per task** linked to a project, persisting the spec→project link in `specs`. **Idempotent** — re-running updates the same quests (deduped by a per-spec-task tag signature `spec:<dir-slug>:<task-id>`) instead of duplicating, and picks up newly added tasks.
- `src/spec-kit.ts` — pure parser/util module (`parseTasksMarkdown`, `parseSpecMarkdown`, `readSpecDir`, `specTaskSignature`).
- DB layer: `getSkillGraph(root?)`, `upsertSpec`, `getSpec`, `listSpecs`, `findQuestBySpecSignature`, `updateQuestFields`; `Spec` + `SkillGraphNode` types; `tier`/`governs`/`composes` on the `Skill` type. `wyrm_skill_graph` + `wyrm_spec_register` added to the `standard` tool profile; `WYRM_TOOL_COUNT` 135 → 137.

### Tests
- `god-skill-spec-v2.test.ts` (21): tier/governs/composes register round-trip + upsert + tier filters on list/search; `getSkillGraph` god→mega→atomic walk, root-scoping, missing-target flagging, cycle-safety; tasks.md/spec.md parsing (T-ids, checkboxes, numbered, fences, markers); deterministic signature; spec→quest creation, idempotent re-run, and new-task pickup. **881 tests, 51 suites — full suite green.**

## [6.13.0] - 2026-06-04 - Auto-extraction: wyrm_auto_capture (bet #2 MVP)

### Added
- **`wyrm_auto_capture`** (tool #135) + `src/auto-capture.ts` — extract durable memories (truth / failure / decision / pattern / lesson) from freeform text into the review queue (`needs_review=1`). Pluggable LOCAL extractor: an Ollama model (`WYRM_EXTRACT_MODEL` or the `model` arg — the DragonSpark slot) with a deterministic heuristic fallback. Never a cloud LLM; never blocks/throws; deduped by sig; candidates approved via `wyrm_review`. Closes the "just talk, it remembers" gap.
- Verified with real Ollama (mistral-nemo): cleanly extracted failures, decisions, truths, lessons, patterns from session notes.

### Tests
- `auto-capture.test.ts` (7): deterministic extraction + kind classification, robust LLM-output parsing (markdown-wrapped JSON, malformed/unknown items), mapping, no-model path. **860 tests, 50 suites.**

## [6.12.0] - 2026-06-04 - Convex score-fusion default in recallHybrid (+1.8pt recall@5)

### Changed
- **`recallHybrid` fuses by convex score-blend by default** (was RRF): `α·(min-max-normalized cosine) + (1-α)·(rank-normalized FTS)`, α=0.7, candidate depth 50 (was `max(25, limit*3)`). `WYRM_RERANK_FUSION=rrf` reverts; `WYRM_RERANK_ALPHA` tunes the vector weight. Vector cosine similarity is now threaded through fusion (was discarded). (`memory-artifacts.ts`)
- **Measured (real LoCoMo, 1,982 evidence QA, local):** recall@5 60.3%→62.1%, recall@10 72.2%→72.7%, MRR ≈.447. Prod-path verified (`bench/locomo-prod.mjs`) to reproduce the grid.

### Added
- `bench/locomo-grid.mjs` — embed-once fusion sweep (rrf vs convex × candidate depth × α). `bench/locomo-hybrid.mjs` gained `--fusion`/`--alpha`. `bench/locomo-prod.mjs` runs the real shipped `recallHybrid` path.

### Decision (multi-agent design panel)
- A 5-agent panel scored 4 reranker approaches; chose convex fusion (zero deps, local-fit 10) over the ONNX cross-encoder (720 MB native dep → deferred to opt-in, MiniLM int8 first) and the Ollama-LLM reranker (disqualified: ~88s/query measured on the 8 GB box). The grid falsified "deeper candidates lift recall@10" for RRF.

Suite 853, 49 suites.

## [6.11.0] - 2026-06-04 - Secret vault (`wyrm vault`) — AES-256-GCM local secrets

### Added
- **`src/vault.ts` + `wyrm vault` CLI** — encrypted-at-rest local secret store. Secrets are AES-256-GCM-encrypted in `~/.wyrm/vault.enc`; the key is a random 256-bit value in `~/.wyrm/vault.key` (0600, auto-created) or derived from `WYRM_VAULT_PASSPHRASE` (no key file on disk). Subcommands: `set` (reads STDIN, never argv), `get`, `list` (names only), `rm`, `exec <name> [--as VAR] -- <cmd>` (injects the secret into the subprocess env — never prints it), `import-npm`, `info`. GCM integrity: tamper / wrong key → decrypt throws (fails closed, never garbage). The point: store a credential once, use it without it ever reappearing in plaintext — so a leaked transcript can't expose it and you don't have to roll it. `vault.test.ts` (6).

Suite 853, 49 suites.

## [6.10.0] - 2026-06-04 - Hybrid recall (FTS ⊕ vectors, RRF) in wyrm_recall

### Added
- **`MemoryArtifacts.recallHybrid`** — fuses FTS (bm25) + dense-vector (cosine) candidates via Reciprocal Rank Fusion (k=60). `wyrm_recall` calls it when a vector store is available; **graceful fallback** to lexical `recall()` when there's no store, the provider throws, or nothing is indexed yet. `match_type` extended with `vector` / `hybrid`. (`memory-artifacts.ts`, `index.ts`)
- **Artifact dense-indexing** — active artifacts (`needs_review=0`, not superseded) are embedded best-effort on `add` (fire-and-forget — never blocks/throws) and via `wyrm_reindex` (now indexes `memory_artifacts`; `vectors.addVector` content-type widened with `artifact`).

### Performance / model choice
- Data-driven A/B on real LoCoMo: default embedding model stays **`nomic-embed-text`** (768-d). `mxbai-embed-large` (1024-d) gave only +0.7pp recall@10 for 2.4× the size — opt in with `WYRM_EMBED_MODEL=mxbai-embed-large`.
- **Measured (real LoCoMo, local, no cloud):** recall@5 52.4%→**60.3%**, recall@10 59.9%→**72.2%**, MRR .393→**.447**.

### Tests
- `recall-hybrid.test.ts` (3, stubbed provider): RRF fusion + vector-only-hit surfacing, no-store fallback, provider-throws fallback. **847 tests, 48 suites.**

## [6.9.2] - 2026-06-04 - Severe recall bug fixed (recency→relevance), caught by LoCoMo

### Fixed
- **`memory.recall` returned recency, not relevance (HIGH — affected every `wyrm_recall` / `wyrm_context_build`).** `searchByFts` aliased the FTS5 table (`JOIN memory_artifacts_fts fts … WHERE fts MATCH ?`); `<alias> MATCH ?` throws `no such column` in SQLite FTS5, so every natural-language recall silently fell back to `listRecent()`. Fix: reference the FTS table by **full name** in `MATCH`; also route through `sanitizeFtsQuery`/`buildFtsMatchQuery` (the raw query's `?` threw too) and **propagate bm25 rank** into the relevance score (was a flat 0.7 per FTS hit → no intra-candidate ranking). Isolated to this path — all other FTS searches already use the full table name. (`memory-artifacts.ts`)

### Added
- `bench/locomo-real.mjs` + `bench/locomo-harness.mjs` — LoCoMo/LongMemEval-style **evidence-based retrieval** benchmark (no-LLM recall@k / MRR by category). Real-LoCoMo after the fix: **recall@1 29.5%, recall@5 52.4%, recall@10 59.9%, MRR 0.393** (FTS-only; was 0.2/1.5/2.6/0.008 before).

### Tests
- `recall-ranking.test.ts` (3): relevance-not-recency (relevant=oldest row still ranks #1), punctuated query doesn't fall back, topic discrimination. **844 tests, 47 suites.**

## [6.9.1] - 2026-06-04 - Open the standard: WMP spec + AGPL dual-license clarity

### Added
- **`PROTOCOL.md` — Wyrm Memory Protocol (WMP) v0.1.** Backend-agnostic, freely-implementable open spec (CC-BY-4.0) for sovereign local-first AI memory: data model, event-log/replication format, identity/trust, and the MCP read/write surface. Failure-blocking + decision causality are first-class protocol entities; §9 forbids requiring the network or an LLM for a Core op.
- **`COMMERCIAL.md`** — dual-licensing terms (AGPL open use vs commercial embed/managed-service).

### Changed
- **License reconciled to AGPL-3.0-or-later.** 68 source headers updated "Proprietary" → AGPL dual-license; root + `lsp-server` LICENSE files set to AGPL-3.0 (matching `mcp-server`, which was already AGPL); `package.json` `license` → `AGPL-3.0-or-later`. README badge + License section updated (spec open / implementation copyleft). **No code behaviour change.**

### Tests
- 841 / 46 suites green (metadata + docs only).

## [6.9.0] - 2026-06-04 - Live Memory: full-surface producer coverage + retention

### Added
- **Widened Live Memory producers.** Beyond quest + session, these writes now emit events: ground-truth `set` → `truth` (`intelligence.ts`), active memory `add` → `capture` (`memory-artifacts.ts`), failure `record` → `failure` (`failure-patterns.ts`), decision `link` → `decision` (`causality.ts`). Two new `EventKind`s (`failure`, `decision`) added to the closed anti-spoof set. All emit **reference-only** (refTable + refId, no payload).
- **Event-log retention** — `pruneEvents(db, { olderThanDays, maxPerProject })` in `events.ts` (+ `WyrmDB.pruneEvents`), wired into `wyrm_maintenance`. Default keep 90d / 5000 per project; `WYRM_EVENT_RETAIN_DAYS` / `WYRM_EVENT_MAX_PER_PROJECT` override. Bounds the derived log so widened coverage can't grow the DB unbounded.

### Performance / Security
- Memory `add` emits ONLY when `!needsReview`, so bulk harvest candidates (1000s) don't flood the log.
- Failures emit **private** (`is_shared=0`) — failure detail never replicates. Reference-only everywhere → token-lean reads + no content in the replicated stream.
- `emitEvent` stays failure-isolated (never throws → can never regress a canonical write); presence heartbeats intentionally not wired (flood vector).

### Tests
- New `live-memory-coverage.test.ts` (8): per-kind emission, review-queue gate, reference-only payloads, off-switch no-op, retention age-sweep + per-project cap, no-throw on missing table. **841 tests, 46 suites.**

## [6.8.2] - 2026-06-04 - Scrub the last 🐉 from shipped output; verify clean-room install

### Fixed
- **Dragon emoji removed from shipped output.** 6.8.0 cleared `src/`, but three *shipped* files still carried 🐉: `scripts/postinstall.cjs` (install banner), `ui/index.html` (static dashboard title/favicon/brand — now uses `/ui/dragon-mark.svg`), and `skills/ghost-protocol-portfolio/SKILL.md` (heading). All scrubbed — zero 🐉 in the published tarball.

### Changed
- `prepublish` → `prepublishOnly` (deprecated lifecycle; guarantees `dist/` is rebuilt at publish time).

### Verified (no code change)
- Clean-room `npm install wyrm-mcp` on a pristine box: 137 packages, exit 0, **prebuilt better-sqlite3** binary (no compile), only a harmless transitive `prebuild-install` deprecation warning. First run applies migrations to **v17**, scaffolds `~/.wyrm`, MCP `initialize` + `tools/list` (59 tools, lean `standard` profile) succeed, `wyrm-lsp` loads clean. `preinstall`/`postinstall` are try/catch-wrapped so they can never abort an install; non-prebuilt platforms (Alpine/musl, Termux, uncommon arch) get a build-tool hint + Termux `common.gypi` patch.

### Tests
- New `brand-no-emoji.test.ts` — scans all shipped paths (`src`/`ui`/`skills`/`scripts`/`README`) and fails if 🐉 reappears. **833 tests, 45 suites.**

## [6.8.1] - 2026-06-04 - Self-audit fixes: agent-loop dead-ends, dashboard split, migration typo

### Fixed
- **Agent-loop dead-end (HIGH).** `SAFE_INTERNAL_TOOLS` (agent-loop.ts) whitelisted `wyrm_capture`, `wyrm_recall`, `wyrm_remember`, `wyrm_distill`, `wyrm_global_context` for the OODA loop, but `internalDispatch` (index.ts) had no `case` for them — calling any returned `{ok:false, error:"…not implemented in dispatcher"}`. All five are now implemented. (`index.ts`)
- **Two-dashboard split (MED).** The `wyrm-ui` binary spawned `wyrm-http` (`http-server.ts` → a static `/ui` without the brand mark) while `wyrm ui` / `wyrm serve --ui` served `http-fast.ts` (the live SPA + `/dragon-mark.svg`). Both bind `:3333`. `wyrm-ui` now delegates to `wyrm serve --ui` so every dashboard entry point is consistent. (`wyrm-ui.ts`)
- **Migration named a non-existent table (MED).** Migration 14 added `cross_project_visibility` to a list including `'decisions'` (real table: `decision_edges`), so the `ALTER` silently no-op'd and visibility was never enforced for decision edges. Corrected for fresh installs + **migration 17** backfills already-migrated DBs (verified: applies to v17, column present). (`migrations.ts`)

### Tests
- New `tool-surface-integrity.test.ts`: (1) asserts every `SAFE_INTERNAL_TOOLS` entry has an `internalDispatch` case (anti-drift for the bug above); (2) asserts `WYRM_TOOL_COUNT` equals the real advertised/handled count — which is **134** (the audit's "off-by-one" was a miscount; the constant was already correct). **832 tests, 44 suites.**

## [6.8.0] - 2026-06-04 - Harvest reads code + the silver dragon brand mark

### Added
- **Harvest code signals.** `wyrm_harvest` / `wyrm harvest --code` (`includeCode`) now extracts code signals alongside doc facts + git subjects: the language/framework stack from manifests (`package.json`, `Cargo.toml`, `composer.json`, `pyproject.toml`, `go.mod`, `pom.xml`, `*.csproj`, `Gemfile`) → `lesson`; `TODO`/`FIXME`/`HACK`/`XXX` markers via `git grep` → `anti_pattern` (capped 25/project). The run also indexes the project **symbol graph** (functions/classes/types) so code structure is queryable. Candidates land in the review queue (`needs_review=1`), deduped by sig. `--code`/`includeCode` is opt-in. (`harvest.ts`, `wyrm-cli.ts`, `index.ts`)

### Changed
- **Brand mark: the 🐉 emoji is replaced by the Ghost Protocol silver dragon.** `ICON.brand` is now a Nerd Font dragon glyph (`U+F115D`, override with `WYRM_BRAND`), rendered in silver truecolor (`#C0C0C0`) in the statusline and CLI banners. The web dashboard logo uses the real silver-dragon **SVG**, served at `GET /dragon-mark.svg` (cached, `image/svg+xml`, included in `package.json` `files`). 212 emoji replaced across 16 source files; the npm `description` and READMEs were cleaned too — zero 🐉 in shipped output. (`icons.ts`, `statusline.ts`, `ui-dashboard.ts`, `http-fast.ts`, `ui/dragon-mark.svg`)

### Tests
- `statusline.test.ts` asserts the silver brand mark (`ICON.brand`, not the emoji); `harvest.test.ts` covers code-signal extraction. **830 tests, 43 suites, green.**

## [6.7.1] - 2026-06-04 - Intensive bug hunt: 6 real bugs fixed (2 security)

An adversarial edge-case audit (parallel auditors brute-forcing inputs against the real engine) found genuine bugs in shipped code. All fixed with regression tests.

### Security
- **[High] `removeSkill` with a degenerate slug could `rmSync` the entire skills directory.** A name like `"..."`, `"🐉"`, or `"/////"` slugifies to `""`, so `join(skillsDir, "")` resolved to the skills ROOT — `removeSkill` would have deleted every skill, and `deploySkill` would write `SKILL.md` into the root. Both now hard-refuse an empty slug. (`skill-author.ts`)
- **[High] SSRF: IPv4-mapped IPv6 in hex form bypassed the loopback/private block.** `http://[::ffff:127.0.0.1]/` is normalized by WHATWG URL to the *hex* form `::ffff:7f00:1`, which `isBlockedHost` didn't decode → reached loopback. Now both the dotted and two-hex-group tails are parsed back to IPv4 and checked. Also blocks trailing-dot `localhost.` and the NAT64 `64:ff9b::` prefix. (The decimal/hex/octal IPv4 loopback forms were already safe via URL normalization.) (`repl-guard.ts`)

### Fixed
- **[Med] NUL/control byte in a search query crashed FTS.** `\x00` survived `sanitizeFtsQuery` and made `quests_fts MATCH` throw `unterminated string`. Control chars (`\x00–\x1f`, `\x7f`) are now stripped first, and the `search*` methods wrap `MATCH` in try/catch (defense-in-depth: malformed → `[]`, never throws). (`security.ts`, `database.ts`)
- **[Med] priority-embed orphan/duplicate markers.** A truncated block (START without END) or two blocks could accumulate; embed/remove now strip ALL managed markers (well-formed + orphan) and place exactly one fresh block — orphan-safe, content always preserved. `embedAll`/`removeAll` isolate per-target failures (`'failed'`) instead of aborting the sweep. (`priority-embed.ts`)
- **[Low] harvest in-run duplicates + control chars.** Two byte-identical doc sections in one run now insert once (in-run dedup `Set`); control bytes are stripped from harvested heading/body. (`harvest.ts`)

### Tests
- `tests/intensive.test.ts` (16) — FTS control-byte safety, SSRF host-encoding matrix, skill empty-slug data-loss guard, embed orphan/dup healing, harvest dedup/hygiene. Full suite: **827 passing** (43 suites).

## [6.7.0] - 2026-06-04 - Harvest: auto-populate memory from what you already produce

6.6.0 fixed *retrieval*; the corpus was still thin (94 items) because population was manual. Harvest fixes the supply side — without a daemon, without noise.

### Added
- **`wyrm_harvest`** MCP tool + **`wyrm harvest`** CLI (tool count **133 → 134**). Walks a project (or all registered projects) and pulls:
  - **durable facts from docs** — each heading + lead paragraph of `README.md` / `CLAUDE.md` / `AGENTS.md` / `ARCHITECTURE.md` (your curated ground-truth, previously ignored), and
  - **recent commit subjects** from `git log --no-merges` (skips trivial/wip).
  Everything lands in the **review queue** (`needs_review = 1`) — approve/reject with `wyrm_review`, nothing is auto-trusted. `src/harvest.ts` (DB-injected, so the extraction logic is unit-testable).
- **Idempotent** — each candidate carries a deterministic dedup signature in its tags, so re-running skips what's already harvested. `dryRun` previews.

### Why
The corpus was thin because Wyrm relied on you to type memories in. Harvest turns the artifacts you *already* maintain (docs + commits) into review-gated candidates. On this workspace a dry-run surfaced **1,524 candidates across 90 projects** — a ~16× corpus, all gated. Pairs with 6.6.0: more good memory + retrieval that can now find it.

### Tests
- `tests/harvest.test.ts` (7) — doc extraction (heading+lead, thin-section skip), git extraction (merge/trivial skip, stable `git:` sigs), idempotency, dry-run-writes-nothing, multi-project totals. Full suite: **811 passing** (42 suites). Live dry-run verified on 90 real projects.

## [6.6.0] - 2026-06-04 - Recall that actually recalls: OR + bm25 + porter stemming + measured vector ceiling

The effectiveness harness (6.5.x) exposed the real problem: Wyrm's search was *fast but missed*. On paraphrase queries — the normal case, where the agent uses different words than you stored — it returned **nothing** (0% hit-rate, MRR 0.00). Three root causes, all fixed.

### Fixed
- **Implicit AND → OR + bm25 ranking.** `searchQuests` / `searchSessions` / `searchData` / `searchSkills` (and skill-list) built `MATCH 'a b c'` — an implicit **AND**, so one unfamiliar word killed the whole query, and results were ordered by **recency**, not relevance. They now **OR** the terms (any shared word surfaces the row) and `ORDER BY bm25()` (best match first). New `buildFtsMatchQuery()` in `security.ts`.
- **No stemming → porter tokenizer (migration v16).** The FTS tables are rebuilt with `tokenize='porter unicode61'`, so `override`/`overrides`/`overriding` and `page`/`pages` match. The base-table triggers reference the FTS tables by name, so the rebuild needs no trigger changes; `INSERT … VALUES('rebuild')` repopulates from the external content.
- **Vectors proven (the semantic ceiling).** `wyrm_search` was already hybrid-capable (lexical + vector + RRF fusion) but the index sat at 0% coverage. The harness now measures the ceiling using Wyrm's own `OllamaProvider` (`nomic-embed-text`). Populate the index with `wyrm reindex` to activate hybrid.

### Measured (`bench/wyrm-effectiveness.mjs` — 12 memories, 16 labeled queries)

| Paraphrase (the real case) | hit-rate | MRR |
|---|---|---|
| Before (AND, recency) | **0%** | **0.00** |
| After OR+bm25+porter | 100% | 0.67 |
| After + vectors | 100% | **0.94** |

OR+bm25+porter fixed **recall** (the memory is *found*); vectors fix **ranking** (it's found at *rank 1*). Speed was never the problem — this was.

### Tests
- `tests/search-quality.test.ts` (6) — porter stemming, OR semantics, bm25 ranking, empty-query safety. Full suite: **804 passing** (41 suites).

## [6.5.1] - 2026-06-04 - Lean by default: stop shouting 133 tools at the model

A discipline fix, not a feature. 133 defined tools dumped on the model is a firehose — it dilutes attention and raises tool-selection error. So the **default surfaced set is now the curated `standard` profile (~58), not `full` (133)**. Every tool still exists and still works; `full` is one env var away.

### Changed
- **Default tool profile reformed.** Claude Code, Codex, Windsurf, and unknown clients now default to **`standard` (~58 tools)** instead of `full` (133). Cap-constrained clients (VS Code, Cursor, Antigravity) stay on `essential` (25). `full` is **opt-in** via `WYRM_PROFILE=full`.
- **Zero feature loss.** Hidden ≠ disabled — every tool remains *callable*; the profile only governs what's *advertised* in `ListTools`. The long tail (replication internals, audit, symbol graph, orchestration, hour-ledger, invoice, …) is one env var away when a specific workflow needs it.
- Added the operator-facing 6.5 tools (`wyrm_embed`, `wyrm_skill_create`) to `standard` so the default surface keeps them.
- Package description no longer leads with the tool count.

### Why
A memory layer earns its place by being *used well*, not by *advertising the most tools*. Surfacing all 133 trains the model to thrash; a curated working set it can hold in attention is more effective. This is the first cut, not the last — the standing discipline is to keep the default lean and resist reflexively adding tools to it.

### Tests
- `tests/tool-profiles.test.ts` + `tests/bug-regression.test.ts` updated for the new defaults (Claude Code → standard, unknown → standard, full opt-in). Full suite: **798 passing** (40 suites).

## [6.5.0] - 2026-06-04 - Omnipresence: first-priority memory, persistent buddy, atomic skill authoring

Make Wyrm impossible to ignore: it embeds itself as FIRST-priority, always-loaded memory across CLIs, shows the buddy in the TUI at all times, primes proactively, and can author + deploy atomic skills on demand. Tool count **131 → 133**.

### Added — Priority embedding (Wyrm-first)
- **`wyrm_embed`** MCP tool + **`wyrm embed`** CLI (`action: install | status | remove`). Prepends a managed, marker-delimited "🐉 Wyrm is your memory — consult it FIRST" directive to the TOP of each client's always-loaded instruction file: `~/.claude/CLAUDE.md` (global), and with `projectPath` the project `CLAUDE.md` / `AGENTS.md` (+ `--all`: `.cursorrules`, `.windsurfrules`, `.github/copilot-instructions.md`). Idempotent (updates in place, never duplicates) and fully reversible. `src/priority-embed.ts`.
- **MCP server `instructions`** now assert Wyrm-first at the protocol handshake (clients surface it as server guidance), and the server version string is read from package.json instead of a stale literal.
- **`install` also wires the proactive hooks + the buddy statusline** — so one call makes Wyrm read-first, primed-proactively, and visible-always.

### Added — Persistent buddy + easy dashboard
- **`installClaudeStatusline()`** + **`wyrm statusline [--remove]`** — wires Wyrm's existing statusline into Claude Code's `statusLine` setting, so the dragon + live memory (`🐉 Wyrm · <project> · <open quests> · <blocked> · <truths>`) shows in the TUI **at all times**, not just when a tool is called. Won't clobber a non-Wyrm statusline.
- **`wyrm ui` / `wyrm dashboard`** — aliases for `wyrm serve --ui` that start the HTTP server and open the `/ui` dashboard in the browser.

### Added — Atomic skill authoring
- **`wyrm_skill_create`** MCP tool — author + deploy + register a skill in one clean step. Validates **atomicity** (one focused capability, 12–280 char discoverable description, well-formed frontmatter), writes `<skillsDir>/<slug>/SKILL.md`, and registers it so it's instantly findable via `wyrm_skill_search`. Refuses to clobber an existing skill without `force`. `src/skill-author.ts` (`validateAtomicSkill`, `buildSkillMarkdown`, `deploySkill`, `removeSkill`). Skills dir overridable via `WYRM_SKILLS_DIR`.

### Tests
- `tests/priority-embed.test.ts` (13), `tests/skill-author.test.ts` (10) — idempotent prepend/update/remove/status, target gating, atomicity validation, YAML-safe frontmatter, clobber protection. Full suite: **797 passing** (40 suites). Live-verified: priority block landed at the top of `~/.claude/CLAUDE.md`, the buddy statusline renders `🐉 Wyrm · Wyrm · 3 ⟶`.

## [6.4.4] - 2026-06-04 - Live Memory security hardening + pentest fixes

A rigorous adversarial review (four parallel auditors) of the whole v6.4 surface. The replication **outbound client** was the main exposure — it was written assuming an honest peer. No new tools (still 131); behavior changes are all hardening.

### Security
- **[High] SSRF + bearer-token exfiltration on replication egress.** Outbound replication fetched a caller/registry-supplied `peerUrl` with the token attached, followed redirects, and trusted the response. New chokepoint `src/repl-guard.ts`: scheme allowlist (http/https only), blocks loopback / link-local / RFC-1918 / CGNAT / `169.254.169.254` metadata hosts (`WYRM_REPL_ALLOW_PRIVATE=1` opts in for same-host/LAN peers), `redirect: 'manual'` (no cross-origin token leak on a 3xx), an `AbortSignal` timeout (15 s, so a hung peer can't wedge the daemon), and a byte-bounded streaming body read (4 MB, so a hostile peer can't OOM us).
- **[High] `tokenEnv` secret-exfiltration.** A peer token env-var name must now be `WYRM_`-prefixed (override with `WYRM_REPL_ALLOW_ANY_TOKEN_ENV=1`), so a (possibly attacker-seeded) registry entry can't point `tokenEnv` at `OPENAI_API_KEY` / `AWS_SECRET_ACCESS_KEY` and have the daemon ship it to a peer. Enforced at `peer_add` **and** at tick time.
- **[High] Hostile-peer ingest validation.** `ingestRemoteEvent` now rejects unknown `kind`s, rejects non-integer/negative `origin_seq`, clamps a far-future `created_at` to ~now, bounds all string fields, and **forces `payload_json = NULL` on any non-shared row** — a peer can no longer smuggle a payload onto a private-flagged event or poison ordering/storage.
- **[Medium] Pull-watermark poisoning.** The watermark now advances only from cursors **actually received**, never from a peer-claimed `cursor` scalar — a forged `{cursor: 9e15, events: []}` can no longer make a node skip a peer's real events forever.
- **[Medium] SSE denial-of-service.** Concurrent SSE streams are capped (`WYRM_MAX_SSE_STREAMS`, default 64); the poll + keep-alive timers are now freed on **every** disconnect path (`req`/`res` `close` + `error`), not just `req.close`.

### Fixed (robustness)
- **`emitEvent` `origin_seq` race → silent event loss.** Sequence allocation is now atomic (`INSERT … SELECT COALESCE(MAX(origin_seq),0)+1`), closing the read-then-write race between the server, the `wyrm events` CLI, and the trace hook all writing to one DB (exactly the multi-process concurrency Phase 4 introduced).
- **Non-serializable payload** now degrades to reference-only (the event still emits) instead of dropping the whole event.
- **Daemon** closes the DB (WAL checkpoint) on SIGTERM/SIGINT; `start()` no longer leaks a log fd if `spawn` throws; `interval_minutes` is validated.
- **CLI** `wyrm events` / `wyrm watch` use exact project resolution (path → exact name), matching the HTTP surface — a fuzzy `LIKE` match could have attached the stream to the wrong project.

### Known limitations (documented, deferred)
- Origin identity is **not cryptographically authenticated** — within a trusted peer mesh, a token-holding peer can still forge `origin_device` (impersonation / dedup-key poisoning). Signed per-device identity is a future (Phase 5) design; the current trust model is "an operator-controlled mesh of nodes that already share a bearer token."
- DNS-rebinding (host resolves public at check time, private at connect time) is not fully closed — `fetch` doesn't expose the connect-time IP. The literal-host checks cover the realistic SSRF targets.

### Tests
- `tests/repl-guard.test.ts` — SSRF host matrix, scheme/`tokenEnv` rejection, redirect/timeout/size guards. Plus ingest-validation + watermark-poisoning-resistance cases in `tests/event-replication.test.ts` and the reference-only-degrade case in `tests/live-memory.test.ts`. Full suite: **774 passing** (38 suites). Hardened HTTP path re-smoked live (unknown-kind ingest rejected, zero rows written).

## [6.4.3] - 2026-06-04 - Live Memory (Phase 3b + 4): replication daemon, tool-trace hook, `wyrm watch`

Closes out the Live Memory rollout: cross-device sync goes hands-free, and the event stream gets human + agent tooling.

### Added
- **Always-on replication daemon** (Phase 3b) — `wyrm_replication` MCP tool (tool count **130 → 131**) with `action: start | stop | status | tick | peer_add | peer_remove | peer_list`. Detached child + PID file + liveness check + rotating log (mirrors the cloud-sync daemon). Syncs every registered peer on an interval (default 1 min, floored at 5s) with non-overlapping ticks. `src/replication-daemon.ts` (`ReplicationDaemon` + `ReplicationManager` + registry helpers) and `src/replication-daemon-entrypoint.ts`.
  - **Peer registry** persisted in `wyrm_meta`. **Secret hygiene:** a peer's bearer token is **never** stored in the DB — the registry holds `tokenEnv` (the *name* of an env var) and the daemon resolves the token from `process.env` at tick time.
- **`wyrm-tool-call-trace.mjs`** PostToolUse hook (Phase 4) — **opt-in** (`WYRM_TRACE_TOOL_CALLS=1`), emits a `tool_call` event per tool via `wyrm events publish`. Optional `WYRM_TRACE_TOOLS="Edit,Write,…"` allowlist; bounded (4s) + silent-fail + always exits 0. Registered in the `wyrm-setup` autoconfig table (installed but inert unless enabled).
- **`wyrm events publish|since`** and **`wyrm watch`** CLI subcommands (Phase 4) — publish an event, list events since a cursor, or live-tail a project's stream to stdout (Ctrl-C to stop; `--since N` to replay).

### Tests
- `tests/replication-daemon.test.ts` — 9 tests: registry upsert / remove / corruption-tolerance / **token-never-stored**; daemon `tick` replicates each peer, resolves `tokenEnv` at runtime, reports a missing project without throwing, and no-ops when Live Memory is off. Full suite: **755 passing**. CLI verified end-to-end against a throwaway DB.

## [6.4.2] - 2026-06-04 - Live Memory (Phase 3): cross-device replication

Cross-device sync for the event log — peer-to-peer over the Phase-2 HTTP surface, **no separate cloud relay**. One node pushes its shared events to a peer and pulls the peer's; dedup + echo-suppression make re-delivery and round-trips no-ops; privacy is gated so a node's private events never leave its box.

### Added
- **`POST /events`** (`wyrm-http`, bearer-gated) — replication **ingest**. `INSERT OR IGNORE` on `UNIQUE(origin_device, origin_seq)` dedup; echo-suppresses this device's own events; files incoming events under the resolved local project. Body `{project, events:[…]}` (≤500/req).
- **`?shared=1`** on `GET /events` **and** the SSE stream — restricts the read to `is_shared` events. This is the privacy gate for any **remote** puller: a peer never serves its private events. The same-device local UI/watcher omits the flag and still sees everything.
- **`src/event-replication.ts`** — `pushSharedEvents` / `pullPeerEvents` / `replicateOnce`. Depends on a structural `ReplDb` interface with an injected `fetch`, so it's fully unit-testable against a fake peer. Per-peer watermarks persist in `wyrm_meta`, so repeat calls are incremental.
- **`wyrm_events_replicate`** MCP tool (tool count **129 → 130**) — one-shot push+pull with a peer node: `{projectPath, peerUrl, token?, remoteProject?}`.
- DB primitives: `ingestRemoteEvent`, `eventsForPush`, `getMeta`/`setMeta`.

### Design / why
- **Privacy invariant on BOTH directions.** Push only ever collects `is_shared=1` rows; pull requests `?shared=1` so the peer can't serve its private events even if asked. Reference-only rows replicate with `payload_json = NULL` intact.
- **No conflicts.** The log is append-only; identity is `(origin_device, origin_seq)`. Echo-suppression is keyed on the per-DB device id — which is now cached **per-DB via a `WeakMap`** (previously a module global), fixing a latent bug where a process opening more than one DB would collapse their device ids.
- **Scope.** This ships the replication **core + a manual/one-shot trigger** (`wyrm_events_replicate`), proven end-to-end. The always-on `StreamingCloudSync` daemon — a continuous SSE subscription on an interval, with a peer registry — is the remaining Phase 3 follow-up, deferred so this lands tested and small.

### Tests
- `tests/event-replication.test.ts` — 9 tests: ingest dedup / echo-suppression / privacy-gated push; and a **two-node round-trip** (shared-only push, idempotency on re-run, pull + echo-suppress-on-return, peer-error-without-throw). Full suite: **746 passing**. A non-polluting HTTP smoke confirmed the `POST`/`GET` wiring + validation against a live server.

## [6.4.1] - 2026-06-04 - Live Memory (Phase 2): HTTP SSE event stream + JSON pull

The live transport for the 6.4.0 event log. Two Wyrm-aware processes on the same project now stay in lockstep over HTTP — one emits a `quest`/`session_update` event, the other sees it within ~1s. Read-path only; remote replication ingest is deferred to Phase 3.

### Added
- **`GET /events/stream?project=X&since=N`** (`wyrm-http`) — Server-Sent Events. Catches up the backlog since the caller's cursor, then tails new events (1s poll — better-sqlite3 has no change feed). Resumes on reconnect via the standard `Last-Event-ID` header; 25s keep-alive comments; emits a `retry: 3000` backoff hint and `X-Accel-Buffering: no` to defeat reverse-proxy buffering. Bearer-gated by the existing `wyrm-http` auth (localhost dev-bypass via `WYRM_DEV`).
- **`GET /events?project=X&since=N&limit=M`** — JSON pull fallback for poor networks / suspended clients (Termux). `?project=` accepts a path **or** a name; `limit` clamped 1–1000; returns `{project, cursor, count, events}` so the caller can page forward by cursor.
- **`src/events-sse.ts`** — dependency-free `sseFrame()` + `resolveStartCursor()` (precedence `Last-Event-ID` → `?since` → 0; rejects negative/NaN; strips newlines from `kind` so a malformed kind can't inject extra SSE fields). Split out so it's unit-testable without importing `http-fast.ts` (which opens the real DB at module load).

### Design / notes
- **Off-switch honored.** Both endpoints no-op when `WYRM_LIVE_MEMORY` is off — `{disabled:1}` JSON, or `503` for the stream — consistent with the MCP tools.
- **Failure-isolated tail.** A transient DB read error skips that poll tick and retries; the poll + keep-alive intervals are cleared on client disconnect (`req`/`res` close), no leaked timers.
- **Phase 2 is read-only.** `POST /events` (remote replication ingest) is intentionally NOT shipped here — it belongs with Phase 3 `StreamingCloudSync`, where the `(origin_device, origin_seq)` INSERT-OR-IGNORE dedup + echo-suppression live. No tool-count change (129).

### Tests
- `tests/events-sse.test.ts` — 12 tests (frame format, full-object serialization, cursor-as-id resume, newline-injection defense; cursor precedence/`0`-is-real/`string[]` header/garbage-fallthrough/negative-NaN rejection). Live smoke verified the SSE headers, resolve-by-name, and not-found paths end-to-end. Full suite: **737 passing**.

## [6.4.0] - 2026-06-04 - Live Memory (Phase 1): local append-only event log

The foundation for live, followable memory. Every canonical write now drops an event onto an append-only log that other sessions/agents can subscribe to and tail. Phase 1 is **local-node only** — no HTTP/SSE/cloud replication yet — but the cross-device identity columns are laid down now so federation is purely additive later.

### Added
- **`events` table + `wyrm_meta`** (migration v15). Append-only log: a local `cursor` (AUTOINCREMENT) gives total ordering for tailing; `(origin_device, origin_seq)` with a `UNIQUE` constraint gives a stable cross-device identity + INSERT-OR-IGNORE dedup for the future merge. Indexed on `(project_id, cursor)` and `kind`. `wyrm_meta` persists the lazily-minted, stable `device_id`.
- **3 MCP tools** (tool count **126 → 129**):
  - `wyrm_events_subscribe` — current head cursor + a recent chronological window for a project.
  - `wyrm_events_since` — pull events strictly after a cursor, in order; idempotent, safe to re-poll.
  - `wyrm_events_publish` — manually drop an event (e.g. a `tool_call` / `presence` marker) onto a project's stream.
- **Emission wired into canonical writes** — `createSession`, `updateSession`, `addQuest`, `updateQuest` each emit a `session_update` / `quest` event **after** the canonical row commits.
- **`WYRM_LIVE_MEMORY` feature flag — DEFAULT ON.** Disable with `0` / `false` / `off`. (`src/events.ts`)

### Design / why
- **Strictly additive & failure-isolated.** The emit runs *after* the canonical write and **log-and-drops** on any error (and is a no-op when the flag is off) — it never throws and never rolls back the real write. A broken event table or migration cannot regress the existing write path. Verified by a circular-payload test that confirms the bad event is dropped while the canonical write and subsequent events stay clean.
- **Privacy-aware.** Events are **reference-only by default** (`payload_json = NULL`) — they point at the canonical row rather than copying it. A denormalized snapshot is written **only** when `isShared` is set, so the log never becomes a shadow copy of data the encryption/federation layer protects.
- **Phase 1 scope.** Local only. The `cloud_cursor` column + `(origin_device, origin_seq)` identity exist now but are unused until the replication phase; no wire protocol is shipped in 6.4.0.

### Tests
- `tests/live-memory.test.ts` — 12 tests: migration/schema, emission on each canonical write, project scoping, cursor reads (ordering + idempotency), `origin_seq` monotonicity, privacy (reference-only vs shared payload), the default-ON flag (and `0`/`false`/`off` disabling), and failure isolation. Full suite: **725 passing**.

## [6.3.1] - 2026-06-02 - Security hardening

Pre-publish security review of the 6.3.x line surfaced and fixed a **Critical** data-exfiltration path in the autonomous agent loop plus several High-severity privacy/auth issues. **None were introduced by 6.3.0 — they predate it.** No API changes; behavior changes are all default-deny tightenings.

### Security
- **[Critical] Agent-loop external egress is now default-deny.** `wyrm_call_external` from inside the OODA loop previously dispatched unconditionally — and *before* the `SAFE_INTERNAL_TOOLS` whitelist gate — so a prompt-injected goal could exfiltrate local memory off-box. External calls now require an explicit operator allowlist via `WYRM_LOOP_ALLOW_EXTERNAL` (`"*"`, `"server.*"`, or exact `"server.tool"`, comma-separated). Observed context is also flagged as untrusted data in the decision prompt. (`agent-loop.ts`; regression test `tests/security-6.3.1.test.ts`)
- **[High] Cloud sync honors per-row privacy in `--all` mode.** `wyrm cloud sync --all` no longer overrides `cross_project_visibility = 'within'`; private rows never replicate to the cloud regardless of mode. Use the encrypted full-DB snapshot for a literal everything-backup. (`cloud/sync-engine.ts`)
- **[High] Cloud backup refuses plaintext upload.** The full-DB R2 snapshot now hard-fails unless `WYRM_ENCRYPTION_KEY` is set (override with `WYRM_ALLOW_PLAINTEXT_BACKUP=1`), instead of silently uploading the entire DB unencrypted. (`cloud-backup.ts`)
- **[High] HTTP UI-origin auth bypass hardened.** The `X-Wyrm-Origin: ui` localhost bypass now requires an exact-loopback socket (dropped the broad `127.*` match) **and** a loopback `Host` header, defeating DNS-rebinding. (`http-auth.ts`)

### Known / tracked (follow-up issues, not blockers)
- `wyrm_mcp_register` → `wyrm_call_external` is an arbitrary local command-exec primitive gated only by client trust (needs a command allowlist / first-run consent).
- License `features[]` should be derived from the verified tier rather than trusted from the payload; add hardware/device binding to curb license sharing.
- scrypt cost is below the OWASP floor; the audit hash-chain should be HMAC'd and include `actor`/`project_id`.
- `recordTombstone` is unused, so deletions don't propagate to the cloud (overlaps the sync-protocol hardening track).

## [6.3.0] - 2026-06-02 - Automatic context load (SessionStart rehydrate + opt-in prune)

The load-side counterpart to auto-capture (6.2.x, #14). Capture already remembers every session; now a fresh AI conversation automatically inherits the prior session's brief at startup, with zero manual tool calls. This is the single biggest token-saver Wyrm offers, made automatic.

### Added
- **`wyrm rehydrate`** prints a project's latest-session continuity brief (objectives, ground truths, open quests, validated patterns, unresolved failures) to stdout. Resolves the project by `--session <id>`, `--path <dir>` (exact), `--project <name>`, or the current directory. `--max-chars` bounds the output (default 6000); `--quiet` suppresses the "nothing to restore" note. Read-only, always exits 0.
- **SessionStart hook** (`scripts/hooks/wyrm-session-rehydrate.mjs`) shells to `wyrm rehydrate` and returns the brief as Claude Code `additionalContext`, so a fresh conversation picks up where you left off without a manual call. Set `WYRM_REHYDRATE_DRYRUN=1` to print the raw brief instead of the JSON envelope.
- **SessionEnd auto-prune hook** (`scripts/hooks/wyrm-session-prune.mjs`) is OPT-IN housekeeping: a no-op unless `WYRM_AUTO_PRUNE=1`. When enabled it trims stale, low-confidence, already-reviewed artifacts for the current directory's project at session end. Tunable via `WYRM_PRUNE_MIN_CONFIDENCE` (default 0.3) and `WYRM_PRUNE_OLDER_THAN` (default 90 days). Always exits 0.
- **`wyrm prune` flags:** `--yes` (skip the interactive CONFIRM, for non-interactive callers) and `--path <dir>` (exact-path project scoping, so a loose name match can never delete another project's artifacts).
- **`MemoryArtifacts.pruneStale()`** holds the prune candidate-selection plus delete logic, lifted out of the CLI and unit-tested for exact project scoping (`tests/prune-scoping.test.ts`).

### Changed
- Hook install (`autoconfig`) generalized to a `CLAUDE_HOOK_SPECS` table (script to events). Still idempotent, still merges into `~/.claude/settings.json` without clobbering hooks you added yourself.
- First-install pitch (`postinstall`) now leads with the automatic context-load story.

### Upgrading
- Default behavior is unchanged: rehydrate is read-only, and auto-prune does nothing unless you set `WYRM_AUTO_PRUNE=1`.
- **Run `wyrm-setup` once after upgrading to install the new SessionStart/SessionEnd hooks.** The MCP server does not install them on boot, and `npm install` does not run setup. Re-running setup is idempotent and preserves any hooks you added yourself.

## [6.1.0] — 2026-05-27 — Wyrm Cloud integration (stable)

Graduated from `6.1.0-beta.1` after end-to-end live testing:
- Real `wyrm cloud login` flow against production wyrm.ghosts.lk → ✓
- `status` / `devices` / `sync --dry-run` all return correctly → ✓
- Network-error handling translates to friendly retry messages → ✓
- 60-second fetch timeout via AbortSignal — defeats network-blackhole hangs
- CSRF state-cookie binding shipped server-side (wyrm-cloud 0.5.1)
- Top-level error handler in `cmdCloud` translates `CloudError`s to friendly output

No surface changes from beta.1; this is just graduation to `@latest`.

## [6.1.0-beta.1] — 2026-05-27 — Wyrm Cloud integration (beta)

The `wyrm cloud …` CLI surface lands. Multi-device sync is now a first-class Wyrm feature for operators who opt into the (free) Wyrm Cloud service at `wyrm.ghosts.lk`.

### Added
- **`wyrm cloud login`** — opens a browser flow against `https://wyrm.ghosts.lk/cli`. Sign in with Google or GitHub. Session token saved to `~/.wyrm/cloud.json` (0600). Master AES-256-GCM key auto-generated at `~/.wyrm/cloud.key` (0600).
- **`wyrm cloud logout`** — revokes the session server-side + locally.
- **`wyrm cloud status`** — account email, tier, storage usage + cap, device + delta counts, this-machine's device registration.
- **`wyrm cloud devices`** — list registered devices on this account (last-seen, current marker).
- **`wyrm cloud devices revoke <id>`** — revoke a specific device's sync access.
- **`wyrm cloud sync [--dry-run]`** — push every row with `cross_project_visibility != 'within'` (i.e. opt-in cross-project memory) + pull peer deltas from other devices on the account. AES-256-GCM encrypted client-side; the cloud server stores only ciphertext.

### Constitution alignment
- **Rule I (local-first):** cloud is fully opt-in; Wyrm runs unchanged without ever logging in
- **Rule IV (operator owns data):** master key never leaves the device; cloud cannot decrypt
- The Wyrm Cloud service runs on Cloudflare's free tier ($0/mo at expected scale)

### Sync semantics (v1)
- Tables synced: `ground_truths`, `memory_artifacts`, `quests`, `design_tokens`, `design_references`
- Last-write-wins on `(kind, row_id)` collisions
- Operator opt-in per-row via `cross_project_visibility = 'org'` or `'public'`
- Per-device cursor at `~/.wyrm/cloud-cursor.json`

### Architecture
New `src/cloud/` module:
- `client.ts` — Bearer-auth HTTP wrapper around the Wyrm Cloud API
- `crypto.ts` — AES-256-GCM with versioned envelope, operator-held key
- `cli.ts` — `wyrm cloud …` subcommand dispatcher
- `sync-engine.ts` — change-detection + push/pull + merge

### Out of scope (deferred)
- Team-tier org sync (Stripe billing required) — phase 4
- Web dashboard — phase 5
- Proper CRDT conflict resolution — v2 of sync engine (when team tier ships)
- BIP39 key recovery phrase — separate UX work

## [6.0.2] — 2026-05-26 — DEFCON bug-fix sweep

Found and fixed via comprehensive code-path audit, lint pass, regression-test sweep, and pre-mortem. Seven bugs caught before any user hit them.

### Fixed
- **🔴 P0 — unknown-tool fallback was unreachable.** The `default:` clause was missing from the giant `CallToolRequestSchema` switch in `index.ts`, so the "Unknown tool" return was orphaned after the last `case`. A call to an unknown tool name would fall through, `result` would be `undefined`, then the post-switch analytics code crashed accessing `result.isError`. Could DoS the server with a malformed tool call. Fixed by adding `default:` clause.
- **🟠 P1 — file descriptor leak on agent-loop spawn.** `agent-daemon.ts` opened a log fd in the parent process and passed it to the child via `stdio`, but never closed the parent's copy. One fd leaked per `wyrm_agent_init` call.
- **🟠 P1 — statusline daemon race condition.** Daemon's `server.on('error')` always called `cleanup()` (which deletes the socket + PID files) on any socket error — including `EADDRINUSE`. So a duplicate-spawn scenario would delete the LIVE daemon's files. Now `EADDRINUSE` exits silently without cleanup.
- **🟡 P2 — prototype pollution in digest periodKey lookup.** `DIGEST_PRESETS['__proto__']` resolved to `Object.prototype` (truthy → bypasses `??` fallback), then `.days` was `undefined`, producing broken SQL `datetime('now', '-undefined days')`. Switched to `Object.hasOwn()` for the lookup.
- **🟡 P2 — unhandled promise rejection in browser-launch path.** `import('child_process').then(({spawn}) => spawn(...))` had no `.catch()`. If `xdg-open` / `open` / `cmd` isn't on PATH, the spawn throws inside the `.then` and becomes an unhandled rejection that (Node 15+) terminates the process. Wrapped in try/catch + `.catch()`.
- **🟡 P2 — 4× `no-promise-executor-return` bugs.** `new Promise(resolve => setTimeout(resolve, n))` actually passes the Timer object as the resolved value (TS infers `Promise<Timer>`, not `Promise<void>`). Cleanest fix is explicit-block form. Affects `wyrm-loop.ts`, `wyrm-cli.ts` (3 sites), `agent-daemon.ts`, `resilience.ts`, `index.ts`.
- **🟡 P2 — no global crash handlers.** No `process.on('uncaughtException' | 'unhandledRejection')`. Any background async crash anywhere killed the entire MCP server silently. Added handlers that log to stderr (never stdout — that's the MCP transport).

### Added
- **`eslint.config.js`** (flat config). The 5.11.0 ESLint v10 bump shipped without migrating from `.eslintrc.*` — lint was non-functional since then. Restored full lint coverage with strict rules: `no-promise-executor-return`, `no-unreachable`, `no-fallthrough`, `no-self-compare`, `no-async-promise-executor`, `require-atomic-updates`, etc.
- **6 new regression tests** (`tests/bug-regression.test.ts`) — guards each fixed bug against re-introduction. 699 tests total (was 693).

### Pre-mortem (audited but not vulnerable)
- SQL injection via FTS5 / token values / reference fields — all parameterised, verified by 16-payload pentest suite.
- Path traversal via `wyrm_migrate_prompt` — `migrateOne()` returns `file-not-found` for `../../../etc/passwd` etc.
- Resource exhaustion — constellation query length capped at 200 chars, per-project limit capped at 50.
- `npm audit` clean — 0 vulnerabilities.

## [6.0.1] — 2026-05-26

Two bugs found while smoke-testing the live 6.0 install, both shipped together.

### Fixed
- **`wyrm-statusline` auto-spawn failed when installed globally.** The binary uses `process.argv[1]` to locate its sibling daemon. With a global npm install, `argv[1]` resolves to the symlink at `<prefix>/bin/wyrm-statusline`, not the real path in `dist/`, so the spawn target was looking for `wyrm-statusline-daemon.js` next to the symlink (where it doesn't exist) instead of in the package's `dist/` directory. Now follows the symlink with `realpathSync` before computing the daemon path.
- **`wyrm_intro` advertised a stale tool count** ("122-tool catalogue" — wrong since 5.9.0 already had 124). Replaced the hardcoded number with "full tool catalogue" so the string doesn't decay with each release.

## [6.0.0] — 2026-05-26 — "Present" (spec 018 complete)

The major version. Wyrm stops being invisible to humans and stops being too big for tool-cap-constrained AI clients. Five new pillars land together, with 55 new tests covering them and a full security-audit pass.

### Added

**Statusline (phase 1)**
- `wyrm-statusline` binary + `wyrm-statusline-daemon` — portable one-line status display for Claude Code / Cursor / Windsurf statusline hooks
- Daemon listens on `~/.wyrm/statusline.sock`, runs same `WyrmDB` business-logic layer the MCP server uses (correctness over speed per spec 018 decision #1)
- Auto-spawned on first call; idle-times-out after 5 minutes
- Per-call flags forwarded by binary so daemon's frozen env doesn't dictate every invocation (privacy + savings toggleable per call without daemon restart)

**Tool profiles (phase 2)**
- New `tool-profiles.ts` module — `essential` (≤30 tools), `standard` (~60 tools), `full` (126 tools)
- Auto-detect from `clientInfo.name` per spec 018 decision #4: VS Code / Antigravity / Cursor → `essential`; Windsurf → `standard`; Claude Code / Codex / unknown → `full`
- Operator override via `WYRM_PROFILE=essential|standard|full`
- No functionality removed — `full` profile still exposes every tool. Cap-constrained clients just see a curated subset.

**`wyrm_constellation` — cross-project memory (phase 3)**
- New tool that queries FTS5 across *all* registered Wyrm projects at once
- Returns candidate truths / artifacts / quests / references grouped by project
- Calling AI does semantic ranking on its side — no embedding dependency (per spec 018 decision #2: Wyrm runs inside AI clients that already do semantic ranking)
- Privacy: respects per-row `cross_project_visibility` flag (default `'within'` — no leak); explicit opt-in for `'org'` or `'public'` cross-project surfacing
- Input sanitisation strips SQL-injection metacharacters; query length capped at 200 chars; per-project limit capped at 50

**Cache markers (phase 4)**
- `wyrm_context_build` now segregates stable preamble (ground truths + scaffold) from volatile body (memory artifacts) with `<!-- cache-stable -->` markers
- Anthropic prompt-caching keys off prefix stability — same preamble across calls means 10× cost reduction on cached portions
- Logged as `cached_preamble` category in `token_savings_log` at 90% of preamble token count (conservative)

**`wyrm_migrate_prompt` (phase 5)**
- New tool that rewrites Wyrm-managed blocks in client config files (`.cursor/rules`, `.github/copilot-instructions.md`, `CLAUDE.md`, `.windsurfrules`)
- Diffs current block against canonical; only updates if changed (idempotent)
- Operator-authored content outside `<!-- wyrm:start --> ... <!-- wyrm:end -->` markers is preserved verbatim
- Dry-run by default; `apply: true` to write

**Migration 14**
- `token_savings_log` table (`recovered_context`, `blocked_retry`, `cached_preamble`, `truth_citation` categories with `conservative`/`realistic`/`optimistic` confidence)
- `cross_project_visibility` column on every searchable table (default `'within'` — no existing data leaks across projects)

**Token-savings telemetry — calibrated, honest**
- `wyrm_session_prime` logs `recovered_context` (chars / 4 tokens)
- `wyrm_failure_check` logs `blocked_retry` (1500 × matches, conservative)
- `wyrm_context_build` logs `cached_preamble` (preamble tokens × 0.9)
- Display gated behind `WYRM_STATUSLINE_SHOW_SAVINGS=1` until calibration validated; `wyrm_digest` shows openly with `~` prefix

### Quality + security
- **55 new tests** added (693 total). Covers tool-profiles, statusline, constellation, migrate-prompt, and a dedicated **security-audit suite** that probes SQL injection, path traversal, resource exhaustion, type validation, and the privacy gate.
- **Pentest pass**: 16 distinct adversarial payloads (`'; DROP TABLE`, UNION SELECT, null bytes, `../../etc/passwd`, huge inputs) verified to be sanitised, parameterised, or rejected.
- **`npm audit`** clean — 0 vulnerabilities.
- **638 → 693 tests, 100% pass rate.**

### Bugfix
- `wyrm_context_build` scaffold preservation when ground truths are empty (cache-markers refactor regression caught in code review, fixed before ship)

### Breaking
- Tool count display now reflects per-profile filtering — `wyrm_capabilities.toolCount` may show 24, 60, or 126 depending on active profile. This is the intentional behaviour of the cap-fix.
- Some clients (VS Code Copilot, Cursor, Antigravity) will see ~30 tools instead of 126 by default. Operators who want the full surface set `WYRM_PROFILE=full`.

### Total tools
**126** in `full` profile · **60** in `standard` · **27** in `essential`

## [6.0.0-alpha.1] — 2026-05-26 — Spec 018 Phase 1 preview

First slice of Wyrm 6.0 ("Present") — the statusline daemon, token-savings telemetry, and migration 14. Spec 018, phase 1 of 6. Not production-ready; npm tag `alpha`.

### Added
- **`wyrm-statusline` binary + `wyrm-statusline-daemon`** — a portable one-line status display intended for Claude Code / Cursor / Windsurf statusline hooks. The binary auto-spawns a detached daemon that listens on `~/.wyrm/statusline.sock`, runs the same `WyrmDB` business-logic layer the MCP server uses (so visibility filters / encryption / audit invariants are honoured), and idle-times-out after 5 minutes.
  - Default render: `🐉 Wyrm · ProjectName · 3 ⟶ · ⬢ 7`
  - With savings flag: `🐉 Wyrm · ProjectName · 3 ⟶ · ~12k saved · ✖ 2 · ⬢ 7`
  - With `WYRM_STATUSLINE_PRIVATE=1`: `🐉 ●●●` (screen-share safe)
  - With `WYRM_STATUSLINE_SHOW_SAVINGS=1`: experimental savings counter visible
- **Migration 14** — `token_savings_log` table + `cross_project_visibility` column on every searchable table (default `'within'` — no existing data leaks across projects).
- **Token-savings telemetry, default-on logging.** `wyrm_session_prime` logs recovered-context tokens; `wyrm_failure_check` logs blocked-retry tokens at 1500/blocked-attempt (conservative). Display gated behind `WYRM_STATUSLINE_SHOW_SAVINGS` until calibration is validated.

### Spec mapping
- Phase 1 (statusline + daemon + savings table) — ✅ this release
- Phase 2 (tool profiles) — pending
- Phase 3 (constellation) — pending
- Phase 4 (cache markers) — pending
- Phase 5 (migrate_prompt) — pending
- Phase 6 (release) — pending

### Try it
```bash
# After install
wyrm-statusline --cwd /path/to/your/project

# Or in Claude Code's settings.json:
{
  "statusLine": { "type": "command", "command": "wyrm-statusline", "padding": 0 }
}
```

### Why alpha
Pre-release until phases 2–6 land. Statusline + savings telemetry are stable but the broader 6.0 scope (tool profiles, cross-project memory, cache markers) ships incrementally.

## [5.11.0] — 2026-05-26

### Changed
- **npm description + keywords + README rewritten.** The old framing ("Persistent AI Memory System with encryption, full-text search, and infinite storage") was exactly the framing that made co-founder Azur think Wyrm is "just persistent memory." Replaced with a substrate-focused description that names all five things Wyrm actually does (memory, counter-pattern, agent loop, federation, creative substrate) and surfaces the keywords for the AI-tool ecosystem Wyrm now spans (Claude / Copilot / Cursor / Windsurf / Codex). README "What Wyrm actually is" section now leads with the five pillars instead of a generic memory pitch.

### Added
- **Bundled-skill auto-registration.** On first MCP server start after install, Wyrm reads the manifest `~/.wyrm/bundled-skills.json` written by postinstall and idempotently registers each bundled skill — `sachintha-creative-stack`, `ryan-founder-workflow`, `azur-codirector-workflow`, `buddy-protocol`, `professional-ascii-art`, `dragon-spec-author`, `wyrm-release-workflow`, `ghost-protocol-portfolio`, `wyrm-project-bootstrap`. Operators get the full skill catalogue on day one without manual `wyrm_skill_register` calls.
- **`wyrm-project-bootstrap` skill** — scaffolds a new Ghost Protocol project from zero to working baseline in one shot. Resolves the stack from operator preferences, creates the repo skeleton, applies design tokens, registers in Wyrm, inherits design references, seeds first ground truths.
- **Plain-language postinstall pitch.** Rewrote the first-install banner around what humans actually need to know: what Wyrm does in one sentence, three concrete things to paste into their AI right now (`What does Wyrm do?` / `What has Wyrm done for me?` / `Brief me on this project`). Updated icon set to match 5.10.0 PhantomDragon iconography.
- **Bundled-skill count surfaced in postinstall output.** Each install now reports how many skills shipped with the tarball.

### Changed
- **`better-sqlite3` 11 → 12.** Tested green against 638-test suite.
- **`jest` 29 → 30 + `@types/jest` 29 → 30.** All tests pass.
- **`eslint` 9 → 10.** Lint config compatible.
- **`ts-jest` stays on 29.x** — pinning because their 30.x line hasn't released yet; current 29.4.11 is forward-compatible with jest 30 per their matrix.

### Fixed
- **Moderate-severity `qs` transitive vuln** (GHSA-q8mj-m7cp-5q26) resolved via `npm audit fix`. `qs` updated from 6.15.1 → safe range via `@modelcontextprotocol/sdk` → `express` → `body-parser` chain.

### Why
The whole 5.x line has been adding capability faster than operators can discover it. 5.11.0 closes the discovery loop — bundled skills auto-register, the bootstrap skill turns "make me a new site" into a one-shot, and the postinstall pitch speaks to humans instead of listing features.

## [5.10.0] — 2026-05-26

### Added
- **Curated iconography (`src/icons.ts`)** — Wyrm's output glyphs are now a single coherent dragon-themed Unicode set instead of a grab-bag of generic emoji. The brand mark stays 🐉; everything else aligns to the PhantomDragon visual lineage:
  - `⌇` sessions (serpentine wave)
  - `⚙` tool calls
  - `⟶` quest open
  - `⟜` quest done
  - `✖` blocked
  - `⬢` ground truths (hexagon = constitution)
  - `✱` memory artifacts
  - `✧` design references
  - `↯` federation
  - `⚠` failures
  - `✺` celebrations
- Buddy and digest output migrated. Other surfaces will migrate as they're touched.

### Why
The accreted emoji set mixed pictograph styles (📜 🔧 ⚔️ ⛔ 🧱 🧠 📌 🎉 ✅) that didn't match the sharp, geometric, serpentine PhantomDragon aesthetic. The new set reads as one design system; new surfaces import from a single source.

## [5.9.1] — 2026-05-26

### Fixed
- **Buddy ASCII silhouette.** The full-size wyrm mascot was a corrupted alignment of the canonical Drake ASCII — the head, eye, and chest didn't compose visually, and the iconic swooping tail-curl was missing entirely. Restored canonical column alignment (head/eye/chest at col 9, back-hump at col 34) and added the missing 2-line tail curl. The wyrm now reads as a wyrm at every mood.

## [5.9.0] — 2026-05-26

The **creative + visibility** release. Two themes shipping together because they share a constituency — non-technical operators (designers, co-founders, friends) who use Wyrm without realising it.

### Added — Design tokens & references (creative workflow)
- **`design_tokens` table** (migration 13) — first-class storage for a project's design system primitives: colour, type, spacing, motion, shadow, radius, breakpoint, custom. AI helpers no longer re-derive the system from scattered CSS on every call. Tokens render into context briefs as a grouped markdown table.
- **`design_references` table** (migration 13) with FTS5 — a clip-and-tag library for inspiration URLs, palette references, snippets, and image paths. Searchable by full-text across title / notes / tags.
- **6 new MCP tools:** `wyrm_token_set`, `wyrm_token_get`, `wyrm_token_delete`, `wyrm_reference_add`, `wyrm_reference_list`, `wyrm_reference_search`.

### Added — Visibility & attribution (the discoverability problem)
- **`wyrm_intro` tool** — plain-English explanation of what Wyrm is, aimed at humans (co-founders, designers, anyone unsure what Wyrm does). Distinct from `wyrm_capabilities`, which is for AI-agent self-orientation.
- **`wyrm_digest` tool** — "what Wyrm did for you this period" report with counts and highlights (sessions tracked, quests completed, repeated failures blocked, ground truths set, references clipped). Periods: today / week / month / quarter / year / all.
- **Attribution guidance in `wyrm_inject_prompt`** — system-prompt block now instructs AIs to name Wyrm visibly when they use its data ("Wyrm reminded me that…"), instead of silently absorbing Wyrm's contribution as their own knowledge. Also instructs them to call `wyrm_intro` / `wyrm_digest` when users seem unaware of Wyrm.

### Added — Bundled skills
- **`skills/` directory ships in the npm tarball.** 8 skills bundled: `sachintha-creative-stack`, `ryan-founder-workflow`, `azur-codirector-workflow`, `buddy-protocol`, `professional-ascii-art`, `dragon-spec-author`, `wyrm-release-workflow`, `ghost-protocol-portfolio`. Operators register them with `wyrm_skill_register` against the bundled path.

### Why
- **Co-founder Azur thought Wyrm was "just persistent memory"** — that's the discoverability problem. Wyrm's value lives in the gaps between sessions, which are invisible by definition. `wyrm_intro` + `wyrm_digest` + attribution guidance make the work product legible to humans who didn't read the docs.
- **Sachintha is a creative who designs sites end-to-end** — the design tokens and reference clipper turn Wyrm into a usable creative substrate, not just a code-memory tool. The bundled skill keeps his stack first-class without requiring him to read documentation he won't read.

**Total tools: 124** (was 116 in 5.8.x). All advertised via ListTools (no cap, since 5.8.1).

## [5.8.1] — 2026-05-26

### Fixed
- **Uncapped `ListTools` response.** The handler was returning `[…].slice(0, 99)` — a legacy guard from when a few MCP clients silently truncated long tool lists. As of v5.8.0 there are **116 tools defined**, so the cap was hiding **17 tools** from every client. Removed the slice; all defined tools are now advertised. Empirically verified end-to-end via `wyrm_capabilities` reporting `toolCount: 116` to a connected client.

### Why
The cap pre-dates `wyrm_capabilities`, the agent loop, the buddy protocol, the outbound MCP surface, and most of the 5.x feature set. Client truncation behaviour has moved on; the cap had become silent feature loss instead of safety.

## [5.8.0] — 2026-05-26

Implements **[Buddy Protocol v1.0](https://github.com/Ghosts-Protocol-Pvt-Ltd/copilot-skills/blob/main/buddy-protocol/SKILL.md)** — Wyrm is now bidirectional. Other buddy-compatible MCP servers can call Wyrm; Wyrm can call them. Open convention; no central registry.

### Added
- **`buddy` MCP tool** — the well-known protocol entry point. Other MCP servers' buddies call this to get a brief, data-grounded status reply from Wyrm. Project-agnostic by default; pass `project_hint` to scope. Returns markdown (default) or structured JSON via `format:"json"`. Cycle-protected via `from_buddy` (won't reply to itself). **Total tools: 116.**
- **Buddy Protocol v1.0 skill** in `copilot-skills` — defines the convention (tool name patterns, request/response shape, federation rules, persona/mood/tone, anti-patterns). Any MCP server can adopt the protocol by exposing a `buddy` tool with the documented shape; no compliance test, no registry, the convention is the spec.
- **`PeerBuddyRequest` / `PeerBuddyJsonReply` types** in `buddy.ts` — exported for any TypeScript MCP server author who wants type-safe interop.
- **`BUDDY_PROTOCOL_VERSION` constant** — currently `"1.0"`. Future versions will be additive-only per §6.

### Changed
- **`wyrm_buddy` federation now prefers well-known `buddy`** over `*_buddy` / `buddy_*` patterns. Falls back to the legacy patterns when `buddy` isn't exposed by a peer.
- **Federation now sends Buddy Protocol v1.0 fields** — `from_buddy: "wyrm@5.8.0"` for cycle prevention, `project_hint: <project name>` for peer context, `size: "compact"` to cap peer reply size.
- **`wyrm_buddy` tool description updated** to mention `buddy` as the preferred peer-discovery name.

### Why
5.5.0 made Wyrm a buddy *querier* — it could call other MCPs' buddies. 5.8.0 makes Wyrm a buddy *responder* — other MCPs' buddies can call Wyrm. Two buddies on different MCP servers now federate symmetrically; no prior coordination required.

## [5.7.4] — 2026-05-26

### Added
- **`size` parameter on `wyrm_buddy`** — three mascot sizes for different surfaces:
  - `full` (default): the 10-line PhantomDragon-style head + shoulder banner from 5.7.3
  - `compact`: 5-line workspace-friendly variant — same tilde/spine visual language, smaller footprint
  - `mini`: single-line inline prefix on the greeting (`<--==(o)~~~~~~__   morning, operator…`)
  Mood still lives in the eye character across all three sizes.

### Confirmed
- **Buddy federation is live as of 5.5.0.** Any registered outbound MCP server that exposes a tool matching `*_buddy` or `buddy_*` is queried via `wyrm_call_external` and its reply folds into a "From other buddies" section (up to 3 external buddies per call). Disable per-call with `federate: false`. Quiet mode skips federation to stay token-frugal. Documentation tightened in the tool description.

## [5.7.3] — 2026-05-26

### Changed
- **Wyrm mascot now uses the PhantomDragon house style.** The 5.7.2 design was clean but generic. This is the actual Ghost Protocol visual language — flowing `~~~` tildes for scale-ridges, `==` for spine segments, recognizable kin to the [PhantomDragon AI banner](https://github.com/Ghosts-Protocol-Pvt-Ltd/phantom-dragon-ai). Mascot lineage stays consistent across the Dragon stack.

## [5.7.2] — 2026-05-26

### Changed
- **Wyrm mascot redesigned** per the new [`professional-ascii-art`](https://github.com/Ghosts-Protocol-Pvt-Ltd/copilot-skills/blob/main/professional-ascii-art/SKILL.md) skill. The 5.7.1 placeholder was crude; this is a proper side-profile coiled wyrm — clear horns, eye anchor, `=`-segmented neck (the new mascot signature), wing motif, body, legs. 12 lines × ≤50 cols. Mood lives only in the eye line (`<o>` default / `<^>` celebratory / `<->` stuck), so the figure reads as "same wyrm, different state" across moods. Body is invariant.

## [5.7.1] — 2026-05-26

### Added
- **ASCII wyrm in `wyrm_buddy` output** — the `wyrm` persona (default) now renders a small mood-aware ASCII dragon at the top of non-quiet responses. Three variants: `normal` (default eyes), `celebratory` (`^ ^` + sparkles + scales), `stuck` (`- -` + a thoughtful "hmm."). `quiet` mode stays the single-liner it always was. Other personas (drogo / berlin / sumair) skip the art and stay in their respective voices.

## [5.7.0] — 2026-05-26

Implements [dragon-platform spec 017](https://github.com/ghosts-lk/dragon-platform/tree/main/specs/017-wyrm-user-manual) — Wyrm User Manual MVP.

### Added
- **`docs/`** — VitePress static site, deployable to GitHub Pages at `https://ghosts-lk.github.io/Wyrm/`.
- **Custom theme** — teal `#00B89F` primary, red `#ff5a55` warning, h1 weight 900 / 2.5rem / -0.02em per constitution rule II. Dark-first by default.
- **`docs/.scripts/generate-tool-pages.mjs`** — auto-generates one Markdown page per MCP tool by parsing `packages/mcp-server/src/index.ts`. Hand-written prose preserved across regenerations via `<!-- @hand-written:start -->` markers.
- **`docs/.scripts/generate-skill-pages.mjs`** — emits the skills catalog index with deep links to GitHub source. (Per-skill pages deferred — see "Known limits" below.)
- **Get-started flow** — overview, install, first-scan, connect-ai, your-first-session.
- **Tool reference** — 115 auto-generated tool pages grouped by category (Projects, Sessions, Quests, Truths, Scaffolds, Failure Patterns, Knowledge Graph, Agent Loop, Outbound MCP, Federation, Audit, Skills, Data Lake, Memory Artifacts, Symbol Graph, Hours & Billing, Presence, Search & Capture, Orchestration, Meta).
- **Skills catalog** — index page listing all 94 registered skills grouped by category with deep links to GitHub source.
- **`.github/workflows/docs.yml`** — auto-builds and deploys to GitHub Pages on every push to `main` that touches `docs/` or the tool definitions.
- **Local-first search** via VitePress's built-in `local` provider — no Algolia, no third-party indexing.

### Known limits (deferred)
- **Per-skill pages.** Many SKILL.md bodies contain pseudo-tags (`<example>`, `<commentary>`, `<thinking>`) that Vue's template compiler rejects even when HTML-escaped or wrapped in `v-pre`. The MVP links each skill to its GitHub source instead. A follow-up will vendor a markdown-it preset that strips problem syntax before Vue sees it.
- **Concept / Integration / Reference pages** — stubs in sidebar; bodies not yet authored. Will land via per-concept PRs.
- **Inter font bundling** — falls back to `system-ui`. Bundled woff2 queued.
- **Hand-written prose for top-30 tools** — generator preserves the `<!-- @hand-written -->` slot but bodies not yet authored.
- **Versioned snapshots** — current build deploys to root only; per-minor archives queued.

### Sequencing
This release ships the docs site infrastructure. Content fills in over follow-up minors and patches.

## [5.6.0] — 2026-05-26

Implements [dragon-platform spec 016](https://github.com/ghosts-lk/dragon-platform/tree/main/specs/016-wyrm-web-ui) — Wyrm Web UI MVP.

### Added
- **`wyrm-ui` binary** — `wyrm-ui` opens the local Wyrm dashboard in your default browser. Probes `localhost:3333/health`, spawns `wyrm-http` detached if needed, then `xdg-open` / `open` / `start` to the URL.
- **Web UI bundle** at `packages/mcp-server/ui/` — single-file SPA, hash-routed, vanilla JS. 8 routes: Dashboard / Projects / Quests / Truths / Skills / Buddy / Agent / Search.
- **Constitution rule II palette** — teal `#00B89F` primary, red `#ff5a55` for breach/critical, severity pills (red/orange/amber/blue/grey), system-ui sans + JetBrains Mono for IDs.
- **New HTTP endpoints on `wyrm-http`:** `GET /skills`, `GET /agent`, `GET /buddy`, `GET /ui-token`. All bypass token auth when called from localhost with `X-Wyrm-Origin: ui` header.
- **`X-Wyrm-Origin: ui` auth bypass** — localhost requests with that header skip token auth; non-localhost still requires Bearer token. Spec 016 D2 — local UI is trusted because it ships in the same npm tarball.
- **Mobile responsive** — sidebar collapses to a horizontal scrollable bar under 768px (Termux operator at `localhost:3333/ui/`).
- **Keymap:** `/` focuses search, `g p` / `g q` / `g t` / `g s` / `g b` / `g a` navigate to routes.

### Deviations from spec 016 (deferred to 5.6.1+)
- **D1 SvelteKit** — MVP uses single-file vanilla HTML/CSS/JS instead. SvelteKit refactor queued. Decision: shipping a working UI fast beat the framework hygiene.
- **D3 CI design-check** — manual review only in MVP; CI job queued.
- **Inter font bundling** — MVP falls back to `system-ui`. Bundled Inter queued.
- **`/graph` route** — knowledge graph visualizer deferred (cytoscape adds ~200KB; not worth it for MVP).
- **`/analytics` route** — chart rendering deferred.
- **`/settings` route** — env display deferred.
- **Edit mode + mutating actions** — read-only-only in MVP.
- **SSE for `/agent`** — polling-on-route-load in MVP; SSE follow-up.
- **Export-this-view** — not yet wired.

### Wired
Every UI request goes through the same `db.getDatabase()` handle wyrm-mcp uses. No new state, no duplicate logic — UI is a window onto existing tables.

## [5.5.0] — 2026-05-26

Implements [dragon-platform spec 015](https://github.com/ghosts-lk/dragon-platform/tree/main/specs/015-wyrm-buddy). New `wyrm_buddy` MCP tool — friendly, data-grounded coding companion. Every claim cites a real DB row; no hallucinated encouragement.

### Added
- **`wyrm_buddy`** MCP tool — brief greeting + project state + suggested next step. **115 tools total.**
- **4 personas:** `wyrm` (default, dragon-themed), `drogo` (gruff), `berlin` (precise), `sumair` (warm), plus `custom` with sanitized `persona_name`.
- **Auto-mood detection:** `celebratory` when a quest closed in the last hour, `stuck` when ≥3 unresolved failures in last 7d, `normal` otherwise. `quiet` mode is explicit (single-line output).
- **Deterministic phrasing.** Phrase library is keyed by `(persona, mood, event)`. Selection uses `hash(projectId, dayOfYear)` so the buddy is stable within a day and varies across days. No randomness in voice.
- **Federation.** Auto-discovers `*_buddy` / `buddy_*` tools on registered outbound MCPs (via `wyrm_mcp_register` + `wyrm_call_external`). Folds up to 3 external buddy replies into Wyrm's response. Disable per-call with `federate: false`.
- **Data sources.** Streak detection (consecutive days with `wyrm_session_update` activity), last session recap, top-priority pending quest, recently-completed quest (last hour), unresolved failure patterns, matching reasoning scaffold, active goal + last iteration time, hours-this-week from the ledger.
- **Persona-name sanitization.** Custom names are restricted to `[a-zA-Z0-9 _-]{1,32}`.
- **`WYRM_BUDDY_DEFAULT_PERSONA` env** to pin a preferred persona.
- **Updated `wyrm_inject_prompt`** block — teaches AI clients to call `wyrm_buddy` at session start and on milestone events.

### Token frugal
Quiet mode is one line. Normal mode caps at ~800 tokens output. Plays nicely with spec 014's budgeted `wyrm_context_build`.

## [5.4.0] — 2026-05-26

Implements [dragon-platform spec 014](https://github.com/ghosts-lk/dragon-platform/tree/main/specs/014-wyrm-token-budgeting). Opt-in token-budgeted `wyrm_context_build` — items below the budget cutoff elide to one-line stubs the AI can deref via `wyrm_recall`. No information loss.

### Added
- **`max_tokens` parameter on `wyrm_context_build`.** When set, routes to the new budgeted path. When omitted, falls through to the existing legacy (unbounded) behavior — backward-compatible per spec 014 criterion 7.
- **`session_id` parameter** for session-scoped dedup. Items already shown to the same session render as one-line references on subsequent calls.
- **`strict_budget` parameter** for the operator who wants ground truths elided too when even the preamble overflows. Off by default (criterion D3).
- **`token-budget.ts`** module — pure functions: `makeEstimator()` (4-chars-per-token approximation, rounded up), `applyBudget()`, `resolveBudget()` with precedence (call > env > clientName-derived > fallback 4096).
- **`context-ranking.ts`** module — combined score in [0,1] from confidence × recency-decay × relevance × usefulness. Weights default to {confidence:0.4, recency:0.2, relevance:0.3, usefulness:0.1}, overridable via `WYRM_RANK_WEIGHTS` JSON env.
- **`session-seen.ts`** module — backed by new `session_seen_artifacts` table (migration 12). `mark()`, `markBulk()`, `getSeen()`, `prune(olderThanDays)`.
- **`wyrm_maintenance` extension** — now also prunes `session_seen_artifacts` rows older than `WYRM_SEEN_TTL_DAYS` (default 7d).
- **Stub-and-recall rendering.** Elided items appear in a separate "Elided to stubs" section with their (kind:id), reason (budget / seen), and approximate token cost.
- **Telemetry.** Every budgeted `wyrm_context_build` call records to `tool_call_log` with budget / items_total / items_inline / items_elided / tokens_inline / tokens_stubs in the args column.

### Kill switch
`WYRM_DISABLE_TOKEN_BUDGET=1` routes all calls to the legacy path regardless of `max_tokens`.

### Decisions (per spec 014)
- **D1** model-tier auto-detect from MCP `clientInfo.name` — phase 2; phase 1 ships with env override only. `budgetForClient()` lookup table exists in `token-budget.ts` for the next PR to wire up.
- **D2** 7-day TTL on `session_seen_artifacts`, tunable via `WYRM_SEEN_TTL_DAYS`. Pruned during `wyrm_maintenance`. ✓
- **D3** never elide ground truths unless `strict_budget:true` is explicit. Warn-and-overflow otherwise. ✓

### Not in this release (queued)
- `wyrm_token_savings` aggregation tool — telemetry is in `tool_call_log`; aggregation view ships in 5.4.1
- Auto-detect budget from `clientInfo.name` at MCP initialize-time — phase 2
- Exact `cl100k_base` tokenizer via tiktoken — phase 2 of spec 014

## [5.3.0] — 2026-05-26

The "self-awareness" release. Wyrm now (a) knows what version of itself you're running, (b) tells the AI exactly what it can do, and (c) greets you with a real pitch when you install.

### Added
- **`wyrm_capabilities` MCP tool** — returns Wyrm's full feature inventory + runtime state in either `json` or `markdown` format. 15 feature blocks (counter-patterns, ground truths, scaffolds, knowledge graph, OODA loop, outbound MCP, federation, audit chain, hour ledger, …) each with what / why / which-tools-to-call. The AI can read this once per session to ground itself, instead of re-discovering the surface mid-conversation. Lives in new `src/capabilities.ts`.
- **`wyrm_check_update` MCP tool** — quietly polls `registry.npmjs.org/wyrm-mcp/latest`, compares against the installed version, cached 24h in a new `update_check` table. Pass `force:true` to bypass cache. Lives in new `src/version-check.ts`.
- **`wyrm_self_update` MCP tool** — runs `npm install -g wyrm-mcp@latest` after explicit `confirm:true`. Returns the install transcript. Requires the operator to have write access to the global npm prefix.
- **Startup version banner** — `wyrm-mcp` emits a single stderr line if a newer version is on npm. Stderr (never stdout) so the MCP stdio protocol stays clean.
- **`scripts/postinstall.cjs` first-install pitch** — on first install, prints a multi-line summary of what Wyrm uniquely does beyond AI memory. On subsequent installs, prints a one-line "upgraded to X" instead. Tracked via `~/.wyrm/.first-install-shown`. Silence with `WYRM_SKIP_POSTINSTALL=1`.
- **Updated `wyrm_inject_prompt` block** — now teaches AI clients to call `wyrm_capabilities` at session start and surface update notices to the operator.

### Internal
- New `update_check` SQLite table (idempotent schema bootstrap on first read).
- `WYRM_TOOL_COUNT` constant in `index.ts` — single integer the capabilities report uses; bump when adding tools.

## [5.2.2] — 2026-05-26

### Added
- **Build-tool hint in `scripts/preinstall.cjs`.** When the host platform has no prebuilt `better-sqlite3` binary (anything not in linux/macOS/Windows × x64/arm64), the install will compile from source. The preinstall hook now checks for `cc`, `make`, `python3` on PATH up front and — if any are missing — prints the exact install command for the detected distro (apt / dnf / apk / pacman / zypper / brew). Suppress with `WYRM_SKIP_BUILD_TOOL_HINT=1`. The Termux/PRoot gypi patch from 5.2.1 still runs alongside this.
- **`TROUBLESHOOTING.md`** at the repo root. Covers the three real-world failure modes (Termux gypi, prebuild-install timeout, missing build tools) plus permission-denied and Bun/Deno/WebContainer limitations. Linked from README.

## [5.2.1] — 2026-05-26

### Added
- **Termux / proot-distro on Android compatibility.** New `scripts/preinstall.cjs` in `packages/mcp-server/` detects Termux/PRoot environments and patches the `-flto=4` flag that Node's bundled `common.gypi` ships but the Bionic-side toolchain rejects. Without this, `better-sqlite3` builds die with `cc: error: unsupported argument '4' to option '-flto='` when `prebuild-install` falls back to source build. The patch is idempotent, leaves a `.wyrm-preinstall.bak` for revert, and can be skipped with `WYRM_SKIP_TERMUX_FIX=1`. README has a new Installation section covering the proot-distro flow.
- Detection covers: `TERMUX_VERSION` env, `/data/data/com.termux` (host), `/system/build.prop` (Android), proot-distro guests on Termux.

### Changed
- Conservative devDeps refresh within current majors: `@typescript-eslint/*` 8.0 → 8.60, `ts-jest` 29.1 → 29.4.11, `@types/*` and `typescript` to current minor. Production deps (`@modelcontextprotocol/sdk`, `better-sqlite3`) untouched — major-version bumps for those land in a separate PR with explicit testing.


## [5.2.0] — 2026-05-15

### Added — Cloud sync daemon (Phase 2 multi-device)
- **`CloudSyncDaemon`** (`cloud-sync.ts`) — wraps `WyrmCloudBackup` with a periodic snapshot loop. On startup it compares the latest R2 snapshot vs the local DB mtime (with 1-min clock-skew tolerance) and restores if remote is newer. Every interval (default 10 min) it SHA-256s the local DB; if the hash differs from the last upload it pushes a new encrypted snapshot and prunes to keep N (default 20).
- **`CloudSyncManager`** — spawns the daemon as a detached child via `cloud-sync-entrypoint.js`, with PID file + rotating log at `~/.wyrm/wyrm-cloud-sync.{pid,log,state}`, single-instance guarantee, signal-0 liveness probe.
- **`wyrm_cloud_sync` MCP tool** with five actions: `start`, `stop`, `restart`, `status`, `force-sync` (one-shot tick without the long-running daemon).
- **`WyrmCloudBackup.listBackupsWithKeys()`** — convenience method returning `{ key, metadata }[]` so the sync daemon can call `restore()` without reconstructing the timestamp-format storage key.
- Tool surface **110 → 111**.
- Package version `5.1.0` → `5.2.0`.

### Tests
- 13 new tests in `tests/cloud-sync.test.ts` covering sha256, bootstrap variants (no-config / no-remote / restore / no-op), tick variants (uploaded / unchanged / no-db / error), custom keep_count, and state persistence. Uses an in-memory FakeCloud — no live R2 dependency.
- **Total: 25 suites, 638 tests** (was 24/625).

### Documentation
- `docs/CLOUD-SYNC.md` — full architecture + usage + what-shipped.
- `docs/CLOUD-SETUP.md` (added prior in this thread) — Phase 1 R2 backup setup walkthrough.
- `docs/SAAS-ARCHITECTURE.md` — Phase 3 multi-tenant hosting plan (design only, not implemented).

### Templates
- New `templates/invoices/` directory in Wyrm repo holding the Ghost Protocol 2026 invoice template + README mapping the `{{INVOICE_NO}} {{CLIENT_NAME}} …` substitution slots. Composable with `wyrm_invoice_generate` (markdown → branded PDF).

## [5.1.0] — 2026-05-13

### Added
- **`AgentDaemon` module** (`agent-daemon.ts`) — single-instance process manager for the `wyrm-loop` daemon. PID file + rotating log at `~/.wyrm/`.
- **4 new MCP tools** — `wyrm_agent_init`, `wyrm_agent_status`, `wyrm_agent_stop`, `wyrm_agent_restart`. AI clients can bootstrap the autonomous OODA loop without shell access.
- **`seed_goal` param on `wyrm_agent_init`** — set the first goal in the same call.

### Changed
- **`wyrm_inject_prompt` content updated** — now teaches AI clients about goals, agent bootstrap, counter-pattern check, failure recording, and citation conventions.
- Tool surface **106 → 110**.
- Package version `5.0.0` → `5.1.0`.

### Tests
- 6 new tests in `tests/v510-agent-daemon.test.ts`. Total: **24 suites, 625 tests** (was 23/619).

## [5.0.0] — 2026-05-13

### Added
- **Goals subsystem** — `goals` + `goal_iterations` tables; `Goals` class; 7 tools (`wyrm_goal_set/_list/_complete/_pause/_resume/_abandon/_iterations`).
- **Agent loop (OODA + ReAct)** — `agent-loop.ts` `AgentLoop` class. JSON-envelope LLM protocol works with both Ollama JSON mode and OpenAI native function calling. Whitelisted internal tool dispatch.
- **`wyrm_act(goal_id|query)`** — manually trigger one OODA run; supports ad-hoc goals.
- **Outbound MCP client** — `mcp-client.ts` `OutboundMcpClient` class. Lazy spawn over stdio, idle reap, bounded client pool. 5 tools (`wyrm_mcp_register/_list/_tools/_disable`, `wyrm_call_external`).
- **`wyrm-loop` daemon** — new `bin: wyrm-loop` in `package.json`. Long-lived scheduler that picks next active goal each tick.

### Schema
- **Migration v11** — `goals`, `goal_iterations`, `mcp_client_configs`, `external_call_log`, `agent_actions` tables.

### Changed
- Package version `4.0.0` → `5.0.0` (major — autonomous loop is a paradigm shift).
- Tool surface **93 → 106**.

### Tests
- 22 new tests in `tests/v500-features.test.ts`. Total: **23 suites, 619 tests** (was 22/597).

## [4.0.0] — 2026-05-13

### Added
- **Lossless session rehydration** (Tier 3.11) — `wyrm_session_rehydrate(session_id)` produces a full briefing markdown for a fresh AI agent to inherit prior session state. New `Rehydration` module.
- **Compliance audit trail** (Tier 3.9) — hash-chained `audit_log` table; 3 tools (`wyrm_audit_log`, `wyrm_audit_verify`, `wyrm_audit_export`). Tamper-evident, SOC2/HIPAA-grade.
- **Sub-agent embedding** (Tier 3.10) — `wyrm_ask(query)` assembles context from Wyrm's own data and runs through Ollama (auto-detect) or OpenAI. Degraded mode returns raw context if no LLM.
- **Federated team Wyrm** (Tier 2.8) — `is_shared` flag, `sync_conflicts` + `sync_log` tables, 4 tools (`wyrm_share`, `wyrm_unshare`, `wyrm_sync_conflicts`, `wyrm_sync_resolve`). Conflicts auto-mint quests — no silent overwrites.
- **LSP-facing HTTP endpoints**: `/syms`, `/failures`, `/truths`, `/audit`, `/sync/push`.

### Schema
- **Migration v10** — `audit_log`, `llm_query_log`, `sync_conflicts`, `sync_log` tables; `is_shared` column on 5 tables; supporting indexes.
- `runMigrations` now sorts by version before applying (defensive against out-of-array-order authoring).

### Changed
- Package version `3.9.0` → `4.0.0` (major — new paradigm, no breaking API).
- Tool surface: **84 → 93** MCP tools.

### Tests
- 22 new tests in `tests/v400-features.test.ts`. Total: **22 suites, 597 tests** (was 21/575).

## [3.9.0] — 2026-05-13

### Added
- **Counter-pattern detection** (Tier 1.1) — `failure_patterns` table + FTS5; 4 tools (`wyrm_failure_record`, `wyrm_failure_check`, `wyrm_failure_list`, `wyrm_failure_resolve`). Identical failures coalesce by `(signature, scope)` so the predictive push can BLOCK the same suggestion next time.
- **Multi-agent presence + work-stealing queue** (Tier 1.2) — `agent_presence` + `quest_claims` tables; 5 tools (`wyrm_presence_announce`, `wyrm_presence_list`, `wyrm_presence_release`, `wyrm_quest_claim`, `wyrm_quest_release`). TTL-based heartbeat, exclusive quest claims.
- **Causality chains** (Tier 1.4) — `decision_edges` table; 4 tools (`wyrm_decided_because`, `wyrm_decision_downstream`, `wyrm_decision_upstream`, `wyrm_decision_invalidate`). Cascade-invalidation when a foundational truth goes stale.
- **Cross-repo symbol graph** (Tier 2.6) — `symbol_index` table; 4 tools (`wyrm_symbol_index`, `wyrm_symbol_search`, `wyrm_symbol_callers`, `wyrm_symbol_stats`). Indexes TS/TSX/JS/JSX/Python/Rust/Go/PHP/Ruby. Skips `node_modules/`, `dist/`, etc.
- **Hour ledger + invoice generator** (Tier 2.7) — 2 tools (`wyrm_hours_report`, `wyrm_invoice_generate`). Derives hours from session content density (no new schema). Generates complete markdown invoice with per-session line items.
- **Self-improvement analytics** (Tier 3.12) — `tool_call_log` table + auto-logging from dispatch wrapper; 1 tool (`wyrm_tool_analytics`). Per-tool error rate, p50/p95 latency.

### Schema
- **Migration v9** — 5 new tables, 1 FTS5 virtual table, 4 triggers, 13 indexes. Backward-compatible.

### Changed
- Package version `3.8.0` → `3.9.0`.
- Tool surface: **64 → 84** MCP tools.

### Tests
- **40 new tests** in `tests/v390-features.test.ts`. Total: **21 suites, 575 tests** (was 20/535).

## [3.8.0] — 2026-05-13

### Added
- **Predictive Context Push, Phase 2** — `PreToolUse(Bash)` and `UserPromptSubmit` hook handlers join the existing file-edit hook. Bash commands get tokenized + boosted for destructive heads; prompts ≥6 chars become FTS queries. All three handlers share refactored fetch + render helpers and a single 200ms timeout budget.
- **`GET /d/agg`** — server-side aggregation endpoint in `http-fast.ts` for Fleet Intelligence payload-effectiveness records. Replaces client-side aggregation that didn't scale past a few thousand records.
- **`clampLimit()`** helper (`http-fast.ts`) — single source of truth for `?l=N` bounds-checking across data-lake endpoints.

### Fixed
- **DoS hardening on `/d` and `/d/agg`** — `?l=-1` (SQLite treated negative LIMIT as "no limit" → unbounded streaming) and `?l=abc` (NaN → silent empty) now clamp to safe defaults. Bounds: `/d` → `[1, 100]`, `/d/agg` → `[1, 10000]`.
- **Hook `emit()` exit guard** (`scripts/hooks/wyrm-push.py`) — explicit `sys.exit(0)` after JSON write enforces the silent-fail contract against future refactors.
- **UserPromptSubmit dispatch correctness** — `or` → `and` so a future event that grows a `prompt` field (schema drift) can't be misrouted to the prompt handler.

### Changed
- Package version bumped to `3.8.0`.

## [3.7.2] — 2026-04-24

### Changed
- Version bump to `3.7.2`. No functional changes since 3.7.1.

## [3.7.1] — 2026-04-24

### Fixed
- **Security: command injection pattern removed** — `wyrm serve --ui` browser-open now uses `spawn([url]).unref()` with `shell: false` instead of `exec()` template string; Windows uses `cmd /c start "" <url>` (shell built-in safe form)
- **Dead import removed** — unused `execSync` import removed from `autoconfig.ts`

### Changed
- Package version bumped to `3.7.1`

## [3.7.0] — 2026-04-23

### Added

**Real semantic search — Ollama auto-detection**
- New `OllamaAutoProvider`: automatically detects `nomic-embed-text` on `http://localhost:11434` at first embed call, re-probes every 60 s so a Wyrm instance started before Ollama still picks it up
- Default embedding provider changed from `'none'` → `'auto'` — hybrid search is now on by default when Ollama is running, gracefully disabled otherwise
- `'auto'` added to `ProviderConfig` provider type and `createProvider()` factory
- Sessions and quests now enqueued in `IndexingPipeline` on every write (previously only data-lake entries were indexed)

### Security (DEF CON pentest hardening)
- `database.ts` git operations already use `spawnSync` with `shell: false` — confirmed secure
- `sync.ts` path traversal protection: per-file `validatePath` / `validateProjectPath` on all read/write ops
- `http-auth.ts` Bearer token auth: SHA-256 hash, constant-time compare, rate limiting, CORS — applied to all non-public endpoints
- `crypto.ts` AES-256-GCM — verified key derivation strength

### Changed
- Package version bumped to `3.7.0`



### Added

**Visual Web Dashboard** (`wyrm serve --ui`)
- Self-contained dark-theme SPA served at `GET /ui` — no CDN, no external dependencies
- Tab 1 **Overview**: stats cards (projects, sessions, active quests, data points, memories, ground truths, review queue) + recent sessions list
- Tab 2 **Memories**: paginated artifact browser with live search (FTS5) and kind filter, confidence bar, tags, outcome badges
- Tab 3 **Quests**: kanban board with 4 columns (pending / in_progress / completed / abandoned), priority badges
- Tab 4 **Truths**: paginated ground-truths list with staleness indicator bar and ⚠️ badge when staleness > 70%
- Tab 5 **Review**: approve/reject queue for flagged artifacts (`needs_review = 1`)
- Hash-based tab routing (`#overview`, `#memories`, `#quests`, `#truths`, `#review`)
- All user data rendered via DOM `textContent` — XSS-safe

**New REST endpoints** (`GET/POST /ui/*`)
- `GET /ui` — serves the HTML dashboard (public endpoint, no auth required)
- `GET /ui/stats` — aggregate stats for Overview tab
- `GET /ui/memories` — paginated memory artifacts with `kind` and `search` query params
- `GET /ui/quests` — quests grouped by status for the kanban board
- `GET /ui/truths` — current ground truths with staleness score attached
- `GET /ui/review` — artifacts flagged for review
- `POST /ui/review/:id/approve` — clears `needs_review` flag
- `POST /ui/review/:id/reject` — deletes the artifact

**`--ui` flag for `wyrm serve`**
- `wyrm serve --ui` enables devMode auth (localhost-only, no Bearer token required)
- Opens `http://localhost:<port>/ui` in the default browser after server starts
- Prints dashboard URL to console

**`enableDevMode()` export** in `src/http-auth.ts`
- Programmatically enables devMode at runtime (safe — still verifies `remoteAddress` is localhost)

### Changed
- Package version bumped to `3.6.0`
- `wyrm serve` now resolves port from `WYRM_PORT` (falling back to `PORT` then `3333`) for consistency with `http-fast.ts`

## [3.6.2] — 2026-04-22

### Fixed
- Dashboard `sessEl` ReferenceError on sessions tab when no sessions exist
- Surface caught errors to console for easier debugging

## [3.6.1] — 2026-04-21

### Fixed
- Guard `http-fast` `server.listen()` so it only fires when the file is run directly (not when imported as a module)

## [3.5.0] — 2026-01-28

### Added

**Feature 1: Temporal TTL + Staleness on Ground Truths**
- `ground_truths` table gains `ttl_days` column (migration 7)
- `wyrm_truth_set` accepts optional `ttl_days` parameter
- `computeStaleness()` exported from `intelligence.ts` — returns 0.0–1.0 or null
- `wyrm_truth_get` displays staleness score and prefixes stale truths with `[⚠️ STALE]`
- `wyrm_session_prime` prefixes stale truths (staleness > 0.7) with `[⚠️ STALE]` in the context brief

**Feature 2: Contradiction Detection in wyrm_capture**
- Structural conflict check for truths: same `project_id` + `category` → routes to review queue
- Returns `{ status: 'queued_for_review', reason: 'conflict_check', artifact_id, conflicts_with }` on conflict
- Advisory-only FTS conflict check for heuristics: stores the artifact but appends `advisory_conflicts` to the response

**Feature 3: Memory Decay / Auto-forgetting**
- `memory_artifacts` table gains `last_accessed_at` and `access_count` columns (migration 8)
- `wyrm_recall` updates `last_accessed_at` and increments `access_count` after each retrieval
- `wyrm_recall` moved from `READ_ONLY_TOOLS` to `WRITE_TOOLS`
- New `wyrm_prune` tool: dry-run by default, never deletes `needs_review=1` artifacts, requires `confirm_ids` for live deletion
- New `wyrm prune` CLI subcommand with `--project`, `--min-confidence`, `--older-than`, `--no-dry-run` flags

**Feature 4: Cross-Device Encrypted Snapshot (Export/Restore)**
- New `wyrm_sync_export` tool: AES-256-GCM encrypted snapshot using `VACUUM INTO` for WAL-safe consistency
- New `wyrm_sync_import` tool: `preview` mode returns DB stats; `restore` mode backs up current DB then replaces it
- Binary format: `WYRM` magic (4B) + version (1B) + salt (32B) + IV (16B) + authTag (16B) + ciphertext
- Key derivation: PBKDF2 with 600,000 iterations, SHA-256
- Passphrase from `WYRM_SYNC_PASSPHRASE` env var (MCP) or interactive readline prompt (CLI)
- New CLI subcommands: `wyrm sync export --out <path>`, `wyrm sync import --from <path>`, `wyrm sync preview --from <path>`

### Changed
- Server version bumped to `3.5.0`
- CLI help text updated with new commands and environment variables

### DB Migrations
- **Migration 7**: `ALTER TABLE ground_truths ADD COLUMN ttl_days INTEGER CHECK(ttl_days IS NULL OR ttl_days > 0)`
- **Migration 8**: `ALTER TABLE memory_artifacts ADD COLUMN last_accessed_at TEXT DEFAULT (datetime('now')); ALTER TABLE memory_artifacts ADD COLUMN access_count INTEGER NOT NULL DEFAULT 0`

## [3.4.0] — Previous Release

- Unified `wyrm_capture` with auto-classification
- `wyrm_session_prime` for one-call context assembly
- Git history import (`wyrm_import_git`)
- Rules import (`wyrm_import_rules`)
