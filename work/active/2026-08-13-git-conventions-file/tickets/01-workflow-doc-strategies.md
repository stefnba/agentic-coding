---
status: done
depends_on: []
---

# 01 — Workflow doc: strategy definitions and the git.md artifact

## Scope

`docs/agentic-workflow.md` gains, in its artifacts section, a `docs/agents/git.md`
pointer paragraph per ID-6 — at most two sentences: the artifact, its content areas, that
setup scaffolds it, the read-before-git-operations trigger, both mode tokens, the
missing-file `trunk` fallback — plus a layout-tree line. Mode procedures and the
declaration format deliberately do not live here (amended at implement: they belong to
the skills and the template). Keep it generic — no reference to this repo's decision
records; the doc is copied verbatim into consuming repos.

## Done when

- `grep -c "bundle-branch" docs/agentic-workflow.md` ≥ 1 and `grep "trunk"
  docs/agentic-workflow.md` hits the missing-file fallback
- `grep "docs/agents/git.md" docs/agentic-workflow.md` hits the paragraph and the tree
  line
- `grep "Branch strategy: (trunk|bundle-branch)" docs/agentic-workflow.md` returns
  nothing (format owned by the template, AC-1)
- AC-1 passes

## Not in this ticket

Skill edits (→ 02, 03, 04), the template asset (→ 02), this repo's own git.md (→ 05).
