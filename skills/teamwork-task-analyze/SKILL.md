---
name: teamwork-task-analyze
description: "Use when the user provides a Teamwork.com URL (single task or tasklist) and asks to 'analyzuj task', 'priprav task', 'analyze teamwork task', 'rozpíš task', 'priprav AC a technický plán', or invokes '/teamwork-task-analyze'. Fetches the task via the Teamwork REST API v3 (name, description, attachments, comments, comment-attached files, subtasks) reusing the shared config/token from the `teamwork-task` plugin, deep-scans the current repository for related code/specs, then interactively rewrites the task description into the canonical WAME format: [optional preamble] → HR → Akceptačné kritériá → HR → Cieľ → HR → Technický popis (Claude Code Plan-Mode output). Pauses with AskUserQuestion for every clarifying question that arises and folds the answers back into the new description. Reviews subtasks (updates outdated ones, proposes splitting big tasks). Checks the time estimate using the WAME estimate methodology `wame-estimate-v2` (one number per task against fixed outcome anchors, no speedup factor and no risk buffer) and writes it through the classic v1 endpoint, because the v3 PATCH answers 200 and ignores the field — sets one if missing, otherwise shows a highlighted warning with original vs proposed estimate and reasoning, never silently overwrites. Every write back to Teamwork is preceded by a preview and an explicit AskUserQuestion confirmation. For tasklist URLs reuses the `teamwork-task` filter — only tasks in column 'To Do' assigned to the current user are analysed for write; the rest are listed read-only. Read-only against the board (never moves it) and against time logs (never writes one)."
argument-hint: "<teamwork-url> [--write-back=ask|auto|never] [--language=sk|en] [--max-questions=N] [--no-subtasks-review] [--no-estimate-check] [--include-completed]"
allowed-tools: [Bash, Read, Write, Edit, Grep, Glob, AskUserQuestion]
---

# Teamwork Task Analyzer

Fetch a Teamwork.com task (or whole tasklist) and prepare it for implementation —
rewrite the description into a canonical, unambiguous form, review subtasks,
and check the estimate. The goal is that *after this skill finishes, an engineer
(or another Claude Code skill like `teamwork-task`) can pick up the task and
implement it without asking a single follow-up question*.

The user invoked this skill with: `$ARGUMENTS`

This skill is the **analytical step between the PM handing a task over and the
engineer starting to implement it**. It is read-only against the board and
against time logs — the only things it ever writes back to Teamwork are the
task description, the time estimate, and (when the user explicitly approves)
new or updated subtask descriptions. Every write is gated by a preview and an
`AskUserQuestion` confirmation.

---

## Arguments

Expected first positional argument: a Teamwork.com URL pointing to either a
tasklist or a single task. Same URL shapes as the `teamwork-task` plugin.

Recognized URL shapes:
- Single task: `https://<workspace>.teamwork.com/app/tasks/<taskId>` (also `/#/tasks/<id>`, `/tasks/<id>`)
- Tasklist:    `https://<workspace>.teamwork.com/app/tasklists/<tasklistId>` (also legacy `/tasklists/<id>`)

Optional flags (override config for this run only — not persisted):
- `--write-back=ask|auto|never` — gate every Teamwork write (default: `ask`).
  `auto` skips the confirmation and PATCHes directly; `never` only renders the
  proposal and never writes.
- `--language=sk|en` — language for the rewritten description and the
  clarifying questions (default: from config, fallback `sk`).
- `--max-questions=N` — cap for clarifying questions per run (default from config, 6).
- `--no-subtasks-review` — skip Step 7 entirely.
- `--no-estimate-check` — skip Step 8 entirely.
- `--include-completed` — by default already-completed tasks are skipped in
  tasklist mode; pass this to analyse them too.

If `$ARGUMENTS` is empty or contains no URL, ask via **AskUserQuestion** for
the URL before doing anything else.

---

## Step 1 — Parse the URL

Identical parser to the `teamwork-task` and `teamwork-task-test` plugins:

```bash
URL="<the url>"
WORKSPACE=$(echo "$URL"   | sed -nE 's|https?://([^.]+)\.teamwork\.com/.*|\1|p')
ENTITY_ID=$(echo "$URL"   | sed -nE 's#.*/(tasks|tasklists)/([0-9]+).*#\2#p')
URL_KIND=$(echo "$URL"    | sed -nE 's#.*/(tasks|tasklists)/[0-9]+.*#\1#p' | sed 's/s$//')
BASE_URL="https://${WORKSPACE}.teamwork.com"
```

If any of the three is empty, ask the user via **AskUserQuestion** to confirm
or paste a corrected URL.

---

## Step 2 — Load shared config (reuse `teamwork-task` plugin config)

This skill deliberately shares the config file with the `teamwork-task` and
`teamwork-task-test` plugins, so users who already configured one have nothing
extra to do.

```
~/.claude/plugins/data/teamwork-task-wamesk/config.json
```

Algorithm:

1. `CONFIG_DIR="$HOME/.claude/plugins/data/teamwork-task-wamesk"` — `mkdir -p "$CONFIG_DIR"`.
2. `CONFIG_FILE="$CONFIG_DIR/config.json"`.
3. If `CONFIG_FILE` does not exist:
   - Prefer copying from the `teamwork-task` plugin's bundled template if it is installed (look at `${CLAUDE_PLUGIN_ROOT}/../teamwork-task/config.example.json` or `$HOME/.claude/plugins/cache/.../teamwork-task/config.example.json`).
   - Otherwise copy this plugin's own template from `${CLAUDE_PLUGIN_ROOT}/config.example.json`.
   - `chmod 600 "$CONFIG_FILE"`.
4. Validate with `jq . "$CONFIG_FILE" >/dev/null`. If invalid → report and stop.
5. **First-run check** — if `.teamwork.base_url` or `.teamwork.api_token` are missing/empty/`<workspace>` placeholder, prompt the user via **AskUserQuestion** for both values (prefill `base_url` from the parsed URL). Write them back atomically with `jq` + `mv`, then `chmod 600`. The token comes from `https://<WORKSPACE>.teamwork.com/launchpad/apikey/manage`.
6. **Apply config migration / defaults for this skill's own keys** — see Step 2.5 below.
7. **Apply CLI flag overrides** (`--write-back`, `--language`, `--max-questions`, `--no-subtasks-review`, `--no-estimate-check`, `--include-completed`) to the in-memory config — do not persist.
8. **Never echo the API token.** Always pass auth to `curl` via `-u` (kept out of `ps`), never in the URL.

### Step 2.5 — Analyze-skill config defaults (idempotent merge)

Add this skill's own keys to the same shared config file, defaulting any
missing keys. Run on every invocation — `//=` is idempotent.

```bash
jq '
  (.analyze_skill //= {}) |
  (.analyze_skill.write_back_mode //= "ask") |
  (.analyze_skill.description_format //= "wame_canonical") |
  (.analyze_skill.sections //= {
    "preamble_label": "",
    "acceptance_label_sk": "Akceptačné kritériá",
    "acceptance_label_en": "Acceptance criteria",
    "goal_label_sk": "Cieľ",
    "goal_label_en": "Goal",
    "tech_label_sk": "Technický popis",
    "tech_label_en": "Technical plan",
    "hr_marker": "---"
  }) |
  (.analyze_skill.estimate //= {
    "methodology": "wame_estimate_v2",
    "set_if_missing": true,
    "warn_on_divergence_minutes": 60,
    "step_minutes": 15,
    "max_task_minutes": 480
  }) |
  (.analyze_skill.estimate |= del(.buffer_pct_min, .buffer_pct_max,
                                  .speedup_pct_min, .speedup_pct_max)) |
  (if .analyze_skill.estimate.methodology == "wame_senior_claude_code"
   then .analyze_skill.estimate.methodology = "wame_estimate_v2" else . end) |
  (.analyze_skill.subtasks //= {
    "review_existing": true,
    "propose_split_threshold_minutes": 240,
    "min_split_subtasks": 2,
    "max_split_subtasks": 6
  }) |
  (.analyze_skill.clarifying_questions //= {
    "max_per_run": 6,
    "fold_into_description": true,
    "leave_unresolved_as_open_marker": "[OTVORENÉ]"
  }) |
  (.analyze_skill.fetch_attachments //= true) |
  (.analyze_skill.fetch_comment_files //= true) |
  (.analyze_skill.max_attachment_size_mb //= 25) |
  (.analyze_skill.attachment_extensions_allow //= [
    "md","txt","pdf","docx","xlsx","csv","json","html","eml","msg","sql","png","jpg","jpeg"
  ]) |
  (.analyze_skill.attachments_cleanup //= "after_run") |
  (.tasklist_filter //= {
    "enabled": true,
    "todo_stage": "To Do",
    "todo_stage_match_mode": "case_sensitive",
    "only_assigned_to_me": true,
    "analyze_all_tasks": true,
    "skip_reason_render": "inline"
  })
' "$CONFIG_FILE" > "$CONFIG_FILE.tmp" && mv "$CONFIG_FILE.tmp" "$CONFIG_FILE"
chmod 600 "$CONFIG_FILE"
```

CLI flags override `.analyze_skill.*` values for the duration of this run.

---

## Step 3 — Fetch tasks via Teamwork REST API v3

Same auth pattern as `teamwork-task`:

```bash
TOKEN=$(jq -r '.teamwork.api_token' "$CONFIG_FILE")
BASE=$(jq  -r '.teamwork.base_url'  "$CONFIG_FILE")
AUTH="${TOKEN}:xxx"
```

### Step 3.1 — Single task (`URL_KIND=task`)

```bash
curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/projects/api/v3/tasks/${ENTITY_ID}.json?include=projects,tasklists,assignees,subtasks"
```

Extract from `.task`:
- `id, name, description, projectId, tasklistId, estimateMinutes, priority, status, dueAt, commentsCount, parentTaskId, assigneeUserIds, hasSubtasks`

If `commentsCount > 0`, fetch comments:

```bash
curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/projects/api/v3/tasks/${ENTITY_ID}/comments.json?pageSize=100&page=1&orderBy=postedAt&orderMode=asc"
```

Per-comment fields of interest: `htmlBody, postedAt, author, files[]` (the
files array holds attachments uploaded with the comment).

Always fetch task-level attachments (separate from comment files):

```bash
curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/projects/api/v3/tasks/${ENTITY_ID}/attachments.json?pageSize=100&page=1"
```

Fetch subtasks (one-shot, recursion via `parentTaskId` chain happens in Step 7):

```bash
curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/projects/api/v3/tasks/${ENTITY_ID}/subtasks.json?pageSize=100&page=1&include=cards,stages"
```

Fallback if the `/subtasks.json` endpoint is unavailable on the workspace:

```bash
curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/projects/api/v3/tasks.json?parentTaskIds=${ENTITY_ID}&pageSize=100&page=1"
```

### Step 3.2 — Tasklist (`URL_KIND=tasklist`)

```bash
curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/projects/api/v3/tasklists/${ENTITY_ID}.json"

curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/projects/api/v3/tasklists/${ENTITY_ID}/tasks.json?pageSize=100&page=1&include=cards,stages"
```

Paginate while `.meta.page.hasMore == true`.

**Apply the `tasklist_filter` from config** — same logic as `teamwork-task` v1.4.0:

- If `.tasklist_filter.enabled == true`:
  - Resolve `me.id` from `GET /projects/api/v3/me.json`.
  - For each task look up its board column from `.task.cardId` → `included.cards[cardId].stageId` → `included.stages[stageId].name`.
  - Tasks matching **both** `stage_name == "To Do"` (case-sensitive per config) **AND** `me.id ∈ task.assigneeUserIds` are marked `eligible_for_writeback = true`.
  - All other tasks are still fetched and **analysed read-only** (so the user can sanity-check teammates' work) — they appear in the final report with a `📋 read-only (not in To Do or not assigned to me)` badge and are **never** PATCHed.

- By default skip tasks where `status == "completed"` unless `--include-completed` was passed.

### Step 3.3 — Download attachments + comment files

Only when `.analyze_skill.fetch_attachments == true` (and `.fetch_comment_files`
for comment-attached files). For each file:

- Skip if `size_mb > max_attachment_size_mb` (default 25); record `[SKIPPED: too large]` in the analysis log.
- Skip if extension not in `attachment_extensions_allow`.
- Download via the `downloadURL` field from the file metadata (`curl -sS -u "$AUTH" -L -o "$TMPDIR/<safe-name>"`).
- Convert binary docs to plain text:
  - `.docx` / `.pdf` / `.xlsx` — use whichever converter is on `PATH` (`pandoc`, `pdftotext`, `python3 -m pdfminer.high_level`, etc.). If none is available → record `[SKIPPED: no converter]` and continue.
  - `.md` / `.txt` / `.csv` / `.json` / `.html` — read as-is.
  - `.png` / `.jpg` / `.jpeg` — leave on disk and reference by path in the analysis (the LLM can view them).

When the run ends, honour `attachments_cleanup` (`after_run` → `rm -rf $TMPDIR/teamwork-task-analyze-<run-id>`).

### Step 3.4 — HTTP error handling

- HTTP 401 → token invalid; re-prompt the user (re-run Step 2's first-run flow).
- HTTP 403/404 → report and stop with the URL that failed.
- Network error → retry once with a 2s backoff, then stop.

---

## Step 4 — Local repository context discovery

Only when `.local_context_discovery.enabled == true` (default). The point is to
find **unattached** specs, samples, DNR docs, or related source files in the
current working tree that the task references but does not actually attach.

Algorithm:

1. Build a keyword set from the task name, description, and the latest 1–3 comments
   (lowercase, strip diacritics, drop stop words, keep tokens of length ≥ 3).
2. Walk the directories listed in `scan_dirs` up to `max_depth`. Respect
   `ignore_globs`.
3. For each candidate file, score by:
   - `+2` if filename matches any `filename_hints` substring
   - `+1` per keyword found in filename
   - `+1` per keyword found in the first 4 KB of the file content (only for text-ish extensions)
4. Keep files with `score >= min_keyword_score`; sort by score desc; cap at
   `max_files_to_offer` (default 12).
5. If at least one match is found, present them via **AskUserQuestion**:
   - "Should I include these as additional context for the analysis?"
   - Options: `Yes, include all` / `Let me pick` / `No, ignore`
6. When the user picks individual files, read them via the `Read` tool and add
   their plain text to the analysis context for Step 6.

If no `wamesk/*` / `Modules/*` / `app/` folder exists, skip step 4 silently
(this lets the skill work outside Laravel projects without noise).

Always run a project-shape sniff regardless of file matches:

```bash
[ -f composer.json ] && jq -r '.name' composer.json
[ -f package.json ]  && jq -r '.name' package.json
[ -d wamesk ]        && ls -1 wamesk | head -20
[ -d Modules ]       && ls -1 Modules | head -20
```

The detected stack (Laravel/Vue/Ionic/Nuxt/etc.) goes into the Technical
section of the rewritten description (Step 6).

---

## Step 5 — Parse the current description into sections

WAME convention for an analysed task description:

```
[optional preamble — any text that was there before our format took over]

---

## Akceptačné kritériá
- [ ] …
- [ ] …

---

## Cieľ
2–4 sentences explaining WHY this task exists (business outcome).

---

## Technický popis
Plan-mode style: files, functions, DB schema, step-by-step,
edge cases, references to existing code.
```

Parser rules:

- Strip HTML to plain markdown first (Teamwork stores rich HTML). Preserve
  `<ul>` / `<li>` / `<input type="checkbox">` as markdown bullets / checkboxes.
- Detect known H2 headings case-insensitively, sk **and** en variants:
  - Acceptance: `Akceptačné kritériá`, `Akceptacne kriteria`, `Acceptance criteria`, `Acceptance Criteria`
  - Goal: `Cieľ`, `Ciel`, `Goal`, `Final summary`
  - Tech: `Technický popis`, `Technicky popis`, `Technical plan`, `Technical description`, `Implementation plan`
- Detect HR markers: `<hr>`, `<hr/>`, `<hr />`, or markdown `---` / `***` / `___` on its own line.
- Build a map `{ preamble, acceptance, goal, tech }`:
  - Text **before** the first recognised heading **and** before the first HR → `preamble`.
  - Text under each recognised heading → its section, stopping at the next HR or H2.
  - If a section is missing, store an empty string for it (will be filled in Step 6).
- If the description has **no recognised structure at all** (no headings, no HR),
  store the whole thing as `preamble` so that nothing is lost when we rewrite.

Output this internal `parsed` object for use in Step 6.

---

## Step 6 — Analyze and rewrite (Plan-mode style)

This is the heart of the skill. You (Claude) now act as a senior engineer
preparing the task for implementation.

Inputs to your reasoning:
- `parsed` from Step 5 (preamble + existing sections)
- Task title
- All comments (chronological, with author + timestamp)
- All attachment contents (downloaded + converted in Step 3.3)
- All extra repo context files the user accepted in Step 4
- Detected project stack from Step 4 (Laravel modular / Vue / Ionic / Nuxt / …)

Produce a **draft** new description with this exact structure:

```
{preamble — preserved verbatim if it existed, otherwise omitted entirely (no empty preamble)}

---

## {acceptance_label}
- [ ] AC 1 — atomic, testable, written from the user's perspective
- [ ] AC 2 — …
- [ ] AC 3 — …
…

---

## {goal_label}
2–4 sentences. WHY this task exists. The business / user outcome,
not the technical means.

---

## {tech_label}
Plan-Mode style — what an engineer (or another Claude Code skill) needs to
implement the task without asking a single follow-up question.

Sections inside the technical plan:

### Stack & scope
- Project stack: …
- Affected modules / files: `path/to/file.php:line`, `path/to/Component.vue`, …
- Out of scope: …

### Step-by-step
1. …
2. …
3. …

### Data model / schema changes (if any)
- Table `xxx`: add column `yyy` (type, default, nullable)
- Migration file naming: `YYYY_MM_DD_HHMMSS_add_yyy_to_xxx_table.php`

### Edge cases & risks
- What can go wrong
- What concurrent access looks like
- Idempotency / retry behaviour

### Tests to add
- Pest: …
- Vitest / Playwright: …

### Open questions
- Anything the analyst (you) could not answer with confidence — marked with
  the `[OTVORENÉ]` marker from config so the implementer treats them as
  blockers.
```

Use `{acceptance_label}` / `{goal_label}` / `{tech_label}` from
`.analyze_skill.sections` based on the resolved language (CLI flag wins, else
config `default_language`, fallback `sk`).

### Step 6.1 — Gather clarifying questions

While producing the draft, keep a list of **clarifying questions** for which:

- The answer materially changes AC, the goal, or the technical plan.
- You cannot infer the answer with reasonable confidence from the task,
  comments, attachments, and the repo context.

Cap the list at `.analyze_skill.clarifying_questions.max_per_run` (default 6).
Prioritise: AC ambiguities → data model unknowns → integration points →
UI/UX details.

If there are zero clarifying questions, skip Step 6.2 and go straight to
Step 6.3.

### Step 6.2 — Ask the questions (batched)

Send the questions via **AskUserQuestion** in **batches of up to 4** (the
Claude Code UI limit). After each batch:

- Fold the answers into the draft:
  - Behaviour answers → AC
  - "Why" answers → Goal
  - Implementation answers → Technical plan
- Any question the user answered with "Other" + a freeform note → reproduce
  the note verbatim in the relevant section.
- Any question the user could not answer → leave it as
  `[OTVORENÉ] <question>` in the relevant section (configurable marker).

If after the batches more questions remain than the cap, drop the lowest
priority ones and add a `[OTVORENÉ] dropped: N more low-priority questions
not asked` line in the technical plan.

### Step 6.3 — Preserve preamble

If `parsed.preamble` is non-empty after trimming, place it **at the very top**,
followed by the HR marker, then the three structured sections. Do **not**
re-format the preamble — preserve line breaks, lists, links exactly.

### Step 6.4 — Render the preview

Render the new description as fenced markdown and show it to the user. Compute
a unified diff against the original description (`diff -u <(printf '%s' "$old")
<(printf '%s' "$new")`) and show the first 80 lines of it.

Then via **AskUserQuestion** ask:

- `Apply` — proceed to Step 9 with this description
- `Edit` — user pastes corrections; re-render the preview and ask again
- `Skip description rewrite` — leave the original description untouched, but
  still proceed to Step 7 (subtasks) and Step 8 (estimate)

---

## Step 7 — Subtasks review

Skip entirely when `--no-subtasks-review` was passed.

### Step 7.1 — Existing subtasks

For each subtask returned in Step 3:

1. Fetch its description (the subtasks endpoint already returns it; if not,
   `GET /projects/api/v3/tasks/{subtaskId}.json`).
2. Parse its description with the Step 5 parser.
3. Compare the subtask's apparent scope against the **new** parent AC from
   Step 6:
   - If every parent AC line is covered (directly or by another subtask) → mark
     subtask as `coherent`, no action.
   - If the subtask's scope is now narrower / wider / redirected by the new
     parent plan → draft an updated subtask description (same canonical
     structure) and queue it as a write proposal.
4. When `review_existing == true` (default), present each proposed subtask
   update via **AskUserQuestion** (`Apply` / `Edit` / `Skip`).

### Step 7.2 — Proposing a split for big monolithic tasks

Only when **all** of the following are true:

- The task has **no** subtasks (`hasSubtasks == false` AND `subtasks` array
  empty).
- The proposed estimate from Step 8 exceeds `propose_split_threshold_minutes`
  (default 240).
- The AC list contains at least `min_split_subtasks` items that fall into
  **naturally separable** groups (e.g. different layers — BE / FE / migration;
  different components; clearly independent flows).

If so, draft a split of 2–6 atomic subtasks. Each subtask gets:
- Name (short, business-friendly, prefixed with `[BE]` / `[FE]` / `[QA]` / …
  if the project uses role tags)
- Its own canonical description (preamble omitted; AC + Goal + Technical plan)
- Its own `estimateMinutes` (use the methodology from Step 8)
- Optional `parentTaskId` set to the current task

Show the split as a numbered list with each subtask's name + estimate +
1-line goal. Ask via **AskUserQuestion**:
- `Apply the split (create N subtasks)` — proceed to Step 9
- `Edit the split` — accept user edits, re-render
- `Skip the split (keep monolithic)` — go to Step 8 without creating subtasks

**Do not force a split for small tasks.** If the task is below the threshold,
or if the AC don't have natural cut points, this step is silently skipped —
small tasks live happily as a single line item.

---

## Step 8 — Estimate check

Skip entirely when `--no-estimate-check` was passed.

Compute your own estimate **using the WAME methodology below**. That means one
number picked against the anchors — not a traditional estimate multiplied by a
speedup and a buffer. Round to the nearest `step_minutes` (15 by default). Cap at
`max_task_minutes` (480), and propose a split above
`subtasks.propose_split_threshold_minutes` (240).

### Step 8.1 — Compare with the existing estimate

- If `task.estimateMinutes` is `0` or `null` → queue a "set estimate" proposal
  with your value (preview + `AskUserQuestion` `Apply` / `Edit` / `Skip` in
  Step 9).
- If `task.estimateMinutes > 0`:
  - Let `delta = abs(proposed - original)`.
  - If `delta <= warn_on_divergence_minutes` (default 60) → leave it
    untouched, log "estimate within tolerance" in the final report.
  - Else → render a highlighted block in the final report **and** queue a
    decision question:

    ```
    ⚠️  Estimate divergence

        Original:  <original>m  (<original/60>h)
        Proposed:  <proposed>m  (<proposed/60>h)
        Delta:     <delta>m

        Reasoning:
        <2–4 short lines: which anchor this work sits on, why it moves off
         that anchor (at most one 15-minute step), and what the number
         deliberately excludes. Never cite a speedup or buffer percentage —
         v2 has none.>
    ```

    Then via **AskUserQuestion**:
    - `Replace with proposed` — queue PATCH for Step 9
    - `Keep original` — no change
    - `Custom value` — user types a number (validated as integer multiple of
      `step_minutes`)

---

## WAME estimate methodology

> Block version `wame-estimate-v2`. Shared **verbatim** across the plugins
> `teamwork-task-analyze`, `teamwork-tasks-from-dnr`, `teamwork-tasks-from-desk`,
> `teamwork-tasks-from-session` and `dnr-business`. Change it in all five or in
> none — a per-plugin variant is how two skills start quoting different numbers
> for the same task.

**Estimate one number, directly.** Do not produce a "traditional" estimate and
then multiply it by a speedup and a buffer. Two percentages stacked on a guess
open a band almost twice as wide as the guess itself, and in a negotiation the
widest end of that band always wins. Name the minutes the work takes and defend
that number.

**Who does the work.** A senior engineer who already knows this codebase,
directing Claude Code. Claude Code writes the implementation and the tests; the
engineer decides, reviews and runs the suite. There is no separate QA pass and
no handover to a second person.

**What the number covers**

- Reading the relevant code and reproducing the reported behaviour
- The implementation itself
- Writing or extending the test, and running the affected tests
- Self-review and the fixes it produces
- One round of review feedback

**What the number never covers** — estimate each of these as its own task instead
of folding it in

- Deployment, running the migration on production, fixing production data
- Talking to the client or the PO, and waiting for the answer
- Any work that sits behind an unanswered `[OTVORENÉ]` question
- Anything the task itself declares out of scope

**Shape of the number**

- A multiple of 15 minutes. Never below 15.
- Above 240 minutes: propose a split into 2–6 atomic subtasks. That threshold is
  `propose_split_threshold_minutes` and it is the real ceiling in daily use.
- 480 minutes is a hard cap. Work that will not fit under it is not a task yet.

**Anchors.** These are finished outcomes, not categories of feeling. Pick the
closest line and move by at most one 15-minute step. If the number you want is
more than one step away from every anchor, write down in the reasoning what makes
this case different — that sentence is what a reviewer checks.

| Finished work | Minutes |
|---|---|
| Text, label, translation key or config value, plus the test that guards it | 15 |
| One field, filter or validation rule on one screen, plus a test | 30 |
| Bug with a stack trace or a one-line repro: fix plus regression test | 60 |
| Vue/React component wired to an API that already exists, plus a test | 90 |
| Bug that reproduces but spans 2–3 layers: fix plus tests | 120 |
| One CRUD endpoint or one screen end to end, plus tests | 120 |
| Bug with no repro yet: investigate, then fix | 180 |
| Schema migration with a data backfill and a copy-back assertion | 180 |
| New module in `wamesk/*` (model, migration, Nova screen, policy, tests) | 300 |

**Uncertainty is an open question, not a surcharge.** When you cannot size the
work, you have found something the task does not say yet. Write that question
into the task, estimate the investigation that answers it, and state in the
reasoning what the fix costs under each likely answer. A number with a written
assumption survives review. A number padded for "unknown unknowns" does not, and
it hides the question that was worth asking.

**Do not pad a task because it is labelled TBD**, and do not shrink a real
multi-layer bug so the list looks cheap. Both errors cost the same trust.

**Why this replaced the old rule.** Hand-written estimates used to run about
twice the real cost, which lost us work we should have won. The first fix was a
30–50 % speedup factor with a 15–30 % buffer on top — but that chain put the
padding straight back while sounding rigorous, and it produced a 0.58×–0.91×
band on every single task. The anchors above carry the same judgement as one
number. The measured feedback loop is the `teamwork-tasks-from-session` plugin,
which shows the methodology estimate and the real logged session time side by
side. When those two drift apart on the same kind of work, change the anchors
here — never re-introduce a buffer percentage.

---

## Step 9 — Apply writes (gated)

Honour `.analyze_skill.write_back_mode` (or the `--write-back` flag override):

- `ask` (default) — for every queued proposal show a per-item preview and a
  one-question `AskUserQuestion` (`Apply` / `Skip`). Apply the user's choices
  one by one.
- `auto` — apply every queued proposal directly without re-asking (the user
  already accepted them in Steps 6/7/8; this flag short-circuits the final
  confirmation).
- `never` — print the full list of proposals as a markdown summary and **stop**
  without any HTTP write. Useful for dry runs.

Queued proposal kinds and their API calls:

### 9.1 PATCH task description

```bash
PAYLOAD=$(jq -n --arg desc "$NEW_DESC" '{task:{description:$desc}}')
curl -sS -u "$AUTH" -H "Content-Type: application/json" -H "Accept: application/json" \
  -X PATCH \
  -d "$PAYLOAD" \
  "${BASE}/projects/api/v3/tasks/${TASK_ID}.json"
```

### 9.2 PUT task estimate (classic v1 endpoint — **not** v3)

> **Do not PATCH the estimate through v3.** `PATCH /projects/api/v3/tasks/{id}.json`
> with `{"task":{"estimateMinutes":N}}` answers **HTTP 200 and ignores the
> field** — the task still reads `estimateMinutes: 0` afterwards. A skill that
> uses it reports success and changes nothing, which is worse than failing. The
> estimate is the one field in this skill that must go through the classic v1
> endpoint. (Descriptions and subtask creation are fine on v3 — see 9.1, 9.3, 9.4.)

```bash
PAYLOAD=$(jq -n --arg m "$NEW_ESTIMATE_MIN" \
  '{"todo-item":{"estimated-minutes":($m|tonumber)}}')
curl -sS -u "$AUTH" -H "Content-Type: application/json" -H "Accept: application/json" \
  -X PUT \
  -d "$PAYLOAD" \
  "${BASE}/tasks/${TASK_ID}.json"
```

Read the value back before reporting success — a 200 here proves nothing either:

```bash
curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/projects/api/v3/tasks/${TASK_ID}.json" | jq '.task.estimateMinutes'
```

If the read-back does not match `$NEW_ESTIMATE_MIN`, record the proposal as
`❌ failed: estimate not persisted` in the final report and continue.

### 9.3 PATCH an existing subtask description

Same as 9.1 but with the subtask's id.

### 9.4 POST a new subtask (split flow)

```bash
PAYLOAD=$(jq -n \
  --arg name "$SUB_NAME" \
  --arg desc "$SUB_DESC" \
  --arg parent "$TASK_ID" \
  --arg est "$SUB_EST" \
  '{
    task: {
      name: $name,
      description: $desc,
      parentTaskId: ($parent|tonumber),
      estimateMinutes: ($est|tonumber)
    }
  }')
curl -sS -u "$AUTH" -H "Content-Type: application/json" -H "Accept: application/json" \
  -X POST \
  -d "$PAYLOAD" \
  "${BASE}/projects/api/v3/tasklists/${TASKLIST_ID}/tasks.json"
```

(`TASKLIST_ID` is the parent task's `tasklistId` from Step 3.)

### 9.5 Failures

If any HTTP call returns a non-2xx, render the response body, log the
proposal as `❌ failed: <reason>` in the final report, and **continue** with
the remaining proposals — do not bail out.

---

## Step 10 — Final report

Always render the following summary at the end, even when nothing was written
(useful for `--write-back=never` dry runs):

```markdown
## 📝 Teamwork task analysis — <task title>

URL: <original URL>
Task ID: <id>
Project: <projectName>

### Description rewrite
- Status: ✅ applied / ⏭ skipped / ❌ failed
- Preamble preserved: yes / no
- New sections: AC (<n> items), Goal (<n> sentences), Technical plan (<n> lines)
- Clarifying questions asked: <n> (answered <n>, unresolved <n>)

### Subtasks
- Existing: <n>  — updated <n>, kept as-is <n>
- Proposed split: <n> new subtasks / not proposed

### Estimate
- Original: <original> min
- Proposed: <proposed> min
- Action: ✅ set (was missing) / ⚠️ overwritten on user approval / kept original / ❌ skipped

### Unresolved questions
- [OTVORENÉ] <question 1> — left in <section> of the new description
- …

### Read-only tasks (tasklist mode)
- <count> tasks were fetched and analysed read-only (not in `To Do` or not
  assigned to me); listed below with their current state.
  - <task title> (<id>) — <stage> — assigned to <user>
  - …

### Links
- Task: <BASE>/app/tasks/<id>
- Project: <BASE>/app/projects/<projectId>
```

For tasklist URLs, render one block per task (eligible + read-only), in the
same order the API returned them.

---

## Error handling and stop conditions

Stop **immediately** on:

- Invalid URL after the user's correction attempt in Step 1
- HTTP 401 after the user re-entered the token in Step 2's first-run flow
- Missing `jq` on the system (this skill is hard-bound to `jq`)
- User picks `Cancel` on the very first preview

Continue with a recorded failure on:

- HTTP 4xx/5xx on individual writes in Step 9 (per-proposal failure)
- Missing converter for a single attachment in Step 3.3
- Empty local context discovery in Step 4 (just skip the question)

Never:

- Move the task on the board (this skill is **not** a workflow plugin)
- Post a comment to Teamwork
- Write a time log
- Mark the task complete
- Touch the board column / stage
- Push to git or run any commit hook

---

## Output style

Default to Slovak (`sk`) for the description sections and clarifying
questions unless `--language=en` or `default_language=en` says otherwise.
Headings always use the labels from `.analyze_skill.sections` so projects
that prefer English can configure them centrally.

Keep summaries compact. Prefer numbered lists over prose for plans, and
checkbox lists for AC. Code identifiers, file paths, and brand names stay
in their original form regardless of language.
