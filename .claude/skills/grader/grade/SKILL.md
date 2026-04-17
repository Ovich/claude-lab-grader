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
criteria, deadline, grading template, and scoring table all come from there.

**Student code only in analyses.** grading-analysis.md is published to
students. Never reference the solution or expected implementation — only
the student's own code.

**Keep the scoring table current.** Update MIND.md after every group is
graded. Never leave the table in a state that does not reflect actual
progress.

**Consistent penalties across all groups.** Read the Penalties table in
MIND.md before analysing each group. Apply every listed penalty that fits.
When a new penalty type is discovered, add it to the table immediately and
retroactively check all already-graded groups — update their analyses and
scores if the same issue is present.

---

## Step 0 — Load context from MIND.md

Read `labs/<lab-slug>/MIND.md`. Extract:
- Grading criteria (criteria names, max points, automated vs manual).
- Grade formula and late penalty rule.
- Deadline.
- Grading analysis template.
- **Penalties table** — the running catalogue of known deductions. Note each
  entry; they must be applied to every group where the condition is met.
  If the section is absent (MIND.md predates this feature), add it now:
  ```markdown
  ## Penalties
  | Penalty | Description | Deduction | First seen in |
  |---------|-------------|-----------|---------------|
  ```
- Scoring table — identify which groups have an empty Analysis column
  (not yet graded) and which have already been graded.

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

## Step 2 — For each group: run automated tests

For each group to grade:

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

### Read the student's diff

Generate the diff between the starting template and the student's submission
to identify what they changed. Limit the diff to the directories students
were expected to modify (infer from the grading criteria and solution diff
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

Fill in the Score Decisions table:
- Automated test scores: from Step 2 results.
- Manual scores: based on code analysis so far (visual run in Step 4 may
  adjust these).
- Apply the late penalty using **exactly the formula in MIND.md** (confirmed
  by professor during init). Determine the student's last student commit date
  with `git log` (exclude `github-classroom[bot]` commits). Count late days
  according to the method in MIND.md (e.g. started late day = any fraction
  of a 24 h period past the deadline counts as one full day). Show your
  calculation explicitly in the Score Decisions table comment field.
- Compute the final grade using the formula from MIND.md.

---

## Step 3b — Hidden test coverage (if enabled)

Skip this step if the `## Grading Criteria` section in MIND.md does not
contain `Hidden test coverage: enabled`.

### Hidden test file convention

Hidden tests are stored in `labs/<lab-slug>/submissions/<group-slug>/hidden-tests/`
and named using the pattern:

```
hidden-<criterion-slug>-<scenario-slug>.<ext>
```

Examples: `hidden-input-listener-mousedown-edge.test.js`, `hidden-game-slam-double.test.js`

This naming makes them unambiguous and prevents confusion with starter tests.

### 3b-check — Check for existing hidden tests

Before anything else, check whether `hidden-tests/` already exists and
contains files matching the `hidden-*` convention:

```bash
ls labs/<lab-slug>/submissions/<group-slug>/hidden-tests/hidden-* 2>/dev/null
```

**Files found:** present a summary to the professor:

```
Hidden tests already exist for <group-slug>:
  hidden-input-listener-mousedown-edge.test.js   ✅ passed / ❌ failed
  hidden-game-slam-double.test.js                ✅ passed / ❌ failed
  ...

What would you like to do?
  1. Re-run existing tests (no changes)
  2. Add new scenarios
  3. Remove or replace a specific test
  4. Regenerate all from scratch
```

Handle the professor's choice and skip to the relevant sub-step.
Re-running goes directly to 3b-3. Adding goes to 3b-1 (gap analysis)
then 3b-2 (new scenarios only). Regenerating clears the folder and
runs the full flow from 3b-0.

**No files found:** proceed to 3b-0 (full procedure).

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
   (same file format, same assertion style).
2. Save the generated test file to
   `labs/<lab-slug>/submissions/<group-slug>/hidden-tests/`.
3. Run it:
   ```bash
   cd labs/<lab-slug>/submissions/<group-slug>
   <test command> hidden-tests/<test-file> 2>&1 | tee hidden-test-evidence.log
   ```
4. Record pass / fail.

### 3b-4 — Map failures to deductions

For each failing hidden test:
- Identify which existing criterion it targets.
- Propose a deduction amount (consistent with similar deductions in the
  Penalties table).
- Present to the professor for confirmation before applying.

Once confirmed, apply as a deduction on the relevant criterion in the
Score Decisions table of `grading-analysis.md`. Add a finding line:
`> ❌ Hidden test failed: <scenario name> — <what went wrong>`

If the failure reveals a recurring pattern not yet in the Penalties table,
follow the Step 4b flow to register it and apply it retroactively.

---

## Step 4 — Visual run

### Pre-run summary

Before running, summarise to the professor what to expect based on the
code analysis:
- Which interactions should work correctly.
- Which have known issues (wrong event type, wrong scope, etc.).
- Any edge cases to watch for.

This sets up a focused test session and helps confirm or challenge the
preliminary scores.

### Instrument the submission

Add an identification log line at startup: `[<group-slug>] started`.
This confirms which project is running when multiple submissions are tested
in sequence. Add it at the earliest point in the entry file (server-side
and/or client-side as appropriate for the project type).

### Detect how to run

Inspect the submission to determine the appropriate run method. Look for:
- A `Dockerfile` or `docker-compose.yml` → use Docker (preferred for
  isolation and reproducibility).
- A language-specific entry point (`package.json`, `Makefile`, `pom.xml`,
  `requirements.txt`, `mix.exs`, etc.) → use the appropriate toolchain
  directly.
- A run script provided in the repo.

If the run method is ambiguous, ask the professor before proceeding.

**Always bind to host port 3333** to avoid conflicts with services
commonly running on default ports.

> **Example — Node.js project with Docker:**
> ```bash
> docker run --rm \
>   -v "$(pwd)/labs/<lab-slug>/submissions/<group-slug>:/app" \
>   -w /app -p 3333:3000 \
>   node:lts-alpine sh -c "npm install && node server.js"
> # open http://localhost:3333
> ```
> *(This is an example. Adapt the image, install command, and start
> command to match the actual project.)*

Save console output to `project-evidence.log` in the submission folder.

### Test the required interactions

The interactions to test are defined by the grading criteria in MIND.md.
Test each one and note findings. At minimum, verify:
- The project starts without errors.
- Each required user interaction produces the expected result.
- The browser/client console has no unexpected errors.
- Edge/failure cases behave correctly (invalid actions are ignored).

### Adjust scores

Update the grading-analysis.md Score Decisions table if the visual run
reveals issues not caught by code analysis alone, or confirms that
preliminary penalties were too harsh.

Recompute the final grade.

---

## Step 4b — Check for new penalty patterns

After completing the code analysis and visual run, review the findings:

1. **Identify any penalty applied that is not yet in the Penalties table.**
   A penalty is "new" if it describes a mistake pattern that could recur in
   other submissions (e.g. using the wrong API, skipping a required step,
   misunderstanding a spec requirement).

2. **For each new pattern found:**

   **If the grader identified the penalty independently:** register it and
   proceed — no confirmation needed before the retroactive check.

   **If the professor proposed the penalty:** verify it in the student's code
   before registering it:
   - Confirm the issue is present with a code snippet or finding line.
   - If evidence is ambiguous, ask for clarification before applying.
   - If the issue is not found, report back with evidence and ask whether to
     apply it anyway.
   - Do not add to the Penalties table or update any analysis until the
     penalty is confirmed.

   Once the penalty is confirmed (by observation or by professor):
   - Add a row to the Penalties table in MIND.md:
     `| <short name> | <what the issue is> | <deduction> | <group-slug> |`
   - Immediately check all previously graded groups for the same issue
     (read their grading-analysis.md and, if needed, their code diff).
   - For each previously graded group where the same issue is present:
     - Update their `grading-analysis.md` — add a finding line and adjust
       the Score Decisions table.
     - Update their row in the MIND.md scoring table.
   - Report to the professor:

   ```
   ⚠️  New penalty registered: <name> (−N pts)
       Applied to current group: <group-slug>
       Retroactive check: <list affected previously-graded groups, or "none">
   ```

   Wait for the professor to confirm or adjust before proceeding.

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
