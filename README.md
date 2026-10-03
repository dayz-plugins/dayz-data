# dayz-data

Memory addresses and struct offsets for DayZ, as data rather than code, so that a game update
costs a file in this repository instead of a new release of every plugin.

Read by [dayz-plugin-loader](https://github.com/dayz-plugins/dayz-plugin-loader), which
resolves the names at startup and hands plugins addresses by name. A plugin never contains an
address of its own.

## The two halves

**`patterns.json` is the source of truth.** It holds a byte signature per symbol, independent
of any build. When the running executable is not one this repository knows, the loader scans
these signatures against the mapped image and usually finds everything anyway. That is the
point of storing patterns rather than addresses: most game updates need no human at all.

**`builds/<version>.json` is a cache.** One file per executable, keyed by its SHA-256, holding
the addresses already resolved for it. Each code address carries a short `check`: the bytes
expected there. If the bytes do not match, the loader discards the entry and scans instead, so
a stale file reports a missing symbol rather than aiming a hook at the wrong instruction.

A build file is also matched on `pe_timestamp` together with `image_size`, which is what
identifies a build when the executable on disk is not the one in memory, or when an entry came
from somewhere that never recorded a hash. A hash match is preferred and the loader logs which
key matched.

```
patterns.json              signatures, the source of truth
builds/1.29.163709.json    resolved addresses for one executable hash
seeds/1.29.163709.json     the addresses a person established, input to the generator
schema/*.json              JSON Schema for both file kinds
```

## Names

`subsystem.thing`, lower case, matching how the research notes are organised: `render.*`,
`camera.*`, `gui.*`, `input.*`, `engine.*`. Struct field offsets live in a separate table
because they are a different kind of thing and resolve differently.

A symbol's `kind` is `function`, `global` or `instruction`. Globals get an address but never a
`check` or a pattern: their bytes are data the game writes at runtime, so a signature over
them would identify nothing and a check over them would fail on every launch.

## Adding a build

1. Write a seed file under `seeds/` listing the addresses you have established, as
   `"0x8E77C0"`, each with a `kind` and a note saying what it is.
2. Run the generator from the loader repository. It hashes the executable, lays its sections
   out the way Windows maps them, records the build metadata, takes a byte check at every
   address and derives a candidate pattern:

   ```bash
   cargo run -p dayz-data-tool -- generate /path/to/dayz-data \
       --exe "/path/to/DayZ/DayZ_x64.exe" \
       --seed /path/to/dayz-data/seeds/<version>.json \
       --date "$(date -I)"
   ```

3. Check the result:

   ```bash
   cargo run -p dayz-data-tool -- validate /path/to/dayz-data --exe "/path/to/DayZ/DayZ_x64.exe"
   ```

`validate` also cross-checks the two halves against each other: it resolves once from the
cache and once from the patterns alone, and reports any symbol where the two disagree.
`generate` never overwrites a pattern that already exists, so that a hand-wildcarded signature
survives regeneration — which means a seed address that moves leaves the old pattern behind,
and this check is what catches it.

**Generated patterns are literal byte runs.** They are the shortest run at the address that
occurs exactly once in the image, which makes them correct for the build they came from and a
coin flip for the next one, because a run may contain a call target or a displacement that
moves. Replacing those bytes with `??` by hand is what turns a candidate into a durable
signature, and it is the one part of this that a tool cannot do.

## What the loader does at runtime

1. Hashes `DayZ_x64.exe` and looks for a matching build file.
2. Verifies every cached address against its check.
3. Scans `patterns.json` for anything the cache did not cover.
4. Logs one line per symbol saying whether it was cached, scanned or unresolved.
5. For an unknown executable, writes what it found into the game's own data directory as a
   candidate, marked `"provenance": "scan"`, ready to be reviewed and contributed back here.

Plugins can declare the symbols they cannot work without. A missing one keeps that plugin
unloaded with its name in the log, instead of letting it run into a null dereference.

In the game folder the database lives at `DayZ/dayz-plugins/data/`, and the loader's
`scripts/build.sh --deploy` copies it there from a checkout of this repository.

## Current contents

| Build | Executable | Symbols | Offsets | Provenance |
| --- | --- | --- | --- | --- |
| 1.29.163709 | `DayZ_x64.exe`, PE timestamp `0x6A72FC58` | 31 | 12 | analysis |
| external-0x6A47B9AA | `DayZ_x64.exe`, PE timestamp `0x6A47B9AA` | 15 | 4 | external |
| external-diag-0x6A47BAF9 | `DayZDiag_x64.exe`, PE timestamp `0x6A47BAF9` | 8 | 4 | external |

The 1.29 addresses come from the reverse-engineering notes in
[dayz-plugins.github.io/research](https://github.com/dayz-plugins/dayz-plugins.github.io/tree/main/research),
which explain what each one is and how it was found. "Analysis" means read from a
disassembler and resolved against the real executable, not yet exercised by a running hook.

The two `external` builds are an older game version and its Diag executable, extracted from
[maksidze/DayZ-VR](https://github.com/maksidze/DayZ-VR), which gated on the PE timestamp and
carried no byte signatures. They have no hash and no checks, so for those builds a wrong
address cannot be caught at load time; the loader says so in a warning. They are here because
they make a second data point for every symbol, which is what tells a pattern from a
coincidence.

## License

Public domain (Unlicense), like the rest of the organisation's repositories.
