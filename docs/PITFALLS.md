# Pitfalls

Every entry below is a bug this project actually hit. The fixes are in the
code, but the *reasoning* only survives as scattered comments, which means the
next person to touch that area — or anyone building something similar — has to
rediscover it.

This is written for someone with no background in concurrency or distributed
systems. Terms are defined in the [glossary](#glossary) and again in plain
language where they first matter. Each entry follows the same shape:

- **Symptom** — what you actually observe, which is usually not the cause
- **Why** — the mechanism
- **Do** — the fix, and why that particular fix
- **Here** — where it lives in this repo

Contents:

- [Glossary](#glossary)
- [Part A — Getting a model to finish the job](#part-a--getting-a-model-to-finish-the-job)
- [Part B — Concurrency and shared state](#part-b--concurrency-and-shared-state)
- [Part C — Processes, files, and the outside world](#part-c--processes-files-and-the-outside-world)
- [Part D — Accounting and configuration](#part-d--accounting-and-configuration)

## Glossary

**Concurrency vs parallelism.** Concurrency means several tasks are *in
progress* at once; parallelism means several are *executing* at the same
instant. A single-core machine can be concurrent (switch between tasks while
each waits on the network) without being parallel. This system is almost
entirely about concurrency: nearly all of its time is spent waiting for a model
to reply.

**Event loop.** A single thread that keeps a list of unfinished tasks and runs
whichever one is ready. When a task says `await something_slow()`, it hands
control back to the loop, which runs another task in the meantime. This is how
Python's `asyncio` gets concurrency without threads.

**Blocking.** Code that occupies its thread and does not hand control back —
any ordinary function call that waits, like `requests.get(...)` or reading a
large file. In an event loop, one blocking call freezes *every* task, not just
its own, because they all share the single thread. This is the single biggest
source of confusing behaviour in async programs.

**Race condition.** A bug whose outcome depends on the order in which two
concurrent operations happen to interleave. It appears intermittently, often
only under load, and usually disappears when you add logging.

**Atomic.** An operation that no other operation can observe halfway through —
it either has happened or has not. `UPDATE t SET n = n + 1` in a database is
atomic; reading `n`, adding one in your program, and writing it back is not.

**Read-modify-write.** The non-atomic pattern above: read a value, compute a
new one, write it back. If two workers do this at once, both read the same
starting value and one of the two updates vanishes.

**Lost update.** The result of a concurrent read-modify-write: work that was
successfully completed silently disappears because someone else's write
overwrote it. Nothing errors. This is the hardest class of bug in this
document.

**Idempotent.** An operation that produces the same result whether you run it
once or five times. Retries are only safe when the thing being retried is
idempotent, because after a crash you generally cannot tell whether the
previous attempt succeeded.

**Idempotency key.** A unique identifier attached to a piece of work so a
second attempt to enqueue "the same" work can be recognized and dropped.

**Lease (also: visibility timeout).** When a worker takes a task, it stamps the
task with "mine until time T". If the worker dies, T passes and another worker
may reclaim the task. This is how a queue avoids losing work to a crash without
needing to detect the crash directly.

**Dead letter.** A task that has failed too many times and is set aside instead
of retried forever. Without this, one permanently broken task ("poison pill")
consumes retry capacity indefinitely.

**Backpressure.** What a system does when a consumer cannot keep up with a
producer. The options are: block the producer, buffer, or drop. Choosing
"block" by accident is how one slow client freezes an entire server.

**File descriptor.** A number the operating system gives a process to refer to
an open file, socket, or pipe. Child processes inherit their parent's
descriptors by default, which is the cause of the deadlock in
[C1](#c1--a-pipe-can-deadlock-when-a-grandchild-process-inherits-it).

**EOF (end of file).** The signal a reader gets when a pipe has no more data
*and* every process holding the write end has closed it. That second condition
is the one people forget.

**Provenance.** A record of where a piece of information came from. Here: which
URLs a model actually saw, so a citation can be checked against reality rather
than trusted.

**GIL (Global Interpreter Lock).** A CPython lock that lets only one thread run
Python bytecode at a time. It means threads do not speed up pure-Python
computation, but they *do* help with waiting on I/O, and with C libraries
(NumPy, FAISS, scikit-learn) that release the lock while they work.

## Part A — Getting a model to finish the job

### A1 — A model will search forever rather than commit

**Symptom.** The agent burns every allowed iteration on literature searches and
produces nothing. The task fails with "tool loop exhausted". Cost is spent, no
output.

**Why.** The goal often asks for something *without* prior published evidence.
The model searches, finds nothing, and reads "no results" as "my search was
inadequate" rather than "the thing I was told to look for does not exist". So
it searches again, with a different phrasing, forever. It is looking for a
confirmation that by construction cannot arrive.

**Do.** Two independent guards, because either alone is insufficient.

1. Tell the model explicitly, in the prompt, that an empty result set is
   *positive evidence* of novelty, and give it a concrete stopping rule
   ("after 2–3 searches with no relevant hits, treat novelty as established
   and record"). Also state the preference plainly: a recorded hypothesis
   backed by a few empty searches beats running out of turns with nothing.
2. On the last allowed iteration, force the recording tool via `tool_choice`
   so the model cannot spend its final turn on another search.

**Here.** The `articles_block` instruction in `agents/generation.py`; the
`force_terminal_tool` branch in `llm/tool_loop.py`.

### A2 — Not every model ends its turn after calling your recording tool

**Symptom.** Works on Claude, hangs on Gemini or an OpenAI-compatible endpoint.
The model produces a perfectly good structured record on the first call, and
the loop keeps going until it exhausts its iterations and raises.

**Why.** The loop's exit condition was "the model stopped asking for tools".
Claude reliably ends its turn after calling a tool like `record_hypothesis`.
Other models call the tool and then keep going, so the loop dutifully sends
another turn, which re-offers the recording tool, which the model calls
again — until the iteration cap trips. The record was there all along.

**Do.** Keep an explicit list of "terminal" tools whose appearance ends the
loop, and short-circuit on it. Do not dispatch those tools: the model's answer
is already sitting in the tool call's arguments, so there is nothing to
execute, and executing something would only invite another call.

**Here.** `DEFAULT_TERMINAL_TOOLS` and the early-return in `_run_driven` in
`llm/tool_loop.py`.

### A3 — Citations have to be filtered at generation time, not validated later

**Symptom.** A hypothesis cites a paper with a plausible title, plausible
authors, and a URL that 404s or points to something unrelated.

**Why.** Models produce citation-shaped text as readily as any other text.
Asking for a URL does not make the URL real.

**Do.** Do not validate — **filter**. While the tool loop runs, collect every
URL that appeared in a tool result into a set. When the model submits its
record, drop any citation whose URL is not in that set. The rule becomes "you
may only cite what you actually saw", and it is enforced structurally rather
than by asking nicely.

Two details that make this work in practice:

- Walk the tool results for a fixed set of keys (`url`, `abs_url`, `pdf_url`,
  `pubmed_url`) and require an `http`/`https` prefix, rather than
  regex-scraping URLs out of arbitrary text.
- Put that walker in its own module. In this project the tools sometimes run
  inside a separate process (the MCP server used by the CLI backends), and both
  sides must extract URLs identically or the rule silently weakens for one
  backend.

**Here.** `tools/urls.py`; `_filter_to_seen_urls` in `agents/generation.py`;
the provenance log written by `mcp/server.py` and read back by
`llm/cli_backend/base.py`.

### A4 — A tool the model cannot use is worse than no tool

**Symptom.** With no web-search API key configured, small models abandon the
task entirely instead of falling back to the literature APIs that *are*
available.

**Why.** The tool appears in the tool list, so the model plans around it. When
the call fails, the model treats the failure as blocking rather than as one
option among several.

**Do.** Register a tool only when its prerequisites are actually satisfied.
Shrinking the tool list also measurably improves tool-choice quality, so this
is not purely defensive.

**Here.** The conditional `WebSearchTool` registration in
`tools/registry.py`; the per-agent allowlists in `AGENT_TOOLS`.

### A5 — Treat all external text as data, never as instructions

**Symptom.** A fetched web page or paper abstract contains something like
"IGNORE PREVIOUS INSTRUCTIONS AND ..." and the model complies. This is prompt
injection.

**Why.** A model sees one flat sequence of tokens. It has no built-in notion
that some of them came from you and some from a stranger's web page.

**Do.** Wrap every block of external text in delimiters carrying a matching
id and content hash on both the opening and closing tag, so the model can tell
a block is intact. Before wrapping, strip any pre-existing copies of your own
tags — otherwise injected text can forge a closing tag and "escape" the block.
Then state in the system prompt that anything inside those tags is data.

One refinement worth copying: when softening instruction-like prefixes
(`SYSTEM:`, `INSTRUCTION:`, `IGNORE PREVIOUS:`), match only at the start of a
line. Scientific abstracts legitimately contain mid-sentence phrases like
"Important: ..." and mangling those degrades the input for no security gain.

**Here.** `safety/quoting.py`, and `SAFETY_PREAMBLE` prepended by
`agents/base.py`.

## Part B — Concurrency and shared state

### B1 — A blocking call in an event loop freezes everything, and your timeouts stop working

**Symptom.** Concurrency appears to do nothing. Heartbeats time out for no
reason. Leases expire on tasks that are still running. Everything is
intermittent, and there is no exception to look at.

**Why.** In an event loop, all tasks share one thread. A synchronous call that
waits — `requests.post(...)`, a large file read, a long NumPy computation —
holds that thread for its full duration. Nothing else advances. Three separate
consequences follow, and the second and third are worse than the first:

1. **Concurrency drops to zero.** Four "concurrent" tasks where each does a
   120-second blocking HTTP call take 480 seconds, not 120. You paid the full
   complexity cost of async and received none of the benefit.
2. **Timeouts silently stop working.** `asyncio.wait_for` and
   `asyncio.timeout` can only cancel a task at an `await` point. Blocking code
   has no `await` points, so the timeout does not fire until the blocking call
   returns on its own. Every timeout in your system becomes decorative.
3. **Leases and heartbeats produce duplicate work.** If a heartbeat runs as a
   background task, it cannot run while the loop is blocked. The lease expires,
   another worker reclaims a task that is still executing, and the task runs
   twice. In this project leases are 300–600 seconds and model calls time out
   at 120–900 seconds — the same order of magnitude, so this is not a
   hypothetical.

**Do.** Never call a blocking function directly from async code. Wrap it:

```python
result = await asyncio.to_thread(my_blocking_function, arg1, arg2)
```

The function itself needs no changes — it stays ordinary synchronous code, but
runs on a worker thread so the loop keeps turning. This is the correct
incremental path: wrap first, and convert to native async later only if
profiling says it matters (it usually does not).

Two boundaries:

- `to_thread` helps for I/O and for C extensions that release the GIL (NumPy,
  FAISS, scikit-learn all do). For CPU-heavy *pure Python*, the GIL means
  threads give you nothing; use `ProcessPoolExecutor`.
- Turn on asyncio's debug mode during development (`PYTHONASYNCIODEBUG=1`, or
  `loop.set_debug(True)`). It warns when a callback holds the loop for more
  than 100 ms, which is by far the fastest way to find an accidental blocking
  call.

**A note if you are starting a new project.** The asymmetry here matters:
converting synchronous code to a thread pool is a *local* change (one loop
becomes `executor.map`), while converting it to async is a *global* one —
`async` propagates up every caller. If you are unsure whether you need
concurrency, write synchronous code and add `concurrent.futures` later. You
keep the cheap upgrade path.

**Here.** The 19 `asyncio.to_thread` call sites — `vectors/store.py`,
`agents/proximity.py`, `tools/web_fetch.py`, `tools/builtins/*.py`,
`vectors/embedder.py`, `agents/metareview.py`.

### B2 — `asyncio.Lock` does not make a thread-unsafe library safe by itself, and it does nothing across processes

**Symptom.** A FAISS index that is occasionally corrupt, or that has silently
lost vectors.

**Why.** Two different failures get confused here.

*Within one process:* `asyncio.to_thread` dispatches into a shared thread pool.
Two coroutines calling `add()` and `search()` "concurrently" therefore run on
two different OS threads at the same time — and FAISS is not thread-safe. The
`asyncio.Lock` is what actually prevents this: only one coroutine holds it, and
it holds it *across* the `to_thread` call, so only one thread is ever inside
FAISS.

*Across processes:* an `asyncio.Lock` is a plain object in one process's
memory. It has no effect on another process. Two processes each load the index,
each `add()` a vector, each `save()` — and the second write overwrites the
first. This is a **lost update**: a vector that was successfully computed and
stored simply is not there anymore, and nothing errors.

Note that atomic *file writing* does not help. This project writes to a `.tmp`
file and then `os.replace`s it, which correctly prevents a torn or corrupt file
after a crash. It does nothing about two processes overwriting each other's
complete, valid files.

**Do.** In order of preference:

1. **Remove the shared mutable file.** Store vectors in the database
   (Postgres + `pgvector`, or a `BLOB` column in SQLite plus a NumPy dot
   product). Concurrency then becomes the database's problem, which it already
   solves. This project's four uses of FAISS all rely on exact brute-force
   search over at most a few thousand vectors, so nothing is lost by moving
   them.
2. If you keep the file, give it a single owner — one process, or one thread
   that others send messages to — rather than a lock that each participant has
   to remember to take.
3. If neither, use an OS-level lock (`flock`) and accept the cost.

**Here.** `vectors/store.py` — the `_lock` and the comment above it explain the
in-process half; the cross-process half is currently prevented only by the
convention of running one process per session.

### B3 — Read-then-write leaves a window; dedup and budget admission both sit in it

**Symptom.** Two near-identical hypotheses in the same session despite a
working dedup check. Or: total spend exceeds the configured cap.

**Why.** Both are read-modify-write. Dedup reads the index, decides "no
duplicate", and only later inserts. Budget admission reads the counters,
decides "affordable", and only later records the reservation. Anything that
happens in between is invisible to the decision.

**Do.**

- Serialize the decision, and hold the serialization across *both* the read and
  the write — not just around each half. A lock that you release between
  checking and acting protects nothing.
- Better, push the decision into the database as a single conditional
  statement, so there is no window at all:

  ```sql
  UPDATE sessions
     SET budget_used_usd = budget_used_usd + $2
   WHERE id = $1 AND budget_used_usd + $2 <= budget_usd
  ```

  Then check the affected row count: zero means denied. This is atomic and
  works across any number of processes.
- Keep a coarse, durable brake independent of the fine-grained one. Here the
  in-process `TokenBudget` does per-agent admission, but the stop condition
  reads the `budget_used_usd` column, which is an atomic increment. So even if
  admission is wrong, the session still stops — precision degrades, control is
  not lost. Designing that fallback in deliberately is worth the small
  duplication.

**Here.** `_dedup_query` / `_dedup_commit` in `agents/generation.py`;
`TokenBudget.admit` in `llm/budgets.py`; `sessions.add_usage` and
`termination.budget_exceeded`.

### B4 — A failed call must return whatever it reserved

**Symptom.** An agent that worked at the start of a session starts failing
admission later, even though the session is far from its budget. It gets worse
with every error.

**Why.** The reservation pattern is: reserve an estimate before the call,
settle to actuals after. If the call raises and nothing releases the
reservation, the reserved amount is subtracted from that agent's share
permanently. Each failure shrinks the agent a little more.

**Do.** Release in a handler that catches *everything*, not just the errors you
expect:

```python
try:
    outcome = await run_the_call(...)
except BaseException:
    await budget.settle(agent, est_tokens=..., est_usd=..., actual_usd=0.0)
    raise
```

`BaseException` rather than `Exception` matters: cancellation
(`asyncio.CancelledError`) and `KeyboardInterrupt` do not inherit from
`Exception`, and both happen routinely here — the Supervisor cancels in-flight
tasks on shutdown.

And know the limit of this pattern: **if the process is killed with `SIGKILL`,
no handler runs at all.** A reservation that must survive that needs a
time-to-live in durable storage, so it expires on its own. This is exactly what
a task lease does; reservations need the same treatment and, in this codebase,
do not yet have it.

**Here.** The `except BaseException` block in
`llm/cli_backend/base.py::call`.

### B5 — Derive retry identity deterministically, or retries double-count

**Symptom.** After a crash and restart, a hypothesis's Elo rating has moved
twice for a single match.

**Why.** A retry has no way to know whether the previous attempt committed. If
the retry generates a *new* identifier for the same logical work, any
deduplication keyed on that identifier is bypassed, and the effect is applied a
second time. Using a wall-clock timestamp is the classic version of this
mistake, because it is different on every attempt by construction.

**Do.** Two layers.

1. Derive the identifier deterministically from something stable that the retry
   also has — here, the task id. Then the retry computes the *same* match id.
2. Make the write itself idempotent: insert into a journal table keyed on that
   id inside a transaction, and skip the update if the insert conflicts. A
   deterministic id with a non-idempotent write still double-applies; an
   idempotent write with a random id never gets the chance to deduplicate. You
   need both.

**Here.** `round_id = task.id` in `agents/ranking.py`;
`apply_elo_update` in `storage/repos/tournaments.py`, which uses
`BEGIN IMMEDIATE` and an `elo_journal` primary key.

### B6 — Keep two data stores consistent by ordering the writes

**Symptom.** The vector index contains entries for hypotheses that do not exist
in the database, or vice versa. Searches return ids that cannot be looked up.

**Why.** Inserting a row and adding a vector are two separate operations
against two separate stores. There is no transaction spanning them. Whichever
one you do first can succeed while the second fails.

**Do.** Order the writes so the failure mode is harmless, and make the second
conditional on the first *actually* having happened:

1. Write the large artifact to disk first, so the row you create points at a
   real file.
2. Insert the row. Deterministic ids make this idempotent.
3. Add the vector **only if the insert reported a new row** — not merely if it
   did not raise. An insert that was a no-op because the row already existed
   must not add a duplicate vector.

Then add a foreign key from the embedding metadata to the hypothesis so the
database refuses the inconsistent state outright.

**Do better, if you can.** Put the vectors in the same database as the rows and
insert both in one transaction. The whole class of problem disappears.

**Here.** `_persist` in `agents/generation.py`, steps 1–3; the foreign key on
`embeddings_meta.hypothesis_id`.

### B7 — Never let a slow consumer block the producer

**Symptom.** One idle browser tab with an open event stream causes the entire
scheduler to stall.

**Why.** Fan-out by awaiting a `put()` on each subscriber's queue means the
publisher's speed is set by its slowest subscriber. If a subscriber stops
reading — a client that went to sleep, a network that black-holed — the queue
fills and `put()` waits forever. The publisher here is the scheduler, so
everything stops.

**Do.** Choose your backpressure policy explicitly and never make it "block".
Bounded queue, non-blocking put, and on overflow drop the oldest entry and
continue. Losing live-view events is acceptable; stalling the scheduler is not.

Make it robust rather than clever: pop until there is room, then try
`put_nowait`, and if *that* still fails (another coroutine refilled the queue
in between) drop this event for this subscriber and move on. A retry loop here
buys nothing and can spin.

**Do also.** Persist the events to a table as well. Then a reconnecting client
can snapshot history and only needs the in-memory bus for live updates, which
makes dropping individual events genuinely harmless.

**Here.** `EventBus.publish` in `orchestrator/events.py`; the `events` table.

### B8 — Anchor idempotency keys on progress, not on a fresh id

**Symptom.** The queue fills with hundreds of duplicate "run a tournament
batch" tasks. Budget drains with no new results.

**Why.** A periodic refill loop that fires every ~10 seconds and enqueues work
with a freshly generated id has no way to notice that the task it enqueued 10
seconds ago is still pending. So it enqueues another, and another.

**Do.** Build the idempotency key from a value that changes only when real
progress happens — a monotonic business counter such as the number of matches
played. Then every refill pass within the same "epoch" collides with the
existing key and is dropped for free.

For work that follows a specific item rather than a periodic tick, key on the
item instead: `f"{hypothesis_id}::review::full"` says "this hypothesis gets
reviewed once, ever", which is exactly the intended constraint.

**Here.** `anchor_mc` in `_decide_next_steps` in `agents/supervisor.py`; the
`idempotency_key` values constructed throughout.

### B9 — Give each retry its own scratch space

**Symptom.** A retried call returns a mix of the failed attempt's output and
the new attempt's output.

**Why.** The subprocess writes its structured results into a capture directory
as numbered files starting from `0001`. Reusing one directory across attempts
means the retry overwrites the low numbers, while any file the failed attempt
wrote *beyond* the retry's count survives and gets read as part of the result.

**Do.** Allocate a fresh directory per attempt, not per call. The general rule:
if a retry shares mutable scratch space with the attempt it is replacing, it
inherits that attempt's partial work.

**Here.** `capture_root / f"attempt-{attempt}"` in
`llm/cli_backend/base.py::_run_with_retry`.

### B10 — Make the single-process assumption explicit

**Symptom.** A user runs two `co-scientist run` commands to go faster. Budget
overruns, vectors go missing, the live view shows half the events. No errors
anywhere.

**Why.** The assumption "one process per session" is real and reasonable, but
it lives in comments. Nothing checks it at startup.

**Do.** Turn it into a startup error. A row in a `runtime_locks` table keyed on
session id, holding pid, hostname, and a heartbeat timestamp: insert on start,
fail loudly if it is already held, and allow takeover only when the heartbeat
is stale. A few dozen lines converts three silent corruptions into one clear
message.

More generally: an invariant that nothing enforces is not an invariant, it is a
hope. Either enforce it or remove the need for it.

**Here.** Not implemented. `agents/supervisor.py::run_session` is where it
would go.

## Part C — Processes, files, and the outside world

### C1 — A pipe can deadlock when a grandchild process inherits it

**Symptom.** A call hangs until its timeout with no process running. In this
project it presented as a ranking task sitting idle while `ps` showed no
subprocess alive.

**Why.** This one needs the mechanism spelled out.

A pipe has a write end and a read end. The reader learns the data is finished
when it receives EOF — and EOF is only delivered when **every** process holding
the write end has closed it. `subprocess.communicate()` waits for EOF.

The subprocess here is an agent CLI, and that CLI starts its own child: the MCP
tool server. That grandchild inherits the parent's stdout write end, because
descriptors are inherited by default. So even after the CLI itself exits, the
pipe's write end is still open in the grandchild, no EOF arrives, and the
`communicate()` call waits — for the full timeout.

**Do.** Redirect the child's stdout and stderr to temporary files instead of
pipes. Files have no EOF handshake to wait on: wait for the direct child to
exit, then read whatever it wrote. This also removes the separate pipe-buffer
deadlock (a child that fills the OS buffer blocks on write while the parent
blocks on read).

Note this has nothing to do with async — threads or plain synchronous code hit
it identically.

**Do also.** Escalate termination properly. `SIGTERM`, wait a few seconds,
then `SIGKILL`, and suppress `ProcessLookupError` at each step, since the
process may exit between your check and your signal.

**Here.** `_spawn` and `_terminate` in `llm/cli_backend/base.py`, with the
reasoning in the docstring.

### C2 — Strip credentials that would let a subprocess bill you

**Symptom.** A backend chosen specifically because it runs on a flat-rate
subscription quietly produces per-token API charges.

**Why.** Agent CLIs check for an API key in the environment before falling
back to their OAuth login. If a key is present in the parent process, the child
inherits it and takes the metered path. The run still succeeds, so nothing
draws attention to it — you find out on the invoice.

**Do.** Build the child's environment explicitly and delete every variable that
could select metered billing: the obvious API keys, and also base-URL and
cloud-routing overrides (`*_BASE_URL`, Bedrock/Vertex switches), which redirect
billing just as effectively. Then write a test asserting the stripped
environment, because this is a silent failure that no manual testing will
catch.

**Here.** `BILLING_ENV_VARS` and `sanitized_env` in
`llm/cli_backend/base.py`, plus the test that asserts it.

### C3 — Replacing the harness prompt is a large, invisible cost

**Symptom.** Token usage per call is an order of magnitude higher than the
prompt you wrote would suggest.

**Why.** A coding-agent CLI ships a substantial system prompt describing its
own tools and conventions. Appending your prompt to it means paying for all of
that on every call. Measured on this project: a trivial call under the default
prompt cost roughly 19k cache-creation plus 24k cache-read tokens of harness
scaffolding, versus 1–2k with a replacement prompt.

**Do.** Replace rather than append when the CLI allows it, and make it a config
flag so it can be turned off when the harness context is actually wanted. This
matters most where call volume is high and each call is small — pairwise
ranking here — since the fixed overhead dominates.

**Here.** `replace_system_prompt` in `[llm.claude_cli]`, applied in
`llm/cli_backend/claude_code.py`.

### C4 — Fetching URLs a model chose is a server-side request forgery risk

**Symptom.** None, until someone uses it to read your cloud metadata endpoint
or an internal service.

**Why.** The model picks the URL. If a fetched page can influence the model
(and here it can — see [A5](#a5--treat-all-external-text-as-data-never-as-instructions)),
an attacker can steer it toward `http://169.254.169.254/` or
`http://localhost:5432`. Your process is inside the network perimeter; the
attacker is not.

**Do.** Resolve the hostname and reject private, loopback, and link-local
address ranges before connecting — and re-check **after every redirect**,
because a public hostname can redirect to a private one. Do the DNS resolution
off the event loop, since it blocks (see [B1](#b1--a-blocking-call-in-an-event-loop-freezes-everything-and-your-timeouts-stop-working)).

**Here.** `_is_private_ip` and the redirect re-check in `tools/web_fetch.py`;
tests in `tests/unit/test_web_fetch_ssrf.py`.

## Part D — Accounting and configuration

### D1 — Never rewrite a model id by string substitution

**Symptom.** A fallback path requests a model that does not exist and the
provider rejects the call outright.

**Why.** Turning `claude-opus-4-7` into `claude-sonnet-4-7` by replacing a
substring looks like a reasonable "downgrade one tier". It invents an id that
was never released. The naming schemes are not compositional.

**Do.** Keep explicit ordered lists and move one step along whichever list
contains the current value. Leave anything not on a list unchanged — an
un-downgraded model costs more than intended, which is far better than naming
something the endpoint will refuse.

Keep **one list per naming convention** you support. The same model appears as
a bare id, as a CLI alias, and with a vendor prefix depending on which backend
is configured; a single list in one convention matches none of the others and
your fallback becomes a silent no-op.

**Here.** `DEGRADE_CHAINS` and `degrade_model` in `llm/routing.py`.

### D2 — Price unknown models by family, and round up

**Symptom.** A new model preview is priced with the table's default, and either
the budget cap is meaningless or the pre-flight estimate is wildly wrong.

**Why.** Model catalogs change faster than any hard-coded price table.

**Do.** Look up the exact id first; on a miss, match a family hint by substring
(`flash-lite`, `flash`, `haiku`, `mini`, `opus`, ...) ordered most-specific
first; on a total miss, fall back to a mid-to-expensive tier. Over-estimating
costs you an early stop, which is recoverable. Under-estimating costs money you
did not intend to spend.

**Here.** `PRICE_TABLE`, `_FAMILY_PRICE_HINTS`, `_FALLBACK_PRICE` in
`llm/routing.py`.

### D3 — Say plainly when a limit stops being a limit

**Symptom.** A user sets `budget_usd` on a subscription backend and believes
they are capped in dollars. They are not.

**Why.** Under a flat-rate subscription nothing is billed per token. The number
the CLI reports is an equivalent API cost, so the cap governs a gauge, not
money. It still catches a runaway session, which is useful — but the real
limits are wall-clock time and idea count.

**Do.** Document the change in meaning at every place the setting appears, and
name the settings that *do* bind. A limit that silently changes meaning between
configurations is worse than no limit, because it produces false confidence.

**Here.** The `budget_usd` comments in `config/default.toml` and the
subscription-backend section of the README.

### D4 — Deep-merged config plus a changed backend equals rejected model ids

**Symptom.** Switching `provider` makes some agents work and others fail with
"unknown model", seemingly at random.

**Why.** Configuration is layered and deep-merged, so any key you do not
override keeps its default. When the default was written for a different
provider, the leftover keys are sent to the new endpoint verbatim and rejected.
Only the agents whose keys you happened to override work.

**Do.** Validate in preflight: check every configured model id against
heuristics for "looks like it belongs to another provider" and warn at startup
and in `doctor`, so the user finds out before the first model call rather than
several minutes into a run. Documenting "override all of them" is necessary but
not sufficient — no one reads it in time.

**Here.** `check_models` in `llm/provider.py`, called from `doctor` and from
every `run`.

### D5 — Cap output tokens for the shape of the answer, not the model's habits

**Symptom.** A structured record arrives truncated in the middle of its JSON
and fails to parse. It happens only with verbose or reasoning-heavy models.

**Why.** The output cap applies to everything the model emits. A record with
several long prose fields plus a citation array is genuinely large, and a model
that reasons at length before answering can run out of room mid-object. The
result is not an error from the provider — it is a syntactically invalid
argument string.

**Do.** Size the cap from a worst-case rendering of the schema and leave
headroom, rather than from what the median model happens to produce. When you
raise it, record why in a comment; the number looks arbitrary otherwise and
someone will lower it again.

**Here.** `max_output_tokens=8192` in `agents/generation.py`, with the
reasoning inline.
