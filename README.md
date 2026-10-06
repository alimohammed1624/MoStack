# Test Project

This repo exists to test Claude Code skills — it's a harness, not a product. The `skills/`
directory holds the skills under test, exercised against this repo and its org to check that they
trigger correctly and behave as specified. [`docs/github-surfaces.md`](docs/github-surfaces.md) is
the reference for the GitHub surfaces gh-wrapper routes through.

## Skills under test

Each skill is a `SKILL.md` with YAML frontmatter (`name`, `description`) that Claude auto-loads when
its description matches the task. The description is itself under test: does the right skill fire
for a given phrasing, and does it stay silent when another is the better fit?

There are **two** skills: herdr-agents and gh-wrapper. Each has a `SKILL.md` entrypoint; some include
supporting references.

### herdr-agents

[Herdr agents](skills/herdr-agents/SKILL.md) drives Claude Code, Codex, Cursor Agent, and other supported coding
agents when the user requests them. It uses the dedicated `agents` Herdr session inside and outside
Herdr, switching to the current or another session only on explicit user request. It covers task
briefs, completion and output recovery, and follow-up. Agents and
panes remain open after results are collected; the driver asks whether to close them.

**What to probe:** all calls target `agents` unless the user explicitly selects another session,
even when running inside Herdr; a stalled prompt is
inspected before retrying; blocked UI is read before responding; cleanup preserves existing panes.

### gh-wrapper

Portable — it encodes no policy from any particular org or workflow. Whenever a `gh` CLI command would otherwise run — typed by
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

## Environment notes for testers

Verified against the `msa1624` org:

- **The field set is discovered, not fixed.** What matters for testing is that gh-wrapper reads the
  set at call time rather than recalling one, and that a field the org doesn't define is reported
  absent rather than invented. `Size` and `Estimate` are the standing example: neither exists here.
- **Relationships is writable at rung 1** (`gh issue edit --add-blocked-by`).
- **Org-only features.** Issue Fields, issue types, and Teams do not exist on a personally-owned
  account. **Projects v2 are the exception and are not org-only** — a personal account has
  `user(login:){projectsV2}`, so an empty `organization(...)` result there is the wrong query, not
  an absence.
- **Projects v2 board membership is a fourth mechanism** on an issue, alongside Issue Fields,
  Milestone, and Relationships — and the board *item's* fields are a fifth. Those last two are the
  ones that fail silently, and they fail independently: an issue on no board looks entirely normal,
  and an item whose `Status` never landed looks planned. Rung 1 is
  `gh project item-add --url` and rung 2 is `addProjectV2ItemById`. Adding is idempotent.
  gh-wrapper discovers and **reports**, because it has no confirmation surface. **What to probe:**
  that an issue created in a repo *outside* the board's auto-add scope is reported rather than
  silently landing nowhere; that "no project exists" and "a project exists and this issue isn't on
  it" never collapse into one silence; and that several discovered projects render `— ask` rather
  than a pick.

## Notes for testers

- The tracked deletions of `auth.js`, `middleware.js`, and `worker.js` in this repo's history are
  fixture data — sample commits for the skills to reference (e.g. "Fix memory leak in background
  worker (fixes #7)"), not application code.
