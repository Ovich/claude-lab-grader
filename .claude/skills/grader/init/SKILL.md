---
name: grader/init
description: >
  Initialize a grading session for one lab assignment. Gathers all resources,
  clones student submissions, defines grading criteria, and generates MIND.md
  as the central tracker for the entire grading session. Run once per lab.
  All subsequent grader/* skills read MIND.md for context.
---

# grader/init

## Guiding Principles

**Never modify this skill file.** The skill is read-only during execution.
Any session-specific information — professor instructions, notes, decisions,
discoveries — belongs in `MIND.md`, which is the session's central memory.
If the professor asks to remember something, write it to MIND.md, not here.

**Workspace isolation — strictly enforced.**
All file reads, writes, searches, and shell commands must stay within the
current working directory. Never access, read, or reference anything outside
it — no `../`, no absolute paths to other folders, no parent directory scans.
If a resource seems to be missing, ask the professor rather than looking
outside the workspace.

This skill enforces a default structure, workflow, and conventions. They exist
for consistency and to make all `grader/*` skills work reliably.

If the professor expresses a preference that differs from the defaults:
- Analyse the request: note what it changes, any trade-offs or risks.
- Present your analysis and ask for explicit confirmation before proceeding.
- If confirmed, adapt MIND.md to reflect the agreed deviation so all subsequent
  skills inherit it. Never silently deviate from the defaults.

---

## Purpose

Set up everything needed to grade one lab assignment. The single output is a
fully populated `MIND.md` in the lab's dedicated folder. Every `grader/*`
skill reads MIND.md as its primary source of context — it is the single
source of truth for the entire grading session.

---

## Step 0 — Audit the workspace

Before doing anything, scan the entire workspace and build a picture of what
is already present. This determines whether to start fresh, reorganise, or
resume an existing session.

### 0a — Check for existing MIND.md

Scan for any file matching `labs/*/MIND.md`.

- **Found:** read it, report the current phase and completion status to the
  professor, and ask whether to **resume** (continue from the last unchecked
  step) or **reinitialise** (start over). If resuming, skip to the first
  unchecked step.
- **Not found:** continue to 0b.

### 0b — Scan the workspace for existing content

List all files and folders at the workspace root and one level deep. For each
item, attempt to classify it into one of the following categories:

| Category | Signals |
|----------|---------|
| **Course material** | `course-material/`, `slides/`, `docs/`, folder with many markdown/HTML pages |
| **Lab solution** | folder named after a lab slug, contains `src/` + git history, has a `solution` branch |
| **Student submissions** | folder or nested folders containing multiple git repos with student commits |
| **Grading criteria** | `GRADING_CRITERIA*.md`, `criteria*.md`, `rubric*.md` |
| **Grading template / old MIND** | `*grading-template.md`, `*grading-table*` |
| **Solution diff** | `*solution-diff.md` |
| **Test runner scripts** | `run-tests.sh`, `run-project.sh` |
| **Unknown** | anything that does not fit the above |

### 0c — Report and resolve

Present the classification to the professor in a compact table:

```
Found in workspace:
  ✅ course-material/         → _global/course-material/       (will move)
  ✅ lab-solutions/4-tetris-ii → _global/lab-solutions/4-tetris-ii (already correct)
  ✅ lab-submissions/4-tetris-ii/... → labs/4-tetris-ii/submissions/ (will reorganise)
  ✅ GRADING_CRITERIA_lab4.md → labs/4-tetris-ii/ (will move)
  ✅ 4-tetris-ii-solution-diff.md → labs/4-tetris-ii/ (will move)
  ❓ WEB-A-2026.xlsm          → unknown — please clarify
  ❓ some-other-folder/        → unknown — please clarify
```

For each **unknown** item, ask the professor:
> What is `<item>`? Should I move it somewhere, leave it, or ignore it?

Do not move or delete anything until the professor has confirmed the plan.

### 0d — Pre-check init steps

Based on what was found, mark the corresponding Phase 1 checklist items as
already done when the MIND.md skeleton is created in Step 3:

| If found | Pre-check |
|----------|-----------|
| Course material present | `Course material located` |
| Solution folder present | `Solution diff` can be generated immediately |
| Submissions folder with repos | `Submissions cloned` (verify count) |
| Criteria file present | `Grading criteria defined` |

Report to the professor which steps will be pre-checked and which still need
to be completed.

---

## Step 1 — Gather lab information

Ask the professor for the following. Collect **all answers before proceeding**.
Mark optional items clearly.

| # | Information | Required |
|---|-------------|----------|
| 1 | Lab name and number (e.g. "4 · Tetris II") | ✅ |
| 2 | Course name (e.g. "TWEB26 · HEIG-VD") | ✅ |
| 3 | Student submissions source (see options below) | ✅ |
| 4 | Solution reference — git URL or local path, and which branch | optional ⭐ highly recommended — enables solution diff, which significantly improves grading accuracy |
| 5 | Course material — already present, or provide URL | optional |
| 6 | Existing grading criteria — file path or document | optional |

**Submissions source options (examples — not exhaustive):**
- A GitHub Classroom assignment URL
- A GitHub organisation or team
- A list of GitHub / GitLab / Bitbucket repo URLs
- A local folder already containing repos
- Any other source the professor describes

Do not ask about tooling yet — that is handled in Step 1b once all
information is known.

---

## Step 1b — Identify and verify required tools

Once all lab information is collected, determine which CLI tools are needed
based on the submission source, platform, and test setup described.

### Identify required tools

Derive the tool list from what the professor provided. Examples:

| Situation | Likely tools needed |
|-----------|-------------------|
| GitHub repos / Classroom | `git`, `gh` |
| GitLab repos | `git`, `glab` |
| Running JS tests | `node`, `npm` |
| Running tests in Docker | `docker` |
| Any git repo | `git` (always required) |

This list is not exhaustive — use judgement based on the actual submission
source and test setup described.

### Check availability

For each required tool, check if it is already installed:

```bash
<tool> --version 2>/dev/null && echo "ok" || echo "missing"
```

Prefer tools already available on the system. Only flag a tool as missing
if there is no installed alternative that can do the same job.

### Handle missing tools

For each missing tool, tell the professor clearly:

> To work with `<source>` I need `<tool>`. It is not installed.
> You can install it with: `<install command>`
> Or run: `! <install command>` directly in this terminal.

For tools that require authentication (e.g. `gh`, `glab`), provide the
auth command after installation:

> Once installed, authenticate with: `! <auth command>`

Wait for confirmation that all required tools are available before
proceeding.

---

## Step 2 — Set up folder structure

Create the following layout under the workspace root. Use the audit from
Step 0 to move existing files into their correct locations. Only move items
the professor has confirmed in Step 0c — never move or delete unresolved items.

```
<workspace>/
├── _global/
│   ├── course-material/         ← clone here if URL provided
│   └── lab-solutions/
│       └── <lab-slug>/          ← clone here if URL provided
└── labs/
    └── <lab-slug>/
        ├── MIND.md              ← created in Step 3
        ├── lab-spec.md          ← saved in Step 1
        └── submissions/         ← populated in Step 4
            └── <group-slug>/
```

**Slug convention:** lowercase, hyphens, lab number prefix.
Examples: `4-tetris-ii`, `2-tetris-i`.

After creating the structure, immediately create a skeleton MIND.md
(Step 3) so all subsequent progress is tracked inside it.

When writing the skeleton, populate the **Workspace Layout** section using
the actual paths and items confirmed in Steps 0–2:
- Include only `_global/` subdirectories that are used by this lab.
- List every file present under `labs/<lab-slug>/` (criteria file, diff, etc.).
- Expand `submissions/` with one line per cloned group slug (filled after Step 4).
- Add a short description after each `←` explaining the item's role.

---

## Step 3 — Create MIND.md skeleton

Write `labs/<lab-slug>/MIND.md` with the structure below. Fill in what is
known now; leave `<!-- TODO -->` placeholders for later steps.
Update the checklist items as each step completes — never leave the file
in a state that does not reflect actual progress.

```markdown
# MIND — <lab-name>
**Course:** <course>
**Deadline:** <!-- filled in Step 4 -->
**Classroom:** <link or n/a>
**Lab spec:** lab-spec.md
**Solution ref:** <path or n/a>
**Course material:** <path or n/a>

---

## Workspace Layout
<!-- Effective folder structure for this lab — globals included, sibling labs omitted.  -->
<!-- Populate from Steps 0–2 findings. Update if paths change during the session.      -->

```
<workspace>/
├── _global/
│   ├── course-material/         ← <!-- description, e.g. "shared slides and reference pages" -->
│   └── lab-solutions/
│       └── <lab-slug>/          ← <!-- description, e.g. "reference implementation (template: main, solution: origin/solution)" -->
└── labs/
    └── <lab-slug>/              ← this lab's grading session root
        ├── MIND.md              ← this file — central tracker for the grading session
        ├── lab-spec.md          ← lab assignment specification (converted to markdown)
        └── submissions/         ← one folder per student group
            └── <group-slug>/    ← cloned student repo
```
<!-- Remove any global rows that are not used by this lab (e.g. no solution ref → remove lab-solutions row). -->
<!-- Add rows for any other files present (e.g. grading-criteria.md, solution-diff.md).                    -->

---

## Phase 1 · Init
- [ ] Lab info gathered
- [ ] Folder structure created
- [ ] Submissions cloned (0 / ?)
- [ ] Grading criteria defined
- [ ] Grading procedure approved
- [ ] Solution diff generated
- [ ] Scoring table initialised
- [ ] Grading analysis template generated

## Phase 2 · Grade
- [ ] All groups graded (0 / ?)

## Phase 3 · Feedback
- [ ] All feedback published (0 / ?)

---

## Grading Criteria
<!-- filled by Step 5 -->

---

## Grading Procedure
<!-- Generated by grader/procedure. First line will be: <!-- status: DRAFT --> -->
<!-- until professor approves it, then changed to: <!-- status: READY -->    -->
<!-- grader/grade is blocked until status is READY.                          -->

---

## Solution Diff
<!-- filled by Step 5 if solution ref provided -->

---

## Grading Analysis Template
<!-- filled by Step 6 -->

---

## Run Plan
<!-- Generated by grader/procedure when live criteria exist.         -->
<!-- AI-run: approved = AI starts the project, professor tests.      -->
<!-- AI-run: opted-out = professor runs everything independently.    -->

---

## Penalty Archive
<!-- Penalties retired during grading — disabled or replaced by newer versions. -->
<!-- Active penalties live in the Grading Procedure with short IDs.             -->
<!-- When a penalty is retired, move it here from the procedure.                -->

| ID | Description | Deduction | Status | Replaced by |
|----|-------------|-----------|--------|-------------|

---

## Scoring Table
<!-- filled by Step 4 -->
<!-- single source of truth — update scores here as groups are graded -->
<!-- Analysis column: relative path to submissions/<group-slug>/grading-analysis.md -->
```

Tick `Lab info gathered` and `Folder structure created`.

---

## Step 4 — Clone and index submissions

Using the tools confirmed in Step 1b, fetch submissions from whatever source
the professor provided. The goal is always the same: repos cloned into
`labs/<lab-slug>/submissions/<group-slug>/`.

### Remote repos (any platform)

Use the available CLI to list and clone repos. Examples:

```bash
# GitHub Classroom — fetch roster and deadline, then clone
gh api classrooms/<id>/assignments         # deadline
gh api assignments/<id>/accepted_assignments | jq '.[].github_username'
git clone <repo-url> labs/<lab-slug>/submissions/<group-slug>

# Plain git — clone a known list
git clone <repo-url> labs/<lab-slug>/submissions/<group-slug>

# GitLab — list group repos then clone
glab api groups/<id>/projects | jq '.[].ssh_url_to_repo'
git clone <repo-url> labs/<lab-slug>/submissions/<group-slug>
```

Fetch the deadline from the platform if available (Classroom API, GitLab
milestone, etc.) and store it in the MIND.md header. If no deadline is
available programmatically, ask the professor.

Clone repos in parallel where the tool supports it.

Derive `<group-slug>` by stripping the lab prefix from the repo name
(e.g. `4-tetris-ii-bekele` → `bekele`).

### Local folder (repos already present)

Repos detected in Step 0b — move into `labs/<lab-slug>/submissions/`
following the naming convention. No cloning needed.

---

After cloning, inspect each repo:

```bash
git log --oneline --all -- | grep -v "github-classroom\[bot\]"
```

- Repos with **zero student commits**: mark as `not-submitted` in the
  scoring table; record 0 for all criteria.
- Repos with student commits: note the latest commit date for late
  penalty calculation.

**Populate the Scoring Table** in MIND.md: one row per group.

| Group | Students | Repo | <criteria columns> | Total | Grade | Analysis |
|-------|----------|------|--------------------|-------|-------|----------|

Criteria columns are added in Step 5 once criteria are known. Leave the
header with `[criteria TBD]` for now.

The `Analysis` column holds a relative path to the group's individual
analysis file (e.g. `submissions/bekele/grading-analysis.md`), filled
by `grader/grade` as each group is graded.

Tick `Submissions cloned (N / N)`.

---

## Step 4b — Locate lab specification

The lab spec is mandatory before defining criteria. Search in this order —
stop at the first hit:

1. **Solution reference** — look for a README, spec document, or lab
   instructions page in `_global/lab-solutions/<lab-slug>/`.
2. **Course material** — search `_global/course-material/` for a page or
   document matching the lab name/number.
3. **Student repo README** — open any cloned submission and check the README
   for a link or embedded description of the lab assignment.
4. **Professor** — if none of the above yields a result:
   > I could not find the lab specification in the solution reference, course
   > material, or student READMEs. Can you point me to it or share its content?

Once found, save the full content as `labs/<lab-slug>/lab-spec.md`
(convert from HTML or other formats to markdown). Update the MIND.md header:

```
**Lab spec:** lab-spec.md
```

Do not proceed to Step 5 until `lab-spec.md` exists.

---

## Step 5 — Define grading criteria

Ask the professor:
> Do you have an existing criteria document or rubric to load, or should I
> generate criteria from scratch based on the lab materials?

### If existing criteria provided

1. Ask the professor to share or point to the criteria document.
2. Extract the criteria table, max points, source (automated / manual), and
   grade formula.
3. Confirm the late penalty formula explicitly (same two-block format as
   `grader/criteria` Step 2).
4. Write the confirmed criteria to `## Grading Criteria` in MIND.md.
5. Update the Scoring Table header with criterion columns.
6. Tick `Grading criteria defined`.
7. If a solution reference is set in the MIND.md header, generate the
   solution diff now:

```bash
cd _global/lab-solutions/<lab-slug>
git diff <template-branch>..<solution-branch> -- <relevant dirs>
```

Write the result to `## Solution Diff` in MIND.md. Tick `Solution diff generated`.

8. Invoke `grader/procedure` to generate the grading procedure.

### If generating from scratch

If a solution reference is set in the MIND.md header, generate the solution
diff now so it is ready for `grader/criteria` to use:

```bash
cd _global/lab-solutions/<lab-slug>
git diff <template-branch>..<solution-branch> -- <relevant dirs>
```

Write the result to `## Solution Diff` in MIND.md. Tick `Solution diff generated`.

Then invoke `grader/criteria`. It will analyse submissions, the solution
diff, lab spec, and course material, propose criteria, write them to MIND.md,
and invoke `grader/procedure` when done.

Tick `Grading criteria defined` once `grader/criteria` completes.

---

## Step 6 — Generate grading analysis template

Using the grading criteria, lab spec, and solution diff, generate the
**Grading Analysis Template** section in MIND.md.

### Rules

**Snippets come from student code — never from the solution reference.**
The solution diff is used only to understand which files and functions
students were asked to implement. The actual code shown in the template
is extracted from the student submission diff (what changed vs the
starting template).

- One subsection per file students were asked to implement (inferred from
  criteria and solution diff).
- For each subsection, prefer showing the **full function or class** so the
  student has complete visibility of what was reviewed. Only trim to the
  relevant lines if the function or class is too large (rough guide: over
  ~30 lines).
- If the lab gives significant implementation freedom (no strict template),
  extract the function or block most relevant to each criterion — still
  full, but scoped to what is being evaluated.
- Each snippet is followed by finding lines: `> ✅ / ⚠️ / ❌`
- If course material is present and a deduction relates to a specific
  concept covered there, add a reference:
  `> 📖 See: lab-spec.md § <section> / course-material/<page>#<anchor>`
- Ends with a **Score Decisions** table matching the criteria columns.
- Ends with the grade formula and a Comment field.

```markdown
## Grading Analysis Template

### `<path/to/file>`
```
// student snippet — relevant to the finding
```
> ✅ ...
> ⚠️ ... 📖 See: lab-spec.md § Expected behaviour
> ❌ ...

---

### Score Decisions

| Criterion | Score | Reason |
|-----------|-------|--------|
| CriterionA [N] | | |
| CriterionB [N] | | |
| **Total** | **/Max** | |

**Final Grade:** round((Total / Max × 5) + 1 − Penalties, 0.1) = **X.X**

**Comment:**
```

Present the generated template to the professor and ask for confirmation
before writing it to MIND.md.

Tick `Grading analysis template generated`.

---

## Completion

When all Phase 1 checklist items are ticked, report a summary to the
professor:

```
✅ Init complete — <lab-name>

  Groups found:      N
  With submissions:  N
  Not submitted:     N

  Criteria:
    - CriterionA  [max]  automated
    - CriterionB  [max]  manual
    ...
    Total: /Max  →  grade formula: (pts/max × 5) + 1 − penalties

  Issues:
    - <any auth errors, missing repos, test failures>

Next: run grader/grade to begin grading groups.
```
