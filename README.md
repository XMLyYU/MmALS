[README.md](https://github.com/user-attachments/files/29578726/README.md)
# MmALS: AI-Assisted Directed Evolution Pipeline for CSCE

## Background

This repository documents the complete process of multi-round AI-assisted mutation design combined with wet-lab closed-loop experiments for an enzyme (CSCE). The entire workflow is organized through a series of Jupyter Notebooks, sequentially accomplishing the following steps:

Mutation Region Screening (Region Search, RSO) — Identifying modifiable "hot spots" based on stability zero-shot deep mutational scanning (DMS).
AI Round 1 / 2 / 3 — In each round: analyzing experimental data from the previous round → generating candidates through multi-objective optimization (MOO) → filtering using multiple scoring models → selecting final sequences for testing via embedding-based clustering.

---

## 1. Repository Layout

```
aldes/
└── csce/
    ├── region_search/                    # Step 0. Mutation hotspot search (RSO)
    │   ├── Mutation_region_search.ipynb     # main RSO notebook
    │   ├── all_single_mut.csv               # all single-point mutants of WT (input)
    │   └── all_single_mut_stability_input.pred.csv  # stability DMS predictions
    │
    ├── AI-round1/                        # Step 1. AI Round 1
    │   ├── 1.Expert_dms_result_analyse.ipynb  # calibrate score thresholds on expert DMS
    │   └── 2.Sequence_pick.ipynb              # filter MOO pool + embedding clustering
    │
    ├── AI-round2/                        # Step 2. AI Round 2
    │   ├── 1.Exp_data_analyse.ipynb           # analyse round-1 wet-lab data
    │   └── 2.MOO_mutants_pick.ipynb           # filter round-2 MOO pool
    │
    ├── AI-round3/                        # Step 3. AI Round 3 (three batches)
    │   ├── 1.Seen_region_search.ipynb         # batch 1: known region (top-3 derived)
    │   ├── 2.Unseed_rigion_wo_ai.ipynb        # batch 2a: unseen region, no activity prior
    │   └── 3.Unseed_rigion_w_ai.ipynb         # batch 2b: unseen region, with activity prior
    │
    └── data/                             # intermediate data, candidates and wet-lab results
        ├── data_expert_r1_dms.csv               # expert prior DMS (used to calibrate thresholds)
        ├── data_ai_r1_all_scaned.csv        # MOO pool fully scored, r1 (input)
        ├── data_ai_r2_all_scaned.csv        # MOO pool fully scored, r2 (input)
        ├── data_ai_r3_all_scaned_seen_region.csv
        ├── data_ai_r3_moo_optimal_unseen_region.csv
        ├── data_ai_r3_moo_optimal_unseen_region_high_ac.csv
        ├── candidates_ai_round1.csv         # final round-1 candidates sent to wet lab
        ├── candidates_ai_round2.csv         # final round-2 candidates sent to wet lab
        ├── candidates_ai_round3_batch1_seen_region.csv
        ├── candidates_ai_round3_batch2_unseen_region_w_ai_region1.csv
        ├── candidates_ai_round3_batch2_unseen_region_wo_ai.csv
        ├── exp_result/                          # raw / aggregated wet-lab data (xlsx, csv)
            └── r1r2-fix0619.xlsx                      # final validated wet-lab tracking
        └── xtrimo_activity/                     # isolated model component details & reproducibility data
            ├── README.md                              # documentation for independent model validation
            └── train_split_v12.csv                    # transparent sample-level train/val data splits
```

---

## 2. End-to-End Reproduction

Each step below corresponds to one or two notebooks. Inputs and outputs of
every step are explicit CSV files under `csce/data/`, so any individual step
can be re-run on its own.

### Step 0 — Mutation Region Search

`csce/region_search/Mutation_region_search.ipynb`

- **Inputs:** `all_single_mut.csv` (every single-point mutant of WT) and
  `all_single_mut_stability_input.pred.csv` (stability ΔΔG prediction for each
  mutant).
- **Pipeline:** per-position aggregate `mean(ddg) - std(ddg)` → 7-residue
  sliding-window average → threshold at `-0.45` to keep tolerant positions →
  window-size / threshold sensitivity sweep.
- **Output:** the set of "hotspot" residues that constrains the MOO search
  space for later rounds.

### Step 1 — AI Round 1

1. `AI-round1/1.Expert_dms_result_analyse.ipynb`
   - **Input:** `data/data_expert_r1_dms.csv`.
   - **Output:** correlation between each of the four scores and ground-truth
     labels, plus per-percentile hit-rate of "better-than-WT" mutants. The
     thresholds used in all later rounds come from this notebook.
2. `AI-round1/2.Sequence_pick.ipynb`
   - **Inputs:** `data/data_ai_r1_all_scaned.csv` (round-1 MOO pool with all
     four scores) and `data/r1_activity_zero_shot_pred/hidden_states/*.pt`
     (hidden states of the activity model used for clustering).
   - **Pipeline:** threshold filter → split by `mut_num` and run KMeans
     (2-mut → 30 clusters, 3-mut → 12, 4-mut → 8, 5-mut → 6) → pick the
     highest-activity sequence per cluster → t-SNE visualisation.
   - **Output:** `data/candidates_ai_round1.csv` (sequences sent to wet lab).

### Step 2 — AI Round 2

1. `AI-round2/1.Exp_data_analyse.ipynb`
   - **Input:** `data/exp_result/r1r2-fix0619.xlsx` (raw round-1 enzyme
     activity readouts).
   - **Pipeline:** per-plate `QE` normalisation, compute `sig_to_qe`, build
     the labelled dataset that is used to train `activity_v1`.
2. `AI-round2/2.MOO_mutants_pick.ipynb`
   - **Input:** `data/data_ai_r2_all_scaned.csv` (round-2 MOO pool, includes
     `activity_v1_seed1` / `activity_v1_seed2`).
   - **Pipeline:** five-way joint threshold filter (the four base scores plus
     a two-seed consistency check on `activity_v1`).
   - **Output:** `data/candidates_ai_round2.csv`.

### Step 3 — AI Round 3 (three batches over different regions)

1. `AI-round3/1.Seen_region_search.ipynb` — **Batch 1 / Seen Region**
   - **Input:** `data/data_ai_r3_all_scaned_seen_region.csv` (MOO over the
     region already explored in rounds 1–2, with `activity_v2`).
   - **Output:** `data/candidates_ai_round3_batch1_seen_region.csv`.

2. `AI-round3/2.Unseed_rigion_wo_ai.ipynb` — **Batch 2a / Unseen Region, no
   activity prior**
   - **Inputs:** `data/data_ai_r3_moo_optimal_unseen_region.csv` and
     `data/r3_embeddings/*.npy`.
   - **Pipeline:** threshold filter → KMeans clustering on embeddings, take
     one representative per cluster.
   - **Output:** `data/candidates_ai_round3_batch2_unseen_region_wo_ai.csv`.

3. `AI-round3/3.Unseed_rigion_w_ai.ipynb` — **Batch 2b / Unseen Region, with
   activity prior**
   - **Input:** `data/data_ai_r3_moo_optimal_unseen_region_high_ac.csv` (MOO
     pool already pre-filtered with the activity model for high-activity
     candidates).
   - **Output:** `data/candidates_ai_round3_batch2_unseen_region_w_ai_region1.csv`.

---

## 4. Environment

Main dependencies:

```
python>=3.8
pandas>=1.3.0,<3.0.0, numpy>=1.21.0,<2.0.0, scipy>=1.7.0,<2.0.0, scikit-learn>=1.0.0,<2.0.0, matplotlib>=3.4.0,<4.0.0, plotly>=5.0.0,<6.0.0, tqdm>=4.62.0,<5.0.0
torch>=1.13.0,<3.0.0          # to load hidden_states-*.pt
openpyxl>=3.0.10,<4.0.0       # to read exp_result/*.xlsx
pyyaml>=6.0,<7.0              # to read mapie inference results
```

