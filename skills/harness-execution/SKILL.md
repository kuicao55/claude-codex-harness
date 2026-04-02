---
name: harness-execution
description: "Execute implementation plans using the Generator vs. Evaluator (GvE) architecture. Enforces TDD discipline, manages optional Codex review prompts, and ensures every task passes adversarial evaluation before completion."
---

# Harness Execution — GvE Architecture

Execute a plan task by task using the Generator vs. Evaluator (GvE) separation. The Generator implements; the Evaluator attacks. Only an explicit Evaluator Pass counts as task completion.

**Announce at start:** "I'm using the harness-execution skill with GvE architecture."

## The Iron Law

```
NO TASK IS COMPLETE UNTIL THE EVALUATOR GIVES AN EXPLICIT PASS.
```

Generator's self-review does not count. Codex findings do not count alone. Only the Evaluator's verdict closes a task.

## Setup

### Step 1: Load the Plan

Read the plan file. If not specified, ask: "Which plan file should I execute? (path to the `.md` file in `docs/harness/plans/`)"

Review critically — identify any questions or concerns. If the plan has critical gaps (missing code, missing test commands, contradictory steps), raise them with the user before starting.

### Step 2: Check Codex Availability

Run a silent check: are `/codex` commands available in this session?

- If `/codex` commands are available: set `codex_available = true`. You will offer Codex options at decision points.
- If not available: set `codex_available = false`. Skip all Codex prompts silently.

### Step 3: Create Task List

Create a TodoWrite entry for every task in the plan. Mark the first task as `in_progress`.

---

## GvE Flow — Per Task

Repeat this flow for each task in the plan:

### Phase G: Generator

Dispatch a Generator subagent using the template at `skills/harness-execution/generator-prompt.md`.

**Provide to Generator:**

- Full text of the current task (copy from plan — never make the Generator read the plan file itself)
- Scene-setting context: what was built in prior tasks, architectural decisions, relevant file structure
- Working directory

**Handle Generator status:**

| Status               | Action                                                                                            |
| -------------------- | ------------------------------------------------------------------------------------------------- |
| `DONE`               | Proceed to Codex Decision Point                                                                   |
| `DONE_WITH_CONCERNS` | Read concerns. If they affect correctness or scope, address before proceeding. Otherwise proceed. |
| `NEEDS_CONTEXT`      | Provide the missing information and re-dispatch Generator                                         |
| `BLOCKED`            | Go to Codex Rescue Decision Point (see below)                                                     |

### Codex Rescue Decision Point (only if Generator is BLOCKED and `codex_available`)

Ask the user:

> "Generator is blocked on Task N: **<task name>**.
>
> Reason: <what Generator reported>
>
> What would you like to do?
>
> 1. Provide more context to Generator and retry
> 2. Use `/codex:rescue` to hand this task off to Codex
> 3. Skip this task and note it as blocked (not recommended)"

- If user chooses 1: gather more context, re-dispatch Generator
- If user chooses 2: invoke `/codex:rescue` with context from the task + Generator's blocked report. When Codex completes, treat its output as the Generator result and proceed to the Codex Review Decision Point.
- If user chooses 3: log the task as blocked in activity log, mark it with a `⚠️ BLOCKED` note, continue to next task

### Codex Review Decision Point (only if `codex_available`)

After Generator reports DONE:

> "Generator completed Task N: **<task name>**.
>
> Before Evaluator review, would you like to involve Codex?
>
> 1. No — proceed with Evaluator only
> 2. Yes — `/codex:review` (read-only second opinion on architecture/patterns)
> 3. Yes — `/codex:adversarial-review` (aggressive hunt for bugs, security issues, performance problems)"

- If user chooses 1: proceed directly to Evaluator
- If user chooses 2: format request using `codex-review-prompt.md` for review mode. Collect findings. Pass findings to Evaluator as supplementary context.
- If user chooses 3: format request using `codex-review-prompt.md` for adversarial mode. Collect findings. Pass findings to Evaluator as supplementary context.

**Note:** When `codex_available` is false, skip this prompt entirely and go straight to the Evaluator.

### Phase E: Evaluator

Dispatch an Evaluator subagent using the template at `skills/harness-execution/evaluator-prompt.md`.

**Provide to Evaluator:**

- Full text of the current task requirements
- Generator's implementation report (what was built)
- Codex findings (if any) — attach as "Supplementary Codex Findings" section
- Working directory

**Evaluator verdict:**

| Verdict | Action                                                                                            |
| ------- | ------------------------------------------------------------------------------------------------- |
| `PASS`  | Task complete. Proceed to Post-Task actions.                                                      |
| `FAIL`  | Return to Generator with Evaluator's specific issues. Generator re-implements. Return to Phase G. |

There is no partial pass. A FAIL means the entire Generator phase restarts for this task.

**Re-implementation limit:** If the same task has failed Evaluator review 3 times, stop and escalate to the user:

> "Task N has failed Evaluator review 3 times. Issues: <summary>. Would you like to: (1) provide architectural guidance and retry, (2) simplify the task scope, or (3) skip this task and flag it for later?"

### Post-Task: Log and Update Progress

After Evaluator PASS:

1. **Invoke `harness:activity-logging`** — record this task completion with full context
2. **Update plan file** — mark the task checkbox: `- [ ]` → `- [x]`
3. **If large project** — check if ALL tasks in the current milestone plan are now `- [x]`:
   - If yes: prompt user: "All tasks in this milestone are complete and Evaluator-approved. Mark milestone **<title>** as passed? (yes/no)"
   - If user confirms: invoke `harness:progress-management` to set `passed: true` for this milestone
4. Announce: "Task N complete. Moving to Task N+1."
5. Mark next task as `in_progress` in TodoWrite

---

## After All Tasks Complete

1. Run the full project test suite to verify nothing regressed:

   > "Running full test suite to verify all tasks integrate correctly..."

   If tests fail: stop and debug using `harness:systematic-debugging` principles before claiming completion.

2. Announce:
   > "All N tasks complete and Evaluator-approved. Here's a summary:
   >
   > - Tasks completed: N
   > - Codex invocations: X (review: Y, adversarial: Z, rescue: W)
   > - Evaluator iterations: X total (Y re-implementations required)
   >
   > What would you like to do next?
   >
   > 1. Start the next milestone (`/harness:resume`)
   > 2. Create a PR / merge the branch
   > 3. Review the activity log (`logs/activity-<date>.jsonl`)"

---

## Red Flags — STOP

- Starting implementation on `main`/`master` without explicit user consent
- Proceeding to next task while Evaluator has open issues
- Trusting Generator's self-review instead of running Evaluator
- Letting Generator write production code before a failing test exists
- Skipping activity logging after task completion
- Marking a milestone passed without all tasks being Evaluator-approved

---

## Integration

**Skills used by this skill:**

- `harness:activity-logging` — mandatory after every task
- `harness:progress-management` — to mark milestones passed
- Subagent templates: `generator-prompt.md`, `evaluator-prompt.md`, `codex-review-prompt.md`
