---
description: Set up an existing project with TODO.md and SCHEDULE.md checklists and wire CLAUDE.md (or its equivalent, e.g. AGENTS.md) to reference both, so future Claude Code sessions — attended or scheduled/autonomous — can find and pick up pending work without being told what to do. Trigger on /autonomy:tasks, or when the user asks to set up a to-do list, task list, or scheduled-task file for a project, or to make a project so "future sessions can automatically work on" pending items. Works whether or not the project already has a CLAUDE.md — creates one if absent, appends to it if present, and follows the symlink if CLAUDE.md points elsewhere (e.g. AGENTS.md).
allowed-tools: Read Write Edit Bash(readlink *) Bash(git rev-parse --show-toplevel) Bash(ls *) Bash(test *)
---

# tasks

Sets up a project with two checkbox-style task files and a pointer to them in
the project's agent-instructions file, so any future session — whether a
human is attending it or it's running on a schedule — can find pending work
and pick the next item without being told explicitly what to do.

## Step 1 — Find the project root

Prefer the git root:

```bash
git rev-parse --show-toplevel
```

If that fails (not a git repo), use the current working directory. Confirm
with the user if it's ambiguous which directory they mean.

## Step 2 — Create `TODO.md` (general backlog)

If `TODO.md` already exists, leave it untouched and tell the user you found
it (don't overwrite someone's existing backlog). Otherwise create it:

```markdown
# TODO

Pending work for this project. Items are checked off as they're completed.

- [ ] (add your first task here)
```

## Step 3 — Create `SCHEDULE.md` (scheduled/autonomous backlog)

Same rule — if it exists, leave it alone and say so. Otherwise create it:

```markdown
# SCHEDULE

Tasks for the scheduled/autonomous routine — kept separate from `TODO.md` so
unattended runs only touch items that are explicitly safe to do without a
human in the loop. Items are checked off as they're completed.

A human-attended agent may also pick tasks from here once `TODO.md` is empty.

- [ ] (add your first scheduled task here)
```

## Step 4 — Locate the agent-instructions file

Claude Code loads `CLAUDE.md`; some projects instead keep an `AGENTS.md` and
symlink `CLAUDE.md -> AGENTS.md` (or vice versa), and some have neither yet.
Handle each case:

1. **`CLAUDE.md` exists and is a symlink** — run `readlink CLAUDE.md` and edit
   the file it resolves to (e.g. `AGENTS.md`), not the symlink itself (writing
   through a project's `CLAUDE.md` symlink will be refused). Resolve relative
   targets against the project root.
2. **`CLAUDE.md` exists as a regular file** — edit it directly.
3. **`CLAUDE.md` doesn't exist, but `AGENTS.md` does** — edit `AGENTS.md`
   (it's the de facto instructions file here; don't create a second one).
4. **Neither exists** — create a new `CLAUDE.md`.

## Step 5 — Add (or update) the "Task lists" section

Read the resolved file (skip this read if you're creating a brand-new one).
Look for a section that already references `TODO.md` and `SCHEDULE.md` — if
one exists, leave it alone and tell the user. Otherwise append:

```markdown

## Task lists

Two checkbox-style task lists (`- [ ] task`, checked off as `- [x]` when done):

- `TODO.md` — general backlog. If the user doesn't say what to work on, pick
  the next unchecked item here.
- `SCHEDULE.md` — tasks for the scheduled/autonomous routine, scoped to
  things safe to complete without a human in the loop. A human-attended
  session may also draw from `SCHEDULE.md`, but only once `TODO.md` has no
  unchecked items left.
```

If you're creating a brand-new `CLAUDE.md` (case 4 above), write a minimal
file consisting of a top-level `# CLAUDE.md` heading followed by this same
section — don't invent unrelated project documentation.

## Step 6 — Report

Summarize what was created vs. left alone vs. appended to, and which file
ended up holding the "Task lists" section (since it may be `AGENTS.md` rather
than `CLAUDE.md`). Suggest the user populate `TODO.md` / `SCHEDULE.md` with
real items, replacing the placeholder line.

**Don't** set up a `/schedule` routine yourself — that's a separate, billed,
user-triggered action. Just mention it's available if they want fully
unattended runs.
