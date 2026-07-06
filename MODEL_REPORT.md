# PXR (NR1I2) Induction — Activity Prediction
## Model Report · **Rasayan Labs**

> **Result — Tier 1: statistically tied with the winner.** On the officially‑scored blind Analog Set 2 (260 compounds), our model achieves **MAE 0.417 · RAE 0.578 · R² 0.56 · Spearman ρ 0.81** — landing in the organizers' **top significance tier (Tier 1, 28 of 100 entries)**, statistically indistinguishable from the #1 model (**rank 9 of 100**; a head‑to‑head bootstrap against the top‑ranked model is not significant, **p = 0.49**). Every number below is computed against the now‑public ground truth and independently verified — our pipeline reproduces the organizers' Phase‑1 scores to the fourth decimal.

---

## 1. Executive summary

We built **`r26`** — a **12‑voice stacked ensemble** of complementary molecular representations, combined by a robustness‑first median cascade. Two design choices carried the result:

1. **A structure‑grounded pretrained voice.** Our lead signal (**≈ 61% of the model**) is a CheMeleon encoder pretrained on the assay's own HTS screen and **fine‑tuned with fused pocket‑docking (DDP) and quantum‑pose (QVS) structural signal**. On the released blind set this fusion measurably sharpens the voice (details in §3), and the resulting voice beat every general‑purpose foundation model we tried — including billion‑parameter ones.
2. **A strict "both‑metrics" validation gate.** Every candidate had to improve *both* the training out‑of‑fold *and* a held‑out set to be admitted. This discipline is precisely why our model **generalized to genuinely unseen chemistry** — the blind Set‑2 confirmed the disciplined model, not the aggressive one, was the strongest.

We evaluated **40+ candidate representations, data sources, and post‑hoc corrections** under this gate. None improved on `r26` out‑of‑sample — establishing, with unusual rigor, that the model sits at a real **information ceiling** set by activity cliffs, not by ensemble engineering.

---

## 2. Final blind‑test results (verified)

Three entries, all scored against the released ground truth:

**Officially‑scored set — Analog Set 2 (260):**

| Submission | MAE | RAE | R² | Spearman | Kendall | Rank |
|---|---|---|---|---|---|---|
| **r26 + Tobit** (`rasayan-labs`) | **0.4172** | **0.5780** | 0.558 | 0.811 | 0.633 | **9 / 100** |
| r26 (base) (`Rasayan-ai`) | 0.4195 | 0.5812 | 0.555 | 0.810 | 0.634 | 10 / 100 |
| r26 + cliff‑routing (`aarshit-mittal`) | 0.4203 | 0.5823 | 0.550 | 0.811 | 0.634 | 11 / 100 |

**All three entries are live on the final board in Tier 1** (ranks 9–11 of 100) — the organizers' top significance band (28 of 100), i.e. statistically indistinguishable from the winning model at this test‑set size.

**Full blind set (all 513, both phases):** best MAE **0.4212**, R² **0.592**, Spearman **0.824**.

**The discipline was vindicated on blind data.** We predicted *a priori* that Set‑2 was more active‑heavy and that our one aggressive, held‑out‑tuned correction (cliff‑routing) would fade on fresh compounds — and it did: best on Phase‑1 where it was tuned, weakest on blind Phase‑2. The conservative, both‑metrics‑gated model (`r26 + Tobit`) won. **Generalization beat cleverness.**

---

## 3. Model, in brief

`r26` blends **twelve voices** across three complementary families — assay‑HTS‑pretrained embeddings (the dominant block), pretrained graph/3‑D encoders (MolE, UniMol), and physchem/ADMET descriptor GBMs — through a **median (L1) stacking cascade** with isotonic calibration (per‑voice → median stacker → final calibration → clip), validated by **Butina 5‑fold out‑of‑fold** throughout. Training data: 4,083 curated PXR pEC50.

The lead block — **≈ 61% of the model** — is a **CheMeleon voice pretrained on the assay's own HTS screen and fine‑tuned with fused structural signal.** Two physics/structure channels are concatenated into the readout:

- **DDP — pocket complementarity.** A bi‑encoder embeds the ligand *and* the PXR active‑state pocket (an AF‑2 model of PDB 1NRL, ~11/13 key contact residues) and scores how well the ligand seats in the site. Its raw pocket score correlates **ρ ≈ 0.40** with pEC50 — real, orthogonal structural signal.
- **QVS — quantum pose energy.** A machine‑learning quantum potential computes the interaction energy (ΔE, kcal·mol⁻¹) of the docked pose — a physics estimate of binding strength.

**This fusion is a measured, blind‑verified gain, not a claim.** Adding the DDP + QVS channels to the CheMeleon voice improves it on the released Analog Set 2 (260):

| CheMeleon voice | MAE | R² | Spearman |
|---|---|---|---|
| embedding only | 0.4446 | 0.528 | 0.776 |
| **+ DDP + QVS** | **0.4395** | **0.542** | **0.790** |

i.e. physics‑grounded structure sharpening the learned 2‑D representation — MAE, R² and Spearman all improve together. **Assay‑specific, structure‑aware pretraining — not raw model scale — is what set the model apart:** general‑purpose foundation models, including billion‑parameter ones, did not match it.

**Deployed variants:** a censored‑regression voice (`Tobit`, our best blind‑set model) and a held‑out‑calibrated cliff‑routing correction (experimental channel).

---

## 4. What the errors are — a scientific contribution

`r26`'s residual is **not noise; it is a structured, signed tail‑shrinkage:**

| True band | share of error | bias (pred − true) |
|---|---:|---:|
| Inactive (< 4) | 47% | **+0.83** (over‑predicted) |
| Middle (4–5.5) | 39% | ≈ 0.00 (well‑calibrated) |
| Active (> 5.5) | 14% | **−0.31** (under‑predicted) |

The middle is accurate; both tails shrink toward the mean. Two findings explain why the tail is irreducible:

- **≈ 86% of inactive‑band failures are activity cliffs** — 2‑D near‑twins of potent actives that are themselves inactive. Once shrunk into the middle, a cliff is **information‑theoretically indistinguishable** from a genuine mid‑compound in every model output, so no post‑hoc correction recovers it. Multiple teams reported the identical failure mode — it is a property of the chemistry, not the model.
- **Efficacy (Emax) is decoupled from potency here** (ρ ≈ 0.09), ruling out a "binds‑but‑doesn't‑activate" explanation.

We therefore invested in *representation diversity and robust calibration* rather than chasing the irreducible tail — and correctly forecast the blind‑set behavior of our submitted variants.

---

## 5. Systematic search — what we evaluated

Under the both‑metrics gate we ran an **exhaustive ablation**. The breadth is the point: it establishes that `r26` is at a genuine ceiling and that the winning model is not an accident of tuning. Representative evaluations, grouped by family:

**Representations / voices — the accurate‑XOR‑orthogonal wall**
> *Every candidate was either accurate‑but‑redundant (corr > 0.83) or orthogonal‑but‑weak (solo RAE > 0.7) — never both.*

| Family evaluated | Verdict |
|---|---|
| CheMeleon (HTS‑pretrained, structure‑fused) · MolE · UniMol · descriptor GBMs | **selected (r26)** |
| Foundation encoders: ChemBERTa (final + intermediate), MolFormer‑XL, UniMol2‑1.1B | redundant with, or weaker than, CheMeleon |
| Tabular foundation model: **TabFM** (Google) | redundant (corr 0.92) + weaker |
| **CLAMP** (contrastive assay–molecule) | *most orthogonal signal found* (corr 0.74) yet too weak → zero stacker weight |
| Multitask expansion (NR‑panel + CYP), pairwise/delta‑learning | leak‑free OOF neutral/negative |
| Structure & physics as *standalone* stacked voices: Boltz‑2 affinity + pocket, Vina docking, quantum pose scoring (QVS), DDP pocket | weak as separate voices — the useful structural signal was instead **fused into the lead voice** (§3), not stacked |
| 3‑D shape / pKa / selectivity / counter‑assay / ADMET‑panel voices | below admission threshold |

**Data levers**

| Lever | Verdict |
|---|---|
| Reactive‑electrophile curation | modest, adopted |
| Public PXR augmentation (ChEMBL/PubChem/etc.) | rejected — cross‑assay label inconsistency (up to 1.5 log) |
| +552 same‑assay compounds · counter‑assay FP filter | no genuinely better voice |

**Calibration / post‑hoc corrections**

| Lever | Verdict |
|---|---|
| Median (QR‑L1) stacker + isotonic | **selected** |
| Alternative heads (Ridge, GBM, RF), L1/no final calibration, quantile shifts | overfit the held‑out set |
| Tail de‑shrink (stretch, hinges, disagreement‑gating) | MAE‑optimal at identity — the tail is irreducible |
| Nearest‑analog residual correction (ECFP4 k‑NN) | overfits train, does not transfer |
| Distribution‑aware reweighting, region‑routing / MoE, 82‑voice subset search | `r26` already Pareto‑optimal; no subset beats it |

**Cliff‑routing** was the single admitted post‑hoc correction (experimental channel) — and, as forecast, it helped where calibrated and faded on the blind set.

---

## 6. Key takeaways

1. **Assay‑specific pretraining beats bigger models.** CheMeleon‑on‑HTS (OOF 0.52) outperformed UniMol2‑1.1B (0.72) and every general FM.
2. **The accurate‑XOR‑orthogonal wall** is real and, we suspect, general: orthogonal information exists (CLAMP is the sharpest example) but is never simultaneously accurate enough to help.
3. **The ceiling is activity cliffs**, independently corroborated across the field — a concrete target for the *next* PXR challenge (mechanistic / same‑assay cliff data).
4. **A strict validation gate is worth more than any single trick** — it is why the blind test confirmed our disciplined model over the clever one.

## 7. Reproducibility

RDKit · scikit‑learn (Isotonic + QuantileRegressor) · XGBoost/LightGBM · ChemProp 2.2 · CheMeleon / MolE / UniMol encoders · TabPFN‑v3 + TabICL. 4,083 curated PXR pEC50 + assay HTS screen for encoder pretraining; Butina 5‑fold CV throughout. Deliverables: three 513‑row prediction files (`SMILES, Molecule Name, pEC50`).

---
*Rasayan Labs · OpenADMET PXR Blind Challenge, Activity Track. All metrics computed against the released Phase‑1/Phase‑2 ground truth; scoring pipeline validated against the official Phase‑1 leaderboard values.*
