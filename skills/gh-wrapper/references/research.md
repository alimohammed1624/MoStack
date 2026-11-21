# Read-only research protocol

Read before delegating multi-call GitHub research or accepting its result.
Parallel independent research agents may be dispatched together. This protocol
specifies the brief, return envelope, and caller checks; it does not define a
calling workflow's policy. Follow the host's delegation mechanism.

## Agent capability and responsibility

Where available, use the built-in `Explore` type. It has no Write, Edit, or
NotebookEdit tools, so those file-edit tools are structurally unavailable. It
does have Bash, which can reach GitHub writes. GitHub read-only access is an
instruction checked by the return gate, not a structural guarantee. A custom
agent definition's `tools:` allowlist can restrict access; a skill cannot set
that allowlist per invocation.

The agent proposes. Every write executes in the calling workflow, in the main
conversation and visible to the user. An agent response is not a write receipt.
A caller's stronger requirements for verified data still apply.

## Required brief

Include the goal, scope, bounded queries, and these rules in every brief:

```text
You are read-only. Do not call setIssueFieldValue, addProjectV2ItemById,
updateProjectV2ItemFieldValue, any GraphQL mutation, or a GitHub write command,
including gh issue edit/create/close/comment, gh project item-add/item-edit,
or gh stack init/add/submit/link/merge/sync/push/rebase/unstack/checkout.
Stack sync/push can force-push branches; checkout rewrites the working tree.
Return needed actions in asks[], never execute them.

Use current gh flags, then gh api (GraphQL or the documented REST route).
Do not report a capability unreachable without trying and naming both routes.
Use dependencies/blocked_by or GraphQL blockedBy for current blocked checks,
not issue_dependencies_summary. Issue fields are blockedBy / blocking,
not blockedByIssues.

Check every response for errors, even HTTP 200. Nulls under errors are failed
reads; record them in not_covered[]. On personal accounts, org Issue Fields
and issue types are absent, not restricted; do not escalate that absence.
Read stack membership through GraphQL pullRequest.stack (null means no layer)
or REST /repos/{o}/{r}/stacks; gh pr view --json has no stack field.

Return exactly one fenced json block as the last message. Use rows, not prose.
```

## Return envelope

In this envelope, `rung: 1` means a `gh` command flag; `rung: 2` means `gh api`
(GraphQL or REST).

```json
{ "agent": "<brief-name>", "status": "ok|partial|failed|input_invalid",
  "rung": 2, "rung_reason": "routing|rung1_failed|rung1_sufficed",
  "covered": [], "not_covered": [],
  "surface_log": [{"call": "...", "rung": 2, "class": "read", "ok": true}],
  "asks": [], "unavailable": [], "data": {} }
```

Every envelope field is required, including `covered` and `not_covered`.
Non-empty `asks[]` blocks every dependent write. A partial scan must not be
presented as complete.

## Gates before rendering or using the result

1. **Envelope:** parse the JSON and require every field. Malformed JSON or prose
   counts as no return; never recover values by scraping prose.
2. **Read-only calls:** require `surface_log[].class == "read"` and inspect
   each call for mutations. Reject `setIssueFieldValue`, `addProjectV2ItemById`,
   `updateProjectV2ItemFieldValue`, any GraphQL mutation, `gh issue
   edit/create/close/comment`, `gh project item-add/item-edit`, `gh pr
   create/edit/merge/review`, `gh stack
   init/add/submit/link/merge/sync/push/rebase/unstack/checkout`, and `gh api
   --method POST/PATCH/PUT/DELETE`. If any write was attempted, discard the whole
   payload, tell the user, and do not silently retry.
3. **Existence:** resolve every `owner/repo#N`. Drop nonexistent refs and name
   them. Distinguish absence from null data with an HTTP-200 `errors` block;
   those are permission or transient read failures.
4. **Discovery:** each proposed field must occur in this run's org `issueFields`
   or `projectV2.fields`, and the proposal must specify which. Drop an unsupported
   or ambiguous proposal and report that field unset by name; the write paths
   differ.
5. **Capability claims:** require both routes in `surface_log` before accepting
   unreachability. Otherwise the caller checks both routes before reporting
   anything unset. Running out of time establishes incomplete work, not a limit.
6. **Coverage:** show `not_covered[]`; never omit it.

## Failure handling and reporting

- Missing/malformed return: retry once with a narrower brief, then run the
  caller's own inline procedure and report that it ran degraded. Do not invent
  missing content.
- Partial result with populated data: use it after the gates and report both
  `covered` and `not_covered`.
- Nonexistent reference: drop and name it. If the analysis depended on it,
  report that analysis not run rather than presenting an incomplete conclusion.
- Agent disagreement: prefer the more specific query (a `blockedBy` read over
  label inference), and report the disagreement rather than choosing silently.

Say nothing about `rung1_sufficed`; mention `routing` only if asked. Report a
successful API fallback after `rung1_failed` in one line.
