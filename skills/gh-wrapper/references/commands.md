# Routine command reference

Read for the operation at hand. Check current `--help` and apply the workflow,
confirmation, and verification rules in [SKILL.md](../SKILL.md). Creating issues
always requires its six-step preflight. PR merge, close, and retarget always
require its stack-membership check.

## Issues

| Task | Command | Notes |
|---|---|---|
| List | `gh issue list` | Open by default; `--state all` for all |
| Filter label | `gh issue list --label bug` | |
| Search one repo | `gh issue list --search "..."` | Cross-repo: `gh search issues --owner`; see [search mechanics](search.md) for edges and bounds |
| Read | `gh issue view N` | `--comments` for thread, `--json` for fields |
| Create | `gh issue create -t -b` | Run all six preflight lookups before writing; use `--assignee "@me"` unless another person was named; report dependency scan outcome even when empty |
| Labels | `gh issue edit N --add-label X` | Additive; REST/GraphQL `labels` arrays replace, so read current labels first |
| Assign | `gh issue edit N --add-assignee u` | `@me` and `@copilot` accepted; named person replaces create-time `@me`; reassigning an owned issue is a handoff |
| Type | `gh issue edit N --type X` | Validate `/orgs/<owner>/issue-types`; org-only |
| Milestone | `gh issue edit N --milestone "name"` | `--remove-milestone` clears |
| Close | `gh issue close N` | `--reason "not planned"` maps to `state_reason: not_planned` |
| Reopen | `gh issue reopen N` | |
| Comment | `gh issue comment N -b` | |
| Sub-issue add / move / remove | `gh issue edit N --add-sub-issue` / `--parent` / `--remove-sub-issue` | Number or URL; URLs cross repos |
| Read hierarchy | `gh issue view N --json parent,subIssues` | Reorder has no CLI flag; use `reprioritizeSubIssue` |
| Siblings with edges | GraphQL parent's `subIssues` | CLI returns titles/states without edges; [search mechanics](search.md) has the query |

## Pull requests

| Task | Command | Notes |
|---|---|---|
| List | `gh pr list` | `--search` filters; `gh search prs` crosses repos |
| Read | `gh pr view N` | `--json` includes structured fields such as `files` |
| Diff | `gh pr diff N` | |
| Checks | `gh pr checks N` | |
| Create | `gh pr create --base --head --title --body` | Check template and issue-link rules; link a new upper layer into its stack in the same workflow |
| Membership | GraphQL `pullRequest { stack stackEntry }` | No `gh pr view --json` stack field; read before merge/close/retarget |
| Verify issue link | `gh pr view N --json closingIssuesReferences` | A rendered keyword does not prove a link |
| Request review | `gh pr edit N --add-reviewer u` | |
| Retitle / retarget | `gh pr edit N --title` / `--base` | Layer retarget is refused; restructure via unstack/link |
| Ready | `gh pr ready N` | |
| Review | `gh pr review N --approve` / `--request-changes` | Top-level body only |
| Line-level review comment | GraphQL `addPullRequestReviewThread` | No CLI flag |
| Comment | `gh pr comment N -b` | Uses the issue endpoint |
| Merge ordinary PR | `gh pr merge N --squash` | Caller authorization required; fails for stack layers |
| Merge stack layer | `gh stack merge N --yes --squash` or `merge-async` | Name all landing layers before confirmation; `--auto` unsupported |
| Close | `gh pr close N` | Closing a stack layer blocks every layer above; explain first |
| Update branch | `gh pr update-branch N` | Prefer `gh stack sync` for a stack; it can repair the full descendant chain |
| Stack read / link / extend / dissolve | `gh stack view --json` / `link` / `link <n>` / `unstack <n>` | [Stack mechanics](stacks.md); API writes are REST, never GraphQL |
| Requested screenshots | `--attach` if current help supports it, otherwise user-attachments upload | [Screenshots](../screenshots.md), also for issues; private repo paths/raw links do not render |

## Repositories, files, search, and users

| Task | Command | Notes |
|---|---|---|
| Create repo | `gh repo create name --private` | |
| Fork | `gh repo fork` | |
| Branches | `gh api repos/<o>/<r>/branches` | |
| Commits | `gh api repos/<o>/<r>/commits` | Or local `git log` |
| Read file at ref | `gh api repos/<o>/<r>/contents/<path>?ref=<ref>` | |
| Write remote file | `gh api --method PUT repos/<o>/<r>/contents/<path>` | Overwrite needs blob `sha`; obtain with `git rev-parse <branch>:<path>` |
| Releases | `gh release list` / `gh release view <tag>` | |
| Search | `gh search issues/prs/repos/code/commits` | Cross-repo under `--owner`; issue scope/cost/index details in [search mechanics](search.md) |
| Identity | `gh api user --jq .login` | |
| Team members | `gh api /orgs/<org>/teams/<slug>/members` | Org-only |
| Collaborators | `gh api /repos/<o>/<r>/collaborators` | Personal-account alternative to teams |

`gh workflow`, `gh secret`, `gh gist`, `gh alias`, `gh extension`, `gh run`,
rulesets, milestones, and webhooks use the ordinary CLI route. Plain local `git`
needs no routing through this skill.
