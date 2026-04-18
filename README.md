# heigvd-lab-grader

A set of Claude Code skills for grading lab assignments. Designed for courses where students submit work via Git repositories — GitHub Classroom, GitLab, or any other platform. Language and framework agnostic.

---

## What it does

The grader works in three phases:

**Phase 1 — Init** (`grader/init`): Set up a grading session for one lab. Audits the workspace, clones student submissions, locates the lab specification, and produces a central `MIND.md` file as the session's single source of truth. Criteria and grading procedure are delegated to dedicated skills.

**Phase 2 — Grade** (`grader/grade`): Grade one group at a time. Runs automated tests, analyses the student's code diff, optionally generates hidden edge-case tests, writes a `grading-analysis.md` per group, runs the project live for visual verification, and keeps the scoring table in `MIND.md` up to date. Handles late penalties, retroactive score corrections, and consistency across all groups.

**Phase 3 — Feedback** (`grader/feedback`): *(planned)* Publish each group's `grading-analysis.md` to students.

---

## Skill reference

| Skill | When to run | What it does |
|-------|-------------|--------------|
| `grader/init` | Once per lab | Workspace setup, submission cloning, folder structure, MIND.md skeleton |
| `grader/criteria` | After init, or to update criteria | Generates or updates the grading criteria table from submissions, solution diff, lab spec, and course material |
| `grader/procedure` | After criteria are confirmed | Generates a per-criterion grading procedure (what to look for, full marks, common deductions) |
| `grader/grade` | Once per group | Runs tests, analyses code, generates hidden tests if enabled, visual run, finalises scores |

---

## Key design principles

- **MIND.md is the single source of truth.** All criteria, deadlines, penalties, scoring, grading procedure, and session notes live there. Skills read it on every invocation — never from memory.
- **Skills are read-only.** A skill file is never modified during a session. Professor instructions and discoveries are written to MIND.md.
- **Student-facing analyses never reference the solution.** `grading-analysis.md` is published to students — it shows only their own code with findings.
- **Consistent penalties across all groups.** Active penalties are defined in the Grading Procedure with short readable IDs (e.g. `IMPL-01`). Retired or replaced penalties move to a Penalty Archive in MIND.md for traceability. When a new penalty pattern is discovered, all previously graded groups are retroactively checked.
- **Workspace isolation.** Skills never read or write outside the current working directory.

---

## Criteria opt-ins

When generating criteria, the following standard opt-ins are available:

| Opt-in | What it grades | Cost |
|--------|---------------|------|
| **[A] Automated tests — visible** | Starter-kit tests students know about. 1 pt/test (adjustable). | Fast — seconds per group |
| **[B] Automated tests — hidden** | Edge-case tests generated per group from their implementation, mapped to existing criteria as deductions. Tooling detected and proposed per group. | 1–3 min/group, significant tokens |
| **Git & workflow practices** | Commit history quality, work spread over time, no last-minute dumps. For group labs: adds contribution balance, PR usage, DevOps practices. Depth brainstormed with professor. | Manual check during grading |

---

## Workspace layout

```
<workspace>/
├── _global/
│   ├── course-material/
│   └── lab-solutions/
│       └── <lab-slug>/
└── labs/
    └── <lab-slug>/
        ├── MIND.md
        ├── lab-spec.md
        ├── criteria-work.md         ← scratch pad used by grader/criteria
        └── submissions/
            └── <group-slug>/
                ├── grading-analysis.md
                ├── test-evidence.log
                ├── hidden-test-evidence.log
                ├── project-evidence.log
                └── hidden-tests/
```

---

## Installation

Clone the repo and copy (or symlink) the `.claude/` folder into your workspace:

```bash
git clone https://github.com/Ovich/heigvd-lab-grader

# Project-scoped (one course workspace)
cp -r heigvd-lab-grader/.claude/ <workspace>/

# User-scoped (available in all projects)
cp -r heigvd-lab-grader/.claude/skills/grader ~/.claude/skills/
```

Then invoke from Claude Code:
```
grader/init        — start a new grading session
grader/criteria    — generate or update grading criteria
grader/procedure   — generate or update the grading procedure
grader/grade       — grade the next group
```

---

## Submissions sources supported

- GitHub Classroom (via `gh` CLI)
- GitHub / GitLab organisation repos
- Plain list of git URLs
- Local folder of already-cloned repos

---

## Test toolchains supported

Detected automatically from the repo: `npm test`, `make test`, `mvn test`, `mix test`, or any command found in the project's build file.
