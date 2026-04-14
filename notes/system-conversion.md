# System conversion (`convert`)

> **Status: design proposal.** `find-conversion` exists in `system.cljc:207`. `convert` does not. This note sketches the algorithm and edge cases.

## API

```clj
(convert registry umap target-system) ;; -> umap shaped to target-system
```

Builds on `find-conversion` (which produces a structural plan as a vector of segment maps).

## Per-segment handling

Each segment from `find-conversion` is one of:

- **Identity segment** `{[u] [u]}` — copy: `(assoc result u (get umap u))`.
- **Branch segment** `{src-branch tgt-branch}` — base-conversion through a shared quantum (the lower anchor unit). This is the only interesting case.

## Branch reduction algorithm

Both `src-branch` and `tgt-branch` sit between the same two anchor segments (lower anchor `B`, upper anchor `T`). Both express "how to subdivide one `T` into `B`s". Conversion = base conversion via a shared integer quantum.

### Step 1 — source branch → integer total in lower-anchor units

Walk source branch high-to-low. Multiply accumulator by next-lower useq length, add next-lower index.

```
total = 0
for unit in (reverse src-branch):       ;; high to low
  useq   = useq for [unit, next-lower-unit-in-branch-or-anchor]
  idx    = useq-index-of(useq, src-umap, get(src-umap, unit))
  length = useq-length(useq, src-umap)
  total  = total * length + idx
```

After loop: `total` = "number of bottom-anchor-units offset within one top-anchor-unit".

### Step 2 — total → target branch values

Walk target branch low-to-high. Peel `mod length` per unit, divide for next.

```
remaining = total
for unit in tgt-branch:                  ;; low to high
  useq   = useq for [unit, next-higher-unit-in-branch-or-anchor]
  length = useq-length(useq, tgt-umap-so-far)   ;; partial result so far
  idx    = mod(remaining, length)
  result[unit] = useq-nth(useq, tgt-umap-so-far, idx)
  remaining = quot(remaining, length)
```

Final `remaining` should be 0. If not, source umap was unnormalized or plan was structurally wrong → assert.

## Worked example: am/pm → 24h

- Source: `{:hour 11, :period :am, :day 5, :month :jan, :year 2020}` in `[:ms :sec :min :hour :period :day :month :year]`
- Target system: `[:ms :sec :min :hour :day :month :year]`
- Branch segment in question: `{[:hour :period] [:hour]}` (anchors: `:min` below, `:day` above)

Source reduction (top-down: `:period`, then `:hour`):

| step    | useq             | value | idx | length | total            |
|---------|------------------|-------|-----|--------|------------------|
| `:period` | `[:am :pm]`     | `:am` | 0   | 2      | 0                |
| `:hour`   | `[12 1 2 .. 11]`| 11    | 11  | 12     | `0 * 12 + 11 = 11` |

Target decomposition (bottom-up: `:hour`):

| step  | useq        | length | idx (= mod) | value | remaining |
|-------|-------------|--------|-------------|-------|-----------|
| `:hour` | `[0 .. 23]` | 24     | 11          | 11    | 0         |

Result: `{:hour 11, :day 5, :month :jan, :year 2020}` (other units copy via identity segments). ✓

For 11pm input: source total = `1 * 12 + 11 = 23` → target `:hour` = 23. ✓
For 12am input: source total = `0 * 12 + 0 = 0` → target `:hour` = 0. ✓

## Edge cases

- **Empty branch on one side** (e.g. `{[:period] []}`). One system has extra subdivision the other lacks. Empty side contributes 0 going up or accepts 0 going down. If source has the extra unit and decomposing yields nonzero remaining → information loss. Detect and either error or return a `lossy?` flag.
- **Dynamic useq lengths.** All `useq-length` calls need umap context. Source: full source umap. Target: partial target umap built bottom-up — anchor units below the current segment must already be populated. Process plan in low-to-high order so this holds.
- **Missing values in source umap.** Treat as min value for the unit (matches `add-to-unit` semantics) or nil-propagate. Pick one, document.
- **Anchor at top of system** (no upper anchor). Total may be unbounded — emit as-is into target's top unit. No carry to absorb.
- **Eq-units relabeling.** `find-conversion`'s validity check uses `registry/eq-units` to permit branches whose endpoints use different unit names for the same quantum. Decomposition must honor this — when looking up the useq edge for a target unit, the "anchor below" may be named differently. Resolve via eq-units mapping.
- **Branches deeper than 2 units.** Not seen in current scratch but algorithm extends naturally — accumulator iterates more times.

## Risks

- **Correctness is subtle.** Off-by-one in index↔value (for useqs that don't start at 0, like `[12 1 2 .. 11]`) is the main hazard.
- **`find-conversion` may return nil** when `valid?` is false. `convert` should propagate as error or nil.
- **Performance.** Each branch reduction is O(branch-depth) lookups. Cheap. The expensive part is `useq-length` on dynamic useqs — bounded.

## Recommendation

1. **Write tests first.** Drive with two cases: am/pm ↔ 24h, and timestamp ↔ ms-year. Property test for round-trip:
   ```clj
   (eq? @reg umap (convert reg (convert reg umap target) source))
   ```
   for randomly generated valid umaps.
2. **Implement `convert-segment`** `[registry src-umap tgt-umap-acc segment] -> tgt-umap-acc'`.
3. **`convert`** = reduce `convert-segment` over `(find-conversion ...)` plan.
4. **Defer transitive conversion** (A → B → C through intermediate systems) until needed. `find-conversion` doesn't currently chain.

## Out of scope

- Lossy conversions with explicit rounding strategy (truncate / nearest / etc.).
- Conversions across registries.
- Bidirectional invertibility checks at registration time (cool but premature).
