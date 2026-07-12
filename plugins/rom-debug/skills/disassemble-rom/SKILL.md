---
description: Turn a bare retro-console ROM into a rebuildable, byte-exact disassembly project using the romdev MCP — one call disassembles every bank, a second call reassembles it, and a byte-compare against the original proves the source is trustworthy before any annotation starts. Scaffolds the repo (canonical src/, CLAUDE.md build rule, RAM_MAP.md, TODO.md backlog) so later sessions can comment the code incrementally. Trigger on /rom-debug:disassemble-rom, or when the user says "disassemble this ROM", "set up a disassembly project", "I have a ROM and want commented source", or hands you a .nes/.gb/.sms/etc. file with no source.
allowed-tools: Read Write Edit Bash AskUserQuestion mcp__romdev__disasm mcp__romdev__build mcp__romdev__symbols mcp__romdev__platform mcp__romdev__catalog
---

# disassemble-rom

Go from `game.nes` (or any romdev-supported ROM) to a git repo containing
source that **provably reassembles byte-for-byte into the original ROM**. That
proof is the whole point: annotation work (renaming labels, commenting
routines) is only safe when an unmodified rebuild is byte-identical, because
then any future diff is attributable to your edit and nothing else.

This is a one-session skill. It produces the *skeleton* — structurally
complete, semantically empty source. Understanding what the code *does* is the
follow-on skill (`annotate-disassembly`), which runs across many sessions.

(Tool names use the `mcp__romdev__` prefix, which assumes the server was
registered as `romdev`. If yours is registered under another name, match the
prefix — in `allowed-tools` and when calling — to that name.)

## The hard rule

> **Never commit source that doesn't rebuild byte-identical to the original
> ROM.** Verify with `cmp` after every build. An unmodified checkout must
> always rebuild identical — check this before starting any work session, so
> a later mismatch is attributable to that session's edits.

Labels and comments are free (they don't change emitted bytes). Anything else
— reordering, "cleanup", changing a `.byte` to an instruction — must be proven
by the byte-compare.

## Step 1 — Orient

```python
platform(op="list", platform="nes")       # quirks + toolchain for this platform
symbols(op="analyze", romPath="/abs/path/game.nes")
```

`symbols(op:'analyze')` gives a structural map with no setup: entrypoints,
auto-detected functions, strings. Skim it to learn the ROM's shape (how many
banks, where the big code masses are) before generating anything. Keep the
original ROM somewhere permanent — it's the reference forever.

## Step 2 — Disassemble the whole ROM as a project

```python
disasm(target="project", path="/abs/path/game.nes", outputDir="/abs/path/repo/src")
```

One call, all banks. It writes:

- `bankN.asm` — one file per PRG bank (or per region on non-banked
  platforms), disassembled with data/code separation. This is the source
  you'll annotate.
- `*_seg.asm`, `nes_header.asm`, `nes_rebuild.cfg` — build glue extracted
  from the ROM. **Do not hand-edit these**; they exist only to make the
  rebuild exact.
- `rebuild.json` — the exact `build()` arguments for a one-call rebuild
  (NES/C64/7800/Lynx/PCE; other platforms get a proven native recipe in
  `BUILD.md` instead).
- `BUILD.md` — human-readable rebuild instructions.

Check the response before proceeding:

- `roundTrip.allByteExact` must be `true`. The tool already reassembled each
  region and verified it; if any region failed it fell back to `.byte` lines,
  so this should essentially always pass — treat `false` as a stop-and-report.
- `readablePercent` per region tells you how much came out as instructions
  vs. raw `.byte` data. 100% is common on pure-code banks; a low number
  usually means a data bank (graphics, level maps, music), which is fine.

**Gotcha:** `rebuild.json` contains *absolute paths*. If you move or rename
the project directory, regenerate it (rerun `disasm(target='project')` into
the new location) or fix the paths — don't let a stale rebuild.json rot.

## Step 3 — Prove the round trip yourself

Trust, but verify with your own build + compare. Read `rebuild.json` and make
the `build()` call it specifies (it uses `sourcesPaths` + `includePaths` +
`linkerConfigPath`, so the linker config never enters your context), with an
`outputPath` you choose:

```python
build(output="rom", platform="nes",
      sourcesPaths={...from rebuild.json...},
      includePaths={...from rebuild.json...},
      linkerConfigPath=".../nes_rebuild.cfg",
      outputPath="/abs/path/build/rebuilt.nes")
```

```bash
cmp game.nes build/rebuilt.nes && echo BYTE-IDENTICAL
```

If `cmp` is silent and echoes BYTE-IDENTICAL, the project is real. If not,
stop — do not scaffold a repo around a broken round trip; report what
differs (`cmp -l | head` shows offsets).

## Step 4 — Scaffold the repo

```
repo/
  game_original.nes    # the reference ROM (or note its location if it can't be committed)
  src/                 # canonical disassembly — stays byte-exact forever
  build/               # git-ignored build output
  hack-src/            # git-ignored scratch copy of src/ for experiments
  CLAUDE.md            # build command + the hard rule
  RAM_MAP.md           # starts nearly empty; grows with every finding
  TODO.md              # annotation backlog (checkbox list)
```

- `git init`; `.gitignore` gets `build/`, `hack-src/`, `*.o`, `*.state`.
- **CLAUDE.md** must record: the exact rebuild command (point at
  `src/rebuild.json`), the byte-exact hard rule ("compile and `cmp` before
  every commit; never commit a mismatch"), and the `hack-src/` convention
  (experimental edits happen in a copy, never in canonical `src/`).
- **RAM_MAP.md** starts as a table header plus whatever is already known
  (e.g. platform-standard regions like NES `$0200` OAM shadow, hardware
  registers). Every address discovered later gets a row. This file is the
  program's "variable names" — nearly every instruction reads or writes RAM,
  so each mapped address makes every routine touching it partially legible.
- **TODO.md** is the handoff to future annotation sessions. Seed it now
  (step 5) so no session ever starts with "what should I work on?".

## Step 5 — Seed the annotation backlog

Cheap, high-value structural facts to gather while everything is fresh:

1. **Vectors**: the ROM disassembly auto-labels reset/NMI/IRQ. Add TODO items
   to trace each ("- [ ] Trace reset init sequence", "- [ ] Document the NMI
   frame handler — it's the per-frame heartbeat").
2. **Function map**: `disasm(target="functions", path=...)` lists
   auto-detected functions with size, complexity, and caller counts. The
   biggest and most-called ones are the main loop, the object/entity
   dispatcher, the sound engine — add the top handful as TODO items.
3. **Obvious data**: banks with low `readablePercent` are probably assets;
   add "- [ ] Identify contents of bankN (graphics? music? level data?)".
4. Player-facing targets that annotation sessions will want: "- [ ] Find the
   lives/score/HP RAM addresses" (the `locate-value` skill does this), "- [ ]
   Map controller reading to actions" (the `drive-input` skill).

Do **not** write behavioral comments yet — at this stage any claim about what
a routine does is a guess, and wrong comments are worse than none. The
annotate-disassembly skill establishes facts with breakpoints and live
evidence.

## Step 6 — Commit

Rebuild once more from the final `src/`, `cmp`, then make the initial commit:
ROM reference, `src/`, scaffolding files. State in the commit message that the
rebuild was verified byte-identical.
