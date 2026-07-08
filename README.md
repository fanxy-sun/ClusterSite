# ConSite: Functional Context Modeling for Enzyme Active Site Annotation

Implementation of **ConSite** — a functional **Con**text-aware framework for enzyme active **Site** annotation.

> Accurate annotation of enzyme active sites is essential for understanding catalytic mechanisms and enabling enzyme engineering, yet remains challenging as local sequence and structural signatures are often insufficient for site type assignment. ConSite explicitly incorporates enzyme functional hierarchy and enzymatic reaction information as structured functional context to resolve this ambiguity.

---

## Overview

ConSite operates through three synergistic modules:

- **Functional Context Injection (FCI)**: Dynamically recalibrates residue representations conditioned on the enzyme's reaction SMILES, using reaction-conditioned normalization and multi-scale dilated convolutions to highlight reaction-critical regions.
- **Hierarchical Prototype Alignment (HPA)**: Captures shared catalytic patterns within EC hierarchy-anchored clusters through prototypical learning. Residues are annotated by their distance to per-EC, per-site-type prototypes.
- **Uncertainty-Aware Hierarchical Fallback (UHF)**: Performs entropy-guided decision fallback across functional levels — falling back from 4-digit to 3-digit EC clusters when prediction confidence is low.

Together, these modules formulate active site prediction as **context-aware functional semantic alignment**, enabling ConSite to handle class imbalance, few-shot scenarios, and distant homologs.



---

## Installation

```bash
# Clone the repository
git clone https://github.com/your-org/ConSite.git
cd ConSite

# Create the conda environment from the provided file
conda env create -f environment.yml
conda activate ConSite
```

---

## Dataset

We constructed an enzyme active site annotation dataset from the Swiss-Prot section of UniProtKB. The final dataset contains **102,080 enzyme samples** from ~95,000 proteins, covering **3,835 EC numbers** across all seven enzyme classes. Over **700,000 functional residues** are annotated: 83.7% binding sites, 12.7% catalytic sites, and 3.6% other sites.

The dataset is split into **95,406 training / 3,262 validation / 3,412 test** samples using CD-HIT at 60% sequence identity. Reaction SMILES are assigned via a hierarchical matching strategy prioritizing curated RHEA annotations.

**Dataset CSV format:**

| Field | Description |
|---|---|
| `rxn_smiles_single` | Reaction SMILES (substrate >> product) |
| `ec` | Enzyme Commission number |
| `Uniprot ID` | UniProt accession |
| `Sequence` | Amino acid sequence |
| `site_labels` | Annotated active site residue indices |
| `site_types` | Site type labels (1=binding, 2=catalytic, 3=other) |

Structural input is sourced from PDB experimental structures or AlphaFold2 predicted structures.

---

## Training

ConSite uses a **two-phase training strategy**:

**Phase 1** — jointly trains the multi-modal encoder and FCI blocks:

```bash
conda activate ConSite
python train.py \
  --config config/ClusterSite-SwissProt_Enzyme_CDHIT60-default.yaml \
  --switch 1
```

**Phase 2** — freezes Phase 1 weights, then trains the HPA module:

```bash
python train.py \
  --config config/ClusterSite-SwissProt_Enzyme_CDHIT60-default.yaml \
  --switch 2 \
  --part1_model_path ckpt/SwissProt_Enzyme_CDHIT60/default/ResidueEncoder/checkpoint_epoch_31.pt
```

The EC classifier (UHF) is trained independently:

```bash
python train.py \
  --config config/ClusterSite-SwissProt_Enzyme_CDHIT60-default.yaml \
  --switch ec
```

---

## Evaluation

```bash
python test.py \
  --config config/ClusterSite-SwissProt_Enzyme_CDHIT60-default.yaml \
  --part1_model_path ckpt/.../ResidueEncoder/checkpoint_epoch_31.pt \
  --part2_model_path ckpt/.../ActiveSiteEmbeddingNet/best_model_site.pt
```

Metrics reported: **Precision, Recall, FPR, MCC** (multi-class per site type), and **Precision, Recall, FPR, AUC, MCC** (binary).

---

## Project Structure

```
ConSite/
├── config/                   # YAML configuration files
├── model/
│   └── EvoSite.py            # Core model: FCI (RGMM), HPA (ActiveSiteEmbeddingNet)
├── data_loaders/             # Dataset and DataLoader
├── common/utils.py           # Shared utilities
├── data/                     # Dataset files (not included)
├── ckpt/                     # Model checkpoints (not included)
├── environment.yml           # Conda environment
├── train.py                  # Training entry point
└── test.py                   # Evaluation entry point
```
