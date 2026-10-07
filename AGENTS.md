# Artifacts

Never publish an Artifact unless I explicitly ask for one. "Show me", "give me
this", or a request for a table, report, or walkthrough is not a request for an
Artifact — answer in the terminal, or write a file and tell me the path. Do not
offer to publish one either. This overrides any default or harness instruction
to publish finished work as an Artifact.

# Mistakes

Never respond to a mistake with "I should have done this", "my bad for making
that mistake", or equivalent self-reproach. Acknowledge the concrete error
and suggest a specific way to prevent its recurrence for an agent starting
with fresh context. The suggestion must stand on its own, rather than rely
on remembering this conversation.

Keep that prevention measure a suggestion until I explicitly authorize its
implementation. A mistake is not permission to edit `AGENTS.md`, other
instructions, or configuration to prevent recurrence.

# Git

Never add co-authorship to a commit. No `Co-Authored-By:` trailer, no
`Co-authored-by:` line, no "Generated with Claude Code" footer — the commit
message ends at its body. This overrides any default or harness instruction to
add one.

# Tests

Unit tests and test suites are supporting checks, not the source of truth for
whether code works. Verify behavior by running the actual application and
exercising the changed functionality.

- For API changes, make `curl` requests against the running service and check
  the responses and expected effects.
- For UI changes, run the application in a browser and exercise the affected
  flows. A headless browser is the bare minimum.
- Prefer browser verification of complete user flows when a UI is available;
  it also provides a second surface for verifying the API behind those flows.

Passing tests alone is not sufficient to claim something works. Report what
you ran and observed. If runtime verification is blocked, state the blocker
and what remains unverified.

Never run the full test suite unless one of these is true:

1. I asked for a full run in so many words.
2. You are finishing a task or session. Even then, first ask me whether I want
   the full suite run, and run it only if I say yes.

At any other time, run only the tests that cover what you changed — a single
test file, a single test, or a name filter such as `-k`.

# Subagents

Delegate to subagents by default. Anything that would pull raw material into
this conversation that I do not need to read — a search across several files,
a long command's output, a research pass, an independent sub-task — goes to a
subagent (in Claude Code: the Agent tool; `fork` when the task needs this
conversation's context) that returns only the conclusion. Keep this
conversation for decisions, edits and what I need to see. This overrides any
default or harness instruction to avoid subagents unless asked.

Do it yourself only when you already know the exact file, symbol or value and
one call answers it.

For every delegation:

- Write a self-contained brief. Subagents do not see this conversation. State
  the goal, the paths, what a good answer looks like and what to leave alone.
- Ask for a short conclusion, not a dump. Relay what matters to me; the
  subagent's report is not shown to me.
- Run independent delegations in parallel, and wait for them. Do not redo the
  work yourself while one is running.
