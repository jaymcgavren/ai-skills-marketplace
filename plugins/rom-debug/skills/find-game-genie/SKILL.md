---
name: game-genie
description: >
  Mine a documented disassembly for interesting single-byte ROM patches and
  turn them into verified, published Game Genie codes. Use when the user asks
  for Game Genie / cheat codes for a game that has an annotated disassembly,
  ROM map, or RAM map available — especially when they want *unusual* codes
  that demonstrate how the program works, not the classic infinite-lives set.
---

# Game Genie codes from a disassembly

Turn documented reverse-engineering findings into single-byte ROM-patch codes.
The pipeline: **mine candidates → verify bytes → live-test raw → encode →
round-trip check → live-test letters → publish**. Never publish a code that
hasn't passed the live tests.

## Prerequisites

- An annotated disassembly (or at minimum a ROM map) for the target game.
- The romdev MCP tools (`memory`, `cheats`, `state`, `frame`) with the game
  loaded, plus save states covering the game modes the codes touch (title,
  gameplay, ending, etc.).
- Know the mapper/banking layout before encoding anything (see "Banking").

## Step 1 — Mine the source for candidates

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

## Step 2 — Verify the original bytes

For each candidate, confirm the exact address and original byte against BOTH
the annotated source listing and the actual ROM (`memory({op:'readCart'})`
at the CPU address). Quote the source line in your notes. If they disagree,
the disassembly is wrong there — stop and investigate.

## Step 3 — Live-test as a raw patch first

Apply the raw three-part ROM-patch cheat `ADDR:NEW:OLD` (address:value:compare
— the three-part form is a ROM patch, not a RAM freeze) and play the relevant
moment from a suitable save state. Gotchas learned the hard way:

- Addresses must be **all 4 hex digits**; short forms are silently accepted
  but do nothing.
- Never trust the `applied: true` echo — verify by *behavior* (and for RAM
  cheats, by reading the byte back).
- Cheats are volatile: re-apply after every `state({op:'load'})` or reset.
- Screenshot the effect; note anything unexpected (glitches, side effects on
  other code paths that read the same byte).

A candidate that doesn't produce its predicted effect is a finding too —
record why and drop or revise it.

## Step 4 — Banking determines the code length

Game Genie patches by CPU address, so bank switching matters:

- Addresses in a **fixed** (always-mapped) region → a plain code with no
  compare value is safe (NES: 6-letter, for `$C000`–`$FFFF` on most mappers
  with a fixed upper bank).
- Addresses in a **switchable** region → you MUST use the compare-byte form
  (NES: 8-letter) with compare = the original byte, so the patch only lands
  when the intended bank is mapped in. Note that another bank with the same
  byte at that offset would also match — check for that if behavior is odd.

Work out the mapper's fixed/switchable split from the disassembly build
layout before encoding.

## Step 5 — Encode, then round-trip

NES letter alphabet (letter → nibble): `A=0 P=1 Z=2 L=3 G=4 I=5 T=6 Y=7
E=8 O=9 X=A U=B K=C S=D V=E N=F`. Canonical **decode** (letters n[0..5] or
n[0..7] as nibble values):

```python
address = 0x8000 + (((n[3] & 7) << 12)
        | ((n[5] & 7) << 8) | ((n[4] & 8) << 8)
        | ((n[2] & 7) << 4) | ((n[1] & 8) << 4)
        |  (n[4] & 7)       |  (n[3] & 8))
if six_letter:
    data = ((n[1] & 7) << 4) | ((n[0] & 8) << 4) | (n[0] & 7) | (n[5] & 8)
else:  # 8-letter; bit 3 of n[2] must be SET to mark it
    data    = ((n[1] & 7) << 4) | ((n[0] & 8) << 4) | (n[0] & 7) | (n[7] & 8)
    compare = ((n[7] & 7) << 4) | ((n[6] & 8) << 4) | (n[6] & 7) | (n[5] & 8)
```

Write the **encoder as the inverse of this decoder** in a scratchpad script,
and make the script assert the round trip: encode(addr, data[, compare]) →
decode → must reproduce the inputs exactly. Do not hand-derive bit shuffles.

Other platforms (SNES, Game Boy, Genesis) have Game Genie schemes with
different alphabets and scrambles — look the scheme up per platform; steps
1–3 and 6–7 are platform-independent.

## Step 6 — Prove the letter code itself

The encoder being self-consistent is not enough: apply the **letter** code
via `cheats({op:'apply', code:'XXXXXXXX'})` and ask a human to play and
re-verify the behavior. (Do not play the game yourself, that is too
token-inefficient.) The emulator core's own decoder is the independent
referee — if the letter code works live, the encoding is right.

## Step 7 — Publish

Write `notes/game_genie_codes.md` (or the repo's convention) with one entry
per code:

- The letter code, the raw `ADDR:NEW:OLD` form, and which banks/regions.
- A one-paragraph **mechanism** citing the routine/table names from the
  disassembly — the point of these codes is that they're explainable.
- Evidence: what was observed live, screenshots, save state used.
- Caveats: shared code paths, glitches, interactions with other codes.

Do not modify the canonical disassembly source for any of this — the whole
pipeline runs on volatile cheats, so no rebuild/byte-compare gate applies.
