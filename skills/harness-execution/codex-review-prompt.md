# Codex Review Request Template

Use this template to format a request for Codex review after a Generator completes a task.

Covers both `/codex:review` (standard) and `/codex:adversarial-review` (aggressive) modes.

---

## For `/codex:review` (Standard Read-Only Review)

Use when the user wants a second perspective on architecture, naming, and patterns.

**Context to provide to Codex:**

```
Task just completed: Task N — <task name>

What was built:
<Generator's implementation report summary>

Files changed:
<list of files>

Please review this implementation for:
- Architecture quality: Does it follow clean separation of concerns?
- Naming clarity: Are names accurate and descriptive?
- Pattern consistency: Does it match the existing codebase style?
- Missing abstractions: Is anything obviously over-complicated or under-abstracted?
- Any obvious issues you notice

This is a read-only review — do not modify any files.
Return a structured list of observations.
```

---

## For `/codex:adversarial-review` (Aggressive Adversarial Audit)

Use when the user wants Codex to actively hunt for vulnerabilities, edge case failures, and performance issues.

**Note:** This mode is token-intensive. The user has already confirmed they want this level of scrutiny.

**Context to provide to Codex:**

```
Task just completed: Task N — <task name>

What was built:
<Generator's implementation report summary>

Files changed:
<list of files>

Task requirements:
<full task requirements text>

Please perform an adversarial review focused on:

1. ATTACK VECTORS — actively try to find ways to break this code:
   - Injection vulnerabilities (SQL, command, template)
   - Authentication/authorization bypass
   - Sensitive data exposure
   - Input validation failures

2. EDGE CASES — what inputs would cause failures?
   - Null/empty/undefined inputs
   - Boundary values (0, -1, MAX_INT)
   - Concurrent access
   - Malformed data

3. SECURITY COMPLIANCE:
   - Any security best practices violated?
   - Any OWASP Top 10 concerns?

4. PERFORMANCE RISKS:
   - O(n²) or worse complexity?
   - Database/network calls in loops?
   - Memory leaks or unbounded growth?

5. SPEC VIOLATIONS:
   - Does the implementation match ALL requirements?
   - Anything built that wasn't requested?

Return findings categorized by severity: Critical | Important | Minor.
Be specific with file:line references.
```

---

## Collecting Codex Findings

After Codex responds, extract the findings and format them as:

```
--- Codex Review Findings (mode: review | adversarial-review) ---
[Paste Codex's response here]
-----------------------------------------------------------------
```

These findings are then passed to the Evaluator subagent as supplementary context (see `evaluator-prompt.md`).

The Evaluator incorporates Codex findings into its own independent review — Codex findings alone do not determine PASS/FAIL. The Evaluator's verdict is final.
