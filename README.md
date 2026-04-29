# Bexley Crime Hotspot Predictor — UI Summary
___

## Live Demo
[Click here to open the interactive UI](https://callumhudson0.github.io/Thompson-Final-Project/bexley_safety_ui-2.html)

---

## 1. What does the notebook do?

The notebook (`Bexley_Crime_Hotspot_Project_Final_Clean.ipynb`) follows a standard classification pipeline:

1. **Data source** — Metropolitan Police monthly CSV files (`metropolitan-street.csv`) covering March 2023 to February 2026, filtered to the London Borough of Bexley.
2. **Target engineering** — Incident rows are aggregated to an `LSOA name × Year × MonthNum` grain. An LSOA-month is labelled `Hotspot = 1` if its crime count meets or exceeds the **75th percentile** of all monthly crime counts across Bexley.
3. **Features** — Three predictors are used: `LSOA name` (categorical, one-hot encoded), `Year` (numeric), and `MonthNum` (numeric).
4. **Preprocessing** — A `ColumnTransformer` pipeline applies `SimpleImputer → OneHotEncoder` to the categorical feature and `SimpleImputer` (median) to the numeric ones.
5. **Models compared** — Logistic Regression, Decision Tree, and Random Forest, evaluated on Accuracy, Precision, Recall, F1, and Cohen's Kappa.
6. **Tuning** — A `GridSearchCV` (3-fold, scoring on F1) tunes the Random Forest over `n_estimators`, `max_depth`, `min_samples_leaf`, and `class_weight`.
7. **Interpretation** — Feature importances are extracted from the tuned RF, and a second map visualises LSOA centroids sized and coloured by total crime count.

---

## 2. How the UI was built from the notebook

### What was carried across directly

| Notebook element | UI equivalent |
| --- | --- |
| Three model features: `LSOA name`, `Year`, `MonthNum` | The three input controls in the sidebar |
| Hotspot defined as ≥ 75th percentile of monthly crime count | Used to calibrate the base risk index for each LSOA |
| Random Forest as the primary model | Described in the code panel; scoring weights reflect RF feature importance ordering |
| `LSOA name` has highest feature importance | Assigned 60% weight in the composite score |
| `MonthNum` captures seasonality | Assigned 25% weight; month adjustments follow summer-peak pattern consistent with Met Police data |
| `Year` reflects temporal trend | Assigned 15% weight; slight upward safety trend modelled from 2023 → 2026 |
| Bexley borough scope | Enforced via a point-in-polygon boundary check on every map click |

### Code panels in the UI

The two collapsible code sections in the sidebar display **the actual code from your notebook**, lightly syntax-highlighted for readability. No logic was altered — these are direct reproductions of:

* The hotspot engineering cell (aggregation + 75th-percentile threshold)
* The preprocessing pipeline, baseline RF configuration, and `GridSearchCV` tuning block

---

## 3. Where approximations were made

### 3.1 — LSOA risk scores are approximated, not model-predicted

The notebook trains a Random Forest that predicts `Hotspot = 0/1` for a given `LSOA × Year × MonthNum`. The **actual fitted model is not embedded in the HTML**. Embedding it would require serialising the trained scikit-learn pipeline (e.g. via `joblib`) and running Python in the browser, which is not feasible in a standalone HTML file.

Instead, each LSOA is assigned a **base risk index** (0–100) that approximates its historical hotspot rate from the notebook's `hotspot_rank` output. Higher values mean safer (lower hotspot probability). These are calibrated estimates — they reflect the relative ordering the notebook reveals, but they are not the exact predicted probabilities from the fitted model.

**To make this fully accurate**, the base risk index for each LSOA should be replaced with the real `HotspotRate` value from the final cell of your notebook:

```
hotspot_rank = (
    monthly_hotspots_with_code
    .groupby(['LSOA code', 'LSOA name'])['Hotspot']
    .mean()
    .reset_index(name='HotspotRate')
    .sort_values('HotspotRate', ascending=False)
)
```

The `HotspotRate` column (a value between 0 and 1) can be converted to a safety score as `round((1 - HotspotRate) * 100)` and placed directly into the `risk` field of the `LSOAS` array in the HTML.

### 3.2 — LSOA centroids are approximate

The `lat`/`lng` coordinates assigned to each LSOA in the UI are **estimated centroids**, not the true ONS LSOA polygon centroids. They are close enough to place pins in the correct general area, but the "nearest LSOA" snapping on map click will not respect actual LSOA boundaries — it uses straight-line distance to centroids only.

For precise LSOA boundary matching, the ONS Open Geography Portal provides the official LSOA boundary GeoJSON, which could be loaded into Leaflet to enable proper spatial joins.

### 3.3 — Borough boundary polygon is simplified

The Bexley borough outline drawn on the map is a **56-point approximation** traced from ONS data. It correctly excludes most out-of-borough clicks but may be slightly inaccurate near the edges — particularly the Thames riverbank (north) and the Bromley border (south). The official boundary is far more detailed.

### 3.4 — Scoring weights are interpretive

The composite score formula:

```
score = (LSOA risk × 0.60) + (season score × 0.25) + (year score × 0.15)
```

The 60/25/15 split was chosen to reflect the **ordering of feature importance** that the notebook's Random Forest typically produces (LSOA name dominates, month adds meaningful signal, year has least weight). These are not the literal importance values extracted from the fitted model — those would require running `feature_importances_` from the saved `best_rf` object.

### 3.5 — Day of week is absent

The notebook uses `MonthNum` but not day of week, because the Metropolitan Police street-level CSV does not record the time or day of individual incidents — only the month. The UI therefore cannot include day-of-week as an input, and any real-world differences between weekday and weekend risk are not captured.

---

## 4. What was not modified

* The problem framing (binary hotspot classification at LSOA-month level) was not changed.
* The 75th-percentile hotspot threshold was not changed.
* The feature set (`LSOA name`, `Year`, `MonthNum`) was not extended or reduced.
* The Random Forest hyperparameters shown in the code panel match the notebook exactly.
* No additional data sources were introduced.

---

## 5. Notable future implementations

* **Embed real hotspot rates** by exporting `hotspot_rank` from the notebook as JSON and loading it into the HTML, replacing the approximated risk scores.
* **Use official LSOA boundaries** from the ONS Open Geography Portal for accurate spatial snapping and a precise borough outline.
* **Add model probability output** by serialising the fitted `best_rf` pipeline with `joblib` and serving predictions via a lightweight Flask or FastAPI backend, allowing the UI to call the real model.
* **Extend features** if a richer dataset becomes available (e.g. time of day from police CAD logs, weather API, footfall data).

---

## 6. AI Disclosure

Portions of this project were developed with the assistance of AI tools:

- **User Interface (`bexley_safety_ui.html`)** — The interactive front-end was built with the assistance of [Claude](https://claude.ai) (Anthropic). Claude was used to help scaffold and refine the HTML, CSS, and JavaScript, including the Leaflet map integration, sidebar layout, and composite scoring logic.
- **Dataset (`metropolitan-street.csv`)** — The CSV data used in this project was sourced and/or structured with assistance from [OpenAI Codex](https://openai.com/blog/openai-codex). Codex was used to help query, filter, and prepare the Metropolitan Police street-level crime data for use in the notebook pipeline.

All AI-assisted outputs were reviewed and verified by the project author. The analytical decisions — including model selection, feature engineering, and hotspot thresholding — were made independently.

---

## 7. File reference

| File | Description |
| --- | --- |
| `Bexley_Crime_Hotspot_Project_Final.ipynb` | Original notebook — data pipeline, EDA, RF model |
| `bexley_safety_ui.html` | Standalone front-end UI — open in any browser, no server needed |
| `UI_SUMMARY.md` | This document |
