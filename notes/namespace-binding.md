# Namespace-bound registry pattern

> **Status: experimental.** Lives in `scratch/lib.cljc` + `scratch/user.cljc` + `scratch/user2.cljc` as a sketch. Not yet promoted to `src/`. API, naming, and macro internals all subject to change. Validate the open concerns below before committing.

Alternative to threading `@registry` through every call. Inspired by reader-monad / namespace-as-monad idea: instead of passing the registry per-call or stuffing it in a dynamic var, generate a **wrapped namespace** that closes over a specific registry atom.

## Sketch

```clj
;; library declares per-fn registry role via metadata
(defn eq?  {:registry :get} [registry x y] ...)
(defn reg-useq! {:registry :set} [registry-atom & opts] ...)

;; consumer instantiates a wrapped ns
(ns my.calendar
  (:require [unit-map.core]
            [unit-map.bind :as bind]))

(def my-registry (atom {}))
(bind/wrap-ns unit-map.core my-registry)

;; call sites lose the registry arg
(my.calendar/reg-useq! :unit :day :delta :days :useq ...)
(my.calendar/eq? {:day 1 ...} {:day 2 ...})
```

`:set` fns receive the atom; `:get` fns receive the deref. Macro generates one `defn` per marked var, preserving `:arglists`, doc, and adding `:inline` for compiler-side substitution.

## Why this is interesting

- **Lexical, not dynamic.** Registry choice fixed at wrap-time, visible in `:require`. No `binding` boilerplate at async/lazy-seq boundaries.
- **Honest dependencies.** Reading a module's `:require` block tells you which calendar it uses.
- **No global mutable default.** Each app/module makes its own wrapped ns. No "whose registry won?" question.
- **Mechanical.** Library author marks fns once via metadata; wrapper generated.
- **Per-call cost: zero noise.** Call sites identical to explicit-threading, minus the `@reg` arg.
- **Trust model unchanged.** Malicious dep can still `intern` into wrapped ns — but same as any Clojure code, no extra exposure.

## Open concerns to validate

1. **Reload semantics.** Current macro captures `@fvar` (the deref'd function value) at wrap-time. After `(require 'unit-map.core :reload)` the wrapped ns will hold stale references. Likely fix: capture the `Var` itself and let the call site deref each invocation. Test before promoting.
2. **Tooling.** `M-.` / jump-to-def, refactor-rename, `cider-doc` on generated vars — test in CIDER and Cursive. Doc and `:arglists` are preserved (good), but stack traces will show wrapper frames.
3. **`:inline` metadata.** Nice perf win, compounds debugging weirdness. Decide if the trade-off is worth keeping.
4. **Variadic + multi-arity dispatch.** `mk-overload` handles `&` but combinations like `eq?`'s `[reg x]` / `[reg x y]` / `[reg x y & more]` need explicit tests.
5. **Naming.** `set-registry/set-registry` reads ambiguously. Candidates: `unit-map.bind/wrap-ns`, `unit-map.bind/instantiate`. Pick before promoting from scratch.
6. **Pattern reuse.** This is library-agnostic — any registry-driven Clojure lib (spec, multimethod-heavy code, etc.) could adopt it. Worth considering whether the macro lives in `unit-map.bind` (project-local) or as a separate micro-lib.

## Recommended layering (if promoted)

Ship both:

1. **`unit-map.core`** — explicit registry threading. Honest, debuggable, no macro magic. Always works. Stays the canonical API.
2. **`unit-map.bind`** — the macro from scratch, polished. Opt-in for users who want per-namespace curried instances.
3. **No global default registry, no dynamic var.** The scratch pattern is better than dynamic vars precisely because it's lexical — don't muddy the water with both.

## Why not dynamic var sugar instead

Considered `^:dynamic *registry*` + `with-registry` macro. Rejected because:

- Hidden state — call site doesn't reveal the dependency.
- `binding` doesn't cross thread boundaries without `bound-fn`.
- Lazy-seq realization outside `binding` scope picks wrong value.
- Encourages a global default, which reintroduces the "whose registry?" question.

The namespace-binding pattern avoids all four.

## Doesn't solve

- Cannot hide registry from determined attackers (any code can `intern`/`alter-var-root`). Trust model unchanged from baseline Clojure — see prior discussion.
- Doesn't change the umap-vs-delta shape problem (see `plural-deltas.md`).
- Doesn't replace good naming, validation, or docs in the underlying API.
