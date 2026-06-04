# Process & Context Management

## What Is Dreaming?

When the prediction pipeline runs, it does something worth naming properly. It takes fragments of information from eleven independent sources — pace data, weather, tyre behaviour, team tendencies, driver history, reliability patterns — and synthesises them into a single coherent picture of how a race might unfold. It does not look up an answer. It constructs one, from the ground up, every time.

That process is dreaming.

The term matters because it shapes how you should read the output. A dream is not a fact. It is the most coherent story the available data can tell at a given moment. A pre-weekend dream is built from historical fragments and forecasts. A race-day dream is built from a full weekend of evidence. Both are dreams. The race-day one is just better informed.

**Re-dreaming** means discarding the previous dream entirely and building a new one from raw data. Not updating the old prediction — replacing it. The old dream was not wrong; it was the best story the data could tell at the time. But you do not want to be anchored to it when making the next one. Starting fresh is what keeps each prediction honest.

This is why the agents are stateless. No agent knows what it scored a driver last time. The Orchestrator does not remember its previous output. The only continuity between runs is the data itself — which has grown — and the reports folder, which holds the previous dreams for comparison.

---

## Weekly Process

### Before the Weekend

Before FP1, do a brief manual data review:

- **Team upgrade tracker** — check Autosport, The Race, or team press releases for any announced upgrades. Log them in the static tracker file with component and confidence level (confirmed / rumoured).
- **Driver notes** — any significant news since the last round: injuries, team tensions, contract situations, public comments about the circuit. Add brief notes to the driver profile annotations.
- **FIA focus** — any new directives or regulatory clarifications issued since the last race. Update the enforcement focus label if relevant.
- **Rookie transition check** — if a rookie has crossed the 4–5 race threshold, confirm their weighting has shifted toward F1 actuals.

This takes 10–15 minutes. It is the only manual input required before the weekend begins.

---

### During the Weekend

Run a fresh dream at each prediction stage. Each run is independent.

| Stage | When to run | What's new |
|---|---|---|
| **Pre-weekend** | Evening before FP1 | Historical baseline only. Broad prediction, wide uncertainty. |
| **Post-practice** | After FP2 (or FP1 on sprint weekends) | Session pace data, long-run tyre data, first upgrade delta. |
| **Post-qualifying** | After qualifying, before parc fermé | Grid positions, qualifying gaps, sector analysis. Highest pre-race confidence. |
| **Race-day** | Morning of race, after tyre selections confirmed | Final weather forecast, tyre choices declared, any overnight news. |

Each run produces a PDF saved to `reports/YEAR_RXX_CircuitName/`. Do not delete earlier dreams — the comparison across the weekend is part of the value.

**How to read each dream:**

Read the predicted result alongside the factor breakdown and input data summary, not instead of them. The ranked list is the conclusion. The rest of the report is the argument. If the argument does not feel right, trust that instinct and note it — that is useful feedback for the end-of-season review.

Pay attention to the flags. Elevated uncertainty, data gaps, rookie labels, and ceiling context notes are not decoration. They are the model being honest about where it is confident and where it is guessing.

---

### After the Race

Before the next round, spend a few minutes on a brief post-race review. This is not about recrimination — it is about keeping the underlying data honest.

**Check:**
- Did any agent's output diverge significantly from reality? Note which one and why (car was faster than expected, unexpected retirement, strategy call that was out of character).
- Were any flags vindicated? (Safety car probability flagged high and a SC happened; heat stress flag fired and a driver struggled.)
- Do any manual data sources need updating? (Upgrade success/fail logged, strategy characterisation updated based on the race, reliability tracker updated for any retirements.)

Do not adjust weights based on one race. Log the observation. Move on.

---

## Cross-Weekend State: What Carries Forward and What Doesn't

### What carries forward (data, not memory)

The underlying data evolves naturally each race and this is the only cross-weekend learning the model does:

- Team strategy profiles update with each race result
- Driver form rolling average shifts forward one race
- Reliability tracker updates with any retirements or component failures
- Upgrade tracker logs the observed delta from the previous weekend
- FIA patterns tracker adds the latest penalty and investigation data
- Rookie weighting shifts progressively toward F1 actuals

None of this requires manual intervention beyond the post-race review above. It happens because the data sources are updated.

### What does not carry forward (predictions and weights)

The prediction itself has no memory. Running post-qualifying does not know what pre-weekend said. The Orchestrator weights are fixed defaults throughout the season.

**Why weights stay fixed in-season:**

A season is roughly 20 races. That is not a large enough sample to draw statistically confident conclusions about which agents are most predictive. One unusual race — a Monaco, a wet Spa, a red-flagged street circuit — can skew a single-season accuracy log in ways that do not reflect genuine model quality. Adjusting weights mid-season based on a handful of data points risks fitting the model to noise rather than signal.

The accuracy log is kept as a record. Read it. Notice patterns. But do not let it automatically change anything during the season.

---

## End-of-Season Review

At the end of each season, before the new season begins, run a structured review:

1. **Accuracy log analysis** — across all race weekends, which agents were most and least predictive? Were there patterns (e.g. historical pace agent consistently underestimated a specific team's development pace)?

2. **Flag validation** — how often did flags fire accurately? Were any flags consistently wrong (false positives)?

3. **Manual data quality** — which manually maintained sources drifted or became stale? What would have made them easier to keep up to date?

4. **Weight review** — based on the accuracy analysis, consider whether any agent weights should be adjusted for the coming season. Changes should be deliberate, documented, and conservative. The default weights are a reasonable baseline; deviate from them only with clear evidence.

5. **New season setup** — update the static circuit file for any circuit changes or resurfacing, reset the team upgrade tracker, carry forward any driver profile annotations that remain relevant, and flag any new rookies for the rookie handler.

---

## On Bias and Trust

The model will sometimes tell you something you do not want to hear. A driver you rate highly will be predicted sixth. A team you think is being underestimated will be given a modest score. A strategy call you think is obvious will not appear in the output.

Resist the urge to override it.

The point of building a structured, multi-agent prediction system is to counterbalance the intuitions that are hard to separate from bias. The model does not know who you support. It does not have opinions about team management or championship politics. When its output surprises you, the right response is to read the factor breakdown and understand why — not to assume it is wrong.

If after reading the reasoning you still think the model is missing something real, the right action is to update the underlying data (a driver note, an upgrade log, a strategy annotation) and re-run. Not to mentally adjust the output upward for the driver you think deserves better.

The dream should be trusted until the data changes.
