# ai-skills-marketplace

Jay McGavren's personal collection of LLM skill plugins.

## Install in Claude Code

Add the marketplace:

```
/plugin marketplace add jaymcgavren/ai-skills-marketplace
```

Then install individual plugins:

```
/plugin install retro-build@ai-skills-marketplace
/plugin install todo-sh@ai-skills-marketplace
```

For local testing, use the repo path instead:

```
/plugin marketplace add ./path/to/ai-skills-marketplace
```

## Plugins

### retro-build

Build NES ROMs from 6502 assembly using the `cc65` toolchain (`ca65` + `ld65`) on macOS, and optionally launch the result in Mesen.

**Prerequisites:** `brew install cc65` and [Mesen](https://www.mesen.ca) installed at `/Applications/Mesen.app`.

**Skills:**

- `/retro-build:nes-build` — confirm the project is NES, build the ROM, and optionally open it in Mesen.

### todo-sh

Manage your personal to-do list via the `todo.sh` CLI.

**Prerequisites:** `brew install todo-txt`

**Skills:**

- `/todo-sh:list` — run `todo.sh ls` and return the output verbatim.

## Releasing

Both plugins pin their version in `plugin.json`. Bump the `version` field in the relevant `plugin.json` before tagging a release, or users won't receive the update.
