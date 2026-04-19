---
name: grader/grade/opt-ins/penalty-patterns
description: >
  Register a new penalty pattern discovered during grading, apply it to
  the current group, and retroactively check all previously graded groups.
  Invoked by grader/grade when the professor chooses to register a new
  pattern at the end of a grading session.
---

# grader/grade/opt-ins/penalty-patterns

## Guiding Principles

**Never modify this skill file.** All output goes to `grading-analysis.md`
files and MIND.md.

**Verify before applying.** Never accept a professor-proposed penalty
without confirming the issue is present in the student's code.

**Retroactive consistency.** Once a penalty is registered, every
previously graded group must be checked.

---

## Context expected from grader/grade

When invoked, the following context is already available in the session:

- `<lab-slug>` and `<group-slug>` (the group just graded)
- The current `grading-analysis.md` for this group
- The Grading Procedure loaded from MIND.md
- Any unregistered deductions flagged by grader/grade

---

## Step 1 — Identify the new pattern

If grader/grade flagged a suspected pattern, present it for confirmation:

> ⚠️ Suspected new pattern: `<description>` (proposed: −N pts)
> Found in: `<file>:<lines>`
> Should I register this as a new penalty?

If the professor is proposing a pattern not yet in the analysis:
1. **Verify first — do not accept blindly.** Check whether the issue is
   actually present in the student's code.
2. If clearly confirmed: proceed to Step 2.
3. If evidence is ambiguous:
   > I checked `<file>:<lines>` and I'm not certain this applies because
   > `<what you observed>`. Could you clarify what specifically to look for?
4. If not present:
   > I looked at `<file>:<lines>` and did not find this issue — here is
   > what I see: `<snippet>`. Should I still apply the penalty?

Never apply a penalty without verification or explicit professor override.

---

## Step 2 — Register via grader/procedure

Once confirmed, invoke `grader/procedure` and tell it to use **option 3 — Add a new penalty** directly, skipping the menu. Pass the criterion name, description, deduction amount, and suggested ID prefix so it can assign the next available index without prompting.

> ⚠️ New penalty pattern confirmed: `<description>` (proposed: −N pts)
> Invoking `grader/procedure` (option 3 — add penalty) to register it.

`grader/procedure` will assign an ID (e.g. `IMPL-02`), add the row to the criterion's "Common deductions" table, and keep status READY — no re-approval needed.

---

## Step 3 — Apply to current group

Update the current group's `grading-analysis.md`:
- Add a finding line: `> ❌ <penalty-ID>: <description>`
- Adjust the Score Decisions table for the affected criterion.
- Recompute the final grade.
- Update the group's row in the MIND.md scoring table.

---

## Step 4 — Retroactive check (opt-in)

Before running any retroactive check, ask the professor:

```
⚠️ <ID> registered. N groups have already been graded.

Retroactive check recommended — ensures all groups are penalised
consistently for the same mistake. Without it, earlier groups keep
an advantage they shouldn't have.
```

**If `**Nature:** free` is set in MIND.md**, add:

```
This is a free lab — groups worked on different projects. This penalty
may not apply to other groups. Should it be:
  1. Group-scoped — don't check others (their projects are different)
  2. Cross-group — run retroactive check (I'll go one by one)
```

**If `**Nature:** guided`**, ask:

```
This can take a few minutes depending on cohort size. Want to run it now?
  1. Yes — I'll go group by group and confirm with you after each
  2. No — skip for now (you can re-run grader/grade on specific groups later)
```

If the professor chooses to skip (option 1 in free, option 2 in guided),
stop here.

### Per-group retroactive flow

Go through each previously graded group **one at a time**:

1. Read their `grading-analysis.md` and, if needed, their code diff.
2. Report findings to the professor:

```
Checking <group-slug>…
  ❌ Issue found at <file>:<lines> — apply −N pts?
  ✅ Not affected — moving on.
```

3. Wait for confirmation before updating anything.
4. If confirmed: update their `grading-analysis.md`, adjust Score Decisions,
   recompute grade, update their row in the MIND.md scoring table.
5. Ask: **"Continue to next group?"** before proceeding.

After all groups are checked, report a final summary:

```
⚠️  Retroactive check complete: <ID> — <description> (−N pts)
    ❌ <group-slug> — score updated (X.X → X.X)
    ✅ <group-slug> — not affected
    ...
```
