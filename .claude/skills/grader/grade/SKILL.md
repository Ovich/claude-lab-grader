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
the professor — do not modify the procedure directly.

---

## Step 0 — Load context from MIND.md

Read `labs/<lab-slug>/MIND.md`. Extract:
- Deadline.
- **Nature** (`guided` or `free`) — carry as a session variable used by Step 3 (diff scope), Step 5 (free-mode warning), and opt-in guards.
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

Check that the following Phase 1 items are ticked — stop if any are missing:
- `Submissions cloned`
- `Grading criteria defined`
- `Grading procedure approved`
- `Grading analysis template generated`

Also check `## Solution Diff` in MIND.md. If present, read it now —
use it as reference context during code analysis (Step 3) to understand
what a correct implementation looks like. Never quote solution code in
grading-analysis.md.

---

## Step 1 — Select group to grade

If `## Re-grading Tracker` exists in MIND.md, prioritise groups with
status `⏳ needs re-grading` over ungraded ones.

Find the first group in the scoring table whose Analysis column is empty
(or is flagged for re-grading).
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

**Skip this step** if the `## Grading Procedure` preamble in MIND.md does not
contain `**Hidden test coverage:** enabled`.

Invoke `grader/grade/opt-ins/hidden-tests`. It has full context from the
current session (lab-slug, group-slug, student diff, visible test results,
and the loaded Grading Procedure) and handles the complete hidden test flow:
tooling detection, gap analysis, scenario proposal, test generation and
execution, and mapping failures to deductions.

---

## Step 4 — Visual run

**Skip this step entirely if the grading plan has no `live` criteria.**

### Read the run plan

Read `## Run Plan` in MIND.md. Check `**AI-run:**`:

**`approved`** — check `**Stack:**` in the run plan:
- **`common`** — adapt the start command to the current group (substitute `<group-slug>` in any paths) and start the project.
- **`per-group`** — inspect the current group's repo to detect the run method (Dockerfile, package.json, Makefile, run script, etc.), applying any professor constraints noted in the run plan. Determine the start command before proceeding.

Save console output to `project-evidence.log` in the submission folder.

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

## Step 5 — Finalise and present

1. Update the group's row in the MIND.md scoring table:
   - Fill all criterion columns with their scores.
   - Fill Total and Grade.
   - Set the Analysis column to: `submissions/<group-slug>/grading-analysis.md`

2. If the group was in the Re-grading Tracker, mark it `✅ re-graded`.

3. Tick the Phase 2 counter: `All groups graded (N / Total)`.
   When all groups are done, fully tick `- [x] All groups graded`.

4. Scan the grading-analysis.md findings for any deduction that has no
   matching penalty ID in the Grading Procedure — this is a suspected new
   pattern. Note it (one line) to surface in the prompt below; do not
   register it unilaterally.

5. Present the summary and next-action prompt:

```
✅ <group-slug> — graded

  Tests:     N / Max   ← N/A if no automated criteria
  Manual:    N / Max   ← N/A if no manual criteria
  Live:      N / Max   ← N/A if no live criteria
  Penalty:   −X.X (N days late) / 0
  Total:     N / Max
  Grade:     X.X

  Notable: <one-line summary of key findings>

  ⚠️ Suspected new pattern: <description> (proposed: −N pts)   ← omit if none

  📋 Free lab — grading procedure is abstract. Please review the analysis
     above and confirm the scores reflect this group's specific project.   ← omit if guided

What next?
  1. Change something in the analysis
  2. Register new penalty pattern → grader/grade/opt-ins/penalty-patterns
  3. Next group: <suggested-group-slug> (N remaining)
     (or name a different group)
```

Handle the professor's choice:
- **Option 1:** make the requested change, recompute if needed, update
  MIND.md, then re-present this prompt.
- **Option 2:** invoke `grader/grade/opt-ins/penalty-patterns`.
- **Option 3:** confirm the group and loop back to Step 1 for the next group.
  If all groups are done, say so and suggest running `grader/feedback`.
