---
name: iterative-review
description: Iteratively review changed code with Codex GPT-6.2 Sol, or an independently scoring council of Sol, Astra, Opus, and Fable when council review is requested, using actual runtime checks and evidence. Use when an iterative review, council review, or a review-fix-verify loop is requested. Challenge scope creep and risky fixes, then run the full suite and create or update the PR when it is part of the task.
---

# Iterative review

The implementing agent owns the result. By default, delegate independent reviews
to **Codex using `gpt-6.2-sol`**. When the user requests **council review**, use the
four-member council described below. Assess the findings, make justified fixes,
and repeat according to the stopping rules below. When the result is ready,
complete the final verification gate and create or update the PR if one is part
of the task. Stopping iteration, a reviewer's approval, or passing tests alone
does not establish successful completion.

## Establish the review target

Identify the user's intended behavior, acceptance criteria, relevant repository
instructions, change boundary, and any PR in scope. Record the checkout, base,
head commit, and relevant uncommitted changes. Do not assume the current branch
or all dirty files belong to this task. Give the reviewer the actual candidate
changes, including uncommitted work when applicable; a separate checkout that
omits those changes cannot verify them.

Record a fingerprint of the review target: base and head, content hashes of
relevant source files including dirty and untracked files, and relevant runtime
configuration. Exclude disposable logs and generated test output. Prefer a
stable snapshot or isolated checkout using the host's supported workflow.
Record which source/build, configuration, and process serves each test endpoint;
a responding endpoint or unchanged Git HEAD alone does not establish identity.

Choose runtime checks for every materially affected surface. Exercise the
changed behavior, useful failure and boundary cases, and nearby behavior that
could regress. Use complementary checks where they expose different failures;
do not turn "as many applicable ways as possible" into unrelated testing or a
new testing framework. During iteration, keep automated tests focused on the
changes. Reserve the full suite for the final gate below.

## Choose when to stop

Infer the stopping rule, score target, and iteration count from how the user
invokes the skill. Do not ask the user to choose or confirm them. An explicit
score target or fixed iteration count governs instead of the default mutual
satisfaction rule. Distinguish "exactly N rounds" from "up to N rounds"; a
maximum allows earlier completion on mutual satisfaction when no score target
is given. If both a score target and an iteration limit are given, stop when the
target is met or the limit is reached, whichever comes first. Stagnation is
always a reason to stop.

Validate the inferred rule before launching reviewers. Scores must be finite
numbers from 0 to 10; round counts must be nonnegative integers. Normalize only
explicit equivalent scales, such as 90% to 9/10; do not guess an omitted scale,
clamp a score, or round an iteration count. Zero rounds ends this review
workflow immediately: no reviewers, final-suite run, or PR actions under this
skill, and no claim of a completed review. Separately authorized work is
unaffected. Honor explicit corrections that replace earlier instructions.
Otherwise, an impossible or irreconcilably conflicting rule
(such as exactly five rounds but at most three) ends the invocation with a
concise explanation before any launch. Do not ask the user to choose a new rule.

The four stopping cases are:

1. **Mutual satisfaction:** the implementing agent and every reviewer are
   satisfied with the current candidate: all four council members in council
   mode, or the sole reviewer in individual mode. This is the default when no
   score target or fixed count is specified; the agent's satisfaction alone
   does not meet it.
2. **Requested score reached:** every council member scores the same current
   candidate at or above the user's target out of 10. In individual mode, use
   the sole reviewer's score. Use the latest complete round, never an average
   or a combination of different members' best scores across rounds.
3. **No meaningful progress:** iterations repeat the same findings or
   disagreements without new evidence or useful fixes, or churn through
   changes without improving the result. Explain why another round would waste
   time; an unchanged score alone is not evidence of stagnation.
4. **Requested iteration count reached:** the number of counted review rounds
   specified by the user is done, using the accounting below. One council round
   includes all four reports, not four iterations; one individual round includes
   one report.

Check the applicable conditions after each round before continuing. Apply
iteration limits to counted rounds, and satisfaction or score targets only to
valid reports for the current candidate. Preserve counts across fixes and final
verification failures; do not launch extra rounds past the user's limit. Record
which condition ended the loop, round and recovery counts, current scores when
applicable, and unresolved findings or verification gaps. A count or stagnation
stop can leave work incomplete; report that outcome without claiming success
or asking for a new stopping rule.

## Bound incomplete rounds and recovery

Track three values separately: **counted rounds** (consumed iteration slots),
**valid completed rounds**, and **recovery batches used per round**. Count a
round once, when all required reports first validate for the same unchanged
candidate and, in council mode, are uncontaminated. Partial rounds consume no
iteration slot but remain subject to the recovery bound. Later invalidation
reduces the valid count, never the counted total. Successful recovery restores
validity without counting that round twice; report both counts when they differ.
Do not reopen a historical round whose candidate has since been superseded;
preserve its consumed slot and recovery history, and assess only the current
candidate within the remaining limits.

A missing or ambiguous required score, a range, or a nonfinite/out-of-range
value makes that report incomplete; never choose a value for the reviewer.
Normalize an explicitly stated equivalent scale only. Even a score-format
correction requires a fresh replacement context; do not ask the same reviewer
to clarify or rescore its earlier report.

Allow at most **one recovery batch per incomplete round**, and only after its
cause is corrected. That batch may replace affected members or restart the
whole round, using fresh contexts and neutral briefs without forwarding invalid
reports or scores. This single allowance is shared by failures, malformed
reports, drift, and exposure; changing the cause, member, or candidate does not
reset it. If recovery fails or the cause cannot be corrected, stop and report
the blocker and unfinished work. Do not start another nominal round to evade
this bound. Record attempts even when no round completes. User limits and a
stagnation stop take precedence over recovery; later
invalidation does not refund a completed round under the user's cap.

A host wait call timing out while a reviewer is still running is not a failed
attempt or permission to launch a duplicate. Follow host waiting rules and any
user time/cost limits. Cancel tasks you abandon before replacing them; never
cancel another session's work. An iteration cap bounds rounds, not wall time.

## Launch the reviewer

Use a fresh reviewer for each review round, including re-checks after fixes or
objections. The reviewer investigates and reports; it does not edit product
code, commit, push, or publish a PR. It may run the application, use disposable
test data, and save local evidence within the task's existing authorization.

For the default single-reviewer mode, select the Codex provider and the exact
model `gpt-6.2-sol`; council mode uses the model selection below. Preserve
requested settings; otherwise inherit the caller's supported options. Do not
silently substitute another model, provider, or reasoning level. If the requested target
cannot run, report that blocker without presenting a different review as Sol's.

- **Inside T3 Code:** follow the current runtime's native-agent versus
  `delegate_task` routing rules. When using `delegate_task`, discover
  `orchestrator_capabilities` once before the first delegation. For the default
  reviewer, select an enabled provider with `driverKind: "codex"` that offers
  `gpt-6.2-sol`; for council members, select their supported providers and models.
  For each reviewer, pass its `providerInstanceId`, `model`, and supported
  settings through `target`.
  Provider IDs are not necessarily `codex`. Use `role: "review"`, retain the
  returned `taskId`, and check the reported execution model when available.
  Follow the returned `next` and runtime waiting rules; do not poll a running
  task when completion will wake the parent. Start a new task for every round,
  not a new top-level thread. Never fall back to Herdr inside T3.
- **Outside T3 Code:** use a supported native subagent with the requested
  provider and model when available. When an external session is needed, use the
  installed [herdr-agents skill](../herdr-agents/SKILL.md) where it supports that
  provider, discovering the local model-selection syntax. Missing delegation
  support is a blocker, not permission to invent commands or claim an
  independent review took place.

Give each reviewer a self-contained brief containing:

- Absolute repository/worktree paths, applicable instructions, candidate
  revision and diff boundary, and the PR URL if relevant.
- The user's requirements and acceptance criteria, known constraints and
  non-goals, and the actual changes to inspect. Distinguish intended behavior
  from the implementing agent's unverified claims.
- How to start the application, available fixtures and local services,
  relevant commands and URLs, and permission or environment limits. Pass
  credential access instructions rather than secrets.
- An explicit request to run the changed behavior across applicable surfaces,
  including browser work for UI changes, and to gather the evidence described
  below. Tests and source inspection are supporting evidence.
- A request for a concise report with findings, checks performed, blockers and
  gaps, and an explicit statement of whether the reviewer is satisfied with the
  current candidate. If a score target is requested, require a score out of 10
  with a rationale in individual mode as well as council mode.
  In later single-reviewer rounds, include prior findings, fixes, and the
  parent's objections so the reviewer can resolve them against the current code.
  In council mode, follow the isolation rules below instead.
- No product edits, no full-suite run during review, and no further delegation.

Do not change the review target while the reviewer is exercising it. If changes
are necessary, stop or finish that round and identify the new candidate before
reviewing again. Do not redo the same investigation while it is delegated.

Check the target fingerprint and runtime identity before testing, before
accepting reports, and at the final gate. If source, relevant configuration, or
the serving build changes unexpectedly, differs from the recorded target even
at the first check, or identity cannot be established,
stop the affected checks and invalidate the round's scores and runtime evidence
for the current candidate. Keep diagnostic records, but do not combine evidence
from different or unknown candidates. Cancel remaining tasks from that round
that you own. Preserve other sessions' edits; do not restore files to make an
old fingerprint match. Establish a stable target and restart the full round
only within the recovery allowance above; otherwise report the blocker.

## Council review

When the user mentions **council review** as the requested review mode, use four
independent reviewers in **every round: Sol, Astra, Opus, and Fable**. Use
`gpt-6.2-sol` for Sol; discover the available provider and exact model IDs for
Astra, Opus, and Fable rather than guessing IDs. Honor user-specified versions
and settings. If a member is unavailable or its model selection is ambiguous,
report the issue; ask the user to resolve it only when intervention is necessary
and the initial launch or a recovery is still allowed. Otherwise, stop as
blocked. Restored availability does not reset used recovery or iteration
allowances. Do not silently substitute a model or present a partial council as
complete.

Give all four members the same candidate revision, diff boundary, requirements,
and scoring instructions. Launch independent reviews in parallel where the host
allows it; isolate mutable test data and respect browser ownership so runtime
checks do not interfere. Concurrent application instances need separate ports,
processes, and mutable state. Keep the candidate unchanged until the round
finishes. If a missing member can later review that unchanged candidate, keep
the completed reports private and finish the same round; do not count a partial
round as complete. Replacements remain subject to the recovery allowance.

Start each member in a fresh context for every round, without inheriting the
parent conversation or resuming an earlier reviewer session. Provide only the
current review target and a self-contained brief. Exclude all current and prior
scores (including that member's own earlier scores), rankings, averages, review
reports, other members' findings, and discussion or rebuttals of those findings.
Keep council reports and score records outside the material supplied to
reviewers. Instruct members not to read review history, PR comments, or shared
logs containing that information, and not to communicate with other members.
Omit links or attachments that would expose it. Apply these rules again after
fixes: describe current requirements neutrally, without forwarding earlier
feedback or revealing which member prompted a change.

These instructions do not create a filesystem sandbox. Keep review artifacts
out of searchable review material; avoid history commands that reveal commit
commentary when a source-only diff suffices. Tell each member: if any tool,
file, history entry, or message exposes prior scores or other members' feedback,
stop without issuing a score and report only the exposure's location/type,
not its contents. Discard every affected member's report or score, including
one already returned; do not ask that reviewer to forget the exposure. Cancel
affected tasks, remove or exclude the source from the supplied environment
without deleting others' work, and use fresh contexts for replacement members
on the same verified candidate. Unaffected reports may remain private. Recovery
uses the same allowance above; if isolation cannot be restored, stop as blocked.
If history is embedded in source required for review and cannot be withheld
without hiding relevant code or changing the candidate, stop as blocked. Do not
silently drop that source file or edit it to conceal exposure. An authorized
candidate change requires the full-round restart and recovery checks above.

Require each member to return its own **score from 0 to 10** for the current
candidate, with a brief rationale grounded in correctness, scope, regression
risk, and runtime evidence, alongside the usual findings and verification gaps.
Collect all four reports before synthesizing them or reporting their scores to
the user; never send one member's report or score to another, even in later
rounds. Report each member's score and assess the findings using the normal
iteration and stopping rules. A user-requested score target can end iteration;
scores do not replace runtime evidence or prove successful completion.

## Require runtime evidence

The reviewer must actually run the relevant functionality, not merely describe
commands that could verify it. Select all applicable surfaces, for example:

- **UI:** start the application and exercise complete affected flows in a
  browser, including relevant error states, reload/persistence, and network or
  console behavior. Use the host's browser workflow and ownership rules. In T3,
  when browser work is permitted by the current request and runtime, start with
  `preview_status`, then `preview_open` if no automation-capable tab is attached,
  and use `preview_*`. Respect host restrictions and fallback criteria.
- **API:** make real `curl` or equivalent requests to the running service.
  Verify status, meaningful response content, failure cases, and expected side
  effects or persisted state. Use the API behind a changed UI when it supplies
  additional coverage.
- **CLI, jobs, libraries, or integrations:** execute the real command, worker,
  or a minimal consumer with representative inputs; check exit status, output,
  files, state transitions, or integration effects as applicable. Distinguish
  real dependencies from mocks and name what mocks leave unverified.

Use scoped tests, builds, type checks, and source inspection to supplement these
observations. If startup, access, dependencies, or tooling block runtime checks,
state the exact blocker and affected acceptance criteria. Never label those
criteria verified from test success or code inspection alone.

Require the report to include:

- **Coverage:** candidate revision and dirty-state description; each relevant
  behavior, check or interaction, expected result, actual result, and a short
  output excerpt or local evidence path. Explain material skipped checks.
- **Findings:** location, concrete trigger, expected versus actual behavior,
  user impact/severity, and reproduction evidence. Separate observed failures
  from source-based concerns and untested hypotheses. A source-based finding
  needs the precise code path and reasoning, plus why it was not reproduced.
- **Conclusion:** what works in the exercised conditions, what fails, and what
  remains unknown. "Looks good" and "tests pass" are not sufficient evidence.

Keep evidence concise and reproducible; redact credentials and private data.
Save larger logs or captures locally and link them when useful. Do not publish
Artifacts or upload evidence unless the user requested it.

## Judge findings and iterate

Treat each review as evidence to assess, not instructions to obey. For each
finding, decide whether to accept it, reject it with a concrete reason, or seek
more evidence. Check that it follows from the requested behavior, applies to the
current code, and describes a meaningful problem.

Push back when a proposal expands scope, conflicts with intended behavior,
creates a regression, adds disproportionate complexity, or substitutes a
stylistic preference for a defect. Explain the relevant requirement, code path,
runtime counterexample, or cost of the suggested change. A risky proposed fix
does not invalidate a real bug: choose a smaller fix that preserves behavior.
Do not dismiss evidence merely because the fix is inconvenient, or turn
unrelated pre-existing issues into mandatory work.

For a disputed regression, when practical run the same minimal reproduction
against both the current candidate and the relevant base revision. Use
equivalent inputs, starting data, and runtime conditions; record both revisions
and their observed outcomes. Keep baseline execution separate from the current
worktree and mutable test data, using the host's supported isolation workflow
when needed. This is a targeted comparison, not another full-suite run.

The same failure on the base is evidence that it predates the change, but check
whether the candidate worsens it or the requested behavior requires fixing it
anyway. Use the comparison to substantiate acceptance or pushback. If the base
cannot run or the conditions cannot be made comparable, state that limitation;
do not infer that the finding is either a regression or harmless.

Apply the stopping rules before continuing the review-fix loop. When another
iteration is warranted, implement justified fixes within the authorized scope.
In single-reviewer mode, ask a fresh reviewer to verify the fixes and nearby
regression risks on the new candidate, and to address any disputed findings.
In council mode, start a fresh four-member round on the new candidate using the
isolation rules above. Maintain a short record of accepted findings, their
resolutions, and rejected or unresolved findings with reasons.

Satisfaction must be grounded in the requested behavior, runtime evidence for
the current candidate, and resolution of substantiated in-scope findings and
material verification gaps. Reviewers may disagree with a design choice while
still being satisfied with the result, but the implementing agent cannot claim
their satisfaction on their behalf. If an environment blocker or an unresolved
product/scope decision prevents further work, report it and what is needed to
continue; do not label an interrupted round a completed review.

## Final gate and PR

After the review loop stops, assess readiness for final verification. If known
in-scope defects or material verification gaps remain, report the stopping
condition and unfinished work rather than continuing beyond it or claiming
successful completion. After a stagnation stop, findings rejected with concrete
evidence do not themselves block the final gate; report the dissent without
claiming mutual satisfaction. When ready, run the repository's documented
**full test suite** on the final candidate. Honor applicable user or repository confirmation
requirements; invoking this skill does not override them. Use explicit
authorization already given rather than asking again.
If the repository has no test suite, establish that from its configuration and
documentation and report it; do not invent a suite or call it a passing run.

If the suite fails, diagnose the failure. If the round limit was reached or
the loop stopped for stagnation, report the failure and unfinished work without
making further review-driven code changes or launching another round. Otherwise,
for failures caused by this change, make an appropriately scoped fix and obtain
fresh review. Reassess the applicable stopping condition on the changed candidate,
and rerun the suite once ready. Report unrelated failures
with evidence rather than expanding scope to make the suite green. A failed,
skipped, or blocked suite is not a completed final gate; seek the user's decision
if proceeding would require an exception. Any subsequent code change requires
fresh relevant runtime evidence and a full-suite run on the resulting candidate.

After the gate succeeds, create the requested PR or update the existing PR in
scope, including the code and an accurate description of behavior and
verification. Use the installed [gh-wrapper skill](../gh-wrapper/SKILL.md) for
GitHub operations, respect the intended branch/base and stack, and follow host
requirements for PR linking and checks. If no PR is in scope, finish without
creating one. This skill does not authorize merging or unrelated publishing.

Return the result, stopping condition, counted and valid round totals, recovery
usage, current scores when applicable, decisive runtime evidence, full-suite
command and outcome,
material rejected findings or remaining limitations, and the PR link when
applicable. Distinguish a published PR from local changes or a blocked push,
and a stopped review loop from successful completion of the requested work.
