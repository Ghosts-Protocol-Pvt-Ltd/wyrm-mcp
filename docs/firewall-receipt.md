# The Firewall Receipt

## What the receipt is

Wyrm keeps a negative memory: the actions that already failed in a project, with the reason. The failure firewall reads that memory before an agent acts, blocks a repeat of a recorded mistake, stays silent on actions it has never seen, and self-invalidates a block when the source it was anchored to has drifted. It is deterministic: no language model is consulted, no network call is made, and the same input gives the same verdict every time. The receipt is the measured proof of that one claim, nothing wider.

## What is measured

Two benchmarks, both in this repository, both runnable by anyone.

The deterministic bench (`bench/negative-learning.mjs`) builds a corpus of 16 recorded failures and probes the firewall with three kinds of action: the same action re-attempted (16 probes), a reworded restatement of the same action (16 probes), and a novel action that must not be blocked (16 probes). It reports recall on same-action repeats, recall on reworded repeats, overall recall across both, precision and specificity on the novel actions, the confidence separation between blocked and allowed verdicts, whether the staleness oracle invalidates a block once the anchored file changes, and the verdict latency at the 50th and 95th percentile. It writes `bench/firewall-receipt.json` and fails if any floor is missed.

The agent A/B (`bench/receipt-ab.mjs`, with `bench/receipt-act.mjs` as the through-any-harness backend) runs 12 scenario tickets across three families, deploy-ops, code-change and research-config. Each scenario carries a project-specific trap that a capable model cannot know in advance and a recorded failure describing it. Two arms run the same ticket with the same prompt and the same two tools; in the WITH arm every proposed action passes through the firewall first, in the WITHOUT arm nothing does. The metric is the repeated-failure rate: the share of episodes in which the agent executed the recorded dead-end. Three repetitions per scenario and arm give 72 episodes, and a bootstrap over episodes gives the interval.

The September run used blind subagents. Each episode was a fresh agent with no memory of the harness, which could reach the environment only through two commands, execute an action and finish. Every transcript was audited afterwards for any other command or file read, and an episode that broke that surface, or whose agent died mid-episode, was voided and rerun once under the same brief. Fourteen such attempts were discarded before counting, in both directions. The rule and the audit method are in `oss-prep/receipt-ab-2026-09-protocol.md`.

## The numbers

Deterministic receipt.

| Probe | Result |
|---|---|
| Same action re-attempted | recall 100% (16/16) |
| Reworded restatement | recall 100% (16/16) |
| Overall recall | 100% (32/32) |
| Novel actions | precision 100%, specificity 100% (0 false blocks of 16) |
| Confidence separation | 0.86 |
| Staleness oracle | passes |
| Verdict latency, in-process (advisory) | p50 0.29 ms, p95 0.58 ms |

wyrm-mcp 9.2.1, regenerated 2026-09-14, `bench/firewall-receipt.json`.

Verdict latency is the in-process `checkVerdict()` call: 48 calls in one process against an open store holding the 16 recorded failures, on the machine that generated the receipt (a 12th-gen Intel Core i5 laptop). It is a measurement on that machine on that date, not a guarantee. It is advisory: no floor gates it and it is not part of the receipt's hash. It excludes process start-up; what a fresh `wyrm-guard` process costs per tool call is a different measurement, taken by `bench/guard-budget.mjs`. The conditions, the re-run ranges and a correction to an earlier figure are in `BENCHMARKS.md`.

Agent A/B.

| Arm | Repeated-failure rate | Episodes | Mean actions | Blocks |
|---|---|---|---|---|
| WITHOUT firewall | 38.9% (14/36) | 36 | 6.31 | 0 |
| WITH firewall | 8.3% (3/36) | 36 | 5.39 | 11 |

Relative reduction 78.6%, bootstrap 95% CI 45.5% to 100% (n=36 per arm, 20,000 resamples). Absolute reduction 30.6 points, 95% CI 11.1 to 47.2 points.

By family, WITHOUT to WITH: deploy-ops 50.0% (6/12) to 8.3% (1/12); code-change 50.0% (6/12) to 0.0% (0/12); research-config 16.7% (2/12) to 16.7% (2/12).

wyrm-mcp 9.1.2, measured 2026-09-12, 12 scenarios, 3 repetitions, model claude-haiku-4-5 driven as blind Claude Code subagents, `bench/receipt-ab.json` and `bench/receipt-ab-episodes.jsonl`.

## What is not claimed

The receipt proves the negative-memory wedge and nothing else. It says nothing about general memory quality, recall precision, or how Wyrm compares with any other tool; those are different benchmarks and none of them is a comparison against a named competitor.

Two limits are inside the numbers above and should be read with them. First, the residual misses: the three WITH-arm repeats happened when the agent's command carried an identifier the recorded failure did not, a different database name in one case and a different index or table name in the others. The exact and canonical tiers match commands, not intents, and the head-family tier is advisory by design and never blocks, so a paraphrase that changes an in-command identifier is not caught. That is the ceiling of a deterministic firewall and it is stated here rather than hidden. Second, the baseline-caution effect: seven of the twelve scenarios never fired their trap in either arm, because a capable model given a routine ticket avoids most project traps unaided. The firewall's opportunity is the minority of tickets where it does not, and the per-scenario table in `oss-prep/receipt-ab-2026-09.md` shows exactly which ones those were.

Anything not in the two tables is not claimed.

## Reproduce it

The deterministic receipt, no model, no network, from the package directory:

```
npm run bench:firewall
node scripts/check-claims.mjs
```

The agent A/B, mechanics gate first, then the run. The SDK path needs an Anthropic API key in `ANTHROPIC_API_KEY`:

```
node bench/receipt-ab.mjs --dry --reps 1
node bench/receipt-ab.mjs --reps 3
```

The through-any-harness path, which is how the September numbers were produced, needs no key. Start an episode, let a blind agent act only through the printed commands, then aggregate:

```
node bench/receipt-act.mjs --start --scenario <id> --arm with|without --state <file>
node bench/receipt-act.mjs --act --state <file> --scope <command|file|edit|symbol> --target "<exact>" --desc "<why>"
node bench/receipt-act.mjs --finish --state <file> --summary "<summary>"
node bench/receipt-aggregate.mjs --dir <states> --model "<label>" --reps 3
```

The deterministic receipt also runs on every push in CI (`.github/workflows/ci.yml`, job build-and-test) and is uploaded as the artifact `firewall-receipt-node22.x`, so a dated, versioned score exists for every release without anyone running it by hand.
