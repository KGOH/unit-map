# Portable system definitions

> **Status: early idea.** Thinking about making unit system definitions language-agnostic so the concept isn't locked to Clojure.

## Goal

Define a counting system once (as data), use it from any language. The Clojure library becomes the reference implementation; the **spec** becomes the real product.

## Layered picture

1. **unit-map-spec** — JSON schema for system definitions. Language-agnostic. Dynamic useq endpoints expressed via a portable expression language.
2. **unit-map** (Clojure) — reference implementation, reads the spec natively.
3. **unit-forms** (ClojureScript UI) — visual editor that reads/writes the spec. See `unit-forms-ui.md`.
4. **unit-map-{py,js,go,...}** — thin interpreters of the spec in other languages (community or us).

## Expression language for dynamic parts

Static useqs are pure data — already portable. Dynamic useqs (e.g. `days-in-month` depending on month/year) need a portable expression language for endpoint fns.

Three candidates worth pursuing:

### JSONLogic (jsonlogic.com)
- Rules as JSON objects. Data IS code.
- Interpreters in 15+ languages (JS, Python, Ruby, PHP, Java, Go, C#, .NET, Elixir, Rust...).
- Simple, stable, community-maintained.
- Example `days-in-month`:
  ```json
  {"if": [
    {"==": [{"var": "month"}, "feb"]},
    {"if": [{"==": [{"%": [{"var": "year"}, 4]}, 0]}, 29, 28]},
    31
  ]}
  ```
- Pro: zero parsing, it's already JSON. Fits naturally in system definition files.
- Con: verbose for complex logic. No user-defined functions (flat rule evaluation only).

### Lua
- Tiny embeddable scripting language. Classic choice for "embed a language."
- Runtimes in C, Java (LuaJ), JS (fengari), Go (gopher-lua), Rust (mlua/rlua), Python (lupa), .NET (MoonSharp/NLua)...
- Example: `function(umap) if umap.month == "feb" then ... end end`
- Pro: full language, familiar syntax, battle-tested embedding story.
- Con: not JSON-native — expressions live as strings inside JSON. Sandboxing needed per host.

### WASM
- Compile core arithmetic to WASM once, call from any host.
- Universal runtime support (browsers, Node, Go, Rust, Python, Java, .NET, ...).
- Pro: single implementation, guaranteed identical behavior everywhere. No reimplementation per language.
- Con: solves the runtime problem, not the definition problem. System definitions still need a data format + expression language. WASM is the execution layer, not the spec layer.
- Could combine: spec uses JSONLogic for definitions, WASM for the arithmetic engine.

### Could layer them

- JSONLogic for **static + simple dynamic** definitions (covers 90% of systems).
- Lua as escape hatch for **complex dynamic** logic (leap seconds with side-effecting lookups, etc.).
- WASM as optional **execution engine** for the core arithmetic — guaranteed identical behavior across hosts.

## What goes in the spec

```json
{
  "useqs": [
    {
      "unit": "day",
      "delta": "days",
      "next_unit": "month",
      "useq": [{"range": {"start": 1, "step": 1, "end": "<jsonlogic or lua>"}}]
    },
    {
      "unit": "month",
      "delta": "months",
      "next_unit": "year",
      "useq": ["jan", "feb", "mar", "apr", "may", "jun",
               "jul", "aug", "sep", "oct", "nov", "dec"]
    }
  ],
  "systems": [
    {
      "name": "gregorian-date",
      "units": ["day", "month", "year"],
      "overflow": "error"
    }
  ]
}
```

Static useqs = plain arrays. Dynamic endpoints = JSONLogic object or Lua string, tagged by type.

## Open questions

- JSON vs EDN vs both? JSON for portability, EDN for Clojure ergonomics. Could support both with a thin translation layer.
- How to express `#unit-map/useq[1 2 .. N]` range syntax in JSON? The `{"range": {"start", "end", "step"}}` object above is one option.
- Versioning the spec format itself.
- Test suite: a set of system definitions + expected arithmetic results that any implementation must pass. This is what enforces cross-language correctness.
- Sandboxing Lua expressions (no filesystem, no network unless explicitly allowed).
