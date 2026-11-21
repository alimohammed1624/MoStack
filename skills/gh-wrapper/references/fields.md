# Field and project API mechanics

Read for owner capability checks, Issue Field discovery/writes, project discovery,
or board item field operations. The workflow and confirmation rules remain in
[SKILL.md](../SKILL.md). Contents: owner and org fields; membership; board field
discovery/classification; writes; iterations.

## Owner and Issue Field discovery

Probe owner type and repository capabilities:

```graphql
query { repositoryOwner(login: "<owner>") { __typename }
  repository(owner: "<owner>", name: "<repo>") {
    viewerCanSeeIssueFields
    issueTypes(first: 1) { totalCount } } }
```

Combine the `organization(login:) { issueFields ... }` selection below with that
query to avoid a separate request. `__typename` is `Organization` or `User`.
On a personal account the organization node is null; `User.issueFields` does
not exist. Prefer `viewerCanSeeIssueFields` to owner type alone for permissions.

For read-only discovery, REST is simpler:

```bash
gh api /orgs/<owner>/issue-fields
gh api /orgs/<owner>/issue-types
```

The Issue Field endpoint returns names, types, and single-select options;
POST/PATCH/DELETE also exist for definitions. Types can be single-select, date,
number, or text. Neither endpoint has a personal-account equivalent. Discovery
does not expose a `required` flag.

For GraphQL field and option ids, `issueFields` is a union and needs fragments.
This example selects single-select and date fields; account for the full field
set returned by discovery, including other types in use:

```graphql
query { organization(login: "<owner>") { issueFields(first: 10) { nodes {
  ... on IssueFieldSingleSelect { id name options { id name } }
  ... on IssueFieldDate { id name }
} } } }
```

## Issue Field writes

As of `gh` 2.96.0, org Issue Field values have no CLI flag. `gh project item-edit`
is for board items. Get the issue node id and use the verified batch mutation:

```graphql
mutation { setIssueFieldValue(input: {
  issueId: "I_..."
  issueFields: [
    { fieldId: "IFSS_...", singleSelectOptionId: "IFSSO_..." },
    { fieldId: "IFD_...",  dateValue: "YYYY-MM-DD" }
  ]}) { issue { id } } }
```

Clear with `{ fieldId: "IFSS_...", delete: true }`. Verify with:

```bash
gh api /repos/{o}/{r}/issues/{n} --jq '.issue_field_values'
```

Introspect `IssueFieldCreateOrUpdateInput` when another input shape is needed.
Both `updateIssueFieldValue` and `setIssueFieldValue` exist; the latter is the
verified multi-field path. An error naming the former does not invalidate it.

Org ids (`IFSS_` / `IFD_` / `IFSSO_`) and Projects v2 ids (`PVTF_` / `PVTSSF_` /
`PVTI_`) are separate spaces. A board mirror written through the board mutation
returns “Issue field values cannot be updated using the
updateProjectV2ItemFieldValue mutation.” Use the org id and `setIssueFieldValue`.

A mirrored single-select returns `options: []` from the board even when real
options exist. Read them from org `issueFields` or REST; the empty board list
identifies the mirror, not a configuration failure. Board-native single-selects
return their real options. Never replace the intended field with another field
that happens to accept a write.

## Project discovery and membership

Discover at call time:

```bash
gh api graphql -f query='{ organization(login:"<owner>"){
  projectsV2(first:20){ nodes{ number title id } } } }'
```

For a personal account, replace `organization(login:)` with `user(login:)`.
Projects support both roots, unlike Issue Fields. Use the discovered number and
owner:

```bash
gh project item-add <number> --owner <owner> --url <issue-or-pr-url>
```

The API alternative is `addProjectV2ItemById(input:{projectId, contentId})`.
`contentId` is the issue/PR node id, not its number. Re-adding returns the existing
item id and does not error. Save that id for field writes. Membership creates no
custom field values. `--owner` accepts an org login, user login, or `@me`.
`projectV2.repositories` describes linked repositories, not repositories with
items; it cannot establish coverage.

## Board field discovery and classification

The complete type-specific selection is:

```bash
gh api graphql -f query='{ organization(login:"<owner>"){ projectV2(number:<n>){
  fields(first:50){ totalCount nodes{
    ... on ProjectV2FieldCommon       { id name dataType }
    ... on ProjectV2SingleSelectField { id name dataType options { id name } }
    ... on ProjectV2MultiSelectField  { id name dataType multiSelectOptions { id name } }
    ... on ProjectV2IterationField    { id name dataType configuration {
        duration startDay
        iterations          { id title startDate duration }
        completedIterations { id title startDate duration } } }
  } } } } }'
```

Use the user root for a personal board. Compare nodes returned to `totalCount`;
raise the cap or page until discovery is complete. Field ids belong to this
board, and option ids must come from this run.

`ProjectV2FieldCommon` has four implementors: `ProjectV2Field`,
`ProjectV2SingleSelectField`, `ProjectV2MultiSelectField`, and
`ProjectV2IterationField`. It returns identity for all four, so a missing
concrete fragment can leave an apparently discovered field without its values.
Select `options` for single-selects, `multiSelectOptions` for multi-selects, and
`configuration` for iterations. `options` on a multi-select is `undefinedField`,
not an empty list; such responses have `errors` and no `data`.

Cross-reference board names against this run's org Issue Fields before writing.
A matching name is a mirror, including dates that have no empty-options signal.
Built-in projections (Title, Assignees, Labels, Milestone, Repository, Linked
pull requests, Reviewers, Parent issue, Sub-issues progress, Created, Updated,
Closed) are not writable through the board mutation. Only board-native fields
use the item write path.

Observed on a live org board: native `Size` (single-select with real options)
and `Estimate` (number) accepted `updateProjectV2ItemFieldValue`. `Priority`,
`Start date`, and `Target date`, also org fields, rejected it with the mirror
error quoted above. This identifies a mechanism mismatch, not permissions.

## Board item writes

The CLI takes an item id:

```bash
gh project item-edit --id <item-id> --project-id <project-id> \
  --field-id <field-id> --single-select-option-id <option-id>
```

Other flags are `--text`, `--number`, `--date`, and `--iteration-id`. The verified
API route is `updateProjectV2ItemFieldValue(input:{projectId, itemId, fieldId,
value:{...}})`. `ProjectV2FieldValue` accepts exactly six typed values:

```text
{text:"…"}
{number:3}
{date:"YYYY-MM-DD"}
{singleSelectOptionId:"…"}
{multiSelectOptionIds:["…","…"]}
{iterationId:"…"}
```

A wrong key is a validation error. Multi-select requires an array even for one
option and replaces the whole selection. It has no CLI flag in `gh` 2.96.0;
the other five types do. Clear with
`clearProjectV2ItemFieldValue(input:{projectId, itemId, fieldId})`.

Use the item id from the add, or query it back:

```bash
gh api graphql -f query='{ organization(login:"<owner>"){ projectV2(number:<n>){
  items(first:50){ nodes{ id content{ ... on Issue { number repository{ name } } } } } } }'
```

## Iterations

Verified 2026-07-27: schema introspection confirmed these selections and a live
`msa1624` board #3 returned an `ITERATION` field named `Sprint`.
`ProjectV2FieldValue` accepts `iterationId`; `gh` 2.96.0 supports `--iteration-id`.
A populated `completedIterations` list was **not observed live**; that warning
is based on the schema.

Identify the field by `dataType: ITERATION`; its name may be Sprint, Cycle,
Iteration, or anything else. It is board-native and uses the ordinary
sourced / marked-proposal / unset rule.

`configuration.iterations` contains current and future iterations: choose an id
from this list. `completedIterations` contains closed windows; those ids are
schema-legal but file work into a completed sprint. Do not choose from that list
or resolve by title alone, since titles recur across cycles.

The current iteration is derived from the window containing today. `duration`
is days, not weeks. Cite `startDate` and `duration` as the source; choosing the
next iteration instead requires the caller's agreed policy and the same
confirmation conditions as any other policy-derived value.

The live board example:

```text
Sprint (ITERATION), duration 14, startDay 1
  iterations: Sprint 1  2026-07-27 → 08-09
              Sprint 2  2026-08-10 → 08-23
              Sprint 3  2026-08-24 → 09-06
  completedIterations: []

For 2026-07-27, propose Sprint 1 as derived from its 07-27 → 08-09 window.
Write its id (edcbc333…), not its title.
```

If `configuration` was omitted, rerun with the full fragment. If it was selected
and `iterations: []`, report the field unset by name: the board has no iterations
configured. That is a complete answer, not grounds to try another mutation.
