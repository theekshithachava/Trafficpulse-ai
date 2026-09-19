# Urban Traffic Flow & Incident Intelligence

**NEURAX Hackathon 3.0 · Domain 1 – AI in Smart Cities · Checkpoint 1**

A software-only, advisory decision-support system for a city traffic control room. It reads the
organiser-provided traffic and road-network data, works out what is happening on the roads right now,
predicts the next 15–60 minutes, and recommends what to do — a quick diversion for a sudden jam, or a
tested road improvement for a jam that happens every day. **It only advises. It never controls anything.**

> Think of it as a *doctor for the road network*: check the patient → predict what happens next → suggest a treatment.

---

## Table of contents

1. [What the system does](#1-what-the-system-does)
2. [What it is NOT](#2-what-it-is-not)
3. [The dataset](#3-the-dataset)
4. [How it is built (architecture)](#4-how-it-is-built-architecture)
5. [Planned project structure](#5-planned-project-structure)
6. [Setup instructions](#6-setup-instructions)
7. [Configuration](#7-configuration)
8. [How to run](#8-how-to-run)
9. [Checkpoint status](#9-checkpoint-status)
10. [Team](#10-team)

---

## 1. What the system does

| # | Job | In plain words |
|---|-----|----------------|
| 1 | See the network now | A picture of every road, refreshed every 5 minutes |
| 2 | Spot jams and unusual traffic | Compare live traffic with what is normal for that day and hour |
| 3 | Detect incidents | A stalled vehicle or a sudden rush of vehicles, when the data shows it |
| 4 | Predict 15–60 minutes ahead | Speed, flow and jam level at 15, 30, 45 and 60 minutes |
| 5 | Suggest fixes with proof | Quick diversions now; long-term road improvements for everyday jams, with before/after impact |

**The key idea — two kinds of jams.** A *sudden* jam (accident, rain, event, road work) needs a quick
diversion. An *everyday* jam (the road is too small — the data even flags this as `structural_bottleneck`)
needs a road improvement. The system tells them apart instead of treating everything as an alert.

## 2. What it is NOT

- **Not Google Maps** — it serves the people managing the whole city, not one driver.
- **Not a chatbot** — it analyses data and produces decisions with evidence.
- **Nothing real is controlled** — no live signals, cameras, GPS, sensors or construction. Every plan is a simulation or a suggestion (a hackathon rule).

## 3. The dataset

Organiser dataset: **NeuraX Smart Cities v2** (folder `NEURAX_SMART_CITIES_TRAINING_V2`).

| Property | Value |
|----------|-------|
| Road network | 436 directed segments joining 120 junctions |
| Resolution | one reading every 5 minutes |
| Training | 15 days · ~1.88 M rows |
| Validation | 4 days · ~0.5 M rows |
| Hidden test (judges) | 8 days · ~1.0 M rows · 36 hidden scenarios |

| File | What is inside | Why we need it |
|------|----------------|----------------|
| `nodes.csv` | location of the 120 junctions | draw the map |
| `network.csv` | which road joins which junctions; lanes, speed, capacity, bottleneck flag | build the road map; each road's limit |
| `traffic_train/validation.csv` | 5-min readings: speed, flow, occupancy, delay, queue, congestion index, sensor quality | current state; model inputs |
| `forecast_targets_train/validation.csv` | the *answer key*: true speed/flow/congestion 15–60 min later | **training labels only — never an input** |
| `incidents_train/validation.csv` | incident type, severity, lanes blocked, time window | teach the incident detector |
| `roadworks_train/validation.csv` | planned closures: road, fraction closed, window | expected slowdown, not an anomaly |
| `context_train/validation.csv` | weather, rain, events, holidays, day of week, hour | drivers of demand and speed |
| `signal_plans.csv` | traffic-light timings per junction | junction capacity |
| `turn_restrictions.csv` | turns not allowed (always / time window) | diversions never suggest an illegal turn |
| `od_demand_profiles.csv` | origin → destination demand and purpose | where demand comes from |
| `planning_candidates.csv` | 90 possible road improvements: capacity added, cost, feasibility | the menu of long-term fixes |
| `scenario_examples.csv` | 30 sample situations, "compare baseline vs counterfactual" | how the judges will test us |

**The data is messy on purpose** (declared in `DATASET_MANIFEST.json`): missing values, duplicates, spikes,
stuck sensors, impossible negative readings, row shuffle. Cleaning comes first.

**No-cheating rule:** `forecast_targets_*.csv` is used only as labels and for checking. Predictions use only
past and present data. The hidden test days are never touched.

## 4. How it is built (architecture)

A 7-step pipeline. The road-map graph (step 2) is shared by every later step, which is how the system
understands that a jam on one road affects the roads next to it.

```
Raw CSVs
  │
  ▼
[1] Clean the data ──► [2] Build the road map (graph) ──► [3] Find problems
                                                              (jams · anomalies · incidents · spillback)
                                                                     │
                       weather / events / holidays ─────────────────►│
                                                                     ▼
                                                        [4] Predict 15 / 30 / 45 / 60 min
                                                                     │
                                                                     ▼
                                                        [5] Suggest actions
                                                            (diversions · road improvements via simulation)
                                                                     │
                                                                     ▼
                                                        [6] Explain (evidence + confidence)
                                                                     │
                                                                     ▼
                                                        [7] Operator dashboard (advisory only)
```

| Step | What it does |
|------|--------------|
| 1 Clean | de-duplicate, restore time order, drop impossible values, detect spikes & stuck sensors, weight by `sensor_quality`, fill gaps from neighbouring roads |
| 2 Road map | NetworkX graph of 120 nodes / 436 edges with capacity, speed, lanes, signals, turn rules, bottleneck flags; engineered features (lags, neighbours, time, weather) |
| 3 Find problems | congestion state vs recurring baseline (day-of-week × hour); supervised incident detection & classification; spillback over the graph; roadworks treated as expected |
| 4 Predict | multi-horizon model (gradient boosting baseline) for speed / flow / congestion with a confidence value |
| 5 Suggest | diversions = constrained routing that obeys turn rules and checks capacity; road improvements = simulate each `planning_candidates` entry and rank by benefit per cost |
| 6 Explain | evidence, confidence and limits attached to every output |
| 7 Dashboard | Streamlit: map, alerts, forecasts, recommendations, before-vs-after view |

**Tools:** Python 3.10+, pandas, NumPy, NetworkX, scikit-learn, LightGBM, a small custom traffic simulator, Streamlit.

## 5. Planned project structure

```
neurax-traffic-intelligence/
├── README.md                 ← this file
├── .env.example              ← configuration template (copy to .env)
├── requirements.txt
├── data/
│   └── NEURAX_SMART_CITIES_TRAINING_V2/   ← organiser dataset goes here (not committed)
├── src/
│   ├── clean.py              ← step 1
│   ├── graph.py              ← step 2
│   ├── detect.py             ← step 3
│   ├── forecast.py           ← step 4
│   ├── recommend.py          ← step 5 (diversions + planning)
│   ├── explain.py            ← step 6
│   └── simulate.py           ← the small traffic simulator
├── app/
│   └── dashboard.py          ← step 7 (Streamlit)
├── outputs/                  ← alerts, forecasts, recommendations (generated)
└── docs/
    └── Research_Document.docx
```

## 6. Setup instructions

**Prerequisites:** Python 3.10 or newer, `pip`, and the organiser dataset folder.

```bash
# 1. Get the code (clone the repo or unzip the submission)
git clone <repo-url> neurax-traffic-intelligence
cd neurax-traffic-intelligence

# 2. Create and activate a virtual environment
python -m venv .venv
# Windows:
.venv\Scripts\activate
# macOS / Linux:
source .venv/bin/activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Put the organiser dataset in place
#    Copy the folder NEURAX_SMART_CITIES_TRAINING_V2 into ./data/

# 5. Create your configuration file
#    Windows:
copy .env.example .env
#    macOS / Linux:
cp .env.example .env
#    Then open .env and check DATA_DIR points at the dataset folder.
```

## 7. Configuration

All settings live in `.env` (see `.env.example`). No secrets are required to run the core system —
everything is computed locally from the organiser CSVs.

| Variable | Meaning | Default |
|----------|---------|---------|
| `DATA_DIR` | path to the organiser dataset folder | `./data/NEURAX_SMART_CITIES_TRAINING_V2` |
| `OUTPUT_DIR` | where alerts, forecasts and recommendations are written | `./outputs` |
| `RESOLUTION_MINUTES` | time between readings | `5` |
| `FORECAST_HORIZONS` | prediction horizons in minutes | `15,30,45,60` |
| `RANDOM_SEED` | fixed seed for reproducible results | `42` |
| `ANOMALY_Z_THRESHOLD` | how far from normal counts as "unusual" | `3.0` |
| `INCIDENT_CONFIDENCE_MIN` | minimum confidence before raising an incident alert | `0.70` |
| `DASHBOARD_PORT` | Streamlit port | `8501` |
| `LLM_API_KEY` | *optional* — only if natural-language explanations use an external model; leave blank otherwise | *(blank)* |

## 8. How to run

```bash
# Run the full pipeline (clean → graph → detect → forecast → recommend → explain)
python -m src.pipeline --config .env

# Evaluate on the validation days (prints detection, forecasting and recommendation metrics)
python -m src.evaluate --split validation

# Start the operator dashboard
streamlit run app/dashboard.py
```

Outputs are written to `OUTPUT_DIR` as CSV/JSON: `alerts.csv`, `forecasts.csv`, `recommendations.json`,
each row carrying its evidence and a confidence value.

## 9. Checkpoint status

| Checkpoint | Marks | Status |
|------------|-------|--------|
| **1 — README / Research / configuration** | 15 | ✅ This README, `.env.example`, `requirements.txt`, and `docs/Research_Document.docx` |
| 2 — Partial execution | 25 | 🔜 cleaning report, road-map graph, basic detection, baseline forecaster, first diversion example, minimal dashboard |
| 3 — Full solution | 60 | 🔜 trained incident detector with false-alarm control, stronger forecaster with confidence, full recommendations, stress tests, explainability, polished UI |

## 10. Team

| Name | Role |
|------|------|
| _____________ | _____________ |
| _____________ | _____________ |
| _____________ | _____________ |

---

*All actions, diversion plans and network suggestions produced by this system are simulated or advisory, as required by the problem statement.*
