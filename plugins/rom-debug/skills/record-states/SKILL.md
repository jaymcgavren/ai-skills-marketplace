---
description: Build a reusable library of save states for documenting a retro ROM, using the romdev MCP. A human plays each notable moment in a playtest window and the agent records the state to disk, paired with a per-state analysis task for later sessions. Trigger on /rom-debug:record-states, or when the user asks to record/capture save states for reverse-engineering a game, set up a state library so future sessions can analyze specific moments, or "park the game at the interesting bits so we can document them later". The capture loop is guided by sonnet subagents; the analysis itself is left as a checklist.
allowed-tools: Read Write Edit Task AskUserQuestion Bash(echo *) Bash(basename *) Bash(git rev-parse --show-toplevel) Bash(mkdir *) Bash(ls *) Bash(test *) Bash(cp *) mcp__romdev__loadMedia mcp__romdev__state mcp__romdev__playtest mcp__romdev__frame mcp__romdev__memory mcp__romdev__sprites mcp__romdev__input mcp__romdev__host mcp__romdev__cpu mcp__romdev__catalog mcp__romdev__platform
---

# record-states

Build a **library of save states** that lets future sessions document a ROM
cheaply. A `state(op:'load')` that drops you one input away from a behavior —
or onto a screen full of varied live objects — turns each documentation task
from minutes of re-navigation into a single call. The most valuable state in a
project routinely unlocks *many* analyses (in one real project a single
"one-input-from-the-event" capture unlocked 6+ separate tasks, and doubled as a
reusable multi-object RAM snapshot).

This skill is **capture-only**. It drives a human to record the states and
writes a paired **CAPTURE / ANALYSIS** checklist; it does **not** run the
analysis. The unchecked ANALYSIS items are left for later or scheduled sessions
(any model) to pick up — the same shape as a project's `TODO.md`, so those
sessions recognize them.

The capture loop is long, interactive, and token-heavy — Claude can't navigate
an unfamiliar game, so a human drives while the agent watches and records. That
loop runs in **sonnet** subagent(s); the session that invokes the skill stays a
thin orchestrator.

This skill assumes a romdev disassembly project already exists (what later
ANALYSIS sessions will annotate). If there's no disassembly yet, use
`disasm({target:'project'})` first; that's out of scope here.

(Tool names here use the `mcp__romdev__` prefix, which assumes the server was
registered as `romdev`. If yours is registered under another name, match the
prefix — in `allowed-tools` and when calling — to that name.)

## Step 0 — Set up the data folder

All state and progress live under one folder, so sessions can find them:

```bash
PROJECT="$(basename "$(git rev-parse --show-toplevel 2>/dev/null || pwd)")"
# ROM name = the loaded ROM's filename without extension, e.g. "rygar"
DATA="${CLAUDE_PLUGIN_DATA}/${PROJECT}-${ROM}"
mkdir -p "$DATA/states" "$DATA/tasks"
```

Use `<project name>-<rom name>` (e.g. `rygar-nes-guts-rygar`) so two ROMs in
one project don't collide. Confirm a ROM is loaded with `catalog(op:'status')`;
if not, `loadMedia` it first.

## Step 1 — Ensure `start_gameplay.state`

Every capture starts from a known mid-gameplay moment. If
`$DATA/states/start_gameplay.state` doesn't exist, it's just the first capture
target (the human plays past the title/intro into ordinary gameplay — a flat,
safe, enemy-free spot — and the agent records it via the loop below). Every
later capture loads it as the starting point.

> The state library is the single most valuable artifact here: it turns every
> future session's "get to the interesting moment" from minutes of navigation
> into one `state(op:'load')`. Record once, reuse forever.

## Step 2 — Spawn the sonnet capture-guide subagent

The orchestrator hands the whole interactive loop to a **sonnet** subagent, then
on its return verifies the files, backs them up into the repo, and reports.

```
Task(subagent_type="general-purpose", model="sonnet",
     description="Guide human through recording save states",
     prompt="""
You guide a HUMAN through recording a library of save states for ROM
reverse-engineering, using the romdev MCP. DATA=<$DATA>. REPO=<git root>.
Checklist=<$DATA/tasks/record_states.md>. start_gameplay exists=<yes/no>.

A) PLAN THE SET. Present the recommended-menu categories (below) and use
   AskUserQuestion to interview the human for game-specific moments worth a
   state. Produce an ordered list; each item = {name (kebab-case),
   moment (one line), setup-from-start_gameplay (exact steps), analysis-brief
   (what a later session should document from it)}. If start_gameplay is
   missing, make it the FIRST item.

B) FOR EACH item, run the capture loop (below), then write its paired CAPTURE
   (checked) + ANALYSIS (unchecked) entries to the checklist immediately, in
   the format given. Use today's date.

C) RETURN a summary: states written (paths), the checklist path, and anything
   the human skipped or deferred.

Follow the capture loop, recommended categories, checklist format, and gotchas
exactly as written in this skill. You OWN the single playtest window for the
whole run — never open a second interactive window.
""")
```

`model="sonnet"` is the point of this skill — the heavy interactive work must
land on sonnet. The orchestrator (whatever model invoked the skill) does only
Step 0/1, this spawn, and the post-return verification.

After the subagent returns, the orchestrator:

1. Confirms each promised `$DATA/states/<name>.state` exists (`ls`/`test`).
2. Backs each one up into the **repo's** `states/` dir, the durable copy:
   `mkdir -p "$REPO/states" && cp "$DATA/states/<name>.state" "$REPO/states/"`.
3. Sanity-checks `record_states.md` (every CAPTURE that's `[x]` has a real
   file; every CAPTURE has a paired ANALYSIS).
4. **Reports** (see below).

## The capture loop (run by the sonnet subagent, once per target state)

1. **Open the window:**
   `playtest(op:"open", scale:3, title:"Record <name> — play to <moment>, pause, and let me know")`.
2. **Brief the human:** load the starting point first
   (`state(op:"load", path=".../start_gameplay.state")`, except when recording
   start_gameplay itself), tell them the exact setup, and ask them to **play to
   the moment, pause, and signal** — e.g. answer an `AskUserQuestion` "Ready?"
   prompt. The pause may be the **game's own pause** OR the **emulator's
   pause** — either is fine. The human does **not** touch any save-state hotkey;
   the agent records the state.
3. **Record it yourself:** once the human signals,
   `host(op:"pause")` (idempotent — a no-op if they used the emulator pause,
   and it freezes the core if they only used the in-game pause), then
   `state(op:"save", path:".../states/<name>.state")` writes the blob straight
   to disk. Saving needs no stepping, so the window's real-time loop doesn't
   fight it. `host(op:"resume")` afterward in case they want to keep playing
   toward the next target.
4. **Verify the capture:** `playtest(op:"stop")`, then `state(op:"load")` the
   file you just wrote and inspect it — `frame(op:"screenshot")` plus
   `memory(op:"read")` / `sprites(op:"inspect")` — to confirm it froze the
   intended moment. Note a few concrete live RAM/slot values (player state byte,
   populated object slots, on-screen entity count) to enrich the analysis brief;
   those observations are what make the ANALYSIS item actionable later.
5. **Write the checklist entries** (CAPTURE checked, ANALYSIS unchecked) in the
   format below, then move to the next target.

> Optional: a human who'd rather use the **emulator's own save-state hotkey**
> may — then persist that slot with `state(op:"export", fromSlot:<slot>,
> path:".../states/<name>.state")` (find the slot with `state(op:"list")`).
> But the default is the agent saving it, so the human only has to play + pause.

## Recommended-menu categories

Game-agnostic starting points (the human adds specifics). The best state puts a
later session as close as possible to the code of interest — ideally one known
input or one RAM poke away from the behavior — and freezes many objects in
varied states so one capture serves many analyses.

1. **Base `start_gameplay`** — past the intro, ordinary gameplay, calm spot.
   The reusable foundation every other capture (and session) loads first.
2. **One-input-from-a-complex-event** *(highest ROI)* — parked on the cusp of a
   rare event so a later session replays one input + sets a watchpoint: a
   special/charged move, item or weapon use, a pickup, taking damage, dying.
3. **Rich live-RAM snapshot** — a frame with many entity/object slots populated
   with *varied* values (a busy screen), reusable across many analyses with no
   new capture.
4. **Just-before-a-transition** — right before a level/area/mode change or a
   cutscene boundary, for the advance condition and the transition handler.
5. **Encounter-in-progress** — a boss/miniboss live on screen, for
   `/rom-debug:locate-value` HP hunts and the defeated branch.

## Checklist format (`$DATA/tasks/record_states.md`)

One section per state. A CAPTURE must be `[x]` before its ANALYSIS can start;
the ANALYSIS items are deliberately left unchecked for later/scheduled sessions
(any model) to pick up.

```markdown
# Record states — <project>-<rom>

CAPTURE tasks (human at the playtest window) + paired ANALYSIS tasks (any
model, later/scheduled). A CAPTURE must be [x] before its ANALYSIS can start.

## <moment name>  (category: one-input-from-event)
- [x] CAPTURE `states/<name>.state` (<date>) — in $DATA/states AND backed up at
      states/<name>.state in the repo. Setup from start_gameplay: <exact steps>.
      Live snapshot: <key RAM/slot values confirming the moment>.
- [ ] ANALYSIS (needs <name>.state): <subroutines / RAM addresses to document,
      and the technique — replay+write-watchpoint, /rom-debug:locate-value, etc.>.
```

## Resuming across sessions

There is no in-memory progress — disk is the source of truth:

1. Re-derive `$DATA` (Step 0).
2. Read `record_states.md`. The first `- [ ]` **CAPTURE** is where this skill
   continues. (Unchecked **ANALYSIS** items are not this skill's job — they're
   for later/scheduled documentation sessions.)
3. A CAPTURE that's already `[x]` with a state file on disk is done; don't
   re-record it.

## Gotchas (romdev)

- **Close the playtest window before deterministic stepping.** Its real-time
  loop overshoots `state(op:'load')` and races `frame(op:'step')` — so do the
  verify step (Step 4) only after `playtest(op:'stop')`. Saving while the window
  is open is fine: you `host(op:'pause')` first and the human has stopped
  pressing.
- **The window only fights you while a human is actively pressing** (~2s).
  `frame`/`input` responses carry a `humanCoDriveWarning`, and
  `playtest({op:'status'})` / `catalog({op:'status'})` expose `humanInputActive`
  — wait for the human to settle before you pause + save.
- **`state(op:'load')` needs the media loaded first and clears active cheats.**
  `loadMedia` before any `state(op:'load')`, and re-apply cheats after if you
  rely on them. A load also rewrites all of RAM — expected here, since loading
  is exactly how you reset to `start_gameplay`.
- **A `.state` is ROM-content-independent** (RAM/CPU/PPU/APU, not the ROM), so a
  state recorded on the stock ROM reloads cleanly into a rebuilt/patched ROM of
  the same core — which is what makes these states durable across the project.
- Call `catalog({op:'categories'})` for the tool set and `platform({op:'doc'})`
  for the platform's memory map before interpreting any address in a snapshot.

## Report

When you pause or finish, summarize: which states were recorded this session
(names + paths, and that each is backed up in the repo's `states/`), which
checklist CAPTURE items are now `[x]`, the next unchecked CAPTURE a future
session should pick up, and a reminder that the ANALYSIS items are queued for
later documentation sessions.
