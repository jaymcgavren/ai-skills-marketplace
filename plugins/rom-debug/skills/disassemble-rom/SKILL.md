---
description: Turn a bare retro-console ROM into a rebuildable, byte-exact disassembly project using the romdev MCP — one call disassembles every bank, a second call reassembles it, and a byte-compare against the original proves the source is trustworthy before any annotation starts. Scaffolds the repo (canonical src/, CLAUDE.md build rule, RAM_MAP.md, TODO.md backlog) so later sessions can comment the code incrementally. Works across romdev's classic platforms (NES, SNES, Game Boy, Genesis, …) via one universal rebuild path. Trigger on /rom-debug:disassemble-rom, or when the user says "disassemble this ROM", "set up a disassembly project", "I have a ROM and want commented source", or hands you a .nes/.sfc/.gb/.md/etc. file with no source.
allowed-tools: Read Write Edit Bash AskUserQuestion mcp__romdev__disasm mcp__romdev__build mcp__romdev__symbols mcp__romdev__platform mcp__romdev__catalog
---

# disassemble-rom

Go from a bare ROM (`game.nes`, `game.sfc`, `game.gb`, `game.md`, …) to a git
repo containing source that **provably reassembles byte-for-byte into the
original ROM**. That proof is the whole point: annotation work (renaming
labels, commenting routines) is only safe when an unmodified rebuild is
byte-identical, because then any future diff is attributable to your edit and
nothing else.

This is a one-session skill. It produces the *skeleton* — structurally
complete, semantically empty source. Understanding what the code *does* is the
follow-on skill (`annotate-disassembly`), which runs across many sessions.

(Tool names use the `mcp__romdev__` prefix, which assumes the server was
registered as `romdev`. If yours is registered under another name, match the
prefix — in `allowed-tools` and when calling — to that name.)

## Works on any romdev classic platform

The flow below is platform-parameterized. One universal rebuild call —
`build({output:'reassemble'})` — rebuilds the project on every classic
platform, so the same six steps work regardless of console. Fill in your
platform's row and use it throughout:

| Platform | ROM ext          | CPU        | Vector table to seed | Notes / readability |
|----------|------------------|------------|----------------------|---------------------|
| NES      | `.nes`           | 6502       | reset/NMI/IRQ @ `$FFFA` | 6502 disassembles cleanly — high readability, real instructions. **Validated.** |
| SNES     | `.sfc` / `.smc`  | 65816      | native+emulation @ `$FFE0–$FFFF` | 65816 usually lands on the **data-only floor** (all `.byte`, ~0% readable) — expected, not a failure. See the readability note in Step 2. **Validated.** |
| Game Boy | `.gb` / `.gbc`   | LR35902    | rst/interrupt vectors `$0000–$0067` | Should disassemble to real instructions. **Untested here — prove it in Step 3 before trusting.** |
| Genesis  | `.md` / `.gen`   | m68k (BE)  | 68k vector table @ `$000000` | Big-endian; auto-detected. **Untested here — prove it in Step 3 before trusting.** |

For anything not in the table, run `platform(op="list", platform=<p>)` for its
quirks and treat it like an untested row: the universal rebuild path still
applies, but the Step 3 byte-compare is what earns your trust in it.

## The hard rule

> **Never commit source that doesn't rebuild byte-identical to the original
> ROM.** Verify with `cmp` after every build. An unmodified checkout must
> always rebuild identical — check this before starting any work session, so
> a later mismatch is attributable to that session's edits.

Labels and comments are free (they don't change emitted bytes). Anything else
— reordering, "cleanup", changing a `.byte` to an instruction — must be proven
by the byte-compare.

> **Never commit ROM bytes.** The original ROM (and `src/original.rom`, the
> verbatim copy the rebuild splices into) are copyrighted game data — they
> must not enter git history, ever. They stay on disk, git-ignored. See
> Step 4. If you ever find you committed ROM bytes, amend/purge them out
> (`git rm --cached`, then expire the reflog and `git gc --prune=now`).

## Step 1 — Orient

```python
platform(op="list", platform="<platform>")   # quirks + toolchain for this platform
symbols(op="analyze", romPath="/abs/path/game.<ext>")
```

`symbols(op:'analyze')` gives a structural map with no setup: arch,
entrypoints, auto-detected functions, strings. Skim it to learn the ROM's
shape — arch (confirm it matches your table row), how many banks, where the
big code masses are — before generating anything. On a large ROM the analyze
result can be big; query it with `jq` for `{arch, functionCount, entrypoints}`
and the top functions rather than reading it whole. Keep the original ROM
somewhere permanent — it's the reference forever.

## Step 2 — Disassemble the whole ROM as a project

```python
disasm(target="project", path="/abs/path/game.<ext>", outputDir="/abs/path/repo/src")
```

One call, all banks. It writes:

- `bankN.asm` — one file per bank (or per region on non-banked platforms),
  disassembled with data/code separation. This is the source you'll annotate.
- `*_seg.asm` and a platform rebuild `.cfg` — build glue extracted from the
  ROM. **Do not hand-edit these**; they exist only to make the rebuild exact.
- `reassemble.json` — the region→offset manifest the universal rebuild reads.
  (The cc65-native family — NES/C64/7800/Lynx/PCE — *also* gets a `rebuild.json`
  for an optional one-call `build({output:'rom'})`; SNES/GB/Genesis do not.)
- `original.rom` — a **verbatim copy of the source ROM**, the template the
  rebuild splices assembled regions into. This is ROM data → git-ignored in
  Step 4, never committed.
- `BUILD.md` — human-readable rebuild recipe. **Always present; it is the
  authoritative rebuild instructions for this platform** — read it in Step 3.

**Large ROMs — run it in the background.** For a ROM more than ~512 KB, start
with the detached form so a long reassembly can't time out the call:

```python
disasm(target="project", path="/abs/path/game.<ext>", outputDir="/abs/path/repo/src",
       background=True)                                  # returns {jobId} immediately
disasm(target="project", job="<jobId>", outputDir="/abs/path/repo/src")   # poll
```

The poll reports `regionsDone/regionsTotal` while running, then returns the same
completion payload the sync call would. (Banks reassemble concurrently, capped
by `ROM_DEV_WASM_POOL_SIZE`, default 2.) The sync form above stays the default
for normal-size ROMs.

**Reading the response.** Skim it: `roundTrip.allByteExact` should be `true`,
and `readablePercent` per region reports instructions vs. raw `.byte`. Treat
this as advisory — **Step 3's own `cmp` is the authoritative proof.**

**Interpreting `readablePercent` (CPU-dependent — don't over-read it):**

- 6502 / LR35902 / m68k tend to come out as real instructions; a low number
  there usually means a genuine data region (graphics, level maps, music).
- **65816 (SNES) commonly comes out ~0% across every code bank** — the
  disassembler's `.a8/.i8` width state desyncs where code and pinned `.byte`
  mix, so it falls back to the byte-exact **data-only floor** (everything as
  `.byte`). This still rebuilds byte-identical; it just means readability has
  to be earned address-by-address later. It is **not** a failure.
- **Uniform-fill regions are flagged, not scored.** A bank that's all `$FF` (or
  `$00`) padding reports `readablePercent: null` + `fill: true` + `fillByte`, is
  excluded from `readablePercentAvg`, and is counted in the payload's
  `fillRegions`. Key off `fill === true`; treat `readablePercent: null` as "not
  code," never as a percentage.

**Gotcha:** the rebuild manifest contains *absolute paths*. If you move or
rename the project directory, regenerate it (rerun `disasm(target='project')`
into the new location) — don't let a stale manifest rot.

## Step 3 — Prove the round trip yourself

Trust, but verify with your own rebuild + compare. Read `BUILD.md` (the
authoritative recipe for this platform), then make the **universal rebuild
call** — it reads the project dir, assembles each region, and splices the
result into a copy of `original.rom`, so the header/gaps/pad return verbatim:

```python
build(output="reassemble", platform="<platform>", path="/abs/path/repo/src",
      outputPath="/abs/path/repo/build/rebuilt.<ext>")
```

(Create `build/` first — the tool won't `mkdir` it. The response reports
`byteExact` overall and per region.) On the cc65-native family only, `BUILD.md`
may offer a `build({output:'rom'}, sourcesPaths/includePaths/linkerConfigPath)`
alternative from `rebuild.json` — either proves the round trip; `reassemble`
is the one that works everywhere.

Then do the byte-compare yourself — this is the real proof:

```bash
cmp game.<ext> build/rebuilt.<ext> && echo BYTE-IDENTICAL
shasum -a 256 game.<ext> build/rebuilt.<ext>   # optional: matching hashes
```

If `cmp` is silent and echoes BYTE-IDENTICAL, the project is real. If not,
stop — do not scaffold a repo around a broken round trip; report what
differs (`cmp -l | head` shows offsets).

## Step 4 — Scaffold the repo

```
repo/
  game.<ext>           # the reference ROM — git-IGNORED, stays local (never committed)
  src/                 # canonical disassembly — stays byte-exact forever
    original.rom       # verbatim ROM copy the rebuild splices into — git-IGNORED
  build/               # git-ignored build output
  hack-src/            # git-ignored scratch copy of src/ for experiments
  CLAUDE.md            # rebuild command + the hard rules
  RAM_MAP.md           # starts nearly empty; grows with every finding
  TODO.md              # annotation backlog (checkbox list)
```

- `git init` (if needed); write `.gitignore` **before** the first `git add`,
  and make it cover both build scratch *and all ROM data*:
  `build/`, `hack-src/`, `*.o`, `*.state`, the ROM extension(s) for this
  platform (`*.sfc`, `*.gb`, …), and `*.rom` (catches `src/original.rom`).
  Do not add comments to `.gitignore` unless the repo's existing one uses
  them. After scaffolding, verify nothing ROM-shaped is staged:
  `git ls-files | grep -Ei '\.(rom|nes|sfc|smc|gb|gbc|md|gen|bin)$'` must
  print nothing.
- **The rebuild needs `src/original.rom` locally.** Because it's git-ignored,
  a fresh checkout can't rebuild until the user drops their own legally-obtained
  ROM back in as `src/original.rom` (or reruns `disasm(target='project')`).
  Say this in CLAUDE.md — it's the one thing a clone can't do on its own.
- **CLAUDE.md** must record: the exact rebuild command (the
  `build({output:'reassemble'})` call for this platform), both hard rules
  (byte-exact — "compile and `cmp` before every commit, never commit a
  mismatch"; and never-commit-ROM-bytes), the `hack-src/` convention
  (experimental edits happen in a copy, never in canonical `src/`), and the
  "restore `src/original.rom` to rebuild" note above. Record the original
  ROM's SHA-256 here as the identity check, since the ROM itself isn't in git.
- **RAM_MAP.md** starts as a table header plus the platform's standard regions
  (from `platform(op:'list')` quirks or the platform doc — e.g. NES `$0200`
  OAM shadow + PPU/APU registers; SNES `$2100–$21FF` PPU + `$4200–$43FF` CPU/DMA
  + `$7E0000` WRAM; GB `$FF00` I/O). Every address discovered later gets a row.
  This file is the program's "variable names" — nearly every instruction reads
  or writes RAM, so each mapped address makes every routine touching it
  partially legible.
- **TODO.md** is the handoff to future annotation sessions. Seed it now
  (step 5) so no session ever starts with "what should I work on?".

## Step 5 — Seed the annotation backlog

Cheap, high-value structural facts to gather while everything is fresh:

1. **Vectors**: seed a task to trace each entry point for this platform's
   vector table (your table row names it) — reset/boot init, and the per-frame
   interrupt handler (NES/SNES NMI, GB VBlank, Genesis VInt) which is the
   heartbeat. On the data-only floor these currently sit in `.byte` form; the
   task is to carve and name them.
2. **Function map**: `disasm(target="functions", path=...)` (or the
   `functions` array from Step 1's analyze) lists auto-detected functions with
   size and caller counts. The biggest and most-called ones are the main loop,
   the object/entity dispatcher, the sound engine — add the top handful as
   TODO items with their addresses.
3. **Data vs. code**: on CPUs that disassemble cleanly, add "- [ ] Identify
   contents of bankN (graphics? music? level data?)" for low-`readablePercent`
   banks (skip `fill: true` regions — those are padding, not data).
   **On the 65816 data-only floor this heuristic doesn't work** — every
   bank is `.byte`, so instead seed one task to establish the carve-from-`.byte`
   workflow (pick a known code address → `disasm(target='rom', startAddress,
   endAddress)` or the live debugger for real instructions → convert that range
   in `hack-src/` → rebuild + `cmp` → promote to `src/`) and one to sort banks
   into code / graphics / audio / level data.
4. Player-facing targets that annotation sessions will want: "- [ ] Find the
   lives/score/HP RAM addresses" (the `locate-value` skill does this), "- [ ]
   Map controller reading to actions" (the `drive-input` skill).

Do **not** write behavioral comments yet — at this stage any claim about what
a routine does is a guess, and wrong comments are worse than none. The
annotate-disassembly skill establishes facts with breakpoints and live
evidence.

## Step 6 — Commit

Rebuild once more from the final `src/`, `cmp`, then make the initial commit of
`src/`, the scaffolding files (CLAUDE.md, RAM_MAP.md, TODO.md), and
`.gitignore` — **but not the ROM or `src/original.rom`** (Step 4's grep must
still come back empty). State in the commit message that the rebuild was
verified byte-identical, and include the ROM's SHA-256.
