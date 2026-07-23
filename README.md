# Web development Agent Harness

A leash for coding agents working in an existing React codebase. Four phases: a Researcher and Planner produce a `plan.md`, **you edit and approve it**, a Generator builds against it with a test-writer alongside, and a separate Evaluator grades the result against weighted criteria with hard thresholds.

The core idea is that agents grade their own work generously, so the agent building and the agent judging must be different agents holding the same criteria.

**Use:** drop `harness.md` in your repo root, point `CLAUDE.md` at it, then ask Claude to run the harness for a feature request.
