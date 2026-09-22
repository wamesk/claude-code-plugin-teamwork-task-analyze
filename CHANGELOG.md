# Changelog

All notable changes to the `teamwork-task-analyze` plugin are documented in
this file. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)
and the project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.1.0] — 2026-09-22

### Fixed
- **The estimate write never actually worked.** Step 9.2 sent
  `PATCH /projects/api/v3/tasks/{id}.json` with `{"task":{"estimateMinutes":N}}`.
  Teamwork answers **HTTP 200 and ignores the field** — the task still reads
  `estimateMinutes: 0` afterwards. Every run since 1.0.0 therefore reported the
  estimate as applied and changed nothing. The estimate now goes through the
  classic v1 endpoint, `PUT /tasks/{id}.json` with
  `{"todo-item":{"estimated-minutes":N}}`, and the skill reads the value back
  before reporting success — a 200 on this endpoint proves nothing either.
  Descriptions and subtask creation stay on v3, where they do work.

### Changed
- **Estimate methodology replaced — `wame-estimate-v2`.** The old rule produced a
  "traditional" estimate, cut it by 30–50 % for Claude Code, then added a 15–30 %
  buffer on top. Two percentages stacked on a guess give a 0.58×–0.91× band on
  every task, so the same work could legitimately be quoted at 60 or at 95
  minutes and the wider end always won the argument. The methodology now
  estimates **one number directly** against a table of finished-outcome anchors.
  The anchors are tighter (a single figure each, adjust by at most one 15-minute
  step) and the block states explicitly what the number covers — reproduce,
  implement, test, run the suite, self-review, one review round — and what it
  never covers: deployment, production data fixes, client communication, and any
  work behind an unanswered `[OTVORENÉ]` question.
- **Uncertainty is now an open question, not a surcharge.** Where the old text
  told you to pad for "unknown unknowns", the new one tells you to write the
  question into the task, estimate the investigation that answers it, and state
  what the fix costs under each answer.
- **The 240-minute split threshold is now named as the working ceiling**, so it
  no longer contradicts the 480-minute hard cap sitting in the same paragraph.
- The methodology block is byte-identical across `teamwork-task-analyze`,
  `teamwork-tasks-from-dnr`, `teamwork-tasks-from-desk`,
  `teamwork-tasks-from-session` and `dnr-business`, and now carries a version
  marker so a drifted copy is visible.
- Step 8's divergence report no longer asks for "the speedup % applied and the
  buffer % applied" in its reasoning — there are none. It asks which anchor the
  work sits on, why it moves off that anchor, and what the number excludes.

### Removed
- Config keys `estimate.buffer_pct_min`, `estimate.buffer_pct_max`,
  `estimate.speedup_pct_min` and `estimate.speedup_pct_max`. The config
  migration deletes them from files written by earlier versions and renames
  `methodology` from `wame_senior_claude_code` to `wame_estimate_v2` — a stale
  `buffer_pct_max` left in the file reads like a rule somebody still follows.

## [1.0.1] — 2026-06-11

### Fixed
- **URL parser on BSD/macOS sed.** The `ENTITY_ID` / `URL_KIND` extraction in
  Step 1 used `|` as the `sed` substitution delimiter, which collided with the
  `(tasks|tasklists)` alternation and aborted with "parentheses not balanced"
  on BSD sed — leaving both variables empty for every URL. Switched the
  delimiter to `#` on both lines.
- **`write_back_mode` default mismatch.** The persisted default
  (`config.example.json` + the idempotent config merge in Step 2.5) was
  `ask_each`, but the Step 9 dispatcher and the `--write-back` flag only
  recognise `ask` / `auto` / `never`, so the default never matched any branch.
  Reconciled the default to `ask` everywhere it is referenced.
- Removed the `version:` field from the SKILL.md YAML frontmatter —
  `.claude-plugin/plugin.json` is now the single source of truth for the
  plugin version.

## [1.0.0] — 2026-06-04

### Added
- Initial release. Sibling to `teamwork-task` and `teamwork-task-test`.
- **Fetch:** task name, description, attachments, comments, comment-attached files
  and subtasks via Teamwork REST API v3 (`/projects/api/v3/tasks/{id}.json`,
  `/comments.json`, `/attachments.json`, `/subtasks.json`). Reuses the shared
  config at `~/.claude/plugins/data/teamwork-task-wamesk/config.json`, so users
  who already configured the `teamwork-task` plugin do not have to enter the
  API token again.
- **Local repo context discovery** — when invoked inside a project repo, scans
  the working tree (`docs/`, `specs/`, `samples/`, keyword hints) for
  unattached specs / DNR docs that match the task and offers them as
  additional context for the analysis.
- **Description rewrite** into the canonical WAME format:
  optional preamble → HR → **Akceptačné kritériá** → HR → **Cieľ** → HR →
  **Technický popis** (Claude Code Plan-Mode output — files, functions, DB
  schema, step-by-step plan, edge cases). Any text in the original description
  that does not belong to a recognised section is preserved verbatim as the
  preamble.
- **Interactive clarifying-question flow** via `AskUserQuestion` — up to 6
  questions per run, batched in groups of 4 (Claude Code UI limit), answers
  folded back into the new description; anything the user cannot answer is
  preserved as `[OTVORENÉ]` markers in the relevant section so it does not
  silently disappear.
- **Subtasks review** — for each existing subtask checks whether the parent
  task's new AC still covers it and offers an updated description when there
  is a gap. For tasks without subtasks whose proposed estimate exceeds 240 min
  **and** whose AC have natural cut points, proposes splitting into 2–6 atomic
  subtasks with their own names and AC.
- **Estimate check** using the WAME senior-engineer-with-Claude-Code
  methodology (see SKILL.md). Sets the estimate when missing; for existing
  estimates that diverge by more than 60 min from the proposed value, renders
  a highlighted warning with original vs proposed values + reasoning and asks
  the user via `AskUserQuestion` to overwrite, keep, or enter a custom value.
- **Write-back is gated.** Every Teamwork write (PATCH description, PATCH
  estimate, POST subtask, PATCH subtask) is preceded by a preview and an
  explicit `AskUserQuestion` confirmation. Read-only against the board (never
  moves it) and against time logs (never writes one).
- **Tasklist URL support.** When given a tasklist URL, reuses the `teamwork-task`
  v1.4.0 filter — only tasks in column `To Do` assigned to the current user
  are analysed for write-back; the rest are still fetched and listed
  read-only so the user can sanity-check teammates' work.
- Persistent config (API token survives plugin updates). All `analyze_skill.*`
  defaults are merged into the existing shared config file idempotently on
  first run.
