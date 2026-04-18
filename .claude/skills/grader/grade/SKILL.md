---
name: grader/grade
description: >
  Grade one or all remaining groups for a lab. Reads MIND.md for context,
  runs tests, analyses student code, produces grading-analysis.md per group,
  and keeps the MIND.md scoring table up to date. Requires grader/init to
  have completed Phase 1 first.
---

# grader/grade

## Guiding Principles

**Never modify this skill file.** The skill is read-only during execution.
Any session-specific information — professor instructions, notes, decisions,
discoveries — belongs in `MIND.md`, which is the session's central memory.
If the professor asks to remember something, write it to MIND.md, not here.

**Workspace isolation:** never read or write outside the current working
directory.

**MIND.md is the source of truth.** Read it first on every invocation —
the grading procedure, deadline, grading template, and scoring table all come from there.

**Student code only in analyses.** grading-analysis.md is published to
students. Never reference the solution or expected implementation — only
the student's own code.

**Keep the scoring table current.** Update MIND.md after every group is
graded. Never leave the table in a state that does not reflect actual
progress.

**The Grading Procedure is the authority for every scoring decision.**
Read it in Step 0 and derive which grading actions are required. Do not
run tests, analyse code, or start the project unless the procedure requires
it. Every criterion score must trace back to a procedure subsection.

**Consistent penalties across all groups.** The Grading Procedure defines
all active penalties with short IDs (e.g. `IMPL-01`). Apply every listed
penalty that fits. When a new penalty pattern is discovered, raise it with
the professor — do not modify the procedure or Penalty Archive directly.

---

## Step 0 — Load context from MIND.md

Read `labs/<lab-slug>/MIND.md`. Extract:
- Deadline.
- Grading analysis template.
- Scoring table — identify which groups have an empty Analysis column
  (not yet graded) and which have already been graded.
- **Grading Procedure** — check the status annotation on the first line of
  the `## Grading Procedure` section:
  - `<!-- status: READY -->` → proceed normally.
  - `<!-- status: DRAFT -->` or status missing → **stop**:
    > The grading procedure has not been approved yet (status: DRAFT).
    > Please review and approve it with `grader/procedure` before grading.

  Load the full procedure. From it, derive the **grading plan** for this
  session — which steps are needed based on source types present:

  | If procedure contains… | Then… |
  |------------------------|-------|
  | Any `automated` criterion | Step 2 (run tests) is required |
  | Any `manual` criterion | Step 3 (code analysis) is required |
  | `Hidden test coverage: enabled` | Step 3b (hidden tests) is required |
  | Any `live` criterion | Step 4 (visual run) is required |

  Also note: grade formula, late penalty rule, all penalty IDs and
  deductions — these must be applied consistently to every group.

If Phase 1 is not fully checked off, stop and tell the professor:
> Init is not complete. Please finish `grader/init` before grading.

---

## Step 1 — Select group to grade

Find the first group in the scoring table whose Analysis column is empty.
Present it to the professor for confirmation:

> Next up: **<group-slug>** (N remaining). Shall I proceed, or grade a
> different group?

If the professor names a different group, use that instead. Otherwise
proceed with the suggested one.

---

## Step 2 — Run automated tests

**Skip this step if the grading plan has no `automated` criteria.**

### Check submission status

```bash
git -C labs/<lab-slug>/submissions/<group-slug> log --oneline \
  | grep -v "github-classroom\[bot\]"
```

- **Zero student commits:** mark all criteria as 0, set Comment to
  "Pas soumis — aucun commit étudiant", write a minimal
  `grading-analysis.md`, update scoring table, skip to next group.

### Run tests

Detect the test toolchain from the submission (e.g. `package.json` → `npm test`,
`Makefile` → `make test`, `pom.xml` → `mvn test`, `mix.exs` → `mix test`).
Install dependencies if needed, then run:

```bash
cd labs/<lab-slug>/submissions/<group-slug>
<install command>
<test command> 2>&1 | tee test-evidence.log
```

If Docker is required and not running:
> Docker Desktop is not running — please start it, then let me know.

Parse the output: count passing tests per named suite. Fill the automated
test columns in the scoring table immediately.

---

## Step 3 — Analyse student code

**Skip the manual analysis subsection if the grading plan has no `manual`
criteria — but always read the student's diff as context for all steps.**

### Read the student's diff

Generate the diff between the starting template and the student's submission
to identify what they changed. Limit the diff to the directories students
were expected to modify (infer from the grading procedure and solution diff
in MIND.md — typically `src/`, `test/`, or equivalent for the language):

```bash
git -C labs/<lab-slug>/submissions/<group-slug> \
  diff <template-commit>..HEAD -- <relevant dirs>
```

If the template commit is unknown, diff against the first student commit:
```bash
git -C labs/<lab-slug>/submissions/<group-slug> \
  log --oneline | grep -v "github-classroom\[bot\]" | tail -1
# use that commit hash as base
```

### Write grading-analysis.md

Create `labs/<lab-slug>/submissions/<group-slug>/grading-analysis.md`
using the Grading Analysis Template from MIND.md as the structure.

For each file section in the template:
- Extract the relevant function(s) or class(es) from the student's code.
- Prefer showing the **full function or class**. Trim only if over ~30 lines.
- Add finding lines below each snippet: `> ✅ / ⚠️ / ❌`
- If a deduction maps to something in the lab spec or course material,
  add a reference: `> 📖 See: lab-spec.md § <section>`
- **Never include solution code or expected implementations.**

Fill in the Score Decisions table, using the grading procedure as the
authority for each criterion:
- **Automated criteria:** scores from Step 2 test results.
- **Manual criteria:** scores from code analysis above.
- **Live criteria:** leave blank — filled after Step 4 professor findings.
- **Late penalty:** apply exactly the formula from the procedure preamble.
  Determine the student's last commit date with `git log` (exclude
  `github-classroom[bot]` commits). Show the calculation explicitly.
- **Final grade:** compute using the formula from the procedure preamble.

---

## Step 3b — Hidden test coverage (if enabled)

Skip this step if the `## Grading Procedure` preamble in MIND.md does not
contain `**Hidden test coverage:** enabled`.

### Hidden test convention

**Before writing any hidden test**, locate the existing test suite by
inspecting the submission: check `package.json` (test script, mocha/jest
config), `.mocharc*`, `jest.config*`, or any config file that reveals
where tests live and how they are run. Read at least one existing test
file to understand the exact import paths, fixture setup, helper usage,
and assertion style. Mirror that pattern exactly — do not invent a new
structure.

Each hidden test's `describe` or `it` label must be prefixed with
`[HIDDEN]` — this is the only hard rule. Where to place the test file
is a judgment call: use whatever location fits the project's structure
(alongside starter tests, in a subfolder, or wherever the test runner
will pick them up). Mirror the existing framework, imports, and assertion
style exactly.

```js
// existing starter test (example — actual style may differ)
describe("Game interaction", () => { ... });

// hidden test — same framework and style, [HIDDEN] prefix required
describe("[HIDDEN] Game interaction — edge cases", () => {
  it("[HIDDEN] slam on already-placed shape does nothing", () => { ... });
});
```

### 3b-check — Check for existing hidden tests

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
Re-running goes directly to 3b-3. Adding goes to 3b-1 (gap analysis)
then 3b-2 (new scenarios only). Regenerating removes all `[HIDDEN]`
blocks and runs the full flow from 3b-0.

**No matches found:** proceed to 3b-0 (full procedure).

### 3b-0 — Detect and verify tooling

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

### 3b-1 — Identify coverage gaps

Using the student's diff (from Step 3) and the existing test results
(from Step 2), identify what is implemented but not covered by the
visible test suite:
- Functions or branches that exist in the student's code but are never
  called by the starter tests.
- Edge cases mentioned in the lab spec or grading procedure that have
  no corresponding test.
- Interactions between components that the visible tests exercise in
  isolation but not together.

### 3b-2 — Propose test scenarios

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

### 3b-3 — Generate and run selected tests

For each selected scenario:
1. Generate a test in the same framework as the existing test suite
   (same file format, same assertion style). Place it wherever the test
   runner will pick it up given the project's structure — no fixed location
   is required. All labels must include the `[HIDDEN]` prefix.
2. Run it and capture output to `hidden-test-evidence.log` in the
   submission folder.
3. Record pass / fail.

### 3b-4 — Map failures to deductions

For each failing hidden test:
- Identify which existing criterion it targets.
- Propose a deduction amount (consistent with similar penalties in the
  Grading Procedure).
- Present to the professor for confirmation before applying.

Once confirmed, apply as a deduction on the relevant criterion in the
Score Decisions table of `grading-analysis.md`. Add a finding line:
`> ❌ Hidden test failed: <scenario name> — <what went wrong>`

If the failure reveals a recurring pattern not yet in the Grading Procedure,
follow the Step 4b flow to register it and apply it retroactively.

---

## Step 4 — Visual run

**Skip this step entirely if the grading plan has no `live` criteria.**

### Read the run plan

Read `## Run Plan` in MIND.md. Check `**AI-run:**`:

**`approved`** — adapt the start command to the current group (substitute
`<group-slug>` in any paths), then start the project. Always bind to host
port 3333 to avoid conflicts. Save console output to `project-evidence.log`
in the submission folder.

If Docker is required but not running:
> Docker Desktop is not running — please start it, then let me know.

**`opted-out`** — do not start the project. The professor runs it independently.

### Collect findings (both paths)

Before prompting the professor, summarise what to expect based on the
code analysis:
- Which interactions should work correctly.
- Which have suspected issues and why.
- Any edge cases worth testing.

Then prompt:
> **<group-slug>** is ready. Please test the live criteria listed in the
> procedure and report your findings for each interaction.
> (`approved`: project is running at <url> / `opted-out`: please start it yourself)

Wait for the professor's findings. Record each result as a finding line
in grading-analysis.md (`> ✅ / ⚠️ / ❌`).

### Adjust scores

Update the grading-analysis.md Score Decisions table if the visual run
reveals issues not caught by code analysis alone, or confirms that
preliminary penalties were too harsh.

Recompute the final grade.

---

## Step 4b — Check for new penalty patterns

After completing the code analysis and visual run, review the findings:

1. **Identify any deduction applied that has no matching penalty ID in the
   Grading Procedure.** A pattern is "new" if it describes a mistake that
   could recur in other submissions (e.g. using the wrong API, skipping a
   required step, misunderstanding a spec requirement).

2. **For each new pattern found:**

   **If the grader identified it independently:** raise it with the professor
   immediately — do not register it unilaterally.

   **If the professor proposed the penalty:** verify it in the student's code
   before accepting:
   - Confirm the issue is present with a code snippet or finding line.
   - If evidence is ambiguous, ask for clarification before applying.
   - If the issue is not found, report back with evidence and ask whether to
     apply it anyway.
   - Do not update any analysis until the penalty is confirmed.

   Once the penalty is confirmed (by observation or by professor), surface
   it to the professor and invoke `grader/procedure` to handle the update:
   > ⚠️ New penalty pattern found: `<description>` (proposed: −N pts)
   > Invoking `grader/procedure` to add it to the procedure.

   After `grader/procedure` completes and the new penalty has an assigned ID:
   - Immediately check all previously graded groups for the same issue
     (read their grading-analysis.md and, if needed, their code diff).
   - For each previously graded group where the same issue is present:
     - Update their `grading-analysis.md` — add a finding line and adjust
       the Score Decisions table.
     - Update their row in the MIND.md scoring table.
   - Report to the professor:

   ```
   ⚠️  New penalty applied: <ID> — <description> (−N pts)
       Applied to current group: <group-slug>
       Retroactive check: <list affected previously-graded groups, or "none">
   ```

   Wait for the professor to confirm or adjust before proceeding.

   **If the professor wants to retire or replace an existing penalty:**
   invoke `grader/procedure` — it owns procedure and archive management.

---

## Step 4c — First-grading confirmation gate

**Only applies when this is the first group being graded** (all other rows
in the Scoring Table have an empty Analysis column).

Before writing anything to the MIND.md scoring table, present the full
grading-analysis.md to the professor:

> This is the **first grading** for this lab. Please review the full
> analysis below before I finalise the scores. Confirm when ready, or
> tell me what to adjust.

[show full grading-analysis.md content]

Wait for explicit professor confirmation ("ok", "go ahead", or specific
corrections).

### Handling professor-proposed penalties

If the professor proposes a penalty that was not in the grading analysis:

1. **Verify first — do not accept blindly.** Go back to the student's code
   and check whether the issue the professor described is actually present.
2. If the issue is clearly confirmed in the code: accept the penalty, update
   the analysis, and register it in the Penalties table (Step 4b).
3. If the evidence is ambiguous or the penalty description is unclear: ask
   for clarification before applying anything:
   > I checked `<file>:<lines>` and I'm not certain this applies because
   > `<what you observed>`. Could you clarify what specifically to look for?
4. If the issue is **not present** in the student's code: say so, with the
   relevant snippet as evidence, and ask the professor to confirm:
   > I looked at `<file>:<lines>` and did not find this issue — here is what
   > I see: `<snippet>`. Should I still apply the penalty?

Never silently skip a professor-proposed penalty, and never apply one without
verifying it exists in the student's code.

For subsequent groups this gate is skipped automatically.

---

## Step 5 — Finalise and update MIND.md

Once grading-analysis.md is complete and scores are final:

1. Update the group's row in the MIND.md scoring table:
   - Fill all criterion columns with their scores.
   - Fill Total and Grade.
   - Set the Analysis column to the relative path:
     `submissions/<group-slug>/grading-analysis.md`

2. Tick the Phase 2 counter: `All groups graded (N / Total)`.
   When all groups are done, fully tick `- [x] All groups graded`.

3. Report a brief summary to the professor:

```
✅ <group-slug> — graded

  Tests:     N / Max (automated)
  Manual:    N / Max
  Live:      N / Max
  Penalty:   −X.X (N days late) / 0
  Total:     N / Max
  Grade:     X.X

  Notable: <one-line summary of key findings>
```

---

## Step 6 — Done

Tell the professor the result and what comes next:

> ✅ <group-slug> graded. N groups remaining.
> Run `grader/grade` again for the next group, or `grader/feedback` to
> publish feedback once all groups are done.
