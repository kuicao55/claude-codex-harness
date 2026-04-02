---
name: activity-logging
description: "Mandatory post-task activity logging for claude-codex-harness. Prevents session memory loss by recording every completed task to a JSONL log file. Must be invoked after every task completion."
---

# Activity Logging

Record every completed task to a structured JSONL log. This prevents "memory loss" across sessions — when `/harness:resume` is invoked in a new session, the log provides precise context about what happened before.

**Announce at start:** "I'm using the activity-logging skill to record this task."

## The Iron Law

```
LOG EVERY TASK AFTER EVALUATOR PASS. NO EXCEPTIONS.
```

Skipping a log entry means a future session cannot reconstruct what happened. The activity log is the harness's long-term memory.

---

## When to Invoke This Skill

Invoke immediately after every task that receives an Evaluator PASS verdict. Also invoke for:

- Brainstorming session completion (phase: `brainstorming`)
- Plan-writing session completion (phase: `planning`)
- A task marked as BLOCKED (to record the blockage)

---

## Log File Location and Format

**File:** `logs/activity-YYYY-MM-DD.jsonl` (one file per calendar day, in the project repo)

**Format:** JSONL — one JSON object per line, newline-delimited. Append only.

**Directory:** Ensure `logs/` exists: `mkdir -p logs`

---

## Log Entry Schema

Each entry is a single JSON object on one line:

```json
{
  "timestamp": "2026-04-02T14:32:15Z",
  "session_id": "session-2026-04-02-001",
  "milestone_id": "milestone-2",
  "task_id": "task-3",
  "task_title": "Implement task assignment endpoint",
  "phase": "execution",
  "action": "Implemented POST /tasks/:id/assign with auth check and validation",
  "generator_status": "DONE",
  "evaluator_status": "PASS",
  "codex_used": false,
  "codex_mode": null,
  "files_changed": ["src/routes/tasks.py", "tests/routes/test_tasks.py"],
  "notes": "Evaluator flagged missing rate limiting — added as Minor, deferred to later milestone"
}
```

**Field definitions:**

| Field              | Type           | Description                                                                              |
| ------------------ | -------------- | ---------------------------------------------------------------------------------------- |
| `timestamp`        | ISO 8601       | When the log entry was written                                                           |
| `session_id`       | string         | `"session-YYYY-MM-DD-NNN"` — increment NNN if multiple sessions per day                  |
| `milestone_id`     | string or null | The milestone ID from `claude-progress.json`, or `null` for small projects               |
| `task_id`          | string         | `"task-N"` matching the task number in the plan                                          |
| `task_title`       | string         | Short description of the task                                                            |
| `phase`            | enum           | `"brainstorming"` \| `"planning"` \| `"execution"`                                       |
| `action`           | string         | 1-2 sentences describing what was actually done                                          |
| `generator_status` | enum           | `"DONE"` \| `"DONE_WITH_CONCERNS"` \| `"BLOCKED"` \| `"SKIPPED"`                         |
| `evaluator_status` | enum           | `"PASS"` \| `"FAIL_THEN_PASS"` \| `"SKIPPED"` \| `"BLOCKED"`                             |
| `codex_used`       | boolean        | Whether Codex was invoked for this task                                                  |
| `codex_mode`       | string or null | `"review"` \| `"adversarial-review"` \| `"rescue"` \| `null`                             |
| `files_changed`    | array          | List of file paths modified by this task                                                 |
| `notes`            | string or null | Important context for next session — Evaluator concerns, deferred items, blocking issues |

**`evaluator_status` values explained:**

- `PASS` — passed on first Evaluator review
- `FAIL_THEN_PASS` — failed at least once, then passed after Generator re-implementation
- `SKIPPED` — phase was `brainstorming` or `planning` (no Evaluator in these phases)
- `BLOCKED` — task could not be completed (use with `generator_status: "BLOCKED"`)

---

## How to Write a Log Entry

1. Gather the information from the current execution context
2. Construct the JSON object (single line, no pretty printing)
3. Append to `logs/activity-YYYY-MM-DD.jsonl`:

```bash
echo '{"timestamp":"...","session_id":"...","milestone_id":"...","task_id":"...","task_title":"...","phase":"execution","action":"...","generator_status":"DONE","evaluator_status":"PASS","codex_used":false,"codex_mode":null,"files_changed":["..."],"notes":null}' >> logs/activity-2026-04-02.jsonl
```

Or write the JSON object using a file write tool — append to the file, do not overwrite.

4. Commit: `git add logs/ && git commit -m "harness: log task-N completion"`

---

## How Resume Uses the Log

When `/harness:resume` is invoked and a plan file is found but partially executed, the activity log supplements the plan's checkboxes with richer context. The `harness-entry` skill reads the log to:

- Understand which tasks had Evaluator re-iterations (and why)
- Surface any `notes` about deferred items or concerns
- Confirm the session context before continuing

When displaying the resume summary, the entry skill may show recent log entries:

```
Recent activity (from logs/activity-2026-04-02.jsonl):
  14:32 — task-3 PASS (DONE → PASS on first review)
  14:55 — task-4 PASS (DONE → FAIL_THEN_PASS — Generator missed null check on line 42)
  15:20 — task-5 BLOCKED — Codex rescue invoked, waiting for result
```

---

## Brainstorming and Planning Log Entries

For non-execution phases, use simplified entries:

```json
{
  "timestamp": "2026-04-01T10:15:00Z",
  "session_id": "session-2026-04-01-001",
  "milestone_id": null,
  "task_id": "brainstorm-session",
  "task_title": "Project brainstorming session",
  "phase": "brainstorming",
  "action": "Completed brainstorming for task-manager project. Spec saved to docs/harness/specs/2026-04-01-task-manager-design.md",
  "generator_status": "SKIPPED",
  "evaluator_status": "SKIPPED",
  "codex_used": false,
  "codex_mode": null,
  "files_changed": ["docs/harness/specs/2026-04-01-task-manager-design.md"],
  "notes": "User prefers REST over GraphQL. Authentication deferred to milestone-1."
}
```
