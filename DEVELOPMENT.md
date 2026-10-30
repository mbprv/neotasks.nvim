# Development

Developer notes for `neotasks.nvim`. Architectural overview:
[ARCHITECTURE.md](ARCHITECTURE.md).

## Repository layout

```
lua/neotasks/            plugin source
  init.lua                public API (setup, register_* hooks)
  config.lua              runtime config
  commands.lua            :Neotasks user command
  project.lua             project-root discovery
  runner/                 task resolution + execution
  types/                  task-type registry + built-ins + schema merge
  expressions.lua              ${name} value substitutions
  lsp/                    in-process language server for the tasks file
  runview.lua             run -> panel bridge (runner signal subscriber)
  panel/                  task output panel (window, tabs, winbar)
  util/                   shared helpers
  tomltools/              VENDORED TOML engine (git subtree, see below)
tests/                    busted specs
```

## Running tests

[busted](https://lunarmodules.github.io/busted/), run through
[`tests/nvim-lua`](tests/nvim-lua) so each spec executes inside a real Neovim.

```sh
make test          # alias for unit_test
make unit_test     # busted specs under tests/
```

A single spec, or a filter:

```sh
make test BUSTED_ARGS=tests/completion_spec.lua
make test BUSTED_ARGS="--filter=runner -o gtest"
```

- busted must already be installed for Lua 5.1 (the version Neovim embeds),
  with `luarocks --lua-version=5.1 --local install busted`; `make test` fails
  if it is missing rather than installing anything.
- Specs are discovered through [`.busted`](.busted).
- [`tests/init.lua`](tests/init.lua) is the busted helper that sets up the
  environment.

## The vendored TOML engine (`tomltools`)

The TOML parser/decoder/encoder/validator/formatter and the schema navigation
the LSP uses are in the separate
[`tomltools`](https://github.com/<owner>/tomltools) repository, vendored here as
a **git subtree** (not a submodule), so a fresh clone needs no extra fetch.

### Why it is namespaced under `neotasks.`

- Upstream ships its library at `lua/tomltools/`, its modules requiring each
  other by the absolute name `tomltools.*`.
- Vendored at the runtimepath-visible `lua/tomltools/`, the top-level module
  name `tomltools` would be **global to Neovim**: any other plugin vendoring
  `tomltools` would collide, and whichever loaded first would silently win for
  both.
- So it is vendored under this plugin's own namespace:

| | |
|---|---|
| Vendored at | `lua/neotasks/tomltools/` |
| Imported as | `require("neotasks.tomltools")` (and `.parser`, `.Cst`, …) |

**Invariant:** every internal `require("tomltools…")` inside the vendored files
is rewritten to `require("neotasks.tomltools…")`; the update script re-applies
this on every sync. LuaCATS annotations (`---@class tomltools.Cst`, etc.) keep
the upstream names, for documentation only, with no effect on module
resolution.

### Updating the vendored engine

```sh
scripts/update-tomltools.sh          # vendor upstream main
scripts/update-tomltools.sh v1.2.3   # …or a specific tag / branch / commit
```

The script adds the upstream remote if missing, mirrors `lua/tomltools/*.lua`
into `lua/neotasks/tomltools/` with the namespace rewrite, prunes files
upstream deleted, verifies no bare `tomltools` require survived, and records
the pinned commit in [scripts/tomltools.lock](scripts/tomltools.lock).

It does **not** commit:

```sh
git diff lua/neotasks/tomltools
make test
git add lua/neotasks/tomltools scripts/tomltools.lock
git commit -m "Update vendored tomltools"
```

The pinned commit may lag `main` on purpose; pass an explicit ref to move it.

### After updating: check the consuming API

If the `tomltools` public or submodule API changed, these call sites must
follow:

- `runner/exec.lua`, `commands.lua`: `toml.parse`, `toml.find_path`,
  `toml.encode` (whole-document → `string`), `toml.encode_entry` (styled
  snippet → `string[]`).
- `lsp/server/*`: direct use of submodules `parser`, `decoder`, `formatter`,
  `validator`, `Cst`, `schema_nav`, `schema_util`.

Smoke test, besides `make test`: open a `neotasks.toml` (LSP
completion/diagnostics/hover) and run a task via `:Neotasks`.
