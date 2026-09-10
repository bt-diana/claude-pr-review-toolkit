---
name: rs-school-react-post-pending-review
description: >-
  Post a single fresh PENDING review on an RS School React PR from inline-comment
  JSON files. Helper skill invoked by the rs-school-react-pr-review skill after its
  check subagents finish: it merges every agent's comment JSON file into one payload
  and creates one PENDING review in a single request. Do NOT use on its own for
  general PRs — it is the posting step of the RS School React review workflow.
---

# RS School React — Post Pending Review

You take the inline-comment JSON files that the check subagents wrote and post them as
**one fresh PENDING review** on the PR in a **single request**. Nothing is published to
the mentee — a pending review is a draft that shows up in the user's VS Code GitHub Pull
Request extension under "Review in progress" for them to read, edit, and submit.

You do not review code, score anything, or read the mentee's source. You merge JSON and
make one API call.

## Your input

The caller gives you these values:

```
GitHub repo URL: https://github.com/<owner>/<repo>
PR number: <number>
PR head sha: <sha>
Comments dir: .claude/reviews/<student>-<task>.comments
Mentee repo path: <absolute path to the local repo>
Base branch: <branch the PR targets, e.g. origin/hooks-and-routing>
```

Parse `<owner>` and `<repo>` from the repo URL. You need **Mentee repo path** and
**Base branch** to convert whole-file comments to a line (see below). Use `git -C <path>`
with the absolute path — never `cd` into the repo.

## What the JSON files look like

Each check subagent wrote one file in **Comments dir**, e.g. `typescript.json`,
`lint-format.json`, `tests.json`, `code-quality.json`, `security.json`.
Every file has this shape:

```json
{
  "agent": "code-quality",
  "comments": [
    { "path": "src/pages/MainPage.tsx", "start_line": 26, "line": 58, "start_side": "RIGHT", "side": "RIGHT", "body": "..." },
    { "path": "src/constants.tsx", "line": 2, "side": "RIGHT", "body": "..." },
    { "path": "src/constants.tsx", "subject_type": "file", "body": "..." }
  ]
}
```

A comment comes in three shapes:

- **Single line** — `path`, `line`, `side`.
- **Multi-line range** — `path`, `start_line`, `line`, `start_side`, `side`.
- **Whole file** — `path`, `subject_type: "file"`, and `body` only, no line fields.

Keep the single-line and multi-line comments as-is. A `comments` array may be empty. Some
agents (commits, general) write no file at all — that is fine.

### Whole-file comments must be converted to a line — GitHub limitation

GitHub's pending/draft-review API does **not** support file-level comments. The draft
comment type (`DraftPullRequestReviewComment`) has no `subjectType` field and requires a
non-null line position — posting a `subject_type: "file"` comment returns **HTTP 422**.

So before posting, **convert every `subject_type: "file"` comment to a single-line
comment** anchored to that file's **first changed line** on the new side of the diff
(usually line 1 for a newly added file). Drop the `subject_type` field; set `line` to the
first changed line and `side: "RIGHT"`. The body is unchanged — it still describes the
whole-file issue.

If a path produces no first changed line (the file is not in the diff), it cannot be
posted inline — send that comment to `unposted.json` (see "Validate against the three-dot
diff" below).

### Validate against the three-dot diff — or GitHub returns 422

GitHub anchors PR comments against the **three-dot** diff (from the merge-base, with
context), **not** a two-dot `base..HEAD` diff. A comment whose line is not part of that
diff is rejected with `HTTP 422 "Line could not be resolved"`, and GitHub fails the
**whole** review — one bad line drops every comment. Subagents sometimes anchor a real
finding to a line the PR did not actually change, so you must validate before posting.

Build the set of addressable right-side lines from the merge-base diff and check every
comment against it. A comment is **postable** only when each of its `line` / `start_line`
values is in that set. Any comment that is not postable (line outside the diff, or a
file-level comment on a file not in the diff) goes into **`unposted.json`** so the user can
post it by hand — it is never silently dropped.

## Always fresh — no find, no delete

There is **always no review posted** when you run, so do **not** look for an existing
pending review and do **not** delete anything. Just create a new pending review with all
the comments. Skip every find/append/clear/GraphQL step — they are not needed.

## Steps

### Step 1 — Merge every JSON file's comments into one array

Read each `*.json` file in **Comments dir** and concatenate all the `comments` arrays
into one. Build a single review payload:

```json
{
  "commit_id": "<PR head sha>",
  "comments": [ ... every comment from every file ... ]
}
```

Leave out `body` and `event`: no review-level body (the user wants the PR review body
empty), and no `event` so GitHub keeps the review **PENDING**.

PowerShell snippet (the shell here is PowerShell). It (1) builds the addressable
right-side line set from the **merge-base** diff, (2) converts whole-file comments to a
first-changed-line comment, (3) splits comments into **postable** and **unposted**, (4)
writes the payload for the postable ones and `unposted.json` for the rest:

```powershell
$dir  = ".claude/reviews/<student>-<task>.comments"
$sha  = "<PR head sha>"
$repo = "<mentee repo path>"
$base = "<base branch>"   # e.g. origin/hooks-and-routing

$mergeBase = (git -C $repo merge-base $base HEAD).Trim()

# Addressable right-side lines per file, from the THREE-DOT (merge-base) diff with context
$diff = git -C $repo diff "$mergeBase..HEAD" --unified=3
$valid = @{}; $curFile = $null; $curLine = 0
foreach ($l in $diff) {
  if ($l -match '^\+\+\+ b/(.+)$') { $curFile = $Matches[1]; if (-not $valid.ContainsKey($curFile)) { $valid[$curFile] = New-Object System.Collections.Generic.HashSet[int] }; continue }
  if ($l -match '^@@ .*\+(\d+)') { $curLine = [int]$Matches[1]; continue }
  if ($curFile -eq $null) { continue }
  if ($l -match '^-') { continue }                       # left-only line, no right number
  if ($l -match '^[ +]') { [void]$valid[$curFile].Add($curLine); $curLine++ }   # context or added
}

function Test-Postable($c) {
  $lines = @(); if ($c.start_line) { $lines += [int]$c.start_line }; $lines += [int]$c.line
  foreach ($ln in $lines) { if (-not ($valid.ContainsKey($c.path) -and $valid[$c.path].Contains($ln))) { return $false } }
  return $true
}

# Read every agent file EXCEPT unposted.json (do not re-ingest a previous run's reject file)
$raw = Get-ChildItem $dir -Filter *.json | Where-Object { $_.Name -ne 'unposted.json' } | ForEach-Object {
  $agent = $_.BaseName
  (Get-Content $_.FullName -Raw | ConvertFrom-Json).comments | ForEach-Object { if ($_) { $_ | Add-Member -NotePropertyName _agent -NotePropertyValue $agent -PassThru -Force } }
} | Where-Object { $_ }

$firstLineCache = @{}
$postable = New-Object System.Collections.ArrayList
$unposted = New-Object System.Collections.ArrayList
foreach ($c in $raw) {
  if ($c.subject_type -eq 'file') {
    if (-not $firstLineCache.ContainsKey($c.path)) {
      $hunk = git -C $repo diff "$mergeBase..HEAD" --unified=0 -- $c.path | Select-String -Pattern '^@@ .*\+(\d+)' | Select-Object -First 1
      $firstLineCache[$c.path] = if ($hunk) { [int]$hunk.Matches[0].Groups[1].Value } else { $null }
    }
    $ln = $firstLineCache[$c.path]
    if ($null -eq $ln) { [void]$unposted.Add(([pscustomobject]@{ path=$c.path; subject_type='file'; body=$c.body; _agent=$c._agent; reason='file not in PR diff' })); continue }
    $cmt = [pscustomobject]@{ path=$c.path; line=$ln; side='RIGHT'; body=$c.body; _agent=$c._agent }
  } else { $cmt = $c }

  if (Test-Postable $cmt) { [void]$postable.Add($cmt) }
  else { $u = $cmt | Select-Object *; $u | Add-Member -NotePropertyName reason -NotePropertyValue 'line not in three-dot PR diff' -Force; [void]$unposted.Add($u) }
}

# Write unposted.json (keep _agent + reason so the user knows source and why) — only if any
if ($unposted.Count -gt 0) {
  $u = ($unposted | ForEach-Object { $_ | ConvertTo-Json -Depth 10 -Compress }) -join ",`n"
  Set-Content -Path (Join-Path $dir 'unposted.json') -Value "{`n  ""comments"": [`n$u`n  ]`n}" -Encoding utf8
}

# Write the payload for postable comments (strip the helper _agent field)
$body = ($postable | ForEach-Object { $_ | Select-Object * -ExcludeProperty _agent | ConvertTo-Json -Depth 10 -Compress }) -join ",`n"
$out = Join-Path $env:TEMP "rs-pending-review.json"
Set-Content -Path $out -Value "{`n  ""commit_id"": ""$sha"",`n  ""comments"": [`n$body`n  ]`n}" -Encoding utf8
Write-Host "Postable:" $postable.Count " Unposted:" $unposted.Count
$out
```

If `$postable` is empty (no agent finding could be anchored), do **not** post — report
"0 postable inline comments, nothing to post" and the `unposted.json` count, then stop.

### Step 2 — Post one fresh PENDING review

One request, no `event`:

```bash
gh api --method POST repos/<owner>/<repo>/pulls/<number>/reviews --input <temp-payload.json>
```

No `event` field → the review stays `PENDING`. Never pass `"event": "COMMENT"`,
`"APPROVE"`, or `"REQUEST_CHANGES"` — that would publish it to the mentee.

### If posting still fails

The three-dot validation in Step 1 should prevent the usual `422 "Line could not be
resolved"`, but if GitHub still rejects the whole review because one comment has a bad
line:

1. First run `gh auth status`. If `gh` is not authenticated or the repo is unreachable, do
   not loop — report the failure and the count of comments you tried to post.
2. If it is a line-resolution `422`, GitHub's response names (or implies) the offending
   comment. Move that comment from the payload into `unposted.json` (append it, with a
   `reason`), then retry the POST **once**. Note which comment you moved.

Never publish the review — no `event` field, ever.

## Output

Return a short report to the caller:

- Whether the pending review was posted (yes/no).
- How many inline comments it contains.
- The `unposted.json` path and how many comments it holds (so the user can post those by
  hand), or "none" if every comment was posted.
- Any error message.

Do not print the full payload back.
