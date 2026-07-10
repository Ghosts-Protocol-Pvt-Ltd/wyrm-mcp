
# Negative learning

Most memory tools remember what worked. Wyrm also remembers what did not, and
uses it to block a repeat. This is the failure firewall, and it is the feature
that separates Wyrm from a vector database with good search.

## The firewall

Three tools, one loop:

- **`wyrm_failure_record`** stores a dead-end: the approach you tried and why it
  did not work.
- **`wyrm_failure_check`** is called before you propose a fix. If the approach
  matches a recorded failure, Wyrm surfaces it so you do not walk back into it.
- **`wyrm_failure_resolve`** closes a failure once it is genuinely fixed, so a
  once-dead path can be reconsidered when the world has changed.

The discipline is simple: check before you retry, record when something fails,
resolve when it is fixed. An agent that does this stops re-litigating solved
problems across sessions.

## Write a failure so the check catches it

A vague failure does not match later. A specific one does. Record three things:

1. **The approach**, concretely enough to recognize. "Bumped the timeout" is too
   thin; "raised the NIM rerank timeout to 1500ms" is matchable.
2. **Why it failed**, the actual cause, not the symptom. "Still slow" is a
   symptom; "LK round-trips are 0.5 to 1.2s so 1500ms is too tight and it times
   out under normal latency" is a cause.
3. **The fix or the constraint it revealed**, if you have one. That is what
   turns a dead-end into a lesson rather than a wall.

Written this way, a future `wyrm_failure_check` for "increase the timeout"
surfaces the real constraint instead of letting you rediscover it.

## Fleet negative learning

When several agents share one Wyrm (a swarm, a team, a run of parallel
workers), failures are attributed to the agent and run that produced them. That
gives you an accountable shared memory: one agent's dead-end warns the others,
and you can see which agent learned what. Recorded failures stay private to your
account by default; they are not shared outward unless you choose to.

## Why it compounds

A single agent that records failures saves its future self a repeat. A fleet
that records failures turns every dead-end into a warning for every other
worker, once. The value is not any single entry; it is that the same wrong turn
is not taken twice across an entire body of work. Check first, record honestly,
resolve when fixed, and the firewall pays for itself.
