---
name: teamwork-task-analyze
description: "Use when the user provides a Teamwork.com URL (task or tasklist) and asks to 'analyzuj task', 'priprav task', 'analyze teamwork task', 'rozpíš task', 'priprav AC a technický plán', or invokes '/teamwork-task-analyze'. Fetches the task via the Teamwork REST API v3 (description, attachments, comments, subtasks), scans the repository for related code and specs, asks every open question, and rewrites the description into the WAME format additively: the reporter's text stays verbatim on top, then Akceptačné kritériá, Cieľ and Technický popis. Adds the cross-cutting criteria that apply (menu entry for a new screen, authorization for a new action, data volume for a new list). Reviews subtasks and proposes splits. Checks the estimate with wame-estimate-v2 and never silently overwrites one. Every write is previewed and confirmed. For a tasklist URL only tasks in a start column assigned to the current user are analysed for write-back. Never moves the board, never logs time."
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

A rewrite here is **additive**. Whatever the reporter already wrote survives
verbatim at the top of the new description — their wording, their spelling, their
links, and their inline screenshots — and the canonical sections are appended
below it. The skill never writes the estimate into the description or the title;
that number lives in the Teamwork estimate field alone. Both rules are stated in
full under **Task record rules** near the bottom of this document.

The rewritten acceptance criteria also carry the **cross-cutting requirements**
that apply to the task — the menu entry and the inbound links for a new screen,
the authorization of a new action, the expected data volume of a new list —
the same four dimensions (`ui_ux`, `performance`, `security`, `reachability`)
that `teamwork-task-test` reviews at QA time in its Step 6.6 (see Step 6.0
below). The fifth key, `framework`, lives in the **technical plan only**: it
names the framework versions the repository actually has installed and the
idiomatic feature of that version to use for this task. It never becomes an
acceptance criterion — `teamwork-task-test` treats `framework` as advisory, so
such a criterion could never be ticked.

---

## Shell portability contract

Every `bash` block in this file runs in the user's login shell — **zsh on
macOS** (Claude Code's Bash tool is `/bin/zsh`), bash elsewhere — and every
Bash tool call is a fresh shell, so no function or variable survives from one
snippet to the next. Keep each snippet valid in both shells:

- **Never pass JSON through `echo`.** zsh's builtin `echo` expands `\n`, `\t`
  and `\\` inside the JSON into raw control characters and `jq` rejects the
  result. Use `jq … <<<"$JSON"` or `printf '%s\n' "$JSON" | jq …`. The same
  goes for description or comment text piped into `grep`, `sed`, `diff` or
  `python3`. (`echo "$URL" | sed` is harmless — a URL has no backslashes.)
- **Check the HTTP status of every GET** (`curl -w '\n%{http_code}'`) and
  print a `⚠` naming the endpoint on a non-200 or a parse error, then continue
  with an explicit, reported fallback. **Never** `2>/dev/null || echo 0` after
  a parse — that is how a failed request turns into a silent "0 comments".
- **No word splitting in zsh:** `for X in $VAR` iterates once. Iterate lines
  with `while IFS= read -r X; do …; done <<<"$VAR"`.
- `[ "$a" = "$b" ]` with a single `=` (`==` fails in zsh); no `${!ARR[@]}` or
  `${!name}`; no numeric array subscripts (zsh is 1-based) — use `${ARR[@]:$i:1}`.
- **An unmatched glob is a hard error in zsh** and aborts the whole command
  list. Quote URLs and paths; use `find … -name` instead of a bare glob that may
  match nothing.
- Quote any word that starts with `=` (zsh expands `=word` to a command path).
- Tool calls share no functions: repeat the inline status check in each
  snippet rather than defining a helper in one call and using it in another.

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
  # --- 1.3.0: cross-cutting labels (Step 6.0). Per-key defaults, because the
  #     whole-object `//=` above never touches a `sections` object that exists.
  #     The quality label enumerates all five keys, `framework` included; it is
  #     new in 1.3.0, so no released config carries an older value to migrate.
  (.analyze_skill.sections.cross_cutting_label_sk //= "Prierezové požiadavky") |
  (.analyze_skill.sections.cross_cutting_label_en //= "Cross-cutting requirements") |
  (.analyze_skill.sections.quality_label_sk //= "Kvalita (UI/UX, výkon, bezpečnosť, dostupnosť, framework)") |
  (.analyze_skill.sections.quality_label_en //= "Quality (UI/UX, performance, security, reachability, framework)") |
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
  # Booleans: `//=` would also replace a deliberate `false`, so a user could
  # never switch attachment downloads off — default them only when null.
  (.analyze_skill.fetch_attachments |= if . == null then true else . end) |
  (.analyze_skill.fetch_comment_files |= if . == null then true else . end) |
  (.analyze_skill.max_attachment_size_mb //= 25) |
  (.analyze_skill.attachment_extensions_allow //= [
    "md","txt","pdf","docx","xlsx","csv","json","html","eml","msg","sql","png","jpg","jpeg"
  ]) |
  (.analyze_skill.attachments_cleanup //= "after_run") |
  (.tasklist_filter //= {
    "enabled": true,
    "todo_stages": ["Ready for Development", "To Do"],
    "todo_stage_match_mode": "case_sensitive",
    "only_assigned_to_me": true,
    "analyze_all_tasks": true,
    "skip_reason_render": "inline"
  }) |
  # --- 1.3.0: start columns — the SAME rule as teamwork-task 1.5.0 Step 2.6,
  #     because both plugins write this shared key. `todo_stages` is written
  #     once, only when absent (or null) — an existing list is never
  #     overwritten: the default pair when the legacy `todo_stage` is missing,
  #     empty or the old default "To Do"; `[<legacy>]` when it was customised.
  #     The legacy key stays. `board_workflow` is not touched here — this
  #     skill never moves a card, and teamwork-task owns that key.
  (if (.tasklist_filter | type) == "object" and .tasklist_filter.todo_stages == null
   then .tasklist_filter.todo_stages =
          (.tasklist_filter.todo_stage as $legacy
           | if ($legacy | type) == "string" and $legacy != "" and $legacy != "To Do"
             then [$legacy]
             else ["Ready for Development", "To Do"] end)
   else . end)
' "$CONFIG_FILE" > "$CONFIG_FILE.tmp" && mv "$CONFIG_FILE.tmp" "$CONFIG_FILE"
chmod 600 "$CONFIG_FILE"
```

CLI flags override `.analyze_skill.*` values for the duration of this run.

**Shared keys (1.3.0).** `tasklist_filter` is shared with `teamwork-task`, and
both plugins may run first against the same file, so this merge only **adds**
what is missing and applies exactly the rule `teamwork-task` 1.5.0 applies —
the file ends in the same state whichever plugin runs first, and a value the
user chose is never rewritten. For the user's config shape
`{"todo_stage": "To Do", …}` that means `todo_stages` becomes
`["Ready for Development", "To Do"]`; `{"todo_stage": "Backlog"}` becomes
`["Backlog"]`; a fresh file gets the default pair.

---

## Step 3 — Fetch tasks via Teamwork REST API v3

Same auth pattern as `teamwork-task`:

```bash
TOKEN=$(jq -r '.teamwork.api_token' "$CONFIG_FILE")
BASE=$(jq  -r '.teamwork.base_url'  "$CONFIG_FILE")
AUTH="${TOKEN}:xxx"
```

Every GET below checks its HTTP status inline and parses the body with a
here-string — see the **Shell portability contract** at the top. Tool calls do
not share shell functions, so the check is repeated in each snippet. Variables
do not survive between calls either: run the three credential lines above (and
set `ENTITY_ID`) in the **same** Bash call as each snippet; the Step 3.2 filter
snippet also needs `PROJECT_ID` and `TASKS_FILE`, so run it together with the
tasklist snippet. In tasklist mode the Step 3.1 comments and subtasks snippets
run once per task, with `ENTITY_ID` set to that task's id.

### Step 3.1 — Single task (`URL_KIND=task`)

```bash
# --- Step 3.1: task ----------------------------------------------------------
URL="${BASE}/projects/api/v3/tasks/${ENTITY_ID}.json?include=projects,tasklists,attachments"
RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' "$URL")
HTTP=${RESP##*$'\n'}; BODY=${RESP%$'\n'*}
if [ "$HTTP" != "200" ]; then
  echo "⚠ GET $URL → HTTP $HTTP: $(jq -r '.errors[0].detail // .message // empty' <<<"$BODY" 2>/dev/null)" >&2
fi
# v3 task objects carry NO top-level `projectId` (and no `commentsCount`, no
# `hasSubtasks`) — the project id lives on the embedded tasklist reference.
PROJECT_ID=$(jq -r '.task.projectId // .task.tasklist.meta.projectId // empty' <<<"$BODY")
PROJECT_NAME=$(jq -r --arg p "$PROJECT_ID" '.included.projects[$p].name // empty' <<<"$BODY")
TASKLIST_ID=$(jq -r '.task.tasklistId // empty' <<<"$BODY")
# Task-level attachments: `.task.attachments` holds {id, type:"files"} refs and
# `include=attachments` resolves them into `.included.files` (the separate
# `/tasks/{id}/attachments.json` endpoint answers 404).
jq -r '.included.files // {} | to_entries[] | .value
       | [.id, .originalName, .size, .downloadURL] | @tsv' <<<"$BODY" > "/tmp/tw_analyze_files_${ENTITY_ID}.tsv"
echo "task ${ENTITY_ID}: project ${PROJECT_ID:-⚠ unknown} (${PROJECT_NAME:-?}), tasklist ${TASKLIST_ID:-?}, $(wc -l < "/tmp/tw_analyze_files_${ENTITY_ID}.tsv" | tr -d ' ') attachment(s)"
```

Extract from `.task` — these are the keys v3 actually returns:
- `id, name, description, tasklistId, estimateMinutes, priority, status, dueDate, parentTaskId, assigneeUserIds, tagIds, workflowStages, attachments`
- **`projectId` = `.projectId // .tasklist.meta.projectId`.** v3 has no
  top-level `projectId` (it reads `null`); the value sits on the embedded
  tasklist reference. The project name comes from `.included.projects["<id>"].name`.
  Both feed the final report (Step 10) — the project link there is built from
  this id.
- There is **no `commentsCount`** and **no `hasSubtasks`** — nothing may be
  gated on either. The comments and subtasks fetches below answer those
  questions directly.

**Always fetch the comments** (v1.3.0: no count gate — v3 never returns
`commentsCount`, so a `commentsCount > 0` gate never opened, and the reporter's
clarifications in the comments are half the input to Step 6). Chronological
order matters because the latest comment usually wins:
`orderBy=date&orderMode=asc` (`orderBy=postedAt` is rejected with HTTP 400
*"orderBy: unknown comment sort."*).

```bash
# --- Step 3.1: comments (v3, v1 fallback) ------------------------------------
# One comment per line, oldest first, normalised to the v3 field names:
# {id, postedDateTime, postedByUserId, author, htmlBody, files}.
COMMENTS_FILE="/tmp/tw_analyze_comments_${ENTITY_ID}.jsonl"; : > "$COMMENTS_FILE"
COMMENTS_SRC=v3; PAGE=1
while :; do
  URL="${BASE}/projects/api/v3/tasks/${ENTITY_ID}/comments.json?pageSize=100&page=${PAGE}&orderBy=date&orderMode=asc&include=users"
  RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' "$URL")
  HTTP=${RESP##*$'\n'}; BODY=${RESP%$'\n'*}
  if [ "$HTTP" != "200" ] || ! jq -c '(.included.users // {}) as $u | .comments[]
        | {id, postedDateTime, postedByUserId,
           author: (($u[(.postedByUserId|tostring)] // {}) | [.firstName, .lastName] | map(select(. != null and . != "")) | join(" ")),
           htmlBody, files: (.files // [])}' <<<"$BODY" >> "$COMMENTS_FILE"; then
    echo "⚠ GET $URL → HTTP $HTTP: $(jq -r '.errors[0].detail // .message // empty' <<<"$BODY" 2>/dev/null) — falling back to the v1 comments endpoint" >&2
    COMMENTS_SRC=v1; break
  fi
  [ "$(jq -r '.meta.page.hasMore // false' <<<"$BODY")" = "true" ] || break
  PAGE=$((PAGE + 1))
done

if [ "$COMMENTS_SRC" = "v1" ]; then
  # v1 → v3 field map: datetime → postedDateTime · html-body (plain: body) →
  # htmlBody · author-id → postedByUserId · author-firstname/-lastname → author ·
  # attachments → files. v1 server order is unspecified — sort by datetime.
  # Page until an EMPTY page: v1 may cap pageSize below the 100 requested, so a
  # short page does not prove it was the last one.
  : > "$COMMENTS_FILE"; PAGE=1
  while :; do
    URL="${BASE}/tasks/${ENTITY_ID}/comments.json?page=${PAGE}&pageSize=100"
    RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' "$URL")
    HTTP=${RESP##*$'\n'}; BODY=${RESP%$'\n'*}
    N=$(jq -r '.comments | length' <<<"$BODY" 2>/dev/null)
    if [ "$HTTP" != "200" ] || [ -z "$N" ] || ! jq -c '.comments[]
        | {id: (.id|tonumber? // .), postedDateTime: .datetime, postedByUserId: (."author-id"|tonumber? // .),
           author: ([."author-firstname", ."author-lastname"] | map(select(. != null and . != "")) | join(" ")),
           htmlBody: (."html-body" // .body), files: (.attachments // [])}' <<<"$BODY" >> "$COMMENTS_FILE"; then
      echo "⚠ GET $URL → HTTP $HTTP — comments UNAVAILABLE for task ${ENTITY_ID}; report them as unavailable, never as \"no comments\"" >&2
      COMMENTS_SRC=unavailable; : > "$COMMENTS_FILE"; break
    fi
    [ "$N" -eq 0 ] && break
    PAGE=$((PAGE + 1))
  done
  [ "$COMMENTS_SRC" = "v1" ] && jq -sc 'sort_by(.postedDateTime)[]' "$COMMENTS_FILE" > "$COMMENTS_FILE.tmp" && mv "$COMMENTS_FILE.tmp" "$COMMENTS_FILE"
fi
if [ "$COMMENTS_SRC" = "unavailable" ]; then
  echo "task ${ENTITY_ID}: comments UNAVAILABLE"
else
  echo "task ${ENTITY_ID}: $(wc -l < "$COMMENTS_FILE" | tr -d ' ') comment(s) via ${COMMENTS_SRC}"
fi
```

| Datum | v3 comment | v1 comment |
| --- | --- | --- |
| time | `postedDateTime` (there is **no** `postedAt`) | `datetime` |
| HTML body | `htmlBody` | `html-body` (plain: `body`) |
| author | `postedByUserId` (id only; names via `include=users` → `.included.users`) | `author-id`, `author-firstname`, `author-lastname` |
| files | `files[]` — attachments uploaded with the comment | `attachments[]` |

Resolve a comment file reference to its metadata (`originalName`, `size`,
`downloadURL`) with `GET /projects/api/v3/files/{id}.json` → `.file`, using the
same inline status check. `COMMENTS_SRC=unavailable` is a different result from
an empty file: Step 6 and the final report say *"comments unavailable (HTTP …)"*,
never *"no comments"* — the two read the same otherwise, and only one of them is
true.

Fetch subtasks (one-shot, recursion via `parentTaskId` chain happens in Step 7),
with the same inline status check. The fallback, when the `/subtasks.json`
endpoint is unavailable on the workspace, is the task list filtered by
`parentTaskId` (singular). `parentTaskIds` is silently **ignored** by v3, which
then returns unrelated tasks from the whole site, so both results are also
filtered on `.parentTaskId` before anything counts as a subtask:

```bash
# --- Step 3.1: subtasks (v3 subtasks endpoint, filtered task list fallback) --
SUBTASKS_FILE="/tmp/tw_analyze_subtasks_${ENTITY_ID}.jsonl"; : > "$SUBTASKS_FILE"
SUBTASKS_SRC=unavailable
for URL in "${BASE}/projects/api/v3/tasks/${ENTITY_ID}/subtasks.json?pageSize=100&page=1" \
           "${BASE}/projects/api/v3/tasks.json?parentTaskId=${ENTITY_ID}&pageSize=100&page=1"; do
  RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' "$URL")
  HTTP=${RESP##*$'\n'}; BODY=${RESP%$'\n'*}
  if [ "$HTTP" = "200" ] && jq -c --argjson p "$ENTITY_ID" '.tasks[] | select(.parentTaskId == $p)' \
       <<<"$BODY" > "$SUBTASKS_FILE"; then
    SUBTASKS_SRC=ok
    [ "$(jq -r '.meta.page.hasMore // false' <<<"$BODY")" = "true" ] \
      && echo "⚠ GET $URL → more than 100 subtasks; only the first 100 are reviewed" >&2
    break
  fi
  echo "⚠ GET $URL → HTTP $HTTP: $(jq -r '.errors[0].detail // .message // empty' <<<"$BODY" 2>/dev/null)" >&2
  : > "$SUBTASKS_FILE"
done
if [ "$SUBTASKS_SRC" = "unavailable" ]; then
  echo "task ${ENTITY_ID}: subtasks UNAVAILABLE"
else
  echo "task ${ENTITY_ID}: $(wc -l < "$SUBTASKS_FILE" | tr -d ' ') open subtask(s)"
fi
```

Both endpoints list open subtasks only (completed ones need
`includeCompletedTasks=true`). If both fail (`SUBTASKS_SRC=unavailable`), the
subtask review (Step 7) reports *"subtasks unavailable"* and does **not** treat
the task as having none — Step 7.2 would otherwise propose splitting a task that
already is split.

### Step 3.2 — Tasklist (`URL_KIND=tasklist`)

```bash
# --- Step 3.2: tasklist ------------------------------------------------------
URL="${BASE}/projects/api/v3/tasklists/${ENTITY_ID}.json"
RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' "$URL")
HTTP=${RESP##*$'\n'}; BODY=${RESP%$'\n'*}
if [ "$HTTP" != "200" ]; then
  echo "⚠ GET $URL → HTTP $HTTP: $(jq -r '.errors[0].detail // .message // empty' <<<"$BODY" 2>/dev/null)" >&2
fi
PROJECT_ID=$(jq -r '.tasklist.projectId // empty' <<<"$BODY")

# One task per line (compact JSON) so later snippets can read it without echo.
TASKS_FILE="/tmp/tw_analyze_tasks_${ENTITY_ID}.jsonl"; : > "$TASKS_FILE"
PAGE=1
while :; do
  URL="${BASE}/projects/api/v3/tasklists/${ENTITY_ID}/tasks.json?pageSize=100&page=${PAGE}"
  RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' "$URL")
  HTTP=${RESP##*$'\n'}; BODY=${RESP%$'\n'*}
  if [ "$HTTP" != "200" ] || ! jq -c '.tasks[]' <<<"$BODY" >> "$TASKS_FILE"; then
    echo "⚠ GET $URL → HTTP $HTTP: $(jq -r '.errors[0].detail // .message // empty' <<<"$BODY" 2>/dev/null) — the task list is INCOMPLETE from page ${PAGE} on" >&2
    break
  fi
  [ "$(jq -r '.meta.page.hasMore // false' <<<"$BODY")" = "true" ] || break
  PAGE=$((PAGE + 1))
done
# The tasklist GET failed or carried no project id → take it from the tasks
# themselves (v3 task objects: `.tasklist.meta.projectId`; no `projectId` key).
if [ -z "$PROJECT_ID" ] && [ -s "$TASKS_FILE" ]; then
  PROJECT_ID=$(jq -rs '[.[] | .projectId // .tasklist.meta.projectId // empty][0] // empty' "$TASKS_FILE")
  [ -n "$PROJECT_ID" ] && echo "ℹ project id ${PROJECT_ID} taken from the tasks (tasklist GET gave none)" >&2
fi
echo "tasklist ${ENTITY_ID}: $(wc -l < "$TASKS_FILE" | tr -d ' ') task(s), project ${PROJECT_ID:-⚠ unknown}"
```

A tasklist item has the same shape as a single task — take each task's own
project id from `.projectId // .tasklist.meta.projectId`, as in Step 3.1.

**Apply the `tasklist_filter` from config** — same rules as `teamwork-task` v1.5.0:

- Unless `.tasklist_filter.enabled` is `false` (a missing key counts as
  `true` — the same reading as `teamwork-task`):
  - Resolve `me.id` from `GET /projects/api/v3/me.json` → `.person.id`.
  - Resolve each task's board column from its **workflow stage**:
    `.workflowStages[0].stageId` → the stage name from the project's workflows.
    (Up to 1.2.0 this step read `.cardId` → `included.cards` → `included.stages`
    from `?include=cards,stages`. On the live API `cardId` is `null` and that
    `included` comes back empty, so every task resolved to "no column" and was
    silently listed read-only.)
  - **Start columns** *(1.3.0)* — a task is in a start column when its stage
    name equals **any** entry of `.tasklist_filter.todo_stages` (default
    `["Ready for Development", "To Do"]` — the two start columns of the WAME
    board, workflow "WAME workflow"; an older per-project board has only
    `To Do`, which still matches). The list order is display priority only.
    The legacy string `.tasklist_filter.todo_stage` is still read when
    `todo_stages` is absent or empty. `todo_stage_match_mode` (default
    `case_sensitive`) applies to each column name: `to do` or `TO DO` does not
    match `To Do` unless the mode is `case_insensitive`.
  - Tasks matching **both** a start column **AND** `me.id ∈ task.assigneeUserIds`
    (the assignee half only when `only_assigned_to_me` is not `false`) are
    marked `eligible_for_writeback = true`.
  - All other tasks are still fetched and **analysed read-only** (so the user can sanity-check teammates' work) — they appear in the final report with a `📋 read-only (<reason>)` badge and are **never** PATCHed.
  - A task that is **not on the board** (`workflowStages` empty, or `stageId`
    `0`) is never eligible — only the configured start columns are. It is
    listed read-only with the reason *"not on the board"* rather than an empty
    stage. A task that **is** on a board whose column name could not be
    resolved (the workflows GET failed, or the project id is unknown) is
    listed read-only as *"column unresolved"* — fail closed, never a guess.

```bash
# --- Step 3.2: tasklist filter (board column + assignee) ---------------------
# stageId (string) → stage name; `.included.stages` is an object keyed by id
# (tolerate an array too). An empty PROJECT_ID would make `projectIds=` answer
# 200 with the whole workspace's workflows — never ask without one.
STAGES='{}'
if [ -z "$PROJECT_ID" ]; then
  echo "⚠ project id unknown — board columns UNKNOWN; every task on a board is listed read-only (column unresolved)" >&2
else
  URL="${BASE}/projects/api/v3/workflows.json?projectIds=${PROJECT_ID}&include=stages"
  RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' "$URL")
  HTTP=${RESP##*$'\n'}; BODY=${RESP%$'\n'*}
  if [ "$HTTP" != "200" ] || ! STAGES=$(jq -c '.included.stages // {}
        | if type == "array" then map({key: (.id|tostring), value: .name}) | from_entries
          else map_values(.name) end' <<<"$BODY"); then
    echo "⚠ GET $URL → HTTP $HTTP — board columns UNKNOWN; every task on a board is listed read-only (column unresolved)" >&2
    STAGES='{}'
  fi
fi

URL="${BASE}/projects/api/v3/me.json"
RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' "$URL")
HTTP=${RESP##*$'\n'}; BODY=${RESP%$'\n'*}
ME_ID=$(jq -r '.person.id // empty' <<<"$BODY" 2>/dev/null)
[ -n "$ME_ID" ] || echo "⚠ GET $URL → HTTP $HTTP — current user UNKNOWN; every task is listed read-only and the report says why" >&2

# Start columns: `todo_stages` (array; absent or empty → the legacy string
# `todo_stage`), else the default pair — the same reading as teamwork-task. A
# string written where the list belongs counts as a one-column list.
# `// true` would turn an explicit `false` into true — so test for `false`.
START_STAGES=$(jq -c '.tasklist_filter as $f
  | ($f.todo_stages | if type == "string" then [.] elif type == "array" then . else [] end
     | map(select(type == "string" and length > 0))) as $list
  | if ($list | length) > 0 then $list
    elif ($f.todo_stage | type) == "string" and $f.todo_stage != "" then [$f.todo_stage]
    else ["Ready for Development", "To Do"] end' "$CONFIG_FILE")
MATCH_MODE=$(jq -r '.tasklist_filter.todo_stage_match_mode // "case_sensitive"' "$CONFIG_FILE")
ONLY_MINE=$(jq -r 'if .tasklist_filter.only_assigned_to_me == false then "false" else "true" end' "$CONFIG_FILE")

# taskId <TAB> stage ("-" = not on the board, "?<id>" = column unresolved)
#        <TAB> assignee ids ("-" = none) <TAB> "eligible" | "read_only:<reasons>"
jq -r --argjson st "$STAGES" --argjson start "$START_STAGES" --arg mode "$MATCH_MODE" \
      --arg me "$ME_ID" --arg mine "$ONLY_MINE" '
  def key: if $mode == "case_insensitive" then ascii_downcase else . end;
  (((.workflowStages // [])[0].stageId // 0) | tostring) as $sid
  | (if $sid == "0" then "-" else ($st[$sid] // ("?" + $sid)) end) as $stage
  | ((.assigneeUserIds // []) | map(tostring)) as $who
  | [ (if $stage == "-" then "no_card"
       elif ($stage | startswith("?")) then "stage_unresolved(\($sid))"
       elif any($start[]; key == ($stage | key)) then empty
       else "wrong_stage(\($stage))" end),
      (if $mine != "true" then empty
       elif $me == "" then "me_unknown"
       elif any($who[]; . == $me) then empty
       else "wrong_assignee" end)
    ] as $why
  | [ .id, $stage, (if $who == [] then "-" else ($who | join(",")) end),
      (if $why == [] then "eligible" else "read_only:" + ($why | join("+")) end) ] | @tsv' \
  "$TASKS_FILE" > "/tmp/tw_analyze_stages_${ENTITY_ID}.tsv"
echo "start columns ${START_STAGES} (${MATCH_MODE}): $(awk -F '\t' '$4 == "eligible" { e++ } $4 != "eligible" { r++ }
  END { print e+0 " eligible, " r+0 " read-only" }' "/tmp/tw_analyze_stages_${ENTITY_ID}.tsv")"
```

The fourth column decides `eligible_for_writeback`. The reason codes are the
ones `teamwork-task`'s Step 3.45 uses, and they render in the report as:
`no_card` → *"not on the board"*, `stage_unresolved(<id>)` → *"column
unresolved"*, `wrong_stage(<stage>)` → *"column <stage> is not a start column"*,
`wrong_assignee` → *"not assigned to me"*, `me_unknown` → *"current user
unknown"* (analysis-only here: without a known user nothing is written back).

- By default skip tasks where `status == "completed"` unless `--include-completed` was passed.

### Step 3.3 — Download attachments + comment files

Only when `.analyze_skill.fetch_attachments == true` (and `.fetch_comment_files`
for comment-attached files). For each file:

- Skip if `size_mb > max_attachment_size_mb` (default 25); record `[SKIPPED: too large]` in the analysis log.
- Skip if extension not in `attachment_extensions_allow`.
- Take the metadata (`originalName`, `size` in bytes, `downloadURL`) from
  `/tmp/tw_analyze_files_<taskId>.tsv` (Step 3.1, `include=attachments`) for task
  files, and from `GET /projects/api/v3/files/{id}.json` → `.file` for a comment's
  `files[]` reference.
- Download via the `downloadURL` field from the file metadata (`curl -sS -u "$AUTH" -L -o "$TMPDIR/<safe-name>"`, path quoted).
- Convert binary docs to plain text:
  - `.docx` / `.pdf` / `.xlsx` — use whichever converter is on `PATH` (`pandoc`, `pdftotext`, `python3 -m pdfminer.high_level`, etc.). If none is available → record `[SKIPPED: no converter]` and continue.
  - `.md` / `.txt` / `.csv` / `.json` / `.html` — read as-is.
  - `.png` / `.jpg` / `.jpeg` — leave on disk and reference by path in the analysis (the LLM can view them). That local path is for **your own reasoning only**: `attachments_cleanup: after_run` deletes the directory when the run ends, so a path written into the description is a dead link before the user reads the report. Anything that goes into the task keeps the original Teamwork URL.

When the run ends, honour `attachments_cleanup` (`after_run` → `rm -rf "$TMPDIR/teamwork-task-analyze-<run-id>"`, and the Step 3 scratch files with `find /tmp -maxdepth 1 -name 'tw_analyze_*_<id>.*' -delete` — not a bare `/tmp/tw_analyze_*` glob, which is a hard error in zsh when nothing matches).

### Step 3.4 — HTTP error handling

Every snippet in Step 3 captures the status with `-w '\n%{http_code}'` and prints
a `⚠` naming the endpoint — a failed request must never look like an empty
result.

- HTTP 401 → token invalid; re-prompt the user (re-run Step 2's first-run flow).
- HTTP 403/404 on the **task or tasklist itself** → report and stop with the URL that failed.
- HTTP 4xx/5xx on a **secondary** fetch (comments, subtasks, workflows, a comment
  file) → follow that snippet's stated fallback, and carry the gap into the final
  report as *"unavailable"*, never as *"none"*.
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

**Installed framework versions (the `framework` plan line, Step 6.0).** When the
skill runs inside the project's repository, read the versions it actually has
installed — from the lock files, never from memory, because a model's idea of
"the current Laravel" is stale by design. With Laravel Boost installed
(`laravel/boost`), its `application-info` tool gives them directly. Run once per
run and reuse the result for every task:

```bash
# --- Step 4: installed framework versions ------------------------------------
# PHP side: resolved versions from composer.lock (else vendor/composer/installed.json),
# the PHP floor from composer.json. A package repo without either prints its
# composer.json constraints and says that they are constraints.
PHP_PKGS='^(laravel/(framework|nova|boost)|livewire/livewire|inertiajs/inertia-laravel|pestphp/pest|phpunit/phpunit)$'
if [ -f composer.lock ]; then
  jq -r --arg re "$PHP_PKGS" '.packages[]?, ."packages-dev"[]? | select(.name | test($re))
                              | "\(.name) \(.version)"' composer.lock
elif [ -f vendor/composer/installed.json ]; then
  jq -r --arg re "$PHP_PKGS" '(.packages? // .)[] | select(.name | test($re))
                              | "\(.name) \(.version)"' vendor/composer/installed.json
elif [ -f composer.json ]; then
  jq -r --arg re "$PHP_PKGS" '(.require // {}) + (."require-dev" // {}) | to_entries[]
                              | select(.key | test($re))
                              | "\(.key) \(.value) (constraint only, not installed)"' composer.json
fi
if [ -f composer.json ]; then
  jq -r '"php " + ((.config.platform.php // .require.php // "?") | tostring)' composer.json
fi

# JS side: the version installed in node_modules (works for npm, pnpm and yarn),
# else the one resolved in package-lock.json; without either print the
# package.json constraint and say that it is one.
if [ -f package.json ]; then
  while IFS= read -r P; do
    [ -z "$P" ] && continue
    V=""
    if [ -f "node_modules/$P/package.json" ]; then
      V=$(jq -r '.version // empty' "node_modules/$P/package.json")
    elif [ -f package-lock.json ]; then
      V=$(jq -r --arg p "$P" '.packages["node_modules/" + $p].version // .dependencies[$p].version // empty' package-lock.json)
    fi
    if [ -n "$V" ]; then
      printf '%s %s\n' "$P" "$V"
    else
      printf '%s %s (constraint only, not installed)\n' "$P" \
        "$(jq -r --arg p "$P" '.dependencies[$p] // .devDependencies[$p] // "?"' package.json)"
    fi
  done < <(jq -r '(.dependencies // {}) + (.devDependencies // {}) | keys[]
                  | select(test("^(vue|react|nuxt|vite|typescript|tailwindcss|@tailwindcss/.+|@inertiajs/.+|@ionic/.+)$"))' package.json)
  # The target browsers decide which CSS / JS features are safe to plan with.
  if [ -f .browserslistrc ]; then
    printf 'browserslist %s\n' "$(grep -vE '^[[:space:]]*(#|$)' .browserslistrc | paste -sd ',' -)"
  else
    jq -r '"browserslist " + ((.browserslist // "defaults (none configured)")
           | if type == "object" then (.production // (to_entries[0].value) // []) else . end
           | if type == "array" then join(",") else tostring end)' package.json
  fi
  jq -r '"node (engines) " + (.engines.node // "?")' package.json
fi
if [ -f .nvmrc ]; then printf 'node (.nvmrc) %s\n' "$(cat .nvmrc)"; fi
```

A constraint printed as *"constraint only"* is a floor, not a version — plan
only with features that floor already has. PHP's `config.platform.php` /
`require.php` is the lowest version the code must run on, and the browserslist
target plays the same role for CSS and JS. When the skill does **not** run
inside the project's repository (no `composer.json`, no `package.json`), there
are no versions — Step 6.0 then writes one generic line and never a guessed
version number.

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
  `<ul>` / `<li>` / `<input type="checkbox">` as markdown bullets / checkboxes,
  **and every `<img src="…">` as `![alt](src)` — never drop it.** A screenshot
  pasted into a Teamwork description is an *inline image*
  (`https://tw-inlineimages.s3-accelerate.amazonaws.com/…`), not an attachment:
  it is stored nowhere else in the task, so stripping the tag deletes the
  screenshot. The same goes for `<a href>` links, `![…](…)` markdown images
  already in the body, and any `[filename](…)` attachment reference — carry
  every one of them through unchanged, URL included.
- Detect known H2 headings case-insensitively, sk **and** en variants:
  - Acceptance: `Akceptačné kritériá`, `Akceptacne kriteria`, `Acceptance criteria`, `Acceptance Criteria`
  - Goal: `Cieľ`, `Ciel`, `Goal`, `Final summary`
  - Tech: `Technický popis`, `Technicky popis`, `Technical plan`, `Technical description`, `Implementation plan`
- Detect HR markers: `<hr>`, `<hr/>`, `<hr />`, or markdown `---` / `***` / `___` on its own line.
- Build a map `{ preamble, acceptance, goal, tech, leftovers }`:
  - Text **before** the first recognised heading **and** before the first HR → `preamble`.
  - Text under each recognised heading → its section, stopping at the next HR or H2.
  - Text under any **unrecognised** heading, and any block that belongs to no
    section at all → `leftovers`, verbatim. This is where a screenshot pasted at
    the bottom of the task lives; a bucket that Step 6 does not carry forward is
    a bucket that deletes it.
  - If a section is missing, store an empty string for it (will be filled in Step 6).
- **Nothing in the original description may be dropped by this parser.** Every
  character of the original ends up in exactly one bucket. Before Step 6, assert
  that every `![…](…)`, `<img …>` and `<a href=…>` found in the original appears
  in one of them; if one does not, stop and report it instead of rewriting the
  task.
- If the description has **no recognised structure at all** (no headings, no HR),
  store the whole thing as `preamble` so that nothing is lost when we rewrite.
  This is the **narrow fallback** for one shape, not the no-loss guarantee — that
  is the unconditional rule stated above, and it applies to every shape.

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
  and its installed framework versions

Produce a **draft** new description with this exact structure:

```
{preamble — copied verbatim if it existed, including every inline image (`![…](…)` / `<img …>`), link and attachment reference; otherwise omitted entirely (no empty preamble)}

---

## {acceptance_label}
- [ ] AC 1 — atomic, testable, written from the user's perspective
- [ ] AC 2 — …
- [ ] AC 3 — …
…

### {cross_cutting_label}
- [ ] only the cross-cutting criteria that apply (Step 6.0), one per line
- [ ] …

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

### {quality_label}
Only the lines that apply (Step 6.0), each naming the concrete thing:
- Menu: section, label, roles, and the file that registers it
- Inbound links: parent screen → relation field / tab, action, button, breadcrumb
- Authorization: policy class + method, gate, tenant / company scope
- Performance: eager loads, pagination, indexes on new FK / filter / sort columns,
  `chunkById` for batches and data migrations, the row count where it starts to hurt
- UI/UX: the sibling screen to copy, new translation keys (module lang file,
  `hint.<thing>` not `<thing>.hint`), disabled / empty / error / loading states
- Framework: the installed version and the idiomatic feature of that version to
  use for this task, one line each — e.g. *Laravel 12.28: použiť `casts()` +
  enum cast pre `status`*, *Vue 3.5: `defineModel()` namiesto `modelValue` +
  `emit`* — with the docs page checked when the choice is not obvious

### Open questions
- Anything the analyst (you) could not answer with confidence — marked with
  the `[OTVORENÉ]` marker from config so the implementer treats them as
  blockers.

{leftovers — every part of the original description that matched no section
 (including screenshots and links pasted at the bottom), appended verbatim under
 its original heading; omitted entirely when empty}
```

Use `{acceptance_label}` / `{goal_label}` / `{tech_label}` /
`{cross_cutting_label}` / `{quality_label}` from `.analyze_skill.sections` based
on the resolved language (CLI flag wins, else config `default_language`,
fallback `sk`). Omit the `{cross_cutting_label}` sub-heading and the
`{quality_label}` subsection entirely when no dimension applies.

### Step 6.0 — Cross-cutting requirements (four dimensions in the AC, `framework` in the plan)

Acceptance criteria written from the reporter's text say what the reporter
thought to ask for. They are silent about the menu entry nobody added, the page
only reachable by typing its URL, the new action any tenant can trigger, and the
list that runs one query per row. `teamwork-task-test` checks exactly these at
QA time — its **Step 6.6** reviews the dimensions `ui_ux`, `performance`,
`security` and `reachability` (plus `framework`, which it treats as advisory —
see the end of this step). Writing the applicable ones into the AC here
means the implementer builds them in the first place and the tester can tick
them, instead of reporting them as findings afterwards.

For each dimension decide whether it **applies**. If it does, add concrete,
testable `- [ ]` criteria under `### {cross_cutting_label}` in the task's
language. Slovak examples follow; `<…>` stands for the real name:

| Dimension | Applies when the task … | Criteria (examples) |
| --- | --- | --- |
| `reachability` | adds, renames or removes a screen — page, route rendering a view, Nova resource / lens / dashboard / tool, SPA route | `- [ ] Nová obrazovka <X> je dostupná z menu <sekcia> pre rolu <rola>` · `- [ ] Z detailu <rodič> vedie na <X> preklik (záložka vzťahu / tlačidlo)` · on a rename or removal: `- [ ] Žiadna položka menu ani odkaz nevedie na zrušenú obrazovku <X>` |
| `security` | adds a route, action, button, endpoint or form | `- [ ] Akciu <akcia> spustí len rola <rola> a len nad záznamami vlastnej firmy — iný používateľ dostane 403, nie chybu 500` · `- [ ] Rola bez oprávnenia položku <X> v menu nevidí a priamy prístup na jej URL dostane 403` · `- [ ] Neplatný vstup vo formulári <X> vráti validačnú chybu pri poli, nie chybu 500` |
| `performance` | adds or changes a list, report, export, batch job or data migration | `- [ ] Zoznam <X> s <N> záznamami sa načíta do <T> s — stránkovaný, bez dotazu na každý riadok` · `- [ ] Migrácia / dávka <X> spracuje <N> záznamov po častiach, bez načítania celej tabuľky do pamäte` |
| `ui_ux` | adds or changes UI | `- [ ] Neaktívne tlačidlo <X> ukáže dôvod (tooltip) a čítačka obrazovky ho ohlási ako neaktívne` · `- [ ] Všetky nové texty sú preložené — na obrazovke sa nezobrazí surový prekladový kľúč` · `- [ ] Prázdny zoznam aj chyba načítania <X> zobrazia zrozumiteľnú správu` · `- [ ] Zmazanie <X> vyžaduje potvrdenie` |

Rules:

- **Only dimensions that apply.** A pure backend task without UI gets no UI
  criteria; a task that changes nothing a user or an attacker can reach may get
  none at all — then omit the sub-heading and say *"cross-cutting: none
  applicable — <reason>"* in the final report. Proportion beats boilerplate.
- **Concrete or not at all.** Name the real menu section, role, parent screen,
  action and data volume — take them from the repo scan (Step 4): the existing
  menu registration, the sibling resources' policies, the parent model's
  relations. *"Stránka je dostupná"* cannot be tested; *"Nová obrazovka Faktúry
  je dostupná z menu Účtovníctvo pre rolu Účtovník"* can.
- **A deliberately URL-only page** (e-mail deep link, landing page) gets a
  criterion that says so — `- [ ] Stránka <X> je zámerne dostupná len cez odkaz
  z e-mailu, nie z menu` — so the tester knows to put it into
  `teamwork-task-test`'s `reachability.allow_orphans` instead of reporting it.
- **Unknowns are clarifying questions** (Step 6.1). When the menu placement, the
  roles or the expected data volume cannot be inferred, ask — inside the same
  batch-of-4 / max-6 budget, never as an extra round. If the answer does not
  come, keep the criterion and mark the unknown part with the open marker:
  `- [ ] Nová obrazovka <X> je dostupná z menu [OTVORENÉ] pre rolu [OTVORENÉ]`.
- **Format — the QA side depends on it.** `teamwork-task-test` extracts
  criteria from the AC block (its Step 4.1) and ticks them by an exact `- [ ] …`
  line match that stops at the next `---` (its Step 9.3). So the
  `### {cross_cutting_label}` sub-heading sits **inside** the AC section, after
  the task-specific criteria and before the `---`; every criterion is a single
  `- [ ] …` line — no nesting, no line wrap, no `*` bullets; and there is never
  a `---` inside the AC block.
- **The additive contract still holds.** The cross-cutting criteria are new text
  in the AC section below the reporter's verbatim preamble; nothing above the
  first `---` changes. A criterion the reporter already wrote is not repeated.
  No estimate goes here — or anywhere in the description. The preview and
  confirmation gate in Step 6.4 cover these lines like every other.

The technical plan gets the matching **`### {quality_label}`** subsection (see
the template): a few lines that name the menu location, the inbound links, the
policies and gates, the eager loads and indexes — the *how* behind each
cross-cutting criterion. Only the lines that apply.

**The fifth key, `framework` — plan only, never an acceptance criterion.** The
implementer should build with the idioms and built-in features of the framework
versions the project actually has installed, not with patterns remembered from
older versions or hand-rolled code for something the framework ships. So the
`{quality_label}` subsection gets a *Framework* line for each idiom that
matters to this task, naming the installed version from Step 4 and the concrete
feature — *"Laravel 12.28: použiť `casts()` + enum cast pre `status`"*,
*"Nova 5: `dependsOn()` na poli `typ` namiesto vlastného JS"*, *"Tailwind 4:
nové tokeny cez `@theme` v CSS, nie `tailwind.config.js`"*. It is **never** a
`- [ ]` criterion: `teamwork-task-test` treats `framework` as advisory
(recommendations only), so such a criterion could never be ticked, and the
acceptance criteria describe behaviour, not idiom. Rules for the line:

- **Versions from Step 4, idioms from current docs.** Look the feature up for
  the installed version — Laravel Boost `search-docs` when the project has
  `laravel/boost`, otherwise the context7 MCP (`resolve-library-id` →
  `query-docs`), otherwise the official docs and the upgrade guide of that
  major. Never plan with a feature newer than the installed version or the
  browserslist target, and never with an API that version deprecates.
- **Consistency beats novelty.** The project's `CLAUDE.md` and the conventions
  of the sibling code win over a newer idiom; do not plan a second pattern next
  to an established one unless the task is a refactor. No new dependency when
  the framework already ships the capability.
- **Only the code this task adds or changes.** No drive-by rewrites of untouched
  code — an opportunity there is at most a line under *Out of scope*.
- **Proportionate.** One to three lines; none when the task writes no code (a
  pure content or configuration change).
- **Outside the project's repository** (Step 4 found no `composer.json` and no
  `package.json`) write one generic line instead — *"Framework: pri
  implementácii použiť idiómy nainštalovaných verzií (overiť v `composer.lock`
  / `package.json`)"* — and never a guessed version number.

### Step 6.1 — Gather clarifying questions

While producing the draft, keep a list of **clarifying questions** for which:

- The answer materially changes AC, the goal, or the technical plan.
- You cannot infer the answer with reasonable confidence from the task,
  comments, attachments, and the repo context.

Cap the list at `.analyze_skill.clarifying_questions.max_per_run` (default 6).
Prioritise: AC ambiguities (including the menu placement, roles and expected
data volume a cross-cutting criterion needs — Step 6.0) → data model unknowns →
integration points → UI/UX details. Cross-cutting questions share this budget;
they never open an extra round.

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
re-format the preamble — preserve line breaks, lists, links, inline images
(`![…](…)` and `<img …>`, including the full `tw-inlineimages` URL) and
attachment references exactly, character for character. Do not re-spell it, do
not add or remove diacritics, do not translate it, do not "tidy" or normalise
it: the reporter's own words are a record of what was asked for, and a rewritten
record is a lost one. If `parsed.leftovers` is non-empty, append it verbatim
after the technical section under its original heading — unrecognised is not the
same as unwanted.

### Step 6.4 — Render the preview

Render the new description as fenced markdown and show it to the user. Compute
a unified diff against the original description (`diff -u <(printf '%s' "$old")
<(printf '%s' "$new")`) and show the first 80 lines of it.

The 80-line cap must never hide a removal. Above the diff, list **in full**
every `-` line that contains an image (`![…](…)`, `<img …>`), a link or an
attachment reference, however far down the description it sits. If that list is
non-empty, the default answer is `Edit`, not `Apply` — a screenshot deleted from
a Teamwork description cannot be recovered.

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
     structure) and queue it as a write proposal. Step 6.3 applies to subtasks
     too: the subtask's own preamble, its inline images and its links are copied
     into the new description verbatim — never dropped, never reformatted. So
     does Step 6.0: the updated subtask carries the cross-cutting criteria that
     apply to its own slice of the work.
4. When `review_existing == true` (default), present each proposed subtask
   update via **AskUserQuestion** (`Apply` / `Edit` / `Skip`).

### Step 7.2 — Proposing a split for big monolithic tasks

Only when **all** of the following are true:

- The task has **no** subtasks — the Step 3.1 subtasks fetch succeeded and
  returned an empty list. (v3 has no `hasSubtasks` key, so do not gate on it; a
  subtasks fetch that failed means *unknown*, not *none*, and blocks the split.)
- The proposed estimate from Step 8 exceeds `propose_split_threshold_minutes`
  (default 240).
- The AC list contains at least `min_split_subtasks` items that fall into
  **naturally separable** groups (e.g. different layers — BE / FE / migration;
  different components; clearly independent flows).

If so, draft a split of 2–6 atomic subtasks. Each subtask gets:
- Name (short, business-friendly, prefixed with `[BE]` / `[FE]` / `[QA]` / …
  if the project uses role tags)
- Its own canonical description (preamble omitted; AC + Goal + Technical plan),
  including the Step 6.0 cross-cutting criteria for **its own slice** — the menu
  entry and inbound links go to the subtask that builds the screen, the
  authorization criterion to the one that adds the action or endpoint
- Its own estimate (use the methodology from Step 8), written on the POST as
  `estimatedMinutes` — the v3 **write** field; `estimateMinutes` is the read
  field and lands as 0 on a POST
- Optional `parentTaskId` set to the current task

**When the split is applied, the parent's own estimate goes to 0.** Teamwork rolls
subtask estimates up into the parent's `accumulatedEstimatedMinutes` and adds the
parent's own field on top; a parent that keeps the total reports the work twice
(`estimateMinutes: 480` + four subtasks = `accumulatedEstimatedMinutes: 960`,
observed 2026-10-08). Step 8 then sizes the subtasks, and Step 9.2 PUTs `0` on
the parent.

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

**The estimate lives in the Teamwork estimate field and nowhere else.** Never
write the number of minutes or hours into the task **title** or into the task
**description** — not as `**Odhad:** 120 min`, not in the preamble, not in the
technical plan, not in a subtask name. The estimate gets revised; a number
duplicated into prose has to be revised in two places, and the two will
disagree. Show it in the preview and in the final report (Step 10), and write it
through the estimate field — Step 9.2 for the task, `estimateMinutes` on a new
subtask in Step 9.4 — that is the whole of its life.

### Step 8.1 — Compare with the existing estimate

The task's *own* field is `task.estimateMinutes`; with subtasks Teamwork also
returns `task.accumulatedEstimatedMinutes` = own field + every subtask. When the
Step 3.1 subtasks fetch returned at least one subtask:

- compare your proposal against `accumulatedEstimatedMinutes`, not the own field;
- if the own field is `> 0` while the subtasks carry estimates, the parent is
  double-counted — queue a "zero the parent estimate" proposal (Step 9.2 with
  `0`) regardless of the divergence check below.

Without subtasks:

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

## Task record rules

> Block version `wame-task-record-v1`. Shared **verbatim** across the plugins that write a Teamwork
> task — `teamwork-task-analyze`, `teamwork-tasks-from-desk`, `teamwork-tasks-from-session` and
> `teamwork-tasks-from-dnr`. Change it in all of them or in none.

### The estimate lives in the estimate field, and nowhere else

Never write minutes or hours into a task's **title** or **description**. Not in the preamble, not
inside the technical plan, not as a footer line under it.

The reason is maintenance, not taste. An estimate gets revised — after the first hour of work, after
a clarifying answer comes back, after the task is split into subtasks. A number that also sits in
prose has to be found and changed in every copy it was written to, and the copy somebody misses is
the one the next reader believes. One field, one number, nothing to reconcile.

This binds every surface that writes a task:

- the description body, including any `**Odhad:** … min` or `**Estimate:** … min` line
- the task title, including a `(120 min)` or `· 2h` suffix
- the description column of a generated import file
- a subtask's title and description, on the same terms as the parent

Where the estimate belongs instead: the API estimate field — `estimateMinutes` on a v3 read,
`estimatedMinutes` on a v3 create, `estimated-minutes` on the classic v1 update.

Where it is still fine to show: the terminal preview, the confirmation gate, the final report, and
any companion document that is not the task itself — a Desk internal note, a Markdown plan sitting
next to an XLSX. Those are read once and thrown away. The task record is not.

### Never lose what the reporter wrote

When this skill rewrites an **existing** task description, everything already there survives
**verbatim** at the top, above the first `---`.

- **Inline images.** A screenshot pasted into a Teamwork description is ordinary Markdown:
  `![image.png](https://tw-inlineimages.s3-accelerate.amazonaws.com/…)`. It is **not** a separate
  attachment, and Teamwork shows it **nowhere else** — `GET /projects/api/v3/tasks/{id}/files.json`
  returns nothing for it. Dropping that link deletes the screenshot from the task. This is not
  hypothetical: one run stripped the image links out of 18 descriptions on the assumption that
  Teamwork rendered them separately, and destroyed 25 screenshots. They came back only because the
  original text happened to still be in a scratch file.
- **The reporter's own wording, spelling and punctuation.** Do not add diacritics, do not fix
  grammar, do not translate, do not tighten, do not re-order. A bug report is the record of what
  somebody saw and how they described it. A tidied version is no longer that record, and the tester
  cannot recognise their own report in it.
- **Links, lists, line breaks, and any HTML that is already there.** Pass the block through
  untouched.

Before writing, read the current description. After composing the new one, check that the old text
still occurs inside it character for character. When it does not, you are about to delete somebody's
work — stop and ask the user instead of writing.

Attachments and comments are separate records and this skill never touches them. If a change would
need one removed, that is a question for the user, not a step in the plan.

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

This is a **full replacement** — Teamwork keeps no copy of the previous
description, and an inline image removed here is gone from the task for good.
Before sending, compare `$NEW_DESC` against the original one last time and
**abort this write** if any inline image, link or attachment reference present in
the original is missing from `$NEW_DESC`. Record it as
`❌ refused: would drop <n> image(s)/link(s)` in the final report and continue
with the remaining proposals.

Hold the original byte-exact for that comparison, in a file —
`jq -j '.task.description' <<<"$BODY" > "/tmp/tw_analyze_desc_orig_${TASK_ID}.md"`
(`$BODY` = a fresh task GET) — never `echo "$BODY" | jq`: zsh's `echo` rewrites
every escaped newline in the JSON, and the "old text survives verbatim" check
then runs against a corrupted copy. Not `$(jq …)` either: a command
substitution drops the trailing newlines.

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

`NEW_ESTIMATE_MIN` is `0` when the task has (or just received) subtasks — the
subtasks carry the estimate and Teamwork rolls it up (Step 7 / Step 8.1).

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
      estimatedMinutes: ($est|tonumber)
    }
  }')
curl -sS -u "$AUTH" -H "Content-Type: application/json" -H "Accept: application/json" \
  -X POST \
  -d "$PAYLOAD" \
  "${BASE}/projects/api/v3/tasklists/${TASKLIST_ID}/tasks.json"
```

(`TASKLIST_ID` is the parent task's `tasklistId` from Step 3.)

`estimatedMinutes` — with the `d` — is the v3 **write** field. Until 1.3.1 this
payload sent `estimateMinutes` (the read field): HTTP 201, subtask created, and
its estimate silently `0`. Read it back (`.task.estimateMinutes` on the new id)
before reporting the split as applied, and after the last subtask check the
parent: `.task.estimateMinutes` must be `0` and `.task.accumulatedEstimatedMinutes`
the sum of the subtasks.

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
- Inline images carried over: <n>/<n> · links <n>/<n> · attachment refs <n>/<n>
  (each pair must be equal — a mismatch means the rewrite deleted something the
  reporter put into the task)
- New sections: AC (<n> items), Goal (<n> sentences), Technical plan (<n> lines)
- Cross-cutting criteria (Step 6.0): reachability <n> · security <n> ·
  performance <n> · ui_ux <n> — not applicable: <dimensions> (<one-line reason>)
- Framework (plan only): <installed versions, e.g. Laravel v12.28.1 · Vue 3.5.13>
  — <n> idiom line(s) in the technical plan · or ⚠ versions not detected (not
  run inside the project's repository — generic line written)
- Comments: <n> via v3 / v1 · or ⚠ unavailable (HTTP <code>)
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
- Start columns: <todo_stages, e.g. `Ready for Development`, `To Do`>
- <count> tasks were fetched and analysed read-only (not in a start column,
  not on the board, or not assigned to me); listed below with their current
  state.
  - <task title> (<id>) — <stage / "not on the board" / "column unresolved">
    — assigned to <user> — 📋 read-only (<reason>)
  - …

### Links
- Task: <BASE>/app/tasks/<id>
- Project: <BASE>/app/projects/<projectId>
```

`<projectId>` and `<projectName>` come from Step 3.1 (`.task.tasklist.meta.projectId`
and `.included.projects["<projectId>"].name`). Up to 1.2.0 they were read from
`.task.projectId`, which v3 does not return, so the project link came out as
`/app/projects/null`. If the id is still empty, print `Project: ⚠ unknown`
rather than a broken link.

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
- A secondary fetch that fails on every endpoint (comments, subtasks, workflow
  stages) — the analysis goes on, and the report names the gap as *"unavailable"*
  rather than presenting it as *"none"*

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
