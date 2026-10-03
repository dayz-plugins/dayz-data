# Agent notes: dayz-data

## What this repository is

Data, not code. JSON only: byte signatures, per-build address caches, seeds and schemas. The
reader, the resolver and the generator all live in
[dayz-plugin-loader](https://github.com/dayz-plugins/dayz-plugin-loader) (`crates/dayz-data`
and `tools/dayz-data-tool`). Keeping the data separate means a correction ships without a
loader release and needs no toolchain to contribute.

Prose documentation belongs in
[dayz-plugins.github.io](https://github.com/dayz-plugins/dayz-plugins.github.io); the design
behind this format is `design/address-database.md` there. This repository carries only
`README.md` and `AGENTS.md`.

## Rules that matter

- **Never hand-edit `builds/*.json`.** Regenerate from the seed, so the checks and the
  metadata stay consistent with the executable. Hand-edit `seeds/*.json` instead.
- **Never overwrite a hand-wildcarded pattern with a generated one.** The generator merges
  and keeps what is already there; preserve that behaviour if you touch it.
- **When you move an address in a seed, delete that symbol's pattern.** The merge above keeps
  the old byte run, which then resolves to the old address while the cache holds the new one.
  `validate` cross-checks the cache against the patterns and reports the disagreement; a run
  with such a mismatch is not a passing run.
- **A global gets no check and no pattern.** Its bytes in the file are initialisation data,
  not what memory holds while the game runs. Writing a check for one produces a symbol that
  fails on every launch.
- **Verify before claiming.** `provenance` is `verified` only when a hook or a read actually
  exercised the address at runtime; `analysis` when it came from a disassembler; `scan` when a
  loader produced it automatically. Do not promote an entry without the evidence.
- **Do not invent addresses or patterns.** Every entry traces to the research notes or to a
  generator run against a real executable. A plausible-looking address nobody verified is
  worse than a missing one, because the loader will use it.
- **Names are `subsystem.thing`**, lower case, `[a-z0-9_]+(\.[a-z0-9_]+)+`, matching the
  research notes' sections. Renaming a symbol breaks every plugin that asks for it, so treat
  the names as the public interface they are.

## Checks

The schemas under `schema/` describe both file kinds, and the loader repository's tool is the
real check:

```bash
cargo run -p dayz-data-tool -- validate /path/to/dayz-data --exe "/path/to/DayZ_x64.exe"
```

It reports per symbol whether it resolved from the cache or by scanning, and names anything
that did not. Zero issues against the executable a build file claims is the bar for a commit.

## Conventions

- Conventional Commits, imperative subject. Commit trailer:
  `Co-Authored-By: <model name> <noreply@anthropic.com>`.
- Push to `origin` (`github.com/dayz-plugins/dayz-data`).
- GitHub Actions never run under this account: keep `workflow_dispatch` live and leave every
  automatic trigger commented out with a note saying why.
