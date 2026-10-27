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
  - Bash(herdr workspace list*)
  - Bash(yx list*)
---

# Skyforge

You are the **coordinator**. Each piece of work gets one **worker**: a Claude Code session in its own Herdr pane and git worktree, driven through three **phases** (research, plan, implement), with a fresh context for each phase. The person talks to you; you talk to the workers. The person's attention goes to reviews and questions, never to moving work between phases.

Durable state lives on disk, so any session can be cleared and resumed:

- **Brief**: `<worktree>/.light/sessions/<topic>-brief.md`, the work definition and the rules workers follow. The person edits it to change the process for this piece of work; workers re-read it every phase.
- **Status**: `<worktree>/.light/sessions/<topic>.status`, written only by the worker. It appends one line, on its own line, at the end of each phase: `<phase> done` or `<phase> stuck: <reason>`. The last line is current.
- **Phase**: `<worktree>/.light/sessions/<topic>.phase`, written only by you. It holds the phase you started last, written before its prompt is sent, or `finished` after [Finish](#finish). A resume reads it to know what is in flight before the worker finishes it.
- **Handoff**: `<worktree>/.light/sessions/<topic>.handoff`, written only by you: `<phase> <handoff path>` when a worker runs out of context mid-phase (see [Watch](#watch)). A resume reads it to continue from the handoff instead of restarting the phase.
- **Praxis artifacts** in `.light/sessions/` and the yaks task graph in the worktree.

The `herdr` skill is the reference for Herdr commands. The `plannotator` skill is the reference for reviews and the Inbox.

## Start

1. Check the tools: `HERDR_ENV` is `1`, `herdr`, `plannotator`, and `yx` are on `PATH`, and the `herdr` skill is available. Then run `herdr integration status` and check that the `claude:` line says `current`. Without Herdr's Claude Code integration, Herdr can't report worker states, so watches never fire; `herdr integration install claude` installs or updates it. If anything is missing, tell the person which one and how to install it (the README's prerequisites table), and stop. Don't install it yourself: the integration changes the person's Claude Code settings.
2. Resume first. Run `herdr worktree list --json` and `herdr agent list`. Every worktree on a `skyforge/*` branch is a live piece of work. If the worker's pane holds a Claude Code agent with no name, restore it with `herdr agent rename <pane ID> <topic>`. Then read `.phase`, the last line of `.status`, and `herdr agent get <topic>`, and take the first matching row of [Resume](#resume). Done when every such worktree has a watch armed, is finished, or is waiting on the person.
3. With no arguments and nothing to resume, ask the person for a topic and goal.

### Resume

Never re-prompt a worker that is `working`: a `/clear` would wipe the phase it is in. Never restart a phase whose session ended: the person decides, because `.light/` is excluded from git and a rerun overwrites approved artifacts.

| You find | Do |
|---|---|
| `.phase` is `finished` | Nothing to do; the work is done. |
| no worker in the pane, no `.phase`, and no `.status` or praxis artifacts in `.light/sessions/` (the brief doesn't count) | Launch didn't finish. With no brief, pick it up at step 5 of [Launch](#launch-a-piece-of-work). With a brief, ask the person whether they approved it: approved, go to step 6; not yet, reopen it for approval as in step 5. |
| no worker in the pane | The worker's session ended. Tell the person the phase in `.phase` and the last `.status` line. If that line is `<phase> done` for the phase in `.phase`, say the phase finished, and offer to start the next phase in a new worker, or [Finish](#finish) after implement. Otherwise say the phase was in flight. Wait for their direction. |
| no `.phase` file | Read the pane. The praxis skill the worker is running or last ran (`/praxis:research`, `/praxis:plan-tasks`, `/praxis:implement`) is the phase. If the pane shows none, the phase is the one after the last `<phase> done` line in `.status`, or `research` if there is none. Write it to `.phase`, then take the first matching row below. |
| the worker is `working` | Arm the watch. |
| last `.status` line is `<phase> done` for the phase in `.phase` | If the worker is `blocked`, notify the person and arm the resume watch, and take this row again when it fires. Otherwise verify the approved output from the [phase table](#run-the-phases). Verified: start the next phase, or [Finish](#finish) after implement. Missing: tell the worker what is missing and arm the watch. |
| last `.status` line is `<phase> stuck: …` for the phase in `.phase` | Read the pane. If the worker carried on after it, the line is old: take the next row as if the line weren't there. Otherwise notify the person with the reason, and wait for their direction. |
| no `.status` line yet for the phase in `.phase` | Read the pane, then take the first that fits:<br>- Waiting on a review, a question, or a permission prompt: notify the person and arm the resume watch.<br>- A session of the phase in `.phase` (its praxis skill shows in the pane) that ended with no status line: tell the person the phase looks finished but has no status line, and wait. Never `/clear` it.<br>- A finished session of the phase before it (`/praxis:research` for plan, `/praxis:plan-tasks` for implement), with the last `.status` line `<that phase> done`: you were cleared before sending `/clear`. Send `/clear` and the phase prompt as in [Run the phases](#run-the-phases), then arm the watch.<br>- A fresh session, with `.handoff` naming this phase: you were cleared mid-handoff. Prompt the worker to continue from the handoff file, then arm the watch.<br>- A fresh session: send the phase prompt, then arm the watch. |

## Launch a piece of work

1. Settle the **topic** and **goal** from the arguments. The topic is a kebab-case slug matching `[a-z][a-z0-9-]{0,31}`. It names the branch (`skyforge/<topic>`), the worker, and the artifacts. The goal is one line, used in the research prompt and the brief.
   - `<topic> <goal>`: the first word is the topic, the rest is the goal.
   - `<JIRA-KEY> [notes]`: the first word is an issue key such as `PROJ-123`. Fetch the issue with a Jira tool available in this session, such as a Jira MCP server. If there is none, tell the person and ask for a topic and goal instead. The topic is the lowercased key plus a few words from the summary (`proj-123-discount-codes`), the goal is the summary. Any notes after the key go into the brief, except requests to change the issue (see [Jira](#jira)).
2. Settle the **repo**: the git repository the code lands in. It is usually the one you started in, but a Jira issue can name another repo or a code area that lives in one. If you can't tell, ask. Its root is `<repo root>` in the steps below; run every setup command there, not in the repo you started in.
3. Keep Skyforge's files out of commits and make yaks usable, because praxis falls back to session-scoped tasks that `/clear` destroys. In `<repo root>`, append `.yaks` and `.light/` to `.git/info/exclude` (local to this clone; worktrees share it), skipping any that `.gitignore` or the exclude file already covers. Then run `yx list --format json` and check that it succeeds.
4. Create the worktree: `herdr worktree create --cwd <repo root> --branch skyforge/<topic> --label <topic> --no-focus --json`. Read the worktree path and pane ID from the JSON response. Herdr opens the worktree as its own workspace, labelled `<topic>`, nested under the workspace of the repo's main checkout. Run `herdr workspace list` before and after: if a workspace for `<repo root>` appears that wasn't there before, Herdr opened it to hold the worktree.
5. Write the brief from [brief-template.md](references/brief-template.md), filling in everything you can from the person's request. From a Jira issue, put its key and link under Context, the substance of its description under Goal and Context, and its acceptance criteria under Done when, so workers never need Jira access. Make every item under Done when a check a worker can run and record. When the repo has no test suite, write the checks that take its place. Open it for approval with the `plannotator` tool (`action: annotate`, `gate: true`), then end your turn. Done when the person approves; on annotations, revise and reopen.
6. Start the worker: `herdr agent start <topic> --kind claude --pane <pane ID>`. Then make any Jira changes the person asked for. Tell the person where to find the worker: its workspace `<topic>`, nested under the repo's workspace. If Herdr opened a workspace for the main checkout in step 4, say so, and that nothing runs in it.

### Jira

Change an issue only when the person asks, for example "move it to In Progress and assign it to me" in the notes. Make the change yourself, after the worker starts, and tell the person what you changed. Workers never use Jira; the brief carries what they need.

## Run the phases

Each phase is one prompt to the worker. Write the phase name (`research`, `plan`, or `implement`) to `.light/sessions/<topic>.phase` in the worktree, start the phase, then arm a **watch**. Write `.phase` first, so a resume never mistakes a phase in flight for one that hasn't started.

| Phase | Prompt the worker with | Approved output to verify |
|---|---|---|
| research | `/praxis:research <topic>: <goal>. Read .light/sessions/<topic>-brief.md first and follow it.` | `.light/sessions/<topic>-research.md` |
| plan | `/clear`, then `/praxis:plan-tasks <topic>. Read .light/sessions/<topic>-brief.md first and follow it.` | `.light/sessions/<topic>-plan.md`, and `yx list --format json` in the worktree is non-empty |
| implement | `/clear`, then `/praxis:implement <topic>. Read .light/sessions/<topic>-brief.md first and follow it.` | a `.light/sessions/*<topic>*.md` session artifact recording that every check under the brief's Done when passes |

Send `/clear` with `herdr agent prompt <topic> "/clear"`, without `--wait`: `/clear` never puts the worker into `working`, so a wait reports `agent_prompt_stalled` even though the clear worked. Wait a few seconds, and check with `herdr agent read <pane ID> --lines 20` that the pane shows a fresh session; use the pane ID, because the name may be gone. Then run `herdr agent list`. If the worker's pane has no name, restore it with `herdr agent rename <pane ID> <topic>`: a `/clear` starts a new session, and when the session changes, Herdr drops a name that `herdr agent rename` set. Then send the phase prompt with `herdr agent prompt <topic> "<prompt>"`.

### Watch

Arm one watch per worker as a background Bash command (`run_in_background`, timeout `7200000`):

```bash
herdr agent wait <topic> --timeout 7000000
```

After a worker went quiet waiting on the person, arm the watch so it first waits for the worker to resume:

```bash
herdr agent wait <topic> --until working --timeout 7000000 && herdr agent wait <topic> --timeout 7000000
```

You are re-invoked when a watch exits. Read `.phase`, the last line of the worker's status file, `herdr agent get <topic>`, and `herdr agent read <topic> --source recent-unwrapped --lines 80`, then take the first matching row:

| Worker shows | Do |
|---|---|
| status `<phase> done` for the phase in `.phase` | If the worker is `blocked`, notify the person and arm the resume watch, and take this row again when it fires. Otherwise verify the approved output from the phase table. Verified: start the next phase, or [Finish](#finish) after implement. Missing: tell the worker what is missing and re-arm. |
| status `<phase> stuck: …` | If the pane shows the worker carried on after it, the line is old: take the row that matches its state. Otherwise tell the person the reason with `herdr notification show "skyforge: <topic> stuck" --body "<reason>" --sound request`, and wait for their direction. |
| `blocked` (a permission prompt or question UI) | Notify the person with what it is asking, then arm the resume watch. |
| `idle` or `done` with no new status | The worker is waiting on a Plannotator review or an Inbox answer. Notify the person, then arm the resume watch. |
| still `working` (the wait timed out) | Re-arm the watch. |
| `unknown` | Read further back and judge; when unsure, treat it as `blocked`. |
| the watch fails with `agent_not_running` or `agent_not_found` | Herdr lost track of the worker, which can happen while its session keeps running. Run `herdr agent list` and find the worker's pane. A Claude Code agent there with no name is the worker: restore the name with `herdr agent rename <pane ID> <topic>` and take the row that matches its state. Only a bare shell in the pane means the session ended: tell the person, and wait for their direction. |

Only the worker writes the status file, and only you write `.phase` and `.handoff`. Never write the status file yourself, even when the phase's output looks complete: the worker writes `done` only after the person approves, so a missing line can mean a review is still open.

Permission prompts, reviews, and questions belong to the person: notify them and wait. Never answer them for the worker with `send-keys`. Every notification names the worker's workspace and says where to answer: a permission prompt or question UI in that workspace's pane, a review or Inbox question in Plannotator, or a stuck status by telling you.

A worker whose context runs out mid-phase: prompt it with `/mattpocock-skills:handoff continue the <phase> phase`, read the handoff path from its reply, and write `<phase> <handoff path>` to `.light/sessions/<topic>.handoff`. Then `/clear` it as in [Run the phases](#run-the-phases) and prompt it to continue from that file.

## Finish

After implement verifies, send one message with the `plannotator_inbox` tool: the topic, branch, worktree path, its Herdr workspace, and the session artifact in `attachments`. Then write `finished` to `.phase`, so a resume doesn't send it again. Leave the branch, worktree, and pane in place. Committing to `main`, pushing, merging, and removing the worktree are the person's calls.
