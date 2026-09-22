# teamwork-task-analyze

The **analytical step** between a PM handing a task over and an engineer
starting to implement it. Fetches a Teamwork.com task by URL, rewrites the
description into a canonical, unambiguous form, reviews subtasks, and checks
the estimate — interactively, with previews and confirmations for every
Teamwork write.

Goal: after this skill finishes, an engineer (or another Claude Code skill
like `teamwork-task`) can pick up the task and implement it without asking a
single follow-up question.

## Install

```
/plugin marketplace add wamesk/claude-code
/plugin install teamwork-task-analyze@wame
```

The Teamwork API token is shared with the `teamwork-task` and
`teamwork-task-test` plugins — if one of those is already configured, this
plugin uses the same token. On first run otherwise the skill prompts for the
workspace and token (`https://<WORKSPACE>.teamwork.com/launchpad/apikey/manage`).

## Usage

```
/teamwork-task-analyze https://<workspace>.teamwork.com/app/tasks/12345
/teamwork-task-analyze https://<workspace>.teamwork.com/app/tasklists/67890
```

Optional flags:

- `--write-back=ask|auto|never` — gate Teamwork writes (default `ask`)
- `--language=sk|en` — language for the rewritten description (default from config)
- `--max-questions=N` — cap for clarifying questions per run (default 6)
- `--no-subtasks-review` — skip subtasks review
- `--no-estimate-check` — skip estimate check
- `--include-completed` — analyse already-completed tasks too (tasklist mode)

## What it does

1. **Fetches** the task name, description, attachments, comments,
   comment-attached files, and subtasks via Teamwork REST API v3.
2. **Scans** the current repository for unattached specs / DNR docs / source
   files that the task references and offers them as extra context.
3. **Parses** the existing description into preamble + AC + Goal + Technical
   plan sections.
4. **Asks** clarifying questions via `AskUserQuestion` (in batches of 4, up
   to 6 per run). Folds answers back into the new draft; preserves anything
   unresolved as `[OTVORENÉ]` markers in the relevant section.
5. **Rewrites** the description into the canonical WAME format — everything the
   reporter already wrote is carried into the new text **verbatim** at the top,
   inline screenshots and links included; the rewrite only adds sections below it:
   `[preamble] → HR → ## Akceptačné kritériá → HR → ## Cieľ → HR → ## Technický popis`
   where the technical plan is Claude Code Plan-Mode output (files, functions,
   DB schema, step-by-step, edge cases, tests to add).
6. **Reviews subtasks** — updates outdated ones; for tasks above 240 min
   without subtasks **and** with naturally separable AC, proposes a split
   into 2–6 atomic subtasks (never forces a split for small tasks).
7. **Checks the estimate** using the WAME senior-engineer-with-Claude-Code
   methodology. Sets one when missing; warns + asks before overwriting a
   diverging existing value.

Every Teamwork write (PATCH description, PATCH estimate, POST subtask,
PATCH subtask) is preceded by a preview and an explicit `AskUserQuestion`.

## What it **does not** do

- Never moves the task on the board
- Never posts a comment to Teamwork
- Never writes a time log
- Never completes the task
- Never runs `git commit` / `git push`
- Never marks subtasks as done
- Never loses anything from an existing description — the original text, its
  inline images (a pasted screenshot lives only in the description body) and its
  links are carried into the rewrite verbatim

It is purely **read-only against the board and timesheet** — only the task
description, estimate, and (with consent) subtasks are written.

## Tasklist URLs

Same filter as `teamwork-task` v1.4.0 — only tasks in column `To Do` (exact
case-sensitive match) **and** assigned to the current user are analysed for
write-back. Everything else in the tasklist is still fetched and listed
read-only with its stage and assignee so you can sanity-check teammates'
work without touching it.

## Config

Stored at `~/.claude/plugins/data/teamwork-task-wamesk/config.json` (shared
with the `teamwork-task` plugin family). This plugin adds an `analyze_skill.*`
namespace — see `config.example.json` for the full list of keys (description
section labels, estimate methodology parameters, subtask split thresholds,
clarifying-question cap, attachment handling).

The shared config survives plugin updates. Token is `chmod 600`.

## License

MIT — © 2026 Stanislav Červeňák
