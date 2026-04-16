# heigvd-lab-grader

A set of Claude Code skills for grading lab assignments. Designed for courses where students submit work via Git repositories — GitHub Classroom, GitLab, or any other platform. Language and framework agnostic.

---

## What it does

The grader works in three phases:

**Phase 1 — Init** (`grader/init`): Set up a grading session for one lab. Audits the workspace, clones student submissions, locates the lab specification, defines grading criteria with the professor, generates a solution diff, and produces a grading analysis template. All session state is stored in a central `MIND.md` file.

**Phase 2 — Grade** (`grader/grade`): Grade one group at a time. Runs automated tests, analyses the student's code diff, writes a `grading-analysis.md` per group addressed to the student, runs the project live for visual verification, and keeps the scoring table in `MIND.md` up to date. Handles late penalties, retroactive score corrections, and consistency across all groups.

**Phase 3 — Feedback** (`grader/feedback`): *(planned)* Publish each group's `grading-analysis.md` to students.

---

## Key design principles

- **MIND.md is the single source of truth.** All criteria, deadlines, penalties, scoring, and session notes live there. Skills read it on every invocation — never from memory.
- **Skills are read-only.** A skill file is never modified during a session. Professor instructions and discoveries are written to MIND.md.
- **Student-facing analyses never reference the solution.** `grading-analysis.md` is published to students — it shows only their own code with findings.
- **Consistent penalties across all groups.** A running Penalties table in MIND.md ensures every group is evaluated against the same standard. When a new penalty pattern is discovered, all previously graded groups are retroactively checked.
- **Workspace isolation.** Skills never read or write outside the current working directory.

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
        └── submissions/
            └── <group-slug>/
                ├── grading-analysis.md
                ├── test-evidence.log
                └── project-evidence.log
```

---

## Installation

Copy the `grader/` folder into your Claude Code skills directory:

```bash
# Project-scoped (one course workspace)
cp -r grader/ <workspace>/.claude/skills/

# User-scoped (available in all projects)
cp -r grader/ ~/.claude/skills/
```

Then invoke from Claude Code:
```
grader/init    — start a new grading session
grader/grade   — grade the next group
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
