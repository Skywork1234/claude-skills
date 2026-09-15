---
name: reverse-engineer
description: Systematically reverse-engineer an unfamiliar or inherited codebase — map its architecture, trace data flow, extract conventions, and flag risk areas — before making changes. Use when the user picks up a legacy project, inherits a module/repo with no docs or CLAUDE.md, or asks to understand/analyze an existing codebase before working in it.
---

## Goal

Build an accurate mental model of a codebase you didn't write, fast, without reading every file. Output is a structured understanding the user can act on — not an exhaustive file-by-file summary.

## Step 1 — Establish the terrain

Before reading application code, determine:
- **Stack**: language(s), framework, package manager (check `pubspec.yaml`, `package.json`, `go.mod`, etc.)
- **Entry points**: `main.dart`/`main.ts`/etc., routing config, app bootstrap
- **Size/shape**: `git log --oneline | wc -l`, `find` for file counts per top-level dir — decides whether to read directly or delegate to Explore/general-purpose agents for breadth
- **Existing docs**: README, CLAUDE.md, ADRs, wiki links in comments — read these first, they're cheaper than inferring

If the repo has no git history (fresh checkout/zip), say so and rely on file structure only.

## Step 2 — Map the architecture

Identify the layering/pattern actually used (not the textbook name for it):
- Folder convention: feature-first vs type-first vs layer-first (data/domain/presentation, MVC, MVVM, clean architecture variants)
- State management approach (bloc/cubit, riverpod, redux, plain setState, etc.) and where it lives
- How network/data layer is structured: client setup (dio/axios/fetch/retrofit), interceptors, base response envelope shape, error handling style (try/catch vs status-code branching)
- DI approach, if any (get_it, riverpod, manual constructor injection, none)

For a large repo, don't read every feature folder — read 2-3 representative examples of the same pattern (e.g. two datasources, two blocs) and confirm the pattern holds via grep before generalizing from one file.

## Step 3 — Trace one flow end-to-end

Pick one real user-facing feature (login, a list screen, a checkout step — whatever exists) and trace it from UI trigger → state layer → data/API layer → model → back to UI. This single trace surfaces more real conventions (naming, error propagation, null handling) than scanning many files shallowly, and gives you a template to reason about the rest of the codebase.

## Step 4 — Extract conventions worth documenting

Note anything a newcomer would get wrong by guessing:
- Naming conventions for files/classes that deviate from framework defaults
- Manual serialization vs codegen (json_serializable/freezed/none) and why
- Non-obvious dependencies between packages/modules (monorepo boundaries, shared core packages like `next_core`)
- Deliberate inconsistencies or known-broken areas (check for TODO/FIXME, disabled tests, commented-out code) — these are signal, not noise
- Anything that looks like tech debt vs. anything that looks like an intentional but unusual choice (git blame / commit messages help distinguish these)

## Step 5 — Flag risk areas and open questions

Call out explicitly:
- Areas with no tests, or tests that are skipped/mocked-out in a way that could hide breakage
- Places where the pattern is inconsistent across the codebase (two different error-handling styles, two datasource shapes) — ask the user which one to follow going forward rather than guessing
- Any security-relevant findings (hardcoded secrets/keys, disabled cert pinning, overly broad permissions) — surface these regardless of what else was asked

## Step 6 — Deliver findings

Default to a concise chat summary: stack, architecture pattern, one traced flow as a concrete example, conventions list, risk/open-questions list. Keep it scannable — headers and short bullets, not prose paragraphs.

Only write it to a file (e.g. `ARCHITECTURE.md` or similar) if the user asks for a persisted doc, or if they're clearly about to hand this analysis to someone else. Don't create the file by default — a one-off understanding pass doesn't need a permanent artifact per this user's general no-unsolicited-docs preference.

If the codebase is a Flutter project, additionally check: bloc vs cubit usage and event-based vs pure-function conventions, dio interceptor setup (auth/refresh/logging), and whether repositories wrap datasources or datasources are consumed directly — these are the areas most likely to differ project-to-project even within Flutter.

## When to stop and ask

If two seemingly-current patterns coexist with no clear "old vs new" signal (no deprecation comments, similar recency in git blame), don't silently pick one to describe as "the" convention — tell the user both exist and ask which to treat as canonical before they start building on top of your summary.
