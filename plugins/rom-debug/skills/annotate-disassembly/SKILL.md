---
description: Identify what the routines in a ROM disassembly actually do and write evidence-backed comments, using the romdev MCP's live debugger (breakpoints, watchpoints, RAM injection, save states) instead of playing the game — plus deliberate, batched human help for the things only a human can do (play to a moment, name what's on screen, confirm a visual effect). Trigger on /rom-debug:annotate-disassembly, or when the user asks to comment/annotate/document disassembled code, figure out what a routine does, trace a game behavior to code, or continue reverse-engineering a disassembly project. Spans many sessions — progress lives in TODO.md, RAM_MAP.md, and the source comments themselves.
allowed-tools: Read Write Edit Bash AskUserQuestion mcp__romdev__memory mcp__romdev__breakpoint mcp__romdev__watch mcp__romdev__disasm mcp__romdev__cpu mcp__romdev__runUntil mcp__romdev__state mcp__romdev__frame mcp__romdev__input mcp__romdev__playtest mcp__romdev__cheats mcp__romdev__loadMedia mcp__romdev__symbols mcp__romdev__build mcp__romdev__sprites mcp__romdev__catalog mcp__romdev__platform
---

# annotate-disassembly

Turn a byte-exact but semantically empty disassembly (see `disassemble-rom`)
into commented, navigable source — one verified behavior at a time. The method
is **dynamic first**: instead of staring at raw assembly and guessing, you run
the game under the debugger, make the behavior happen (usually by *injecting*
it, not playing to it), and let a breakpoint tell you which code is
responsible. Static tools (`xrefs`, `cfg`, `decompile`) then fill in the
surrounding structure.

Prerequisites: a project with canonical `src/`, a rebuild recipe, `TODO.md`,
and `RAM_MAP.md`. Load the ROM (`loadMedia`) at session start. Rebuild with
the recipe the project's CLAUDE.md records — don't assume a particular
platform's path. `build({output:'reassemble'})` is the universal one (all
classic platforms); some projects ship a platform-native recipe instead
(e.g. NES cc65 `build({output:'rom'})`).

(Tool names use the `mcp__romdev__` prefix, which assumes the server was
registered as `romdev`; match the prefix to your registered name. If the
romdev tools are deferred and you load them with `ToolSearch`, select them by
their **full prefixed name** — `ToolSearch("select:mcp__romdev__loadMedia,…")`;
bare names like `loadMedia` match nothing and silently cost a round trip.)

## Two standing rules

**1. Byte-exact or it didn't happen.** Renaming labels and adding comments
never changes emitted bytes — but verify anyway: rebuild and `cmp` against the
original ROM before every commit. Never commit a mismatch. Experimental
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

**Known commercial title?** If the ROM turns out to be a documented game
(the title screen usually tells you — bank a screenshot early), existing
community RAM maps and disassemblies are a legitimate way to *prioritize
targets and predict* what an address or routine is — use them to form
hypotheses fast. They are never a substitute for proof: confirm each claim
with the live debugger before you annotate, and grade it as verified only
when your own breakpoint/watch says so, not because a wiki agreed.

Batch questions into short, focused sessions with a stated purpose — not
open-ended "play for a while". When asking them to record moments, capture
several in one sitting (each `state(op='save', path=...)` is instant).

**Save-state gotcha (live vs. frozen):** a state captured at a paused or
transitional moment can have key dispatchers not running — code you're
watching never executes and everything looks dead. Before trusting a state
for tracing, set a breakpoint on a routine you *know* runs constantly (the
vblank/NMI handler, the main loop) and confirm it fires within a few frames.
If it doesn't, advance frames to a live moment and re-save. Restore the
anchor explicitly — `state(op='load')` first, *then* arm the breakpoint,
rather than folding the restore into the breakpoint call; if the expected
every-frame hit doesn't arrive within a frame or two, suspect the restore,
not the code.

## When the disassembly is a data-only floor (carve first)

Some projects don't start with readable instructions: on platforms where the
disassembler can't statically prove what's code (documented case: SNES —
the 65816's `rep`/`sep` register-width state defeats static decoding; the
same applies anywhere decode state is dynamic, e.g. ARM-vs-Thumb on GBA),
`disassemble-rom` emits entire banks as `.byte` rows — byte-exact, 0%
readable. The project's CLAUDE.md and bank-file headers say so. There are no
instructions to comment and no auto-labels to rename yet; annotation starts
one level lower, by **carving** proven-code ranges out of the `.byte` data:

1. **Prove it's code before carving.** Entry points come from the platform's
   vector/header layout (6502 vectors at `$FFFA`; SNES header+vectors at
   `$00FFC0–$00FFFF`; GB entry/RST/IRQ vectors at `$0000–$0104`; m68k vector
   table at `$000000`), from the auto-detected function list, and — the real
   proof — from the live debugger: a `breakpoint(on='pc')` that fires at an
   address, or a `watch(on='pc')` coverage trace over a range, shows actual
   execution. One caveat: some cores auto-run the whole reset sequence
   during `loadMedia` before the debugger attaches (documented case:
   snes9x), so a pc-breakpoint on one-shot boot code never fires from a
   fresh load — carve those by static decode + the rebuild gate, and say so
   in TODO.md.
2. **Decode the range** (`disasm(target='rom')` on the address range, or
   step the live core), tracking any dynamic decode state across it — on
   65816 that's the `rep`/`sep` width changes; each immediate's size depends
   on the state at that instruction.
   - **`disasm(target='rom')` may reject a high/banked address.** On SNES,
     da65 aborts with `Precondition violated: StartAddr < 0x10000` for any
     CPU address outside bank 0 — i.e. exactly the higher banks you're
     usually carving. Fall back to reading the raw bytes and hand-decoding:
     `memory(op='readCart', {cpuAddress})` with the bank encoded in the
     address itself — the full banked CPU address (e.g. SNES `$02:87B1` is
     `cpuAddress: 0x0287B1`), never a bank-local address plus a separate
     `bank` param — and **cross-check the echoed `fileOffset` against the
     offset you expect** before trusting the bytes; that check is cheap
     insurance on any banked read or disasm call. On a dynamic-width
     CPU you're hand-tracking widths against the bytes anyway, so a byte dump
     is often all `target='rom'` would have given you.
3. **Convert the `.byte` range in `hack-src/`, never directly in `src/`.**
   Find the lines by arithmetic: each bank file's header comment records its
   base address and file offset, and the emitted `.byte` rows are 16 bytes
   each from the `.org`. Rules that keep the assembler's output identical:
   - **Force encodings the assembler would otherwise shorten.** Assemblers
     pick the smallest form by default; re-emit the original's. On ca65
     65816 that means `.a8/.a16/.i8/.i16` directives at every `rep`/`sep`
     (immediate widths) and `f:`/`a:` operand prefixes where the original
     used long/absolute addressing on a numerically small operand. Other
     CPUs/assemblers have their own equivalents — the principle is the same.
   - **Literal addresses are fine as operands** (`jsr $8241`, `brl $80E5`) —
     no label needed for targets still buried in `.byte` data. Define labels
     only for branch targets inside the carve.
   - **Far operands stay literal.** Labels defined under a bank-local `.org`
     don't carry a bank — a long/far operand written through a label
     assembles with the wrong bank (ca65 emits bank $00). Keep far operands
     literal (`f:$0287FE`); bare labels are fine only for bank-local forms
     (`jsr Label`). The byte-compare gate catches the mistake, but knowing
     the rule saves the cycle.
   - **Width-safe routines first.** A routine whose only immediates sit
     under decode state it sets *itself* (e.g. its own `rep`/`sep` on
     65816) is safe to carve even when the entry state is unknown — the
     entry-width declarations can't change the emitted bytes. Prefer such
     routines when live evidence for the entry state is unavailable.
   - **Start and end on instruction boundaries**, and keep the leftover
     bytes of a partially-converted `.byte` row as a shorter `.byte` row.
4. **Rebuild + `cmp`, then promote.** Only a byte-identical carve moves to
   canonical `src/` (copy the file, rebuild from `src/`, `cmp` again,
   commit). The reassemble path's refusal to accept length changes works in
   your favor: a wrong width or shortened encoding changes the length and
   fails loudly.

**`cmp` proves the bytes; single-stepping proves the meaning.** A
byte-identical rebuild only says your carve *re-emits* the original — it says
nothing about whether your comments describe what the code *does*. On a
dynamic-width CPU, the durable proof of the decode itself is live: arm a
`breakpoint(on='pc')` at the entry, then ONE
`frame(op='stepInstructions', count=N)` call returns an ordered `{pc, width}`
trace across the whole carved range — every PC delta checks a decoded
instruction length (a mismatched `.a8`/`.i16` immediate shows up immediately
as a wrong-sized step), and the terminal return's landing address is
captured in the same trace (it must be the address right after the call
site). (On older servers without the bulk op, loop
`frame(op='stepInstruction')`.) Drive conditional
blocks that don't fire in the captured state by injecting the gate variable
(`memory(op='write')` the flag, then step through the taken branch) — that's
how you verify a path the idle state never takes. Cite both in the header
("every boundary live single-stepped; body forced via `$DC:=$0100`") so the
grade is legible next session.

Carve breadth-first — entry points (reset, the vblank/NMI handler) and the
routines your traces actually hit — not bank-at-a-time sweeps. A carved
routine's header cites its evidence like any other annotation, and each carve
makes the surrounding `.byte` mass more navigable.

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

- `watch` (or `breakpoint(on='write'/'read')`) on the anchor address, then
  trigger the behavior — by RAM injection where possible — and the PC that
  trips is your routine. This beats any amount of static reading.
- `disasm(target='references', address=...)` / `disasm(target='xrefs')` for
  who calls it — `references` can also scan the ROM for the address stored
  as a raw *pointer* (`includeTableHits`, auto-on when no direct refs),
  which finds jump-table call sites no jsr/jmp names. On banked or
  data-floor projects where the reference scanner is unreliable, a raw
  byte-pattern scan of the ROM image for the call encodings is cheap and
  decisive (e.g. 65816: `jsr $87B1` = `20 B1 87` scoped to the bank that
  can reach it; `jsl` = `22` + 24-bit address, anywhere). **Zero hits is
  durable negative evidence** — record "no static callers: indirectly
  dispatched or dead" in the routine header instead of guessing a caller.
  **Limitation:** static reference scans miss indirect and
  jump-table dispatch — object-state dispatchers are common in engines on
  every platform. If a routine has no visible callers, it's probably
  table-dispatched: `breakpoint(on='pc')` at its entry and read the hit's
  `registersAtHit` snapshot (what do the stack/registers say?);
  `breakpoint(on='jumptable')` at a suspected dispatcher records its real
  computed targets live; or `disasm(target='pointerTable')` on a
  suspected table (`reverseHandler` answers "which state index lands
  here?"). CPUs with explicit far calls (65816 `jsl`, m68k absolute `jsr`)
  make cross-bank callers findable statically; window-banked platforms (NES
  mappers, GB MBC, Sega mappers) hide different callers behind the same CPU
  address, so expect to lean on the dynamic tools more there.
- `disasm(target='cfg')` and `disasm(target='decompile')` to understand the
  routine's shape once you've pinned it. Decompiled pseudocode is for *your*
  comprehension — comments go on the asm.
- `frame(op='stepInstructions', count=N)` through short stretches when the
  flow is confusing.

**Cheat/poke discipline:** after any `cheats(op='apply')` or RAM write meant
to hold a value, *read the byte back* to confirm it actually took —
`applied: true` does not mean the memory changed (short-form addresses can be
silently ignored; some cores don't re-poke frozen values). Cheats and
breakpoints are volatile: re-apply after `state(op='load')` or a reset.

### 4. Write the annotation

- Rename auto-labels (`L8F78` → `PlayerDeathHandler`) at the definition and
  every reference (on a data-only floor there are none yet — you create the
  names as you carve). Name routines for what they *do*, data for what it
  *is*.
  **Do it safely:** `grep -rn "L8F78" src/` across *all* bank files first to
  see every occurrence (definition + callers can span banks), replace them
  all, then rebuild + `cmp`. A *partial* rename is self-catching — the
  assembler fails on the now-undefined symbol — so the danger isn't a silent
  byte change, it's a broken build; the `cmp` gate catches both.
- **Prefer inserting comment lines *above* a line over editing the line
  in place.** Disassembly lines often carry wide, exact comment columns
  (`; 8000 86 10  ..`); an in-line `Edit` has to match that whitespace
  exactly and often fails. A new comment line above the instruction (or a
  header block above the routine) is lower-friction and just as useful.
  When you write carved instructions yourself, give each one an address
  comment (`; $8053`) — data-only banks have no per-line addresses, and
  those comments are what makes the file navigable next session.
- Comment the routine header with what it does **and the evidence grade**:
  "verified: write-breakpoint on $32 fired here on death" is durable;
  "inferred from callers, unverified" tells the next session what still
  needs proof. Negative live evidence is a grade too — "did not fire in 600
  title frames incl. Start presses" is honest and durable; write that
  rather than silently upgrading a static decode. Never state a guess as
  fact — a wrong comment poisons every later reading of that code.
- Add every new address to `RAM_MAP.md` with its meaning and how it was
  verified.

### 5. Verify and persist

- Rebuild, `cmp` against the original ROM: must be byte-identical. Then
  commit.
- **Batch the trivial ones.** The one-behavior-per-commit rhythm is for
  findings that took a real trace. A cluster of self-evident leaf helpers
  (a 4-instruction scroll-setter, a sprite-slot allocator you read at a
  glance) doesn't each deserve its own rebuild+`cmp`+commit — verify them,
  name them together, and commit the batch. Keep the strict one-behavior
  rule for anything you had to prove dynamically.
- Tick the `TODO.md` item (add follow-up items you uncovered — unexplained
  branches, suspicious tables). Note surprising dead-ends too: knowing that
  "$037B is a timer, not lives" saves the next session from re-deriving it.

### When a trace looks dead

If breakpoints don't fire or nothing responds: (a) confirm the state is live
(see gotcha above); (b) question your input assumptions — games bind actions
to unexpected buttons, and holding the wrong one can mask a whole subsystem;
(c) confirm your poke actually landed by reading it back; (d) check you're in
the right bank — on window-banked platforms the same CPU address means
different code after a bank switch. Vary one assumption at a time, and ask
the human what *they* do to trigger the behavior on real hardware.
