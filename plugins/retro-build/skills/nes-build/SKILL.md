---
description: Build NES ROMs from 6502 assembly using ca65 + ld65 on macOS, and optionally launch the result in Mesen. Trigger on /retro-build:nes-build, or when the user asks to build/compile/assemble an NES ROM, run `make` in a clear NES project, or open a .nes file in Mesen. Do NOT trigger on generic 6502 assembly, .s files for Apple II / Commodore 64 / Atari / other targets, or ambiguous "build this assembly" requests — only when the project is unambiguously NES.
allowed-tools: Bash(ca65 *) Bash(ld65 *) Bash(make) Bash(make *) Bash(pgrep -x Mesen)
---

# nes-build

Builds NES ROMs using the `cc65` toolchain (`ca65` assembler, `ld65` linker) and, on request, launches the resulting ROM in Mesen.

## Step 1 — Confirm this is an NES project

Before running any build command, verify the current directory contains at least one strong NES signal:

- An existing `.nes` file.
- A `.cfg` file (any name) whose contents reference NES-shaped segments like `CHARS`, `VECTORS`, or memory regions at `$8000` / `$FFFA`.
- A `.s` file containing iNES header bytes (`"NES"` or `$4E, $45, $53, $1A`), PPU register addresses (`$2000`–`$2007`), or PPU register names (`PPUCTRL`, `PPUMASK`, `PPUSTATUS`, `PPUADDR`, `PPUDATA`).
- A `Makefile` that invokes both `ca65` and `ld65`.

If none of these are present, stop and tell the user: *"This doesn't look like an NES project — I expected an iNES header, PPU register references, or an NES-shaped .cfg."* Do **not** fall back to building it as generic 6502 assembly.

## Step 2 — Build

Once confirmed as NES, pick the right command based on what's in the directory:

- **Makefile present** → `make`.
- **No Makefile but `.s` + `.cfg` present** → before linking, run the iNES header check below; then `ca65 <name>.s -o <name>.o` and `ld65 -C <cfg> <name>.o -o <name>.nes`.
- **Only a `.s` file** (with NES signals from Step 1) → show the user the default NROM linker config and Makefile templates below (substituting the actual `.s` basename for `NAME`) and ask whether to write them. Only write `nes.cfg` and `Makefile` once the user confirms. Then run the iNES header check and `make`.
- **Multiple `.s` or `.cfg` files** with no Makefile → ask the user which to build rather than guessing.

On success, report the path to the produced `.nes`. **Do not launch Mesen automatically** — wait for the user to ask.

## Step 2a — iNES header check (run before every `ld65` call)

The iNES header declares PRG and CHR bank counts that must match the linker config's memory sizes. A mismatch produces a ROM that loads but misbehaves silently.

1. Locate the `.byte` line in the `HEADER` segment of the `.s` file. The 5th byte (index 4, after the 4-byte magic) is the PRG bank count (×16 KB = ×`$4000`); the 6th byte (index 5) is the CHR bank count (×8 KB = ×`$2000`).
2. In `nes.cfg`, read `PRG: size=…` and `CHR: size=…`.
3. Compute expected sizes: `prg_expected = prg_banks × $4000`, `chr_expected = chr_banks × $2000`.
4. If either doesn't match, surface a clear warning — e.g. *"Header declares 2 PRG banks = $8000, but nes.cfg PRG size is $4000 — the ROM will likely not run correctly."* — and ask the user whether to proceed before linking.

## Step 3 — Open in Mesen (only when asked)

When the user explicitly asks to open / run / test / play the ROM in Mesen, first check whether Mesen is already running:

```bash
if pgrep -x Mesen >/dev/null; then
    echo "Mesen is already running — press F11 (Power Cycle) or use File → Reload ROM to load the new build."
else
    /Applications/Mesen.app/Contents/MacOS/Mesen <rom>.nes &
fi
```

The direct binary path matters — `open -a Mesen <rom>` does **not** pass the file argument through to Mesen on this system (Mesen opens to the recent-games list instead of loading the ROM).

If `/Applications/Mesen.app` is missing, run `find /Applications -maxdepth 3 -name "Mesen*"` and use whatever's there, or surface the error.

## Toolchain notes

- `ca65` and `ld65` come from the `cc65` package: `brew install cc65`.
- If `ca65` or `ld65` is not on PATH, stop and surface the install command. Don't try to build.
- On assembler/linker failure, surface stderr verbatim — don't summarize errors away. Line/column references from `ca65` are precise and useful.

## Default NROM linker config

Use this as `nes.cfg` for the Step 2 fallback (2×16 KB PRG = 32 KB, 1×8 KB CHR, mapper 0):

```
MEMORY {
    ZP:   file="",  start=$0000, size=$0100, type=rw;
    RAM:  file="",  start=$0200, size=$0600, type=rw;
    HDR:  file=%O,  start=$0000, size=$0010, fill=yes, fillval=$00;
    PRG:  file=%O,  start=$8000, size=$8000, fill=yes, fillval=$FF;
    CHR:  file=%O,  start=$0000, size=$2000, fill=yes, fillval=$00;
}
SEGMENTS {
    ZEROPAGE: load=ZP,  type=zp;
    BSS:      load=RAM, type=bss;
    HEADER:   load=HDR, type=ro;
    CODE:     load=PRG, type=ro, start=$8000;
    RODATA:   load=PRG, type=ro;
    VECTORS:  load=PRG, type=ro, start=$FFFA;
    CHARS:    load=CHR, type=ro;
}
```

## Default Makefile

Use this as `Makefile` for the Step 2 fallback. Replace `hello` with the detected `.s` basename:

```make
NAME = hello
MESEN = /Applications/Mesen.app/Contents/MacOS/Mesen

$(NAME).nes: $(NAME).o nes.cfg
	ld65 -C nes.cfg $(NAME).o -o $@

$(NAME).o: $(NAME).s
	ca65 $< -o $@

run: $(NAME).nes
	$(MESEN) $< &

clean:
	rm -f $(NAME).o $(NAME).nes

.PHONY: run clean
```

Note: the indentation in recipe lines must be a **tab**, not spaces — `make` will error otherwise.
