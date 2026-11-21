# Dependencies, search, and rollups

Read for issue preflight dependency scans, sibling reads, cross-repo search, or
status rollups. The evidence and reporting requirements are in
[SKILL.md](../SKILL.md).

## Relationship writes and current reads

Relationship and sub-issue flags accept issue URLs across repos:

```bash
gh issue edit N --add-blocked-by https://github.com/<owner>/<other-repo>/issues/M
gh issue create --blocked-by <numbers-or-URLs> --blocking <numbers-or-URLs>
gh issue edit N --add-sub-issue https://github.com/<owner>/<other-repo>/issues/M
gh issue create --parent <number-or-URL> --type Task
```

Use a quoted statement of direction from the candidate or conversation to support
an edge; matching labels, milestones, or titles only finds candidates. Name the
other issue that receives a `--blocking` edge. Omit empty relationship flags.

For read-after-write checks:

```bash
gh api /repos/{o}/{r}/issues/{n}/dependencies/blocked_by
```

That list is correct immediately. `.issue_dependencies_summary` in the issue
response can report 0 for about one second after adding a blocker. Use the list
or GraphQL `blockedBy` for current blocked checks; the summary is suitable for a
cold read. The GraphQL fields are `blockedBy` / `blocking`, not `blockedByIssues`.
A wrong name returns `errors` without `data`; inspect/introspect before concluding
that relationships are unsupported.

## Project status rollup

The documented comparison was one GraphQL query at `cost: 1`, versus 18 REST
requests for the same 16 issues:

```graphql
organization(login: "<owner>") { projectV2(number: <project-number>) { items(first: 30) { nodes {
  content { ... on Issue {
    number  repository { nameWithOwner }  state
    blockedBy(first: 5) { nodes { number repository { nameWithOwner } state } }
  } } } } } }
```

For a personal project, swap the root to `user(login: "<owner>")`; the inner
selection is unchanged. This swap is supported for Projects, unlike Issue Fields.

## Search and sibling queries

**Evidence limit:** the search and sibling queries in this section are
**unverified in this repo** and have not been run against a live org. Confirm
edge selections on first use before promoting this note. This differs from the
verified Issue Field mutations.

`gh issue list --search` searches only `-R` or the current remote's repository.
For owner-wide issue fields without edges, use `gh search issues --owner`.
Prefer GraphQL `search(type: ISSUE)` when you also need each hit's edges; it
avoids a separate REST read per hit:

```graphql
search(query: "org:<owner> is:issue is:open <terms>", type: ISSUE, first: 20) {
  issueCount
  nodes { ... on Issue {
    number  title  state  url  updatedAt  repository { nameWithOwner }
    blockedBy(first: 5) { nodes { number repository { nameWithOwner } state } }
    blocking(first: 5)  { nodes { number repository { nameWithOwner } state } }
  } } }
```

CLI fallback, or the route when only titles/issue fields are needed:

```bash
gh search issues "<terms>" --owner <owner> --state open --limit 20 \
  --sort updated --order desc \
  --json number,title,state,repository,url,updatedAt
```

Supported query qualifiers include `org:`, `repo:`, `is:issue`, `is:open`,
`is:closed`, `in:title`, `in:body`, `label:`, `milestone:`, `author:`, `assignee:`,
`no:assignee`, `updated:>=YYYY-MM-DD`, and `closed:>=YYYY-MM-DD`.
For `gh search`, use dedicated `--sort` / `--order` flags; `sort:` in the string
is treated as a search term. `gh search issues` excludes PRs unless
`--include-prs` is passed; `gh search prs` is the separate PR surface.

The parent's `subIssues` can return siblings and declared edges in one query.
`gh issue view N --json parent,subIssues` returns titles/states but no edges:

```graphql
repository(owner: "<owner>", name: "<repo>") { issue(number: <n>) {
  number  title  state
  blockedBy(first: 5) { nodes { number repository { nameWithOwner } state title } }
  subIssues(first: <cap>) { totalCount nodes {
    number  title  state  url  repository { nameWithOwner }
    blockedBy(first: 5) { nodes { number repository { nameWithOwner } state title } }
    blocking(first: 5)  { nodes { number repository { nameWithOwner } state title } }
  } } } }
```

## Bounds, costs, and indexing

A GraphQL search costs 1 point. REST search has its own 30 requests/minute limit,
separate from the 5,000/hour core limit; repeated searches can hit that ceiling
while ordinary issue reads do not. GraphQL search allows `first` up to 100 and
at most 1,000 reachable results. Those API limits do not expand this skill's scan:
state the query count and limit, and use page one only, without a cursor or a
second page.

The create-time scan runs even for fabricated/test issues or when there is no
parent and no siblings. Report its scope and result even when nothing qualifies,
for example: “Scanned N open issues across Q queries, first page each; none
contained a quoted dependency direction.” Distinguish a candidate, an evidenced
proposal, and an edge actually written.

Search indexing is asynchronous. Immediately after creation, a title search may
omit the new issue. An empty search never proves absence; read the issue by number:

```bash
gh api /repos/{o}/{r}/issues/{n}
```
