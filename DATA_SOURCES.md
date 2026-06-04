# Data Sources & Schema

## Overview

This document describes the data sources used for predictions, how they are combined, and the known limitations of the model — including factors that are genuinely unpredictable and outside the scope of any data-driven analysis.

---

## Data Sources

### 1. FastF1 API
**Type:** Dynamic — fetched per session  
**Coverage:** 2018–present

The primary data source for all live race weekend data. FastF1 provides:

- Lap times, sector times, speed trap data
- Tyre compound per lap and tyre age (stint length)
- Car telemetry (throttle, brake, speed, gear)
- Pit stop timing and duration
- Session weather: track temperature, air temperature, humidity, wind speed/direction, rainfall
- Qualifying gaps and session classifications
- Driver and team identifiers

FastF1 caches data locally after first fetch, so repeat runs within a weekend are fast.

---

### 2. Jolpica API (Historical F1 Data)
**Type:** Dynamic — fetched as needed  
**Coverage:** 1950–present

The successor to the Ergast API. Used for historical baseline statistics:

- Race results and finishing positions
- Championship standings (drivers and constructors) at any point in the season
- Grid positions and qualifying results
- Pit stop records
- Circuit information (location, country, first race year)
- Lap time records per race (used for historical pace comparison)
- DNF / retirement records (used for reliability analysis)

---

### 3. Open-Meteo
**Type:** Dynamic — fetched pre-session  
**Coverage:** Global, historical + 7-day forecast

Used for pre-session weather forecasting. No API key required. Weather analysis covers four distinct dimensions — precipitation is the least of them.

**Track temperature**
Track surface temp is typically 15–20°C above air temp and is the primary driver of tyre degradation. It also changes through a race: sessions spanning an afternoon-to-evening window (Abu Dhabi, Qatar) see track cooling mid-race that shifts tyre behaviour and viable strategy windows. Track temp forecasts are the primary input to the Tyre Degradation Model.

**Air / atmospheric temperature**
Hotter ambient air forces wider cooling configurations on the car, affecting drag and downforce balance. This varies by constructor — some PU and chassis combinations are more sensitive to high ambient temperatures than others. Air density also affects turbocharger efficiency at peak demand. Humidity compounds engine cooling demands and driver heat stress.

**Driver heat stress**
Cockpit temperatures can exceed 50°C in high-heat conditions. Some drivers demonstrably underperform relative to car pace at hot circuits — this is tracked historically and combined with ambient temperature forecasts to flag heat stress risk per driver. Extreme heat events (sustained track temp above ~55°C) are flagged explicitly.

**Wind**
Wind direction and speed are captured because at exposed circuits (Baku, Jeddah, Spa) they materially affect corner behaviour and straight-line speed. Headwind/tailwind on the primary straight affects top speed and fuel consumption differentials between teams.

**Precipitation**
Wet race probability per session window, scenario probabilities (dry / mixed / wet), and historical precipitation patterns at this circuit and date.

Note: FastF1 provides actual measured weather during a live session. Open-Meteo is used for pre-session forecasting only.

---

### 4. Circuit Characteristics (Static)
**Type:** Static — manually curated JSON file  
**Coverage:** All current calendar circuits

A hand-maintained reference file for circuit properties that rarely change and are not available programmatically. Includes:

- Circuit type: permanent, street, hybrid
- Tarmac abrasiveness rating (low / medium / high)
- Dominant corner type: high-speed, medium-speed, slow/traction
- Number of DRS zones
- Historical overtaking difficulty rating
- Track evolution sensitivity (how much the surface improves across a weekend)
- Lap length and number of laps
- Notable quirks (e.g. Monaco tunnel, Baku walls, Spa weather variability)

This file is updated manually at the start of each season or when a circuit is resurfaced.

---

### 5. Tyre Degradation Model
**Type:** Derived — built from FastF1 historical data + Circuit Characteristics

A per-circuit, per-compound degradation characterisation built from historical race data. Captures the relationship between track conditions and tyre behaviour:

**Input factors:**
- Track temperature at time of stint (from FastF1 session weather)
- Tyre compound and age (laps on set)
- Circuit abrasiveness and corner loading type (from static circuit file)
- Fuel load proxy (lap number in race)
- Track evolution stage (early vs. late in weekend)

**Derived outputs:**
- Expected lap time delta per lap on each compound at a given temperature band
- Degradation curve per circuit per compound (e.g. soft at 50°C at Barcelona degrades at ~0.08s/lap)
- Cliff identification: the lap range at which a compound typically drops off sharply

**Notes:**
- Historical analysis limited to last 3 seasons to account for Pirelli compound changes year-to-year
- Flagged as lower confidence at circuits with limited recent history (new venues, resurfaced tracks)
- Driver-specific deg tendencies (some drivers are notoriously hard or easy on tyres) overlaid as a modifier

---

### 6. Team Upgrade Tracker
**Type:** Static — manually curated, updated each race weekend  
**Coverage:** Current season

Tracks which teams are bringing upgrades, when, and how successful they proved to be:

- Upgrade announced (Y/N) for the weekend
- Components affected (floor, front wing, rear wing, sidepod, suspension)
- Performance delta observed in first session after introduction (from FastF1 pace analysis)
- Running success rate: upgrades that delivered measurable pace gain vs. those that did not or were reverted
- Pattern analysis: are certain teams consistently effective or ineffective at translating upgrades into pace?

---

### 7. Team Strategy Profile
**Type:** Derived — built from Jolpica + FastF1 historical pit data  
**Coverage:** Rolling last 2 seasons

Characterises how each team approaches race strategy, updated as the season progresses:

- Preferred pit window timing (early, standard, late)
- Undercut vs. overcut tendency
- One-stop vs. multi-stop preference by circuit type
- Safety car / VSC response pattern (do they pit or hold position?)
- Strategy consistency: do they commit to a plan or react to others?
- How their strategic approach has evolved across the current season

---

### 8. Driver Profile & Track Relationships
**Type:** Derived (historical stats) + Static (manual annotations)  
**Coverage:** Current grid

Per-driver analytical profile combining data-derived statistics with manually noted context:

**Data-derived:**
- Historical average finishing position at each circuit
- Qualifying vs. race pace delta per circuit type
- Overtaking rate: positions gained/lost from grid to finish
- Wet weather performance rating (derived from results in wet/mixed sessions)
- Tyre management tendency (from FastF1 stint length and deg data)
- DNF rate by cause (driver error vs. mechanical)

**Manually annotated:**
- Home race flag (home country or region)
- Circuits of personal significance (first win, home fan base, career milestone venue)
- Known difficult circuits for the driver (historically underperforms relative to car pace)
- Current form indicator: momentum across last 3–5 races

**Emotional / psychological layer:**  
Some drivers demonstrably perform differently at circuits that carry personal significance — better at home races, or conversely, carrying psychological weight from past incidents. This is flagged as a qualitative modifier in predictions rather than a hard number, given how difficult it is to quantify reliably.

---

### 9. Reliability & DNF Tracker
**Type:** Derived — built from Jolpica historical data  
**Coverage:** Current season + last 2 seasons

Tracks mechanical reliability patterns to apply a finishing probability modifier:

- DNF rate per constructor (current season and rolling average)
- DNF rate per engine supplier
- Component failure patterns (gearbox, PU, hydraulics, suspension)
- Drivers approaching power unit component limits (flag for potential grid penalty)
- Recent reliability trajectory: improving or worsening across the season

---

### 10. Safety Car Probability
**Type:** Derived — built from Jolpica historical data  
**Coverage:** All current calendar circuits

Per-circuit historical safety car and virtual safety car (VSC) deployment rates:

- Full safety car probability (%)
- VSC probability (%)
- Average number of SC/VSC deployments per race
- Typical lap of first deployment (early vs. late race tendency)

Used to weight strategy scenarios — a high-SC circuit makes one-stop strategies riskier and can invalidate pre-race strategy predictions.

---

### 11. Sprint Weekend Flag
**Type:** Static — derived from season calendar

A simple flag indicating whether a weekend is a sprint format. Sprint weekends have:
- Reduced free practice (FP1 only before qualifying)
- Less practice data available for pace analysis
- A sprint race result that can be used as additional pace reference

All predictions generated at a sprint weekend are automatically flagged with reduced confidence in practice-derived inputs.

---

### 12. FIA Regulatory & Penalty Patterns
**Type:** Derived — built from FIA published stewards' decisions + community datasets  
**Coverage:** Rolling last 3 seasons

The FIA's enforcement approach shifts year to year — different regulatory focuses, varying penalty severity for similar incidents, and changing steward panels all introduce a layer of unpredictability that can materially affect results. This source does not attempt to model favouritism, but instead tracks objectively measurable patterns.

**What is tracked:**
- Penalty frequency per driver and constructor (current season and rolling average)
- Penalty severity for categorised incident types (e.g. racing contact, track limits, unsafe release) — are similar incidents receiving consistent outcomes?
- Investigation-to-penalty conversion rate per team: some teams are investigated frequently but penalties rarely stick; others convert at higher rates
- Seasonal regulatory focus: each season the FIA tends to prioritise enforcement of specific infraction types (track limits, flexi-wings, pit lane speed, etc.) — characterise the current season's focus area
- Post-race technical scrutineering outcomes: how often do teams face DSQ risk and which teams carry elevated exposure
- Steward panel composition where available: decision patterns can vary across different panels

**Derived outputs:**
- Per-driver penalty risk flag: drivers on warning thresholds or with elevated investigation rates this season
- Per-team scrutineering risk flag: teams with recent technical investigations
- Current season enforcement focus note: a short label describing what the FIA is prioritising (e.g. "track limits at high-speed corners", "flexi-floor sensitivity")
- Circuit-specific risk flag: where the current enforcement focus intersects with a specific circuit's characteristics

**Data sources:**
- FIA official stewards' decisions (published as PDFs on fia.com — requires parsing or community-compiled datasets)
- Community-maintained GitHub datasets that compile FIA decisions into structured form
- Manual curation for current season where automated sources lag

**Important caveat:**  
This analysis describes statistical patterns in outcomes, not intent. Outputs are framed as risk indicators only — "Driver X has a higher-than-average penalty conversion rate this season" — never as claims about bias. Treat this as one factor among many, weighted modestly.

---

### 13. Rookie Driver Profiles
**Type:** Derived (junior formula data) + Static (manual annotations)  
**Coverage:** Any driver in their first or second F1 season

Rookie drivers have no F1 historical data, so predictions for them carry inherently higher uncertainty. This source defines how junior formula data is used as a prior, and how that prior is updated as F1 data accumulates.

**Junior formula baseline (F2 / F3):**
- Qualifying pace relative to field — translates reasonably well to F1 single-lap ability
- Championship position and points trajectory
- Race pace and tyre management — used cautiously; junior tyres behave differently to F1
- Wet weather performance — useful signal if the junior series had mixed-condition races
- Incident and DNF rate — early indicator of racecraft under pressure
- Circuit overlap: where the junior series calendar shares circuits with F1, circuit-specific performance is weighted significantly more heavily than overall stats

**F1 transition weighting:**
- Pre-season and early rounds: junior formula data is the primary input, with wide uncertainty bands applied to all outputs
- After 4–5 F1 races: real F1 data begins taking over as the primary signal; junior data weighting reduces race by race
- After a full season: treated as an experienced driver with a full F1 profile; junior data is archived

**Teammate delta as primary metric:**  
Absolute rookie results are heavily influenced by car quality and are difficult to interpret in isolation. The primary performance signal is pace relative to their experienced teammate — this strips out car quality and gives a cleaner driver-vs-driver comparison. This is the most reliable indicator of a rookie's true level during their first season.

**Car quality context:**  
All rookie outputs explicitly note the car context. A rookie finishing P14 in a backmarker team and a rookie finishing P14 in a top team are completely different situations. Predictions are normalised against expected car performance, not absolute grid position.

**Manually annotated:**
- First F1 season flag
- Circuits where the driver has prior experience from junior formulae
- Known strengths or weaknesses carried over from junior categories
- Team environment assessment: some teams develop rookies effectively; others provide limited support infrastructure

**Known limitations for rookies:**
- How quickly a specific driver adapts to F1 tyre behaviour varies enormously and is not predictable from junior data
- Psychological response to the step up in media pressure and championship stakes is unquantifiable
- Some drivers adapt faster than any data would suggest; others take longer than expected

---

## Known Limitations

These are factors that are genuinely outside the reach of any data-driven prediction model. They are documented here so that predictions are interpreted with appropriate honesty about what the model cannot know.

### Driver Errors and Incidents
Random driver mistakes — misjudged braking, wheel-to-wheel contact, track limits violations — are by definition not predictable from historical data. A driver's past error rate gives a rough tendency, but specific incidents cannot be anticipated. A driver can be on course for a podium and put it in the wall on lap 3.

*Example: Oscar Piastri qualifying off the pace in Australia 2024 following two separate crashes. No model predicted that.*

### First-Lap Chaos
The opening lap of a race is the highest-risk period for contact and position swings. Grid density, track characteristics, and driver aggression combine in ways that are highly chaotic. Predictions assume a reasonably clean start unless there is specific evidence otherwise.

### Psychological Blockers
Some drivers carry negative associations with specific circuits or race situations that measurably affect performance but cannot be cleanly quantified. A driver under intense championship pressure, at a circuit where they have historically struggled, facing a must-win situation behaves differently to the same driver in a neutral context. The driver profile attempts to flag known patterns, but the magnitude is not reliably predictable.

### Freak Weather Events
Probabilistic weather forecasts are incorporated, but sudden localised weather changes mid-race — a sudden downpour at one sector, a drying track that ruins a wet-tyre call — introduce outcomes that diverge completely from pre-race strategy models.

### Stewards' Decisions
Penalties for track limits, unsafe releases, driving infringements, and post-race investigations can materially change finishing positions. Stewarding consistency is itself inconsistent and cannot be modelled.

### Strategic Game Theory
Teams react to each other in real time. A strategy that looks optimal in isolation may be undermined by a competitor's unexpected call, triggering a chain of reactive decisions. The strategy profile characterises tendencies but cannot simulate real-time multi-team interactions.

### Mechanical Failures Without Pattern
The reliability tracker captures trends, but a random hydraulic failure or gearbox issue on a car with an otherwise clean record is not foreseeable. High-reliability teams can and do retire unexpectedly.

### Red Flag Stoppages
A red flag resets the race in ways that can completely override pre-race strategy — allowing tyre changes under red flag conditions, compressing field gaps, and forcing teams into decisions not in any pre-race plan.

---

## Source Summary

| Data | Source | Type | Update Frequency |
|---|---|---|---|
| Session telemetry, lap times, tyres, in-session weather | FastF1 | API | Per session |
| Historical results, standings, pit stops, DNFs | Jolpica API | API | Per race |
| Pre-session weather forecast | Open-Meteo | API | Per session |
| Circuit characteristics | Static JSON | Manual | Start of season / resurfacing |
| Tyre degradation model | Derived (FastF1 + circuit data) | Computed | Rebuilt each weekend |
| Team upgrade tracker | Manual | Manual | Each race weekend |
| Team strategy profile | Derived (Jolpica + FastF1) | Computed | Rolling, updated each race |
| Driver profiles & track relationships | Derived + Manual | Mixed | Rolling, updated each race |
| Reliability & DNF tracker | Derived (Jolpica) | Computed | Updated each race |
| Safety car probability | Derived (Jolpica) | Computed | Static per circuit, reviewed annually |
| Sprint weekend flag | Calendar | Static | Set at season start |
| FIA penalty & regulatory patterns | FIA decisions + community datasets | Derived + Manual | Updated each race |
| Rookie driver profiles | F2/F3 data + manual annotations | Derived + Manual | Updated each race; transitions to F1 actuals |
