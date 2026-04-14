# unit-forms UI

> **Status: idea.** Separate repo project, powered by unit-map as a dependency. Not part of the core library.

Interactive web UI for defining counting systems and doing operations within them. The "map" becomes a "form". Runs entirely in browser — library is `.cljc`, compiles to ClojureScript, no backend needed.

One-line pitch: **"Define any counting system and do math in it."**

## Why

- Removes "learn Clojure" barrier. Expands audience beyond developers.
- Makes the abstract pitch tangible — dynamic useqs updating field constraints live is a visceral demo.
- Doubles as marketing page, teaching tool, and playground.

## Concept mapping

| Library concept | UI element |
|---|---|
| umap `{:hour 11, :min 30}` | Form with labeled fields |
| useq `[0 1 .. 23]` | Number input with min/max |
| useq `[:jan :feb .. :dec]` | Dropdown / select |
| Dynamic useq | Field constraints that react to other fields live |
| system ordering | Field ordering in form |
| delta (plural keys) | Second form with plural labels |
| `add-delta` / `subtract-delta` | Action between two forms |
| `cmp` / `eq?` / `lt?` | Visual indicator (=, <, >) between two forms |
| `convert` | "Convert to..." dropdown, morphs form into different system |
| `format` | Live text preview below form |
| `parse` | Text input that populates form |

## Persistence

- **Shareable URLs** — encode system definition + current values in URL fragment. Someone defines a calendar → shares link → recipient sees it instantly. No account, no install.
- **LocalStorage** — user's custom systems persist across sessions. Edit, revisit, iterate.

## Audience

- **Educators** — teaching positional number systems, calendar math, base conversion.
- **Worldbuilders** — conlang/fantasy calendar design (D&D, fiction). Surprisingly large community.
- **Historians / ethnographers** — non-Gregorian calendar lookup and conversion.
- **Developers evaluating the lib** — play before committing to Clojure dependency.
- **Curious people** — "what's today in the Aztec calendar?" as a shareable link.

## MVP scope

Preloaded gallery only. No system builder UI in v1.

1. Gallery of interesting systems (Gregorian, Aztec, binary, semver, DMS, imperial length).
2. Form view of a umap. Fill in values, see formatted output.
3. Add/subtract delta. Result updates live.
4. Compare two umaps.
5. Shareable URL per system + values.
6. LocalStorage for user's value history.

System builder (user defines custom useqs/systems via UI) comes in v2.

## Constraints

- **Dynamic useqs with fn endpoints** don't serialize to URL/JSON. Preloaded systems can have fns; user-defined systems (v2) limited to static useqs unless a safe expression language is added.
- **Stack:** ClojureScript + Reagent/Re-frame is natural. Static site, no backend.
- **Name:** "unit-forms" is working title. Alternatives worth considering: "counting.systems" (as domain), "unitlab", "unit-map playground".

## Risks

- Building a good UI is a project as large as the library itself.
- Risk of spending effort on UI before core lib APIs (`convert`, overflow, plural deltas) are stable. Probably sequence: stabilize core → MVP UI.
