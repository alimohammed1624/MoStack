# Running an agent swarm with `codex exec`

This document describes how to build a multi-agent "swarm" using only the OpenAI Codex CLI, by scripting many headless `codex exec` runs. It covers the building blocks, the orchestration patterns, shared state, verification, driver implementations, and the operational problems you will hit.

Written against `codex-cli 0.160.1` on 2026-10-07. Items marked **(verified)** were checked against `codex exec --help` or a real run on that version. Everything else is design guidance; check it against your installed version.

---

## 1. Why script `codex exec`

Codex has a built-in `multi_agent` feature (`spawn_agent`, `wait_agent`, `close_agent`), but it is hierarchical, capped per session by `[agents] max_concurrent_threads_per_session`, and the children cannot message one another. It suits a team of roughly 8–16 workers inside one session.

A swarm in the sense of OpenAI's Navier–Stokes run (thousands of agents, groups attacking different formulations, results merged between rounds, a formal verifier at the end) needs an external driver. With `codex exec` you get:

- any number of independent agent processes, limited by rate limits and budget rather than by Codex;
- full control of what each agent sees, where it writes, and how its result is judged;
- durable, inspectable state in files and git rather than in model context.

What you give up: agents do not talk to each other natively, and nothing coordinates them unless your script does. The patterns below supply that coordination.

---

## 2. Building blocks

### 2.1 `codex exec` options that matter for swarms (verified)

| Option | Use in a swarm |
| --- | --- |
| `[PROMPT]` or `-` | The worker brief. With `-`, the brief is read from stdin. If stdin is piped and a prompt is also given, stdin is appended as a `<stdin>` block. |
| `-C, --cd <DIR>` | Working root for the worker. Point each worker at its own git worktree. |
| `--add-dir <DIR>` | Extra writable directories, e.g. a shared `inbox/` or `scratch/`. |
| `--worktree` | Run in a new Codex-managed git worktree. Convenient, but you don't choose the path or branch name; manual worktrees (section 4.2) give more control. |
| `--skip-git-repo-check` | Allow running outside a git repo. |
| `-s, --sandbox` | `read-only`, `workspace-write`, or `danger-full-access`. |
| `--approve-for-me` | Route approval requests through automatic review with the `workspace-write` sandbox. |
| `--dangerously-bypass-approvals-and-sandbox` | No prompts, no sandbox. Use only inside an external sandbox (container or VM). |
| `-m, --model` | Per-worker model. Use cheaper models for wide exploration, stronger ones for merging. |
| `-c key=value` | Override any `config.toml` value per worker (e.g. `-c model_reasoning_effort="high"`). |
| `-p, --profile <NAME>` | Layer `$CODEX_HOME/<name>.config.toml` on top of the base config. One profile per role is a clean way to vary model, effort and instructions. |
| `--enable` / `--disable <FEATURE>` | Toggle features per run, e.g. `--disable multi_agent` for flat workers or `--enable multi_agent` for sub-teams. |
| `--ignore-user-config` | Ignore `~/.codex/config.toml` so workers get a reproducible config (auth still comes from `CODEX_HOME`). |
| `--json` | Emit JSONL events on stdout. |
| `--output-schema <FILE>` | Constrain the final message to a JSON Schema. |
| `-o, --output-last-message <FILE>` | Write the final message to a file. Combined with `--output-schema`, this file is the worker's machine-readable result. |
| `--ephemeral` | Don't persist the session to disk. Saves space, but the run cannot be resumed or forked. |
| `codex exec resume <ID> [PROMPT]` | Continue a previous session, keeping its context. |
| `codex exec fork <ID>` | Branch a new session from a previous one. |

### 2.2 Gotcha: close stdin (verified)

`codex exec` reads extra input from stdin when stdin is not a terminal. In a script, a background job, or a pipeline, it prints `Reading additional input from stdin...` and **hangs** until stdin closes. Always redirect:

```bash
codex exec ... "brief" < /dev/null
```

or pass the brief itself on stdin with `-`:

```bash
codex exec ... - < brief.md
```

### 2.3 Gotcha: your shell function does not apply

If `codex` is a shell function or alias that adds flags (for example `--dangerously-bypass-approvals-and-sandbox`), scripts, `xargs`, GNU `parallel` and Python's `subprocess` call the real binary without those flags. Pass every sandbox and approval flag explicitly in the driver. In non-interactive runs, actions that need approval fail instead of prompting.

### 2.4 JSONL event stream (verified)

With `--json`, a run emits one JSON object per line. Observed event types on 0.160.1:

```json
{"type":"thread.started","thread_id":"01a116d8-..."}
{"type":"turn.started"}
{"type":"item.completed","item":{...}}
{"type":"turn.completed","usage":{"input_tokens":17881,"cached_input_tokens":0,"cache_write_input_tokens":0,"output_tokens":19,"reasoning_output_tokens":0}}
```

What to extract:

- `thread.started.thread_id`: store it so you can `codex exec resume` or `fork` that worker later. Not useful with `--ephemeral`.
- `turn.completed.usage`: token accounting for budget control (section 8.3).
- `item.completed`: tool calls, commands, file edits and messages, for logs and debugging.

Longer runs may emit other event types (errors, command executions). Treat unknown types as pass-through and log them rather than failing.

### 2.5 Structured results with `--output-schema` (verified)

Give every worker a schema so the driver never has to parse prose. Tested schema:

```json
{
  "type": "object",
  "properties": {
    "status": { "type": "string", "enum": ["ok", "fail"] },
    "note":   { "type": "string" }
  },
  "required": ["status", "note"],
  "additionalProperties": false
}
```

With `-o last.json`, the file contained exactly `{"status":"ok","note":"hello"}`. Keep schemas strict: every property listed in `required`, `additionalProperties: false`. A useful worker result schema:

```json
{
  "type": "object",
  "properties": {
    "status":      { "type": "string", "enum": ["claim", "partial", "dead_end", "blocked"] },
    "summary":     { "type": "string" },
    "claim":       { "type": "string" },
    "artifacts":   { "type": "array", "items": { "type": "string" } },
    "verify_cmd":  { "type": "string" },
    "messages":    {
      "type": "array",
      "items": {
        "type": "object",
        "properties": { "to": { "type": "string" }, "body": { "type": "string" } },
        "required": ["to", "body"],
        "additionalProperties": false
      }
    },
    "next_steps":  { "type": "array", "items": { "type": "string" } }
  },
  "required": ["status", "summary", "claim", "artifacts", "verify_cmd", "messages", "next_steps"],
  "additionalProperties": false
}
```

`messages` is how a worker talks to other agents (section 3.4). `verify_cmd` lets the worker suggest how its claim can be checked, but the driver decides what actually runs.

### 2.6 One worker, end to end

```bash
codex exec \
  -C "worktrees/r03-prove-07" \
  --add-dir "$PWD/inbox" \
  -s workspace-write \
  -m gpt-5-codex \
  --json \
  --output-schema schemas/worker.json \
  -o "runs/r03/prove-07/result.json" \
  - < "runs/r03/prove-07/brief.md" \
  > "runs/r03/prove-07/events.jsonl" \
  2> "runs/r03/prove-07/stderr.log"
```

Replace the model name with one from your account. Exit code, `result.json`, and `events.jsonl` are the three things the driver reads.

---

## 3. Orchestration patterns

These patterns combine. A serious run is usually pattern 3.3 with pieces of the others inside it.

### 3.1 Best-of-N fan-out

Run N independent workers on the same brief, with variation (model, temperature-like effort settings, a different hint or seed per worker). Keep whichever results pass the verifier, or rank them with a judge run.

- **Good for:** problems with a cheap, reliable checker (tests, a Lean build, a numerical tolerance).
- **Shape:** one round, no communication.
- **Cost:** linear in N; most attempts are wasted, so it only pays off with a strong verifier.

```
brief ──► worker 1 ─┐
      ──► worker 2 ─┼──► verify each ──► keep passes / judge ranks
      ──► worker N ─┘
```

### 3.2 Map-reduce (decompose, solve, merge)

A planner run splits the problem into independent sub-problems (lemmas, modules, parameter ranges). Workers solve shards in parallel; a reducer run combines them.

- **Planner:** `codex exec` with a schema that returns a list of shards, each with its own brief and acceptance check.
- **Workers:** one per shard, each in its own worktree.
- **Reducer:** reads all passing shard results and produces the combined artifact, then the verifier runs on the whole.
- **Weakness:** depends on the planner's decomposition being right; add a re-plan step when too many shards fail.

### 3.3 Rounds with groups, a verifier gate and a merger (the main swarm pattern)

This is the closest analogue to the OpenAI Navier–Stokes setup.

```
             ┌─────────── round r ───────────┐
 findings/ ─►│ group A (prove)    × kA workers│
 inbox/    ─►│ group B (disprove) × kB workers│─► results ─► VERIFY ─► passing ─► MERGER ─► findings/ (r+1)
 problem/  ─►│ group C (numerics) × kC workers│                       results            inbox/    (r+1)
             │ group D (literature)× kD       │                                          briefs    (r+1)
             └────────────────────────────────┘
```

1. **Groups.** Each group gets a different version of the problem statement or a different strategy: prove it, disprove it, search for counterexamples numerically, survey known partial results, attack a simplified special case. Diversity matters more than worker count.
2. **Workers.** Each worker gets its group's problem statement, the current `findings/` (read-only), its inbox, and a narrow assignment for this round.
3. **Verify.** The driver runs the verification gate (section 5) on every claim. Failed claims are recorded as dead ends with the failure output, so the next round doesn't repeat them.
4. **Merge.** One merger run (stronger model, high effort) reads the passing results and the dead-end log. It updates `findings/`, delivers messages between groups, and writes the next round's briefs, including how many workers each group should get.
5. **Repeat** until the target claim passes the gate, the budget runs out, or progress stalls for a set number of rounds.

The merger is the swarm's only planner. It can move workers from a dead group to a promising one, seed a group with another group's result (as OpenAI seeded the Euler blow-up result into the main run), or open a new group for an idea that emerged.

### 3.4 Peer messaging through file inboxes

Codex agents cannot message each other directly, so build mailboxes:

```
inbox/
  prove-03/
    2026-10-07T14-02-11_from-numerics-01.md
  numerics-01/
    2026-10-07T13-55-40_from-merger.md
```

Two ways to deliver:

- **Between rounds (simpler, deterministic).** Workers return messages in the `messages` field of their result. The driver writes them into `inbox/<to>/`. Next round, each worker's brief includes its unread inbox. No locking needed.
- **During a round (closer to OpenAI's setup).** Give workers `--add-dir inbox/` and tell them to write messages directly into `inbox/<to>/` and check their own inbox between steps. Use unique filenames (timestamp plus sender) so writes never collide. Messages only help if the recipient is still running; long-lived workers (section 3.5) make this worthwhile.

Address messages to roles or groups (`group:numerics`) as well as individual workers, and have the driver fan them out. Agents rarely know which specific worker to ask.

### 3.5 Long-lived workers with `resume`

Instead of a fresh worker every round, keep a fixed set of named workers and continue each one with `codex exec resume <thread_id> "<next brief>"`.

- **Pros:** workers accumulate context and intuition about their sub-problem; cheaper briefs.
- **Cons:** contexts grow and get compacted; stale assumptions persist; a confused worker stays confused.
- **Middle ground:** resume for a few rounds, then retire the worker and start a fresh one seeded with a handoff note the old one wrote (ask for it in the schema). Use `codex exec fork <thread_id>` to branch a promising worker into several variations.

Do not use `--ephemeral` for workers you want to resume.

### 3.6 Hierarchical: each exec worker runs a sub-team

Start each group lead with `--enable multi_agent` and a profile that sets `[agents]` limits. The lead uses `spawn_agent` for local helpers, while the external driver handles cross-group coordination. This multiplies parallelism without the driver tracking every agent, at the cost of less visibility into the helpers.

### 3.7 Adversarial pairs: prover and critic

For every claim, run a critic worker whose only job is to break it: find the gap in the argument, the failing input, the numerical instability. A claim reaches `findings/` only after it survives the critic and the formal gate. This catches errors that a formal verifier cannot express (wrong problem statement, a formalization that proves something weaker than intended).

### 3.8 Codex Cloud attempts

`codex cloud exec --env <ENV_ID> --attempts N "<task>"` (verified flags) runs best-of-N attempts on OpenAI's infrastructure against a branch (`--branch`). Useful for offloading a pattern-3.1 step without local compute. Each call is one task, so the multi-round loop still lives in your driver. Marked experimental in the CLI.

---

## 4. Shared state

### 4.1 Repository layout

Keep everything in one git repository, so every change is reviewable and revertible.

```
swarm/
  problem/              # problem statements per group (A.md, B.md, ...) - read-only for workers
  findings/             # verified results only; the shared knowledge base
    FINDINGS.md         # index: one line per result, with status and source
    lemmas/ data/ proofs/
  deadends.md           # claims that failed verification, with the failure output
  inbox/<agent>/        # messages
  schemas/              # JSON Schemas for worker, merger, planner results
  profiles/             # per-role Codex profile files (copied into $CODEX_HOME)
  verify/               # verification scripts (section 5)
  runs/r<NN>/<agent>/   # brief.md, events.jsonl, result.json, stderr.log, verify.log
  worktrees/            # git worktrees, one per worker (gitignored)
  driver.py
```

### 4.2 Worktrees for isolation

Each worker writes in its own git worktree on its own branch:

```bash
git worktree add -b "r03/prove-07" "worktrees/r03-prove-07" main
```

- Workers never overwrite each other's files.
- The driver (or merger) inspects each branch with `git diff main...r03/prove-07` and merges only what passed verification.
- Clean up with `git worktree remove` and delete branches of rejected work after logging them.

`--worktree` does this automatically, but you then have to discover the worktree path and branch afterwards. Manual worktrees keep paths predictable.

### 4.3 Who may write what

| Path | Workers | Merger | Driver |
| --- | --- | --- | --- |
| `problem/` | read | read | write (setup only) |
| `findings/` | read | write | write (after the gate) |
| `deadends.md` | read | write | write |
| `inbox/<self>/` | read | write | write |
| `inbox/<other>/` | write only in live-messaging mode | write | write |
| own worktree | write | read | read |

Enforce this by what you mount: workers get `findings/` through their worktree's checkout of `main` (a read-only copy at round start), not as a writable `--add-dir`.

### 4.4 Keeping context small

`findings/` grows. Workers should not read all of it every round.

- Keep `FINDINGS.md` as a short index the merger maintains.
- Have the merger write a per-group digest for each round's briefs.
- Tell workers to open full findings only when the index points to something relevant.

---

## 5. The verification gate

The gate decides what counts as progress. Without one, a swarm amplifies confident mistakes.

### 5.1 Kinds of verifiers

| Domain | Gate |
| --- | --- |
| Formal mathematics | `lake build` on a Lean project; reject any `sorry`, `admit` or new `axiom`; diff the theorem statement against a frozen copy so workers cannot weaken it. |
| Numerical science | Re-run the computation in a clean environment at higher precision or resolution; check convergence and tolerance; check against known solutions. |
| Software | Test suite, type checker, linters, plus a hidden test set workers never see. |
| Symbolic claims | Independent CAS check (SymPy, Sage) of each identity. |
| Anything else | A critic worker (section 3.7) plus a judge run with a strict rubric; weaker, so require agreement from several. |

### 5.2 Rules for the gate

- **The driver runs it, never the worker.** A worker's own "tests pass" is a claim, not a result. Re-run verification yourself on the worker's branch.
- **Freeze the target.** Keep the exact statement being proved (or the hidden tests) outside the worktrees and compare against it.
- **Run it in a clean checkout,** so leftovers in the worker's environment can't make it pass.
- **Log failures in full** to `deadends.md`. A good failure record is half the next round's brief.
- **Make it cheap where possible.** Use a fast pre-check (syntax, shape, quick numerics) before the expensive one (full Lean build, fine-grid simulation).

---

## 6. Driver implementations

### 6.1 Bash with `xargs -P` (minimal fan-out)

Good for pattern 3.1 and quick experiments.

```bash
#!/usr/bin/env bash
set -euo pipefail

round="${1:?round number}"
max_parallel="${MAX_PARALLEL:-8}"
codex_bin="$(command -v codex)"   # the real binary, not a shell function
export codex_bin round

run_worker() {
  local name="$1"
  local dir="runs/r${round}/${name}"
  local wt="worktrees/r${round}-${name}"
  mkdir -p "$dir"
  git worktree add -q -b "r${round}/${name}" "$wt" main
  timeout 2h "$codex_bin" exec \
    -C "$wt" -s workspace-write \
    --json --output-schema schemas/worker.json \
    -o "$dir/result.json" \
    - < "$dir/brief.md" > "$dir/events.jsonl" 2> "$dir/stderr.log" \
    || echo "$name exit=$?" >> "runs/r${round}/failures.log"
}
export -f run_worker

# Briefs were written to runs/r<round>/<name>/brief.md beforehand.
ls "runs/r${round}" | xargs -P "$max_parallel" -I{} bash -c 'run_worker "$1"' _ {}
```

GNU `parallel` adds retries (`--retries 2`), a job log (`--joblog`), and resumable batches (`--resume-failed`), which are worth having once runs get long.

### 6.2 Python asyncio (round-based swarm)

Pattern 3.3 needs state between rounds, so use a real language. A sketch of the structure:

```python
import asyncio, json, shutil, subprocess
from dataclasses import dataclass
from pathlib import Path

CODEX = shutil.which("codex")          # real binary; no shell function flags
ROOT = Path(__file__).resolve().parent
MAX_PARALLEL = 16

@dataclass
class Worker:
    name: str        # e.g. "prove-07"
    group: str       # e.g. "prove"
    brief: str
    model: str
    thread_id: str | None = None   # set when resuming a long-lived worker

async def run_worker(sem: asyncio.Semaphore, rnd: int, w: Worker) -> dict:
    async with sem:
        run_dir = ROOT / f"runs/r{rnd:02d}/{w.name}"
        run_dir.mkdir(parents=True, exist_ok=True)
        wt = ROOT / f"worktrees/r{rnd:02d}-{w.name}"
        subprocess.run(["git", "worktree", "add", "-q", "-b", f"r{rnd:02d}/{w.name}", str(wt), "main"],
                       cwd=ROOT, check=True)
        (run_dir / "brief.md").write_text(w.brief)

        args = [CODEX, "exec"]
        if w.thread_id:
            args += ["resume", w.thread_id]
        args += ["-C", str(wt), "--add-dir", str(ROOT / "inbox"),
                 "-s", "workspace-write", "-m", w.model,
                 "--json", "--output-schema", str(ROOT / "schemas/worker.json"),
                 "-o", str(run_dir / "result.json"), "-"]

        with open(run_dir / "brief.md", "rb") as stdin, \
             open(run_dir / "events.jsonl", "wb") as out, \
             open(run_dir / "stderr.log", "wb") as err:
            proc = await asyncio.create_subprocess_exec(*args, stdin=stdin, stdout=out, stderr=err)
            try:
                code = await asyncio.wait_for(proc.wait(), timeout=2 * 3600)
            except asyncio.TimeoutError:
                proc.kill()
                code = -1

        thread_id, usage = None, {}
        for line in (run_dir / "events.jsonl").read_text().splitlines():
            ev = json.loads(line)
            if ev.get("type") == "thread.started":
                thread_id = ev["thread_id"]
            elif ev.get("type") == "turn.completed":
                usage = ev.get("usage", {})

        result_path = run_dir / "result.json"
        result = json.loads(result_path.read_text()) if result_path.exists() else {"status": "blocked"}
        return {"worker": w, "exit": code, "thread_id": thread_id, "usage": usage,
                "result": result, "worktree": wt}

def verify(outcome: dict) -> bool:
    """Run the gate on the worker's branch in a clean checkout. Never trust the worker's own report."""
    ...

def merge_round(rnd: int, passed: list[dict], failed: list[dict]) -> list[Worker]:
    """Run the merger (one codex exec with a merger schema). It updates findings/, deadends.md,
    delivers messages into inbox/, and returns next round's workers."""
    ...

async def main(initial: list[Worker], max_rounds: int) -> None:
    sem = asyncio.Semaphore(MAX_PARALLEL)
    workers = initial
    for rnd in range(1, max_rounds + 1):
        outcomes = await asyncio.gather(*(run_worker(sem, rnd, w) for w in workers))
        passed = [o for o in outcomes if o["result"].get("status") == "claim" and verify(o)]
        failed = [o for o in outcomes if o not in passed]
        workers = merge_round(rnd, passed, failed)
        if not workers:      # merger signals completion or exhaustion
            break
```

This is a skeleton, not tested code. Fill in `verify` and `merge_round`, add retries, budget checks (section 8.3), and resumable state (section 8.5).

### 6.3 Profiles per role

Keep role settings out of the driver by using profiles (`-p <name>` layers `$CODEX_HOME/<name>.config.toml`):

```toml
# $CODEX_HOME/swarm-explorer.config.toml - wide, cheap exploration
model = "<fast model>"
model_reasoning_effort = "medium"

# $CODEX_HOME/swarm-merger.config.toml - careful synthesis
model = "<strongest model>"
model_reasoning_effort = "high"
```

Check the config key names against your installed version (`--strict-config` errors on unknown fields).

---

## 7. Writing briefs

Workers start with no context but the brief and the files you expose. Every brief should contain:

1. **Role and group:** "You are `prove-07` in the PROVE group. Other groups are trying to disprove the statement and to find counterexamples numerically."
2. **The problem version** for this group (path in `problem/`).
3. **Current state:** the merger's digest for this group, and pointers into `findings/` and `deadends.md`.
4. **This round's assignment:** narrow and concrete. "Prove lemma 4 for the case k ≥ 3 using the energy estimate in findings/lemmas/L2.md", not "make progress".
5. **Inbox:** unread messages, inlined.
6. **Rules:** where it may write, that the gate decides acceptance, that the theorem statement must not change, no network unless allowed, time budget.
7. **Output contract:** fill the result schema; put artifacts in its worktree; report dead ends honestly, because a documented dead end has value.

Templates with per-group placeholders (written by the merger) keep briefs consistent across hundreds of workers.

---

## 8. Operations

### 8.1 Concurrency

- Start at 4–8 parallel workers and raise gradually while watching for rate-limit errors in `stderr.log` and `events.jsonl`.
- Your account's rate limits are the real ceiling, not Codex or your machine. Each worker is mostly waiting on the API, but tool calls (builds, simulations) use local CPU and RAM; cap heavy verifiers separately from workers.

### 8.2 Timeouts and retries

- Wrap every run in a wall-clock timeout (`timeout 2h` or `asyncio.wait_for`).
- Retry rate-limit and transient failures with exponential backoff. Do not retry a worker that returned a well-formed `dead_end`; that is a result.
- Treat a missing `result.json` with exit 0 as a failure to investigate, not success.

### 8.3 Budget

- Sum `turn.completed.usage` across workers per round and per group.
- Set a token budget per round and a total budget; the driver stops scheduling when either is reached.
- Give the merger the spend per group so it can starve groups that aren't producing.

### 8.4 Observability

- Keep every `brief.md`, `events.jsonl`, `result.json`, `stderr.log` and `verify.log`. They are your only record of what agents did.
- Write a per-round summary (`runs/rNN/SUMMARY.md`): workers, passes, failures, spend, merger decisions.
- A tiny status command (count of running processes, passed claims, spend so far) is worth more than a dashboard.

### 8.5 Resumability of the driver itself

Runs last hours or days, so the driver must survive crashes:

- Persist round state (`state.json`: round number, worker list, thread IDs, budget used) after every step.
- On restart, skip workers whose `result.json` already exists and re-run only missing ones.
- Run the driver under `tmux`, `systemd-run --user`, or similar, not in a terminal you might close.

---

## 9. Safety

- **Sandbox by default.** Use `-s workspace-write` and add only the directories a worker needs with `--add-dir`.
- **Bypass only inside isolation.** If workers need `--dangerously-bypass-approvals-and-sandbox` (e.g. to run arbitrary build tools), run the whole swarm in a container or VM with no credentials you care about, and a copy of the repo.
- **Network.** Decide whether workers may use the network. For research swarms, a cached or read-only source mirror is safer than open internet access.
- **Secrets.** Workers can read anything in their sandbox. Keep API keys and tokens out of the worktrees and environment except what Codex itself needs.
- **Nothing external.** Workers should not push, open PRs, or post anywhere. The driver or you publish results after review.
- **Disk.** Hundreds of worktrees and session files add up. Use `--ephemeral` for one-shot workers, prune worktrees each round, and archive old runs.

---

## 10. Failure modes and fixes

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Workers hang with `Reading additional input from stdin...` | stdin not closed | `< /dev/null` or pass the brief with `-` |
| Workers fail on every command | approvals required in non-interactive mode; shell function flags not applied | pass `-s` / `--approve-for-me` / bypass flag explicitly |
| Many workers produce the same idea | briefs too similar | diversify problem versions, strategies, models; feed `deadends.md` |
| Claims pass the worker's own check but fail the gate | worker verified a weakened or different statement | frozen target, clean checkout, driver-run gate |
| Findings grow but nothing converges | merger lacks a goal-directed plan | require the merger to state the current best route and what each group contributes to it |
| Context overflow in long-lived workers | resumed too many rounds | retire workers after N rounds with a handoff note |
| Merge conflicts in `findings/` | several writers | only the merger and driver write `findings/` |
| Spend spikes | runaway worker loops | per-worker timeouts, per-round budgets, kill on budget |

---

## 11. Suggested path

1. **Toy run.** Pick a problem with a fast, unambiguous verifier (e.g. a small Lean lemma or a function with hidden tests). Run pattern 3.1 with 4 workers using the bash driver. Confirm the event parsing, schema output, worktrees and gate work end to end.
2. **Two rounds.** Add the merger and file inboxes (between-rounds delivery). Check that round 2 briefs actually use round 1 findings and dead ends.
3. **Groups.** Split into 2–4 groups with different problem versions. Add the budget and state files.
4. **Scale.** Raise concurrency step by step; add long-lived workers, critics and live messaging only where they improve results.

---

## 12. Not verified here

- `codex exec resume` and `fork` behaviour with `--json` and `--output-schema` in long runs.
- Whether `--worktree` reports its path or branch in the event stream.
- Event types beyond the four observed in a minimal run.
- Config key names in the profile examples (`model_reasoning_effort` and similar).
- Rate limits for your account and the concurrency they allow.
- `codex cloud exec` end to end (only its `--help` was read).
