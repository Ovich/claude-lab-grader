---
name: grader/grade/opt-ins/hidden-tests
description: >
  Generate, run, and map hidden edge-case tests for one student group.
  Invoked by grader/grade when hidden test coverage is enabled in the
  Grading Procedure. Requires the student's diff and visible test results
  to already be available in the current grading session.
---

# grader/grade/opt-ins/hidden-tests

## Guiding Principles

**Never modify this skill file.** All output goes to
`grading-analysis.md` and `hidden-test-evidence.log` in the group's
submission folder.

**Mirror the existing test suite exactly.** Discover the framework,
import paths, fixture setup, and assertion style from the submission
before writing a single test. Do not invent a new structure.

**All hidden test labels must include `[HIDDEN]`.** This is the only
hard rule on naming. Placement is a judgment call — put the file
wherever the test runner will pick it up.

**Never apply a deduction without professor confirmation.** Propose
failing hidden tests and their deduction amounts; wait for explicit
approval before updating `grading-analysis.md`.

---

## Context expected from grader/grade

When invoked, the following context is already available in the session:

- `<lab-slug>` and `<group-slug>`
- The student's diff (from Step 3 of grader/grade)
- Visible test results (from Step 2 of grader/grade)
- The Grading Procedure loaded from MIND.md

---

## Hidden test convention

**Before writing any hidden test**, locate the existing test suite by
inspecting the submission: check `package.json` (test script, mocha/jest
config), `.mocharc*`, `jest.config*`, or any config file that reveals
where tests live and how they are run. Read at least one existing test
file to understand the exact import paths, fixture setup, helper usage,
and assertion style. Mirror that pattern exactly.

Each hidden test's `describe` or `it` label must be prefixed with
`[HIDDEN]`. Where to place the test file is a judgment call: use whatever
location fits the project's structure (alongside starter tests, in a
subfolder, or wherever the test runner will pick them up).

```js
// existing starter test (example — actual style may differ)
describe("Game interaction", () => { ... });

// hidden test — same framework and style, [HIDDEN] prefix required
describe("[HIDDEN] Game interaction — edge cases", () => {
  it("[HIDDEN] slam on already-placed shape does nothing", () => { ... });
});
```

---

## Step check — Check for existing hidden tests

Before anything else, locate the test directory from the submission config,
then scan it for any test labelled `[HIDDEN]`:

```bash
grep -rl "\[HIDDEN\]" labs/<lab-slug>/submissions/<group-slug>/ 2>/dev/null
```

**Matches found:** present a summary to the professor:

```
Hidden tests already exist for <group-slug>:
  test/game.test.js          [HIDDEN] Game interaction — edge cases (✅ passed / ❌ failed)
  test/inputListener.test.js [HIDDEN] mousedown vs click (❌ failed)
  ...

What would you like to do?
  1. Re-run existing hidden tests (no changes)
  2. Add new hidden scenarios
  3. Remove or replace a specific hidden test
  4. Regenerate all hidden tests from scratch
```

Handle the professor's choice and skip to the relevant sub-step.
Re-running goes directly to Step 3. Adding goes to Step 1 (gap analysis)
then Step 2 (new scenarios only). Regenerating removes all `[HIDDEN]`
blocks and runs the full flow from Step 0.

**No matches found:** proceed to Step 0 (full procedure).

---

## Step 0 — Detect and verify tooling

Inspect the submission to determine the test framework in use:

| Signal | Framework | Run command |
|--------|-----------|-------------|
| `package.json` with `jest`, `mocha`, `vitest` | Node.js | `npm test` / `npx <runner>` |
| `Makefile` with test target | Make | `make test` |
| `pom.xml` | Maven/JUnit | `mvn test` |
| `requirements.txt` / `pytest.ini` | pytest | `pytest` |
| other | ask professor | — |

For each required tool, check availability:
```bash
<tool> --version 2>/dev/null && echo "ok" || echo "missing"
```

For any missing tool, propose installation before proceeding:
> To run hidden tests for **<group-slug>** I need `<tool>`.
> Install with: `<install command>`
> Or run: `! <install command>` directly in this terminal.

Wait for confirmation that all required tools are available before
generating or running any tests. If the professor declines installation,
skip hidden tests for this group and note it in grading-analysis.md:
`> ⏭️ Hidden tests skipped — <tool> not available`

---

## Step 1 — Identify coverage gaps

Using the student's diff and the existing test results, identify what is
implemented but not covered by the visible test suite:
- Functions or branches that exist in the student's code but are never
  called by the starter tests.
- Edge cases mentioned in the lab spec or grading procedure that have
  no corresponding test.
- Interactions between components that the visible tests exercise in
  isolation but not together.

---

## Step 2 — Propose test scenarios

Present a numbered list of candidate scenarios to the professor. For each:

```
[1] <Scenario name>
    Criterion: <which existing criterion this validates>
    What it tests: <one sentence — the behaviour or edge case>
    Why it matters: <what student code might get wrong here>

[2] ...
```

Ask the professor:
> Which of these would you like me to generate and run?
> (Answer with numbers, "all", or "none")

---

## Step 3 — Generate and run selected tests

For each selected scenario:
1. Generate a test in the same framework as the existing test suite
   (same file format, same assertion style). Place it wherever the test
   runner will pick it up given the project's structure. All labels must
   include the `[HIDDEN]` prefix.
2. Run it and capture output to `hidden-test-evidence.log` in the
   submission folder.
3. Record pass / fail.

---

## Step 4 — Map failures to deductions

For each failing hidden test:
- Identify which existing criterion it targets.
- Propose a deduction amount (consistent with similar penalties in the
  Grading Procedure).
- Present to the professor for confirmation before applying.

Once confirmed, apply as a deduction on the relevant criterion in the
Score Decisions table of `grading-analysis.md`. Add a finding line:
`> ❌ Hidden test failed: <scenario name> — <what went wrong>`

If the failure reveals a recurring pattern not yet in the Grading Procedure,
follow the Step 4b flow in grader/grade to register it and apply it
retroactively.
