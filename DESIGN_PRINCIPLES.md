# Design Principles

## Purpose

A personal, fun project to predict Formula 1 race results across each race weekend. Built as a learning exercise in data science and ML, with Claude AI as a development aid. Not production-grade — simplicity and usability matter more than sophistication.

---

## Prediction Cadence

Predictions are made multiple times across a race weekend, with each stage using the data available at that point:

| Stage | When | Key Data Available |
|---|---|---|
| Pre-weekend | Before FP1 | Historical, season standings, circuit history |
| Post-practice | After FP1/FP2/FP3 | Practice lap times, long-run pace, tyre data |
| Post-qualifying | After quali | Grid positions, quali gaps, sector times |
| Race-day | Just before lights out | Grid, weather forecast, tyre choices |

Each run produces a snapshot prediction. All snapshots for a weekend are saved together so they can be compared.

---

## Interface

The tool is driven by an **interactive menu** in the terminal — no command-line flags to remember.

On launch, the user is presented with:

1. Select a race weekend (from the current season calendar)
2. Select the prediction stage (pre-weekend / post-practice / post-quali / race-day)
3. Review a summary of available data before confirming
4. Generate prediction

The interface should feel fast and lightweight — each menu step should be clear and require minimal input.

---

## Simplicity

- Prefer a small number of readable Python scripts over a complex package structure
- Use lightweight prediction logic (weighted scoring, simple ranking models) — no heavy ML frameworks
- Dependencies should be minimal and easy to install
- A new developer (or future me) should be able to understand any script within a few minutes of reading it

---

## Low Compute

- Predictions must run in seconds on a standard laptop
- No GPU, no cloud inference, no large model downloads
- Data fetching via FastF1 (with caching) to avoid redundant API calls
- If a computation takes more than a few seconds, it should say so

---

## Explainability

Every prediction output must show its working. The output includes:

- **Predicted finishing order** — ranked list of drivers with predicted position
- **Input data used** — every data point that influenced the prediction, clearly labelled with its source and freshness
- **Factor breakdown** — for each driver, which factors pushed them up or down and by how much
- **Confidence indicators** — flag where data is estimated, missing, or based on small samples
- **Key things to watch** — 3–5 narrative bullets on the factors most likely to shape the actual result (e.g. tyre strategy, weather risk, safety car likelihood)

The goal: after reading the output, it should be obvious *why* the model made the prediction it did.

---

## Output: PDF Report

Each prediction run saves a single PDF report. Reports are stored per race weekend and accumulate across the weekend so predictions can be compared.

### PDF Structure

1. **Header** — Race name, circuit, prediction stage, timestamp
2. **Predicted Result** — Ranked finishing order with driver, team, and predicted points
3. **Factor Breakdown** — Per-driver explanation of key inputs and their influence
4. **Input Data Summary** — Full table of raw data used (lap times, standings, weather, etc.)
5. **Key Things to Watch** — Short narrative bullets on the biggest uncertainty factors
6. **Weekend Comparison** *(if multiple predictions exist)* — Side-by-side view of how the predicted order has shifted across stages, with a note on what changed and why

### File Naming

```
reports/
  2025_R05_Monaco/
    pre_weekend.pdf
    post_practice.pdf
    post_qualifying.pdf
    race_day.pdf
```

---

## Data Transparency

- Every data point in the output is labelled with its source (e.g. FastF1, manual entry)
- If data is unavailable for a stage, the model notes what is missing and how it compensated
- No silent defaults — if an assumption is made, it is stated

---

## What This Project Is Not

- Not a betting tool
- Not optimised for accuracy at the cost of understandability
- Not built to scale beyond personal use
- Not trying to compete with professional F1 analytics
