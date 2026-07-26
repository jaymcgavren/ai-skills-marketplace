---
description: >
  Mine a documented disassembly for interesting single-byte ROM patches and
  turn them into byte-verified, published Game Genie codes. Use when the user asks
  for Game Genie / cheat codes for a game that has an annotated disassembly,
  ROM map, or RAM map available — especially when they want *unusual* codes
  that demonstrate how the program works, not the classic infinite-lives set.
  Runs a systematic candidate sweep first, then works codes one at a time.
allowed-tools: Task Read Write Edit Grep Glob Bash AskUserQuestion mcp__romdev__memory mcp__romdev__cheats mcp__romdev__state mcp__romdev__frame mcp__romdev__input mcp__romdev__playtest mcp__romdev__disasm mcp__romdev__loadMedia mcp__romdev__symbols mcp__romdev__catalog mcp__romdev__platform
---

# Game Genie codes from a disassembly

Turn documented reverse-engineering findings into single-byte ROM-patch codes.
The work runs in **two phases**:

1. **Systematic sweep** — mine the whole disassembly, subsystem by subsystem,
   into a written *candidate backlog* with a *coverage log*, so the search is
   resumable and nothing gets mined twice. Breadth first; no verifying or
   encoding here. Do this yourself, in the main session.
2. **Per-code pipeline** — take one candidate at a time through **verify bytes
   → encode → round-trip → publish**. **Delegate this to a subagent running a
   lower-power model** (see Phase 2) — the backlog is written precisely so a
   cheaper model can execute each item.

**Always finish the sweep before working individual codes.** A half-mined
disassembly yields a lopsided code set — a dozen codes from the first subsystem
you happened to read and nothing from the rest. The sweep is what makes the
final set *broad* and *representative*, which is the whole point of these codes.

## Prerequisites

- An annotated disassembly (or at minimum a ROM map) for the target game.
- The romdev MCP `memory` tool with the game loaded, so you can read the
  original bytes out of the actual ROM when verifying (Step 1).
- Know the mapper/banking layout before you start — you need it to read the
  original byte from the right bank when verifying (Step 1). It does not
  affect the code format: every code is 8-letter with a compare byte.

## Where to start — check the sweep state first

Before anything else, look for an existing candidate backlog and coverage log.
By convention they live in the project's task list (e.g. `TODO.md`) or wherever
the repo keeps its backlog — a section of checklist items plus a "coverage log"
listing which parts of the disassembly have been swept. There are three cases:

- **No backlog recorded** → go to **Phase 1** and build it from scratch.
- **A sweep is in progress** (the coverage log still lists unswept regions) →
  go to **Phase 1** and *continue* it — extend the same backlog and log, don't
  restart. Pick up at the next unswept region.
- **The sweep is complete** (coverage log shows every gameplay-relevant region
  swept, and unchecked candidates are waiting) → go to **Phase 2** and dispatch
  candidates to a lower-power subagent, most interesting first.

Only move to Phase 2 once the sweep is complete. If the user explicitly asks
for one specific code and a backlog already covers it, you may jump straight to
Phase 2 for that item — but a bare "find me Game Genie codes" means sweep first.

## Phase 1 — Systematic sweep: build the candidate backlog

Mine the disassembly for candidates and **record them; do not verify or encode
in this phase.** The output is a backlog a lower-power model can later
execute one item at a time. Keep it systematic and resumable:

**Focus on what affects gameplay most directly.** Enumerate the game's
subsystems from the disassembly's structure (its symbol map, section
labels, and comments), then work them in priority order — player physics and
control, powerups and player state, enemies and their AI/parameter tables,
scoring/items/level-flow, mode and difficulty logic. Data tables and init
immediates in these areas make the best codes because the mechanism is fully
explainable. Leave rendering, sound, and pure-cosmetic tables for last (or skip
them) unless the user wants visual-oddity codes.

**Go region by region and log coverage as you go.** Sweep one subsystem, record
its candidates, then tick that region off in a coverage log that also lists the
regions still to do (with line ranges). This is what lets a later session — or
a cheaper model — resume exactly where you stopped instead of re-reading from
the top.

### What to look for

Grep the annotated source and maps for these patterns. The goal is codes that
each demonstrate a *different documented subsystem* — breadth over quantity.

1. **Secret / easter-egg gates** — button-chord checks (`and #$C0 / cmp
   #$C0`-style). Change the `cmp` operand to `$00` to invert the secret: it
   fires with *no* input, and the chord gets the normal path.
2. **Debug leftovers** — features gated on a debug flag (`lda flag / beq
   skip`). Flip the branch opcode (`beq $F0` ↔ `bne $D0`) so the debug
   display/feature is always on for normal players.
3. **Boot/init immediates** — `lda #imm / sta` in cold-boot or mode-init
   code: starting area/level, timers, initial state. Includes "boot into
   unused content" codes when the disassembly proved an area/level ID is a
   null table row.
4. **Gameplay data tables** — edit one entry in a documented parameter table
   (weapon speed/type rows, per-level tables). These make great codes because
   the mechanism is fully explainable.
5. **Shared state machines** — a routine that writes a state constant into
   object slots (`lda #state / sta slots,x` sweeps). Substituting a different
   documented state redirects behavior in surprising ways (e.g. a
   kill-everything sweep that turns enemies into pickups). Expect and
   document glitches — uninitialized fields are part of the finding.
6. **Attract-mode / RNG parameters** — masks and offsets in demo
   randomizers (`and #mask / adc #base`).
7. **Documented bugs** — publish the *fix* as a code. A Game Genie code that
   repairs the game is the most unusual code of all.

**Anti-goals:** infinite lives, invincibility, 99 ammo — skip unless asked.

**Candidate quality rules:**
- One byte per effect. Real Game Genie hardware takes only 3 codes at once,
  so an effect needing >3 bytes is less desirable; prefer exactly 1.
- Prefer *operands* (immediates, table bytes, branch targets) over opcodes.
  Opcode changes are OK only for well-understood swaps (`beq`↔`bne`,
  branch → `lda #imm` to neuter it).
- The patched byte must be read at a well-understood moment. Beware bytes
  shared by multiple code paths (note every caller in the writeup).

### Record each candidate so a cheaper model can execute it

Write candidates as checklist items in the backlog. Each entry needs enough
that a lower-power model can run Phase 2 without re-deriving anything:

- The **subsystem** and the **label / line number** the byte lives at.
- The **exact original byte(s)** and what each table entry / immediate means.
- A **suggested new value** and the **effect** it should produce (a starting
  point — the executor can pick whatever reads most clearly on screen).
- A one-line **mechanism sketch**: the routine/table that reads the byte, and
  any *shared* readers to watch for.

Also carry the Phase-2 reminders into the backlog header so the executor has
them in front of it: verify the byte against BOTH the source line and the live
ROM, encode with a round-trip-checked script, then publish. Restate the code
format as a hard rule — **8-letter codes with a compare byte, never 6-letter,
regardless of which bank the address is in** — and note that the pipeline
never edits the ROM on disk, so **do not modify the disassembly source.**

### Keep a coverage log

Maintain a log alongside the candidates recording which regions of the
disassembly have been swept and which are still to do (with line ranges), so
future sessions extend the sweep instead of repeating it. A useful shape:

```
Swept (<date>, <pass name>):
- [x] <subsystem> (~<line range>)
- [x] ...

Not yet swept — candidates for the next pass:
- [ ] <subsystem> (~<line range>) — <why it might be rich>
- [ ] ...
```

The sweep is **complete** when every gameplay-relevant region is checked off.
Cosmetic/sound regions may be left explicitly deferred rather than swept.

## Phase 2 — Turn one candidate into a published code

**Delegate this phase to a subagent on a lower-power model** (via the `Task`
tool — e.g. Haiku, or Sonnet if the encoding step needs it). The backlog you
built in Phase 1 already carries everything the executor needs, so a cheaper
model is the right tool; your job in the main session is to dispatch and to
collect the results.

Dispatch one candidate per subagent run (or a small batch of related ones).
Give the subagent: the backlog item, the path to the disassembly and the ROM,
the project's code format, and the instruction to run all three steps below and
report the published entry back.

The per-candidate steps follow. Step 1 is the gate: a candidate whose bytes
don't check out is a finding too — record why and drop or revise it.

### Step 1 — Verify the original bytes

Confirm the exact address and original byte against BOTH the annotated source
listing and the actual ROM (`memory({op:'readCart'})` at the CPU address).
Quote the source line in your notes. If they disagree, the disassembly is wrong
there — stop and investigate.

### Step 2 — Encode, then round-trip

**Always emit the 8-letter form, with a compare byte. Never record a 6-letter
code** — not even for an address in a fixed, always-mapped bank where a
5-letter/6-letter code would technically resolve. The compare byte is what
makes a code state which original byte it expects, so it stays correct if the
address is ever reached with a different bank mapped in, and it is
self-documenting when someone reads the code back later.

NES letter alphabet (letter → nibble): `A=0 P=1 Z=2 L=3 G=4 I=5 T=6 Y=7
E=8 O=9 X=A U=B K=C S=D V=E N=F`. Canonical **decode** (letters n[0..7] as
nibble values):

```python
address = 0x8000 + (((n[3] & 7) << 12)
        | ((n[5] & 7) << 8) | ((n[4] & 8) << 8)
        | ((n[2] & 7) << 4) | ((n[1] & 8) << 4)
        |  (n[4] & 7)       |  (n[3] & 8))
# 8-letter; bit 3 of n[2] must be SET to mark it
data    = ((n[1] & 7) << 4) | ((n[0] & 8) << 4) | (n[0] & 7) | (n[7] & 8)
compare = ((n[7] & 7) << 4) | ((n[6] & 8) << 4) | (n[6] & 7) | (n[5] & 8)
```

Write the **encoder as the inverse of this decoder** in a scratchpad script,
and make the script assert the round trip: encode(addr, data, compare) →
decode → must reproduce the inputs exactly. Do not hand-derive bit shuffles.

One compare-byte caveat worth knowing: a different bank holding the same byte
at that offset will also match, so the patch can land somewhere unintended.
Check the other banks for that byte at that offset and note it in the writeup.

Other platforms (SNES, Game Boy, Genesis) have Game Genie schemes with
different alphabets and scrambles — look the scheme up per platform; the
verify and publish steps are platform-independent. Where a platform's scheme
offers a choice, apply the same principle as the NES rule above and pick the
form that carries a compare value.

### Step 3 — Publish

Write `notes/game_genie_codes.md` (or the repo's convention) with one entry
per code:

- The letter code, the raw `ADDR:NEW:OLD` form, and which banks/regions.
- A one-paragraph **mechanism** citing the routine/table names from the
  disassembly — the point of these codes is that they're explainable, and with
  no playtest in the pipeline the mechanism *is* the evidence.
- Evidence: the source line the byte came from and the ROM read that confirmed
  it (Step 1). State plainly that the effect is derived from the disassembly
  and has not been observed in play — don't write it up as if it had been.
- Caveats: shared code paths, likely glitches, interactions with other codes.

Do not modify the canonical disassembly source for any of this — nothing in
this pipeline writes to the ROM or the sources, so no rebuild/byte-compare gate
applies. Once a candidate is published, check it off in the backlog.
