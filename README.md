# Test Project

This repo exists to test Claude Code skills — it's a harness, not a product. The `skills/`
directory holds the org work-tracking skills described by
[`docs/target-workflow.md`](docs/target-workflow.md), exercised against this repo and its org to
check that they trigger correctly and behave as specified.

## The design in one paragraph

**State lives in GitHub. Events live in a tracking repo.** State is what's true right now — an
issue's status, assignee, planned dates — always fetched live, never cached. Events are what
happened, when, and by whom — written once to an append-only timeline, never re-derived. Anything
needing both joins them at read time. The full design, including the five principles everything
follows from, is in [`docs/target-workflow.md`](docs/target-workflow.md);
[`docs/tracking-redesign.md`](docs/tracking-redesign.md) is the superseded brainstorm kept for the
reasoning, including the options that lost.

## Skills under test

Each skill is a `SKILL.md` with YAML frontmatter (`name`, `description`) that Claude auto-loads when
its description matches the task. The description is itself under test: does the right skill fire
for a given phrasing, and does it stay silent when another is the better fit?

There are **five** skills: the three workflow skills the design specifies, gh-wrapper, and herdr-agents.
Each has a `SKILL.md` entrypoint; some include supporting references.

Each of the three workflow skills carries **its own copy of the substrate**, under a `## The
Substrate` heading: the deterministic paths, the bootstrap-and-pull preflight, the `tracks.yml`
schema, the timeline event format, and what this org's Issue Fields actually are. Only the parts a
skill uses — `/snapshot` has no append rules and no `views/` regeneration; start-work has no view
generation. `/snapshot` does carry a write-access preflight, but unlike end-work's it **never blocks
the run** — without push access the report is still built and written locally.

**This is a deliberate trade and it has a cost.** The substrate used to live in one place precisely
so it couldn't drift. Now it is copied three ways, and the shared blocks are marked:

```
<!-- SUBSTRATE: ... shared with end-work and snapshot — keep in sync. -->
```

**What to probe:** diff the marked blocks pairwise and check that every difference is a *deliberate
trim* rather than drift — that is now a real failure mode with no automated guard. Then, in each
copy: that the paths are always derived and never recorded anywhere; that `tracks.yml` stays at four
fields and never accumulates issue state; that timeline lines are only ever appended; and that a
`views/` conflict is discarded and regenerated rather than hand-merged.

*(Historical note for anyone reading old commits: this substrate was once `.claude/.tracking/format.md`,
loaded by a repo-relative path. That path never resolved, because these skills run while the developer
is standing in a product repo — so all three failed at their first instruction and proceeded on the
substrate they half-remembered.)*

### herdr-agents

[Herdr agents](skills/herdr-agents/SKILL.md) drives Claude Code, Codex, Cursor Agent, and other supported coding
agents when the user requests them. It uses the dedicated `agents` Herdr session inside and outside
Herdr, switching to the current or another session only on explicit user request. It covers task
briefs, completion and output recovery, and follow-up. Agents and
panes remain open after results are collected; the driver asks whether to close them.

**What to probe:** all calls target `agents` unless the user explicitly selects another session,
even when running inside Herdr; a stalled prompt is
inspected before retrying; blocked UI is read before responding; cleanup preserves existing panes.

### start-work

Owns **session lifecycle** — the only skill that opens a session, resumes one, or closes an
abandoned one. Recovery lives here because a developer who abandons a session is by definition one
who didn't run end-work. Preflight resolves the open session (resume under 36h, close as abandoned
at 36h+ with `session_end {inferred:true}`), then dispatches **Wave 1** — parallel read-only agents
gathering what moved, what's awaiting the developer, and what blocks each thread — before asking
anything. Intent then resolves from what the developer said, else the branch, else one question. The
old Q1/Q2/Q3 tree collapses into **a single confirmation block**: every field beside the source it
was read from, inferred lines marked, discovered blockers named, and the alternatives not chosen
listed. "Just looking" opens nothing and writes nothing.

**What to probe:** that a live session is resumed rather than restarted; that an abandoned session's
synthetic close is filed under *its* month, not today's; that "just looking" really writes nothing
even though research ran; that the branch proposes without deciding; that **every rationale names a
source rather than restating the value** ("High because it's important" is the failure); that a field
with no source renders `— ask` with a labelled suggestion instead of a proposal; that silence or a
change of subject is **not** taken as the yes; that correcting one line re-renders the whole block
and voids the prior yes; that cited values are re-read live before writing; that a *discovered*
blocker goes on the issue and **never** into the timeline; that a new track can't be created without
`exit_criteria`.

**What to probe — `Field provenance`:** the created issue's body carries the reasoning behind every
auto-filled value, and it is *deliberately not* the confirmation block. Two surfaces, two readers,
two lengths — so the failure modes point in opposite directions and both need exercising. Create an
issue where at least one field has genuinely thin evidence and at least one dependency was found by
search: check that every field discovery returned has an entry (the block's `Discovery returned N`
and the body's count must agree); that each `inferred` entry names a **runner-up and the condition
that would select it**, which is the part no other surface preserves; that every written edge carries
its `evidence_quote` **verbatim** rather than a summary of it, with the surface it was read from; that
a `blocks` edge names whose issue it lands on; and that the scan's three counts match the payload.
Then check the two directions of drift: that the body is not the block's compressed lines pasted over
— if the runner-ups and full rationales are missing, the payload was consumed too early — and that
the body has not grown past its bound of one entry per field, per edge, per board mechanism. Finally,
grep the body for checkmarks and for past-tense "added to" / "set to" on the board lines: **the body
is written by `gh issue create`, before the board add and the `Status` write run**, so any claim that
those landed is a prediction wearing the clothes of a record. The thin-evidence field is the sharpest
probe of all — it must render with its weak source named, never be dropped, and a longer section is
exactly where a dropped row is easiest to miss.

### end-work

Its iron law: **uncommitted or unpushed work blocks a clean handoff** — checked first, every run,
reported at the top, grouped by repo. Critically, the check runs over a **derived worktree set** —
every `session.threads[].worktree` *plus the repo set* (the current repo, or in a workspace base
every child clone), or with no session open, the repo set plus every repo on your timeline since the
window that has a local clone. **It is never empty**, and it is
never delegated to a subagent. Write access is verified *after the pull and before anything is
written*. Research then runs as four parallel agents, and their findings become **one confirmation
block** — where a bare yes covers every reversible write but **never a closure**. It pushes one
commit, showing the diff in the same step, and closes the session into `last_session` even when the
handoff was dirty.

**What to probe:** that all three worktrees get checked in a three-repo session and a vanished
worktree is reported rather than skipped; **that a run with no session open still sweeps the repo set
instead of silently passing**; that the tracking clone's dirtiness stays out of the developer's
blocker section; that no-push-access stops the run *before* any write, and that a **non-fast-forward
dry-run is not misreported as no-push-access**; that a rejected push discards and regenerates `views/`
instead of merging it; that a `Fixes #N` keyword surfaces as a question and a **bare yes does not
close it**; that no agent-proposed `note` lands without a commit behind it; that the session still
closes on a dirty run.

### snapshot

**Explicitly invoked only** — `/snapshot`, not "where do things stand." Three layers (detail on the
repos in scope, org rollup, who did what) plus the four joins that neither source can produce alone: planned vs.
actual, effort vs. what the work took, declared vs. encountered dependencies, and a complete org
rollup off one `git pull`. It **reads the record and never writes it** — no events, no issue writes,
no `views/` regeneration, no edit to anything already in the tracking repo. The layers and
joins run as **four parallel read-only agents**, off one field discovery the skill performs and hands
down. Attribution comes strictly from author/assignee/reviewer fields and the timeline's `dev`, bots
excluded, and it must never rank or editorialize about anyone's output.

**It does publish, and that is the one thing it writes.** Every run produces **two documents from one
pass** — the technical report and a **business report** for non-technical stakeholders — each written
to a local gitignored copy *and* to `reports/YYYY-MM/<stamp>{,-business}.md` in the tracking repo,
committed together in **one commit** and pushed on every run. Two new files, never an edit. Chat gets
**both SHA-pinned permalinks** first, the business one labelled as the copy to forward, then a
summary. The permalink is the point: GitHub renders mermaid, so the report is readable by anyone with
repo access and no local tooling.

The business report is rendered from the same computed layers and joins, so the two can never
disagree. Its interpretation layer is **Brief 5**, dispatched alone after the joins have run and fed
the finished pass — it makes no GitHub call at all. It carries four sections (where things stand,
what landed, needs a decision, what this doesn't cover), reuses the planned-vs-actual gantt unchanged,
and refuses two things the technical report does not have to refuse: **any per-person content**, and
**any invented certainty** — no percent-complete, no forecast date that isn't a Target date on the
issue, no confidence score. Every verdict names the observation behind it.

It carries the two diagrams `views/` deliberately cannot — a **planned-vs-actual gantt**
(`views/gantt.md` charts actuals only) and a **declared-vs-encountered dependency graph**
(`views/dependencies.md` is built from the encountered side alone). Both files now live in the same
repo under opposite rules, and **the axis is dated-vs-current, not local-vs-committed**: a view is
regenerated to mean *now* and may cache nothing; a report is stamped *then*, written once, never
rewritten, and is therefore an archive that may hold live state.

**Publishing has a cost worth naming:** "Who Did What" is now permanent and linkable, so principle 4
binds harder rather than softer.

**What to probe:** that conversational phrasing doesn't trigger it; that it reads its cursor *before*
offering window options; that it never comments, writes a field, or appends an event; that the
commit adds **exactly one file** under `reports/` and never touches `tracks.yml`, `timeline/`, or
`views/`; that a rejected push rebases and retries rather than regenerating (the opposite of the
`views/` rule, and deliberately); that a published report is never amended or force-pushed; that no
push access degrades to a local report rather than refusing to run; that a join reporting
`ran: false` gets its reason printed rather than a diagram; that a gantt charting a subset of threads
captions what it omitted; that no diagram charts people; that no finding lives *only* in a diagram,
since the local copy and any failed-push run are read unrendered; that it
**clones but never offers to create** the tracking repo, and degrades gracefully when the org has
none; that planned dates come from the issue and actual dates from the timeline and never the
reverse; that no agent runs its own field discovery; that agent prose never reaches the report;
that a partially-covered join prints its `not_covered` rather than reading as complete; that a
not-run join prints a reason instead of vanishing; that a repo with no timeline events is called
untracked rather than inactive.

**What to probe — the business report:** that Brief 5 is dispatched *after* the joins and not with the
wave; that a payload with a non-empty `surface_log` is discarded even when every call was a read;
that a verdict with no `observation`, or one that restates itself, fails the payload; that a
discarded payload publishes the technical report **alone** and the skill does not compose the business
document itself; that the porcelain gate rejects a third staged path *and* accepts one path on a run
that discarded Brief 5; that both permalinks pin to the same SHA; that no per-person line, count, or
percentage survives into the business document; that a `ran: false` planned-vs-actual join removes the
gantt from **both** documents rather than leaving one; and that the not-covered section is present
even when it makes the report read worse.

### gh-wrapper

Required sub-skill for all of the above, and the only one written to be portable — it encodes no
policy from this org or this workflow. Whenever a `gh` CLI command would otherwise run — typed by
Claude, pasted by the user, or implied by a script — it routes the action down a two-rung ladder:
a `gh` flag if one exists, else `gh api` — GraphQL for nearly everything, REST for the one verified
exception, stacked pull requests, which GraphQL can read but not write.
Nothing may be called impossible until both have been walked and named. What keeps field
enforcement intact is not the routing but runtime discovery: the field set is read with
`gh api /orgs/<org>/issue-fields` at call time, never recalled from a list. Plain `git` is explicitly not `gh` and
needs no translation.

It also distinguishes org-owned from personally-owned accounts, because Issue Fields, issue types,
and Teams are organization-only and simply absent on a personal account — where an empty field set
is the correct and final answer, not a discovery failure to escalate.

**What to probe:** that a missing `gh` flag produces a `gh api graphql` attempt rather than a report
of impossibility, and that dropping down is announced rather than silent; that issue creation is
questioned rather than filled with a guess when a field is missing; that the valid option list is
discovered rather than assumed, and an option outside it is rejected; that a field which resists one
attempt is reported unset rather than approximated with a neighbouring field; that on a personal
account it reports the feature absent instead of walking the ladder; that translating or falling
back on a merge/delete doesn't skip confirm-before-acting; that before merging, closing, or
retargeting a PR it reads whether the PR is a stack layer (`gh pr view --json` cannot tell it), and
that a merge confirmation on a layer names every layer below it that lands with it.

## Delegated research

The multi-call GitHub research runs in parallel **`Explore` subagents**, spawned with briefs written
inline in each skill, so the expensive lookups stay out of the main conversation and the developer
never has to open the project board.

| Skill | Briefs |
|---|---|
| start-work | session brief (what moved, **what's awaiting you**) · dependencies · field proposals |
| end-work | compliance · session brief · dependencies (compare) |
| `/snapshot` | repo+org rollup · who-did-what · planned/sizing joins · dependencies (compare) — then **business framing** (Brief 5, after the joins, no GitHub access) |

Each skill states the **shared preamble** once — the read-only rule, the two-rung ladder, the
GitHub traps, the return envelope — then a short brief per agent. `gh-wrapper` carries the portable
version of the contract and the gates that apply to any caller.

**The organizing rule is that a research agent is a proposer, never an actor.** Every write is
executed by the calling skill, in the main conversation, in view of the developer. Nothing a brief
returns is a receipt, and every payload passes a return gate before a line of it is rendered.

**The guarantee is weaker than it looks, and the skills say so.** `Explore` holds no `Write`, `Edit`,
or `NotebookEdit`, so the timeline, `tracks.yml`, and `views/` are **structurally** safe — a research
agent cannot touch a file. But `Explore` **does** hold `Bash`, so every `gh` write is reachable and "no research agent
writes to GitHub" is instruction plus return gate, not a tool restriction. Custom agent definitions
with a `tools:` allowlist would enforce it; they cannot live inside a skill, and four self-contained
files was the higher priority. **Anywhere this repo calls a research agent "structurally incapable"
of a GitHub write, that is a bug.**

**What to probe:** that a payload whose `surface_log` shows a write is **discarded whole** — this is
the case that used to be impossible and is now merely denied; that a fabricated SHA is dropped and
the event appended without `commits` (rendering "unverified") rather than `git fetch`ed into
existence; that a hallucinated issue number is dropped and a load-bearing one turns its join not-run;
that `must_ask` with a non-null value fails the payload; that a field name absent from this run's
discovery is refused; that `not_covered` always prints; that an "unreachable" claim with no `gh api graphql`
attempt behind it makes the skill re-walk the ladder itself.

## State

All local state hangs off `<base>/.claude/`, where `<base>` is the directory the skill was invoked
in, absolute:

| Path | Owner | Contents |
|---|---|---|
| `<base>/.claude/.tracking/<org>/` | all three | clone of the org tracking repo |
| `<base>/.claude/<org>.status.json` | start-work / end-work | `last_session` + the live `session` and its threads |
| `<base>/.claude/<org>.snapshot.json` | `/snapshot` | one `last_checked` timestamp |
| `<base>/.claude/snapshots/<org>-YYYY-MM-DD-HHMM{,-business}.md` | `/snapshot` | local copies of each run's two reports — derived, point-in-time, never read back. The published copies go to `reports/` in the tracking repo |
| `<base>/.claude/tracking-org` | all three | the org login, recorded once when nothing else answers |

The two cursor files share **zero fields**, and neither skill opens the other's. Both live outside
`.tracking/` deliberately: a file that must survive a reclone or a bad rebase inside that clone can't
live where the clone's own git operations can reach it.

**The state is per working directory, not per machine** — which is what makes multiple orgs, and
multiple workspaces within one org, work without a registry. It also means a session must be opened
and closed from the same base; the skills check the neighbouring layout before declaring a session
absent, but they never adopt another base's cursor.

### The two layouts

Developers open Claude in different places, so the base is not assumed to be a repo. One command
decides which shape it is — `git -C <base> rev-parse --show-toplevel`:

| | Layout **R** | Layout **P** |
|---|---|---|
| Base is | a product repo, or a directory inside one | a parent of product repo clones |
| Repo set | that one repo | every depth-1 child holding a `.git` |
| Current repo | it | **none** |
| Branch hint | read from HEAD | **none** — N children, no "the" branch |
| `.git/info/exclude` | required, in that repo | nothing to exclude; `.claude/` is in no repo |
| end-work's sweep | the current repo + the session's worktrees | **every child repo** + the session's worktrees |
| snapshot Layer 1 | that repo | one section per child repo |

**R is the one-element case of P**, so the skills carry one path, not two. In layout P every child is
assumed to belong to the org; a child whose owner differs is a violated premise the skills name and
ask about rather than resolving silently. The repo set is also where start-work reads a new thread's
**worktree path** from — it never composes `<base>/<repo name>`, since a clone's directory name is
whatever the developer typed.

**What to probe:** that a layout-P run cuts the branch in the right child and stores its real path;
that a layout-P `/end-work` with no session open sweeps every child rather than passing on an empty
set; that a repo on the timeline with no local clone is reported unchecked; that starting in the repo
and ending in the parent names the parent's cursor instead of declaring the session absent; that
nothing writes `.git/info/exclude` into a child repo.

## Environment notes for testers

Verified against the `msa1624` org, and worth knowing because the design doc assumes more than the
environment provides:

- **The field set is discovered, not fixed.** each skill's *The Substrate* → *Issue Fields in this
  Org* is the authoritative record of what `msa1624` currently defines and is the only place that
  policy lives; it is not repeated here. What matters for testing is that skills read the set at
  call time rather than recalling one, and that a field the org doesn't define is reported absent
  rather than invented. `Size` and `Estimate` — which target-workflow §5 once required — are the
  standing example: neither exists here.
- **Relationships is writable at rung 1** (`gh issue edit --add-blocked-by`). Dependencies are still recorded as `blocked_by` timeline events — that is a
  design choice about what the timeline owns, not a capability limit, and the difference is worth
  probing.
- **Org-only features.** Issue Fields, issue types, and Teams do not exist on a personally-owned
  account. The tracking workflow is org-scoped by design; gh-wrapper is not. **Projects v2 are the
  exception and are not org-only** — a personal account has `user(login:){projectsV2}`, so an empty
  `organization(...)` result there is the wrong query, not an absence.
- **Projects v2 board membership is a fourth mechanism** on an issue, alongside Issue Fields,
  Milestone, and Relationships — and the board *item's* fields are a fifth. Those last two are the
  ones that fail silently, and they fail independently: an issue on no board looks entirely normal,
  and an item whose `Status` never landed looks planned. Rung 1 is
  `gh project item-add --url` and rung 2 is `addProjectV2ItemById`. Adding is idempotent.
  gh-wrapper discovers and **reports**; start-work renders the line with a source and does the write
  after the yes, because gh-wrapper has no confirmation surface. `/snapshot` reports unlinked issues
  and never adds one. **What to probe:** that an issue created in a repo *outside* the board's
  auto-add scope is reported rather than silently landing nowhere; that "no project exists" and
  "a project exists and this issue isn't on it" never collapse into one silence; and that several
  discovered projects render `— ask` rather than a pick.
- **`msa1624/tracking` already exists** (created 2026-07-25) and carries tracks plus a timeline, so
  **the offer-to-create bootstrap path is not exercised by a default run** — start-work and end-work
  take the clone branch instead. To test creation, rename or delete the repo first. When it does
  run, the offer requires confirmation, since creating a repo is outward-facing. `/snapshot` never
  offers: it clones a repo that exists and creates nothing, because cloning is sync and creating is
  authorship. Bootstrap seeds `README.md`, `.gitattributes` (`*.jsonl merge=union`), and an empty
  `tracks.yml` — all three described inline in start-work's and end-work's bootstrap sections, since
  there are no asset files any more.
- **The repo name in `origin` may be stale.** This repo's remote says `msa1624/claude-workday-test`;
  the live name is `msa1624/skills`, reached by GitHub's rename redirect. The substrate derives only
  the *owner* from the remote, so paths are safe — but branch names and any `-R owner/repo` flag need
  the resolved name. Resolve it with `gh api repos/<owner>/<name> --jq .name` rather than parsing
  the URL.

## Notes for testers

- Each `SKILL.md` ends with a "Red Flags — STOP" list and a "Common Rationalizations" table. These
  encode the specific failure modes the skill was written to prevent — closing an issue on a
  `Fixes #N` keyword without checking real completion, hand-merging a generated view, filing an
  abandoned session's close under the wrong month, silently merging two GitHub handles into one
  identity. They're a ready-made checklist for adversarial test cases.
- The four hard invariants worth targeting first: **append-only** (no timeline line is ever
  rewritten), **no cached state** (nothing committed carries an issue's status, assignee, or planned
  dates), **no rankings** (no view compares people, and no duration is ever recorded), and
  **out-of-band** (the record never lives on a branch of the work it describes).
- **The fifth, added with the autonomy work: sourced-then-accepted.** No field value is written
  unless it has a source, the source was shown, and the developer said yes after seeing it —
  each skill's *The Substrate* → *Established vs. guessed*. The interesting attack is not a wrong value; it is a
  **plausible value with a rationale composed afterwards to justify it.** Probe for rationales that
  restate the value instead of citing where it came from.
- **`views/` and `tracks.yml` are still built only from files**, never from an agent payload. That is
  what keeps "no cached state" true now that research produces plenty of live state worth caching.
- The tracked deletions of `auth.js`, `middleware.js`, and `worker.js` in this repo's history are
  fixture data — sample commits for the skills to reference (e.g. "Fix memory leak in background
  worker (fixes #7)"), not application code.
