---
name: rs-school-react-pr-review
description: >-
  Review an RS School React course pull request from one of the user's mentees.
  Use when the user shares a GitHub PR link from the RS School React course or asks
  to evaluate a mentee submission — including casual phrasings like "look at this
  PR", "help me grade this", "can you check this submission", "review this for me".
  Performs an end-to-end code-quality review: orchestrates seven check
  subagents in parallel, collects each agent's inline-comment JSON, posts one fresh
  pending review on the PR through the post-pending-review skill, and hands off to the
  review-writer subagent for scoring and review.md. Do NOT use for general
  coding questions or non-RS-School projects.
---

# RS School React PR Review

You review React PRs for the user, a mentor in the Rolling Scopes School (RS School)
React course. They share PR links from their mentees and expect a complete,
GitHub-ready code-quality review they can post with minimal edits.

**Why this is a skill and not a subagent:** the workflow below spawns seven check
subagents in parallel and then a review-writer subagent. A subagent cannot spawn
its own subagents — delegation is only one level deep. So this orchestration must
run in the **main loop** (this skill), where the Agent tool works. Run every Agent
call from here; never wrap this workflow inside another Agent call.

## Your workflow

You **orchestrate seven check subagents in parallel** to gather findings. Each
line-anchorable finding is written by its subagent to a per-agent **comment JSON
file**; findings that do not sit on a changed line come back as a text bullet list.
When the subagents finish, you call the **`rs-school-react-post-pending-review`**
skill once to merge every JSON file and post **one fresh pending review** on the PR.
Then you hand the JSON comment files and the text findings to the
**`rs-react-review-writer`** subagent, which maps issues to the rubric, scores them,
and writes `review.md`. **No subagent posts to the PR any more** — only the
post-pending-review skill does. **Scoring and `review.md` live only in
`rs-react-review-writer`.**

**You do not review code yourself. Ever.** You do not read source files, you do not
evaluate code quality, you do not score anything. Your only job is: set up context →
dispatch subagents → collect results → hand off to review-writer. If a subagent
returns a **transient** infrastructure error (e.g. `API Error: Unable to connect`,
`ConnectionRefused`, `FailedToOpenSocket`), relaunch just that subagent once with the
same prompt before continuing. If it fails a second time, or fails for a non-transient
reason, stop and report which subagent failed. Either way, do not compensate by doing
the check yourself.

**Never pause to ask the user questions mid-run.** Go through all steps without
stopping. Everything — missing template, missing PR number, prior review memory — has
a default: skip what's missing, always post a **fresh** pending review (there is
always no review already posted, so never look for or delete one). Surface any gaps in
the final summary.

### Step 1 — Identify the task

If the user's message names a template path, use it directly and skip inference. Otherwise
infer the task name from the PR branch name or PR title (e.g. `hooks-and-routing`,
`state-management`, `forms`, `api-queries`, `performance`, `nextjs-ssr`). Do **not**
follow links to the task spec.

### Step 2 — Load the matching review template

Read `.claude/templates/<task-name>.md` (or the path the user gave in Step 1). The template
is the source of truth for **what** to check, the **point weights**, and the **penalty
table**.

**Only the template's criteria are evaluated.** Do not consult the RS School task
description page. If the template doesn't list a criterion, it isn't evaluated. If no
template exists for the task and the user did not name one, note it in the final summary and
continue with best-effort scoring.

No templates ship with this toolkit — `.claude/templates/` is empty until the user runs the
`rs-school-react-review-template` skill for a task, or writes one by hand. See
[`example/template.md`](../../../example/template.md) for a real one.

### Step 3 — Set up the mentee's code

Take the repo URL, PR number, branch, and local folder from the user's message. Do not
derive them with `git`/`gh`. If they gave a local repo path (a sibling clone next to this
project, e.g. `<workspace>/<mentee>`), use it directly — do not clone. If they did not,
clone the PR branch to a local working directory. Either way, install dependencies
(`npm install`) using the absolute path.

**Never `cd` into the repo directory.** Always use `git -C <absolute-path>` and
`npm --prefix <absolute-path>` so your working directory stays at the project root
throughout. Changing directory breaks subagent lookup.

Then capture the values you will pass to every subagent:

- **Mentee repo path** — absolute path to the local repo
- **PR head sha** — `git -C <path> rev-parse HEAD` (pins every permalink)
- **GitHub repo URL** — `https://github.com/<owner>/<repo>`
- **PR number** — needed when you post the pending review
- **Base branch** — the branch the PR targets
- **Previous task PR links** — open
  `.claude/reviews/pr-links.md`, find the student's table,
  and collect the PR links of every task that comes **before** the current task in the
  task order listed there. The review-writer fetches the reviews already left on those
  PRs so an issue carried over from a previous task is not scored again. If the current
  PR is missing from the registry, add it to the student's table now (`pr-links.md` is
  the one file outside the comments dir you may edit). If the student has no previous
  reviewed task, pass "none".

Never create files inside the mentee's repo.

**Prepare a fresh comments directory.** The line-anchoring subagents each write their
inline PR comments to a JSON file. Use one directory per review, **outside** the mentee
repo:

```
.claude/reviews/<student>-<task>.comments/
```

Delete it if it already exists, then recreate it empty, so no stale comment from a
previous run leaks into this review. Each subagent writes its own file inside it:

| Subagent | Comments JSON file |
|----------|--------------------|
| `rs-react-typescript-check` | `typescript.json` |
| `rs-react-lint-format-check` | `lint-format.json` |
| `rs-react-tests-check` | `tests.json` |
| `rs-react-code-quality-check` | `code-quality.json` |
| `rs-react-security-check` | `security.json` |
| `rs-react-general-review` | `general.json` |

`rs-react-commits-check` (commit messages, not diff lines) writes **no** JSON file — it
returns text only.

`rs-react-security-check` and `rs-react-general-review` **do** write JSON files
(`security.json`, `general.json`) so their findings post as inline comments, but those
comments are **non-scoring** — the review-writer reads them into Additional recommendations,
not the rubric. Each security comment carries `"agent": "security"` and each general comment
carries `"agent": "general"` so the review-writer can tell. (The general subagent still
returns its non-anchorable findings — `npm audit`, config notes — as text.)

### Step 4 — Dispatch the seven subagents in parallel

Launch **all seven subagents in a single message** with multiple Agent tool calls so
they run concurrently. Each subagent writes its line-anchorable findings to its
**Comments JSON file** and returns a flat bullet list of the findings that do **not**
sit on a changed line (each bullet self-contained: permalink + what is wrong + brief
why/how-to-fix in B2 English). **No subagent posts to the PR.**

| Subagent | Area |
|----------|------|
| `rs-react-typescript-check` | TypeScript: `any`, return types, parameter types, enums for constant groups, generics, readonly props, strict mode, suppression comments |
| `rs-react-lint-format-check` | `npm run lint` output, Prettier check output, disabled rules, inline suppression comments, bundler config, path aliases |
| `rs-react-tests-check` | Test pass/fail, coverage, test quality (per-test stubbing, MemoryRouter assertions, descriptive names) |
| `rs-react-commits-check` | Commit messages: Conventional Commits convention + whether each message reflects the work in its diff. Text only — no JSON, no PR comments. |
| `rs-react-code-quality-check` | Repository & Git, architecture, file extensions, React patterns, hooks, naming, DRY/KISS/YAGNI, magic strings, orphan files, test-code quality, plus a folded-in changed-code bug scan (replaces the old `code-review` skill). Writes its line issues — plus short `👍` positive notes on good patterns — to its **Comments JSON file** and returns the rest as a bullet list. |
| `rs-react-general-review` | Anything the others would miss — correctness bugs, accessibility, performance, `npm audit` (uses the built-in `review` skill); non-scoring. Writes line findings to its **Comments JSON file** (`general.json`) so they post inline, and returns non-anchorable findings (`npm audit`, config notes) as text. |
| `rs-react-security-check` | Security problems in the diff via the built-in `security-review` skill (XSS sinks, unsafe URLs, secret leakage, unsafe deserialization, injection). Writes line findings to its **Comments JSON file** (`security.json`) so they post inline, but they are **non-scoring**. |

Each Agent call's `prompt` must include these input values (give each line-anchoring
subagent — `typescript`, `lint-format`, `tests`, `code-quality`, `security`, and
`general` — its own **Comments JSON path** from the Step 3 table; omit that line only for the
`commits` subagent):

```
Mentee repo path: <absolute path>
Task name: <task-name>
Template path: .claude/templates/<task-name>.md
PR head sha: <sha>
GitHub repo URL: <https://github.com/owner/repo>
PR number: <number>
Comments JSON path: .claude/reviews/<student>-<task>.comments/<agent>.json
Base branch: <base>

Run your checks. Check the whole branch, not only the PR diff: verify the template
requirements for your area are implemented somewhere in the code, and report an issue
even when a previous task's review already flagged it (the review-writer decides what
is scored). Write line-anchored findings to your Comments JSON path and return the
rest as a bullet list.
```

The line-anchoring subagents (`typescript`, `lint-format`, `tests`,
`code-quality`, `security`, `general`) write their inline comments to the given **Comments
JSON path** and return the non-line findings as text. Only `commits` ignores the JSON path
and returns text only. **Nothing is posted to the PR in this step** — posting happens
once, in Step 4b, after every subagent has finished writing its JSON.

### Step 4b — Post one fresh pending review

After **all** subagents return, every line-anchoring subagent has written its
`*.json` file into the comments directory. Invoke the
**`rs-school-react-post-pending-review`** skill once. It reads every JSON file, merges
all the comments, and posts them as **one fresh pending review** on the PR in a single
request. Pass it:

```
GitHub repo URL: https://github.com/<owner>/<repo>
PR number: <number>
PR head sha: <sha>
Comments dir: .claude/reviews/<student>-<task>.comments
Mentee repo path: <absolute path to the local repo>
Base branch: <base>
```

It needs **Mentee repo path** and **Base branch** because GitHub's pending-review API does
not support file-level comments: the skill converts every `subject_type: "file"` comment to
a comment on that file's first changed line before posting.

There is always no review posted, so it never looks for or deletes an existing one.
Before posting, the skill validates every comment against the PR's three-dot diff and
moves any comment GitHub cannot anchor into an **`unposted.json`** file in the comments
directory, so the user can post those by hand. Relay its result (posted yes/no, comment
count, and the `unposted.json` path + count if any) into your final summary. The user opens
the PR in the VS Code GitHub Pull Request extension, where the comments appear under
"Review in progress" for them to edit and submit.

### Step 5 — Collect findings

You now have two kinds of input, both already on disk or in hand:

1. The **comment JSON files** in the comments directory (typescript, lint-format,
   tests, code-quality, security, general) — every comment is a finding. The
   review-writer **scores** the typescript / lint-format / tests / code-quality
   comments (except `👍` positive ones). The `security.json` and `general.json` comments are
   **non-scoring** (Additional recommendations). The `unposted.json` file, if present, is a
   manual-posting aid for the user — its comments are already in the agent files, so the
   review-writer ignores it.
2. The **text bullet lists** the subagents returned for findings that did not map to a
   changed line (structural code-quality issues, all commit findings, the tests run
   summary, bundler config facts, security notes, etc.).

You do not need to fetch anything back from the PR — the JSON files are the source of
truth for the inline comments. Do **not** score or write the review yourself. Hand both
inputs to the review-writer in Step 6.

### Step 6 — Dispatch the review-writer subagent

Launch **`rs-react-review-writer`** with one Agent tool call. It maps issues to the
rubric, scores them, applies the penalty table, and writes `review.md`. Its prompt must
include:

```
Student: <handle>
Task name: <task-name>
Template path: .claude/templates/<task-name>.md
Output path: .claude/reviews/<student>-<task>.md
Comments dir: .claude/reviews/<student>-<task>.comments
Previous task PR links (for the carried-over check; "none" if first task):
<the links collected from pr-links.md in Step 3>

Scoring inline comment JSON files (read every *.json in the comments dir EXCEPT
`security.json`, `general.json`, and `unposted.json`; skip any comment whose body starts with 👍):
<list the JSON file paths, or note that the review-writer should read the comments dir>

Check-agent text findings (a scoring source — count each real issue as a deduction):
<the non-line bullet lists from typescript / lint-format / tests / commits>

Non-scoring findings (Additional recommendations only):
<the security.json and general.json comments, plus the general and security bullet lists>

Write review.md and return the path and the total.
```

The review-writer scores from **two sources**: the inline comments in the scoring JSON
files (skipping any `👍` positive comment) and the check-agent text findings (typescript,
lint-format, tests, commits), each of which counts as a deduction. Before scoring,
it fetches the reviews already left on the **previous task PRs** and skips the deduction
for every issue that an earlier task's review already flagged and that the mentee carried
over unchanged (the comment still posts; only the points are not taken twice). The
`security.json` and `general.json` comments and the `general` / `security` text findings stay
non-scoring and feed the Additional recommendations; `unposted.json` is ignored (already
counted via the agent files). Scoring, the penalty table, and the file contents are entirely
the review-writer's job; relay its returned total back to the user.

## Subagents

The criteria glossary lives inside each subagent. If you need the full list of checks
for an area, read the matching subagent file:

- `.claude/agents/rs-react-typescript-check.md`
- `.claude/agents/rs-react-lint-format-check.md`
- `.claude/agents/rs-react-tests-check.md`
- `.claude/agents/rs-react-commits-check.md`
- `.claude/agents/rs-react-code-quality-check.md`
- `.claude/agents/rs-react-general-review.md`
- `.claude/agents/rs-react-security-check.md`
- `.claude/agents/rs-react-review-writer.md` — maps issues to the rubric, scores, and writes `review.md` (the only place scoring happens)

The pending review is posted by the **`rs-school-react-post-pending-review`** skill
(Step 4b), not by any subagent.

You do not need to repeat their checks — invoke them, take their JSON files and bullets,
hand them to `rs-react-review-writer`.

## Scoring, format, and tone

These all live in `rs-react-review-writer` and the task template — not here. Do not
score or format anything yourself. The review-writer owns:

- the award-based checkbox rubric and per-criterion points,
- the penalty table and the `## Total: X/100` line,
- the B2-English writing tone,
- the "no file links in review.md / terse comments" rules,
- the empty `## Overall feedback` heading (the user fills the body themselves).

**PR Format Check — skip it.** The user checks the PR description format themselves (task
link, screenshot, deploy URL, dates, self-assessment) and applies any related penalty
separately. The review file starts at `## Overall feedback` and contains only the
code-quality rubric.

## Memory

The user's feedback rules and profile live in the user-level memory loaded into this main
loop (`MEMORY.md`). Read and apply them. When you learn a new RS School convention,
stylistic preference, recurring mentee issue, or task rubric detail during a
review, save it there as a `feedback`/`project`/`reference` memory so future reviews get
sharper.

## References

- [RS School Git Convention](https://rs.school/docs/git-convention)
- [Conventional Commits spec](https://www.conventionalcommits.org/en/v1.0.0-beta.2/)
- [RS School PR Review Process](https://rs.school/docs/mentoring/pull-request-review-process)
- [Reliable React Component Attributes](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/)

## Override

The template is authoritative. The subagents own the per-area check lists. If the user
states an exception for the current review (e.g. "ignore ESLint warnings for this
one"), follow their instruction and pass the exception through to the affected subagent's
prompt.
