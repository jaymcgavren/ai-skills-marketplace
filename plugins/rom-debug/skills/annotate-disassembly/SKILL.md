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
- **Drive with one long breakpoint run** when the moment is far away but
  reaching it takes no skill: neutralize the failure modes with cheats
  first (invincibility, infinite lives — read back to confirm they took),
  hold the one useful input via `input(op='set')` (e.g. autofire), and arm
  `breakpoint(on='pc', address=<the event's handler>, maxFrames=100000)`.
  That is ONE tool call, not gameplay — a 16k-frame drive through two boss
  fights this way cost the same as any other breakpoint. "Playing" is only
  expensive when it's screenshot-act-screenshot.
- **Ask the human to play** (below) when reaching the moment takes skill or
  more than ~30 seconds of play.

## Enlisting the human (do this deliberately)

A human is cheap and fast at exactly the things you are slow and expensive at.
Ask — via `AskUserQuestion`, or by opening a `playtest` window and telling
them what to do in it — when you need:

- **Reaching a moment**: "Play to the first boss and pause; I'll save-state
  it." One human minute replaces thousands of drive-the-game tokens. Bank
  every such moment as a `.state` file so no one plays it twice. But first
  check whether the moment is *script-triggered*: scrollers and stage-based
  games pace their events (bosses, area transitions) by scripted distance
  or time, and once the level script is decoded (see step 3) the "moment"
  is a known trigger value you can breakpoint your way to — one "needs a
  human" capture task fell exactly this way, the script naming the precise
  distance of the area-end event.
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

Expect "don't know" answers — even owners of the game recall less than you
hope. A negative answer is still a finding: record it in `TODO.md` ("human
doesn't recall a kill-all item — needs to be SEEN") so no session re-asks,
and fall back to showing rather than asking: inject the thing (spawn the
pickup, force the effect) and screenshot it — a human who can't recite an
item list can instantly name one they're looking at.

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

Community documentation (Data Crystal, TCRF, fan wikis) is a rich anchor
source — but import it as *hypotheses*, one TODO item per claim, and verify
each live before adopting it into `RAM_MAP.md`. Conflicts between your map
and theirs are high-value targets, not annoyances: resolving one either
catches your own error early (one session's "weapon power" byte turned out
to be ship tilt — the community map was right) or uncovers a subtlety (one
"conflict" was two independent bits of the same byte; both sources were
right). Record the resolution either way so no session re-litigates it.

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
- **Census a region in one pass.** `watch(on='range', distinctPCsOnly=true)`
  over a cluster of related unknowns returns every distinct PC that reads or
  writes the span in a single run — arm it once over live gameplay instead
  of guessing which single-address breakpoint to try first. But treat the
  PCs attached to a *value timeline* (`watch(on='mem')`) as approximate:
  cores may report the PC at the frame-boundary sample point, not the store
  instruction, and that PC can be pure fiction. Confirm a suspected writer
  with an exact write breakpoint, or statically — byte-scan the ROM for the
  store opcode (`sta $A4` = `85 A4` via `memory(op='readCart', findHex=…)`).
  Byte-scan hits need their own check: the same two bytes can straddle an
  instruction boundary (a `jsr` operand followed by the next opcode), so
  confirm each hit decodes as an instruction *at that address* before
  believing it.
- `disasm(target='references', address=...)` / `disasm(target='xrefs')` for
  who calls it. **Limitation:** static reference scans miss indirect and
  jump-table dispatch — common in NES engines (object-state dispatchers). If
  a routine has no visible callers, it's probably table-dispatched: use
  `breakpoint(on='pc')` and see what the stack/registers say at entry,
  or `disasm(target='pointerTable')` on a suspected table (`reverseHandler`
  answers "which state index lands here?").
- **Mine the call sites of anything you just named.** A raw byte-scan of the
  ROM for the call instruction (`jsr $873C` = `20 3C 87`) beats the desynced
  source text, and the few bytes *preceding* each site are the argument
  setup — they tell you where the parameter comes from with no further
  tracing. Constants (`lda #$0100`) enumerate the routine's fixed uses; an
  indexed load (`lda $2E,x`) names a struct field (that one scan turned
  "+$2E meaning unknown" into "per-enemy score value"). The site *count* is
  a signal too: a 3-line helper with 100+ callers is an engine primitive
  worth naming before anything else, and 0 callers on a routine that
  plainly does something means a dispatch mechanism you haven't found yet —
  both facts belong in the annotation.
- **Read the hit, not the aftermath:** a `breakpoint` hit returns
  `registersAtHit` (the register file at the break instant — a follow-up
  `cpu(op='read')` gives end-of-frame state instead) and takes
  `captureMemory:[…]` to read RAM at the hit in the same call. One breakpoint
  call with both replaces a break + two reads. `pressDuring` schedules input
  during the wait ("hold Start at frame 60") so driving the game to the
  trigger and watching for it is also one call. Timing caveat (observed on
  fceumm): the captured RAM can still hold PRE-handler values — treat
  `captureMemory` as trigger-time context, and verify the handler's *side
  effects* by stepping a frame and re-reading.
- **Decode interpreters, then verify by prediction.** Script/bytecode
  interpreters (level scripts, spawn choreography, cutscene players) are
  the densest annotation wins: one grammar explains a whole data bank. Once
  the dispatch is understood, write a small decoder for the format and
  commit it to the project (`tools/`) — the decoded timeline is
  documentation in itself (where every boss/event sits) and later sessions
  decode sibling scripts for free. Then verify by *prediction*: from the
  cart bytes, predict which handler fires next and at what trigger value,
  and arm a pc-breakpoint on the predicted handler. A hit at exactly the
  predicted trigger — with the dispatcher's register signature at entry —
  proves the grammar, the dispatch decode, and the pointer seeding in one
  shot; far stronger evidence than watching writes after the fact.
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
  gameplay — a negative there needs an in-gameplay save state. Games with
  distinct modes (action vs. sim/menu/overworld) may run entirely separate
  engines per mode, each with its own main loop in a different bank — and
  they can REUSE the same RAM region for different structures, so a table
  layout you proved in one mode is unproven in the others. A per-frame
  routine that never fires in some mode is itself a finding worth
  annotating: it scopes every conclusion built on that routine to the modes
  where it runs.
- **The emulator host is ephemeral.** A tool-server update or restart wipes
  the loaded ROM, save states in memory slots, cheats, and breakpoints —
  check `catalog(op='status')` and re-`loadMedia` before trusting any
  dynamic result after a gap or a version bump.

**Cheat/poke discipline:** after any `cheats(op='apply')` or RAM write meant
to hold a value, *read the byte back* to confirm it actually took —
`applied: true` does not mean the memory changed (short-form addresses can be
silently ignored; some cores don't re-poke frozen values). Cheats and
breakpoints are volatile: re-apply after `state(op='load')` or a reset.
Two poke outcomes that LOOK like "wrong address" but aren't:

- **The poke holds but the screen doesn't change.** HUD counters and bars are
  often redrawn only when the *game's own code* changes the value — an
  external poke never triggers the redraw path. Force one game-driven update
  (take a hit, let a timer tick, respawn) before ruling the address out; the
  next redraw will render your poked value if the address is right.
- **The poke reverts within a frame or two.** The byte is a *derived* value.
  Don't abandon it — write-watchpoint it, and the restorer's PC hands you the
  next value up the derivation chain. Repeat until a poke sticks: that's the
  true source. (An ActRaiser example: HUD population ← per-town array ←
  recomputed every frame from the town map's house records — three
  watchpoint hops, and the "variable" turned out to be a pure function of
  map state.)

When the verification is visual — reading a score or counter off a
screenshot — crop and scale the region first (`frame(op='screenshot')`
takes both). Native-resolution console fonts misread easily; a "wrong"
digit may be your eyes, not the poke, so re-read the RAM before concluding
the poke failed.

### 4. Write the annotation

- Rename auto-labels (`L8F78` → `PlayerDeathHandler`) at the definition and
  every reference. Name routines for what they *do*, data for what it *is*.
- Comment the routine header with what it does **and the evidence grade**:
  "verified: write-breakpoint on $32 fired here on death" is durable;
  "inferred from callers, unverified" tells the next session what still
  needs proof. Never state a guess as fact — a wrong comment poisons every
  later reading of that code.
- **Transcribe conditions from the instructions, not from your notes.** When
  a comment describes a branch ("suppresses X while Y"), re-derive the
  polarity from the opcodes in front of you at write time. Session notes and
  compaction summaries paraphrase, and a paraphrase can silently invert a
  condition — one "suppresses spawning while the boss is locked" note was
  exactly backwards (the gate fired when the boss-locked bit was *clear*);
  reading the `beq` again while writing the comment is what caught it.
- Add every new address to `RAM_MAP.md` with its meaning and how it was
  verified.

### 5. Verify and persist

- Rebuild, `cmp` against the original ROM: must be byte-identical. Then
  commit.
- Tick the `TODO.md` item (add follow-up items you uncovered — unexplained
  branches, suspicious tables). Note surprising dead-ends too: knowing that
  "$037B is a timer, not lives" saves the next session from re-deriving it.
- **Stopping mid-increment (context running out, user interrupt):** if the
  working tree holds an edit that doesn't yet rebuild, do NOT commit
  anything — instead write a prominent IN-FLIGHT section at the top of
  `TODO.md`: exactly what was changed, why it doesn't assemble yet (the
  specific errors/collisions), the finish steps, and the revert path.
  The next session's "unmodified checkout rebuilds identical" check will
  fail on a dirty tree; the note is what tells that session whether to
  finish the edit or `git checkout` it, instead of misreading the
  mismatch as corruption.
- **Inheriting a dirty tree with NO in-flight note** (the other way a
  session can end abruptly): diff it and rebuild it before anything else.
  If the rebuild is byte-identical, you are almost certainly holding
  *finished work that was never persisted* — a verified find whose
  RAM_MAP/TODO updates and commit got cut off. The evidence grade written
  into the annotation comment is what makes this recoverable ("VERIFIED
  LIVE: write-breakpoint fired at..." tells you the proof already
  happened); finish the bookkeeping and commit it as its own increment
  before starting new work. If the rebuild mismatches, treat it as a
  failed experiment: `git checkout` it and note in TODO what was
  attempted. This is why evidence goes in the comment, not the commit
  message — the comment survives an interrupted session.

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
- **Not every stray `brk` is a desync artifact.** The usual heuristic — a
  `brk`/`rti` right after an immediate is a swallowed operand byte — has a
  real exception: engines that use the BRK (or COP) vector as a software
  interrupt. If the vector's handler *stores A somewhere and returns*
  (`php / sta $xx / plp / rti`), then `lda #$0D` + `brk #$00` in genuinely
  8-bit code is a deliberate 2-byte syscall (opcode + signature byte) —
  in one SNES engine it was the sound-effect request convention, sitting
  on hot gameplay paths. Check the vector handler once, early: it decides
  whether paired `00` bytes in that ROM are decode errors or an API. The
  zero-remainder rule still adjudicates — a real syscall decodes cleanly
  in place; a swallowed byte doesn't.
- **Plain data-as-code is the same job without the width games.** On 6502,
  a parameter table decoded as `ora`/`sed` garbage becomes `.byte` rows once
  you know what it holds — same zero-remainder rule, same cmp gate. When
  another instruction references an address *inside* the re-decoded bytes
  (overlapping code/data reads are common in tight NES engines), don't try
  to place a label mid-row: define the reference as an equate
  (`L973A := $973A`) and keep the emitted byte stream label-free.
- **RTS-trick inline dispatch tables** (the bytes right after a
  `jsr Dispatcher` that are really pointer entries stored as handler-1)
  resplit as `.word Cmd0Handler-1, Cmd1Handler-1, …` — the `-1` expressions
  assemble to identical bytes while giving every handler a real, greppable
  label. Related pattern: per-id tables indexed from a nonzero first id
  (ids 2..N) often place their *base* first-id bytes before the first real
  entry, overlapping the preceding instruction's operand bytes or the
  previous table's tail — chains of these are common in tight engines. One
  equate per base, `.byte` rows only for the real entries.
- The byte-exact rebuild then proves the edit emitted identical bytes —
  it's the same free-transformation rule as labels and comments.

### When a trace looks dead

If breakpoints don't fire or nothing responds: (a) confirm the state is live
(see gotcha above); (b) question your input assumptions — games bind actions
to unexpected buttons, and holding the wrong one can mask a whole subsystem;
(c) confirm your poke actually landed by reading it back; (d) check you're in
the right bank — the same CPU address means different code after a bank
switch; (e) **arm the watchpoint on every alias of the address**. On SNES,
WRAM `$0000-$1FFF` is mirrored into banks $00-$3F *and* lives at
`$7E0000-$7E1FFF`, and cores may arm only the literal bus address you gave —
a watch on `$0218` can silently miss the game's `sta f:$7E0218`. A
watchpoint negative is only evidence once you've covered the mirror forms
(and the same goes for read-watches backing a "nothing consumes this"
claim). Vary one assumption at a time, and ask the human what *they* do to
trigger the behavior on real hardware.
