# Setup Guide

> **This file is read by the automated evaluation pipeline. Be precise and complete.**

## Prerequisites

Before you begin, ensure you have the following installed:

- [x] **Python 3.10+** (tested on 3.10 / 3.11 / 3.12)
- [x] **pip** (Python package manager, bundled with Python)
- [x] **Git** (to clone the repository)

> **Note:** No Docker, database server, or cloud account is required. The project runs entirely locally using synthetic data and file-based storage (CSV + pickle).

---

## Environment Variables

This project does **not** require any API keys or external service credentials for the core pipeline or dashboard. The `.env.example` in `src/` is a template from the hackathon scaffold; the actual pipeline does not read environment variables.

If you plan to extend the project with IBM watsonx.ai integration in the future, copy and configure the environment file:

```bash
cd src
cp .env.example .env
# Edit .env with your IBM watsonx.ai credentials (optional, not used by current pipeline)
```

| Variable             | Description                        | Required |
|----------------------|------------------------------------|----------|
| `WATSONX_API_KEY`    | IBM watsonx.ai API key             | No       |
| `WATSONX_PROJECT_ID` | watsonx.ai project ID              | No       |
| `WATSONX_URL`        | watsonx.ai endpoint URL            | No       |

---

## Installation

### 1. Clone the Repository

```bash
git clone https://github.com/Steosumit/bob-ai-hackathon-Et-Tu-Brutus.git
cd bob-ai-hackathon-Et-Tu-Brutus
```

### 2. Create a Virtual Environment (Recommended)

```bash
# Windows
python -m venv venv
venv\Scripts\activate

# macOS / Linux
python3 -m venv venv
source venv/bin/activate
```

### 3. Install Python Dependencies

```bash
pip install -r requirements.txt
```

**Complete dependency list:**

| Package            | Purpose                                              |
|--------------------|------------------------------------------------------|
| `pandas`           | Data manipulation, CSV I/O                           |
| `numpy`            | Numerical operations, random generation              |
| `networkx`         | Forensic transaction graph (MultiDiGraph)             |
| `pyvis`            | Interactive network graph visualization in dashboard |
| `streamlit`        | Web-based investigator dashboard (UI)                |
| `faker`            | Synthetic Indian-name account generation             |
| `scipy`            | Benford's Law chi-square statistical tests           |
| `python-louvain`   | Louvain community detection (fraud ring discovery)   |
| `reportlab`        | PDF generation (Form 'A' reports & FIR case briefs)  |
| `matplotlib`       | Investigative SOP flowchart diagrams in PDFs         |

---

## Project Structure

```
bob-ai-hackathon-Et-Tu-Brutus/
├── src/                          # All source code
│   ├── generate_dataset.py       # Phase 1: Synthetic dataset generator
│   ├── accounts.py               #   └─ Account population generator
│   ├── normal_transactions.py    #   └─ Normal background transactions
│   ├── hard_negatives.py         #   └─ Hard-negative (tricky legit) transactions
│   ├── fraud_storylines.py       #   └─ Fraud ring injection
│   ├── export.py                 #   └─ Per-account CSV export
│   ├── export_account_mapping.py #   └─ Account ID ↔ name mapping
│   ├── config.py                 #   └─ Calibration constants (cited sources)
│   ├── ingest_statements.py      # Phase 2: Statement ingestion & normalization
│   ├── build_graph.py            # Phase 3: Forensic graph construction
│   ├── build_heuristics.py       # Phase 4: Heuristic detection (Fan-In, Layering)
│   ├── evaluate_heuristics.py    # Phase 4b: Detection evaluation vs ground truth
│   ├── run_enrichment_pipeline.py# Phase B: Master enrichment orchestrator
│   ├── feature_engineering.py    #   └─ Graph/flow/temporal feature computation
│   ├── risk_score.py             #   └─ Transparent risk scoring (Tier 1)
│   ├── community_detection.py    #   └─ Louvain unsupervised clustering
│   ├── benford_analysis.py       #   └─ Benford's Law chi-square analysis
│   ├── ablation_test.py          #   └─ Per-feature ablation testing
│   ├── camouflage_retest.py      #   └─ Camouflage stress-testing
│   ├── ring_crossref.py          #   └─ Community vs fraud ring validation
│   ├── export_report.py          # Form 'A' evidence index PDF generator
│   ├── fir_case_brief.py         # FIR-ready case brief PDF generator
│   ├── app.py                    # Streamlit investigator dashboard
│   └── .env.example              # Environment variable template
├── data/                         # Generated & processed data files
├── docs/                         # Documentation
├── demo/                         # Demo artifacts (screenshots, video link)
├── presentation/                 # Slide deck
└── submission.yaml               # Hackathon submission metadata
```

---

## Running the Full Pipeline

The pipeline has **4 sequential phases**. All commands are run from the `src/` directory:

```bash
cd src
```

### Phase 1 — Generate Synthetic Dataset

Generates ~550 accounts and ~30,000+ UPI transactions (PhonePe-style CSV statements) with injected fraud rings.

```bash
python generate_dataset.py
```

**Outputs** (written to `../data/`):
- `statements/` — Per-account CSV statement files
- `account_mapping.csv` — Account ID ↔ name mapping
- `ground_truth.csv` — Answer key (roles: normal / hard_negative / mule)
- `ground_truth_transactions.csv` — Transaction-level labels

### Phase 2 — Ingest & Normalize Statements

Parses raw PhonePe-style statements, resolves entity names to account IDs, and produces a unified transaction ledger.

```bash
python ingest_statements.py
```

**Output:** `../data/normalized_transactions_v3.csv`

### Phase 3 — Build Forensic Graph

Constructs a NetworkX MultiDiGraph where nodes are accounts and edges are transactions (keyed by UTR).

```bash
python build_graph.py
```

**Output:** `../data/forensic_network.gpickle`

### Phase 4 — Run Heuristic Detection

Applies Fan-In (Tier 1, high-confidence) and Layering (Tier 2, low-confidence) detection heuristics.

```bash
python build_heuristics.py
```

**Output:** `../data/heuristic_alerts.csv`

### Phase B — Run Enrichment Pipeline

Runs all Phase B enrichment components in sequence: feature engineering → risk scoring → community detection → Benford's analysis → evaluation.

```bash
python run_enrichment_pipeline.py
```

**Outputs:**
- `../data/account_features.csv` — Graph/flow/temporal features
- `../data/account_risk_scores.csv` — Transparent risk scores & tiers
- `../data/detected_communities.csv` — Louvain community summaries
- `../data/detected_communities_accounts.csv` — Per-account community assignments
- `../data/benford_results.csv` — Benford's Law test results

---

## Running the Dashboard

Launch the Streamlit investigator dashboard:

```bash
cd src
streamlit run app.py
```

The application will open automatically in your browser at: **`http://localhost:8501`**

### Dashboard Tabs

| Tab                        | Description                                                  |
|----------------------------|--------------------------------------------------------------|
| **Overview**               | KPIs, detection performance, risk tier distribution          |
| **Alert Investigation**    | Per-alert ego graphs with evidence highlighting, PDF export  |
| **Fraud Ring Discovery**   | Louvain community clusters with mule density analysis        |
| **Risk Scores**            | Filterable account risk scores with transparent formula      |
| **Benford's Law**          | Exploratory first-digit analysis (supplementary signal)      |

From the **Alert Investigation** tab, you can generate:
- **Form 'A' Report** — Court-compliant electronic evidence index (PDF)
- **FIR Case Brief** — Full investigation summary with legal sections and SOP flowchart (PDF)

Generated PDFs are saved to the `reports/` directory at the project root.

---

## Quick Demo

To run the entire pipeline from scratch and launch the dashboard in one go:

```bash
cd src

# Generate data + build pipeline
python generate_dataset.py
python ingest_statements.py
python build_graph.py
python build_heuristics.py
python run_enrichment_pipeline.py

# Launch dashboard
streamlit run app.py
```

> **Note:** If the `data/` directory already contains the pre-generated CSV files (included in the repository), you can skip directly to launching the dashboard with `streamlit run app.py` from the `src/` directory.

---

## Running Validation Tests

These scripts validate the detection system's integrity (not unit tests — they are validation experiments):

```bash
cd src

# Per-feature ablation testing
python ablation_test.py

# Camouflage stress-test (adds normal activity to mule accounts)
python camouflage_retest.py

# Cross-reference Louvain communities against actual fraud ring IDs
python ring_crossref.py

# Evaluate heuristic detection against ground truth
python evaluate_heuristics.py
```

---

## Troubleshooting

| Issue | Solution |
|-------|---------|
| `ModuleNotFoundError: No module named 'community'` | Install the Louvain package: `pip install python-louvain` (not `pip install community`) |
| `ModuleNotFoundError: No module named 'pyvis'` | Run `pip install pyvis` |
| `ModuleNotFoundError: No module named 'scipy'` | Run `pip install scipy` |
| `ModuleNotFoundError: No module named 'faker'` | Run `pip install faker` |
| `ModuleNotFoundError: No module named 'reportlab'` | Run `pip install reportlab` |
| `FileNotFoundError: ../data/forensic_network.gpickle` | Run the pipeline in order: `generate_dataset.py` → `ingest_statements.py` → `build_graph.py` before launching the dashboard |
| `FileNotFoundError: ../data/heuristic_alerts.csv` | Run `python build_heuristics.py` before launching the dashboard |
| Streamlit dashboard shows "Missing prerequisite files" | Ensure you're running `streamlit run app.py` from the `src/` directory (the app uses relative paths `../data/`) |
| Community detection skipped with `SKIPPED (missing dependency)` | Run `pip install python-louvain` |
| Benford's analysis skipped with `SKIPPED (missing dependency)` | Run `pip install scipy` |
| PDF generation fails | Ensure `reportlab` and `matplotlib` are installed: `pip install reportlab matplotlib` |
| Port 8501 already in use | Run with a custom port: `streamlit run app.py --server.port 8502` |
