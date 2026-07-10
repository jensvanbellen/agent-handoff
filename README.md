# agent-handoff

One `/handoff` skill for transferring work-in-progress between coding agents —
Claude Code ↔ Codex, or any tool that can read a markdown file.

Hit your usage limit mid-task in one tool? Run `/handoff`. It writes a compact
context file to `temp/handoffs/` in the repo you are working in. Open the other
tool, run `/handoff` in a fresh session, and it picks up where you left off.

## Install

```bash
git clone https://github.com/jensvanbellen/agent-handoff.git && cd agent-handoff && ./install.sh
```

This symlinks the skill into both tools:

| Tool | Location | Invoke |
|------|----------|--------|
| Claude Code | `~/.claude/skills/handoff` | `/handoff` |
| Codex | `~/.codex/skills/handoff` + `~/.codex/prompts/handoff.md` | `/handoff` |

Both tools read the exact same `skills/handoff/SKILL.md` — one source of truth,
no drift. Because they are symlinks, edits to this repo take effect immediately.
`./install.sh --uninstall` removes everything.

## Usage

| Command | What happens |
|---------|--------------|
| `/handoff` | Auto-detects: mid-session with work done → writes/updates the handoff file; fresh session → resumes from an existing one |
| `/handoff resume` | Force resume mode |
| `/handoff create` / `/handoff update` | Force write mode |
| `/handoff <topic>` | Write mode with an explicit topic (useful when working on `main`) |
| `/handoff done` | Work finished — deletes the handoff file |

Typical flow:

```
# In Claude Code, limits approaching:
/handoff
→ writes temp/handoffs/feature-jira-123-login.md

# In Codex, fresh session, same repo:
/handoff
→ reads the file, summarizes, continues the work
```

## What goes in a handoff file

Metadata (branch, PR, timestamp, which agent wrote it), the goal and definition
of done, precise state (done / in progress with exact stopping point / not
started / uncommitted changes), key files with one-line reasons, decisions made
and why, dead ends already tried, ordered next steps, verification commands, and
gotchas. See the template inside [`skills/handoff/SKILL.md`](skills/handoff/SKILL.md).

Deliberately capped (≤150 lines target, 250 hard max, no code dumps, no diffs) so
resuming costs the next agent a few thousand tokens, not half its context window.

## Design decisions

- **One skill, not two.** Create and resume are two modes of the same protocol;
  the agent can tell them apart from its own conversation state (work done →
  write; fresh session → resume). Explicit arguments (`create`, `resume`) exist
  as an override when auto-detection would guess wrong.
- **Stable filename per work-stream** — the slugified branch name (or a topic
  slug on `main`). Re-invoking `/handoff` in the same or a later session updates
  the same file instead of leaving stale siblings. The timestamp lives inside
  the file, not in the filename.
- **`temp/handoffs/` subdirectory, not `temp/` itself.** Some repos already have
  a `temp/` dir, possibly with tracked files. The skill only creates and
  gitignores `/temp/handoffs/`, never touches anything else in `temp/`.
- **Tool-agnostic file content.** The file may be written by Opus 4.8 and read
  by GPT-5.6 Terra or vice versa, so the skill forbids harness-specific
  instructions in the handoff file.
- **Symlink install.** Matches editing workflow: change SKILL.md here, both
  tools see it instantly. The Codex prompt is a thin pointer to the skill file
  so `/handoff` is guaranteed as a slash command there too.

## Repo layout

```
skills/handoff/SKILL.md    # the skill — single source of truth
codex/prompts/handoff.md   # thin Codex slash-command pointer
install.sh                 # symlinks into ~/.claude and ~/.codex
```
