---
name: herdr-agents
description: Drive coding agents through Herdr when the user says "use Codex to...", "use Claude to...", "use Cursor to...", or asks to control a Herdr agent or session. Covers starting, prompting, reading results, follow-up, and cleanup.
---

# Drive agents with Herdr

Use Herdr to give another coding agent a bounded task, collect its actual result, and manage the terminals you create. Honor the user's chosen agent, model, directory, session, and cleanup preferences. Otherwise use the selected agent's configured model; ask which agent to use if the choice matters and no preference is established.

Use this workflow when the user requests a particular external agent or Herdr operation. A task merely benefiting from delegation does not select Herdr by itself.

## Choose the session

Sessions persist independently of the attached TUI: detaching or losing the client connection leaves agents and pane processes running, and you can reattach to the same session anytime while its server remains alive. Reuse the existing session after reconnecting; persistence is bounded by the server process lifetime.

Read `herdr --help` and the relevant command groups (`herdr agent`, `herdr pane`, `herdr session`) for installed syntax. Group help can exit 2 while printing valid documentation. `herdr --skill` supplies detailed lifecycle and terminal guidance. Bare `herdr` attaches a TUI; incomplete nested mutators may execute defaults.

There are two supported operating modes for this skill:

- **Inside Herdr:** when `HERDR_ENV=1`, use inherited session context and the caller's pane. Prefer a sibling pane in the current tab with the task's working directory and `--no-focus`.
- **Outside Herdr:** use an explicit named session, defaulting to `agent`. Prefix every session control or inspection command with `herdr --session "$session"` and use explicit pane IDs or live agent names. This skill intentionally permits external named-session control: the bundled skill's `HERDR_ENV=1` prerequisite does not apply to this mode. Leave the environment truthful; target the selected session directly.

Check the selected session with `pane list`. If it returns `server_not_running`, start `herdr --session "$session" server` through a long-running process tool, retain its handle, and check `pane list` again for readiness. `status server` can exit 0 while reporting that no server is running. Keep the server process alive for the work; tool process lifetime may limit persistence after the turn. Report startup errors without switching to another session.

From the live response choose an explicit anchor pane. In an existing session, inspect its layout and preserve existing occupants. IDs are scoped to a server; copy them from JSON, never from an earlier run or another session. If startup yields no pane, consult workspace creation help and create the minimal workspace in the task directory. Record which resources this task created.

The examples below use external named-session mode. In managed mode omit the named-session prefix and use the inherited caller pane as the anchor.

## Start the requested agent

Inspect `herdr agent` for supported kinds. Claude Code maps to `claude`, Codex to `codex`, and Cursor Agent to `cursor` in Herdr 0.9.3; discover other kinds from the installed binary. Advertised support does not prove the native executable is installed or authenticated. Read that executable's help when native arguments are needed (`claude --help`, `codex --help`, or the installed Cursor CLI's help). Put native options after Herdr's `--`; discover model names rather than freezing a list here.

```bash
herdr --session "$session" pane layout --pane "$anchor"
herdr --session "$session" pane split --pane "$anchor" --direction right --cwd "$task_dir" --no-focus
```

Use `down` for a narrow or tall anchor. Capture the new ID from `.result.pane.pane_id`. Start in that available shell pane, using a unique task name matching `[a-z][a-z0-9_-]{0,31}`:

```bash
herdr --session "$session" agent start "$name" --kind "$kind" --pane "$pane" --timeout 30000
```

`agent start` creates no pane. Success means Herdr detected the expected agent and judged it ready. On startup timeout or `agent_not_ready`, inspect the agent and pane before retrying; the process and agent name may already exist. Resolve authentication, trust, or approval prompts according to the user's existing authorization; ask only for decisions not already covered. Preserve native permission settings unless the user asks to change them.

## Delegate and collect

Write a self-contained brief: goal, absolute working paths, relevant context, allowed edits or read-only scope, constraints, completion criteria, and the requested concise result. The child does not inherit this conversation. Pass applicable repository instructions and the user's limits on tests, commits, publishing, and external actions. A prompt is an instruction, not a sandbox.

For independent tasks, use separate named agents and submit their briefs before waiting. Give concurrent writers disjoint files or explicitly chosen worktrees; a pane alone does not isolate filesystem changes. Keep dependent tasks sequential. Do not redo delegated work while its agent is running.

```bash
herdr --session "$session" agent prompt "$name" "$brief" --wait --timeout 120000
herdr --session "$session" agent get "$name"
herdr --session "$session" agent read "$name" --source recent-unwrapped --lines 120
```

For parallel submissions, omit `--wait`, then use `agent wait "$name" --timeout 120000` for each agent. Use asynchronous tool execution so long waits do not prevent progress updates. Quote prompt arguments safely or use structured process arguments; shell interpolation must not execute text from the brief.

Persist toward the goal across turns; you do not need to keep a single thread continuously running. When progress depends on an external event, complete any independent work and use a supported scheduling or notification mechanism that can resume the agent after the turn ends. Prefer an event-triggered callback or a scheduled check over repeated idle polling. A running Herdr session alone does not arrange agent resumption. Confirm registration before promising a future check-in, and preserve enough task context to resume. If no supported mechanism is available or registration fails, report what remains pending and how to resume without promising an automatic follow-up. Honor later user instructions to pause or stop. On pause, stop, or completion, cancel pending check-ins where supported and report whether cancellation was confirmed; preserve the current task status so a late callback does not restart stopped or completed work.

Completion is both a settled agent state and output that answers the assigned brief:

- `idle` and `done` mean ready for input; read the response to assess completion.
- `blocked` means a recognized approval or question UI. Read it and resolve the actual decision rather than sending an unconditional Enter.
- `unknown` is inconclusive. Inspect output and state.
- Timeout or `agent_prompt_stalled` does not prove non-delivery. Read before deciding whether to wait, recover, or resubmit. Waits track lifecycle, not a uniquely identified prompt; wait for an existing turn before assigning a new task to that agent.

Agent names identify live occupants, not durable conversations. Recheck identity before following up after an exit or replacement. Collect actual terminal output; distinguish the child's claims from changes or checks you independently verified.

If the response is truncated, increase `--lines`. Alternate-screen history is not always recoverable. If a larger read still fails, ask the same agent to write its completed answer to a temporary Markdown file and return its path, then read that file on the same machine. Use this as recovery, not a mandatory output format.

## Finish and clean up

Keep task-created agents and panes open after collecting the results, retaining their context for follow-up. Present the result first, identify what remains open, and ask whether the user wants those agents exited and panes closed. Close them only after the user agrees; an earlier explicit cleanup instruction already supplies that agreement. Preserve resources that existed before the task.

Use the native agent's graceful exit and verify the shell returned before closing the pane. Claude Code's `/exit` was verified; discover the appropriate exit action for other agents rather than assuming every agent accepts it. For Claude, send `/exit` with `agent prompt` without a settled-state wait, since the agent is expected to disappear. Inspect the pane to confirm exit.

```bash
herdr --session "$session" pane close "$pane"
herdr --session "$session" pane list
herdr --session "$session" agent list
```

Verify the created pane and agent are gone. Leave a shared or persistent named server running; stop an entire session only when the user requested it or it was explicitly created as disposable and contains no unrelated work. Do not describe a tool-managed server as permanently running without checking its lifetime.

Return the useful answer or change summary, relevant validation, any blocker, and what remains open. Report actual outcomes rather than treating a sent prompt as completed work.
