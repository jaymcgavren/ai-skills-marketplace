---
description: Locate the RAM address that holds a game value using the romdev MCP's Cheat-Engine-style search loop — seed a search on the value, change it in-game, narrow with searchNext, confirm by writing the byte. Works for HUD numbers (lives, score, HP), values you can only push in a known direction (player position), and values you can't read at all (snapshot/diff fallback). Trigger on /rom-debug:locate-value, or when the user asks "which RAM address holds X", "find the lives/score/HP/power byte", "where is the player position stored", or wants to pin any observable value to an address.
allowed-tools: Read Write Edit AskUserQuestion mcp__romdev__memory mcp__romdev__loadMedia mcp__romdev__state mcp__romdev__frame mcp__romdev__input mcp__romdev__playtest mcp__romdev__breakpoint mcp__romdev__disasm mcp__romdev__catalog mcp__romdev__platform
---

# locate-value

Pin any observable game value to its RAM address in 2–3 narrowing rounds — no
code reading, no full-RAM diff spelunking. The loop is the classic Cheat Engine
technique: **search for the value → change it in-game → keep only the addresses
that changed the way you expected → repeat**. Each round typically cuts
hundreds of candidates down by 10–100×, so two or three rounds usually leave
exactly one byte.

Prerequisites: the ROM is loaded (`loadMedia`) and the game is at a moment
where the value exists (in gameplay for lives/HP, on the right screen for a
menu value). A save state of that moment makes retries cheap —
`state(op:'save')` before you start — but nothing here spans sessions; this is
a one-sitting technique.

For a well-known commercial ROM, try `cheats(op:'lookup')` *first* — it's a
crowd-sourced labeled RAM map, and the address you want may already be in it.

(Tool names here use the `mcp__romdev__` prefix, which assumes the server was
registered as `romdev`. If yours is registered under another name, match the
prefix — in `allowed-tools` and when calling — to that name.)

## Pick the approach: diffRuns vs. the search loop

- **If an input drives the value (position, charge, a held-button effect),
  lead with `diffRuns`** — it's a one-call answer:

  ```python
  memory(op="diffRuns", region="system_ram", frames=30, portsA=[{"right": True}])
  ```

  It runs the *same* start state twice (A = input held, B = idle) and returns
  only the bytes that DIVERGED — the position byte, its OAM shadow, the pad
  state, and nothing else. `minDelta:N` filters small churn.

- **For everything you trigger rather than hold** (HUD numbers, pickups,
  scripted events) use the search loop below.

## The core loop

1. **Seed** the search with the value's current number:

   ```python
   memory(op="search", value=3, size=1)        # e.g. 3 lives shown
   ```

   `region` defaults to `system_ram`. `size:1` for stats/lives/levels, `2` for
   scores/timers, `4` for big counters. The response's `count` is the true
   candidate total (`maxCandidates` only caps how many are *listed*); the full
   set is kept server-side.

2. **Change the value in-game.** Use `input(op:'set')` to hold the input for
   *sustained* movement and `frame(op="step", frames=...)` to advance; `press`
   works for one-shot inputs (it always emits a released→pressed edge). Take a
   hit, score points, grab a pickup, move. For changes that are hard to trigger
   headlessly (die to a specific enemy, reach a boss), open `playtest(...)` and
   ask the human to do it, then `host(op:'pause')`.

3. **Narrow:**

   ```python
   memory(op="searchNext", compare="eq", value=2)   # lives now show 2
   ```

   Repeat steps 2–3 until `count` is ~1–3.

The `name` parameter keys the search session — pass the same `name` to keep
narrowing, or different names to run independent searches side by side.

## Choosing `compare` — you don't need to *see* the value

- **Exact new value known** (HUD numbers): `compare:'eq', value:<new>`.
- **Only the direction known** (player X position, a hidden timer, charge
  level): `'inc'` / `'dec'`. These compare each candidate against its own
  last-read value, so no number is needed — move right, `searchNext
  compare:'inc'`; move left, `compare:'dec'`. Two or three alternations
  isolate a position byte fast.
- **Direction unknown too:** alternate `'changed'` right after the action with
  `'unchanged'` after deliberately doing *nothing* for a few frames. The
  do-nothing rounds are what kill the frame counters and other always-churning
  bytes.
- `'gt'` / `'lt'` take a `value` and keep candidates now above/below it —
  useful to drop zeroed bytes (`compare:'gt', value:0`).

`op:'search'` baselines every candidate at seed time, so the relative compares
(`'inc'`/`'dec'`/`'changed'`/`'unchanged'`) work as the *first* `searchNext` —
no warm-up `eq` round needed. Direction-only flow is just: seed → move right →
`'inc'` → move left → `'dec'`.

**The seed still needs a number** even for non-HUD values — read it out of the
machine instead of the screen. For positions, `sprites({op:'inspect'})` (or
`op:'group'` to find the player cluster first) returns every hardware sprite's
exact x/y: seed the search with the metasprite's anchor or center coordinate.
A near-miss seed just means an empty first round — re-seed with the next
plausible number (left edge, center, nose sprite); each attempt costs one call.

## When you can't read the value at all

`op:'search'` requires a seed `value`, so for "I have no idea what number it
is" cases use the snapshot workflow instead:

```python
memory(op="snapshot", region="system_ram", name="before")
# ... trigger the event ...
memory(op="diff", name="before")     # default summary view
```

The summary view clusters the changed bytes and detects strides ("4 islands at
stride 0x80" = a struct array), so one diff is readable instead of a flood.
`minDelta:N` drops bytes that barely moved — the tool-level version of the
"do nothing for a few frames" counter-killer, and free. Once a cluster looks
promising, switch to the search loop on it — read a byte in the cluster, seed
`op:'search'` with that number, and narrow as above.

## When the search finds nothing (representation gotchas)

The stored byte often isn't the displayed number. Before giving up:

- **Stored ≠ displayed (game logic):** lives are frequently stored as
  displayed−1 (the counter of *spares*); scores are often stored ÷10 when the
  last digit is always 0. These are game-logic transforms, not encodings —
  seed with the transformed value.
- **Packed BCD / digit-per-byte (encoding):** don't decode these by hand — pass
  the representation to `op:'search'`. `as:'bcd'` matches packed BCD (2 decimal
  digits per byte, the classic NES score); `as:'digits'` matches one byte per
  on-screen digit at ANY constant tile base (raw 0, ASCII $30, or the game's
  font base — auto-detected per candidate and reported as `digitBase`).
  `searchNext` keeps comparing in the seed's representation, including numeric
  `inc`/`dec` on the decoded value.
- **Endianness/width:** multi-byte values are little-endian on these CPUs; if
  `size:1` rounds keep emptying the candidate list, retry the whole loop with
  `size:2`.
- **Some values defeat every representation** (split across non-adjacent bytes,
  tile-index display buffers, checkpoint copies). Don't keep guessing: switch to
  the snapshot/diff workflow with a *tight* window — pause, trigger exactly one
  change event, diff — and decode whatever cluster moves.

## Confirm the hit

A surviving candidate is a hypothesis; writing to it is the proof:

```python
memory(op="write", region="system_ram", offset=0x067A, hex="09")
frame(op="screenshot")
```

If the HUD now shows 9 (or the player jumps to the forced position, etc.),
that's the byte. If nothing changes, the candidate is a mirror or a display
copy — test the next one. For values without a visible readout, confirm by
behavior (set HP to 1, take a hit, die).

## Follow-ups

The address is usually a means, not the end. `breakpoint(on:'write')` on it,
repeat the in-game change, and the hit's `pc` is the exact instruction that
owns the value — `disasm` around it to read the handler. On every platform the
hit response carries `registersAtHit` (the register file frozen at the hit
instant) and the CPU stays frozen until you clear the hit — read those
registers straight from the response, never a follow-up `cpu(op:'read')`. For a
write hit, `valueByte` is the one byte that actually landed (not the operand).
Annotate the disassembly and record the address in the project's RAM map doc if
it has one. Watchpoints are core-level and keyed to the RAM byte, so bank
switching can't fake them.

## Gotchas (romdev)

- **The playtest window only fights you while a human is actively pressing**
  (~2s). `frame`/`input` responses carry a `humanCoDriveWarning`, and
  `playtest({op:'status'})` / `catalog({op:'status'})` expose `humanInputActive`,
  so you'll know. You can leave the window open and `host({op:'pause'})` for
  frozen-deterministic probing, use a second session (different
  `x-romdev-session` header) for full isolation, or close it with
  `playtest({op:'stop'})`.
- **Hold input with `input(op:'set')` and account for latency** — step several
  frames before assuming the action registered; sampling on frame 1 reads the
  pre-action value and poisons a `'changed'`/`'inc'` round.
- **`state(op:'load')` rewrites all of RAM.** Mid-narrow it silently turns
  every byte into a `'changed'` candidate-killer or -keeper you didn't intend.
  Used *deliberately* it's fine — reloading a state is a legitimate "the value
  went back up" event for an `'inc'` round.
- **`op:'write'` takes `hex` or `base64` only** — `data`, `bytes`, and arrays
  are rejected.
- **Pause between rounds.** A running emulator keeps mutating RAM between your
  action and the `searchNext`; `host({op:'pause'})` (or stepping exact frame
  counts) keeps each round's before/after honest.
- **Narrow in short bursts and screenshot when a round goes to 0.** If the
  player dies or the scene changes mid-step, the value vanishes and *every*
  candidate fails the round (verified live: the ship was exploding during a
  20-frame "move right" step — `'inc'` returned 0 across the board). Save a
  state before narrowing, step 4–8 frames per round, and on a surprise empty
  round look at the screen before blaming your compare op (the server's own
  0-candidates note now flags this too).
- **A candidate that matches one round can still be a coincidence.** Two bytes
  matched "score÷10" once and were exposed on the very next `eq` round. Never
  trust a single round; the loop converging twice is the signal.
- `platform({op:'doc'})` gives the platform's memory map (where work RAM
  actually lives, what's mirrored) before you over-interpret a candidate
  address.

## Report

When done, state: the value, the confirmed address (and representation — raw,
−1, BCD, digit-per-byte), how it was confirmed, and any companion addresses
found on the way (mirrors, display copies), so they can be recorded in the
project's RAM map.
