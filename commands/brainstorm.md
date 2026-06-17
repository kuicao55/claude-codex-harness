---
description: "Start a new feature or project with structured brainstorming. Explores intent, requirements, and design before any implementation."
---

Show version banner first. The version is already in your session context (injected by the harness SessionStart hook as "harness is active (vX.Y.Z ...)"). Extract the version from there and display:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
super-harness v<VERSION>

欢迎使用 super-harness v<VERSION>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Then ask for the exact feature/problem:

> "这次要 brainstorm 的具体功能或问题是什么？请尽量描述目标、当前现象和期望结果。"

After the user provides context, invoke the `harness-entry` skill with context: this is a `/super-harness:brainstorm` invocation and include the user's feature description.

Tell the user: "Starting harness brainstorming session. I'll help you explore your idea, refine requirements, and produce a design spec before any code is written."

Then read and follow the `harness-entry` skill with that context.
