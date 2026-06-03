---
name: project_oneilcode
description: Mentee oneilcode (Viktoria O'Neil) — RS School React Q2 2026 review history and patterns
metadata:
  type: project
---

Mentee: **oneilcode** (Viktoria O'Neil, email: oneilcode111@gmail.com)
GitHub: https://github.com/oneilcode/class-components

## Reviewed tasks

### hooks-and-routing (PR #3) — reviewed 2026-05-30
- Score: **51/100** (before any late penalty)
- Penalty applied: −20 pts for no custom hook for localStorage
- Key issues: no `useLocalStorage` hook (hard penalty), no enums for route paths / localStorage keys, no shared item type, coverage/ committed to git, Prettier not applied to source files, `setup.test.tsx` trivial test, `value`/`handleSearchItem` naming, fetch logic inline in Search component
- Coverage: 95.65% statements / 95.45% branches — well above threshold
- All 26 tests passed

## Recurring patterns for this mentee
- Tends to commit generated files (coverage/) to the repo
- localStorage logic stays inline rather than being extracted to a hook
- Magic strings / no constants module
- Prettier config exists but files not formatted before commit (pre-commit hook may not be running consistently)

**Why:** useful context for future task reviews of this student.
**How to apply:** Check these areas first when reviewing future PRs from oneilcode.
