---
name: grader/criteria
description: >
  Generate or update the grading criteria table for a lab. On first run,
  analyses student submissions, solution diff, lab spec, and course material
  to propose a criteria breakdown. On re-run, shows current criteria and
  guides the professor through changes, handling scoring table archiving and
  re-grading tracking automatically. Always invokes grader/procedure when done.
---

# grader/criteria

## Guiding Principles

**Never modify this skill file.** All session-specific decisions belong in
MIND.md or `criteria-work.md`.

**Output goes to central MIND.md.** Use `labs/<lab-slug>/criteria-work.md`
as a scratch pad for intermediate analysis only. The final confirmed criteria
table belongs in MIND.md `## Grading Criteria`.

**Always invoke `grader/procedure` after criteria are confirmed.** The
grading procedure is a downstream product of criteria — generate or
regenerate it whenever criteria change.

---

## Step 0 — Prerequisites and status check

### 0a — Verify init prerequisites

Scan for `labs/*/MIND.md`. If not found:
> Init has not been run yet. Please run `grader/init` first.

Read MIND.md. Verify each of the following — report any that are unmet and stop:

| Requirement | Check |
|-------------|-------|
| MIND.md exists | Found at `labs/<lab-slug>/MIND.md` |
| Submissions cloned | Scoring Table has at least one row |
| Lab spec present | `labs/<lab-slug>/lab-spec.md` exists |

### 0b — Check current criteria status

Read `## Grading Criteria` in MIND.md.

- **Empty / placeholder** → first run. Proceed to Step 1.
- **Already populated** → re-run. Jump to the **Re-run flow** section below.

---

## Step 1 — Gather resources (first run)

Store all intermediate analysis in `labs/<lab-slug>/criteria-work.md`.

### 1a — Solution diff

Check `## Solution Diff` in MIND.md.
- Already present → use it.
- Not present but solution ref is set in the MIND.md header → generate it:

```bash
cd _global/lab-solutions/<lab-slug>
git diff <template-branch>..<solution-branch> -- <relevant dirs>
```

Write result to `## Solution Diff` in MIND.md. Tick `Solution diff generated`.

- No solution ref → note the absence and continue.

### 1b — Sample student submissions

Select 2–3 submissions to analyse. Aim for variety:
- One with the most commits or largest diff (likely complete)
- One mid-range
- One with minimal changes (likely weak or partial)

For each, compute the diff that represents student work:

**Template-based lab** (students started from a provided template):
```bash
# find the first student commit (exclude bot commits)
BASE=$(git -C labs/<lab-slug>/submissions/<group-slug> log \
  --oneline --all --reverse \
  | grep -v "github-classroom\[bot\]" | head -1 | awk '{print $1}')
git -C labs/<lab-slug>/submissions/<group-slug> diff $BASE HEAD
```

**Free-form lab** (no template — students started from scratch):
```bash
BASE=$(git -C labs/<lab-slug>/submissions/<group-slug> \
  rev-list --max-parents=0 HEAD)
git -C labs/<lab-slug>/submissions/<group-slug> diff $BASE HEAD
```

Save each diff under a labelled section in `criteria-work.md`.

### 1c — Course material (optional)

Check whether `_global/course-material/` is present.

- **Present** → search for pages related to the lab topic (match lab name,
  slug, or key terms from lab-spec.md). Extract the relevant sections and
  append to `criteria-work.md`.
- **Absent** → inform the professor:
  > No course material was found. Adding it to `_global/course-material/`
  > would help generate more accurate criteria and grading procedures.
  > This is optional — I can continue without it.

### 1d — Lab spec

Re-read `labs/<lab-slug>/lab-spec.md`. Look for:
- Explicit grading sections or point breakdowns
- Lists of expected deliverables or features
- Technical constraints or prohibited patterns

Append findings to `criteria-work.md`.

### 1e — Standard plugins

Standard plugins are reusable criteria blocks that apply across many labs.
They are opt-in — always ask the professor before including any.

Present the available plugins and ask which to include:

```
Standard criteria plugins available:

  [A] Automated tests
      Scores based on a test suite included in the repo.
      Suggested: N pts (auto-detected from repo — adjust as needed)
      Include? If yes: how many points total?

  [B] Teamwork (git collaboration)
      Evaluates how the team collaborated using git. Checks:
        • Multiple distinct commit authors
        • Commits spread over time (not all submitted in the last hours)
        • Use of pull requests or merge commits
      Suggested: 2–5 pts depending on lab weight
      Include? If yes: how many points?

Which plugins do you want to include, and with how many points each?
(Answer "none" to skip all, or name specific ones with their point value)
```

Notes:
- **Automated tests** are proposed only if a test suite is detected in the
  sampled submissions (e.g. `package.json` with a test script, `Makefile`
  with a test target). Do not propose if no test suite is found.
- **Teamwork** is always proposed for group labs. For individual labs, skip
  it unless the professor brings it up.
- To determine whether the lab is individual or group, check the number of
  distinct authors across a few submissions. If most repos have only one
  author, treat it as individual and note this when presenting the plugin.

Save the professor's plugin choices to `criteria-work.md` before proceeding.

---

## Step 2 — Analyse and propose criteria

Using everything in `criteria-work.md`, draft a full criteria proposal
combining lab-specific criteria and any selected plugins.

**Weighting guidance:**
- If the lab spec states explicit point values, use them exactly.
- If not, propose weights based on relative importance: complexity of the
  implementation (inferred from solution diff size and submission variation),
  emphasis in the lab spec, and whether the criterion is automated or manual.
  Show your reasoning briefly next to each proposed value so the professor
  can adjust it.
- Mark each value as `explicit` (from spec) or `proposed` (your estimate).

- Each criterion maps to a concrete, gradable deliverable.
- Distinguish `automated` (test suite), `manual` (code review / visual),
  `plugin` (standard block).
- Note confidence: `explicit from spec` vs `proposed`.

Present the proposal to the professor in two explicit confirmation blocks:

**Block 1 — Criteria and grade formula:**
```
Proposed grading criteria for <lab-name>:

  Criterion              Max   Type        Weight basis
  ──────────────────────────────────────────────────────────────────
  CriterionA              8    automated   explicit from spec
  CriterionB              4    manual      proposed — largest diff section
  Teamwork                3    plugin      proposed — group lab, 3 signals
  ...
  Total                  /25

  Grade formula: round((points / 25 × 5) + 1 − penalties, 0.1)

Does this look right? Adjust any points or add/remove criteria before I proceed.
```

**Block 2 — Late penalty formula:**
```
Late penalty proposal:

  Default: −0.1 per started late day
  (Any fraction of a 24 h period past the deadline counts as one day —
   e.g. 1 h late = 1 day = −0.1; 25 h late = 2 days = −0.2)

  Source: <"stated in lab-spec.md" / "course default — not in spec">

  Is this formula correct?
  If different, please provide:
    • Deduction per day (e.g. −0.1, −0.5 …)
    • How partial days are counted (started day / full day / hours)
    • Any grace period or extension policy
```

Wait for explicit confirmation on both blocks before writing to MIND.md.

---

## Step 3 — Write to MIND.md and invoke procedure

Once confirmed:
1. Write the criteria table to `## Grading Criteria` in MIND.md.
2. Update the Scoring Table header with criterion columns.
3. Tick `Grading criteria defined` in the Phase 1 checklist.
4. Invoke `grader/procedure` to generate the grading procedure.

---

## Re-run flow — criteria already exist

### R1 — Show current state

Present a compact summary:
```
Current criteria for <lab-name>:
  CriterionA   8 pts   automated
  CriterionB   4 pts   manual
  ...
  Total: /25   Grade: (pts/25 × 5) + 1 − penalties

Grading progress: N / M groups graded
```

Ask the professor:
> What would you like to change?

### R2 — Classify the change

| Change type | Examples |
|-------------|----------|
| **Points-only** | "CriterionA from 8 to 10", "reduce CriterionB to 3" |
| **Criterion added or removed** | "Add error handling criterion", "Drop live testing" |
| **Both** | Mix of the above |

### R3a — Points-only change

1. Update the criteria table in MIND.md.
2. Offer to recalculate all existing scores using the new weights and update
   the Scoring Table accordingly.
3. Invoke `grader/procedure` to regenerate affected procedure subsections.

### R3b — Criterion added or removed (grading in progress)

1. **Archive** the current Scoring Table by renaming its heading:
   `## Scoring Table` → `## Scoring Table [archived — superseded <YYYY-MM-DD>]`

2. **Create** a new `## Scoring Table` with the updated criterion columns.

3. **Add** a `## Re-grading Tracker` section to MIND.md:

```markdown
## Re-grading Tracker
<!-- Groups that must be re-graded after criteria change on <YYYY-MM-DD>.  -->
<!-- grader/grade: check this table before grading — redo listed groups.   -->

| Group | Status | Notes |
|-------|--------|-------|
| <group-slug> | ⏳ needs re-grading | criterion added: <name> |
```

4. Populate the tracker with every group that had already been graded.

5. Update the criteria table in MIND.md.

6. Invoke `grader/procedure` to regenerate the full procedure.

Report clearly what changed and what the professor needs to do next.
