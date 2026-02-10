# BRCA Phase-1 — Week 1 Lab Notebook

> This document records the exact steps, decisions, issues, and corrections made during BRCA Phase-1 cohort construction.

---

## Project Overview

- **Project:** Image-based molecular subtype prediction  
- **Cancer:** TCGA-BRCA  
- **Phase:** Phase-1 (data foundation)  
- **Goal:** Build a clean, reproducible paired cohort (RNA-seq + WSI) suitable for downstream ML and PAM50 subtyping  

---

## Week 1 Objective

Establish a **production-quality data foundation** by:

- Querying **all available TCGA-BRCA RNA-seq and WSI data**
- Enforcing **RNA ∩ WSI pairing at the case level**
- Resolving **real-world TCGA issues** (duplicates, multiple files per case)
- Producing **explicit, versioned artifacts** that can be audited and reused

> This week intentionally focused on **data correctness and reproducibility**, not modeling.

---

## Environment & Repository Setup

- **OS:** Windows  
- **IDE:** VS Code  
- **Python environment:** conda env `gdc`  
- **Data source:** GDC / TCGA  
- **Repository:** `histo-to-omics-framework`

### Key directory structure

- `scripts/gdc/` — GDC access (shared, cancer-agnostic)
- `scripts/shared/` — generic data plumbing (pairing, QC, manifests)
- `scripts/brca_phase1/` — BRCA-specific biology and labels
- `data/metadata/brca_phase1/` — manifests and metadata
- `data/processed/brca_subtyping/` — paired cohort artifacts
- `outputs/brca_subtyping/` — logs, QC outputs, tables
- `reports/brca_phase1/` — human-readable documentation

---

## Step-by-Step Work Log

### Step 0 — Design decision: shared vs cancer-specific code

**Decision**

- Separate **data access**, **generic processing**, and **cancer-specific biology**

**Outcome**

- Shared GDC scripts → `scripts/gdc/`
- Shared pairing / QC scripts → `scripts/shared/`
- BRCA-specific logic → `scripts/brca_phase1/`

**Rationale**

- Avoids refactoring when adding additional cancers
- Matches industry-grade multi-cancer pipelines

---

### Step 1 — Query full TCGA-BRCA metadata from GDC

**Script**

- `scripts/gdc/gdc_build_manifests.py`

**Actions**

- Queried RNA-seq (STAR-Counts, Primary Tumor)
- Queried WSI (SVS slide images)
- No artificial limits (`--n all`)

**Results**

- RNA-seq files returned: **1111**
- WSI slide files returned: **3112**

**Artifacts**

- `brca_rnaseq_manifest.tsv`
- `brca_rnaseq_metadata.tsv`
- `brca_wsi_manifest.tsv`
- `brca_wsi_metadata.tsv`

---

### Step 2 — Build paired cohort (RNA ∩ WSI)

**Script**

- `scripts/shared/make_paired_cohort.py`

**Logic**

- Group by `case_id`
- Track RNA presence and WSI presence per case
- Record number of WSI slides per case

**Results**

- Cases with RNA: **1095**
- Cases with WSI: **1098**
- **Paired cases (RNA ∩ WSI): 1095**
- WSI-only cases: **3**
- WSI slides per case: **1–9**

**Artifact**

- `data/processed/brca_subtyping/brca_paired_cohort.csv`

---

### Step 3 — QC of paired cohort

**Script**

- `scripts/shared/qc_paired_cohort.py`

**Purpose**

- Validate cohort composition
- Persist QC output to logs

**Results**

- Total cases in table: **1098**
- Paired cases: **1095**
- WSI slide count range confirmed: **1–9**

**Artifact**

- `outputs/brca_subtyping/logs/paired_cohort_qc.txt`

---

### Step 4 — Restrict RNA manifest to paired cases

**Observation**

- RNA metadata contained **1111 RNA files**
- Paired cases were only **1095**

**Script added**

- `scripts/shared/build_manifest_from_paired_cases.py`

**Outcome**

- RNA manifest restricted to paired cases
- Still **1111 RNA rows**, indicating duplicate RNA files per case

**Artifact**

- `brca_rnaseq_manifest_paired.tsv`

---

### Step 5 — Critical issue discovered: multiple RNA-seq files per case

**Observation**

- Paired cases: **1095**
- RNA files: **1111**
- **11 cases had more than one RNA-seq file**

This is a known TCGA characteristic (multiple aliquots / reprocessing).

**Decision**

- Enforce **exactly one RNA-seq file per case**
- Use a **deterministic and reproducible rule**  
  (sorted file name → first file)

**Script added**

- `scripts/shared/build_one_rnaseq_per_case_manifest.py`

**Results**

- Final RNA manifest: **1095 rows**
- One RNA-seq file per paired case
- Duplicate cases handled transparently

**Artifacts**

- `brca_rnaseq_manifest_one_per_case.tsv`
- `outputs/brca_subtyping/tables/rnaseq_one_per_case_selection.csv`

> This decision is documented and reversible if needed.

---

## Current State (End of Week 1)

### Cohort summary

- **Cancer:** BRCA  
- **Paired cases (RNA + WSI):** 1095  
- **RNA per case:** exactly 1  
- **WSI per case:** 1–9 slides  

### Ready-to-use artifacts

- Clean paired cohort table
- Deterministic RNA download manifest
- QC logs
- Explicit record of duplicate resolution

No modeling assumptions have been made yet.

---

## Key Lessons / Decisions Logged

- TCGA **cannot be assumed** to have one RNA-seq file per case
- Pairing must be done at the **case level**, not file level
- Deterministic rules are essential for reproducibility
- All ad-hoc one-liners were converted into scripts

---

## Next Steps (Week 2)

- Download RNA-seq STAR-Counts for 1095 cases (scripted)
- Build expression matrix
- Apply normalization (documented)
- Run PAM50 subtyping
- Produce a non-technical summary for supervisor review
