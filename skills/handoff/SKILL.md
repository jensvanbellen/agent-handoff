---
name: handoff
description: >-
  Transfer work-in-progress between coding agents (Claude Code, Codex, or any
  other tool). Creates or updates a compact handoff file in temp/handoffs/ at
  the repo root, or resumes work from an existing one. Use when the user says
  "handoff", is hitting usage limits and needs to switch tools, wants to park
  work for another agent, or starts a session asking to continue work another
  agent left behind.
---

# Agent Handoff Protocol

Purpose: transfer work-in-progress between coding agents with enough context to
continue seamlessly, without bloating the next agent's context window.

Handoff files live in `temp/handoffs/` at the repository root and are git-ignored.

## Mode selection

An argument may have been passed with the command. Decide the mode in this order:

1. Argument is `resume` or `continue` → RESUME mode.
2. Argument is `done` → DONE mode.
3. Argument is `create`, `update`, or any other non-empty string (treat other
   strings as the topic name) → WRITE mode.
4. No argument — auto-detect:
   - This conversation contains substantive work (edits made, files investigated,
     decisions taken) → WRITE mode.
   - This conversation is fresh (no prior work) → RESUME mode if any file exists in
     `temp/handoffs/`; otherwise tell the user there is nothing to hand off or
     resume, and stop.

## WRITE mode (create or update)

### 1. Locate the repo root
`git rev-parse --show-toplevel`. If not inside a git repository, use the current
working directory and skip the git-specific steps below.

### 2. Prepare the directory
Create `temp/handoffs/` under the repo root if it does not exist. `temp/` may
already exist and contain unrelated files — leave those untouched.

### 3. Ensure it is git-ignored
Run `git check-ignore -q temp/handoffs`. If it is not ignored, append this to the
repo-root `.gitignore` (create the file if missing):

```
# agent handoff files (agent-handoff skill)
/temp/handoffs/
```

Never ignore `/temp/` wholesale — in some repos it contains tracked files.

### 4. Determine the filename
One file per work-stream, with a stable name, so repeated invocations update the
same file instead of creating a new one:

- On a feature branch: slugify the branch name — lowercase, replace `/` and every
  run of non-alphanumeric characters with a single `-`.
  Example: `feature/JIRA-123-login` → `temp/handoffs/feature-jira-123-login.md`.
- On `main`/`master`/`develop`, detached HEAD, or outside git: slugify a short
  topic instead (from the argument if one was given, otherwise infer it from the
  work at hand).

If the file already exists, this is an update: read it first, keep entries under
Decisions, Dead ends, and Gotchas that are still true, and refresh everything
else. Never create a second file for the same work-stream.

### 5. Gather context
From the conversation: goal, progress, decisions, dead ends, next steps.
From the environment (skip whatever is unavailable):

- `git branch --show-current`, `git status --short`, `git log --oneline -5`
- `git diff --stat` for uncommitted work
- `gh pr view --json number,title,url,state` if the `gh` CLI is available
- `date -u +"%Y-%m-%d %H:%M UTC"` for the timestamp

### 6. Write the file
Use the template below. Hard limits, in service of the next agent's context
window:

- Target ≤ 150 lines; never exceed 250.
- No code blocks longer than 10 lines — reference `path:line` instead; the next
  agent can read the file itself.
- No diffs or command-output dumps — git already has them.
- Write tool-agnostic instructions: the reader may be a different model in a
  different harness. No references to harness-specific features ("use the Task
  tool") — only plain actions ("read file X", "run command Y").

### 7. Confirm
Tell the user: the file path, a one-line summary of what was captured, and that
the next agent picks it up by running `/handoff` in a fresh session in this repo.

## Handoff file template

```markdown
# Handoff: <short topic>

- **Updated:** <UTC timestamp>
- **By:** <harness / model, e.g. "Claude Code / Opus 4.8", "Codex / GPT-5.6 Terra">
- **Branch:** `<branch>` (base: `<base branch>`)
- **PR:** <#number + URL, or "none">
- **Status:** in progress | blocked: <reason> | ready for review

## Goal
<1–3 sentences: what is being built or fixed, and what "done" looks like.>

## State
- Done: <completed items, one line each>
- In progress: <exact stopping point — which file, which function, what is half-finished>
- Not started: <remaining known work>
- Uncommitted: <one-line summary of git status, or "clean">

## Key files
- `path/to/file.py:42` — <why it matters, one line>

## Decisions
- <decision> — <reason> (so the next agent does not relitigate it)

## Dead ends
- <approach tried> — <why it failed> (do not retry)

## Next steps
1. <concrete first action>
2. <ordered, specific>

## Verify
- `<command>` — <expected result>

## Gotchas
- <flaky test, env quirk, where credentials live, slow command, etc.>
```

Omit any section with nothing to say, except Goal, State, and Next steps — those
are mandatory.

## RESUME mode

1. Find the repo root and list `temp/handoffs/*.md`. None → tell the user, stop.
2. Pick the file whose name matches the current branch slug. No match and exactly
   one file → use it. Several candidates → show the list (filename + Updated
   line) and ask the user which one.
3. Read the file. Cross-check it against reality: current branch, `git status`,
   `git log` since the Updated timestamp. If the repo has moved on (new commits,
   different branch), tell the user about the drift before acting on stale steps.
4. Give the user a 3–5 line summary: the goal, where the previous agent stopped,
   and what you will do first. Then continue with the Next steps unless
   redirected.
5. Do not delete the file. As you make significant progress, update it (WRITE
   mode rules) so a later handoff back stays cheap.

## DONE mode

The work-stream is finished. Identify its handoff file (same filename logic as
WRITE mode step 4), confirm with the user, then delete it. Leave the `.gitignore`
entry in place for future handoffs.
