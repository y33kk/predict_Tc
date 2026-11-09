
```
predict_tc_main/
├── config.json           all model / training hyperparameters
├── tc_model.ipynb        Colab notebook: trains the model and evaluates it on the test split
├── README.md             this file
├── predict_tc_main.zip   everything above bundled for Colab (has a predict_tc_main/ root folder)
├── external_compouns.zip compounds out of dataset
├── data/
    ├── tc_data.json.gz   1,915 consolidated hydride records, each with Tc
    └── groups.json.gz    5-fold split for cross-validation
```

## Data

One consolidated dataset merged from three sources: the Wines & Choudhary paper (via JARVIS), CompSC, and alex_supercon.

- **Tc label.** Each record's Tc comes from the Allen–Dynes formula with μ\* = 0.1. The paper gives it directly; for CompSC and alex_supercon it is computed from λ and ω_log.
- **Duplicate records.** Identical structures at the same pressure are merged (the higher-priority source is kept: paper > CompSC > alex).
- **Within-source conflicts.** Identical structures (matched with pymatgen's StructureMatcher) at the same pressure but with different Tc inside the *same* source have no priority rule, so both records of each such pair were dropped: GeH3 at 180 GPa, NaH6 at 500 GPa, ScH3 at 0 GPa. Other records that share a formula and pressure are different polymorphs and are kept.
- **Record fields:** `formula`, `chemsys`, `pressure_GPa`, `Tc_K`, `atoms` are used by the model; `lambda`, `wlog_K` are reference-only; `provenance` records the origin (source, original id, DOI).
- **Group split.** Chemical systems are split into 5 groups, balanced by record count, Tc distribution, and pressure. Source is not used when forming the groups. Each record appears exactly once in a test set, and the test/validation chemical systems of a group never appear in its training set.

## Model

**Final prediction = 0.5 · ALIGNN×3 + 0.5 · MACE.** ALIGNN is the core; MACE complements it.

- **ALIGNN×3** — three ALIGNN graph networks on the crystal structure (12 nearest neighbours within 8 Å plus bond angles; each atom's up-to-92 element features are extended with pressure, volume per atom, and the hydrogen fraction). They differ by one setting: `huber` — base, `huber_log` — trained on log(1 + Tc), `huber_256` — hidden width 256 instead of 128. The prediction is the mean of the three. Training: AdamW, lr 1e-3, OneCycle schedule, 80 epochs, batch size 32.
- **MACE** — structural features from the pretrained MACE-MP-0 (medium) interatomic potential (trained on ~1.5M DFT energy/force calculations). Per-atom feature vectors are pooled by mean over all atoms, over H atoms, and over non-H atoms, then extended with the same three global features. A small head network (2 hidden layers) is trained on top, AdamW, up to 200 epochs, averaged over 3 seeds. MACE itself is **not** fine-tuned — it is a fixed feature extractor.
- **Loss** for all networks: Huber on normalized Tc (threshold 0.2 normalized units ≈ 9 K). This fits the heavily right-skewed Tc distribution better than MSE (over-sensitive to rare high-Tc outliers) or L1 (indifferent to error size).
- **Epoch selection:** lowest MAE on the validation split.
- **Evaluation:** each record is predicted only by the models of the group where it sits in the test split — models that never saw it. Metrics (MAE, RMSE, R²) are reported over all 1,915 records together.

## Results

Test set, all 1,915 records, 5-fold cross-validation by chemical system:

| | MAE, K | RMSE, K | R² |
|---|---|---|---|
| **0.5 · ALIGNN×3 + 0.5 · MACE (final)** | **13.31** | **25.40** | **0.703** |

