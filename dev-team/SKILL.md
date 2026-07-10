---
name: dev-team
description: Run a request through a PM -> dev -> tester pipeline. You (the main assistant) act as PM using plan mode to design and get approval, break the approved plan into a shared TaskList checklist, dispatch the dev subagent to implement it while updating the checklist, then dispatch the tester subagent to verify it before reporting back. Use when the user asks for this workflow explicitly, or asks for a nontrivial feature/fix and wants it planned, built, and tested with visible checklist progress.
---

## Roles

There is no separate "PM" agent — plan mode IS the PM role, run by you (the main assistant), since it already does exactly this: take the request, produce a plan, get the user's explicit approval before anything is built. Do not spawn a "pm" subagent.

`dev` and `tester` are real subagents (`~/.claude/agents/dev.md`, `~/.claude/agents/tester.md`), invoked via the Agent tool. Subagents cannot message each other directly — you are always the relay.

**Important, confirmed by a live test run:** `dev` and `tester` do NOT have working access to `TaskList`/`TaskGet`/`TaskUpdate`/`TaskCreate` in their own sessions, even when those tools are listed in their agent frontmatter — those tools appear to be scoped to your (the orchestrator's) session only. Don't brief them to "update the task list yourselves." Instead: you own the TaskList completely. Pass the relevant task subjects/descriptions/IDs directly in the dispatch prompt, and treat the subagent's final return message as a prose report that you translate into `TaskUpdate`/`TaskCreate` calls yourself afterward.

## Workflow

1. **Plan (PM).** Call `EnterPlanMode` before writing or touching the plan file — don't draft plan content to a file of your own choosing first and only formally enter plan mode after. Clarify requirements as needed, and produce a concrete implementation plan. Present it via ExitPlanMode and wait for explicit user approval. Do not proceed past this step without approval — no checklist, no dev dispatch.
2. **Checklist.** Once approved, convert the plan's steps into `TaskCreate` items — one task per concrete, independently-checkable unit of work. Keep subjects in imperative form and descriptions specific enough that dev doesn't have to re-derive scope.
3. **Dispatch dev.** Call `Agent` with `subagent_type: dev`. Since dev can't read the TaskList itself, copy the full subject + description of every relevant task into the prompt (with their IDs), plus any constraints not obvious from the tasks themselves. Ask for a final report keyed by task ID: done / blocked-and-why. One dispatch can cover the whole remaining checklist.
4. **Reconcile and verify, don't assume.** After dev returns, don't just trust its prose report — independently check the actual deliverable yourself (read the file, run the command) for at least the tasks it claims are done. Then call `TaskUpdate` yourself for each task based on what you verified:
   - Confirmed done → `TaskUpdate` to `completed`.
   - Dev reported blocked, or your own check disagrees with dev's claim → leave it `pending`/`in_progress`, surface the blocker to the user, and ask how to proceed. Do not dispatch tester while any relevant task is still open.
5. **Dispatch tester.** Call `Agent` with `subagent_type: tester`. Same constraint as dev — tester can't read the TaskList either, so paste in the task subjects/descriptions/IDs it needs to verify, plus the spec/acceptance criteria. Ask for a final report keyed by task ID: pass/fail, with defect details for any failures.
6. **Reconcile again.** Based on tester's report: tasks that passed stay `completed` (optionally add `metadata` like `{"tested": "pass"}` via `TaskUpdate`); for any real defect tester found, `TaskCreate` a new bug task describing it and set the original task back to `in_progress` via `TaskUpdate`. If new bug tasks exist, loop back to step 3 for just those tasks — don't dispatch tester again until they're back to `completed`.
7. **Report to user.** Summarize: what was built, what tester verified and how, current state of every task, and anything still open. This is a normal turn-ending summary to the user, not a subagent report.

## Mid-pipeline scope changes

If the user sends new requirements while dev or tester is mid-dispatch (or between steps), do not silently fold them into the current checklist or the current dev/tester dispatch. Treat it as an addendum:

1. Let the current dispatch finish and reconcile it normally first, unless the user is explicitly redirecting away from the in-flight work.
2. Use `AskUserQuestion` to pin down any ambiguous part of the new ask (architecture/library choices, how it should integrate with what's already built) — don't guess on anything that's genuinely a judgment call.
3. Call `EnterPlanMode` again for the addendum. Reference what's already done (mark it clearly as "already done, not part of this checklist" in the plan) so the plan file stays an accurate, current picture rather than a duplicate of the original.
4. Once approved, `TaskCreate` new items for just the addendum and continue the normal workflow from step 3. Don't re-litigate or re-verify already-completed tasks unless the new scope actually touches them.

## Rules

- Never skip the plan-mode approval step, even for requests that feel small — it's what makes step 2 onward legitimate.
- Never let dev proceed straight to tester without you independently checking the work and updating `TaskList` yourself in between — agents can be wrong or overly optimistic about their own completion state, and they can't touch the checklist even if they wanted to.
- Keep dev and tester roles separate: dev implements, tester verifies; don't let tester patch code or dev skip verification because "it looked right."
- If the user's request is genuinely small (one obvious change, no ambiguity), say so and suggest skipping the full pipeline rather than running plan mode + two subagent dispatches for a one-line fix.
