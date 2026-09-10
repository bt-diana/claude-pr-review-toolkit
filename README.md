# RS School React — PR Review Toolkit

A [Claude Code](https://claude.com/claude-code) setup that drafts code-quality reviews for
[RS School](https://rs.school/) React course pull requests.

I mentor on the RS School React course. Every task means reading a student's PR against a
rubric, leaving inline comments on the exact lines that need work, and filling in a scored
review document. This toolkit does the mechanical part of that: it runs seven checks over
the branch in parallel, posts every finding as a **pending** (draft) review on the PR, and
writes the scored review file. Nothing is published to the student — I read the draft, edit
it, and submit it myself.

## How it works

```
                     rs-school-react-pr-review  (skill, runs in the main loop)
                                    │
        ┌───────────┬───────────┬───┴───────┬───────────┬───────────┬───────────┐
   typescript  lint-format    tests     commits   code-quality  security    general
      check       check       check      check       check       check      review
        │           │           │          │           │           │           │
        └───────────┴───────────┴────┬─────┴───────────┴───────────┴───────────┘
                                     │  each writes <agent>.json
                                     ▼
                    .claude/reviews/<student>-<task>.comments/
                                     │
                    ┌────────────────┴────────────────┐
                    ▼                                 ▼
   rs-school-react-post-pending-review        rs-react-review-writer
   merges every JSON → one PENDING            maps findings to the rubric,
   review on the PR, in one request           scores, writes review.md
```

The seven check agents run **concurrently** and never talk to GitHub. Each one writes its
line-anchored findings to its own JSON file and returns everything that does not sit on a
changed line as text. Posting happens once, from one place. Scoring happens once, in one
place.

## What's in here

```
.claude/
├── skills/
│   ├── rs-school-react-pr-review/          orchestrator — the entry point
│   ├── rs-school-react-post-pending-review/ merges comment JSON → one pending review
│   └── rs-school-react-review-template/     task description → review template
├── agents/
│   ├── rs-react-typescript-check.md         any, enums, generics, strict mode
│   ├── rs-react-lint-format-check.md        npm run lint, Prettier, disabled rules
│   ├── rs-react-tests-check.md              suite run, coverage, test quality
│   ├── rs-react-commits-check.md            Conventional Commits + message vs. diff
│   ├── rs-react-code-quality-check.md       architecture, hooks, naming, DRY/KISS/YAGNI
│   ├── rs-react-security-check.md           XSS sinks, unsafe URLs, secret leakage
│   ├── rs-react-general-review.md           correctness, a11y, npm audit
│   └── rs-react-review-writer.md            the only place scoring happens
├── templates/                               one rubric per course task
└── settings.json                            permissions (mentee clones are read-only)
```

## Requirements

- Claude Code
- [`gh`](https://cli.github.com/) authenticated as the account that will own the review
  (`gh auth login`) — the pending review is created through it
- Node.js, to install and run each student project

## Using it

### Review a PR

Open Claude Code in this folder and give it the PR:

```
Review https://github.com/<owner>/<repo>/pull/42
Local clone: D:\Projects\React Q2 2026\<student>
```

A local clone is optional — without one the skill clones the PR branch itself. Everything
else runs unattended: install dependencies, dispatch the seven checks, post one pending
review, write `.claude/reviews/<student>-<task>.md`.

The pending review then shows up under **Review in progress** in the VS Code GitHub Pull
Requests extension, where each comment can be edited or deleted before submitting.

### Generate a rubric for a new task

```
Make a review template for
https://github.com/rolling-scopes-school/tasks/blob/master/react/modules/tasks/forms.md
```

The `rs-school-react-review-template` skill reads the task description, keeps only the
requirements that are about **how the code is written** (which library, which pattern, what
must be typed and tested), drops the functional ones ("shows 20 cards per page"), and
writes `.claude/templates/<task>.md` with the points summing to 100.

## Design notes

A few decisions that took a couple of course tasks to arrive at:

**Pending reviews only, one request.** The workflow never submits, approves, or requests
changes — it only creates a draft. Early versions had each agent post its own comments as
it finished, which meant seven separate review threads on the PR and no chance to edit
before the student saw them. Now every agent writes JSON and a single request creates one
draft review.

**Comment lines are validated against the three-dot diff.** GitHub anchors review comments
against the merge-base diff, not `base..HEAD`. One comment on a line outside that diff
returns `422` and drops **the entire review**, not just the bad comment. So the posting
skill rebuilds the addressable line set from `git merge-base` and moves anything it cannot
anchor into `unposted.json` instead of risking the batch.

**Whole-file findings get converted.** GitHub's draft-review API has no file-level comment
type, so a `subject_type: "file"` finding is re-anchored to the file's first changed line
before posting.

**Scoring lives in exactly one agent.** The check agents report problems; they do not know
the rubric and they do not assign points. Only `rs-react-review-writer` reads the template,
maps findings to criteria, and applies deductions — which is what keeps two students'
scores comparable.

**Issues carried over from a previous task are not scored twice.** Each task builds on the
previous branch, so last task's flagged problem is still in this task's diff. The
review-writer fetches the reviews already left on the student's earlier PRs and skips the
deduction for anything already flagged there. The comment is still posted — only the points
are not taken again.

**Templates are the source of truth, not the task page.** A criterion that is not in the
template is not evaluated. This keeps reviews consistent across students, and keeps scope
on code quality rather than feature completeness.

## Notes

- `.claude/reviews/` is gitignored. It holds real student reviews, so it stays local.
- The review workflow expects `.claude/reviews/pr-links.md` — a small registry mapping each
  student to their PR per task. It is created on first use and is also gitignored.
- Paths in the skills are relative to the project root, so the toolkit works from any
  folder. Student repos are expected as sibling clones next to it.
- `settings.json` denies every write-side `git -C` command, so the agents can read a student
  clone but never commit to it.
