# Use cases

> **Status: brainstorm + honest assessment.** Goal of this doc is to figure out which use case the library should be *for*, not enumerate every possible application. Library currently lacks a wedge — the abstraction is general, but generality alone doesn't motivate adoption.

## The honest diagnosis

unit-map is a meta-library: a framework for building positional-unit arithmetic for arbitrary domains. That's powerful but **structurally hard to adopt** because:

- **No single domain it dominates.** Dates → java.time / tick already won. Semver → dedicated libs. Genomic coords → bioinformatics libs. Each domain has a defending incumbent.
- **Generality requires the user to do the modeling.** Adoption cost = "learn the abstraction + design useqs + register systems" before getting any value. java.time gives you `LocalDate.now()` in one line.
- **The pitch ("composable unit systems") is too abstract.** Demos with calendars feel like "yet another date library, but harder to use".

The library is genuinely interesting; it just hasn't found a use case where its leverage > the cost of buying into the abstraction. **Pick one killer wedge, build deep tooling around it, expand from there.** Trying to be everything to everyone is the trap.

## Use cases worth taking seriously

### Wedge candidates (pick one)

**1. Astronomy / aerospace time systems** ⭐ strongest candidate
- Leap seconds via dynamic useq consulting a published leap-second table (IETF / IERS Bulletin C). Side-effecting fetch + cache fits perfectly into a useq endpoint fn.
- Multi-body time: Earth UTC, Mars Sol (24h 39m 35.244s), Lunar Coordinated Time (NIST is actively standardizing for Artemis). Conversion between bodies = `convert` between systems.
- Julian Day Number, Modified Julian Day, Truncated Julian Day, J2000 epoch — astronomers convert between these constantly.
- Sidereal vs solar time.
- Ephemeris time / TAI / TT / TDB / TCG / TCB — alphabet soup of relativistic time scales NASA actually uses.
- **Why this wedge wins:** Existing libs handle UTC well but punt on everything else. Audience (space/astronomy/GPS) is small but underserved and vocal. Demos are vivid. The "calendar agnostic" pitch becomes "time-system agnostic" which is a much sharper value prop.
- **Risks:** Audience is small. Requires real domain knowledge to design useqs correctly. Performance matters (astronomical computations run in tight loops).

**2. Historical / cultural calendars** ⭐⭐ strong concept, smaller audience
- Mayan Long Count, Tzolkin, Haab.
- Aztec tonalpohualli + xiuhpohualli (already sketched in `aztec-calendar.md`).
- Hebrew, Islamic Hijri (with regional moon-sighting variants), Persian Solar Hijri, Ethiopian, Bahá'í, Hindu lunisolar, Coptic.
- Roman calendar (with Kalends/Nones/Ides — weird non-monotonic numbering).
- Revolutionary calendars (French Republican, Soviet 5-day week experiment).
- **Why interesting:** No mainstream lib handles these; java.time has minimal coverage. Demonstrates the "data not code" pitch perfectly.
- **Audience:** Academia (historians, ethnographers, religious institutions). Small but stable.
- **Risks:** Almost no commercial demand. Better as a *demo* of the wedge than the wedge itself.

**3. Fiscal / business calendars**
- Retail 4-4-5, 4-5-4, 5-4-4 calendars (NRF retail, ISO 8601 weeks).
- Broadcast calendar (Sunday-start months, used for ad sales).
- Academic year (varies by institution: quarters, semesters, trimesters, terms).
- Fiscal year offset from calendar year (US Govt FY starts Oct 1, UK Apr 6, etc.).
- 52/53-week years.
- **Why interesting:** Real commercial demand. Existing tools are awkward (Excel formulas, custom code per company).
- **Risks:** Boring. Hard to make demos exciting. java.time has *some* support; users may be tolerant of pain.

### Non-time domains where the shape fits

**4. Music notation**
- Beat / measure / movement with time signatures (4/4 vs 3/4 changes the modulus per measure).
- MIDI tick / pulse / quarter-note conversions.
- Tablature positions.
- Strong fit; existing libs are deeply domain-specific and inflexible.

**5. Genomic coordinates**
- Chromosome / position, with chromosome-specific lengths (just like `days-in-month`).
- Convert between assemblies (hg19 ↔ hg38), genomic ↔ transcript ↔ protein coordinates.
- Walk regions, compute distances, intersect intervals.
- Strong fit. Existing tools are Python/R-centric; a Clojure-side option could win for JVM-shop bioinformatics.

**6. Sports scoring**
- Cricket: overs/balls (variable per format), wickets, partnerships.
- Tennis: sets/games/points with tiebreaks (point useq changes mid-match).
- Boxing/MMA: rounds + time within round.
- Niche but vivid. Good for "look how easy it is" demos.

**7. Tabletop / board game state**
- Turn / round / phase / sub-phase counters with phase-dependent rules.
- Action point economies with custom carries.
- Useful for game implementations and rules-engine systems.

**8. Document positions**
- Chapter / section / paragraph / sentence / word.
- Walk by sentence, count words across chapters, format as "Ch.3 §2 ¶5".
- Real use in publishing tooling, ebook readers, citation management.

**9. Memory / coordinate addressing**
- Segment:offset, page:frame:offset for memory.
- Spreadsheet cells (A1, AA1 — letters with carry: Z+1 = AA, ZZ+1 = AAA).
- Pixel/screen coordinates with sub-pixel positioning.
- Exact shape fit; rarely needed but bulletproof when needed.

**10. Pre-decimal currency**
- £/s/d (12 pence per shilling, 20 shillings per pound).
- Historical accounting / numismatic / genealogical use.
- Fun demo, not a serious use case.

**11. Version numbering**
- Semver, calver, custom dotted hierarchies.
- Compare and increment fits perfectly.
- Multiple existing libs solve this; unit-map's value would only be in shops *already using* unit-map for other things.

**12. Phone number plans**
- Country / area / exchange / subscriber.
- Mostly a parsing/validation problem, less arithmetic. Weak fit.

## Use cases that sound interesting but probably aren't

- **Address hierarchies** — ordering and arithmetic don't really apply. "Next house" isn't well-defined globally.
- **SI unit conversion** — multiplicative, not positional. Wrong shape.
- **Generic hierarchical IDs (Dewey decimal etc.)** — comparison works, arithmetic doesn't have a natural meaning.
- **UCUM (Unified Code for Units of Measure)** — unit-map covers only the **mixed-radix positional** subset (imperial length/weight, apothecary, historical measures). These exist in UCUM's machine-readable XML distribution and could be generated into registry defs programmatically. However, real-world UCUM conversions between systems are mostly **affine transforms** (Celsius ↔ Fahrenheit = offset + scale), which unit-map's base-conversion model can't express. Without affine conversion support, generated UCUM systems can do arithmetic within one system but can't interconvert — which is the main reason people reach for UCUM. Dubious until affine conversions are added. The bulk of UCUM (SI prefixes, compound units like m/s², logarithmic units) is dimensional analysis, not positional — wrong shape entirely.

## Specific candidates user mentioned

### Leap seconds via side-effecting useq
**Verdict: yes, this is great.** It's the single most compelling demonstration of dynamic useq endpoints because it requires *external state* (the IERS leap second table). API sketch:

```clj
(def leap-seconds
  (memoize-with-ttl 1-day
    (fn [] (fetch-iers-bulletin-c))))

(def sec->min-with-leaps
  {:unit :sec :next-unit :min
   :useq #unit-map/useq[0 1 .. (fn [{:keys [year month day hour min]}]
                                  (if (leap-second-at? year month day hour min)
                                    60   ;; 0..60 inclusive = 61 seconds
                                    59))]})
```

The fact that a useq endpoint can be a fn that hits the network is **wild and demonstrably useful**. No other date library handles leap seconds correctly because they treat time as monotonically continuous. unit-map can model them as a first-class data fact.

Caveats: side effects in arithmetic functions are a footgun (cache aggressively, never block, fall back to last-known table). Document loudly.

### Earth/Moon/Mars time conversion
**Verdict: yes, vivid and fits the wedge.** Lunar Coordinated Time is being actively standardized (NIST + ESA, 2024+). Mars Sol time is already used by NASA mission control ("Mars time" parties during MSL/Perseverance landings). Multi-body conversion is *exactly* what `find-conversion` + `convert` was designed for — different planets are different systems sharing some anchor units (atomic seconds).

Possible scope:
- UTC, TAI, TT, GPS time (Earth atomic).
- Mars Coordinated Time (MTC), Mars Sol Date, Mission Sol (relative to landing).
- Lunar UTC (when standardized; sketch with current proposal).
- Conversion between any two via `convert`.

Demo value: very high. "unit-map: the only library that lets you convert a Martian sol to an Aztec tonalpohualli date" is genuinely funny and actually useful for a tiny audience.

## Recommendation

1. **Pick astronomy/aerospace as the primary wedge.** Leap seconds + multi-body time is a coherent story, has an underserved audience, and demos vividly. Build deep examples (`src/unit_map/systems/astro/`?) covering UTC/TAI/TT/Mars/Lunar with a leap-second-aware useq.
2. **Position historical calendars (Aztec, Mayan, etc.) as proof-of-concept demos**, not the main pitch. They show the abstraction works for arbitrary systems but don't drive adoption on their own.
3. **Mention non-time wedges (music, genomics, fiscal) in docs as "also possible"** — but don't try to be the lead implementation in those domains. Let community contribute if interested.
4. **Resist the urge to position the lib as "general unit system framework"** at the marketing level even if that's what it is technically. Lead with a concrete vivid use case ("write your own time system, including leap seconds and Mars time"); the generality becomes apparent once people read the source.
5. **Burnout note:** The lack of clear use case is *the* problem to solve before more code work. Picking the wedge unblocks everything else (`convert`, `:overflow` policies, optimization priorities) because the wedge tells you which corners actually matter.

## What to do next, ranked by leverage

1. **Build a leap-second-aware UTC system** as a reference example. End-to-end: useq with side-effecting endpoint, parse/format `2016-12-31T23:59:60Z`, add 1 second across the leap and verify correctness.
2. **Build Mars Sol Date conversion** as the second reference example. Demonstrates `convert` once it exists.
3. **Write a "why unit-map" README** centered on these two examples.
4. **Then return to the pure-engineering work** (plural deltas, bulk advance, overflow, `convert`) with the wedge use cases as the test/benchmark targets.
