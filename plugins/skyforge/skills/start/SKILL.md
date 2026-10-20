---
name: start
description: Coordinate Claude Code workers in Herdr, one per piece of work, each taken through praxis research, plan, and implement in its own worktree.
disable-model-invocation: true
argument-hint: "<topic> <goal> | <JIRA-KEY> [notes] | nothing to resume"
allowed-tools:
  - Bash(herdr agent list*)
  - Bash(herdr agent get *)
  - Bash(herdr agent read *)
  - Bash(herdr agent wait *)
  - Bash(herdr worktree list*)
  - Bash(herdr pane list*)
  - Bash(herdr integration status*)
  - Bash(yx list*)
---

# Skyforge

You are the **coordinator**. Each piece of work gets one **worker**: a Claude Code session in its own Herdr pane and git worktree, driven through three **phases** (research, plan, implement), with a fresh context for each phase. The person talks to you; you talk to the workers. The person's attention goes to reviews and questions, never to moving work between phases.

Durable state lives on disk, so any session can be cleared and resumed:

- **Brief**: `<worktree>/.light/sessions/<topic>-brief.md`, the work definition and the rules workers follow. The person edits it to change the process for this piece of work; workers re-read it every phase.
- **Status**: `<worktree>/.light/sessions/<topic>.status`, one line the worker writes at the end of each phase: `<phase> done` or `<phase> stuck: <reason>`.
- **Praxis artifacts** in `.light/sessions/` and the yaks task graph in the worktree.

The `herdr` skill is the reference for Herdr commands. The `plannotator` skill is the reference for reviews and the Inbox.

## Start

1. Check the tools: `HERDR_ENV` is `1`, `herdr`, `plannotator`, and `yx` are on `PATH`, and the `herdr` skill is available. Then run `herdr integration status` and check that the `claude:` line says `current`. Without Herdr's Claude Code integration, Herdr can't report worker states, so watches never fire; `herdr integration install claude` installs or updates it. If anything is missing, tell the person which one and how to install it (the README's prerequisites table), and stop. Don't install it yourself: the integration changes the person's Claude Code settings.
2. Resume first. Run `herdr worktree list --json` and `herdr agent list`. Every worktree on a `skyforge/*` branch is a live piece of work: read its status file and pick it up at the matching step of [Run the phases](#run-the-phases). Done when every such worktree has a watch armed or is finished.
3. With no arguments and nothing to resume, ask the person for a topic and goal.

## Launch a piece of work

1. Settle the **topic** and **goal** from the arguments. The topic is a kebab-case slug matching `[a-z][a-z0-9-]{0,31}`. It names the branch (`skyforge/<topic>`), the worker, and the artifacts. The goal is one line, used in the research prompt and the brief.
   - `<topic> <goal>`: the first word is the topic, the rest is the goal.
   - `<JIRA-KEY> [notes]`: the first word is an issue key such as `PROJ-123`. Fetch the issue with a Jira tool available in this session, such as a Jira MCP server. If there is none, tell the person and ask for a topic and goal instead. The topic is the lowercased key plus a few words from the summary (`proj-123-discount-codes`), the goal is the summary. Any notes after the key go into the brief, except requests to change the issue (see [Jira](#jira)).
2. Settle the **repo**: the git repository the code lands in. It is usually the one you started in, but a Jira issue can name another repo or a code area that lives in one. If you can't tell, ask. Its root is `<repo root>` in the steps below; run every setup command there, not in the repo you started in.
3. Keep Skyforge's files out of commits and make yaks usable, because praxis falls back to session-scoped tasks that `/clear` destroys. In `<repo root>`, append `.yaks` and `.light/` to `.git/info/exclude` (local to this clone; worktrees share it), skipping any that `.gitignore` or the exclude file already covers. Then run `yx list --format json` and check that it succeeds.
4. Create the worktree: `herdr worktree create --cwd <repo root> --branch skyforge/<topic> --label <topic> --no-focus --json`. Read the worktree path and pane ID from the JSON response.
5. Write the brief from [brief-template.md](references/brief-template.md), filling in everything you can from the person's request. From a Jira issue, put its key and link under Context, the substance of its description under Goal and Context, and its acceptance criteria under Done when, so workers never need Jira access. Make every item under Done when a check a worker can run and record. When the repo has no test suite, write the checks that take its place. Open it for approval with the `plannotator` tool (`action: annotate`, `gate: true`), then end your turn. Done when the person approves; on annotations, revise and reopen.
6. Start the worker: `herdr agent start <topic> --kind claude --pane <pane ID>`. Then make any Jira changes the person asked for.

### Jira

Change an issue only when the person asks, for example "move it to In Progress and assign it to me" in the notes. Make the change yourself, after the worker starts, and tell the person what you changed. Workers never use Jira; the brief carries what they need.

## Run the phases

Each phase is one prompt to the worker. Start it, then arm a **watch**.

| Phase | Prompt the worker with | Approved output to verify |
|---|---|---|
| research | `/praxis:research <topic>: <goal>. Read .light/sessions/<topic>-brief.md first and follow it.` | `.light/sessions/<topic>-research.md` |
| plan | `/clear`, then `/praxis:plan-tasks <topic>. Read .light/sessions/<topic>-brief.md first and follow it.` | `.light/sessions/<topic>-plan.md`, and `yx list --format json` in the worktree is non-empty |
| implement | `/clear`, then `/praxis:implement <topic>. Read .light/sessions/<topic>-brief.md first and follow it.` | a `.light/sessions/*<topic>*.md` session artifact recording that every check under the brief's Done when passes |

Send `/clear` with `herdr agent prompt <topic> "/clear" --wait --timeout 60000`, and the phase prompt with `herdr agent prompt <topic> "<prompt>"`.

### Watch

Arm one watch per worker as a background Bash command (`run_in_background`, timeout `7200000`):

```bash
herdr agent wait <topic> --timeout 7000000
```

After a worker went quiet waiting on the person, arm the watch so it first waits for the worker to resume:

```bash
herdr agent wait <topic> --until working --timeout 7000000 && herdr agent wait <topic> --timeout 7000000
```

You are re-invoked when a watch exits. Read the worker's status file, `herdr agent get <topic>`, and `herdr agent read <topic> --source recent-unwrapped --lines 80`, then take the first matching row:

| Worker shows | Do |
|---|---|
| status `<phase> done` for the current phase | Verify the approved output from the phase table. Verified: start the next phase. Missing: tell the worker what is missing and re-arm. |
| status `<phase> stuck: …` | Tell the person the reason with `herdr notification show "skyforge: <topic> stuck" --body "<reason>" --sound request`, and wait for their direction. |
| `blocked` (a permission prompt or question UI) | Notify the person with what it is asking, then arm the resume watch. |
| `idle` or `done` with no new status | The worker is waiting on a Plannotator review or an Inbox answer. Notify the person, then arm the resume watch. |
| still `working` (the wait timed out) | Re-arm the watch. |
| `unknown` | Read further back and judge; when unsure, treat it as `blocked`. |

Permission prompts, reviews, and questions belong to the person: notify them and wait. Never answer them for the worker with `send-keys`.

A worker whose context runs out mid-phase: prompt it with `/mattpocock-skills:handoff continue the <phase> phase`, read the handoff path from its reply, `/clear` it, and prompt it to continue from that file.

## Finish

After implement verifies, send one message with the `plannotator_inbox` tool: the topic, branch, worktree path, and the session artifact in `attachments`. Leave the branch, worktree, and pane in place. Committing to `main`, pushing, merging, and removing the worktree are the person's calls.
