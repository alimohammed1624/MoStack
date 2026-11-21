---
name: gh-wrapper
description: Use when about to run any `gh` CLI command (`gh issue`, `gh pr`, `gh repo`, `gh api`), when the user pastes one, when setting a custom Issue Field (org-defined single-select, date, number, or text) or an issue type on an issue, when creating an issue that may belong on a Projects v2 board, when setting Projects v2 board item fields (Status, Size, Estimate, Iteration/sprint, or any board-defined field), when linking issues across repositories, when opening a pull request or linking one to its issue, when running `gh stack` or creating a PR whose base is another PR's head branch, when about to merge, close, or retarget any PR (it may be a stack layer), or when about to report that a GitHub field, board membership, relationship, PR link, or stack membership cannot be set, or when asked to add screenshots or images to a PR or issue body
---

# GitHub CLI routing

Use the current `gh` flags where they cover the task, then `gh api` for the
remaining capability. Most API work here uses GraphQL; stack writes use REST.
Plain `git` operations on a local clone need no routing through this skill.

Read only the reference needed for the current operation:

| Operation | Route and reference |
|---|---|
| Org Issue Fields and issue types | Discover at call time; `setIssueFieldValue` for field values. [Field mechanics](references/fields.md) |
| Projects v2 membership | `gh project item-add`, then `addProjectV2ItemById`. [Field mechanics](references/fields.md) |
| Board-native fields, including iteration | `gh project item-edit`, then `updateProjectV2ItemFieldValue`. [Field mechanics](references/fields.md) |
| PR creation and issue links | `gh pr create`, then `createPullRequest`; closing keywords go in the body. Follow the inline PR rules below. |
| Stack membership and operations | Read GraphQL `stack` / `stackEntry`; write with `gh stack` or REST. Read [stack mechanics](references/stacks.md) before operating on a stack. |
| Dependencies, sibling discovery, search, rollups | Relationship flags accept URLs; GraphQL reads can include edges. Read [search mechanics](references/search.md) for scans or multi-repo research. |
| Delegated research | Read [research protocol](references/research.md) before dispatch or accepting a return. |
| Screenshots or images in issue/PR bodies | Only when asked; read [screenshots.md](screenshots.md). |
| Other issue, PR, repo, file, or user operations | [Command reference](references/commands.md) when the command is needed. |

## Routing and authorization

1. Read `gh <command> --help`; flags change between releases. Use a supported
   flag rather than inventing one or relying on memory.
2. If no flag covers the task, use `gh api`: usually GraphQL, or the REST path
   documented for that feature. An installed `gh stack` extension is the CLI
   route; check `gh extension list` before using it.
3. Before reporting a capability unavailable, try both applicable routes and
   name what you tried. Feature absence established by discovery, such as Issue
   Fields on a personal account or stacks unavailable on a repo, is a complete
   answer and needs no further escalation.

This order describes capability, not cost. A GraphQL rollup or sibling query
that includes edges can be the initial route because it avoids many REST calls.
State the route and reason briefly; delegated research has its own reporting
rules in the research reference.

Inspect every API response for `errors`, including HTTP 200 responses. A null
under an error is a failed read, not evidence of absence. Introspect before
concluding a GraphQL field is missing; an error naming another mutation can
identify the right mechanism. Time limits leave work **unset or unfinished**,
not proven impossible.

Routing does not grant authorization. Confirm a merge, delete, or send before
executing it; existing authorization from the caller still applies. Opening a
PR is the caller's decision because it requests review time. This skill governs
execution after that decision, not automatic PR creation.

## Field values and confirmation

Account for every discovered writable org Issue Field and board-native field,
not just familiar names. Use these outcomes in order:

1. Use a value established by the conversation or another verified source.
2. Otherwise propose a **marked guess with a named, traceable source**, such as
   the parent's value, siblings, or an org convention, for caller confirmation.
3. If neither is possible, report the field **unset by name**, with the reason.

A source must explain where the value came from: “parent #38 and sibling #43
are High” supports a proposal; “High because it is important” does not. A value
with no traceable source cannot be proposed as a sourced guess. Distinguish
proposed values from values actually written.

Hand the complete set, with sources and the dependency-scan outcome, to the
calling workflow for one confirmation rather than one question per field.
This skill has no confirmation surface of its own. Write guesses only after
caller confirmation; if the caller has no confirmation surface, report them
unset. Board field values also pass through that surface; discovering a board
alone authorizes its membership link, not its field values.

Do not guess structural facts: which repository holds the work, which of several
boards it belongs to, or whether a dependency exists. Surface those ambiguities
to the caller. Do not replace an unavailable field with a nearby field, label,
comment, or prose in the issue body. If a requested field is not defined, say so.
Report unset Issue Fields and board fields together, by name.

Delegation preserves these rules. A research agent's inference is a proposal,
not a verified value or write receipt. A caller may require stronger evidence;
its rules do not lower this minimum.

## Context and owner capabilities

Infer owner/repo from `git config --get remote.origin.url` when `-R` is absent.
Resolve “me” with `gh api user --jq .login` when needed.

Before Issue Field or issue-type work, establish whether the owner is an
`Organization` or `User`. Use `repositoryOwner(login:) { __typename }`, or
`gh api /users/<owner> --jq .type` if that is all you need. Prefer
`repository.viewerCanSeeIssueFields` for the field capability check because it
also covers permissions. The field reference combines these checks with
`repository.issueTypes` and discovery in one query.

Issue Fields, issue types, and teams are org-only. `User.issueFields` does not
exist, `Repository.issueTypes` is null for a personal account, and
`/orgs/<owner>/issue-types` returns 404 there by design. Report the feature absent
and set nothing; do not treat these results as an auth or discovery failure. For personal-account
team-like lookups, use repository collaborators.

Projects v2 exist for both owner kinds: use `organization(login:)` for orgs and
`user(login:)` for users. An empty organization-root project query for a personal
account is incomplete discovery; query the user root. Sub-issues and dependencies
are repo-scoped and use the same shapes for both owner kinds.

## Before creating an issue

Run all six preflight lookups before every create, gathering independent inputs
together. Task wording, a test issue, or missing parent/siblings does not exempt
a step. Apply the owner capability checks to org-only lookups.

1. **Issue Fields:** `/orgs/<owner>/issue-fields`; account for every field.
2. **Issue type:** `/orgs/<owner>/issue-types`; validate the type if used by the repo.
3. **Board membership:** discover `projectsV2` and resolve which board(s) apply.
4. **Board item fields:** discover each applicable `projectV2.fields` completely,
   including type-specific values; account for every writable field.
5. **Dependencies:** read the parent and siblings, and run a bounded scan of the
   owner's open issues in both directions. Return proposed edges supported by
   quoted direction, plus the scan count and outcome even when no edges qualify.
6. **Assignee:** use `--assignee "@me"` unless another person was named. A named
   person replaces this default rather than being added alongside it.

Source the field set or propose marked guesses and send it, with the scan
outcome, to the caller's confirmation surface. No parent and no siblings means
the scan supplies step 5's input; it does not eliminate the scan.

Discover everything before writing. Apply issue values, type, supported
relationship flags, and assignee as part of creation; API field writes use the
created issue's node id. Add the issue to the board, then immediately set its
confirmed board fields: those writes require the item id returned by the add.
Membership alone sets no board field values.

Claiming an unassigned existing issue is permitted. Reassigning someone else's
issue is a handoff, not an extension of the create-time `@me` default.

## Issue Fields and types

Org Issue Fields live on the issue independently of Projects v2. “Set the project
fields” refers to those fields; “add this to the project” refers to membership.
Milestones and dependency relationships are separate native mechanisms too.

Discover names, types, and single-select options at call time; never hardcode
an org's schema. Discovery has no `required` flag, so project policy determines
requiredness, while every returned writable field still needs an outcome.
Validate issue types against `/orgs/<owner>/issue-types` before using `--type`.

As of `gh` 2.96.0, no CLI flag writes org Issue Fields. `gh project item-edit`
writes a different feature. Read [field mechanics](references/fields.md) for the
union fragments and verified `setIssueFieldValue` batch mutation, clearing, and
read-back. Verify writes with the issue REST response's `.issue_field_values`.

Org field/option ids (`IFSS_`, `IFD_`, `IFSSO_`) are distinct from Projects v2 ids
(`PVTF_`, `PVTSSF_`, `PVTI_`). Do not substitute a board id or another field.
A mirror-write error identifies the issue-field mechanism; it is not a
permission failure. Both `updateIssueFieldValue` and `setIssueFieldValue` exist;
the batch `setIssueFieldValue` path here is the verified one.

## Project membership

Discover projects at call time; do not remember a project number across runs.
Use `gh project item-add <number> --owner <owner> --url <issue-or-pr-url>`, or
`addProjectV2ItemById` with project and content node ids. URLs work across repos;
`--owner` accepts an org, user, or `@me` for the authenticated user's projects.

Report membership outcomes distinctly:

- No board found: `project: none`; there is nothing to link.
- One board found, already linked: `on_project: true`.
- One board found, missing membership: add automatically, then report
  `on_project: true`. This case needs no further confirmation round.
- Several boards found: list them all and ask the caller which board(s), unless
  the caller already authorized all of them. Do not choose silently.

Adding is idempotent: re-adding returns the existing item id. Link explicitly
even if an auto-add workflow may have run; its scope is not visible here.
`projectV2.repositories` lists linked repositories, not all repositories with
items, so it does not establish board coverage. PRs need their own membership
call just as issues do.

## Board item fields

Read [field mechanics](references/fields.md) for the full discovery query before
setting board fields. IDs are per-board; discover fields and options this run.
`fields` is a union: the common fragment supplies identity, while type-specific
fragments must supply `options`, `multiSelectOptions`, or iteration
`configuration`. Missing fragments leave discovery incomplete even when field
names return. Compare node count with `totalCount`; raise the cap or paginate
until complete before treating the result as the board's field set.

Classify each field before writing:

- **Board-native:** `Status`, `Size`, `Estimate`, iteration, and other fields
  defined on the board. Use `gh project item-edit` or
  `updateProjectV2ItemFieldValue`, with the **item id**, not the issue number.
- **Mirrored org Issue Field:** its name also appears in this run's org field
  list. Use `setIssueFieldValue` on the issue, including date mirrors. A mirrored
  single-select's `options: []` corroborates this; read its real options from
  org discovery. Do not report missing options or ask someone to reconfigure it.
- **Built-in projection:** Title, Assignees, Labels, Milestone, Repository,
  Linked pull requests, Reviewers, Parent issue, Sub-issues progress, Created,
  Updated, or Closed. These are not writable board fields; change the underlying
  issue/PR property or relationship where supported.

The documented live board accepted native Size/Estimate writes and rejected
Priority/Start date/Target date mirrors. That rejection calls for the issue
mutation, not unchanged retries or a permissions escalation.

Board values have six typed forms: text, number, date, single-select option id,
multi-select option ids, and iteration id. Multi-select takes an array and
replaces the whole selection; it has no CLI flag in `gh` 2.96.0. Use
`clearProjectV2ItemFieldValue` to clear a board value.

Find iteration fields by `dataType == "ITERATION"`, not a name such as “Sprint”.
Select ids from `configuration.iterations`, not `completedIterations`, and never
resolve by title alone because titles recur. The default current iteration is
derived from the window containing today (`startDate` plus `duration` in days);
cite those dates. Choosing the next iteration requires the caller's policy.
Missing `configuration` means rerun discovery. A selected but empty `iterations`
list means report that field unset by name because no iterations are configured.
The reference preserves the 2026-07-27 evidence and its untested limits.

Every board-native field uses the sourced / marked-proposal / unset rule.
Status and estimates are not automatic values merely because membership is
automatic. Without a policy, a traceable Status guess may be proposed against
the board's real options; it must not be presented as an established policy.

## Workflow policies

A written, agreed workflow policy can source a field value. For example, a
Status policy maps lifecycle moments to option **names** that the calling
workflow resolves against this run's discovery. All three conditions apply:

1. The policy exists and was agreed to; column names alone do not establish it.
2. Its option name still resolves. Report a stale key instead of substituting a
   nearby option.
3. The caller shows the transition, cited to the policy, on its confirmation
   surface before writing it.

A derived transition needs no separate question per field, but the policy only
covers the board and lifecycle moments it names. This skill does not decide on
its own to move cards. Without a valid policy, return to the marked-proposal or
unset outcome; without caller confirmation, leave the proposed value unset.

## Pull requests and issue links

Check `.github/pull_request_template.md` before creating a PR. Prefer
`gh pr create --base --head --title --body --draft`; `createPullRequest` is the
API fallback. Include the closing keyword in the body, or tell the caller why
no linked-issue relationship exists. No CLI flag or API parameter creates that
keyword relationship directly.

Keep these mechanisms separate: a closing keyword creates the linked-issue
relationship and controls closing; a plain mention provides only a backlink;
board membership is a separate add; `Linked pull requests` is a read-only
projection; stack membership is another explicit link.

Read `gh repo view <owner>/<repo> --json defaultBranchRef` before trusting a
keyword. Do not infer the default branch from the name `main`. Verify the link
with `gh pr view N --json closingIssuesReferences` (observed in `gh` 2.99.0),
not by looking at the rendered body.

- Default-branch base: use `Closes owner/repo#N`, always fully qualified, even
  in the same repo. Cross-repo closing also needs push access to the issue repo.
- Other base, ordinary PR: no linked-issue relationship is created and the issue
  will not auto-close. Keep a plain reference, post the issue comment, and
  report it unlinked. Manual sidebar linking is same-repo only, so there is no
  cross-repo fallback for that base.
- Stack layer: apply the base rule to the **link**, but report the **close**
  separately. The keyword fires when its carrying layer lands in a stack merge,
  even without a link. Put it on the layer whose merge completes the issue's
  work; a bottom-layer keyword closes the issue on the first partial merge.

The live stack test found that retargeting to the default branch did not create
a link, nor did re-saving the same body. Editing the body to different text did.
After retargeting, verify `closingIssuesReferences` again and distinguish the
current link from what the keyword will close. A stack trunk is not necessarily
the default branch when `init` or `link` used `--base`.

After creation, add the PR to its discovered board(s) separately. If its base is
another open PR's head, follow the new-layer linking rule below. Each layer
needs its own board membership and reference or keyword.

## Stacked pull requests

Before merging, closing, or retargeting **any PR**, read membership with GraphQL
`pullRequest.stack` / `stackEntry`, or REST `GET /repos/{o}/{r}/pulls/{n}`'s
`.stack`. `gh pr view --json` has no stack field. Null means it is not a layer.
Compare stack entry count with `totalCount` and complete a truncated read.

Read [stack mechanics](references/stacks.md) before stack operations. It contains
commands, membership queries, error meanings, and the 2026-09-02 runtime versus
documentation-only evidence. This public-preview feature requires current help.
GraphQL is read-only for stacks and has no stack mutation; REST writes require
`X-GitHub-Api-Version: 2026-03-10`. Check/install `github/gh-stack` as a local
extension or use those REST routes.

Stacks are strictly linear within one repository, never across forks or repos.
Their protections/checks are evaluated against the trunk; merging lands a
contiguous run from the bottom atomically. A PR's base branch alone does not
create membership: without it, checks target that layer branch and a merge
lands on that branch. Cross-repo PR dependencies belong in issue relationships.

For a PR you create on another open PR's head, create its complete body first,
then link bottom-to-top in the same workflow, or report that it is a chain-shaped
PR in no stack. Prefer `gh pr create` per layer then `gh stack link`. This also
covers an unstacked lower PR: linking lower and upper preserves the lower PR's
merge scope. For an existing chain-shaped PR you discover, propose the link;
do not silently change the upper PR owner's merge scope or checks.

Before a stack merge, name **every PR that will land**, including every unmerged
layer below the target, and obtain the caller's merge authorization for that
scope. Use an explicit target and method with `gh stack merge N --yes --<method>`,
or `merge-async` and its status poll. `gh pr merge`, including `--auto`, fails on
all layers. A bare non-interactive stack merge can land the entire stack without
a prompt, and an omitted method reuses the last one. A trunk merge queue can
override the method and land the run in separate groups.

Closing a layer leaves it in the stack and blocks layers above it, even if their
merge state says CLEAN; explain this before closing. Reopening restores it.
`gh pr edit --base` is refused for a layer; restructure through unstack/link.
Each merging layer must meet trunk rules and contain the tip below it. Use
`gh stack sync` for an outdated chain; a draft in the run prevents the merge.

Merged layers retain their positions and stack size. Find the lowest unmerged
layer by `state: OPEN`, not position 1. GitHub retargets/rebases the next layer
server-side, so run `gh stack sync` before any subsequent push. Prefer sync over
`gh pr update-branch` for chain repair.

Use non-interactive forms: `view --json`, `submit --auto`, `merge <target> --yes`,
`init <branches>`, `add <branch>`, and `checkout <target>`. Avoid `modify`, `switch`,
and bare interactive forms. Set `--remote <name>` for push/submit/sync/rebase/link
unless `remote.pushDefault` is set. Branch on exit status, not stderr text.
Exit 2 from view means no local stack tracking; checkout imports a linked stack.
Exit 9 means stacks are unavailable, and an empty REST list means none exist.
These are complete answers; other exit meanings are in the reference.

## Dependencies and search

Relationship flags accept issue URLs across repos: create with `--blocked-by`,
`--blocking`, or `--parent`; edit with `--add-blocked-by` or `--add-sub-issue`.
Propose dependencies at create time from parent, siblings, and the bounded scan,
in both directions, whether or not the caller asked about dependencies.

A search hit, shared label/milestone, or similar title retrieves a candidate; it
does not establish an edge. Require a verbatim quote from the candidate or
conversation stating direction, such as “blocked on X”, “after X lands”, or
“unblocks X”. Without that evidence, report a candidate or drop it. Do not guess
structural relationships. For `--blocking`, name whose issue receives the edge.
Omit empty relationship flags entirely.

Always report the scan outcome, including no matches: number of open issues
scanned, query count, first-page scope, and what was ruled out or supported.
Keep scans bounded with a stated query count and limit, page one only; no cursor
or second page. This scan limit differs from completing field/stack discovery.

Use `gh issue list --search` for one repo and `gh search issues --owner` across
repos. Prefer GraphQL search or the parent's `subIssues` when each result also
needs `blockedBy` / `blocking` edges. Read [search mechanics](references/search.md)
for query shapes, qualifiers, costs, and their unverified-in-this-repo label;
confirm edge selections on first use.

For a current blocked check, read `dependencies/blocked_by` or GraphQL
`blockedBy`; `issue_dependencies_summary` can lag about one second after writes
and is suitable only for a cold read. The field is `blockedBy`, not
`blockedByIssues`. Search indexing is also asynchronous: an empty search never
proves an issue absent; read a known issue by number, especially after creation.

## Routine operations

Use the [command reference](references/commands.md) for ordinary issue, PR,
repo, file, and user operations. Label flags are additive; REST/GraphQL label
arrays replace the set, so read current labels before using that path. Remote
file overwrites require the current blob `sha`. Line-level review comments use
`addPullRequestReviewThread`; CLI review bodies are top-level only.

For screenshots requested in issue/PR bodies, use `--attach` if current help
lists it, otherwise the user-attachments workflow in [screenshots.md](screenshots.md).
Repository paths and raw links do not render for private repos.

## Research delegation

Multi-call research may use parallel read-only agents. Read the
[research protocol](references/research.md) before dispatch; its brief, return
envelope, and gates are required. Agents propose; writes execute in the calling
workflow, in the main conversation, visible to the user. Do not treat an agent's
report as a receipt or claim its GitHub access is structurally read-only.

Run all return gates before rendering or using any payload. Required checks
cover valid JSON, read-only calls, reference existence, current field discovery,
capability claims, and reported coverage. Any `asks[]` blocks dependent writes.
Discard a payload that attempted a write and tell the user; do not retry silently.
Never scrape malformed JSON from prose. Retry a missing/malformed return once
with a narrower brief, then use the caller's inline procedure and report degraded
coverage. Report unresolved coverage and agent disagreements explicitly.
