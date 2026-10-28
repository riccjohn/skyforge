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

From a Claude Code session running inside Herdr, in the repo you want to work on, run `/skyforge:start`. You can say what to work on in the same line, in your own words, or leave it out and the coordinator asks:

```text
/skyforge:start
/skyforge:start Let customers apply a discount code at checkout
/skyforge:start SHOP-482
/skyforge:start SHOP-482, but leave the payment provider integration alone
```

| You give | What the coordinator does |
|---|---|
| _(nothing)_ | Resumes the work already running in this repo, after a `/clear` or in a new session. With nothing to resume, asks what the worker should do. |
| A description of the work | Uses it as the **goal**: a one-line description of what you want built, which goes in the brief and the worker's research prompt. Anything longer goes into the brief. |
| A Jira issue key such as `PROJ-123` | Reads the issue and takes the goal from its summary, and its description and acceptance criteria for the brief. Anything else you write goes into the brief. Requests to change the ticket, such as "move it to In Progress and assign it to me", are carried out once the worker starts; the coordinator changes the ticket only when you ask, and workers never touch Jira. Needs a Jira MCP server, or another Jira tool, in the coordinator's session. |

Before creating anything, the coordinator proposes a **topic** and asks you to confirm it or type your own. The topic is a short kebab-case name for the piece of work (`add-discount-codes`, or `proj-123-discount-codes` from an issue): lowercase letters, digits, and hyphens, starting with a letter, up to 32 characters. It names the branch (`skyforge/<topic>`), the Herdr worker, and the files under `.light/sessions/`.

The coordinator:

1. Creates the worktree and the `skyforge/<topic>` branch.
2. Writes a **brief** (`.light/sessions/<topic>-brief.md`) for you to approve. It holds the goal, the definition of done, and the rules workers follow. From a Jira issue, it carries the ticket's content, so workers don't need Jira access.
3. Starts a worker in its own Herdr tab and runs the three phases. It pings you with a Herdr notification whenever a review, a question, or a permission prompt needs you. See [Where things appear in Herdr](#where-things-appear-in-herdr).
4. Sends a summary to the Plannotator Inbox when the work is done.

Run `/skyforge:start` again to start more work in parallel. In a new session, run it on its own to pick up running work from disk.

To change how a piece of work is done, edit its brief. Workers re-read it at the start of every phase.

## Where things appear in Herdr

Skyforge puts each piece of work in its own tab, in the Herdr workspace where you ran the coordinator.

- **The coordinator** stays in the pane where you ran `/skyforge:start`. Talk to it to start work, change direction, or ask how a piece of work is going.
- **Each piece of work** gets a tab named after its topic. The tab opens in a git worktree on the `skyforge/<topic>` branch, under `~/.herdr/worktrees/<repo>/`, and the worker runs in its pane. When the code lands in a different repo from the one you ran the coordinator in, the tab still opens in your workspace; only its directory is in the other repo.
- **The Agent panel** in the Herdr sidebar lists each worker with its tab name and state. The Spaces panel shows your workspace once, with the state of all its tabs rolled up.

For example, `/skyforge:start SHOP-482` run from `shop-api`, for work that lands in `storefront`:

```text
shop-api
  tab 1                         ← the coordinator, where you ran it
  tab shop-482-discount-codes   ← the worker, in its storefront worktree
```

### Following progress

| To see | Look at |
|---|---|
| Whether a worker is working, blocked, or finished | Its row in the Agent panel of the Herdr sidebar, or its tab |
| What a worker is doing now | Its tab. Open it to watch the worker; you don't need to type anything there. |
| Which phase it is in and how the last one ended | Ask the coordinator, or read `.light/sessions/<topic>.phase` (the phase in flight) and the last line of `<topic>.status` (how the last phase ended) in the worktree |
| Plans and code to review | Plannotator, which opens a review in your browser when a phase needs your approval |
| Questions a worker can't answer itself | The Plannotator Inbox |
| The final summary | The Plannotator Inbox, when implement is done |

### When a worker needs you

The coordinator sends a Herdr notification that names the worker's tab and says what it needs:

- **A permission prompt or question in the worker's pane**: open the worker's tab and answer it there. The coordinator never answers for you.
- **A review or an Inbox question**: answer it in Plannotator.
- **Stuck**: the worker wrote `<phase> stuck: <reason>` and stopped. Tell the coordinator what to do, or edit the brief and ask the coordinator to restart the phase.

The coordinator picks the work up again once the worker carries on.

Herdr shows these notifications only if its pop-ups are turned on, and it hides them for the tab you're looking at. Set `delivery` under `[ui.toast]` in your Herdr config (see Herdr's [notification settings](https://herdr.dev/docs/configuration/#notifications)). The coordinator also says what it needs in its own chat, so with pop-ups off, check there.

### When the work is done

The branch, the worktree, and the worker's tab stay in place for you to review. Committing, pushing, and merging are your calls. When you're finished, close the tab, then run `git worktree remove <worktree path>` in the repo; the branch stays until you delete it.

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
      "Bash(herdr pane list*)",
      "Bash(herdr integration status*)",
      "Bash(git worktree list*)",
      "Bash(git -C * branch --show-current)",
      "Bash(yx list*)",
      "Bash(herdr notification show *)",
      "Read(~/.herdr/worktrees/**)",
      "Read(~/.config/herdr/config.toml)"
    ]
  }
}
```

| Rule | Lets the coordinator |
|---|---|
| `herdr agent list`, `get`, `read`, `wait` | Find workers, read their panes, and watch for them to stop. `herdr agent read` can read any Herdr pane, not only workers. |
| `herdr pane list`, `git worktree list`, `git -C <dir> branch --show-current` | Find live work on resume: the panes whose directory is on a `skyforge/*` branch, and worktrees that lost their tab. |
| `herdr integration status` | Check the setup at start. |
| `yx list` | Check the task graph after plan. |
| `herdr notification show` | Ping you when a worker needs you. |
| `Read(~/.herdr/worktrees/**)` | Read status files and phase output in a worker's worktree, and `cd` there to run `yx list`. The worktree is outside the coordinator's directory, so each read asks otherwise. |
| `Read(~/.config/herdr/config.toml)` | Read your Herdr config's `[worktrees] directory`, so new worktrees go where Herdr puts its own. |

If you set `[worktrees] directory` in your Herdr config, use that path in place of `~/.herdr/worktrees`: start it with `~/` for a path under your home directory, or with two slashes for any other absolute path (`//Volumes/dev/wt/**`). A single leading slash anchors at the settings file, and the rule matches nothing.

These still ask, and should:

- **`herdr agent prompt`**, two or three times a phase. Text that starts with `!` runs as a shell command in the target's pane with no permission check, so allowing it is the same as allowing every Bash command, for every session that has the rule. A hook that approves only Skyforge's own prompts (`/clear`, `/praxis:…`) could remove these prompts safely; Skyforge doesn't ship one yet.
- **Writing `.phase`**, once a phase. Workers would get the same rule, and a worker that writes `finished` there ends its own coordination.
- **`herdr agent start`**, once per piece of work. It passes any flags after `--` to the agent, including ones that turn off its permission checks, so read the command before you approve it.
- **`git worktree add` and `herdr tab create`**, once per piece of work.
- **`herdr agent rename`**, only when Herdr loses a worker's name. It can rename any agent, so prompts meant for a worker could reach another session.
- **Writing the brief**, once per piece of work. You approve the brief in Plannotator anyway.
- **The edit to `.git/info/exclude`**, once per repo. Claude Code protects `.git`, and no allow rule covers it.

The coordinator also calls the `plannotator` tool once to review the brief and the `plannotator_inbox` tool once at Finish. If either asks, approve it when it does; they run once per piece of work.

## Known gaps

- Claude Code only, for both the coordinator and the workers. OpenCode support is planned. Herdr and Plannotator already support OpenCode; what would change is installing the skill outside the plugin marketplace, clearing workers with `/new` instead of `/clear`, a coordinator watch that doesn't rely on Claude Code's background-command wake-ups, and, the biggest piece, praxis, which depends on Claude Code subagents and tools.
- Run end to end once so far, on one piece of work. Slash commands and `/clear` sent through `herdr agent prompt` work, and workers used the `plannotator` tool for every review instead of the Bash default in praxis skills. Running several pieces of work at once hasn't been tried yet.
- Herdr sometimes loses track of a worker while its session keeps running, and the worker's name disappears from `herdr agent list`. The coordinator restores the name with `herdr agent rename` and carries on. Herdr's source (0.7.5) explains why the loss repeats: a name given by `herdr agent start` survives a `/clear`, but a name given by `herdr agent rename` belongs to the current session, and a `/clear` starts a new one. The coordinator checks for the name after every `/clear` and restores it. What drops the first name isn't known yet.

## Contributing

See [CONTRIBUTING.md](https://github.com/riccjohn/skyforge/blob/main/CONTRIBUTING.md).

## License

[MIT](./LICENSE)
