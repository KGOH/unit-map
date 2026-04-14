# Aztec calendar definitions

> **Status: example / reference.** Not part of `src/unit_map/systems/`. Useful as a non-trivial test case for dynamic urange endpoints, multi-system registries, and (eventually) `convert`. Domain knowledge from publicly-known descriptions of the Aztec calendar; corrections welcome from anyone with deeper expertise.

Two interlocking calendars plus their Calendar Round meta-cycle:

- **tonalpohualli** — 260-day ritual cycle. Combination of 20 day-signs (`:tonalli`) cycling against 13 numbers (`:teyollia`).
- **xiuhpohualli** — 365-day solar year. 18 named months of 20 days plus a 5-day intercalary period (`:nemontemi`).
- **Calendar Round** — 52 xiuhpohualli = 73 tonalpohualli = 18,980 days. Counted by `:xiuhmolpilli`.

## Defs

```clj
(def gregorian
  (array-map
    :day   [1 2 '.. days-in-month]
    :month [:jan :feb :mar :apr :may :jun :jul :aug :sep :oct :nov :dec]
    :year  [##-Inf '.. -2 -1 1 2 '.. ##Inf]))


;; 260-day ritual cycle: tonalli (20 day-signs) × teyollia (13 numbers).
;; The :tonalpohualli-cycle unit counts which 260-day cycle we're in;
;; one Calendar Round contains 73 such cycles.
(def tonalpohualli
  (array-map
    :tonalli            [:cipactli   :ehecatl    :calli   :cuetzpalin :coatl
                         :miquiztli  :mazatl     :tochtli :atl        :itzcuintli
                         :ozomahtli  :malinalli  :acatl   :ocelotl    :cuauhtli
                         :cozcacuauhtli :ollin   :tecpatl :quiahuitl  :xochitl]
    :teyollia           [1 2 '.. 13]
    :tonalpohualli-cycle [1 2 '.. 73]))   ;; 73 = number of tonalpohualli per Calendar Round


;; 365-day solar year: 18 months of 20 days + 5-day :nemontemi intercalary period.
;; Vientena length depends on month — :nemontemi has only 5 days, all others 20.
(def xiuhpohualli
  (array-map
    :vientena    [1 '.. (fn [{:keys [meztli]}]
                          (if (= :nemontemi meztli) 5 20))]
    :meztli      [:izcalli         :atlcahualo  :tlacaxipehualiztli :tozoztontli
                  :hueytozoztli    :toxcatl     :etzalcualiztli     :tecuilhuitontli
                  :hueytecuilhuitl :tlaxochimaco :xocotlhuetzi      :ochpaniztli
                  :teotleco        :tepeilhuitl :quecholli          :panquetzaliztli
                  :atemoztli       :tititl      :nemontemi]
    :xihuitl     [1 2 '.. 52]    ;; year within Calendar Round
    :xiuhmolpilli [1 2 '.. ##Inf])) ;; Calendar Round count, unbounded
```

## Corrections from the original sketch

1. **`:wayeb` → `:nemontemi`** in the `:vientena` lambda. *Wayeb* is the Maya Haab name for the year-end intercalary period; *nemontemi* is the Aztec equivalent. The data already used `:nemontemi` correctly, but the lambda checked the wrong keyword — meaning the 5-day truncation never fired. Now matches.
2. **`:tonalpohualli [1 .. 72]` → `[1 .. 73]`.** 73 tonalpohualli cycles fit in one Calendar Round (52 × 365 = 73 × 260 = 18,980 days). 72 was off-by-one. Use `##Inf` instead if you want an unbounded cycle counter unrelated to Calendar Round alignment.
3. **System name `tonalpohualli-tonalli` → `tonalpohualli`.** *Tonalpohualli* is the name of the whole 260-day calendar; suffixing it with one of its units (`-tonalli`) is like calling Gregorian `gregorian-day`. Just the calendar name suffices.
4. **Unit name `:tonalpohualli` → `:tonalpohualli-cycle`.** Avoids the unit-named-after-its-own-calendar collision in code (`(get umap :tonalpohualli)` is now unambiguously "which cycle"). Open to a better Nahuatl-native name from someone who knows the language.

## Verification

- 18 named months × 20 days + 5 nemontemi days = 365. ✓
- 20 day-signs × 13 numbers = 260. ✓
- 52 xiuhpohualli × 365 days = 18,980 days = 73 tonalpohualli × 260. ✓ (Calendar Round invariant)

## Why these are useful as test fixtures

- **Dynamic urange endpoint via lambda** — `:vientena` truncates from 20 to 5 based on `:meztli`. Same shape as `days-in-month` for Gregorian but more dramatic ratio. Good stress test for `add-to-unit` bucket-crossing optimizations (see `dynamic-useq-arithmetic.md`).
- **Keyword-valued useqs at multiple levels** — `:tonalli`, `:meztli`, and Gregorian `:month` all use keyword positions. Confirms umap-vs-delta shape separation (see `plural-deltas.md`) holds across non-numeric useqs.
- **Multi-system registry with shared meta-cycle** — once `convert` exists (see `system-conversion.md`), tonalpohualli ↔ xiuhpohualli conversion within a Calendar Round is a non-trivial worked example. Famously every day in a 52-year period has a unique (tonalpohualli, xiuhpohualli) pairing.

## Open questions

- Better Nahuatl name for `:tonalpohualli-cycle`? Convention may not exist since the count was traditionally not numbered.
- Should `:xiuhmolpilli` start at 1 or 0? The first Calendar Round in a numbering scheme is conventionally "1", but if used as a counter from an arbitrary epoch, 0-indexed is cleaner. Pick when there's a use case.
- Tonalpohualli days are conventionally written as `(number) (day-sign)` e.g. "1 Cipactli". The `:teyollia` / `:tonalli` units capture this, but ordering matters for display: number first, then sign. Format vec for IO would be `[:teyollia " " :tonalli]`.
