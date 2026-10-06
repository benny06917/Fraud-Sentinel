<h1 align="center">🛡️ Fraud Sentinel</h1>
<p align="center"><b>A self-monitoring fraud detection system</b> — it scores transactions, watches itself for drift, and retrains when the world changes.</p>

<p align="center">
<img alt="Python"       src="https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white">
<img alt="XGBoost"      src="https://img.shields.io/badge/XGBoost-detector-EB5E28">
<img alt="SHAP"         src="https://img.shields.io/badge/SHAP-explainability-5B9BD5">
<img alt="Monitoring"   src="https://img.shields.io/badge/Model%20Monitoring-PSI%20drift-2DA44E">
<img alt="Streamlit"    src="https://img.shields.io/badge/Streamlit-dashboard-FF4B4B?logo=streamlit&logoColor=white">
<img alt="License"      src="https://img.shields.io/badge/License-MIT-555">
</p>


<!-- Add a dashboard screenshot here — it's the single highest-impact addition:
<p align="center"><img src="docs/dashboard.png" width="820"></p> -->

---

## Why this exists

Most fraud projects stop at a trained model. In production that's the easy part — the hard part is that **a model decays after deployment**: fraudsters adapt, behaviour shifts, and accuracy quietly erodes. Worse, you often can't *measure* the decay in real time, because confirmed-fraud labels arrive weeks late.

**Fraud Sentinel** is built around that reality. It is not a model; it is the loop a real fraud team runs:

> **Serve** predictions → **Monitor** for drift (label-free) → **Respond** by retraining → serve again.

It is demonstrated on a dataset with a genuine regime change — a dark-market shutdown that abruptly rewrites fraud patterns — so the system is tested against *change*, not a frozen benchmark.

## Headline results

Evaluated on a **held-out future period** at a realistic **5% false-positive operating point**.

| | Metric | Result |
|---|---|---|
| **Detector** | ROC-AUC | **0.922** |
| **Detector** | Recall @ 5% FPR | **0.743** |
| **Monitor** | Drift flagged | **~10 steps before** recall collapsed |
| **Retraining** | Recall recovery | near-**0.0 → 0.95** after the regime change |

> The drift monitor raised a **label-free alarm ten steps before** the model's recall collapsed, and retraining recovered the performance the drift had destroyed. That lead time — catching failure before the losses and the labels confirm it — is the entire value of the system.

## How it works

```
  transactions ─► preprocessing ─► XGBoost detector ─► score + SHAP explanation
                                        │
                                        ├─► PSI drift monitor ─────► alert (no labels needed)
                                        │
                                        └─► walk-forward retraining ─► measured recovery
```

| Stage | What it does | Why it's done this way |
|---|---|---|
| **Detect** | XGBoost scores each transaction; SHAP explains the drivers | Trees dominate tabular fraud; SHAP gives per-decision, regulator-ready reasons |
| **Monitor** | Population Stability Index per period vs. training | Catches input drift **without labels** — the only signal available before losses are confirmed |
| **Respond** | Walk-forward retraining vs. a frozen baseline | Proves the fix: refreshing on new data recovers lost recall |

## Dashboard

An interactive console (Streamlit) presents the system the way a fraud-operations team would use it:

- **Live Scoring** — score a transaction, see it against the operating threshold, and read the SHAP drivers.
- **Drift Monitor** — calibrated PSI over time with the drift alert and regime-change marker.
- **Retraining** — frozen vs. retrained recall: the collapse and the recovery.

## Quick start

```bash
conda create -n sentinel python=3.11 -y && conda activate sentinel
pip install -r requirements.txt

# place the Elliptic CSVs in data/ (features + classes), then:
python main.py           # train, evaluate, monitor drift, simulate retraining
python prep_demo.py      # build lightweight demo assets
streamlit run app.py     # launch the dashboard
```

## Engineering decisions

The choices that separate this from a notebook — each made deliberately:

- **Recall at a fixed 5% FPR, not accuracy or F1.** Fraud teams work under an alert budget; a fixed false-positive rate is the real operating constraint and keeps models comparable. Accuracy is meaningless under heavy imbalance.
- **Temporal evaluation, never random splits.** Fraud is a time series — shuffling leaks the future into training and inflates every metric.
- **Calibrated drift thresholds.** Textbook PSI cutoffs assume a stationary population and flag everything on real, non-stationary data. The alert is calibrated to the training period's own variation — "beyond anything seen in training."
- **Leakage-free retraining.** The walk-forward simulation always scores a period with a model trained only on earlier periods, mirroring production exactly.

## Limitations & productionization

Known boundaries, and what a live deployment would add — stated up front:

- **Drift monitoring is an early warning, not a fix.** It shrinks the window of bad predictions; the model still misses the first cases of a truly new pattern until retrained.
- **It can miss fraud engineered to look normal.** If a scheme mimics legitimate behaviour, inputs barely shift and PSI stays quiet — so input-drift monitoring must be paired with **performance monitoring** once labels arrive.
- **The retraining stage is a historical experiment, not a live service.** Productionizing it means **persisting each retrained model** as the new champion and **rolling the drift reference forward**, so the monitor compares against recent data rather than a frozen baseline.

<details>
<summary><b>Note on scope (an earlier unsupervised approach was tested and dropped)</b></summary>

An earlier version tested unsupervised anomaly detection (autoencoder, isolation forest) for catching novel fraud. Controlled experiments across three datasets showed anomaly detection only separates fraud when it is already spatially distinct (a property of easy, pre-processed datasets), and fails on realistic fraud where illicit activity overlaps normal behaviour. That negative result — verified with a positive control to rule out implementation bugs — motivated the final monitor-and-retrain design.
</details>



---

<p align="center"><sub><b>Author</b> · <a href="#">LinkedIn</a> · <a href="#">Portfolio</a> · MIT License</sub></p>
