---
description: Discover and document what every player input does in a retro game you don't know, using the romdev MCP. Reach gameplay from a save state (recording one with a human's help if none exists), then exercise each input, find the code it triggers, and annotate the disassembly. Trigger on /rom-debug:drive-input, or when the user asks to map controls to code, document what buttons do, reverse-engineer player input handling, or "drive the game and see what each action calls" for a ROM project. Spans multiple sessions — progress lives in checklists and source annotations.
allowed-tools: Read Write Edit Task AskUserQuestion Bash(echo *) Bash(basename *) Bash(git rev-parse --show-toplevel) Bash(mkdir *) Bash(ls *) Bash(test *) mcp__romdev__loadMedia mcp__romdev__state mcp__romdev__frame mcp__romdev__input mcp__romdev__memory mcp__romdev__breakpoint mcp__romdev__watch mcp__romdev__cpu mcp__romdev__disasm mcp__romdev__playtest mcp__romdev__catalog mcp__romdev__platform
---

# drive-input

Map a game's **player inputs → the code they run**, for a ROM you start out
knowing nothing about. The loop is always the same: reach a known gameplay
moment, perform one action, find which RAM byte and which instruction that
action drove, and write a label/comment into the disassembly. The job spans
many sessions, so all progress is recorded **on disk** — two checklists plus
the annotations themselves — and any later session resumes from them.

This skill assumes a romdev disassembly project already exists (the source
files you'll annotate). If there's no disassembly yet, use
`disasm({target:'project'})` first; that's out of scope here.

## Step 0 — Set up the data folder

All state and progress live under one folder, so sessions can find them:

```bash
PROJECT="$(basename "$(git rev-parse --show-toplevel 2>/dev/null || pwd)")"
# ROM name = the loaded ROM's filename without extension, e.g. "rygar"
DATA="${CLAUDE_PLUGIN_DATA}/${PROJECT}-${ROM}"
mkdir -p "$DATA/states" "$DATA/tasks"
```

Use `<project name>-<rom name>` (e.g. `rygar-nes-guts-rygar`) so two ROMs in
one project don't collide. Everything below reads/writes under `$DATA`.

## Step 1 — Reach gameplay from a save state

Load the ROM, then try to restore a representative mid-gameplay moment:

```python
loadMedia(platform=..., path="<the ROM>")
state(op="load", path="$DATA/states/start_gameplay.state")   # the goal
```

**If `start_gameplay.state` doesn't exist**, a human has to make it — you can't
reliably navigate menus/intros for an unfamiliar game. Open a window and ask:

```python
playtest(scale=3, title="Record start_gameplay — play to normal gameplay, then hold still")
```

Use `AskUserQuestion` (or just ask) for them to play past the title/intro into
ordinary gameplay — a flat, safe, enemy-free spot if possible — and tell you
when they're parked. Then snapshot it for every future session:

```python
pause()
state(op="save", path="$DATA/states/start_gameplay.state")
```

Close the window (`playtestStop`) before you do your own stepping — its
real-time loop overshoots `loadState` and makes stepping non-deterministic.

> The state file is the single most valuable artifact here: it turns every
> future session's "get to gameplay" from minutes of navigation into one
> `state(op:'load')`. Record it once, reuse it forever.

## Step 2 — Get (or create) the simple-actions checklist

`$DATA/tasks/simple_actions.md` is the checklist of **single, simple button
presses** — the inputs you can perform yourself with `input(op:'set')`. If it
doesn't exist, create it with one unchecked item per control the platform has:

```markdown
# Simple actions — <project>-<rom>

Each item: hold the input from `start_gameplay`, observe, document the code,
then check it off. Record the handler address in the source annotation.

- [ ] D-pad Up
- [ ] D-pad Down
- [ ] D-pad Left
- [ ] D-pad Right
- [ ] A
- [ ] B
- [ ] Start
- [ ] Select
- [ ] Up+Left / Up+Right (diagonals)
- [ ] Down+Left / Down+Right (diagonals)
- [ ] A+B (simultaneous)
- [ ] Direction + A, Direction + B (held together)
```

Trim or extend for the actual platform's controller (call
`platform({op:'doc'})` if unsure what buttons exist).

## Step 3 — Document each simple action

For every unchecked item, run this loop, then **check it off immediately**:

1. **Reset to a known spot:** `state(op:"load", path=".../start_gameplay.state")`.
2. **Hold the input:** `input(op:"set", ...)` for that button. Don't rely on
   one-shot `pressButton`/`pressDuring` — in practice they may not inject; a
   held `input(op:'set')` is what reliably reaches the code.
3. **Let it take effect:** `frame(op:"step", frames=4)` or so — there's input
   latency; sampling on frame 1 often misses the state change.
4. **Confirm it visibly happened:** `frame(op:"screenshot")`. If nothing
   changed on screen, the action may need context (on a rope, near a door…) —
   move it to `complex_actions.md` instead of forcing it.
5. **Find the code.** Two reliable techniques (see Gotchas on why these, not PC
   breakpoints):
   - **Which byte changed:** `memory(op:"search")` before vs. after the action
     to locate the RAM byte the action drives (e.g. a player action-state byte).
   - **Which instruction wrote it:** set a **write-watchpoint**
     `breakpoint(on:"write", ...)` on that byte, repeat the action, and capture
     the writer's PC. A **read-watchpoint** (`breakpoint(on:"read")`) on the
     action-state byte finds the *dispatcher* that branches on it.
   - Disassemble around the captured PC with `disasm` / `cpu` to see the handler.
6. **Annotate the source.** Add a semantic label and a one-line comment at the
   handler — keep any existing address label as an anchor (dual-label), and
   keep the build byte-exact (comments/labels only). Note the action and the
   state-byte value in the comment.
7. **Check the item off** in `simple_actions.md`.

## Step 4 — In parallel, get the complex-actions checklist

While you work through the simple actions, also load
`$DATA/tasks/complex_actions.md` — the actions that **can't** be reached with a
simple press (special moves, item/weapon use, context actions like climbing or
opening a door, mode/menu transitions, taking damage, dying). **If it doesn't
exist, spawn a subagent to interview a human** and write the list:

```
Task(subagent_type="general-purpose",
     description="Interview human for complex actions",
     prompt="Interview the user about <game>: what actions exist that a single
             controller press won't trigger — special/charged moves, item or
             weapon use, context actions (climb, grab, enter door), menu/mode
             changes, taking damage, dying. For each, capture the exact input
             sequence or context needed to perform it. Write the result as a
             markdown checklist to $DATA/tasks/complex_actions.md, one '- [ ]'
             per action with its trigger noted.")
```

## Step 5 — Document each complex action (after simple is complete)

Once every `simple_actions.md` item is checked, work `complex_actions.md`.
These usually need a person driving, because the trigger is precise or
contextual. Use the **playtest live-inspect loop**:

1. `playtest(...)` and ask the human to perform the one action and hold.
2. `pause()` → read the action-state byte / set the watchpoint → `resume()`
   — same find-the-code techniques as Step 3.5.
3. Annotate the handler, **check the item off**, move to the next.

To avoid re-asking the human to replay, `state(op:'save')` at a useful setup
spot (e.g. on a rope, in a menu) under `$DATA/states/` and reuse it.

## Resuming across sessions

There is no in-memory progress — the disk is the source of truth:

1. Re-derive `$DATA` (Step 0).
2. Read both checklists; the first `- [ ]` is where you continue.
3. `simple_actions.md` first; only move to `complex_actions.md` once it's all
   `- [x]`.
4. Trust the source annotations already present — a handler that's already
   labeled is done; don't redo it.

## Gotchas (romdev)

- **PC-execution breakpoints are unreliable on bank-switched code.** On
  mappers where `$8000–$BFFF` (or equivalent) is a swapped window, a
  `breakpoint(on:'pc')` on a banked address often never fires even as the code
  runs. Prefer **read/write watchpoints** (core-level, they do fire) and direct
  `memory(op:'read')` sampling to find handlers.
- **Hold input with `input(op:'set')`, not one-shot presses.** Then step frames;
  the watchpoint/read inherits the held state. One-shot helpers may not inject.
- **Account for input latency** — hold for several frames before sampling the
  state byte, or you'll read the pre-action value.
- **`state(op:'load')` needs the media loaded first and clears active cheats.**
  Apply any cheats *after* loading the state (or re-apply), and `loadMedia`
  before `state(op:'load')`.
- **Close the playtest window before deterministic stepping.** Its real-time
  loop overshoots `loadState` and fights your `frame(op:'step')`. Drive it for
  human-in-the-loop capture; close it for your own headless probing.
- Call `catalog({op:'categories'})` to see the tool set and
  `platform({op:'doc'})` for the platform's memory map and footguns before you
  guess at addresses.

## Report

When you pause or finish, summarize: which checklist items got documented this
session (with handler addresses), what's still unchecked, whether
`start_gameplay.state` and `complex_actions.md` now exist, and the next
unchecked item a future session should pick up.
