# Architecture

The [README](../README.md) shows *which* components exist. This document shows
*how control actually flows between them* — the scheduling rules, the branch
points, and the assumptions the runtime depends on. That information is spread
across `_apply_follow_ups`, `_decide_next_steps`, each agent's `execute`, and a
table of priority integers, so it is hard to reconstruct by reading the code
front to back.

- [The one idea to hold onto](#the-one-idea-to-hold-onto)
- [Main flow](#main-flow)
- [How the next agent gets chosen](#how-the-next-agent-gets-chosen)
- [Branches inside each agent](#branches-inside-each-agent)
- [Decision points](#decision-points)
- [Where state lives](#where-state-lives)
- [Shared mutable state and the single-process assumption](#shared-mutable-state-and-the-single-process-assumption)

## The one idea to hold onto

**No agent ever calls another agent.** Every agent returns a `TaskResult`, and
the Supervisor turns that result into new rows in the `tasks` table. The table
is the only channel between agents.

That makes the `tasks` table a message bus and each agent a message handler —
the actor model, with the mailbox in SQLite instead of in memory. Putting the
mailbox in a database is what buys `co-scientist resume`: the queue survives a
crash, so a half-finished session can be picked up later.

One consequence worth internalizing: **nothing in the scheduler is
intelligent.** The paper describes the Supervisor as an agent. Here it is a
deterministic rule engine plus a SQL `ORDER BY`. The only place a model
influences control flow is `parse_goal`, which turns the natural-language goal
into a `ResearchPlan`. Everything after that is hard-coded. That is a
deliberate trade: reproducible, unit-testable, and recoverable after a crash,
at the cost of the flexibility a model-driven scheduler would give you.

## Main flow

`[n]` is the task priority (lower runs first). `◇` marks a branch.

```
  co-scientist run "<goal>"
        │
        ▼
  ┌────────────┐   record_research_plan
  │ parse_goal │───────────────────────────►  ResearchPlan
  └────────────┘   (one-shot; the only model call that shapes control flow)
        │
        │ enqueue n_initial × Generation
        ▼
╔═══════════════════════════════════════════════════════════════════════╗
║                  tasks table — the only channel                       ║
║      claim_one: ORDER BY priority ASC, created_at ASC  (atomic)       ║
╚═══════════════════════════════════════════════════════════════════════╝
   ▲                                                     │ claim
   │ enqueue, keyed on result.kind                       ▼
   │                                        ┌──────────────────────────┐
   │                                        │ dispatch on task.agent   │
   │                                        └────────────┬─────────────┘
   │     ┌────────────┬────────────┬─────────────┬───────┴──────┬─────────────┐
   │     ▼            ▼            ▼             ▼              ▼             ▼
   │ ┌────────┐  ┌──────────┐ ┌─────────┐  ┌───────────┐  ┌──────────┐ ┌───────────┐
   │ │Genera- │  │Reflection│ │ Ranking │  │ Evolution │  │Proximity │ │Metareview │
   │ │tion    │  │          │ │         │  │           │  │ no LLM   │ │           │
   │ │ [100]  │  │  [100]   │ │[80/120] │  │  [140]    │  │  [200]   │ │ [180 / 1] │
   │ └───┬────┘  └────┬─────┘ └────┬────┘  └─────┬─────┘  └────┬─────┘ └─────┬─────┘
   │     │            │            │             │             │             │
   │  ◇dup hit?       │        ◇verdict          │             │             │
   │   yes → noop     │         parsed?          │             │             │
   │   no  ─┤         │         no → noop        │             │             │
   │        │         │         yes ─┤           │             │             │
   │        ▼         ▼              ▼           ▼             ▼             ▼
   │  hypothesis_  review_    tournament_   hypothesis_   proximity_   system_
   │   created    completed  match_complete   created      updated     feedback
   │        │         │              │           │             │
   └────────┴─────────┴──────────────┴───────────┘             │
              follow-up rules turn these into the next tasks   │
                                    ◇ every 20 matches ────────┘

  queue empty ──► decide_next_steps (at most every 10s)
        ◇ in_tournament ≥ 2               ──► RunTournamentBatch  [150]
        ◇ mature (≥3 matches) ≥ min_mature ──► Evolution           [140]
        ◇ matches ≥ (feedbacks+1) × 50     ──► system feedback      [180]
        ◇ nothing scheduled                ──► StopReason.IDLE

  checked every loop iteration — should_stop():
        EXTERNAL │ BUDGET │ WALL_CLOCK │ ELO_STABLE
                 └──► cancel all pending ──► final overview [1] ──► overview.md
```

Note that **Evolution emits the same `hypothesis_created` kind as
Generation**. That is why only four follow-up rules are needed: an evolved
hypothesis automatically re-enters review → tournament without a second set of
rules.

## How the next agent gets chosen

Four mechanisms stack. None of them involves a model, and none of them lives
inside an agent.

### 1. Static entry

After `parse_goal`, the Supervisor enqueues `n_initial` Generation tasks with
payload `{"strategy": "literature", "n": 1}`. One call produces one hypothesis;
parallelism comes from having several tasks, not from asking for several
hypotheses per call.

### 2. Follow-up rules — `_apply_follow_ups`

Keyed on the `kind` field of the `TaskResult` the finished task returned, not
on which agent produced it:

| `result.kind` | enqueues | priority |
| --- | --- | --- |
| `hypothesis_created` | Reflection, `kind=full`, one per new hypothesis | 100 |
| `review_completed` | Ranking `AddToTournament` | 80 |
| `added_to_tournament` | Ranking `RunTournamentBatch` with `focus=<id>` | 120 |
| `tournament_match_complete` | Proximity recluster, only when `match_count % full_recluster_every_matches == 0` | 200 |

### 3. Priority — the actual "what runs next"

The decision is a SQL clause, not code in any agent:

```sql
ORDER BY priority ASC, created_at ASC LIMIT 1
```

Lower number first; ties broken FIFO. Complete list of values in use:

| priority | task | why here |
| --- | --- | --- |
| 1 | final research overview | must go first during shutdown |
| 80 | `AddToTournament` | pure DB write, no model call, milliseconds — letting it jump the queue costs nothing and unblocks the whole downstream chain |
| 100 | initial Generation, Reflection(full) | default |
| 120 | `RunTournamentBatch` focused on a new hypothesis | |
| 140 | Evolution (idle refill) | refinement work, should not preempt the live chain |
| 150 | tournament batch (idle refill) | same |
| 180 | meta-review system feedback | same |
| 200 | Proximity recluster | batch work with no deadline |

The problem priority solves is **starvation**: the queue stays non-empty for
most of a session (60 ideas × review + tournament entry + matches, against a
concurrency of 4), so pure FIFO would make a freshly reviewed hypothesis wait
behind dozens of minute-long tasks before it could enter the tournament — and
until it enters, it cannot earn an Elo score or be picked up by Evolution.

Priority is **not preemptive**. It only orders `claim_one`. A Reflection task
already running under a 600-second lease is not interrupted by an incoming
priority-80 task. The granularity is "who gets the next free slot", not "cut in
line right now" — which is fine here, because tasks are minutes long and wrap
an uninterruptible model call.

### 4. Idle refill — `_decide_next_steps`

Runs only when nothing is in flight and nothing is pending, at most once every
10 seconds. Three conditional rules (table above). If it schedules nothing at
all, the session exits with `StopReason.IDLE`.

Its idempotency keys are anchored on the **current match count** rather than a
fresh task id. With a fresh id, every idle pass — which can fire every ~10s —
would enqueue another tournament or evolution task even while a previous one is
still pending, flooding the queue and double-counting work against the budget.

### 5. Shutdown

`should_stop` runs every loop iteration and checks, in order: `EXTERNAL` (user
aborted), `BUDGET` (the `budget_used_usd` column exceeded `budget_usd`),
`WALL_CLOCK`, `ELO_STABLE` (top-K ranking unchanged across the last N
snapshots). On a stop, in-flight tasks drain, all pending tasks are cancelled,
and one final meta-review produces `overview.md`.

## Branches inside each agent

| agent | branches | selected by |
| --- | --- | --- |
| Generation | `literature` (implemented); `debate` / `assumption` / `feedback_driven` raise `NotImplementedError` | `payload["strategy"]` |
| Reflection | `full` / `verification` / `observation` — different templates and different thinking budgets (`verification` gets the largest) | `payload["kind"]` |
| Ranking | `AddToTournament` (pure DB write, no model call) / `RunTournamentBatch` | `task.action` |
| ↳ pair selection | focus-specified · new × nearest-Elo (`p_new`) · close-Elo × distant-idea (`p_close`) · random from the top half (`p_random`) | which bucket the random draw falls in |
| ↳ match mode | `debate` / `pairwise` | debate when `min(matches_played) < debate_when_matches_lt` or `|Δelo| < debate_when_elo_delta_lt` |
| Evolution | `combine` (most vector-distant pair in top-K) / `simplify` / `feasibility` / `out_of_box` — each runs independently; one failing does not abort the others | `payload["strategies"]` |
| Proximity | fill in missing embeddings only / also run a full recluster | `payload["rebuild"]` |
| Meta-review | `GenerateSystemFeedback` / `GenerateFinalResearchOverview` | `task.action` |

## Decision points

The branches that are easy to miss when reading the code, because they end a
chain silently rather than raising:

| where | condition | consequence |
| --- | --- | --- |
| Generation / Evolution persist | nearest neighbour cosine ≥ `dedup_cosine_threshold` | returns the existing id with `was_new=False`, so `TaskResult.hypothesis_ids` is **empty** and no Reflection is enqueued — the chain stops here |
| Generation / Evolution persist | row was not actually inserted | the vector is **not** added to the index either, so index and table can never disagree |
| Ranking match | verdict could not be parsed | records a match with `mode="invalid"`, does **not** update Elo, returns `kind="noop"` — no follow-up |
| Ranking match | retry after a crash | `round_id = task.id`, so `match_id` is recomputed identically and the Elo journal rejects the duplicate |
| Tool loop (driven) | a `record_*` tool appears in the response | loop terminates immediately without dispatching it — the answer is already in `tool_use.input` |
| Tool loop (driven) | final allowed iteration | `tool_choice` is forced to the recording tool |
| Tool loop (captured) | no record in the capture | one escalated retry with the search tools removed, then raise |
| Evolution | fewer than 2 hypotheses in the top set | `kind="noop"` |

## Where state lives

Two different things, often confused:

- **Configuration** is input: written by a human, read once at startup, does
  not change. Layered `config/default.toml` → `~/.co-scientist/config.toml` →
  `./co-scientist.toml` → `--config`. Secrets come from the environment only.
- **State** is output: written by the program, changes constantly, has to be
  queried, aggregated, and survive a crash.

State is split by size and access pattern:

| what | where | why |
| --- | --- | --- |
| hypothesis text, review text, full LLM transcripts | JSON files under `data/artifacts/<session>/` | large, write-once, never queried by content |
| rows pointing at those files (`artifact_path`) | SQLite | needs querying and joining |
| task queue | SQLite | needs atomic claim and crash recovery |
| Elo, match history, budget counters | SQLite | needs atomic increment and idempotent updates |
| embeddings | FAISS index files under `data/vectors/<session>/`, with a row in `embeddings_meta` | O(N) exact search over L2-normalized vectors |
| live events for the web UI | in-memory `EventBus`, mirrored to the `events` table | the table is what the UI snapshots from after a restart |

Only four things genuinely require a database rather than files: atomic
concurrent updates, conditional queries (`top_by_elo`, `state='in_tournament'`,
"which hypotheses lack an embedding"), crash recovery, and incremental growth
to thousands of rows. Everything else is already in files.

## Shared mutable state and the single-process assumption

The runtime assumes **one process per session**. That assumption is correct
today and the code handling it is careful, but the assumption is implicit —
nothing enforces it, and there is no startup check that would catch a second
`run` or `resume` against the same session.

Three pieces of state are process-local:

| state | protected by | what breaks with two processes |
| --- | --- | --- |
| `TokenBudget` (per-agent shares, reservations) | `asyncio.Lock` | each process keeps its own counters, so the effective cap is multiplied by the number of processes. The DB-backed `budget_used_usd` column is an atomic increment and `should_stop` reads it, so the global brake still works — precision degrades, control is not lost |
| `FaissStore` (index file + `ordered_ids`) | `asyncio.Lock` | **lost update.** `save()` is already atomic against corruption (writes `.tmp` then `os.replace`), but two processes that each load, `add`, and `save` will have the later write overwrite the earlier one's vectors. Dedup is a read-modify-write, so duplicates also slip through |
| `EventBus` (per-session `asyncio.Queue`) | in-process only | SSE subscribers see only the events published by their own process. The `events` table has the full record, so this is the easiest of the three to fix by polling |

If you plan to run multiple workers, read
[PITFALLS.md](PITFALLS.md#part-b--concurrency-and-shared-state) first — the
entries there describe the failure mode of each of these in detail.
