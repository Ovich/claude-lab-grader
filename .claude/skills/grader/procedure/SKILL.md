---
name: grader/procedure
description: >
  Generate or regenerate the grading procedure for a lab. Produces one
  subsection per criterion describing what to look for, what earns full marks,
  and what common deductions look like. Reads criteria from MIND.md and uses
  the solution diff, criteria-work.md, and lab spec as context. Invoked by
  grader/criteria and grader/init after criteria are confirmed. Can also be
  run standalone to refine the procedure without changing criteria.
---

# grader/procedure

## Guiding Principles

**Never modify this skill file.** All output goes to `## Grading Procedure`
in MIND.md.

**The procedure is derived from criteria.** If criteria change, affected
procedure subsections must be regenerated. Always regenerate when called
after a criteria update.

**The procedure guides human judgment — not just test results.** For
automated criteria, describe what the test suite covers and what to check
manually if tests pass but code quality is poor. For manual criteria, be
concrete about exactly what to inspect.

---

## Step 0 — Prerequisites

Read MIND.md. If `## Grading Criteria` is empty or missing:
> No criteria found in MIND.md. Please run `grader/criteria` first.

Check if `## Grading Procedure` already exists and is populated:
- **Empty / placeholder** → generate fresh. Proceed to Step 1.
- **Populated** → ask the professor:
  > A grading procedure already exists. Do you want to:
  > 1. Regenerate it fully (replaces everything)
  > 2. Update only specific criteria subsections
  > 3. Add a new penalty to an existing criterion
  > 4. Retire or replace an existing penalty

### Option 3 — Add a new penalty

Ask for: which criterion, description, deduction amount, and ID prefix.
Assign the next available index for that prefix (e.g. if `IMPL-01` exists,
use `IMPL-02`). Add the row to the criterion's "Common deductions" table.
Confirm with the professor, then write to MIND.md. Status remains READY —
no re-approval needed for a penalty addition.

### Option 4 — Retire or replace a penalty

Ask which penalty ID to retire and whether it is being replaced.
- **Disable:** remove the row from the procedure. Add to `## Penalty Archive`:
  `| <ID> | <description> | <deduction> | disabled | — |`
- **Replace:** remove the old row, add it to archive with
  `replaced-by: <new-ID>`, then add the new penalty row with the fresh ID.
Confirm with the professor before writing. Status remains READY.

---

## Step 1 — Gather context

Read the following in priority order:

1. `## Grading Criteria` in MIND.md — the criteria to write procedure for
2. `## Solution Diff` in MIND.md — what a correct implementation looks like
3. `labs/<lab-slug>/criteria-work.md` — submission samples and analysis (if present)
4. `labs/<lab-slug>/lab-spec.md` — expected behaviour, constraints, prohibited patterns
5. Course material — if `**Course material:**` in MIND.md is not `n/a`, extract relevant content (see Step 1a)

### Step 1a — Extract course material excerpts (optional)

If the `**Course material:**` field in MIND.md points to a path (not `n/a`),
scan that path for content related to the
current criteria: concept definitions, expected patterns, constraints, or
common pitfalls that are explicitly taught in the course.

For each relevant excerpt found, note:
- The source file and section
- Which criterion it relates to
- The key teaching point (one or two sentences)

Save all excerpts to `labs/<lab-slug>/procedure-course-refs.md`:

```markdown
# Course Material References — <lab-name>

## <Criterion name>
- **Source:** `_global/course-material/<file>#<section>`
- **Teaching point:** <what students were taught, relevant to grading this criterion>

## <Criterion name>
...
```

If no relevant content is found in the course material, skip this file.
Use `procedure-course-refs.md` in Step 2 to enrich "What to look for"
and to add `📖 See:` references in the procedure where applicable.

---

## Step 2 — Generate procedure

Start the procedure with a **preamble block** that carries forward the
key grading parameters from `## Grading Criteria` in MIND.md:

```markdown
**Grade formula:** <!-- e.g. round((Total / Max × 5) + 1 − penalties, 0.1) -->
**Late penalty:** <!-- e.g. −0.5 per started late day, max −2 -->
**Hidden test coverage:** <!-- enabled / disabled -->
```

Copy exact values from the criteria. If hidden test coverage is absent,
write `disabled`. This preamble is the only place `grader/grade` needs
to look — it must not read `## Grading Criteria` at all.

Then write one subsection per criterion using this template:

```markdown
### <Criterion name> [max pts]

**Source:** automated / manual / live

**What to look for:**
- <concrete thing to check — file, function, behaviour>
- <concrete thing to check>

**Full marks:**
- <what a correct, complete implementation looks like>

**Common deductions:**
| Penalty ID | Description | Deduction |
|------------|-------------|-----------|
| IMPL-01 | <description> | −N pts |
| IMPL-02 | <description> | −N pts |

**Reference:** lab-spec.md § <section>   <!-- omit if not applicable -->
```

Guidelines per source type:

- **Automated** — name the test file and test group. Add a manual check note
  for cases where tests pass but the implementation is structurally wrong or
  copies the solution pattern without understanding.
- **Manual** — name the exact file and function(s) to inspect. Describe the
  expected behaviour in one sentence and the most common mistake in one sentence.
- **Live** — list the exact interactions to perform (key press, mouse move,
  click target) and the expected visual or console result for each.
  **Live criteria always require the professor to perform the runtime checks**
  — the AI starts the project but the professor does the interactions and
  reports findings. A run plan is generated separately (see Step 2a).

**Penalty ID naming:** use a short prefix reflecting the criterion or mistake
category, followed by a two-digit index (e.g. `IMPL-01`, `API-02`, `LATE-01`).
IDs must be unique across the entire procedure. They are referenced in
`## Penalty Archive` when a penalty is retired or replaced.

Infer common deductions from:
- The solution diff (what the correct implementation does that a naive one
  might skip or mishandle)
- Submission samples in `criteria-work.md` (actual mistakes seen)
- Lab spec constraints (anything explicitly required or prohibited)

If none of these are available for a criterion, propose plausible deductions
based on the criterion description and typical student mistakes for that type
of task. Mark these as `[proposed — not yet observed]` so the professor can
validate them.

---

## Step 2a — Generate run plan (live criteria only)

Skip this step if no criterion has `**Source:** live`.

If live criteria exist, a run plan is **required** — either AI-managed or
professor-managed. The only valid reason to skip it is if the professor
explicitly decides the live run is entirely their responsibility (opted-out).
In that case the grader will collect findings from the professor without
starting anything.

Ask the professor upfront:
> Live criteria require runtime testing. For each group, I can start the
> project automatically and you perform the interactions — or you can run
> everything yourself and report the findings to me.
> 1. AI starts the project — professor tests
> 2. Professor runs everything independently

- **Option 2 chosen:** set `**AI-run:** opted-out` in `## Run Plan` in
  MIND.md and skip the rest of this step. No run command is needed.
- **Option 1 chosen:** proceed below to detect the run method.

Inspect the starter code or solution reference to determine how the project
should be started. Look for:
- A `Dockerfile` or `docker-compose.yml` → prefer Docker
- A language-specific entry point (`package.json`, `Makefile`, `pom.xml`, etc.)
- A run script in the repo

Generate a run plan and write it to `## Run Plan` in MIND.md:

```markdown
## Run Plan
**Start command:** <!-- e.g. docker run ... / npm start / make run -->
**Port:** <!-- e.g. http://localhost:3333 -->
**AI-run:** <!-- approved / opted-out — filled after professor confirms -->
**Notes:** <!-- any setup steps, env vars, or known issues -->
```

Include the run plan in the DRAFT presented to the professor in Step 3.
Ask the professor:
> Live criteria require runtime testing. For each group, I can start the
> project automatically and you perform the interactions — or you can run
> everything yourself and report the findings to me.
> Which do you prefer?
> 1. AI starts the project — professor tests
> 2. Professor runs everything independently

Set `**AI-run:**` to `approved` or `opted-out` based on the answer.

---

## Step 3 — Save as DRAFT and get approval

1. Write the procedure to `## Grading Procedure` in MIND.md. Place
   `<!-- status: DRAFT -->` on the first line of the section, before any
   criterion subsections.
2. Inform the professor:
   > The grading procedure has been saved to MIND.md as **DRAFT**.
   > Please review it there (scroll to `## Grading Procedure`) and let me
   > know if you'd like any changes.
3. Wait for the professor's response:
   - **No changes / approved:** proceed to Step 3a.
   - **Changes requested:** brainstorm with the professor, update the
     relevant subsection(s) in MIND.md (add, remove, or adjust penalty rows
     or "What to look for" items), then ask again. Repeat until approved.

When updating penalty entries based on professor feedback:
- Retiring a penalty: remove it from the procedure and add it to
  `## Penalty Archive` in MIND.md with status `disabled`.
- Replacing a penalty: remove the old row from the procedure, add it to
  the archive with status `replaced-by: <new-ID>`, and add the new row
  with a fresh ID.

### Step 3a — Mark as READY

Replace `<!-- status: DRAFT -->` with `<!-- status: READY -->` in the
`## Grading Procedure` section of MIND.md.

If this skill was invoked by `grader/init` or `grader/criteria`, tick
`Grading procedure approved` in the Phase 1 checklist in MIND.md.

Report:
```
✅ Grading procedure approved — <N> criteria covered.
Next: run grader/grade to begin grading groups.
```
