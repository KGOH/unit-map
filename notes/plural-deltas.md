# Plural delta keys

Separate umap shape from delta shape. Umap values are positions in a useq (possibly keywords, e.g. `{:month :jul}`). Delta values are integer counts under user-defined plural keys (`{:months 7}`).

## Decisions

- **Option key on `reg-useq!`:** `:delta` (singular spelling, value is the plural keyword).
- **Required at registration:** assert `:delta` is supplied. Fail loud at `reg-useq!` time, not at first arithmetic call.
- **Unknown keys in delta maps:** silently ignored. A delta with no recognized plural keys = empty delta = no-op.
- **No backwards compatibility.** No users yet. Break freely.

## API shape

```clj
(reg-useq! reg
  :unit      :day
  :delta     :days
  :useq      #unit-map/useq[1 2 .. days-in-month]
  :next-unit :month)

(add-delta @reg
  {:day 26 :month :jul :year 2020}
  {:days 7 :months 1})
;; => {:day 2 :month :sep :year 2020}

(difference @reg
  {:day 26 :month :jul :year 2020}
  {:day 1  :month :jan :year 2020})
;; => {:days 25 :months 6}
```

## Implementation steps

### 1. Registry (`src/unit_map/impl/registry.cljc`)

- `reg-useq` — accept `:delta` arg. Assert non-nil at registration. Store in useq-info.
- Add two lookup tables to registry value:
  - `:delta->unit` — `{:days :day, :months :month, ...}`
  - `:unit->delta` — `{:day :days, :month :months, ...}`
- Build/update both in `reg-useq` alongside the useqs graph.
- Helper accessors: `(delta->unit registry plural-kw)`, `(unit->delta registry unit-kw)`.

### 2. Registrator (`src/unit_map/impl/registrator.cljc`)

- `reg-useq!` — pass `:delta` through. Assert before swap.

### 3. Core API (`src/unit_map/core.cljc`)

- `reg-useq!` — add `:delta` to destructure + forward.
- Docstrings / arg lists updated.

### 4. Ops (`src/unit_map/impl/ops.cljc`)

- New helper `delta->umap-keys [registry delta] -> {unit-kw int, ...}`:
  - Walk delta entries, look up `delta->unit`, drop unrecognized keys.
  - Return possibly-empty singular-keyed map of integers.
- `add-delta` — translate delta first, then existing reduce over `system-intersection`. The intersection still operates on singular keys (umap + translated delta).
- `subtract-delta` — same translation step.
- `difference` — translate output: after building singular-keyed result map, walk it and remap to plural via `unit->delta`. Return plural-keyed map.
- Empty delta after translation → return umap unchanged (already a no-op via the reduce, but assert/short-circuit for clarity).

### 5. Predefined date/time (`src/unit_map/systems/date_time/defs.cljc`)

Add `:delta` to each useq def:

| `:unit`   | `:delta`        |
|-----------|-----------------|
| `:ns`     | `:nanoseconds`  |
| `:ms`     | `:milliseconds` |
| `:sec`    | `:seconds`      |
| `:min`    | `:minutes`      |
| `:hour`   | `:hours`        |
| `:day`    | `:days`         |
| `:month`  | `:months`       |
| `:year`   | `:years`        |
| `:period` | `:periods`      |

(Sanity-check `:period` later — may not need a delta in practice, but registration requires it.)

### 6. Tests

Update call sites:

- `test/unit_map/core_test.clj` — fixture `reg-useq!` calls add `:delta`. All `add-delta` / `subtract-delta` / `difference` calls switch to plural keys. `difference` assertions switch to plural-keyed expected maps.
- `test/unit_map/impl/ops_test.clj` — same.
- `test/unit_map/systems/date_time/misc_test.clj` — same.
- `test/unit_map/impl/registrator_test.clj` — add a test for the missing-`:delta` assert.

### 7. Scratch / inline comments

- `src/unit_map/impl/ops.cljc:88-92` — update the `(comment ...)` to plural form.
- Sweep `scratch/` for delta-shape examples and update.

### 8. Docs

- `CLAUDE.md` — add a "umap vs delta" note in Core concepts. Mention `:delta` is required at `reg-useq!`.
- `core.cljc:10` TODO list — remove the "maybe use plural for deltas" line.

## Edge cases / risks

- **`add-delta` on registry where umap and delta share zero unit overlap after translation** → `system-intersection` may return nil. Current behavior with empty delta is to return umap unchanged. Verify the reduce handles `nil` system gracefully (`reverse nil` is `()`, reduce over empty seq returns init = umap. OK.).
- **Delta with mix of recognized + unrecognized keys** → drop unrecognized, proceed with rest. No warning.
- **Same `:delta` registered twice for different units** → registration should reject. Add to assert: plural key must be globally unique in registry.
- **`difference` between umaps using a system whose useqs lack `:delta`** → impossible because reg-time assert prevents it.

## Out of scope

- Auto-deriving plural from singular.
- Accepting both singular and plural shapes (no compat shim).
- Validating delta values are integers (trust input; useqs already index by int).
- Splitting `:delta` into separate read/write keys.
