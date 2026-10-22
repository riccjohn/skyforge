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
| Which phase it is in and how the last one ended | Ask the coordinator, or read `.light/sessions/<topic>.status` in the worktree |
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

The skill pre-approves only read-only Herdr and yaks commands, such as `herdr agent list`, `herdr agent wait`, and `yx list`. Commands that start sessions or type into them ask for approval every time, so each phase brings two or three prompts.

To stop those prompts, add these rules to `permissions.allow` in `~/.claude/settings.json`. That file applies in every repo, which suits a coordinator you start from any of them. To allow them in one repo only, put them in that repo's `.claude/settings.local.json` instead.

```json
{
  "permissions": {
    "allow": [
      "Bash(herdr agent prompt *)",
      "Bash(herdr agent start *)",
      "Bash(herdr agent rename *)",
      "Bash(herdr notification show *)",
      "Bash(herdr worktree create *)"
    ]
  }
}
```

| Rule | Lets the coordinator |
|---|---|
| `herdr agent prompt` | Send a phase prompt or `/clear` to a worker. Runs two or three times a phase. |
| `herdr agent start` | Start a worker in its pane, once per piece of work. |
| `herdr agent rename` | Give a worker back its name when Herdr loses it. |
| `herdr notification show` | Ping you when a worker needs you. |
| `herdr worktree create` | Create the worktree and the `skyforge/<topic>` branch, once per piece of work. |

Two of these deserve thought before you allow them:

- `herdr agent prompt` lets any session, not only the coordinator, type anything into any Herdr agent without asking you.
- `herdr worktree create` lets any session create branches and worktrees in any repo.

These rules cover only Bash commands. The edit to `.git/info/exclude`, made once per repo, still asks, and the `plannotator` tool, which the coordinator uses for reviews and the Inbox, follows its own permission settings.

## Known gaps

- Claude Code only, for both the coordinator and the workers. OpenCode support is planned. Herdr and Plannotator already support OpenCode; what would change is installing the skill outside the plugin marketplace, clearing workers with `/new` instead of `/clear`, a coordinator watch that doesn't rely on Claude Code's background-command wake-ups, and, the biggest piece, praxis, which depends on Claude Code subagents and tools.
- Run end to end once so far, on one piece of work. Slash commands and `/clear` sent through `herdr agent prompt` work, and workers used the `plannotator` tool for every review instead of the Bash default in praxis skills. Running several pieces of work at once hasn't been tried yet.
- Herdr sometimes loses track of a worker while its session keeps running, and the worker's name disappears from `herdr agent list`. The coordinator restores the name and carries on, but the cause isn't known yet.

## Contributing

See [CONTRIBUTING.md](https://github.com/riccjohn/skyforge/blob/main/CONTRIBUTING.md).

## License

[MIT](./LICENSE)
