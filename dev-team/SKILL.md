---
name: dev-team
description: Run a request through a PM -> dev -> tester pipeline. You (the main assistant) act as PM using plan mode to design and get approval, break the approved plan into a shared TaskList checklist, dispatch the dev subagent to implement it while updating the checklist, then dispatch the tester subagent to verify it before reporting back. Use when the user asks for this workflow explicitly, or asks for a nontrivial feature/fix and wants it planned, built, and tested with visible checklist progress.
---

## Roles

There is no separate "PM" agent — plan mode IS the PM role, run by you (the main assistant), since it already does exactly this: take the request, produce a plan, get the user's explicit approval before anything is built. Do not spawn a "pm" subagent.

`dev` and `tester` are real subagents (`~/.claude/agents/dev.md`, `~/.claude/agents/tester.md`), invoked via the Agent tool. Subagents cannot message each other directly — you are always the relay. The shared TaskList is the single source of truth both you and the subagents read/write; that's how dev "reports back to the checklist" without talking to a PM agent directly.

## Workflow

1. **Plan (PM).** Enter plan mode, clarify requirements as needed, and produce a concrete implementation plan. Present it via ExitPlanMode and wait for explicit user approval. Do not proceed past this step without approval — no checklist, no dev dispatch.
2. **Checklist.** Once approved, convert the plan's steps into `TaskCreate` items — one task per concrete, independently-checkable unit of work. Keep subjects in imperative form and descriptions specific enough that dev doesn't have to re-derive scope.
3. **Dispatch dev.** Call `Agent` with `subagent_type: dev`. Brief it with: what the overall feature is, any constraints not obvious from the tasks themselves, and confirmation that it should call `TaskList`/`TaskUpdate` itself rather than reporting progress back to you in prose. One dispatch can cover the whole remaining checklist — dev loops through its own tasks; you don't need to spawn it once per item.
4. **Check status, don't assume.** After dev returns, call `TaskList` yourself. Do not trust the agent's prose summary alone — verify against actual task statuses:
   - All relevant tasks `completed` → go to step 5.
   - Some tasks still `pending`/`in_progress` with a blocker noted → surface the blocker to the user and ask how to proceed (more detail, descope, or you attempt it directly). Do not dispatch tester on an incomplete checklist.
5. **Dispatch tester.** Call `Agent` with `subagent_type: tester`. It will verify each completed task end-to-end and either leave it `completed` or reopen it with a new bug task for dev.
6. **Check status again.** Call `TaskList`. If tester filed new bug tasks, loop back to step 3 (dispatch dev again for just those tasks) — don't dispatch tester a second time until those are back to `completed`.
7. **Report to user.** Summarize: what was built, what tester verified and how, current state of every task, and anything still open. This is a normal turn-ending summary to the user, not a subagent report.

## Rules

- Never skip the plan-mode approval step, even for requests that feel small — it's what makes step 2 onward legitimate.
- Never let dev proceed straight to tester without you checking `TaskList` yourself in between — agents can be wrong about their own completion state.
- Keep dev and tester roles separate: dev implements, tester verifies; don't let tester patch code or dev skip verification because "it looked right."
- If the user's request is genuinely small (one obvious change, no ambiguity), say so and suggest skipping the full pipeline rather than running plan mode + two subagent dispatches for a one-line fix.
