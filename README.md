# RS School React — PR Review Toolkit

A [Claude Code](https://claude.com/claude-code) setup that drafts code-quality reviews for
[RS School](https://rs.school/) React course pull requests.

I mentored on the RS School React course, alongside a full-time job. Five students, one
task a week, and every task meant reading a PR against a rubric, leaving inline comments on
the exact lines that needed work, and filling in a scored review document — five times over,
on the same task, every week. This toolkit is what I built to absorb that.

It runs seven checks over the branch in parallel, posts every finding as a **pending**
(draft) review on the PR, and writes the scored review file. Nothing is published to the
student automatically — the mentor reads the draft, edits it, and submits it themselves.

See [`example/`](example/) for a real review it produced, with every artifact from the run.

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

The seven check agents run **concurrently** and never talk to GitHub. Each writes its
line-anchored findings to its own JSON file and returns everything that does not sit on a
changed line as text. (The commits agent is the exception: a commit message is not a line in
the diff, so it returns text only.) Posting happens once, from one place. Scoring happens
once, in one place.

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

example/                                     a real review, with every artifact from the run
```

## How it got built

None of this was designed up front. Every piece exists because the version before it broke
in a specific way.

**1. Templates first, still reviewing by hand.** I started by writing a rubric template per
task, so I was at least checking the same things for every student and scoring them the same
way. It made the reviews more consistent and saved almost no time — reading the code was
always the slow part. But it left me with a written, precise description of what I check,
which is exactly the kind of thing you can hand to an agent.

**2. Feeding the template to an agent — which did not work.** The first version took a
template and a PR and produced reviews that read well and missed the problems I could see in
thirty seconds. It had the checklist but not the judgement. So I reviewed several PRs by
hand and gave the agent my own reviews to analyse — not "here are the rules" but "here is
what I actually commented on, work out what I pay attention to". Those findings are what the
check agents encode today: that a component doing five things costs a lot of points while a
stray magic number costs one, that a name is only worth flagging when it actively misleads.

**3. One agent doing everything was too slow.** Commit messages, TypeScript, code quality,
tests, tooling, task requirements — one agent, in sequence, over a PR touching 60 files. It
took long enough that I stopped wanting to run it. Splitting it into per-area agents that run
in parallel fixed the speed, and had a second benefit I did not expect: a small agent with
one job is much easier to correct. When it reports something wrong, you know exactly which
file to edit.

**4. A subagent cannot spawn subagents.** The plan was a main agent that dispatches the
checks, collects their output, and writes `review.md`. Claude Code does not allow that —
delegation is one level deep, so an agent cannot launch other agents. Turning the coordinator
into a **skill** solved it: a skill runs in the main loop, where the Agent tool works, so it
can dispatch all seven checks at once.

**5. Comments in a markdown file are not where comments belong.** For a while the agents
wrote every finding into `review.md` as a list of `file:line` references and I copied them
onto the PR by hand — the tedious part of reviewing, and still fully manual. So I moved
posting to `gh`. That surfaced the next problem: several agents each posting their own
comments produced several separate review threads on one PR, and they could not reliably
append to a review that already existed. The fix is the shape the pipeline has now — **every
agent writes its comments to a JSON file, and one posting step merges them into a single
request.** That is where `rs-school-react-post-pending-review` came from.

**6. Scoring moved into its own agent.** Writing `review.md` was still the coordinator's job,
tangled up with dispatching. Pulling it out into `rs-react-review-writer` made scoring the
one thing one agent does, and made scores comparable across students because they all come
from the same place.

**7. Then: run it, correct it, run it again.** Every review surfaced something — a false
positive, a nit not worth flagging, a formatting habit of mine it kept getting wrong. Each
one became a rule. Don't score missing return types. Flag props drilling only when more than
one component just forwards. A repo-wide formatting failure is one finding, not one per file.
After enough rounds it became genuinely good, and it carried me through the reviews left at
the end of the course.

**If you want to do this for your own work:** write down what you actually do first, in a
form precise enough to check against. Then automate one piece at a time and let each failure
tell you what the next piece is. The pipeline here looks deliberate. It is really seven
problems, each fixed in the smallest way that worked.

## Requirements

- Claude Code
- [`gh`](https://cli.github.com/) authenticated as the account that will own the review
  (`gh auth login`) — the pending review is created through it
- Node.js, to install and run each student project

## Using it

### Generate a rubric for a new task

```
Make a review template for
https://github.com/rolling-scopes-school/tasks/blob/master/react/modules/tasks/forms.md
```

The `rs-school-react-review-template` skill reads the task description, keeps only the
requirements that are about **how the code is written** (which library, which pattern, what
must be typed and tested), drops the functional ones ("shows 20 cards per page"), and
writes `.claude/templates/<task>.md` with the points summing to 100.

### Review a PR

Open Claude Code in this folder and give it the PR and the matching template:

```
Review https://github.com/<owner>/<repo>/pull/42
Template: .claude/templates/<task-name>.md
Local clone: <path to a clone of the student's repo>
```

The skill tries to infer the template from the PR's branch name or title, but naming it
explicitly is safer — it's required whenever the branch name doesn't match one of the
templates in `.claude/templates/`, including one just generated with the skill below.

A local clone is optional — without one the skill clones the PR branch itself. Everything
else runs unattended: install dependencies, dispatch the seven checks, post one pending
review, write `.claude/reviews/<student>-<task>.md`.

The pending review then shows up under **Review in progress** in the VS Code GitHub Pull
Requests extension, where each comment can be edited or deleted before submitting.

## What a run produces

[`example/`](example/) holds a complete review of one student's PR for the API Querying task
(the student is anonymized): the rubric it was scored against, the raw JSON each agent
wrote, and the finished `review.md` at 93/100. Its README walks through the run — which
agent found what, why four findings were commented on but deliberately not scored, and which
of the thirteen drafted comments I rewrote, dropped, or added by hand before submitting.

## Design notes

**Pending reviews only.** The workflow never submits, approves, or requests changes — it
only ever creates a draft, so nothing reaches the student that I have not read.

**Praise is anchored to a line too.** Agents post short `👍` notes on code that is done
well; the review-writer skips them when scoring. A review made only of complaints is a bad
review, and "good use of RTK Query here" on the actual lines lands better than a warm
sentence at the top.

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
