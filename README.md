# Skyforge

![The Skyforge in Whiterun, from The Elder Scrolls V: Skyrim: a forge under a giant stone eagle](./assets/skyforge.webp)

<sub>Screenshot from The Elder Scrolls V: Skyrim, © Bethesda Softworks. Not covered by this repository's license.</sub>

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
| Herdr's [Claude Code integration](https://herdr.dev/docs/agents/) | Lets Herdr report when a worker is working, idle, or blocked. Without it the coordinator never sees a worker finish. It adds a `SessionStart` hook to your Claude Code settings. | `herdr integration install claude` |
| The `herdr` skill | Teaches the coordinator Herdr's commands | `npx skills add ogulcancelik/herdr --skill herdr -g` |
| The `plannotator` binary | What the Plannotator plugin runs | See the Plannotator README |
| [yaks](https://github.com/mattwynne/yaks) (`yx`) | The task graph praxis uses. It survives `/clear`, which native tasks don't. | See [Install](https://github.com/mattwynne/yaks#install) in the yaks README |
| A Jira MCP server (optional) | Starting work from a Jira issue key | Whichever server your Jira instance uses |

`/skyforge:start` checks for Herdr and its Claude Code integration, the `herdr` skill, `plannotator`, and `yx`, and tells you which is missing.

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
| `[notes]` | Optional text after a Jira key, such as "the API is done, only the UI is left". It goes into the brief. Requests to change the ticket, such as "move it to In Progress and assign it to me", are carried out by the coordinator once the worker starts. The coordinator changes the ticket only when you ask, and workers never touch Jira. |
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
3. Starts a worker in its own Herdr workspace and runs the three phases. It pings you with a Herdr notification whenever a review, a question, or a permission prompt needs you. See [Where things appear in Herdr](#where-things-appear-in-herdr).
4. Sends a summary to the Plannotator Inbox when the work is done.

Run `/skyforge:start` again with another topic or issue key to start more work in parallel. Run it with no arguments in a new session to pick up running work from disk.

To change how a piece of work is done, edit its brief. Workers re-read it at the start of every phase.

## Where things appear in Herdr

Skyforge puts each piece of work in its own Herdr workspace, so the sidebar shows the state of each one separately.

- **The coordinator** stays in the pane where you ran `/skyforge:start`. Talk to it to start work, change direction, or ask how a piece of work is going.
- **Each piece of work** gets a workspace named after its topic. The workspace is a git worktree on the `skyforge/<topic>` branch, under `~/.herdr/worktrees/`, and the worker runs in its pane.
- **Herdr nests that workspace** under the workspace of the repo's main checkout, because a worktree is an extra checkout of the same repo. The nesting is only a display. Each workspace has its own tabs, panes, and directory.
- **Herdr may open a workspace you didn't create.** If no workspace is open for the repo where the code lands, for example when a Jira issue points at a different repo, Herdr opens one for the main checkout to hold the worktree. It contains a plain shell, and nothing runs in it. The coordinator tells you when this happens.

For example, `/skyforge:start SHOP-482` run from `shop-api`, for work that lands in `storefront`:

```text
spaces
  shop-api                      ← the coordinator, in the tab where you ran it
▾ storefront                    ← opened by Herdr for storefront's main checkout; a plain shell
    shop-482-discount-codes     ← the worker, in its worktree
```

### Following progress

| To see | Look at |
|---|---|
| Whether a worker is working, blocked, or finished | The status shown next to its workspace in the Herdr sidebar |
| What a worker is doing now | Its workspace. Click it to watch the worker; you don't need to type anything there. |
| Which phase it is in and how the last one ended | Ask the coordinator, or read `.light/sessions/<topic>.phase` (the phase in flight) and the last line of `<topic>.status` (how the last phase ended) in the worktree |
| Plans and code to review | Plannotator, which opens a review in your browser when a phase needs your approval |
| Questions a worker can't answer itself | The Plannotator Inbox |
| The final summary | The Plannotator Inbox, when implement is done |

### When a worker needs you

The coordinator sends a Herdr notification that names the worker's workspace and says what it needs:

- **A permission prompt or question in the worker's pane**: open the worker's workspace and answer it there. The coordinator never answers for you.
- **A review or an Inbox question**: answer it in Plannotator.
- **Stuck**: the worker wrote `<phase> stuck: <reason>` and stopped. Tell the coordinator what to do, or edit the brief and ask the coordinator to restart the phase.

The coordinator picks the work up again once the worker carries on.

Herdr shows these notifications only if its pop-ups are turned on, and it hides them for the tab you're looking at. Set `delivery` under `[ui.toast]` in your Herdr config (see Herdr's [notification settings](https://herdr.dev/docs/configuration/#notifications)). The coordinator also says what it needs in its own chat, so with pop-ups off, check there.

### When the work is done

The branch, the worktree, and both workspaces stay in place for you to review. Committing, pushing, and merging are your calls. When you're finished, `herdr worktree remove --workspace <id>` removes the worktree and closes its workspace. A workspace that Herdr opened for the main checkout is yours to close or keep.

## Permissions

The skill's `allowed-tools` pre-approves read-only Herdr and yaks commands, but only for the turn that runs `/skyforge:start`. Once you send the coordinator another message, they ask again in the default permission mode, and the coordinator runs them several times a phase.

These rules allow those read-only commands, notifications, and reads in Herdr's worktrees. Add them to `permissions.allow` in `.claude/settings.local.json` of the repo you run the coordinator from, or in `~/.claude/settings.json`. Either way, workers get them too: a worktree reads the main checkout's settings, and every session reads the global file. So the snippet holds only rules that are safe for any session.

```json
{
  "permissions": {
    "allow": [
      "Bash(herdr agent list*)",
      "Bash(herdr agent get *)",
      "Bash(herdr agent read *)",
      "Bash(herdr agent wait *)",
      "Bash(herdr worktree list*)",
      "Bash(herdr workspace list*)",
      "Bash(herdr integration status*)",
      "Bash(yx list*)",
      "Bash(herdr notification show *)",
      "Read(~/.herdr/worktrees/**)"
    ]
  }
}
```

| Rule | Lets the coordinator |
|---|---|
| `herdr agent list`, `get`, `read`, `wait` | Find workers, read their panes, and watch for them to stop. `herdr agent read` can read any Herdr pane, not only workers. |
| `herdr worktree list`, `herdr workspace list`, `herdr integration status` | Find live work on resume, and check the setup at start. |
| `yx list` | Check the task graph after plan. |
| `herdr notification show` | Ping you when a worker needs you. |
| `Read(~/.herdr/worktrees/**)` | Read status files and phase output in a worker's worktree, and `cd` there to run `yx list`. The worktree is outside the coordinator's directory, so each read asks otherwise. |

If you set `[worktrees] directory` in your Herdr config, use that path in place of `~/.herdr/worktrees`: start it with `~/` for a path under your home directory, or with two slashes for any other absolute path (`//Volumes/dev/wt/**`). A single leading slash anchors at the settings file, and the rule matches nothing.

These still ask, and should:

- **`herdr agent prompt`**, two or three times a phase. Text that starts with `!` runs as a shell command in the target's pane with no permission check, so allowing it is the same as allowing every Bash command, for every session that has the rule. A hook that approves only Skyforge's own prompts (`/clear`, `/praxis:…`) could remove these prompts safely; Skyforge doesn't ship one yet.
- **Writing `.phase`**, once a phase. Workers would get the same rule, and a worker that writes `finished` there ends its own coordination.
- **`herdr agent start`**, once per piece of work. It passes any flags after `--` to the agent, including ones that turn off its permission checks, so read the command before you approve it.
- **`herdr worktree create`**, once per piece of work.
- **`herdr agent rename`**, only when Herdr loses a worker's name. It can rename any agent, so prompts meant for a worker could reach another session.
- **Writing the brief**, once per piece of work. You approve the brief in Plannotator anyway.
- **The edit to `.git/info/exclude`**, once per repo. Claude Code protects `.git`, and no allow rule covers it.

The coordinator also calls the `plannotator` tool once to review the brief and the `plannotator_inbox` tool once at Finish. If either asks, approve it when it does; they run once per piece of work.

## Known gaps

- Claude Code only, for both the coordinator and the workers. OpenCode support is planned. Herdr and Plannotator already support OpenCode; what would change is installing the skill outside the plugin marketplace, clearing workers with `/new` instead of `/clear`, a coordinator watch that doesn't rely on Claude Code's background-command wake-ups, and, the biggest piece, praxis, which depends on Claude Code subagents and tools.
- Run end to end once so far, on one piece of work. Slash commands and `/clear` sent through `herdr agent prompt` work, and workers used the `plannotator` tool for every review instead of the Bash default in praxis skills. Running several pieces of work at once hasn't been tried yet.
- Herdr sometimes loses track of a worker while its session keeps running, and the worker's name disappears from `herdr agent list`. The coordinator restores the name and carries on, but the cause isn't known yet.

## Contributing

See [CONTRIBUTING.md](https://github.com/riccjohn/skyforge/blob/main/CONTRIBUTING.md).

## License

[MIT](./LICENSE)
