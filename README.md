# Hybrid Quantum-Classical Insider-Threat Detection (CERT r4.2)

A controlled, reproducible comparison of a hybrid quantum-classical neural network with a parameter-matched classical network for detecting malicious insiders in enterprise activity logs.

## Why this matters

Quantum machine learning is often claimed to improve cybersecurity analytics, but such claims are rarely tested against a fair classical baseline. Insider-threat detection is a strong test case for two reasons. Confirmed malicious insiders are rare, so models must learn from very few labeled cases. Detectors are often deployed on premises, where small, auditable models are preferred.

This project tests whether a quantum layer helps under those conditions. The two models have identical parameter counts, and each gets its own tuned learning rate. Results are paired across five seeds and tested for significance. Every setting is reported, including those where the classical model wins.

**Result:** with only 4 labeled insiders, the hybrid detector beats the classical detector on all five seeds in ROC-AUC (0.945 vs. 0.891) and F1 (0.184 vs. 0.090), one-sided Wilcoxon p = 0.031. It never collapsed (test PR-AUC below 0.05), while the classical detector collapsed in 40% of runs. In the two smallest models, the collapse rate was 0% for the hybrid and 60% for the classical detector. The advantage disappears when labels and model capacity are ample.

![Test PR-AUC vs. model size and labeled insiders](Images/hybrid_vs_classical_pr_auc.png)

## Dataset

**CERT Insider Threat Test Dataset, release r4.2**, from the CERT Division of the Software Engineering Institute, Carnegie Mellon University.
DOI: [10.1184/R1/12841247.v1](https://doi.org/10.1184/R1/12841247.v1), licensed CC BY 4.0. It is synthetic enterprise data: benign background activity plus labeled malicious-insider scenarios such as after-hours access, removable-media use, and data exfiltration.

| Item | Value |
|---|---|
| Users | 1,000 (72 malicious insiders) |
| Logs used | logon, removable device, file access |
| Records | 330,452 user-days |
| Malicious user-days | 986 (0.30%) |
| Features | 35: activity counts, after-hours activity, workstation and file-type counts, each also expressed as a deviation from the user's previous 30 days |
| Split | by user, 60/20/20 (43 / 14 / 15 insiders) |

The notebook downloads the dataset and its answer key, verifies their MD5 checksums, and caches everything on Google Drive.

## Approach

```
35 features -> encoder (35 -> h -> 4) -> [ 48-parameter block ] -> linear head -> risk score
                                            |-- Hybrid:    4-qubit, 3-layer data re-uploading circuit
                                            |-- Classical: 2 x (Linear 4x4 + tanh + scale)
```

- **Quantum layer.** In each layer the inputs are angle-encoded as RY rotations with trainable scaling, followed by general rotations and a CNOT ring; the layer outputs Pauli-Z expectations. The circuit is simulated exactly on a GPU in PyTorch, and every run cross-checks it against PennyLane `default.qubit` (max difference about 2e-7).
- **Experiments.**
  - (a) model size: encoder width h = 4, 8, 16, 32 (217 to 1,337 parameters)
  - (b) label scarcity: 4, 11, 22 or 43 labeled training insiders
- **Protocol.**
  - The learning rate is tuned separately for every setting and model, chosen on validation PR-AUC from a grid of 3e-4 to 3e-2.
  - Each configuration then runs five fresh seeds: up to 40 epochs with early stopping, then a single evaluation on the test set.
  - The alert threshold is fixed on the validation set at a 1% false-positive rate.
- **Metrics.**
  - PR-AUC, ROC-AUC, recall at 1% false-positive rate, F1
  - share of test insiders alerted
  - collapse rate: share of seeds with test PR-AUC below 0.05
  - paired one-sided Wilcoxon signed-rank tests between the two models

## Repository contents

| File | Purpose |
|---|---|
| `QDFIR_Hybrid_Confirmation_CERT_r42.ipynb` | Single standalone Colab cell: data download or reuse, features, tuning, five-seed runs, statistics, LaTeX tables, and the figure |
| `Images/hybrid_vs_classical_pr_auc.png` | Results figure |

## How to run

1. Open `QDFIR_Hybrid_Confirmation_CERT_r42.ipynb` in Google Colab.
2. Set **Runtime → Change runtime type → GPU** (a T4 is sufficient).
3. Run the cell and authorize Google Drive when prompted.

The first run downloads about 4.8 GB and decompresses it, which takes roughly 20 minutes. The experiments then take about 60 minutes on a T4. Later runs reuse the cached data in `MyDrive/QDFIR_Empirical/data/`.

Outputs are written to `MyDrive/QDFIR_Empirical/results/confirmation/`:

- `all_runs.csv`, `summary.csv`, `paired_deltas.csv`, `tuning.csv`
- `table_width_rows.tex`, `table_labels_rows.tex`
- `pr_auc_width_labels.pdf` / `.png`

## Design decisions

- **Exact simulation instead of quantum hardware.** This isolates the quantum layer's effect on detection quality from hardware noise. The project measures accuracy and stability, not speedup.
- **CERT r4.2** was chosen because it is a public, labeled insider-threat benchmark.
- **Splitting by user** prevents any individual's behavior from leaking between training and test, at the cost of a test set with 15 insiders.
- **Shared learning-rate grid.** Both models search the same grid. Three of the fourteen selections fell at the edge of the grid.

## Dataset citation

```bibtex
@misc{Lindauer2020CERT,
  author    = {Lindauer, Brian},
  title     = {Insider Threat Test Dataset},
  publisher = {Carnegie Mellon University},
  year      = {2020},
  doi       = {10.1184/R1/12841247.v1}
}
```
