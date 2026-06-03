---
name: project-rs-school-q2-2026
description: RS School React Q2 2026 course context — mentee Fayzullo05, hooks-and-routing task
metadata:
  type: project
---

RS School React course, Q2 2026 cohort. Diana mentors multiple mentees including Fayzullo05 (Fayzullaxon Sharipxanov) and solarsungai.

**Reviewed tasks:**
- Fayzullo05: hooks-and-routing
- solarsungai: hooks-and-routing (scored 72/100, review at `.claude/reviews/solarsungai-hooks-and-routing.md`)

Task spec: https://github.com/rolling-scopes-school/tasks/blob/master/react/modules/tasks/functional-routing.md

**Common stack across mentees:** React 19, TypeScript 6, Vite 8, React Router 7, Vitest 4 + Testing Library 16, Husky 9, ESLint 9 flat config, Prettier.

**Recurring issues seen across mentees (hooks-and-routing task):**
- Missing `"strict": true` in tsconfig — seen in solarsungai
- Husky pre-commit hook deleted rather than properly configured — seen in solarsungai
- Coverage thresholds set inconsistently in vite.config (statements: 80 but branches/functions/lines: 50) — seen in solarsungai
- Route paths as bare string literals instead of enum/const — seen in solarsungai
- `test_output.txt` committed to repo — seen in solarsungai
- Missing PR description (all required sections absent) — seen in solarsungai
- React app in `rs-react-app/` subfolder instead of git root (single-package antipattern) — seen in solarsungai

**Why:** Review scope is code quality only per Diana's explicit instruction — no functional-requirement scoring.
**How to apply:** Do not evaluate feature completeness; focus on TypeScript quality, React patterns, architecture, tooling, test quality, accessibility, correctness bugs.
