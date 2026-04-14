# unit-map

Clojure(Script) library for modeling values composed of ordered units (dates, times, custom measurement systems). Successor to [chrono](https://github.com/HealthSamurai/chrono).

A "unit map" (umap) is a plain map like `{:day 26, :month :jul, :year 2020}`. The library lets you define **units**, **useqs** (sequences of valid unit values), and **systems** (ordered lists of units) — then compare, add deltas, parse, and format umaps against a registry.

## Build / Test

- `make repl` — clears `.cpcache/` then starts nREPL on port 55555 with `:test` + `:nrepl` aliases
- `make test` — run kaocha test suite (`clj -A:test:kaocha`)
- `make ci-test` — kaocha with `:ci` profile (dots reporter)
- Test config: `test/test.edn` (kaocha). Test ns pattern: `-test$`. Tests are randomized.
- Deps: Clojure 1.11.1, ClojureScript 1.11.54. Source is `.cljc` (cross-platform).

## Layout

```
src/unit_map/
  core.cljc                  -- public API (re-exports impl)
  impl/
    registry.cljc            -- pure registry data (useqs graph, eq-units, systems)
    registrator.cljc         -- atom-based registry mutation API
    system.cljc              -- urange/useq math, system guessing, conversions
    ops.cljc                 -- cmp/eq/lt/add-delta/subtract-delta/difference
    io.cljc                  -- parse/format string ↔ umap
    reader.cljc              -- #unit-map/useq[...] data reader (handles `..`)
    util.cljc                -- helpers (infinite?, pad-str, try-call, ...)
  systems/date_time/         -- predefined date/time useqs + systems
src/data_readers.cljc        -- registers #unit-map/useq reader
test/unit_map/...            -- mirrors src structure
scratch/                     -- experimental / WIP code, ignore for prod
```

## Core concepts

- **useq** — a value sequence for one unit. Built from literals + `urange`s via the `#unit-map/useq[...]` reader, which transforms `[0 1 .. 999]` into a urange `{:start 0 :end 999 :step 1}`. Endpoints can be functions of the umap (e.g. `days-in-month` for `:day`), making the useq dynamic.
- **registry** — atom holding `{:useqs ..., :eq-units ..., :systems ...}`. Created with `(core/new-registry)`. Useqs are stored as a graph keyed by `[unit next-unit]`. Systems are ordered unit vectors, e.g. `[:day :month :year]`.
- **system guessing** — given a umap, the registry picks the smallest registered system that covers all its units (`system/guess-system`, memoized).
- **eq-units** — alternative units treated as equivalent (e.g. branch resolution in `find-conversion`).

## Public API (`unit-map.core`)

- Registry: `new-registry`, `reg-useq!`, `reg-useqs!`, `reg-system!`, `reg-systems!`
- Compare: `cmp`, `eq?` / `not-eq?`, `lt?` / `lte?`, `gt?` / `gte?`
- Arithmetic: `add-delta`, `subtract-delta`, `difference`
- IO: `parse`, `format` (shadow `clojure.core/format`)

All comparison/arithmetic fns take a **dereffed** registry (`@reg`) as first arg; registration fns take the **atom**.

## IO format vectors

`fmt-vec` is a vector of:
- keyword `:unit` → emit/parse that unit
- `[:unit width pad-char]` → fixed-width with padding
- string/char/regex → literal separator
- `{:value ... :width ... :pad ...}` → explicit map form
- registered format element key (via `io/reg-format!`) → composite

See `unit_map/impl/io.cljc:14` for `reg-format!` and `unit_map/impl/io_test.clj` for examples.

## Conventions / gotchas

- **Source files are `.cljc`** — keep code portable; tests with JVM-only deps live in `.clj` files (e.g. `io_test.clj`, `ops_test.clj`).
- **Data reader** `#unit-map/useq` is registered via `src/data_readers.cljc`. Don't write raw urange maps inline; use the reader.
- `process-useq` is **memoized** (`reader.cljc:41`) — the reader returns identical vectors for identical literals.
- `core/cmp` etc. accept a dereffed registry value, NOT the atom. Check call sites before changing.
- Predefined date/time useqs in `systems/date_time/defs.cljc` are `def`s — not auto-registered. Caller does `(reg-useqs! reg defs/useqs)`.
- Scratch namespaces (`scratch/`) are sandbox; don't refactor unless asked.
- README is WIP — old chrono README lives at `src/unit_map/type/chrono` per the README, but that path isn't in the current tree (stale).
- TODO list at top of `core.cljc:10` tracks open design questions.
