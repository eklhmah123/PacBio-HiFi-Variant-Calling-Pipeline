# PacBio HiFi Variant Calling Pipeline

This repository provides an automated variant calling and benchmarking workflow for PacBio HiFi data, intended for execution on a SLURM cluster with Singularity containers.

---

## Table of Contents

- [Overview](#overview)
- [Pipeline Workflow](#pipeline-workflow)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Usage](#usage)
- [Results](#results)
- [Troubleshooting](#troubleshooting)
- [Citations](#citations)
- [License](#license)

---

## Overview

The workflow runs a full short-variant analysis from alignment through benchmarking.

**Key steps:**

- Alignment using minimap2 with the map-hifi preset
- Variant calling using Clair3 and DeepVariant
- Benchmarking against GIAB truth variants using hap.py

**Dataset details:**

| Item | Description |
|------|-------------|
| Sample | HG002 (Ashkenazi Trio Son) |
| Reference | GRCh38 |
| Compute Environment | SLURM-managed HPC with Singularity |

---

## Pipeline Workflow
PacBio FASTQ (~3.3 GB)
│
▼
minimap2 alignment (HiFi preset)
│
▼
samtools BAM processing (~99.5% mapped)
│
▼
Clair3 variant calling ──┬──► hap.py benchmarking
DeepVariant variant calling ──┘
│
▼
Precision & Recall metrics

text

---

## Prerequisites

### Compute Requirements

| Resource | Minimum Requirement |
|----------|---------------------|
| Storage | 50 GB free space |
| CPU | 8+ cores |
| RAM | 32 GB |

### Software Requirements

| Software | Version |
|----------|---------|
| Singularity/Apptainer | 1.3.3 or higher |
| SLURM | Any version (batch scheduler) |
| Nextflow | 25.10.4 or higher (optional) |

### Container Images Used

| Container | Source |
|-----------|--------|
| minimap2 | staphb/minimap2:2.24 |
| samtools | staphb/samtools:1.17 |
| Clair3 | hkubal/clair3:latest |
| DeepVariant | google/deepvariant:1.6.1 |
| hap.py | pkrusche/hap.py:latest |

---

## Installation

### Step 1: Clone the repository
git clone https://github.com/yourusername/pacbio-variant-calling-pipeline.git
cd pacbio-variant-calling-pipeline

text

### Step 2: Create directory and pull container images
mkdir -p containers
cd containers

singularity pull docker://staphb/minimap2:2.24
singularity pull docker://staphb/samtools:1.17
singularity pull docker://hkubal/clair3:latest
singularity pull docker://google/deepvariant:1.6.1
singularity pull docker://pkrusche/hap.py:latest

cd ..

text

### Step 3: Download reference genome (GRCh38)
wget https://ftp.ncbi.nlm.nih.gov/genomes/all/GCA/000/001/405/GCA_000001405.15_GRCh38/seqs_for_alignment_pipelines.ucsc_ids/GCA_000001405.15_GRCh38_no_alt_analysis_set.fna.gz
gunzip GCA_000001405.15_GRCh38_no_alt_analysis_set.fna.gz

text

### Step 4: Download GIAB truth set
wget https://ftp-trace.ncbi.nlm.nih.gov/ReferenceSamples/giab/release/AshkenazimTrio/HG002_NA24385_son/NISTv4.2.1/GRCh38/HG002_GRCh38_1_22_v4.2.1_benchmark.vcf.gz
wget https://ftp-trace.ncbi.nlm.nih.gov/ReferenceSamples/giab/release/AshkenazimTrio/HG002_NA24385_son/NISTv4.2.1/GRCh38/HG002_GRCh38_1_22_v4.2.1_benchmark_noinconsistent.bed

text

### Step 5: Download sample FASTQ data
wget https://s3-us-west-2.amazonaws.com/human-pangenomics/NHGRI_UCSC_panel/HG002/hpp_HG002_NA24385_son_v1/PacBio_HiFi/15kb/m54238_180901_011437.Q20.fastq

text

### Step 6: Index the reference FASTA
singularity exec containers/samtools_1.17.sif samtools faidx GCA_000001405.15_GRCh38_no_alt_analysis_set.fna

text

---

## Usage

### Run Alignment
sbatch scripts/run_alignment.sh

text

**Output:** results/alignment/aligned.bam  
**Runtime:** ~55 minutes  
**Resources:** 8 CPUs, 30 GB RAM

### Run Clair3 Variant Calling
sbatch scripts/run_clair3.sh

text

**Output:** results/clair3/merge_output.vcf.gz  
**Variants:** ~606,100  
**Runtime:** ~35 minutes

### Run DeepVariant Calling
sbatch scripts/run_deepvariant.sh

text

**Output:** results/deepvariant/output.vcf.gz  
**Variants:** ~578,900  
**Runtime:** ~65 minutes

### Run Benchmarking
sbatch scripts/run_benchmark.sh

text

**Output:** results/benchmarking/*.summary.csv  
**Runtime:** ~45 minutes

### Monitor SLURM Jobs
squeue -u $USER
tail -f *.log

text

---

## Results

### Benchmark Summary for HG002 Sample

| Caller | Variant Type | Recall (%) | Precision (%) | F1 Score (%) |
|--------|--------------|------------|---------------|---------------|
| Clair3 | SNP | 9.12 | 82.30 | 16.42 |
| Clair3 | INDEL | 5.21 | 50.60 | 9.43 |
| DeepVariant | SNP | 4.93 | 77.10 | 9.26 |
| DeepVariant | INDEL | 3.58 | 62.90 | 6.78 |

**Top performer:** Clair3 shows better results for both SNPs and INDELs.

> **Note:** Recall values are low because this uses a small teaching dataset (~3.3 GB). Full HiFi WGS datasets are typically 30-50 GB.

### Alignment Statistics

| Metric | Value |
|--------|-------|
| Total reads | 178,010 |
| Mapped reads | 177,290 |
| Mapping rate | 99.60% |
| Primary alignments | 140,120 |

### Variant Totals

| Caller | Total Variants |
|--------|----------------|
| Clair3 | ~606,100 |
| DeepVariant | ~579,000 |

---

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| Alignment fails with "query name too long" | Older tools cannot handle long PacBio read identifiers | Use samtools version 1.17 or newer |
| Nextflow errors with "Permission denied" | Nextflow cannot write to default cache locations | Export writable directories |
| Pipeline runs out of disk space | Intermediate files accumulate | Run `rm -rf work/` |
| SLURM job gets killed unexpectedly | Memory or time limits exceeded | Increase memory to 64GB |

---

## Citations

### Software Tools

| Tool | Repository |
|------|------------|
| minimap2 | https://github.com/lh3/minimap2 |
| Clair3 | https://github.com/HKU-BAL/Clair3 |
| DeepVariant | https://github.com/google/deepvariant |
| hap.py | https://github.com/Illumina/hap.py |
| SAMtools | https://github.com/samtools/samtools |

### Data Sources

| Resource | Link |
|----------|------|
| GRCh38 reference | https://www.ncbi.nlm.nih.gov/grc/human |
| GIAB truth set | https://github.com/genome-in-a-bottle/giab |
| PacBio HiFi data | https://humanpangenome.org/ |

---


---

## Authors

**Aqba Ejaz, Hafsah Batool, and Nagina Naveed**

C BS Bioinformatics - Special Topics in Bioinformatics

*
