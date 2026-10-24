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

### Symlink skills into your agent

On macOS or Linux, symlink (soft link) each skill directory into your agent's user-level skills directory. This keeps supporting files alongside `SKILL.md`, and changes from `git pull` are available through the links. Keep the checkout in place so the links continue to work.

| Agent | User-level skills directory |
| --- | --- |
| [Claude Code](https://code.claude.com/docs/en/skills#where-skills-live) | `~/.claude/skills` |
| [Cursor](https://cursor.com/docs/skills) | `~/.cursor/skills` |
| [Gemini CLI](https://geminicli.com/docs/cli/skills/#discovery-tiers) | `~/.gemini/skills` |
| [Codex](https://learn.chatgpt.com/docs/build-skills#where-codex-loads-local-skills) | `~/.codex/skills` (or `$CODEX_HOME/skills` if configured); current docs recommend `~/.agents/skills` |
| [Pi](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/configuration.md) | `~/.pi/agent/skills` (or `$PI_CODING_AGENT_DIR/skills` if configured) |
| [OpenCode](https://opencode.ai/docs/skills/) | `~/.config/opencode/skills` (or `$XDG_CONFIG_HOME/opencode/skills` if configured) |

Run this from the directory containing your `mostack` clone. Set `destination` to the agent you want; repeat with another destination to use the skills in multiple agents.

```sh
cd mostack
skills_dir="$(pwd -P)/skills"

# Choose one destination from the table above.
destination="$HOME/.claude/skills"

mkdir -p "$destination"
for skill in "$skills_dir"/*; do
  [ -f "$skill/SKILL.md" ] || continue
  link="$destination/$(basename "$skill")"
  if [ -e "$link" ] || [ -L "$link" ]; then
    printf 'Skipping existing entry: %s\n' "$link"
    continue
  fi
  ln -s "$skill" "$link"
done
```

The command preserves existing files, directories, and symlinks. When copying a destination into the command, use `$HOME` instead of a quoted `~`, which the shell does not expand. Restart your agent or refresh its skills after linking; skill loading and invocation depend on the host agent.

If you use the shared `~/.agents/skills` directory, avoid also linking the same skills into a tool-specific directory that the agent scans.

You'll also need the tools used by the skills:

- **herdr-agents:** Herdr and an installed, authenticated CLI for each coding agent you want to drive.
- **gh-wrapper:** The GitHub CLI (`gh`), authenticated with access to the repositories and organization features you want to manage. The `github/gh-stack` extension is optional for stack CLI operations.

## Approach

- **Discover the current state.** Read available tools, fields, options, and relationships instead of relying on remembered assumptions.
- **Follow work through.** Starting an agent or creating an issue is only part of the task; collect the result and check the requested effects.
- **Keep workflows reusable.** Project and organization policy belongs with the project. The skills provide the operating procedure.
