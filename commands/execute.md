---
description: "Execute an existing plan using the Generator vs. Evaluator (GvE) architecture with TDD enforcement and optional Codex integration."
---

Invoke the `harness:harness-execution` skill to begin executing a plan.

Tell the user: "Starting harness execution with GvE architecture. Each task will go through Generator (implementation) → optional Codex review → Evaluator (adversarial verification). Only Evaluator-approved tasks count as complete."

Then immediately read and follow the `harness:harness-execution` skill.
