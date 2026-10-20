# Skyforge

A Claude Code plugin for running several Claude Code sessions at once. `/skyforge:start` turns your session into a **coordinator**. For each piece of work it creates a git worktree and a **worker** session in a Herdr pane, then drives that worker through praxis research → plan → implement, clearing its context between phases.

You review each phase's output in Plannotator. Workers send questions to the Plannotator Inbox instead of asking in chat, and they never push or merge.

## Prerequisites

Installing Skyforge installs these plugins for you, once their marketplaces are added (see [Installation](#installation)):

| Plugin | Used for |
|---|---|
| [Plannotator](https://github.com/backnotprop/plannotator) (`plannotator@plannotator`) | Plan and code review, the Inbox |
| praxis (`praxis@lightfactory`) | The research, plan-tasks, and implement skills |
| mattpocock-skills (`mattpocock-skills@claude-plugins-official`) | `/mattpocock-skills:handoff`, used when a worker runs out of context |

These aren't Claude Code plugins, so install them yourself:

| Tool | Used for | Install |
|---|---|---|
| [Herdr](https://herdr.dev) | Panes, worktrees, and controlling the worker sessions. Run Claude Code inside Herdr. | See the Herdr README |
| The `herdr` skill | Teaches the coordinator Herdr's commands | `npx skills add ogulcancelik/herdr --skill herdr -g` |
| The `plannotator` binary | What the Plannotator plugin runs | See the Plannotator README |
| [yaks](https://github.com/mattwynne/yaks) (`yx`) | The task graph praxis uses. It survives `/clear`, which native tasks don't. | `curl -fsSL https://raw.githubusercontent.com/mattwynne/yaks/main/install.sh \| bash` |
| A Jira MCP server (optional) | Starting work from a Jira issue key | Whichever server your Jira instance uses |

`/skyforge:start` checks for Herdr, the `herdr` skill, `plannotator`, and `yx`, and tells you which is missing.

## Installation

Claude Code doesn't add a dependency's marketplace for you, so add Plannotator's and praxis's before installing Skyforge. `claude-plugins-official` is added by default.

```shell
claude plugin marketplace add backnotprop/plannotator
claude plugin marketplace add 8thlight/lightfactory
claude plugin marketplace add riccjohn/skyforge
claude plugin install skyforge@skyforge
```

If you skip the first two lines, the install still succeeds but Skyforge stays disabled with `Dependency "…" is not installed`. Add the missing marketplaces and run `/reload-plugins` to install them.

### Updating

```shell
claude plugin marketplace update skyforge
claude plugin update skyforge@skyforge
```

Start a new Claude Code session to pick up the update.

## Usage

From a Claude Code session running inside Herdr, in the repo you want to work on, run `/skyforge:start` in one of three forms:

```text
/skyforge:start <topic> <goal>
/skyforge:start <JIRA-KEY> [notes]
/skyforge:start
```

| Argument | What it is |
|---|---|
| `<topic>` | A short kebab-case name for the piece of work: lowercase letters, digits, and hyphens, starting with a letter, up to 32 characters. It names the branch (`skyforge/<topic>`), the Herdr worker, and the files under `.light/sessions/`. |
| `<goal>` | Everything after the topic: a one-line description of what you want built. The coordinator puts it in the brief and in the worker's research prompt. |
| `<JIRA-KEY>` | A Jira issue key such as `PROJ-123`, in place of a topic and goal. The coordinator reads the issue and derives both: the topic from the key and summary (`proj-123-discount-codes`), the goal from the summary. The description and acceptance criteria go into the brief. Needs a Jira MCP server, or another Jira tool, in the coordinator's session. |
| `[notes]` | Optional text after a Jira key, such as "the API is done, only the UI is left". It goes into the brief. |
| _(nothing)_ | Resume the work already running in this repo, after a `/clear` or in a new session. |

For example:

```text
/skyforge:start add-discount-codes Let customers apply a discount code at checkout
/skyforge:start SHOP-482
/skyforge:start SHOP-482 Leave the payment provider integration alone
```

The coordinator:

1. Creates the worktree and the `skyforge/<topic>` branch.
2. Writes a **brief** (`.light/sessions/<topic>-brief.md`) for you to approve. It holds the goal, the definition of done, and the rules workers follow. From a Jira issue, it carries the ticket's content, so workers don't need Jira access.
3. Starts a worker and runs the three phases. It pings you with a Herdr notification whenever a review, a question, or a permission prompt needs you.
4. Sends a summary to the Plannotator Inbox when the work is done.

Run `/skyforge:start` again with another topic or issue key to start more work in parallel. Run it with no arguments in a new session to pick up running work from disk.

To change how a piece of work is done, edit its brief. Workers re-read it at the start of every phase.

## Permissions

The skill pre-approves only read-only Herdr and yaks commands. Each `herdr agent prompt` and `herdr worktree create` asks for approval. If you are comfortable with that, allow them in your own settings.

## Known gaps

- Claude Code only, for both the coordinator and the workers. OpenCode support is planned. Herdr and Plannotator already support OpenCode; what would change is installing the skill outside the plugin marketplace, clearing workers with `/new` instead of `/clear`, a coordinator watch that doesn't rely on Claude Code's background-command wake-ups, and, the biggest piece, praxis, which depends on Claude Code subagents and tools.
- Not yet tested end to end. In particular, check that `/clear` and slash commands sent through `herdr agent prompt` run as commands.
- Praxis skills launch Plannotator through Bash by default. The brief tells workers to use the `plannotator` tool instead so the session isn't held by a two-minute Bash timeout. Watch for workers that ignore this.

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md).

## License

[MIT](./LICENSE)
