# Dynamic useq arithmetic optimization

> **Status: design proposal.** No code changes yet. Optimization 1 is the easy win and should land first; 2 and 3 are speculative and gated on real demand.

## Current state

`ops/add-to-unit` (`src/unit_map/impl/ops.cljc:95-125`) has two branches:

1. **Static useq + `|x| > 1`** — index-based: `useq-nth (mod (+ idx x) length)`, carry via floor division. O(1) within unit, recurses on carry.
2. **Otherwise (dynamic useq, OR `|x| ≤ 1`)** — literal `n-times inc-unit` / `dec-unit`. O(x) walks of `get-next-unit-value`.

For dynamic useqs (e.g. `:day` with `days-in-month`), adding N is O(N) inc calls. Adding 365 days = 365 walks, even though most steps just hop within the same month.

## Optimization 1: bulk-advance within current bucket

Within a single bucket, a dynamic useq is effectively static — its length is fixed by the **current** umap. Apply the static-branch trick, but evaluate `useq-length` against the live umap on each carry:

```
(defn add-to-unit [registry umap unit x]
  (cond
    (zero? x) umap
    :else
    (let [useq     (system/get-useq registry umap unit)
          idx      (or (some->> (get umap unit) (system/useq-index-of useq umap))
                       (system/useq-first-index useq umap))
          length   (system/useq-length useq umap)
          capacity (- length idx 1)]              ;; positions left this bucket
      (cond
        ;; positive, fits in current bucket
        (and (pos? x) (<= x capacity))
        (assoc umap unit (system/useq-nth useq umap (+ idx x)))

        ;; positive, overflow — consume rest of bucket, carry 1
        (pos? x)
        (let [consumed (inc capacity)
              umap'    (-> umap
                           (assoc unit (system/get-min-value registry umap unit))
                           (->> (inc-unit-via-next-unit-carry registry unit)))]
          (recur registry umap' unit (- x consumed)))

        ;; negative — symmetric (uses idx as remaining capacity downward)
        ...))))
```

(Sketch — actual code needs to handle infinite useqs, missing unit values, and the negative case symmetrically.)

**Cost:** O(buckets crossed) instead of O(x).

| Operation              | Before  | After |
|------------------------|---------|-------|
| +365 days from Jan 1   | 365     | 12    |
| +1,000,000 days        | 1M      | ~33k  |
| +1 hour                | 1       | 1     |

~30× speedup for typical date arithmetic. No useq-author cooperation needed. Static and dynamic branches converge into one algorithm parameterized only by whether `useq-length` is finite.

**Note:** the existing static branch is a special case of this — when the useq is static, `useq-length` doesn't depend on umap and the index math degenerates to the current implementation. After opt 1, the two branches can be unified.

## Optimization 2: cascade bulk-advance up the stack

After opt 1, still O(buckets crossed). For deep stacks crossing many years, that's `~12 × years` ops.

Idea: at each level, after the within-bucket jump consumes the current bucket, check if the **next-unit's bucket** can fit the entire remaining x. If yes, sum bucket lengths over the next-unit's bucket, subtract, carry once at the next level. Repeat.

E.g. adding 1M days:
- Day level: consume rest of current month (~1 op).
- Year level: ask "how many days in this year?" (12 calls to `days-in-month`), subtract if x exceeds, carry one year. Repeat across years until x < year-length.
- Final descent: carry-down through `:month` then `:day` to land on the exact final position.

**Cost:** roughly O(levels × buckets-per-level). Adding 1M days drops from ~33000 ops to a few hundred.

**Catch:** requires summing `useq-length` over hypothetical umaps you haven't constructed yet. For `:day → :month`: cheap (12 calls per year). For `:month → :year`: trivial (always 12). Workable but adds real code complexity.

Skip until benchmarks justify.

## Optimization 3: useq-author escape hatch

For useqs with closed-form arithmetic (e.g. Gregorian "days since epoch", Howard Hinnant's `days_from_civil`), let the author register an optional bulk-advance fn:

```clj
(reg-useq! reg
  :unit         :day
  :delta        :days
  :useq         ...
  :bulk-advance (fn [registry umap x]
                  ...
                  ;; returns updated umap, or nil to fall back to opt 1
                  ))
```

`add-to-unit` checks for it before the generic path.

**Cost:** O(1) per opt-in unit. Adds API surface and most users won't bother. Add only when a real user asks.

## Bonus: shares helpers with `difference`

`ops/difference` (`ops.cljc:147-178`) and the long worked-example comment at `ops.cljc:219+` are wrestling with the same "buckets crossed under varying umap" structure. The bulk-advance helpers from opt 1 likely simplify `difference` too. Worth refactoring together.

## Recommendation

1. **Land opt 1 with the plural-deltas work** (or right after). Pure win, modest change. Unifies the two `add-to-unit` branches.
2. **Add property tests** for `add-delta` / `subtract-delta` round-trips against the old implementation before merging — easy correctness check.
3. **Benchmark** before and after. If opt 1 is enough for realistic workloads, stop there.
4. **Defer opt 2 and opt 3** until a real performance complaint surfaces.

## Risks

- Negative-direction symmetry is easy to get wrong. Test both directions across bucket boundaries with property tests.
- Infinite useqs (`##Inf` end) — `useq-length` returns `##Inf`, capacity is infinite, fast path always applies. Confirm `useq-nth` handles this.
- Missing unit values in umap — current code uses `get-min-value` as default; preserve that semantics.
- Carry into a unit not present in current system (top-of-stack overflow) — current code recurses into `get-next-unit` which may return nil. Existing TODO at `ops.cljc:66`. Same hazard, not made worse.
