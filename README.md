# pj-deno

Plain Deno project layer for [kata](https://github.com/yukimemi/kata)
templates. Compose it under
[`pj-base`](https://github.com/yukimemi/pj-base).

Deno ships its own toolchain — formatter, linter, type checker, test
runner, task runner — so there is no build system to wire up. What
every Deno project *does* share is the CI shape: `setup-deno`, then
one `deno task ci`. That is what this layer carries.

## Not for Denops plugins

[`pj-denops`](https://github.com/yukimemi/pj-denops) covers those. Its
CI installs real Vim / Neovim binaries and exports
`DENOPS_TEST_VIM_EXECUTABLE` / `DENOPS_TEST_NVIM_EXECUTABLE`, which a
plain Deno app has no use for. `pj-deno` is the plain
"setup-deno + deno task ci" workflow that `pj-denops` deliberately is
not.

Use `pj-deno` for Deno apps, CLIs, scrapers, cron jobs and JSR
libraries. Use `pj-denops` for editor plugins.

## What it ships

| File | Mode | Notes |
|---|---|---|
| `.github/workflows/ci.yml` | overwrite, always | OS matrix running `deno task ci` |
| `.kata/vars.toml` | merge-toml, once | `actions.deno_setup`, `deno.version`, `deno.os_matrix` |
| `deno.json` | overwrite, once | Task set + fmt policy for a project without one |
| `deno.json` | merge-json (`fmt.exclude`), always | Keeps `deno fmt --check` off kata-managed files |
| `AGENTS.md` | merge-section, always | `<!-- kata:agents:deno:* -->` block |
| `renovate.json` | merge-json (`extends`), always | Chains to this layer's `default.json` |

## Contract: `deno task ci`

The workflow's only gate is `deno task ci`, so the consumer's
`deno.json` must define it. The convention across these repos:

```json
{
  "tasks": {
    "ci": "deno task check && deno task lint && deno task fmt --check && deno task test"
  }
}
```

Keeping the gate inside the project means a project can add a step
(a build, an extra check) without touching a kata-managed file.

A project that has no `deno.json` yet gets one seeded (`when =
"once"`) with exactly that task set. An existing project is adopted
untouched — it just has to define `ci` itself.

## Prerequisite: `deno.json`, strict JSON

**Not `deno.jsonc`, and no comments.** Both edges are real and were
verified against kata 0.15.0:

- kata's `merge-json` parses the existing file with strict
  `serde_json`, so one `//` comment makes every `kata apply` exit 1
  (`parsing existing …: key must be a string`). `kata-apply.yml` runs
  `kata apply` as a bare step, so the daily template sweep then fails
  forever and no upstream fix ever reaches the repo. Teaching kata to
  read JSONC would not help: re-serialisation drops the comments it
  just parsed.
- the `once` gate keys on the destination *path*, so a project whose
  config is `deno.jsonc` is not adopted — kata writes a fresh
  `deno.json` beside it, Deno prefers `deno.json`, and the project's
  real import map, tasks and JSR `exports` are silently ignored.
  `deno check .` then fails with `TS2307 … not in import map`.

Convert `deno.jsonc` into a comment-free `deno.json` before adding
this layer.

## `fmt.exclude` is layer-owned

`deno fmt` formats markdown, json and yaml, so the kata-managed
files (`AGENTS.md`, `CLAUDE.md`, `apm.lock.yaml`, the renri
`SKILL.md` files, the workflows) fail `deno fmt --check` the moment
kata applies — they are written by their upstream template, not by
`deno fmt`. Formatting them locally is not a fix: they are
`when = "always"` files, so the next apply reverts it.

So `fmt.exclude` lists them and is re-written on every apply
(`deno.fmt.json` here). `when = "always"` rather than a `once` seed
because a `once` entry against an existing `deno.json` is *adopted*,
which would skip every project that predates this layer — exactly
the ones that need it.

Project-specific exclusions go in `deno.json`'s **top-level
`exclude`** (honoured by fmt, lint and check alike), not in
`fmt.exclude`.

## Knobs

Both are seeded once into `.kata/vars.toml` and consumer-owned
afterwards:

```toml
[deno]
version = "2.x"                                                  # setup-deno deno-version
os_matrix = ["ubuntu-latest", "macos-latest", "windows-latest"]   # CI runners
```

`version` is pinned explicitly rather than inheriting
`denoland/setup-deno`'s default, which has moved before (`1.x` ->
`2.x`) and would otherwise swap the toolchain under every consumer at
once.

Trimming `os_matrix` renames the matrix legs, and the required status
check is `ci (<first os>)` — update branch protection to match.

## Usage

Via the `deno` preset in
[`pj-presets`](https://github.com/yukimemi/pj-presets):

```sh
kata init github.com/yukimemi/pj-presets:deno
```

Or added to an existing kata-managed project:

```sh
kata add github.com/yukimemi/pj-deno
```

## License

MIT
