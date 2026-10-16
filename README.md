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

The plugin requires the `work-rules` plugin (skill `wame-estimate`, the estimate methodology `wame-estimate-v3`). It is declared in `dependencies`, so it installs together with this plugin; if it is missing, run `/plugin install work-rules@wame`.

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
   comment-attached files, and subtasks via Teamwork REST API v3. Comments are
   **always** fetched, oldest first *(1.3.0 — there is no comment-count gate any
   more, because v3 never returns a count to gate on)*. When v3 refuses, the
   skill falls back to the classic v1 endpoint. Every request's HTTP status is
   checked, so a failed fetch is reported as *unavailable* and never passes as
   *none*.
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
   DB schema, step-by-step, edge cases, tests to add). The AC section also gets
   the **cross-cutting requirements** that apply *(1.3.0)* — see below.
6. **Reviews subtasks** — updates outdated ones; for tasks above 240 min
   without subtasks **and** with naturally separable AC, proposes a split
   into 2–6 atomic subtasks (never forces a split for small tasks).
7. **Checks the estimate** using the WAME estimate methodology
   `wame-estimate-v3` (from the `work-rules` plugin). Sets one when missing; warns + asks before overwriting a
   diverging existing value.

Every Teamwork write (PATCH description, PUT estimate via the classic v1
endpoint, POST subtask, PATCH subtask) is preceded by a preview and an explicit
`AskUserQuestion`.

## Cross-cutting requirements *(1.3.0)*

Acceptance criteria rewritten from the reporter's text cover what the reporter
asked for. They do not cover the menu entry nobody added, the page you can only
reach by typing its URL, the new action any tenant can trigger, or the list that
runs one query per row. `teamwork-task-test` checks exactly those four
dimensions at QA time (`ui_ux`, `performance`, `security`, `reachability`, its
Step 6.6). This skill writes the ones that apply into the task beforehand, so
they get built rather than reported afterwards:

- In **Akceptačné kritériá**, the applicable criteria go under a
  `### Prierezové požiadavky` sub-heading. Each one is a concrete, testable
  `- [ ]` line in the task's language. Examples:
  `- [ ] Nová obrazovka Faktúry je dostupná z menu Účtovníctvo pre rolu Účtovník`
  and `- [ ] Z detailu Zákazníka vedie na Faktúry preklik (záložka vzťahu)`.
  Each line stays single, and the block contains no `---`, so
  `teamwork-task-test` can tick the criteria it proves.
- In **Technický popis**, a short quality subsection names the concrete menu
  location, the inbound links, the policies, and the eager loads and indexes.
- The same subsection carries the fifth key, **`framework`** — the framework
  versions the repository actually has installed (read from `composer.lock`,
  `node_modules`, browserslist; never assumed) and the idiomatic feature of that
  version to use for this task, e.g. *"Laravel 12.28: použiť `casts()` + enum
  cast pre `status`"*, checked in current docs (Laravel Boost `search-docs`,
  else context7, else the official docs). It is **plan only, never an
  acceptance criterion**: `teamwork-task-test` treats `framework` as advisory,
  so such a criterion could never be ticked. The project's `CLAUDE.md` and its
  sibling code win over a newer idiom, and untouched code is not rewritten. Run
  outside the project's repository, the skill writes one generic line instead of
  guessing versions.
- **Only dimensions that apply.** A backend-only task gets no UI criteria.
- **Asking about unknowns.** When the menu placement, the roles or the data
  volume is unknown, the skill asks inside the usual question budget (batches
  of 4, at most 6). Anything left unanswered is marked `[OTVORENÉ]` in the
  criterion itself.

The additive contract does not change: the reporter's text stays verbatim on
top, the estimate never lands in the description or the title, and nothing is
written without the preview and confirmation.

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

Same filter as `teamwork-task` v1.5.0 — only tasks in one of the configured
**start columns** **and** assigned to the current user are analysed for
write-back. Everything else in the tasklist is still fetched and listed
read-only with its stage and assignee so you can sanity-check teammates'
work without touching it.

The start columns come from `tasklist_filter.todo_stages` *(1.3.0)* — default
`Ready for Development` and `To Do`, the two columns where work starts on the
WAME board (Teamwork workflow "WAME workflow"). An older per-project board that
has only `To Do` still matches. Names are matched exactly (case-sensitive) unless
`todo_stage_match_mode` is `case_insensitive`. The legacy single-column key
`todo_stage` is still read when `todo_stages` is absent or empty.

A task that is **not on the board** is never analysed for write-back — it is
listed read-only as *"not on the board"*. When a column cannot be resolved
because the workflow request failed, the task is listed read-only as *"column
unresolved"*, never guessed.

The column is resolved from each task's `workflowStages` against the project's
workflow stages *(1.3.0)*. The earlier card-based lookup always came back empty,
which left every task read-only.

## Config

Stored at `~/.claude/plugins/data/teamwork-task-wamesk/config.json` (shared
with the `teamwork-task` plugin family). This plugin adds an `analyze_skill.*`
namespace — see `config.example.json` for the full list of keys (description
section labels, estimate methodology parameters, subtask split thresholds,
clarifying-question cap, attachment handling). Since 1.3.0 the labels include
`cross_cutting_label_sk/en` ("Prierezové požiadavky") and `quality_label_sk/en`
("Kvalita (UI/UX, výkon, bezpečnosť, dostupnosť, framework)" / "Quality (UI/UX,
performance, security, reachability, framework)"). They are added to an
existing config file automatically; a value you already set is kept.

The `tasklist_filter` block is shared with `teamwork-task`. Since 1.3.0 this
plugin adds `tasklist_filter.todo_stages` when it is missing (or null), using the same
rule as `teamwork-task` 1.5.0: an untouched legacy `todo_stage: "To Do"` (or no
value) becomes `["Ready for Development", "To Do"]`, and a customised one such
as `"Backlog"` becomes `["Backlog"]`. An existing `todo_stages` is never
rewritten. The file ends in the same state whichever of the two plugins runs
first. This plugin never touches `board_workflow`, because it never moves a card.

The shared config survives plugin updates. Token is `chmod 600`.

## Shell compatibility *(1.3.0)*

The skill's shell snippets run in your login shell. On macOS that is zsh,
which is what Claude Code's Bash tool uses; elsewhere it is bash. `SKILL.md`
opens with a short **shell portability contract** that every snippet follows.
The most important rule is that JSON never goes through `echo`: zsh's `echo`
turns the `\n` escapes inside a Teamwork response into raw newlines, and `jq`
then rejects the response.

## License

MIT — © 2026 Stanislav Červeňák
