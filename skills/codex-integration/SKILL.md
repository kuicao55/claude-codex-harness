---
name: codex-integration
description: "Setup guide and integration reference for the official codex-plugin-cc in claude-codex-harness. Covers installation, available commands, detection, and the interactive user-prompt model."
---

# Codex Integration

Reference guide for integrating the official `@openai/codex-plugin-cc` plugin with claude-codex-harness.

**This skill is a reference, not a workflow.** It is read by other skills (primarily `harness-execution`) to understand how Codex commands work and when to invoke them.

---

## What Is codex-plugin-cc?

The official OpenAI plugin for Claude Code. Published by OpenAI engineers under `openai/codex-plugin-cc`. Adds `/codex` slash commands directly to Claude Code, allowing Claude to call Codex as a collaborating agent.

---

## Prerequisites

Before Codex integration can work:

1. **Codex CLI installed:**

   ```bash
   npm install -g @openai/codex
   ```

2. **ChatGPT subscription active** — Codex commands consume ChatGPT API tokens

3. **Plugin installed:**

   ```bash
   npm install -g @openai/codex-plugin-cc
   ```

4. **Restart Claude Code** — after installation, Claude Code auto-discovers the plugin and `/codex` commands become available in the command palette

---

## Available Commands

### `/codex:review`

**Mode:** Read-only code review

**What it does:** Codex examines the specified code without modifying it. Provides a second perspective from a different model on architecture quality, naming, patterns, and potential issues.

**Best for:**

- Getting a second opinion after Generator completes
- Checking that code follows patterns Codex has seen broadly across codebases
- Architecture and naming review when you want a different eye

**Token cost:** Moderate

**In harness GvE flow:** Optional, invoked at Codex Review Decision Point (Decision Point 1) when user selects option 2.

---

### `/codex:adversarial-review`

**Mode:** Aggressive adversarial audit

**What it does:** Codex actively hunts for security vulnerabilities, boundary case failures, performance regressions, and compliance gaps. Treats the code as potentially broken and tries to prove it.

**Best for:**

- Security-sensitive code (auth, payments, data access)
- Code that handles external input
- Before finalizing a milestone
- When the Evaluator found issues and you want an independent adversarial check

**Token cost:** High — do not run for every task; use judiciously

**In harness GvE flow:** Optional, invoked at Codex Review Decision Point (Decision Point 1) when user selects option 3.

---

### `/codex:rescue`

**Mode:** Background task execution

**What it does:** Hands a stuck task to Codex for background execution. Codex attempts to implement what Claude's Generator could not complete.

**Best for:**

- Generator is BLOCKED and cannot make progress
- Claude's reasoning quality has degraded (late session)
- The task requires knowledge where Codex may have an edge

**Token cost:** Varies by task size

**In harness GvE flow:** Optional, invoked at Codex Rescue Decision Point (Decision Point 2) when Generator reports BLOCKED.

---

## Detection Logic

At the start of `harness-execution`, check if Codex is available:

```
codex_available = are /codex commands available in this session?
```

- If `true`: present Codex prompts at decision points
- If `false`: skip all Codex prompts silently — no error, no mention to user

This ensures the harness degrades gracefully when Codex is not installed.

---

## Integration Model: Interactive, Not Automatic

**The harness never auto-invokes Codex.** The user is always asked first.

**Why:**

- Codex commands consume ChatGPT subscription tokens (separate from Claude's tokens)
- `/codex:adversarial-review` is particularly token-intensive
- User preferences and budget should drive Codex usage

**Two decision points in the GvE execution flow:**

### Decision Point 1 — After Generator completes

Shown only when `codex_available = true`:

```
Generator completed Task N: <task name>.

Before Evaluator review, would you like to involve Codex?
1. No — proceed with Evaluator only
2. Yes — /codex:review (read-only second opinion on architecture/patterns)
3. Yes — /codex:adversarial-review (aggressive hunt for bugs, security issues, performance problems)
```

### Decision Point 2 — When Generator is BLOCKED

Shown only when `codex_available = true` and Generator reports BLOCKED:

```
Generator is blocked on Task N: <task name>.
Reason: <what Generator reported>

What would you like to do?
1. Provide more context to Generator and retry
2. Use /codex:rescue to hand this task off to Codex
3. Skip this task and note it as blocked (not recommended)
```

---

## Formatting Codex Requests

When the user chooses a Codex option, use the templates in `skills/harness-execution/codex-review-prompt.md` to structure the request. The template provides the correct context format for each mode.

After Codex responds, its findings are:

1. Formatted as a "Codex Findings" block
2. Passed to the Evaluator subagent as supplementary context
3. The Evaluator independently decides whether the findings affect its PASS/FAIL verdict

**Codex findings alone do not determine task completion.** The Evaluator's verdict is always final.

---

## Token Awareness

When presenting Codex options to users, briefly note costs where relevant:

- `/codex:review`: "standard review — moderate token cost"
- `/codex:adversarial-review`: "⚠️ aggressive review — higher token cost; recommended for security-sensitive code"
- `/codex:rescue`: "background execution — token cost varies by task size"

This helps users make informed decisions about when to use each mode.
