# Changelog

All notable changes to the `teamwork-task-analyze` plugin are documented in
this file. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)
and the project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.4.0] — 2026-10-13

### Changed
- **Estimate methodology `wame-estimate-v3`.** The shared `## WAME estimate
  methodology` block now says Claude Code writes the code and the engineer
  decides and reviews, and the number is the session time to a done, green
  outcome. The anchors are halved (one screen end to end 120 → 60, new
  `wamesk/*` module 300 → 150, bug with a stack trace 60 → 30) and three anchors
  are new: a query or service method with its rare cases (45), a complex screen
  that saves several records in one transaction (120), a reusable package with
  replaced framework UI components (150). A *Calibration* paragraph records the
  measured pairs that moved them: on 2026-10-13 a settings field with migration,
  model rule and three test files was estimated 90 minutes and took 40, and a
  53.5 h feature plan was rejected as padded and accepted at 27.75 h. The block
  stays identical in all five plugins that carry it.
- Config: `analyze_skill.estimate.methodology` defaults to `wame_estimate_v3`;
  an existing `wame_estimate_v2` (or the older `wame_senior_claude_code`) is
  moved to `wame_estimate_v3` on the next run. README and the plugin
  description name v3.

## [1.3.2] — 2026-10-13

Written on 2026-10-08 as 1.3.1 in a separate checkout and never published; 1.3.1
was released from here with the description fix instead, so the estimate fix
ships now as 1.3.2.

### Fixed
- **A subtask created by the split flow had no estimate.** Step 9.4 posted
  `estimateMinutes` — the v3 *read* field. The POST answered 201 and the
  estimate landed as `0`, the same silent no-op 1.3.0 documented for the
  estimate PATCH. The payload now sends `estimatedMinutes` and the value is read
  back before the split is reported as applied.
- **The parent kept its estimate after a split, so the work counted twice.**
  Teamwork rolls subtask estimates up into the parent's
  `accumulatedEstimatedMinutes` and adds the parent's own field on top. Step 7
  now sets the parent to `0` through Step 9.2 when the split is applied, and
  Step 8.1 compares the proposal against `accumulatedEstimatedMinutes` on a task
  that already has subtasks — a parent with its own number next to estimated
  subtasks is flagged as double-counted and a "zero the parent" proposal is
  queued.

## [1.3.1] — 2026-10-12

### Changed

- **Skill description within the claude.ai limits.** The claude.ai marketplace
  sync stores a skill's frontmatter `description` cut to 1024 characters and
  with angle brackets removed, and warned about this skill. The description is
  now a short trigger summary (under 1024 characters, no angle brackets); the
  behaviour is unchanged and stays in the body of `SKILL.md`.

## [1.3.0] — 2026-09-24

### Added
- **Cross-cutting acceptance criteria (new Step 6.0).** A rewritten AC list
  covered what the reporter asked for and nothing else. It stayed silent about
  the menu entry nobody added, the page only reachable by typing its URL, the
  new action any tenant can trigger, and the list that runs one query per row.
  `teamwork-task-test` has checked exactly these four dimensions (`ui_ux`,
  `performance`, `security`, `reachability`) at QA time since its Step 6.6. The
  analyst now writes the applicable ones into the AC section beforehand,
  under a `### Prierezové požiadavky` sub-heading. Each is a concrete, testable
  `- [ ]` line in the task's language, e.g. *"Nová obrazovka X je dostupná z
  menu <sekcia> pre rolu <rola>"* or *"Z detailu <rodič> vedie na X preklik"*.
  That way they are built in the first place instead of being reported as
  findings afterwards.
  - Only dimensions that apply. A backend-only task gets no UI boilerplate,
    and a task with none applicable says so in the final report.
  - Unknown menu placement, roles or data volume become clarifying questions
    inside the existing batch-of-4 / max-6 budget. An unanswered one stays in
    the criterion as `[OTVORENÉ]`.
  - A deliberately URL-only page gets a criterion that says so, so the tester
    puts it into `allow_orphans` instead of reporting it as an orphan.
  - The format is pinned to what the QA side parses: the sub-heading sits
    inside the AC block, there is one `- [ ]` per line, and there is no `---`
    inside the block. `teamwork-task-test` extracts and ticks criteria by exact
    line match up to the next `---`.
  - Split and updated subtasks carry the criteria for their own slice.
- **Quality subsection in the technical plan.** It names the concrete menu
  location, the inbound links, the policies and gates, and the eager loads,
  indexes and batch sizes. This is the *how* behind each cross-cutting
  criterion.
- **`framework` — the fifth key, in the technical plan only.** An implementer
  who plans from memory reaches for the patterns of older framework versions,
  or hand-rolls what the installed version already ships. When the skill runs
  inside the project's repository, Step 4 now reads the versions actually
  installed — `composer.lock` or `vendor/composer/installed.json` (Laravel,
  Nova, Livewire, Inertia, Pest, the PHP floor from `composer.json`),
  `node_modules` or `package-lock.json` (Vue, React, Nuxt, Vite, TypeScript,
  Tailwind, Inertia, Ionic), browserslist, `.nvmrc` / `engines` — or takes them
  from Laravel Boost's `application-info`. A bare manifest constraint is
  printed as one and planned against as a floor. The quality subsection
  then names the version and the idiomatic feature to use for this task, e.g.
  *"Laravel 12.28: použiť `casts()` + enum cast pre `status`"*, checked in
  current docs (Boost `search-docs`, else context7, else the official docs).
  - It is **never an acceptance criterion.** `teamwork-task-test` treats
    `framework` as advisory — recommendations only — so a criterion about an
    idiom could never be ticked, and the AC list describes behaviour.
  - Guardrails: the project's `CLAUDE.md` and the sibling code's conventions
    win over a newer idiom; nothing newer than the installed version or the
    browserslist target and nothing that version deprecates; only code the task
    adds or changes; one to three lines.
  - Outside the project's repository the skill writes one generic line and
    never a guessed version number. The final report says which versions were
    detected, or that none were.
- New config labels `sections.cross_cutting_label_sk/en` and
  `sections.quality_label_sk/en` — the quality label enumerates all five keys:
  *"Kvalita (UI/UX, výkon, bezpečnosť, dostupnosť, framework)"* /
  *"Quality (UI/UX, performance, security, reachability, framework)"*. They get
  per-key defaults, because the whole-object default of `sections` never
  touches a file that already has one; a label the user already set is kept.
- A **shell portability contract** at the top of `SKILL.md`. The skill runs in
  zsh on macOS, and the note lists the rules below so a later edit does not
  bring them back.
- **Start columns for the new WAME board (`tasklist_filter.todo_stages`).** On
  the shared board (Teamwork workflow "WAME workflow", ~49 projects) work
  starts in two columns: *Ready for Development* and *To Do*. The filter only
  knew the single column `todo_stage: "To Do"`, so a task waiting in *Ready
  for Development* was listed read-only and never got its description
  rewritten. A task is now eligible for write-back when its column is **any** of
  `todo_stages` (default `["Ready for Development", "To Do"]`, still matched
  case-sensitively per `todo_stage_match_mode`). An older per-project board
  that has only `To Do` keeps matching. The legacy string `todo_stage` is
  still read when `todo_stages` is absent or empty. The report names the
  configured start columns instead of a hard-coded "To Do".
  - The config merge adds `todo_stages` **only when it is absent** (or null), with
    exactly the rule `teamwork-task` 1.5.0 applies to the same shared key. The
    two plugins write one file, so neither may undo what the other did: an
    untouched `todo_stage: "To Do"` (or no value) becomes the default pair,
    and a customised `"Backlog"` becomes `["Backlog"]`. An existing
    `todo_stages` is never rewritten, and the legacy key stays. For the
    user's live config (`todo_stage: "To Do"`) the result is
    `["Ready for Development", "To Do"]`. `board_workflow` is not touched,
    because this plugin never moves a card.
  - Verified read-only on tasklist 3365378 (project 589836, "WAME workflow"):
    its 10 tasks in *Ready for Development* were read-only under the old
    single-column filter and are eligible with the new default. The *To Do*
    tasks on project 736882 and on the older board of project 700336 (workflow
    "Default workflow (Copy #1)") match as before.

### Fixed
The first three entries are defects from a colleague's "Štyri cesty k nule"
analysis. Each of them, alone, made the analysis run on **no comments** without
a single error. All of them were re-verified with read-only GETs against a
completed task that has 8 comments.
- **Comments were never fetched.** Step 3.1 fetched them only
  `If commentsCount > 0`, and v3 task objects have no `commentsCount` key, so
  the gate never opened. The reporter's clarifications in the comments never
  reached the rewrite. Comments are now always fetched.
  *Repro:* `jq '.task.commentsCount' <<<"$(curl … /projects/api/v3/tasks/45198800.json)"`
  → `null` on a task with 8 comments.
- **The comments request was invalid.** `orderBy=postedAt` is answered with
  **HTTP 400** `orderBy: unknown comment sort.`. It now uses
  `orderBy=date&orderMode=asc`, checks the HTTP status inline and prints a `⚠`
  naming the endpoint. When v3 refuses, it falls back to the v1
  `GET /tasks/{id}/comments.json` (sorted by `datetime`) with a field map to the
  v3 names. When both fail, the report says *"comments unavailable"*, never
  *"no comments"*.
  *Repro:* `curl -w '%{http_code}' …/tasks/45198800/comments.json?orderBy=postedAt` → `400`.
- **Wrong comment fields.** The documented fields were `postedAt` and `author`,
  and neither exists. They are now `postedDateTime` and `postedByUserId`, with
  names via `include=users`, plus `htmlBody` and `files[]`. "The latest comment
  wins" needs a timestamp that is actually there.
- **The project link in the final report was broken.** Step 3.1 read
  `projectId`, which v3 does not return, so the link came out as
  `/app/projects/null`. It now reads `.projectId // .tasklist.meta.projectId`,
  takes the project name from `.included.projects`, and names only fields v3
  really returns (`dueDate`, not `dueAt`; no `commentsCount`, no `hasSubtasks`).
- **The tasklist filter could never mark a task eligible.** It resolved the
  board column via `.cardId` → `included.cards` → `included.stages` from
  `?include=cards,stages`. On the live API `cardId` is `null` and that
  `included` is empty, so every task in a tasklist was silently listed
  read-only. The column now comes from `.workflowStages[0].stageId` against
  `GET /workflows.json?projectIds=<id>&include=stages`. The current user comes
  from `me.json` → `.person.id`. The snippet now computes the verdict itself
  and records a reason for every read-only task:
  - A task **off the board** (`stageId` `0`) is never eligible. Only the
    configured start columns are, and the task is reported as *"not on the
    board"*.
  - A task on a board whose column cannot be named, because the workflows GET
    failed or the project id is unknown, is reported as *"column unresolved"*
    (fail closed). An empty project id is no longer sent as `projectIds=`,
    which answers 200 with the whole workspace's workflows.
  - `only_assigned_to_me: false` is honoured. A `jq '… // true'` would have
    read the explicit `false` as `true`.
  - When the tasklist GET gives no project id, it is taken from the tasks'
    `.tasklist.meta.projectId` instead of leaving every column unresolved.
  - A `tasklist_filter` block without an `enabled` key counts as enabled, the
    same reading as `teamwork-task`. It used to switch the filter off.
  *Repro:* `jq '.tasks[0].cardId' <<<"$(curl …/tasklists/3361804/tasks.json?include=cards,stages)"`
  → `null`, `.included` → `{}`.
- **Task attachments came from an endpoint that does not exist.**
  `GET /projects/api/v3/tasks/{id}/attachments.json` answers **404**. Attachment
  metadata (`originalName`, `size`, `downloadURL`) now comes from the task fetch
  with `include=attachments` → `.included.files`. Comment files resolve via
  `GET /projects/api/v3/files/{id}.json`.
- **The split proposal gated on `hasSubtasks == false`**, a key v3 does not
  return. Step 7.2 now trusts the subtasks fetch. A failed fetch means
  *unknown*, which blocks the split instead of proposing to split a task that
  already is split.
- **The subtasks fallback returned unrelated tasks.** When `/subtasks.json` was
  unavailable, Step 3.1 fell back to `tasks.json?parentTaskIds=<id>`. v3 ignores
  that parameter and answers with the first 100 tasks of the whole site, which
  the review would then have treated as subtasks. The fallback now uses
  `parentTaskId` (singular), and both results are filtered on `.parentTaskId`
  before anything counts as a subtask. The snippet now also runs the inline
  status check that it used to only describe.
  *Repro:* `jq '[.tasks[].parentTaskId] | unique' <<<"$(curl …/projects/api/v3/tasks.json?parentTaskIds=45379220)"`
  → unrelated parent ids (`0` among them) and `hasMore: true`; with
  `parentTaskId=` → only `45379220`.
- **`fetch_attachments: false` / `fetch_comment_files: false` did not stick.**
  Step 2.5 defaulted both with `//=`, which replaces `false` as well as `null`,
  so every run switched attachment downloads back on. They now default only
  when absent.
  *Repro:* `jq -n '{fetch_attachments: false} | .fetch_attachments //= true'` →
  `{"fetch_attachments": true}`.
- **zsh sweep.** Secondary fetches (comments, subtasks, workflows, files) now
  check their HTTP status and name the gap as *unavailable*. They no longer
  stop the run or pass silently as empty. The Step 9.1 "original survives
  verbatim" check takes the original byte-exact with `jq -j … <<<"$BODY"`, not
  through `echo`, which rewrites escaped newlines. The cleanup path is quoted,
  and the scratch files are removed with `find … -delete` rather than a bare
  glob, which zsh rejects when nothing matches.

### Changed
- README: the estimate step names `wame-estimate-v2`, and the write list says
  the estimate goes through the classic v1 PUT. Both were left over from 1.1.0.

## [1.2.0] — 2026-09-22

### Fixed
- **The description parser deleted pasted screenshots.** Step 5 said "strip HTML to
  plain markdown" and then allowed exactly three tags through — `<ul>`, `<li>` and
  `<input type="checkbox">`. `<img>` was not on that list. A screenshot pasted into a
  Teamwork description is stored **only** as an inline image in the description HTML
  (`tw-inlineimages.s3-accelerate.amazonaws.com`); it is not an attachment and
  `GET /tasks/{id}/files.json` returns nothing for it. So the parser dropped it before
  Step 6 ever saw the content and Step 9.1 then PATCHed a description without it. This
  is not hypothetical — a run over 18 tasks destroyed 25 screenshots, and they came back
  only because the original text was still in a scratch file. `<img>`, `<a href>`,
  existing `![…](…)` markdown and attachment references are now all carried through with
  their URLs intact.
- **The parser had no bucket for unrecognised sections.** The map had four buckets and
  two rules for filling them, so anything under a heading the parser did not know
  (`## Prílohy`, `## Screenshoty`, a bare image after the AC list) matched neither rule
  and was silently lost. The "no recognised structure at all" fallback only saved the
  case where there were no headings at all. There is now a `leftovers` bucket, a slot in
  the draft template that renders it, and an assertion that every image, link and
  attachment reference in the original turns up in one of the buckets.
- **The approval gate could not show the loss.** Step 6.4 renders the first 80 lines of
  the diff, and a screenshot pasted at the end of a long body is removed on a `-` line
  far past that cut — so the human checkpoint showed a clean-looking preview of a
  destructive rewrite. Every removed line that contains an image, a link or an
  attachment reference is now listed in full above the diff, and the default answer
  becomes `Edit`.
- **Subtask rewrites had no preservation rule at all.** Step 7.1 runs subtasks through
  the same parser and the same whole-field PATCH, but the rule was written only for the
  parent task.
- **Step 9.1 overwrote the description field unguarded**, with nothing anywhere stating
  that the original had to be inside the new text. It now stops rather than writing when
  the check fails.
- **A downloaded attachment's local path could reach the description.** Step 3.3 saves
  images to a temp directory and `attachments_cleanup: after_run` deletes it when the
  run ends, so a path written into the task is a dead link before the user reads the
  report. The path is for the analyst's own reasoning; anything written into the task
  keeps the Teamwork URL.

### Changed
- **Two new rules, shared verbatim as the `wame-task-record-v1` block.**
  1. *The estimate lives in the estimate field, and nowhere else.* Minutes never go
     into a task title or description — not in the preamble, not in the technical
     plan, not as a footer line. An estimate gets revised, and a number duplicated
     into prose has to be changed in every copy; the copy somebody misses is the one
     the next reader believes. Previews, confirmation gates, final reports and
     companion documents may still show it — those are read once and thrown away.
  2. *Never lose what the reporter wrote.* When an existing description is rewritten,
     everything already there survives verbatim at the top: inline images, links, the
     reporter's own wording, spelling and punctuation. No diacritics added, no grammar
     fixed, no translation, no tidying.
- The README's "What it does not do" list and the step that describes the rewrite both
  said nothing about the original surviving. A reader — or a model pulling the README
  for context — took the rewrite as a straight format replacement.

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
