# Subagent Schema

## Architecture Overview

Predictions are produced by a set of independent subagents, each responsible for one analytical domain. Subagents run in isolation — no agent sees another's output before producing its own. This prevents one strong signal (e.g. a dominant qualifying gap) from unconsciously anchoring the analysis of unrelated factors like reliability or strategy.

Once all subagents have completed, a central **Orchestrator** collects their structured outputs, applies configurable weights, and produces the final ranked prediction and PDF report.

```
┌─────────────────────────────────────────────────────────┐
│                     ORCHESTRATOR                        │
│  Collects outputs → applies weights → final prediction  │
└────────────────────┬────────────────────────────────────┘
                     │ receives from all agents below
     ┌───────────────┼───────────────────────────────┐
     │               │               │               │
  [Agent]         [Agent]         [Agent]         [Agent]
  Session         Historical      Weather         Tyres &
  Pace            Pace            Forecast        Strategy
     │               │               │               │
  [Agent]         [Agent]         [Agent]         [Agent]
  Driver          Team            Reliability     Safety Car
  Form            Upgrades        & DNF           Probability
     │               │
  [Agent]         [Agent]
  FIA / Penalty   Rookie
  Risk            Handler
```

Each agent is independently runnable and testable. If an agent fails to retrieve data or cannot produce a confident output, it returns a degraded response with a flag — the pipeline continues, noting the gap.

---

## Standardised Agent Output Schema

Every subagent returns a JSON object conforming to this structure:

```json
{
  "agent_id": "string",
  "prediction_stage": "pre_weekend | post_practice | post_qualifying | race_day",
  "circuit": "string",
  "season_round": "integer",
  "generated_at": "ISO8601 timestamp",

  "driver_scores": {
    "DRIVER_CODE": {
      "score": "float (0.0–1.0, higher = stronger predicted performance)",
      "delta": "float (positive = boosted vs. baseline, negative = penalised)",
      "confidence": "high | medium | low",
      "factors": ["list of short strings explaining the score"]
    }
  },

  "flags": [
    {
      "type": "warning | info | data_gap",
      "driver": "DRIVER_CODE or null for circuit-wide flags",
      "message": "string"
    }
  ],

  "agent_confidence": "high | medium | low",
  "confidence_reason": "string — why confidence is at this level",
  "data_sources_used": ["list of source names"],
  "data_freshness": "ISO8601 timestamp of most recent underlying data"
}
```

`score` is a normalised value used for relative ranking within the agent's domain — it does not represent a finishing position directly. The Orchestrator combines scores across agents using weights to produce the final ranking.

---

## Subagent Definitions

### Agent 1: Session Pace
**Domain:** Current weekend lap time and sector data  
**Sources:** FastF1  
**Available from:** Post-practice, post-qualifying, race-day

Analyses the raw pace data from the current race weekend sessions:

- Best and representative lap times per driver per session
- Sector time breakdown (where is pace being found or lost?)
- Long-run pace from practice (fuel-corrected estimate)
- Qualifying gap to pole and inter-driver gaps
- Pace trend across sessions (improving, stable, declining)

*Not available at pre-weekend stage — agent returns a `data_gap` flag and is excluded from that stage's weighting.*

---

### Agent 2: Historical Pace
**Domain:** Multi-year baseline performance at this circuit  
**Sources:** Jolpica API, FastF1 historical  
**Available from:** All stages

Establishes a historical baseline independent of the current weekend:

- Average qualifying position at this circuit per driver (last 3 seasons)
- Average finishing position at this circuit per driver
- Qualifying-to-race delta at this circuit (who typically improves or drops)
- Constructor historical pace at this circuit
- Circuit type affinity: driver/team performance indexed against circuits with similar characteristics

---

### Agent 3: Weather Forecast
**Domain:** Weather conditions and their strategic implications  
**Sources:** Open-Meteo (forecast), FastF1 (session actuals)  
**Available from:** All stages

- Current and forecast temperature, precipitation probability, wind
- Wet race probability for race day window
- Historical weather patterns at this circuit and date
- Scenario flags: dry / mixed / wet, and how likely each is
- Per-driver wet weather performance rating (fed from driver profile data)

Output includes a scenario weighting (e.g. 70% dry, 20% mixed, 10% wet) that the Orchestrator uses to blend scenario-specific predictions.

---

### Agent 4: Tyre & Strategy
**Domain:** Expected tyre behaviour and strategy windows  
**Sources:** Tyre Degradation Model, FastF1, Pirelli allocation data  
**Available from:** All stages (confidence increases post-practice)

- Expected degradation rate per compound at forecast temperature
- Viable strategy options for this circuit and expected conditions (1-stop, 2-stop, etc.)
- Optimal pit window ranges per strategy
- Per-driver tyre management tendency modifier
- Flags where forecast temperature diverges significantly from the historical norm this circuit was characterised at

---

### Agent 5: Team Strategy Profile
**Domain:** How each team is likely to behave strategically  
**Sources:** Derived strategy profile (Jolpica + FastF1 historical)  
**Available from:** All stages

- Each team's strategic tendency: early/standard/late pit windows, undercut/overcut preference
- Safety car response pattern
- Current season strategy evolution (is the team changing approach?)
- Interaction flags: teams that historically react to specific competitors' strategies

This agent scores drivers indirectly — a team with a strong, proactive strategy history gets a modest positive modifier; a team with a history of reactive or poor calls gets a negative modifier.

---

### Agent 6: Team Upgrades
**Domain:** Performance impact of upgrades brought to this weekend  
**Sources:** Team Upgrade Tracker (manual)  
**Available from:** All stages (pre-weekend if announced; post-practice with pace confirmation)

- Upgrade status per team: confirmed, rumoured, none
- Affected components
- Historical upgrade success rate for this constructor
- Pace delta observed in first session (post-practice only)
- Confidence flag: high if pace delta confirmed in session data; low if pre-weekend estimate only

---

### Agent 7: Driver Form & Track Relationships
**Domain:** Current driver form, track affinity, and psychological factors  
**Sources:** Driver Profile (derived + manual), Jolpica  
**Available from:** All stages

- Rolling form: average finish and points haul across last 3–5 races
- Track-specific historical performance delta vs. car expectation
- Circuit significance flag (home race, career milestone, historical incident)
- Psychological modifier: qualitative flag only — not a hard score adjustment
- Wet weather rating
- Overtaking and defending tendency at this circuit type

---

### Agent 8: Reliability & DNF Risk
**Domain:** Probability of each driver/constructor finishing the race  
**Sources:** Reliability & DNF Tracker (derived from Jolpica)  
**Available from:** All stages

- Finishing probability per driver (expressed as a modifier, not an absolute)
- Current season DNF rate per constructor
- Engine supplier reliability trend
- Power unit token / component limit flags (elevated grid penalty risk)
- Any known reliability concerns from practice (from FastF1 session alerts if available)

A low finishing probability applies a downward delta to a driver's score — a fast car that often doesn't finish should not be predicted confidently to podium.

---

### Agent 9: Safety Car Probability
**Domain:** Likelihood and timing of safety car or VSC deployment  
**Sources:** Safety Car Probability model (derived from Jolpica)  
**Available from:** All stages

- Full SC probability (%) for this circuit
- VSC probability (%)
- Typical deployment lap range
- Strategy impact flag: how much does a likely SC change the optimal strategy at this circuit?
- Interaction with Tyre & Strategy agent: a high-SC circuit shifts the viable strategy distribution

This agent does not score drivers directly. It produces probability values and flags that the Orchestrator uses to adjust the weight given to the Tyre & Strategy agent's output.

---

### Agent 10: FIA Regulatory & Penalty Risk
**Domain:** Penalty exposure and current regulatory focus  
**Sources:** FIA Penalty Patterns (derived + manual)  
**Available from:** All stages

- Per-driver penalty risk flag (based on current season investigation/conversion rate)
- Per-team scrutineering risk flag
- Current season enforcement focus label
- Circuit-specific risk flag where enforcement focus intersects with circuit characteristics
- Post-race result change probability (elevated for teams with recent technical investigations)

Outputs are risk indicators only. Applied as small modifiers and prominently labelled as such in the report — never a dominant factor in the prediction.

---

### Agent 11: Rookie Handler
**Domain:** Adjusted profiling for first and second-year drivers  
**Sources:** Rookie Driver Profiles (F2/F3 data + manual)  
**Available from:** All stages

- Identifies any driver in their first or second F1 season
- Applies appropriate uncertainty multiplier to all scores for that driver
- Uses F2/F3 baseline prior (with circuit overlap weighting) until sufficient F1 data exists
- Transitions weighting progressively from junior data to F1 actuals as races accumulate
- Teammate delta comparison: scales rookie scores relative to experienced teammate's agent outputs
- All rookie driver outputs in the final report are tagged with an elevated uncertainty label

*This agent modifies scores produced by other agents rather than generating independent scores. It acts as a post-processing layer for rookie drivers before outputs reach the Orchestrator.*

---

## Orchestrator

The Orchestrator is the main pipeline entry point. It:

1. Determines which agents are available for the current prediction stage
2. Runs all available agents (independently, no shared state between agents)
3. Collects all agent outputs
4. Applies the Rookie Handler as a post-processing pass
5. Applies per-agent weights to driver scores
6. Combines weighted scores into a final ranked prediction
7. Assembles justification text from agent `factors` fields
8. Passes the full output to the PDF report generator

### Agent Weights

Weights are configurable and stage-dependent. Example defaults:

| Agent | Pre-Weekend | Post-Practice | Post-Qualifying | Race-Day |
|---|---|---|---|---|
| Session Pace | — | 0.20 | 0.30 | 0.25 |
| Historical Pace | 0.35 | 0.20 | 0.15 | 0.10 |
| Weather | 0.10 | 0.10 | 0.10 | 0.15 |
| Tyre & Strategy | 0.15 | 0.15 | 0.15 | 0.20 |
| Team Strategy Profile | 0.10 | 0.10 | 0.10 | 0.10 |
| Team Upgrades | 0.10 | 0.10 | 0.05 | 0.05 |
| Driver Form & Track | 0.15 | 0.10 | 0.10 | 0.10 |
| Reliability & DNF | 0.05 | 0.05 | 0.05 | 0.05 |

Weights are reduced proportionally when an agent returns `low` confidence or a `data_gap` flag, and redistributed across remaining agents.

### Confidence Aggregation

The Orchestrator produces an overall prediction confidence level based on:
- How many agents returned `high` vs `low` confidence
- Whether any critical agents (Session Pace, Historical Pace) flagged data gaps
- Sprint weekend flag (automatically reduces overall confidence)
- Rookie driver count on the current grid

### Failure Handling

- If an agent fails to run entirely, the pipeline logs the failure, notes it in the report, and continues without that agent's scores
- If a majority of agents fail, the pipeline aborts and surfaces a clear error rather than producing an unreliable prediction
- All failures are visible in the report under a "Data Gaps" section

---

## File Structure (Planned)

```
f1_prediction/
  agents/
    session_pace.py
    historical_pace.py
    weather_forecast.py
    tyre_strategy.py
    team_strategy.py
    team_upgrades.py
    driver_form.py
    reliability.py
    safety_car.py
    fia_penalties.py
    rookie_handler.py
  pipeline/
    orchestrator.py
    weights.py
    report_generator.py
  data/
    cache/           # FastF1 cache
    static/          # Circuit characteristics JSON, manual trackers
    reports/         # Generated PDFs
  main.py            # Menu entry point
```
