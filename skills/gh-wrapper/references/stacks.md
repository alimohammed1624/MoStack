# Stack mechanics and evidence

Read before stack operations, including a merge, close, or retarget of a stack
layer. Follow [SKILL.md](../SKILL.md) for authorization, merge scope, new versus
existing PR linking, and issue-link behavior. Contents: evidence; surfaces;
membership; merge and close behavior; creation; non-interactive execution.

## Evidence and scope

Verified live **2026-09-02** on `msa1624/stack-lab`, an org test repo with three
stacks and nine PRs, using `gh` **2.99.0** and `gh-stack` **v0.1.0**. Run or read
back: the quoted `gh stack` / `gh pr` commands and errors, GraphQL and REST read
shapes, `merge-async` and its poll, and closing-keyword link/close behavior.

**Documentation-only, not run:** REST stack writes (`POST /stacks`, `/add`,
`/unstack`; their `gh stack` equivalents were run), the stack map, protections
and checks evaluated against the trunk, GitHub Desktop support, the queued or
auto-merge exception on unstack, `link` correcting a wrong base, a trunk merge
queue, and exit 9. The feature is in public preview; check current `--help`.

The documented model is a strictly linear, same-repository chain with its own
repo-scoped stack number. Each layer targets the head branch below; the bottom
targets the trunk. The stack map appears on each layer, checks and protections
are evaluated against the trunk, and a contiguous bottom-up run merges atomically.
Stacks cannot cross forks or repositories and are not supported in GitHub Desktop.

A base branch alone does not create membership. Without a stack object, a PR
based on another PR's head has no stack map, is checked against that layer branch,
and merges onto it.

Closing-keyword findings were each observed once on the live test:

- The link (`closingIssuesReferences`, sidebar, and board projection) existed
  only for a default-branch-based layer. The tested trunk was the default branch;
  `--base` on init/link can choose another trunk, where even position 1 has no link.
- Server retargeting to the trunk created no link. Re-saving identical body text
  did not help; editing to different text created the link.
- A keyword on position 3, whose base was still another layer and which had no
  link, closed its issue when that layer merged. The PR was recorded as closer.
- Closing happens when the carrying layer merges. A keyword on the bottom layer
  closes the issue on a partial merge even if higher feature layers remain.

These observations distinguish linking from closing; report and verify them
separately rather than inferring one from the other.

## CLI and API surfaces

`gh stack` is an extension. Check `gh extension list`; install locally with
`gh extension install github/gh-stack`, or use REST. Installation writes nothing
to GitHub. GraphQL exposes `PullRequest.stack` and `stackEntry` read-only.
Schema introspection on 2026-09-02 found zero `Mutation` fields matching `stack`.

Every REST call below must include:

```text
-H "X-GitHub-Api-Version: 2026-03-10"
```

Keep the header even when a call answers without it.

| Operation | CLI | API |
|---|---|---|
| List stacks | No list command; bare checkout is a picker | `GET /repos/{o}/{r}/stacks`; `?pull_request=N` filters |
| Read stack | `gh stack view --json` uses local tracking; checkout first | `GET /repos/{o}/{r}/stacks/{stack_number}` |
| Check PR membership | No `gh pr view --json` stack field | GraphQL below, or `GET /repos/{o}/{r}/pulls/{n}` → `.stack` |
| Create | `gh stack link <bottom> … <top>` or `submit --auto` | `POST /repos/{o}/{r}/stacks`, `{"pull_requests":[…]}` bottom-to-top (docs-only) |
| Extend | `gh stack link <stack-number> <pr-or-branch> …` | `POST /repos/{o}/{r}/stacks/{n}/add` (docs-only) |
| Dissolve | `gh stack unstack <stack-number>` | `POST /repos/{o}/{r}/stacks/{n}/unstack` (docs-only; queued/auto-merge PRs remain stacked per docs) |
| Merge | `gh stack merge <pr-or-stack-number> --yes --<method>` | `PUT /repos/{o}/{r}/pulls/{n}/merge-async -f merge_method=squash` |

The merge response is `{"status":"pending","details":{"uuid":…}}`. Poll
`GET /repos/{o}/{r}/pulls/{n}/merge-async/{uuid}` until status is `merged`,
`failed`, or `enqueued`; GET on the bare merge-async path returns 404. Merging N
includes every unmerged layer below N.

## Membership read

```graphql
repository(owner: "<o>", name: "<r>") { pullRequest(number: <n>) {
  baseRefName
  stackEntry { position }
  stack { number size baseRefName
    entries(first: 50) { totalCount nodes { position
      pullRequest { number state baseRefName headRefName } } } } } }
```

`stack: null` means the PR is not a layer. Position 1 is nearest the trunk;
`stack.baseRefName` identifies the trunk. Compare `entries.totalCount` to returned
nodes and complete a truncated read.

Merged layers stay in the stack: their state is MERGED, positions do not renumber,
and size is unchanged. Find the lowest unmerged layer by OPEN state, not position
1. GitHub retargets/rebases the next layer to the trunk server-side. Run
`gh stack sync` before a subsequent push; otherwise the remote rebase causes a
non-fast-forward rejection.

## Merge, close, and retarget behavior

- `gh pr merge N`, including `--auto`, fails on every stack layer, even position
  1, with exit 1: “This pull request is part of a stack and must be merged using
  the asynchronous merge REST API.” Use the stack merge route.
- `gh pr close N` leaves a CLOSED entry. The layer above can still show
  `mergeStateStatus: CLEAN`, but merging above it exits 5: “cannot be merged yet:
  #N below it is closed.” Reopening restores the layer.
- `gh pr edit N --base` is refused: “Cannot change the base branch because the
  pull request is part of a stack.” Restructure through unstack and link.

Before merging, name every landing PR and obtain authorization for that scope.
Pass an explicit PR or stack target and method: a bare non-interactive merge
merges the whole current stack without prompting, and an omitted method reuses
the previous one. A trunk merge queue can override that flag and land separate
groups (documentation-only finding).

Every layer must satisfy the trunk rules and contain the tip of the layer below.
If a lower branch advanced without rebasing higher ones, the atomic merge exits
4: “Stack needs to be rebased: PR #9's branch is not a linear descendant of PR
#10's branch. Stack merges are atomic, so nothing was merged.” `gh stack sync`
repairs this. A merge commit from `gh pr update-branch` satisfied the descendant
check and a squash merge succeeded with it: “linear” here means descends from,
not absence of merge commits. Prefer sync for chain repair; update-branch from
the top cannot repair the whole chain.

A draft in the run stops the attempt, exit 2: “nothing to merge: pull request #10
is a draft.” Exit meanings depend on the command; view's exit 2 differs below.

## Creating and changing membership

Prefer full `gh pr create --base <lower-head> ...` bodies followed by one
`gh stack link <bottom> … <top>`. Link adopts existing open PRs and never removes
a member. It corrects a wrong base according to docs (not exercised live).

The other path, `gh stack submit --auto`, pushes all branches, creates a draft
per branch (`--open` makes them ready), and links the stack. It generates titles
and bodies: one commit uses its subject/body; multiple commits use a humanized
branch name and empty body. Every body gets a `gh-stack` footer. There is no
title/body flag, so edit each PR afterwards. A branch with no commits gets a
warning and no PR while the command still exits 0.

Arguments to `link` and `init` are bottom-to-top. `gh stack init a b c` leaves
the checkout on `c`, the top; use `gh stack bottom` before committing on the
bottom. The REST create equivalent takes PR numbers in the same order.

`gh stack unstack <n>` removes grouping on GitHub and locally, leaving PRs open;
a subsequent link creates a new stack number. The documented exception for
queued/auto-merge PRs is noted above. Existing chain-shaped PRs require a proposed
link to their caller; the automatic linking rule applies to a newly authorized
upper PR, not an existing owner's PR.

## Non-interactive execution and exits

Commands branch on stdout being a TTY; a PTY can start a blocking full-screen UI.
Use explicit forms:

```text
view --json
submit --auto
merge <target> --yes --<method>
init <branch>…
add <branch>
checkout <target>
```

Do not run `modify`, `switch`, or bare view/submit/merge/init/add/checkout.
Use `--remote <name>` on push, submit, sync, rebase, and link unless
`remote.pushDefault` is set.

`view --json` writes JSON to stdout and status to stderr. Use exit codes, not
stderr text:

- **2, view:** not in a locally tracked stack. `link` does not create tracking;
  `checkout <stack-number>` imports it. A merge's draft refusal also uses 2.
- **9:** stacked PRs unavailable on this repository (docs-only). This is a
  complete capability answer; an empty REST `/stacks` list means no stacks.
- **3:** rebase conflict.
- **4:** GitHub refused the request, including the atomic merge refusal above.
- **5:** a precondition checked by `gh stack`, such as a closed lower layer.
- **6:** the branch is in several stacks.

The last four are operational conditions, not evidence of a missing capability.
