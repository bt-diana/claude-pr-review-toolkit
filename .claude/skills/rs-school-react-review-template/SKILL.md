---
name: rs-school-react-review-template
description: >-
  Turn an RS School React task description into a review template that the
  rs-school-react-pr-review skill can score against. Use when the user shares a task
  description — a link to a rolling-scopes-school/tasks page or the task text pasted
  into the chat — and asks for a template, a rubric, or a checklist for it, including
  phrasings like "make a template for this task", "we need a rubric for the new task",
  "add this task to the templates". Writes .claude/templates/<task-name>.md. Do NOT use
  to review a PR — that is the rs-school-react-pr-review skill.
---

# RS School React — Review Template Generator

You turn a task description into a **review template**: the rubric file that the
`rs-school-react-pr-review` workflow reads as its source of truth for what to check and
how many points each criterion is worth.

You write exactly one file and nothing else. You do not review code, clone repos, or
touch a PR.

## Your input

The user gives you one of these:

- **A task URL** — usually a page under
  `https://github.com/rolling-scopes-school/tasks/blob/master/react/modules/tasks/`.
  Fetch it with `WebFetch`. If the page is a GitHub blob URL, fetch the `raw` variant
  (`raw.githubusercontent.com/...`) so you get the markdown rather than the rendered
  page.
- **Pasted task text** — the description copied straight into the chat. Use it as-is;
  do not go looking for the original page.

The user may also give the **task name** (the template filename) and the **branch name**.
If not, derive them in Step 1.

If a URL fetch fails, say so and ask the user to paste the description. Do not invent
requirements from the task name alone.

## Step 1 — Identify the task

From the description, work out:

- **Task name** — the kebab-case slug used for the filename and for template lookup
  during a review (`hooks-and-routing`, `state-management`, `api-queries`, `forms`,
  `performance`, `nextjs-ssr`). Prefer the branch name the task asks for. This becomes
  `.claude/templates/<task-name>.md`.
- **Task title** — the human-readable name for the `# ` heading.
- **Branch** — the branch the student must create and the branch it comes from. The task
  description states both ("create a branch `X` from `Y`"). Some tasks are a new
  standalone application, with no parent branch.
- **Whether the task requires tests** and the **coverage threshold** it names.

## Step 2 — Keep only what is about code quality

**This is the important step.** A task description mixes two kinds of requirements, and
only one kind belongs in the template:

| Keep — quality requirements | Drop — functional requirements |
|---|---|
| "use Redux Toolkit or Zustand" | "the app shows 20 cards per page" |
| "theme must go through Context API" | "clicking a card opens a details panel" |
| "no class components" | "the search term survives a page reload" |
| "test coverage at least 80%" | "add a Download CSV button" |
| "use React Hook Form for one form and uncontrolled inputs for the other" | "the form has 8 fields" |

The user's reviews score **code quality only** — never feature completeness. A criterion
that a reviewer would answer by running the app instead of reading the code does not
belong in the template.

The useful signal in a task description is the **constraints on how the code is
written**: which library must be used, which pattern is required or forbidden, what must
be typed, what must be tested. Those become criteria. Everything else is dropped
silently — do not add a "Functionality" section, and do not keep a functional
requirement as a non-scoring note.

## Step 3 — Lay out the rubric

Every template has the same fixed sections plus **one or two task-specific sections**
carrying the requirements from Step 2. Points must sum to **exactly 100**.

Fixed sections (these appear in every template, with roughly these weights):

| Section | Points | Criteria |
|---|---|---|
| Repository & Git | 3 | branch name/parent (1), meaningful commit history (1), Conventional Commits (1) |
| TypeScript | 6–8 | no `any` (2), enums/generics/object types/function types used appropriately (4–6) |
| Architecture & Structure | 15 | logical layers (5), no god components (10) |
| Code Quality Principles | 24–32 | DRY, KISS, YAGNI, no magic numbers/strings, clear names, no commented-out code |
| Tooling | 2–4 | ESLint configured and clean (2), Prettier configured and applied (2) |
| Tests | 20–22 | coverage ≥ threshold (10), all tests pass (4), tests are meaningful (6–8) |

Task-specific section — name it after the task's subject (`State Management`,
`API & Query Layer`, `Forms & Validation`, `Performance Optimizations`, `Next.js
Patterns`, `React & Hooks`) and give it **20–40 points**, split into criteria of about 5
points each. This is where the Step 2 requirements land.

Adjust the fixed weights to make the total land on 100:

- A task with no test requirement (a performance or documentation task) drops the Tests
  section and moves its points into the task-specific section.
- A task with an unusually large subject (Next.js, performance) takes points from
  Architecture and gives them to the task-specific section.
- Never let the total drift off 100. Add up every `**(N pts)**` before you write the
  file, and again after.

**Do not add a module-bundler criterion.** It was removed from every template; its points
live in Tests coverage now.

**Do not add a PR Format Check section** and do not add a PR-description penalty. The user
checks the PR description themselves.

## Step 4 — Write the file

Write `.claude/templates/<task-name>.md` in exactly this shape:

```markdown
# Review Template: <Task Title>

**Task description:** <url, or "provided by the user" if pasted>
**Branch:** `<branch>` (from `<parent-branch>`)
**Max score:** 100 points

## Overall feedback

<!-- LEAVE EMPTY — the user fills this in themselves. Just write the heading. -->

## Code Quality

### Repository & Git (3 pts)

- [ ] **(1 pts)** Branch is created from `<parent-branch>` and named `<branch>`

- [ ] **(1 pts)** Commit history is meaningful and reflects the development process

- [ ] **(1 pts)** Commits follow the [Conventional Commits convention](https://www.conventionalcommits.org/en/v1.0.0/)

**Comment:**

### TypeScript (6 pts)

- [ ] **(2 pts)** `any` type is not used anywhere

- [ ] **(4 pts)** Enums, Generics, Object Types, and Function Types used appropriately

**Comment:**

<!-- ...the remaining sections, same shape... -->

## Total: /100

## Additional recommendations
```

Formatting rules — the review-writer subagent depends on them:

- **Criteria are checkboxes with their points**: `- [ ] **(N pts)** <criterion text>`.
- **A blank line before every criterion and before every `**Comment:**` line.**
- **One `**Comment:**` placeholder per section**, after its criteria. Leave it empty —
  no example text, no sample permalinks. The review-writer fills it and moves it under
  the criterion it explains.
- **No `---` separators.** Headings are enough.
- **No penalty table** unless the task description itself states a penalty.
- **B2 English.** Short sentences, common words, no idioms. Write "the component does too
  much", not "god component" (the section criterion itself may keep the standard
  "no 'god' components" wording).
- Each criterion is one checkable statement. If a requirement needs two checks, write two
  criteria.

## Step 5 — Report

Return:

- The path you wrote.
- The section list with points, and the total (which must be 100).
- Any requirement you dropped as functional, in one line, so the user can overrule you if
  they disagree.
- Anything the task description left unclear (for example, no coverage threshold stated —
  say which default you used).

Do not print the whole template back.
