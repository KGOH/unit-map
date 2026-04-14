# Finite system overflow behavior

> **Status: design proposal.** Currently overflow past the top unit of a finite system is undefined — `inc-unit` (`ops.cljc:67-74`) recurses into `get-next-unit` which returns nil and silently produces a malformed umap. TODO at `ops.cljc:66` flags this gap. Symmetric problem exists for underflow.

## Motivation

Some finite systems want different overflow semantics:

- **Aztec without `:xiuhmolpilli`** — Calendar Rounds traditionally not counted; overflow at 18,980 days should *wrap* to the next Calendar Round position.
- **A bounded version counter** `[:patch :minor :major]` with `:major 1..99` — overflow should *error* (release process problem), not wrap.
- **Game tick clock** — overflow should *saturate* at max ("end of game").
- **Gregorian without `:year`** (just `[:day :month]`) — overflow at year boundary should wrap (like recurring birthdays).

One-size-fits-all default is wrong. Belongs in system definition because the same useqs may be reused in different systems with different policies.

## API

`reg-system!` takes an `:overflow` option (and symmetric `:underflow`):

```clj
(reg-system! reg
  [:vientena :meztli :xihuitl]
  :overflow  :wrap
  :underflow :wrap)

(reg-system! reg
  [:patch :minor :major]
  :overflow :error)

(reg-system! reg
  [:tick]
  :overflow :saturate)
```

### Built-in policies

| Policy      | Behavior on overflow past top                            |
|-------------|----------------------------------------------------------|
| `:wrap`     | Modular: reset top unit to its min, continue arithmetic  |
| `:error`    | Throw `ex-info` with `{:type :unit-map/overflow ...}`    |
| `:saturate` | Clamp at max value of top unit, drop further carry       |
| `:nil`      | Return `nil` from the arithmetic op                      |
| fn          | Custom: `(fn [registry umap unit overflow-amount] ...)`  |

Default: `:error` (loud failure beats silent corruption — current silent-nil is the worst option).

## Storage

Registry gains `:system-overflows` map:

```clj
{:systems       #{[...]}
 :system-overflows {[:patch :minor :major]    {:overflow :error  :underflow :error}
                    [:vientena :meztli :xihuitl] {:overflow :wrap :underflow :wrap}}}
```

Lookup at arithmetic time via `(get-in registry [:system-overflows system])`. Default applies when not specified.

## Implementation touch points

- **`inc-unit`** (`ops.cljc:67`) — when `get-next-unit` returns nil and current unit is at max, consult overflow policy.
- **`dec-unit`** (`ops.cljc:77`) — symmetric for underflow.
- **`add-to-unit`** (`ops.cljc:95`) — when carry would propagate past top, consult policy. With the bulk-advance optimization (see `dynamic-useq-arithmetic.md`) this becomes simpler: check once per top-level overflow rather than per inc.
- **`subtract-from-unit`** — symmetric.
- **`difference`** — already operates within `system-intersection`; overflow shouldn't apply (result is a delta, not a umap). Skip.
- **`convert`** (see `system-conversion.md`) — when source value can't fit in target system's top unit, this is also an overflow case. Same policy applies.

## Interaction with other proposals

- **Plural deltas** (`plural-deltas.md`) — orthogonal. Overflow happens during arithmetic, regardless of delta shape.
- **Bulk-advance arithmetic** (`dynamic-useq-arithmetic.md`) — simplifies overflow check (once at top vs. per inc). Should land overflow API together with the bulk-advance refactor since both touch `add-to-unit`.
- **Namespace binding** (`namespace-binding.md`) — orthogonal. The wrapped namespace just inherits whatever the registry's system declarations say.
- **System conversion** (`system-conversion.md`) — overlapping. Conversion can also overflow. Same `:overflow` policy applies at the destination system's top.

## Edge cases

- **Infinite top unit** (e.g. `:year [-Inf .. Inf]`) — overflow impossible, policy is moot. `reg-system!` could warn if `:overflow` is specified but top is infinite.
- **`:wrap` with dynamic top-level useq** — top length may depend on... what? If the top has no `:next-unit`, what umap context drives length? Usually static in practice. Document the constraint or assert at registration.
- **Custom fn returning a umap in a different system** — allowed but spooky. Don't try to catch this; trust the user.
- **`:saturate` semantics for `add-delta` vs `subtract-delta`** — saturating at top for add, at bottom for subtract. Symmetric and obvious but worth a test.
- **Multiple systems share a useq stack but differ in overflow policy** — fine, policy is per-system not per-useq.

## Defaults question

Default `:error` is safest but breaks any code that's been silently relying on the current nil behavior. Library has no users — break freely. Pick `:error`.

## Recommendation

1. Land alongside bulk-advance refactor (both touch `add-to-unit`).
2. Default `:error`, document loudly.
3. Update Aztec example to demonstrate `:wrap` for the no-Calendar-Round system and absence of `:overflow` (defaults to error) for the version-number example.
4. Tests: each policy exercised via `add-delta` past top and `subtract-delta` past bottom, plus custom-fn callback.
