---
description: "Manually trigger session handoff. Packages current session state into a Handoff Document and triggers /clear for a fresh context. Normally called automatically at session boundaries."
---

Invoke the `harness-handoff` skill with context: this is a `/super-harness:handoff` invocation.

Tell the user: "Packaging current session state for handoff. This will create a Handoff Document and clear the session context for the next session to resume from."

Then immediately read and follow the `harness-handoff` skill, passing the context that this is a `/super-harness:handoff` invocation.
