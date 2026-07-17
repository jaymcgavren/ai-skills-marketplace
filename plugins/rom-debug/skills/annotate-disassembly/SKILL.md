---
description: Identify what the routines in a ROM disassembly actually do and write evidence-backed comments, using the romdev MCP's live debugger (breakpoints, watchpoints, RAM injection, save states) instead of playing the game — plus deliberate, batched human help for the things only a human can do (play to a moment, name what's on screen, confirm a visual effect). Trigger on /rom-debug:annotate-disassembly, or when the user asks to comment/annotate/document disassembled code, figure out what a routine does, trace a game behavior to code, or continue reverse-engineering a disassembly project. Spans many sessions — progress lives in TODO.md, RAM_MAP.md, and the source comments themselves.
allowed-tools: Read Write Edit Bash AskUserQuestion mcp__romdev__memory mcp__romdev__breakpoint mcp__romdev__watch mcp__romdev__disasm mcp__romdev__cpu mcp__romdev__runUntil mcp__romdev__state mcp__romdev__frame mcp__romdev__input mcp__romdev__playtest mcp__romdev__cheats mcp__romdev__loadMedia mcp__romdev__symbols mcp__romdev__build mcp__romdev__sprites mcp__romdev__catalog mcp__romdev__platform
---

# annotate-disassembly

Turn a byte-exact but semantically empty disassembly (see `disassemble-rom`)
into commented, navigable source — one verified behavior at a time. The method
is **dynamic first**: instead of staring at 6502 and guessing, you run the
game under the debugger, make the behavior happen (usually by *injecting* it,
not playing to it), and let a breakpoint tell you which code is responsible.
Static tools (`xrefs`, `cfg`, `decompile`) then fill in the surrounding
structure.

Prerequisites: a project with canonical `src/`, a rebuild recipe, `TODO.md`,
and `RAM_MAP.md`. Load the ROM (`loadMedia`) at session start.

(Tool names use the `mcp__romdev__` prefix, which assumes the server was
registered as `romdev`; match the prefix to your registered name.)

## Two standing rules

**1. Byte-exact or it didn't happen.** Renaming labels and adding comments
never changes emitted bytes — but verify anyway: rebuild and `cmp` against the
original ROM before every commit. Never commit a mismatch. At session start,
rebuild the *untouched* checkout once and confirm it is still byte-identical,
so any later mismatch is attributable to this session's edits. Experimental
*behavioral* edits (testing a hypothesis by patching code) go in the
git-ignored `hack-src/` copy, never in canonical `src/`.

**2. Don't play the game.** Extended agent-driven gameplay burns enormous
token volume for nothing — every frame you drive is screenshots and input
calls that a save state or a RAM poke replaces. The substitutes, in order of
preference:

- **Load a save state** at the moment of interest (`state(op='load')`).
- **Inject the state** you need: write the RAM that makes the event true
  (grant the pickup, set the HP, spawn the object) and watch what the code
  does with it. This is faster AND more precise than causing it organically.
- **Frame-step** (`frame(op='step', frames=N)`) with inputs held via
  `input(op='set')` for short, targeted advances — seconds, not minutes.
- **`runUntil` / breakpoints** to fast-forward to a condition instead of
  watching frames go by.
- **Ask the human to play** (below) when reaching the moment takes skill or
  more than ~30 seconds of play.

## Enlisting the human (do this deliberately)

A human is cheap and fast at exactly the things you are slow and expensive at.
Ask — via `AskUserQuestion`, or by opening a `playtest` window and telling
them what to do in it — when you need:

- **Reaching a moment**: "Play to the first boss and pause; I'll save-state
  it." One human minute replaces thousands of drive-the-game tokens. Bank
  every such moment as a `.state` file so no one plays it twice.
- **Naming things**: you can see a sprite's bytes but not what the manual
  calls it. Show a screenshot, ask "what is this enemy/item called?" Use the
  game's real terminology in comments — future readers (and the user) search
  by it.
- **Confirming effects**: "I'm about to trigger X by poking RAM — watch the
  window and tell me what visibly happens." The human's eyes close the loop
  between code and on-screen meaning.
- **Domain knowledge**: someone who played the game knows there are, say,
  eight weapons and what each does. Ask before reverse-engineering a list the
  human can recite.

Batch questions into short, focused sessions with a stated purpose — not
open-ended "play for a while". When asking them to record moments, capture
several in one sitting (each `state(op='save', path=...)` is instant).

**Save-state gotcha (live vs. frozen):** a state captured at a paused or
transitional moment can have key dispatchers not running — code you're
watching never executes and everything looks dead. Before trusting a state
for tracing, set a breakpoint on a routine you *know* runs constantly (the
NMI handler, the main loop) and confirm it fires within a few frames. If it
doesn't, advance frames to a live moment and re-save.

## The loop (one behavior per iteration)

### 1. Pick a target

One concrete behavior from `TODO.md` (or the user's ask): "what decrements
lives", "how do pickups grant weapons", "what does the routine at $B268 do".
Small targets close; vague ones ("understand bank 3") don't.

### 2. Get an anchor address

Behaviors are found through the RAM they touch. If the relevant address is
already in `RAM_MAP.md`, use it. If not, find it first — the `locate-value`
skill's search loop (`memory(op='search')` → change it → `searchNext`) or
`memory(op='diffRuns')` for input-driven values. Every anchor you find is a
new `RAM_MAP.md` row, and each mapped address makes every routine that
touches it partially legible — the RAM map is the program's variable names.

### 3. Let the debugger name the code

- **Read before you instrument.** If the moment of interest is already live
  (a human just played to it, or a save state restored it), one batched
  `memory(op='read', offsets=[...])` of the mapped addresses may answer the
  question outright — occupancy, flags, which slot is the player — with no
  breakpoint at all. Arm instrumentation only when you need the *writer*,
  not the value.
- `watch` (or `breakpoint(on='write'/'read')`) on the anchor address, then
  trigger the behavior — by RAM injection where possible — and the PC that
  trips is your routine. This beats any amount of static reading.
- `disasm(target='references', address=...)` / `disasm(target='xrefs')` for
  who calls it. **Limitation:** static reference scans miss indirect and
  jump-table dispatch — common in NES engines (object-state dispatchers). If
  a routine has no visible callers, it's probably table-dispatched: use
  `breakpoint(on='pc')` and see what the stack/registers say at entry,
  or `disasm(target='pointerTable')` on a suspected table (`reverseHandler`
  answers "which state index lands here?").
- **Read the hit, not the aftermath:** a `breakpoint` hit returns
  `registersAtHit` (the register file at the break instant — a follow-up
  `cpu(op='read')` gives end-of-frame state instead) and takes
  `captureMemory:[…]` to read RAM at the hit in the same call. One breakpoint
  call with both replaces a break + two reads. `pressDuring` schedules input
  during the wait ("hold Start at frame 60") so driving the game to the
  trigger and watching for it is also one call.
- `disasm(target='cfg')` and `disasm(target='decompile')` to understand the
  routine's shape once you've pinned it. Decompiled pseudocode is for *your*
  comprehension — comments go on the asm.
- `cpu(op='step')` through short stretches when the flow is confusing.
- **Prove absence, not just presence.** A PC breakpoint that never fires
  over hundreds of frames spanning several game phases is real evidence — it
  can disprove a plausible hypothesis (e.g. "the IRQ vector drives raster
  effects" when the handler is a bare `rti` the CPU never reaches because
  the main thread keeps I set). Annotate the negative with its scope ("never
  hit across 600 frames of boot/intro/title") so a later session knows which
  scenes are still unchecked. Give the run enough frames: boot often burns
  hundreds of frames in init busy-waits (APU upload handshakes) before the
  handler you're watching is even enabled. And pick the window's *game
  phase* deliberately: intro/attract sequences often run on a separate code
  path that never touches gameplay subsystems (entity tables, mode
  variables), so "never fired during boot→title" says nothing about
  gameplay — a negative there needs an in-gameplay save state.
- **The emulator host is ephemeral.** A tool-server update or restart wipes
  the loaded ROM, save states in memory slots, cheats, and breakpoints —
  check `catalog(op='status')` and re-`loadMedia` before trusting any
  dynamic result after a gap or a version bump.

**Cheat/poke discipline:** after any `cheats(op='apply')` or RAM write meant
to hold a value, *read the byte back* to confirm it actually took —
`applied: true` does not mean the memory changed (short-form addresses can be
silently ignored; some cores don't re-poke frozen values). Cheats and
breakpoints are volatile: re-apply after `state(op='load')` or a reset.

### 4. Write the annotation

- Rename auto-labels (`L8F78` → `PlayerDeathHandler`) at the definition and
  every reference. Name routines for what they *do*, data for what it *is*.
- Comment the routine header with what it does **and the evidence grade**:
  "verified: write-breakpoint on $32 fired here on death" is durable;
  "inferred from callers, unverified" tells the next session what still
  needs proof. Never state a guess as fact — a wrong comment poisons every
  later reading of that code.
- Add every new address to `RAM_MAP.md` with its meaning and how it was
  verified.

### 5. Verify and persist

- Rebuild, `cmp` against the original ROM: must be byte-identical. Then
  commit.
- Tick the `TODO.md` item (add follow-up items you uncovered — unexplained
  branches, suspicious tables). Note surprising dead-ends too: knowing that
  "$037B is a timer, not lives" saves the next session from re-deriving it.

### Recovering misdecoded regions (65816 width desyncs and kin)

Auto-disassembly sometimes decodes a region under the wrong assumptions
(classic case: 65816 `.a8/.i8` vs `.a16/.i16` — a desynced 16-bit immediate
swallows the next byte, which then decodes as a stray `brk`/`rti`). When
rewriting such a region as real instructions:

- **The printed byte columns are ground truth.** Most generated sources
  carry the original bytes in each line's comment (`; 92E0 E0 00 14`).
  Hand-decode from those bytes under the corrected assumption — don't trust
  the mnemonics, and don't need a second disassembler pass.
- **Demand zero remainder.** The corrected decode must consume *exactly*
  the same bytes, with every branch target landing on an instruction
  boundary. A leftover or straddled byte means the assumption is still
  wrong somewhere.
- The byte-exact rebuild then proves the edit emitted identical bytes —
  it's the same free-transformation rule as labels and comments.

### When a trace looks dead

If breakpoints don't fire or nothing responds: (a) confirm the state is live
(see gotcha above); (b) question your input assumptions — games bind actions
to unexpected buttons, and holding the wrong one can mask a whole subsystem;
(c) confirm your poke actually landed by reading it back; (d) check you're in
the right bank — the same CPU address means different code after a bank
switch. Vary one assumption at a time, and ask the human what *they* do to
trigger the behavior on real hardware.
