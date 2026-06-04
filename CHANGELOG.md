# Changelog

All notable changes to the `teamwork-task-analyze` plugin are documented in
this file. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)
and the project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
