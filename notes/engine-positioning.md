# unit-map as an engine

> **Status: positioning decision, motivates a repo restructure.** Captures the insight that unit-map's current "framework + bundled systems" shape is the source of its adoption friction. Solution: split engine from definition libraries.

## The insight

unit-map is an **engine for definition libraries**, not an end-user product.

- **End users don't care about extensibility.** They want `(gregorian/now)`, `(mars/sol-date)`, `(astro/parse-jd ...)`. They don't want to learn `reg-useq!` and design useqs.
- **Library authors care about extensibility.** They want a battle-tested machine for "ordered units with custom rules" so they don't have to reinvent comparison/arithmetic/parse/format/convert from scratch for their domain.
- **Today the library serves both audiences poorly.** End users see `src/unit_map/systems/date_time/` and ask "why do I need a registry to get a date?". Library authors see `src/unit_map/core.cljc` re-exporting registration fns alongside arithmetic and ask "is this a framework or a date library?".

The fix: stop pretending it's both. **unit-map is the engine.** Definition libraries are separate.

## Architecture

```
unit-map                     <- this repo, the engine. No bundled systems.
  ├─ unit-map.core           registry + ops + io + convert
  ├─ unit-map.bind           ns-wrapping macro (from scratch/, polished)
  └─ unit-map.spec           registry shape validation (future)

unit-map.gregorian           <- separate lib: standard Gregorian date/time
unit-map.astro               <- separate lib: UTC/TAI/TT/GPS/leap-seconds
unit-map.mars                <- separate lib: Mars Sol Date, MTC, mission sols
unit-map.lunar               <- separate lib: Lunar Coordinated Time
unit-map.aztec               <- separate lib: tonalpohualli + xiuhpohualli
unit-map.fiscal              <- separate lib: 4-4-5, broadcast, retail calendars
... etc, community-contributable
```

Each definition library:
- Depends on `unit-map`.
- Holds its own private registry atom.
- Builds the registry once at load time (or lazily on first call).
- Exposes a clean per-domain API that **never mentions registries**.
- Hides the engine via either dynamic var or `unit-map.bind` ns wrapping.

Example shape for `unit-map.mars`:

```clj
(ns unit-map.mars
  (:require [unit-map.core :as um]
            [unit-map.bind :as bind]))

(defonce ^:private registry
  (let [r (um/new-registry)]
    (um/reg-useqs! r mars-useqs)
    (um/reg-systems! r mars-systems)
    r))

(bind/wrap-ns unit-map.core registry)

;; now this namespace re-exports um/eq?, um/add-delta, etc., curried over the
;; mars registry. Users call (mars/eq? sol1 sol2) — registry invisible.
```

End-user experience:

```clj
(require '[unit-map.mars :as mars])

(mars/now)                                   ;; => {:sol 1247 ...}
(mars/add-delta a-sol {:sols 30})
(mars/eq? sol1 sol2)
(mars/format some-sol [:sol "T" :hour ":" :min])
```

No `reg-useq!`. No `@registry`. No knowledge of the engine required.

## Why this fixes the adoption problem

- **Each definition library has a focused pitch.** "Gregorian dates" is a comprehensible product. "Composable unit systems" isn't.
- **The engine retains its generality** without having to sell it. Library authors find it when they need it.
- **Demos compete on their own merits.** `unit-map.mars` doesn't need to convince you that calendars-as-data is a good idea — it just shows you Mars time.
- **Discoverability via concrete domains.** Someone searching "clojure mars time" or "clojure leap second" finds the matching definition lib, not the engine.
- **Burnout relief.** Shipping `unit-map.gregorian` v0.1 = one focused milestone proving the engine works. Way more achievable than "make unit-map appealing to everyone".
- **Community contribution path becomes obvious.** Want a Hebrew calendar? Build `unit-map.hebrew`, depend on engine, publish. No PR-into-monorepo coordination.

## What moves where

### Stays in this repo (engine)

- `src/unit_map/core.cljc`
- `src/unit_map/impl/*` (registry, ops, io, system, reader, util, registrator)
- `src/data_readers.cljc` (the `#unit-map/useq` reader)
- `unit_map.bind` (new, from `scratch/lib.cljc`)
- Tests covering all of the above
- One **reference definition** (probably `unit-map.systems.gregorian`) kept in `test/` or a `dev/` profile — *not shipped in the jar* — purely to exercise the engine end-to-end. Same way Ring's adapter tests use a real server but don't bundle it.

### Moves to a separate repo

- Everything currently under `src/unit_map/systems/` → goes away from this repo.
- `unit-map.gregorian` is the obvious first split. Becomes its own repo, its own deps.edn, its own version.

### Deleted entirely

- The `(reduce < 1 [2 3 4])` / `(transduce ...)` REPL leftovers at `core.cljc:167`.
- Stale README references to chrono path that doesn't exist.
- Probably `scratch/` after `bind` is promoted.

## Implications for the open design notes

- **`namespace-binding.md`** becomes load-bearing. The ns-wrapping macro is now *the* recommended pattern for definition libraries. Polish it (var-vs-deref reload bug, naming) before splitting any def lib. Promote from `scratch/`.
- **`plural-deltas.md`** — engine concern. Land in engine before any def lib stabilizes its API.
- **`dynamic-useq-arithmetic.md`** — engine concern. Same.
- **`system-overflow.md`** — engine concern. Same.
- **`system-conversion.md`** — engine concern, but with a wrinkle: conversion across def libs is impossible (they have separate registries). Either:
  - Conversion is intra-library only (each def lib registers all systems it wants to convert between).
  - Or provide a `unit-map.merged-registry` utility for users who want cross-domain conversion (e.g. `aztec → gregorian`). Engine ships the merging primitive; users opt in.
- **`use-cases.md`** — was a brainstorm; with this positioning, half the doc collapses into "list of candidate definition libraries". The engine doesn't need a wedge — each *definition library* needs its own wedge.
- **`aztec-calendar.md`** — becomes the seed for `unit-map.aztec` (separate repo).

## Risks and trade-offs

- **Repo proliferation overhead.** 5+ small libs to maintain vs. one. Mitigate: treat each def lib as small, stable, low-touch. Most won't change after v1. Engine is the only thing under active development.
- **Engine API stability becomes critical.** Breaking changes cascade through every def lib. This is fine — forces discipline. Land all the open design notes (plural deltas, overflow, etc.) *before* declaring engine v1.0, then commit to backwards compat. Pre-1.0, break freely.
- **Cross-domain conversion harder.** As above — separate registries can't directly converse. Worth noting; not blocking.
- **Marketing diffusion.** Each def lib has to do its own marketing. But each *can* — they have concrete value props. Engine becomes a SEO-irrelevant infrastructure concern, which is the right outcome.
- **First-impression friction for someone reading the engine repo cold.** Without bundled systems, the engine repo is abstract. Solution: README leads with "if you want to USE a calendar, see unit-map.gregorian / unit-map.astro / etc.; this repo is the engine they're built on".

## Recommended sequencing

1. **Land all engine design changes first** (plural deltas, bulk-advance, overflow, convert). Get the engine API into a shape you're willing to commit to.
2. **Polish `unit-map.bind`**, fix the var-vs-deref reload issue, promote from scratch.
3. **Split `unit-map.gregorian` out of this repo** as the reference definition library. This is the proof-of-pattern. Validates that the engine + bind + dynamic var pattern actually produces a clean end-user API.
4. **Rewrite engine README** around the engine framing. Point at gregorian repo as the canonical example.
5. **Then build the wedge libraries** in priority order: `unit-map.astro` (leap seconds + UTC/TAI/TT, the killer demo), `unit-map.mars`, etc. Each is small once the pattern is established.
6. **Solicit community contributions** for niche calendars (hebrew, islamic, ethiopian, mayan...). They become small focused libs, easy to merge or just exist independently.

## Bottom line

The library is an engine. The current shape (engine + bundled systems + extensibility-as-feature) tries to serve two audiences and serves both poorly. Splitting clarifies the value proposition, gives end users focused products, gives library authors a clean toolkit, and breaks the "what is this for?" deadlock that's been blocking momentum.

This is probably the highest-leverage change in the entire notes/ folder.
