# Brief: <topic>

## Goal

<!-- One or two sentences: the outcome, in terms of what someone can do afterwards. -->

## Context

<!-- Links to any issue (a Jira ticket, for example), pull or merge request, docs, or code areas the worker should start from. -->

## Constraints

<!-- What must not change, standards to follow, areas that need human review (security-sensitive code, payments, personal data, migrations). -->

## Done when

<!-- Checkable conditions: tests that pass, behaviour that can be observed, files that exist. The implement phase is done when the worker records that every one passes. If the repo has no test suite, list the checks that take its place. -->

## How to work

The coordinator runs you one phase at a time, each in a fresh session. Follow these rules in every phase; they override the defaults of the skill you are running.

- Run every Plannotator review (annotate or code review) with the `plannotator` tool, then end your turn. The decision arrives as a message.
- Send each decision only the person can make to the Plannotator Inbox as a `:::question` block with your recommendation, then carry on with work that does not depend on the answer.
- When the skill offers next steps, choose the one that stops. The coordinator starts the next phase in a fresh session.
- Work only on the `skyforge/<topic>` branch in this worktree. Committing, pushing, and merging are the person's calls.
- Do not read or change Jira or any other issue tracker. What you need from the ticket is in this brief.
- End the phase by writing one line to `.light/sessions/<topic>.status`: `<phase> done` once its output is approved, or `<phase> stuck: <reason>` when you cannot go on.

<!-- Add rules for this piece of work below. Workers re-read this file every phase. -->
