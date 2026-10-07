# MoStack

My stack of reusable skills for coding agents. MoStack captures the workflows I want agents to follow when coordinating other agents and working with GitHub, so I can give them a task without repeating the operating instructions every time.

The collection is small and focused: two skills, each built around a job I use in day-to-day development.

## What's in the stack

| Skill | What it handles |
| --- | --- |
| [herdr-agents](skills/herdr-agents/SKILL.md) | Drive Claude Code, Codex, Cursor Agent, and other supported coding agents through Herdr. Start work, collect results, recover stalled interactions, and follow up. |
| [gh-wrapper](skills/gh-wrapper/SKILL.md) | Work with GitHub issues, pull requests, custom fields, Projects, relationships, and PR stacks through the CLI and API. |

### Delegate work through Herdr

`herdr-agents` gives the driving agent a workflow for briefing another coding agent and carrying the task through to a usable result. It covers session selection, prompts, output recovery, follow-up, and cleanup.

Example requests:

- “Use Codex to investigate why this build fails.”
- “Use Claude to implement the change, then collect the result.”
- “Send a follow-up to the Cursor agent in Herdr.”

It uses the dedicated `agents` session by default. Agents and panes stay open after results are collected, with cleanup handled explicitly.

### Carry GitHub work through to completion

`gh-wrapper` covers the GitHub operations that go beyond a basic issue or PR command: organization fields, project membership and board fields, cross-repository relationships, and stacked pull requests.

It discovers the available fields and options from GitHub, uses a CLI flag when one exists, and falls back to the API when needed. It checks stack relationships before merging, closing, or retargeting a PR, and reports incomplete operations instead of treating a partial result as done.

Example tasks for the skill:

- “Create an issue and add it to the project with the right status.”
- “Link these issues as dependencies across repositories.”
- “Check this PR's stack before merging it.”

## Use MoStack

Clone the collection:

```sh
git clone https://github.com/alimohammed1624/skills.git mostack
```

Install the skill directories you want from `mostack/skills/` using your coding agent's skill-loading mechanism. Keep each directory intact: a skill can include supporting references alongside its `SKILL.md` entrypoint.

There is no bundled installer. Skill loading and invocation depend on the host agent; explicitly select a skill where your host requires it.

You'll also need the tools used by the skills:

- **herdr-agents:** Herdr and an installed, authenticated CLI for each coding agent you want to drive.
- **gh-wrapper:** The GitHub CLI (`gh`), authenticated with access to the repositories and organization features you want to manage. The `github/gh-stack` extension is optional for stack CLI operations.

## Approach

- **Discover the current state.** Read available tools, fields, options, and relationships instead of relying on remembered assumptions.
- **Follow work through.** Starting an agent or creating an issue is only part of the task; collect the result and check the requested effects.
- **Keep workflows reusable.** Project and organization policy belongs with the project. The skills provide the operating procedure.
