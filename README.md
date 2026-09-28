#  UPI Fraud Trail Reconstruction Engine

> Reconstructing multi-hop UPI money-laundering trails from bank statement
> disclosures — and turning the reconstruction into a court-ready case file.

---

## Team

| Field | Value |
|---|---|
| **Team Name** | Et Tu Brutus!? |
| **Track** | Cyber Forensics |
| **Team Lead** | Atharva — karajgaonkaratharva@gmail.com |
| **Members** | Sumit, Chaitanya, Shreya |

---

## Problem Statement

Cyber crime analysts face alert fatigue while manually investigating thousands of
account transaction records to find mule or suspicious accounts. Every alert has
to be checked by hand against bank and UPI statements, so backlogs grow and fraud
identification slips past the legal timeline — while victims wait.

Existing tooling makes this worse: analysts either scroll through records
manually, or run black-box ML models that flag an account but never show the
*evidence*. An analyst who cannot see which exact transaction proves the pattern
cannot build a case.

---

## Solution

We built a graph-based forensic workbench that ingests real-format PhonePe/bank
CSV statements, resolves counterparty names to accounts, and reconstructs the
transaction graph an investigator would otherwise have to build by hand. Three
detection systems narrow the scope — a **Fan-In** rule, a transparent
**concentrated-outflow risk score**, and unsupervised **Louvain community
detection** that isolates whole fraud rings with zero labels — and every finding
exports as a **Form 'A' evidence index** or a **FIR case brief** citing the exact
UTRs that triggered it.

The core design choice: *we built hard negatives into the dataset.* 20 legitimate
accounts (freelancers, families splitting bills, small merchants) are generated to
look like fraud on purpose, so false-positive rates are measured honestly instead
of being accidentally perfect.

---

## Key Features

- **Realistic statement ingestion** — Parses PhonePe-style CSVs
  (`Date, Time, Transaction Details, Transaction ID, UTR, Transaction Type,
  Credit/Debit Instrument, Amount`), where the counterparty is buried in a messy
  free-text narration field (`Paid to X / UTR No. ... / Paid by XXXXXXXX729`).
  Entity resolution quarantines ambiguous names rather than guessing.
- **Fan-In detection (Tier 1)** — Flags accounts that collect many small incoming
  payments and consolidate them into a near-equal-value outgoing transfer within
  a 4-hour window. **100% precision: 0 false positives** across all 480 normal
  and all 20 hard-negative accounts.
- **Louvain fraud-ring discovery (unsupervised)** — Louvain clustering on the
  amount-weighted transaction graph isolates **7/7 multi-account layering rings
  perfectly**, with no labels used. The 27 single-mule fan-in accounts
  independently cluster into 3 groups by shared victim pool — a structural
  finding the labels were never used to produce.
- **Transparent risk scoring (Tier 1)** —
  `risk_score = 0.5·inv_norm(out_degree) + 0.5·inv_norm(unique_receivers)`.
  Equal weights from domain reasoning, *not* fitted to ground truth. Yields
  **95.8% recall (46/48)** at **85.2% precision**, 1.7% FP on normals and
  **0% FP on hard negatives**.
- **Legal-grade PDF export** — One click generates either a **Form 'A' Index of
  Electronic Exhibits** (Sec. 94 BNSS / Sec. 63 BSA) or a full **FIR case brief**
  with a 3-stage investigative SOP flowchart, mapped legal sections
  (Sec. 102/94 BNSS, Sec. 66C/66D IT Act, PMLA, BNS), and IO/RO sign-off blocks.
- **Adversarial validation harness** — Per-feature ablation across five
  thresholds *with every threshold reported* (no cherry-picking), plus a
  **camouflage retest** that injects 10–25 ordinary-looking transactions into
  every mule account and re-runs the pipeline to prove the surviving features are
  real signal and not artifacts of synthetic data.

---

## Tech Stack

| Category | Technologies |
|---|---|
| **Languages** | Python 3 |
| **Frameworks** | Streamlit (investigator dashboard), NetworkX (forensic graph), python-louvain (community detection), pyvis-network + vis-network 9.1.2 (interactive ego-graphs) |
| **IBM Technologies** | IBM Cloud (Docker Hosting), IBM Bob |
| **Databases** | None. State is derived artifacts on disk (`data/*.csv` + a pickled `MultiDiGraph`) — chosen so every intermediate result is auditable. |
| **Data / Analysis** | pandas, NumPy, SciPy (Chi-square for Benford's test), reportlab (legal PDF generation), matplotlib (SOP flowchart), Faker (synthetic account population) |
| **Other** | GitHub Actions (submission validator), vendored front-end JS under `src/lib/` |

---

## Repository Structure

```
├── src/                              # All source code
│   ├── generate_dataset.py           # Synthetic data factory (seeded, 550 accounts)
│   ├── accounts.py                   # Account population + ground-truth role labels
│   ├── normal_transactions.py        # Background noise (log-normal, social circles)
│   ├── hard_negatives.py             # 20 legitimate accounts designed to look fraudulent
│   ├── fraud_storylines.py           # Injected fraud rings (pass-through / fan-in / fan-out)
│   ├── export.py                     # Writes per-account PhonePe-format CSV statements
│   ├── export_account_mapping.py     # account_id <-> holder-name crosswalk
│   │
│   ├── ingest_statements.py          # Phase 2: parse + entity-resolve + quarantine
│   ├── build_graph.py                # Phase 3: build the MultiDiGraph (UTR = edge key)
│   ├── build_heuristics.py           # Phase 4: Fan-In + Layering detection rules
│   ├── evaluate_heuristics.py        # Phase 4 eval: recall / precision / F1 / confusion matrix
│   │
│   ├── feature_engineering.py        # Phase A: 19 graph / flow / temporal features
│   ├── risk_score.py                 # Phase B: transparent Tier-1 risk score
│   ├── community_detection.py        # Phase B: Louvain ring discovery
│   ├── benford_analysis.py           # Exploratory: Chi-square first-digit test
│   ├── run_enrichment_pipeline.py    # Phase B master orchestrator
│   │
│   ├── ablation_test.py              # Per-feature validation across all thresholds
│   ├── camouflage_retest.py          # Adversarial noise-injection stress test
│   ├── ring_crossref.py              # Community detection vs. ground-truth rings
│   │
│   ├── export_report.py              # Form 'A' evidence-index PDF
│   ├── fir_case_brief.py             # FIR case brief PDF + SOP flowchart
│   ├── app.py                        # Streamlit investigator dashboard (5 tabs)
│   ├── config.py                     # Calibration constants, each with a citation
│   └── lib/                          # Vendored front-end JS (vis-network, tom-select)
│
├── data/                             # Committed intermediate artifacts (auditable)
├── docs/                             # Written documentation
├── demo/                             # Demo artifacts
├── presentation/                     # Slide deck
└── submission.yaml                   # Structured submission metadata
```

---

## How to Run

> The committed `data/` files are the *outputs* of the pipeline. The two inputs it
> needs — `data/statements/` and `data/forensic_network.gpickle` — are generated
> locally, so run the pipeline in order before starting the app.

**Prerequisites:** Python 3.9+, ~500 MB disk for generated statements.

```bash
# 1. Clone the repo
git clone https://github.com/Steosumit/bob-ai-hackathon-Et-Tu-Brutus.git
cd bob-ai-hackathon-Et-Tu-Brutus

# 2. Install dependencies
pip install streamlit networkx python-louvain pandas numpy scipy \
            pyvis-network reportlab matplotlib Faker

# 3. Generate the dataset (550 accounts, ~20.5k transactions,
#    exported as per-account PhonePe-format CSV statements)
cd src && python generate_dataset.py

# 4. Ingest + normalize + build the forensic graph
python ingest_statements.py     # -> data/normalized_transactions_v3.csv
python build_graph.py           # -> data/forensic_network.gpickle

# 5. Run detection heuristics
python build_heuristics.py      # -> data/heuristic_alerts.csv
python evaluate_heuristics.py   # recall / precision / F1 / confusion matrix

# 6. Run the enrichment pipeline (features, risk score, Louvain, Benford)
python run_enrichment_pipeline.py

# 7. Launch the investigator dashboard
streamlit run app.py            # -> http://localhost:8501
```

**Optional validation runs:**

```bash
python ablation_test.py         # per-feature recall vs. FP at 5/10/15/20/25%
python camouflage_retest.py     # re-validate after injecting ordinary traffic into mules
python ring_crossref.py         # Louvain communities vs. true fraud-ring membership
```

> `python-louvain` and `scipy` are optional: the pipeline degrades gracefully and
> prints a `SKIPPED` line for the corresponding step if either is missing.

> **Note:** `docs/setup-guide.md` is still the unmodified template. The steps
> above are the verified, working sequence.

---

## Demo

| Artifact | Link |
|---|---|
|  Demo Video | [Google Drive](https://drive.google.com/file/d/1_IPnM-4xinF0D1EXrcsyPP6nCfhVOgN5/view?usp=sharing) — see [`demo/demo-video-link.txt`](demo/demo-video-link.txt) |
|  Live Demo | **Not deployed** — runs locally via the steps above (see [`demo/live-demo-url.txt`](demo/live-demo-url.txt)) |
|  Screenshots | [`demo/screenshots/`](demo/screenshots/) — landing, alert investigation, community map |
|  Presentation | [`presentation/`](presentation/) — ⚠️ deck not yet uploaded |

---

##  Measured Results

All figures recomputed from the committed `data/*.csv` artifacts.

| System | Recall | Precision | FP on Normals | FP on Hard Negatives |
|---|---|---|---|---|
| **Fan-In alone** | 54.0% (27/50) | **100%** | **0.0%** (0/480) | **0.0%** (0/20) |
| **Tier 1 risk score** (top 10%) | **95.8%** (46/48) | 85.2% | 1.7% (8/480) | **0.0%** (0/20) |
| Fan-In ∪ Tier 1 | 97.9% (47/48) | 85.5% | 1.7% (8/480) | 0.0% (0/20) |
| Layering *(Tier 2, lead-only)* | ~14–28% | — | **~30%** | — |
| Benford's Law *(exploratory)* | 16% | 18.6% | 6% | — |

- **48, not 50:** two mule accounts were quarantined by entity resolution because
  their holder names were ambiguous in the statement data. They are correctly
  excluded from feature-based scoring rather than being force-resolved.
- **Louvain:** 7/7 multi-mule layering rings isolated into single communities.
- **Risk tiers:** 56 CRITICAL · 36 HIGH · 357 MEDIUM · 99 LOW.

**Why Layering and Benford are downgraded:** Layering fires on ~30% of perfectly
normal accounts — the honest read is "unreliable, investigative lead only."
Benford's Law has almost no statistical power at 15–60 transactions per account
(43 flagged vs. ~26 expected by chance). Both are shipped in the dashboard behind
explicit low-confidence warnings rather than quietly dropped.

---

##  Known Limitations

- **The dataset is synthetic.** We designed the fraud patterns *and* the detector,
  so absolute numbers are optimistic by construction. The hard negatives and
  camouflage retest exist to blunt this, but they are not a substitute for
  validating on real FIR data.

- **Entity resolution is name-based only.** Duplicate holder names are quarantined,
  not disambiguated by account number or KYC — a deliberate safe choice, but it
  costs recall.

- **New Pattern Identification**: limited in finding unique and new transaction  relationships

---

##  What We're Most Proud Of

**We built the test that could have embarrassed us, and it changed the answer.**

`src/hard_negatives.py` generates 20 legitimate accounts — freelancers taking
client payments, families splitting a utility bill, small merchants paying
suppliers — each structurally a twin of one of our fraud patterns. Then
`src/camouflage_retest.py` injects ordinary-looking traffic into every mule
account and re-runs the full pipeline.

That second test is why the risk score only uses two features. The ablation table
(`data/ablation_results.csv`) shows why: **`pagerank`, `betweenness_centrality`,
`in_degree` and `core_number` all score 0.0% recall** at every threshold. They
are the features a fraud-detection write-up would lead with, and on our data they
are pure noise. `out_degree`, `unique_receivers` and `reciprocity` reach 100%
recall at the 10% threshold with 1.67% false positives on normals.

We also report **every** threshold (5/10/15/20/25%) rather than only the best one.
Sweeping percentiles and quoting the winner is the same "pick what works on the
50 known examples" overfitting sin as tuning weights on the answer key — just
moved from the model to the evaluation.

Finally, **every alert cites the exact UTRs that triggered it** — the Layering
path is stored as `Sender|UTR|Receiver` triples and indexed into the graph by UTR
key, because querying by account pair alone silently pulled in unrelated
background transactions as "evidence." Over-citing noise in a forensic report is
the kind of bug that destroys a case.

---

##  Suggested Reading Order for Judges

1. **The adversarial tests** — `src/hard_negatives.py`, then
   `src/camouflage_retest.py`, then `data/ablation_results.csv`.
2. **The honest metrics** — `src/build_heuristics.py` docstrings state each
   rule's measured recall and FP rate, including the thresholds that failed.
3. **The legal output** — run the app, flag an account, click
   *"Generate FIR Case Brief"*. The PDF is the artifact a cyber cell would
   actually use.
4. **The architecture** — `docs/architecture.md` and `docs/solution-overview.md`
   are still templates; `src/run_enrichment_pipeline.py` is the accurate
   orchestration reference in the meantime.
