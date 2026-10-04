---
name: code-review
description: Use when reviewing a diff, spec, page object, fixture, or PR in pw_starter against this repo's coding standards. Checks the change against CODING_STANDARDS.md and flags violations.
---

# Code Review (pw_starter)

Reviews changes in this repo against `CODING_STANDARDS.md`, the source of truth for conventions here. Read that file first if it's not already in context — don't restate its prose, apply it.

## Step 0: Static checks

Before reviewing conventions, run the mechanical checks that don't require judgment:

1. `npm run typecheck` — `tsc --noEmit`. Type errors block everything downstream; report them first.
2. `npm run lint` — ESLint with `@typescript-eslint` + `eslint-plugin-playwright` (catches things like `waitForLoadState('networkidle')`, unused vars/imports).
3. `npm run format:check` — Prettier. If only formatting is off, say so and offer `npm run format` (or `lint:fix` for auto-fixable lint issues) rather than hand-editing.

Report these failures grouped by command (typecheck / lint / format), most blocking first, with `file:line` citations. Don't silently fix anything unless asked. Once these pass (or their failures are reported), move to the conventions checklist below — don't re-flag a type error or lint violation there, that's what Step 0 is for.

## What to review

- A git diff (`git diff`, `git diff main...HEAD`) when reviewing a branch or PR.
- Specific files when asked to review a spec, page object, fixture, or facade change.

## Checklist (mapped to CODING_STANDARDS.md sections)

1. **Structure** — new pages extend `BasePage`; multi-page flows added to `ShopFacade`, not duplicated in specs or one-off helpers.
2. **Naming** — test titles follow `<ID> <behavior> @tag`; page objects named `<Feature>Page`; facade methods named after the user flow.
3. **Test design** — Arrange/Act/Assert layout; no inter-test dependency; no `if/else` branching inside spec bodies.
4. **Waits** — no hard sleeps (`page.waitForTimeout`, `setTimeout`); no new `waitForLoadState('networkidle')`; no magic-number timeouts inline without justification.
5. **Locators** — `data-test` > role/accessible name > CSS > XPath; locators defined only in page objects, never inline in specs or `ShopFacade` (`page.locator(...)` / `page.getByRole(...)` in a `.spec.ts` file is a violation).
6. **Test data** — no inline literals that belong in `data/`; no hardcoded credentials; any new generated secrets/session files added to `.gitignore`.
7. **Assertions** — `expect(...)` only in specs, never in page objects/`ShopFacade`; meaningful failure messages on non-obvious assertions.
8. **Configuration** — no hardcoded URLs/timeouts that belong in `playwright.config.ts` or env vars.
9. **Reliability** — no retry logic added inside individual tests to mask flakiness.
10. **Reporting** — new regression tests tagged `@regression`; fast/critical-path tests also tagged `@smoke` where appropriate.
11. **Code quality** — no dead code or reimplementation of existing page-object/facade logic; no leftover scratch/demo tests.
12. **CI/CD** — changes to `playwright.config.ts` or scripts don't break the smoke/regression split or parallelization.

## Step 1: Parallel review subagents

Once Step 0 passes (or its failures are reported), spawn these two subagents in parallel — both `Agent` calls in a single message, `run_in_background: false`, since Step 2 needs both results before it can run. Give each the diff (or files) under review.

1. `subagent_type: coding-standards-reviewer` — checks checklist items 1, 2, 3, 5, 8, 10, 11, 12 (structure, naming, test design, locators, configuration, reporting, code quality, CI/CD).
2. `subagent_type: security-reviewer` — checks checklist item 6 (hardcoded credentials, secrets/session files missing from `.gitignore`) plus PII in test data, unsafe eval/injection, and secrets exposure in CI/workflow changes.

Each returns candidates only — nothing is posted or finalized yet.

## Step 2: Verifier pass

Spawn `subagent_type: review-verifier` as a second opinion. Give it: the diff, `CODING_STANDARDS.md`, and both candidate lists from Step 1. It independently re-checks each candidate (not just trusting the first pass), drops false positives, and tags survivors `CONFIRMED` (verified against the diff) or `PLAUSIBLE` (likely but couldn't fully verify, e.g. needs runtime context).

## How to report findings

Call `ReportFindings` with the verifier's surviving findings, most severe first (correctness/security like hardcoded secrets or leaking locators outside page objects, before style/tagging nits). If nothing survives, report an empty list and say so plainly — don't invent nitpicks.

When reviewing a PR in a context that can post GitHub PR review comments (e.g. CI), additionally post each finding as an inline comment on the exact line it applies to, not as a single summary comment — so the author sees the fix where the change is needed.

Do not fix violations unless explicitly asked; this skill is for review only. If asked to also fix, apply the minimal change needed to satisfy the specific checklist item.
