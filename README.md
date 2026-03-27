# SIP Proteomics Pipeline for _Myo_-Inositol Labeling

## Overview

This repository contains a Jupyter-based workflow for analyzing LC-MS/MS data generated from a stable isotope probing (SIP) experiment using _myo_-inositol. The pipeline processes raw mass spectrometry data, performs peptide/protein identification using Sipros4, and detects isotopic labeling (¹³C incorporation) at the protein level. In this work, different groups were created based on hen, intestinal section and incubation time so for each group, different reference database was created. Hence, this workflow has been modified accordingly from: https://github.com/thepanlab/Sipros4

The workflow is designed for high-performance computing (HPC) environments using PBS job scheduling.

---

## Workflow

Data were analyzed using the High Performance and Cloud Computing (BINAC) cluster at the Zentrum für Datenverarbeitung, University of Tübingen (Baden-Württemberg, Germany).

All bioinformatics tools were installed using Conda environments. Job execution was handled via PBS scheduler (qsub) within JupyterLab.


# Table of Contents

* RAW to FT conversion
* Sample organization
* Database preparation
* Unlabeled search (¹²C samples)
* Post-processing
* Reference sample selection
* Reduced database generation
* Configuration file setup
* Group mapping
* Labeled search (¹³C SIP analysis)

---

# RAW to FT Conversion

Raw mass spectrometry (LC-MS/MS) files are converted into `.FT1` (MS1 scan) and `.FT2` (MS2 scan) formats using `Raxport.exe`.

```
mono Raxport.exe \
  -i raw/sample.raw \
  -o ft/sample \
  -m 1
```

---

# Sample Organization

Samples were categorized into three conditions:

* No _myo_-inositol (baseline control)
* ¹²C _myo_-inositol (unlabeled control)
* ¹³C _myo_-inositol (labeled condition)

Each condition was performed in triplicate across:

* 3 hens
* 2 gut sections
* Multiple incubation time points

A **group** is defined as:

* Same hen
* Same gut section
* Same incubation time

---

# Database Preparation

A protein FASTA database is prepared with reverse decoy sequences for false discovery rate (FDR) estimation. It is suggested to also attach cRAP protein sequences for the contaminants.
For this project, chicken-gut-v1-0-1 (protein_catalogue 95) from MGnify EBI was used. 

```
cp database.fasta fasta/working_db.fasta
```

---

# Unlabeled Search (¹²C Samples)

All ¹²C samples were searched against the full protein database using SiprosEnsembleOMP. This step identifies proteins present in each sample without considering isotopic labeling.

```
SiprosEnsembleOMP \
  -c config.cfg \
  -w ft/ \
  -o regular/sample_output
```

---

# Post-processing

Search outputs were processed to:

* Filter peptide-spectrum matches (PSMs)
* Assemble proteins
* Estimate false discovery rate (FDR)
* Generate spectral counts

```
python sipros_post_processing.py \
  -i regular/sample_output \
  -o regular/processed
```

---

# Reference Sample Selection

For each group, one ¹²C sample was selected based on:

* Highest number of identified proteins

Two output files were compared between samples for the selection: 

1. .ProRefineFDR.txt : contains total proteins after filtering (FDR = 0.005%)
2. .ProCountSummary.txt : contains PSM counts

Sample with highest PSM count was selected since it indicate superior data quality providing more confident peptide identifications and a richer dataset.

This sample was used to construct a reduced database per group for SIP analysis.
This ensures a representative and high-confidence protein set for downstream analysis.

---

# Reduced Database Generation

A group-specific protein database was generated from the selected reference sample.

```
python build_reduced_db.py \
  -i regular/processed/reference_sample \
  -o fasta/reduced_db.fasta
```

---

# Configuration File Setup

Sipros configuration files were updated to use the reduced database.

```
cp template.cfg configs/group.cfg
# manually update database path
```

---

# Group Mapping

Samples were grouped based on:

* Hen
* Section
* Time point

Each group included:

* 3 × No MI samples
* 3 × ¹²C samples
* 3 × ¹³C samples

---

# Labeled Search (¹³C SIP Analysis)

All samples within each group were searched against the corresponding reduced database using SiprosV4OMP. This step identifies proteins incorporating ¹³C isotopes.
Additional processing includes:
* PSM filtering
* Protein assembly
* SIP abundance clustering
* Protein-level FDR refinement
* Quantification of isotopic incorporation

```
SiprosV4OMP \
  -c configs/group.cfg \
  -w ft/ \
  -o sip/group_output
```

---

# SIP Post-processing

Labeled search results were processed to:

* Filter labeled PSMs
* Assemble proteins
* Perform SIP abundance clustering
* Estimate protein-level FDR

```
python sipros_sip_processing.py \
  -i sip/group_output \
  -o sip/processed
```

---

# Output files

## Files after unlabeled search
1. PSM tab file (.tab) : List of all matches between spectra and possible peptide sequence
2. Filtered PSM file (.psm.txt) : Refined list after removing low-confidence matches
3. Protein assembly file (.pro.txt) : Groups peptides into their corresponding protein groups. Helps to identify all proteins found in the sample
4. Refined Protein FDR file (.proRefineFDR.txt) : Ensures the protein identification as per acceptable FDR
5. Spectra count file (.SPcount.txt) : Shows number of spectra each protein got showing the abundance of protein in sample

## Filter after labeled search
1. PSM file (.psm.txt) : Contains info about PSMs and holds isotopic abundance of PSMs and peptides.
2. Clustered Protein file (.pro.cluster.txt) : Includes isotopic enrichment levels (average enrichment) for each protein. Aids in identifying labeled proteins in the sample.
3. Isotopic abundance file (.LabelPCTcount.txt) : Contains % isotopic abundance of proteins and peptides across raw data files.
5. Median isotopic abundance file (medianPCTcount) : Median isotopic abundance for each proteins, aggregated from multiple PSMs. Helps to eliminate teh influence of outliers.
6. Labeled PSM count file (labeledCount) : The count of labeled PSMs supporting each protein across raw files. Reflects the extent to which proteins have been isotopically labeled.

### For final analysis, ".pro.cluster" file is used. 
"AverageEnrichmentLevel" column needs to be multiplied by 1000 to get the Average enrichment percentage.

# Control-Based Filtering

This was done after the processing. 
In R, "labeled" proteins identified in:

* No MI samples
* ¹²C samples

were used to remove false-positive labeling signals from ¹³C samples during downstream analysis and for filtering background isotopic signals.

Other filters included Average Enrichment threshold of 5 or more than 5 i.e. any protein with insufficient labeling is removed.

---

# Notes

* Paths must be adapted to the local HPC environment
* Job submission is handled via custom bash wrappers in Jupyter
* File naming consistency (`SampleID.FT2`) is required
* The config files and sipros files and scripts can be downloaded from : https://github.com/thepanlab/Sipros4

---

## Abbreviations: 
PSM : Peptide Spectrum Match
FDR : False Discovery Rate
MI : Myo-inositol

## Directory Structure

```
workspace/
├── fasta/        # Protein databases - suggested to also merge contaminants database (i.e. cRAP protein sequences)
├── raw/          # Raw MS data
├── ft/           # Converted FT1/FT2 files
├── regular/      # Unlabeled search outputs
├── sip/          # Labeled search outputs
├── configs/      # Generated config files
├── sipros/       # Sipros binaries and templates
```

---

## Requirements

* HPC environment with PBS scheduler
* Conda environment with:

  * Python (compatible with Sipros scripts)
  * R (for downstream analysis)
* Sipros (Ensemble + V4)
* Mono (for Raxport)

---
