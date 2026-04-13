# PacBio HiFi Variant Calling Pipeline

This repository provides an automated variant calling and benchmarking workflow for PacBio HiFi data, intended for execution on a SLURM cluster with Singularity containers.

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

## Overview

The workflow runs a full short-variant analysis from alignment through benchmarking:

- **Alignment**: minimap2 using the `map-hifi` preset
- **Variant calling**: Clair3 and DeepVariant to generate independent callsets
- **Benchmarking**: evaluation against GIAB truth variants using hap.py

**Dataset**: HG002 (Ashkenazi Trio Son)  
**Reference**: GRCh38  
**Compute**: SLURM-managed HPC with Singularity

## Pipeline Workflow
Start from PacBio FASTQ (~3.3 GB)
│
▼
Align reads using minimap2 (HiFi preset) and process BAM with samtools
(~99.5% mapped)
│
▼
Run variant callers on GRCh38:
• Clair3 (HiFi)
• DeepVariant (PACBIO)
│
▼
Compare results to GIAB truth using hap.py
│
▼
Obtain precision and recall metrics

text

## Prerequisites

### Compute environment

This pipeline is designed for an HPC setup, ideally with SLURM for job scheduling.

- **Storage**: ≥ 50 GB free
- **CPU**: 8+ cores recommended
- **RAM**: at least 32 GB

### Software dependencies

- Singularity/Apptainer (v1.3.3+)
- A batch scheduler such as SLURM
- Nextflow (v25.10.4+) to orchestrate the workflow (optional but recommended)

### Containerized tools

All required tools are run via containers:

- `staphb/minimap2:2.24`
- `staphb/samtools:1.17`
- `hkubal/clair3:latest`
- `google/deepvariant:1.6.1`
- `pkrusche/hap.py:latest`

## Installation

### Step 1 — Clone the repository

```bash
git clone https://github.com/yourusername/pacbio-variant-calling-pipeline.git
cd pacbio-variant-calling-pipeline
Step 2 — Fetch container images
Create a directory for containers and pull the required images via Singularity/Apptainer:

bash
mkdir -p containers
cd containers

singularity pull docker://staphb/minimap2:2.24
singularity pull docker://staphb/samtools:1.17
singularity pull docker://hkubal/clair3:latest
singularity pull docker://google/deepvariant:1.6.1
singularity pull docker://pkrusche/hap.py:latest
Step 3 — Download reference and benchmark resources
GRCh38 reference (NCBI)

bash
wget https://ftp.ncbi.nlm.nih.gov/genomes/all/GCA/000/001/405/GCA_000001405.15_GRCh38/seqs_for_alignment_pipelines.ucsc_ids/GCA_000001405.15_GRCh38_no_alt_analysis_set.fna.gz
gunzip GCA_000001405.15_GRCh38_no_alt_analysis_set.fna.gz
GIAB truth set

bash
wget https://ftp-trace.ncbi.nlm.nih.gov/ReferenceSamples/giab/release/AshkenazimTrio/HG002_NA24385_son/NISTv4.2.1/GRCh38/HG002_GRCh38_1_22_v4.2.1_benchmark.vcf.gz
wget https://ftp-trace.ncbi.nlm.nih.gov/ReferenceSamples/giab/release/AshkenazimTrio/HG002_NA24385_son/NISTv4.2.1/GRCh38/HG002_GRCh38_1_22_v4.2.1_benchmark_noinconsistent.bed
Sample FASTQ data

bash
wget https://s3-us-west-2.amazonaws.com/human-pangenomics/NHGRI_UCSC_panel/HG002/hpp_HG002_NA24385_son_v1/PacBio_HiFi/15kb/m54238_180901_011437.Q20.fastq
Step 4 — Index the reference FASTA
bash
singularity exec containers/samtools_1.17.sif \
samtools faidx GCA_000001405.15_GRCh38_no_alt_analysis_set.fna
Usage
Running the Full Pipeline
Alignment
Submit the alignment job to SLURM:

bash
sbatch scripts/run_alignment.sh
Output: results/alignment/aligned.bam

Estimated runtime: ~55 minutes

Requested resources: 8 CPUs, 30 GB RAM

Variant Calling — Clair3
Launch Clair3 variant calling:

bash
sbatch scripts/run_clair3.sh
Output: results/clair3/merge_output.vcf.gz

Variants identified: ~606,100

Estimated runtime: ~35 minutes

Variant Calling — DeepVariant
Run DeepVariant using the PacBio model:

bash
sbatch scripts/run_deepvariant.sh
Output: results/deepvariant/output.vcf.gz

Variants identified: ~578,900

Estimated runtime: ~65 minutes

Benchmarking
Compare callsets against the GIAB truth set:

bash
sbatch scripts/run_benchmark.sh
Output: results/benchmarking/*_benchmark.summary.csv

Estimated runtime: ~45 minutes

Monitoring SLURM Jobs
Check running/queued jobs and follow logs:

bash
# Job queue
squeue -u $USER

# Watch log output
tail -f *.log
Results
Benchmark Summary (HG002)
Below is a performance snapshot for the HG002 sample, comparing calls against the GIAB truth set.

Caller	Variant Class	Recall (%)	Precision (%)	F1 (%)
Clair3	SNP	9.12	82.30	16.42
Clair3	INDEL	5.21	50.60	9.43
DeepVariant	SNP	4.93	77.10	9.26
DeepVariant	INDEL	3.58	62.90	6.78
Top performer: Clair3 shows better overall results on this subset for both SNPs and INDELs.

Note: Recall is expected to be lower here because this run uses a small teaching dataset (~3.3–3.5 GB) rather than a full-genome HiFi dataset. Full HiFi WGS inputs are commonly on the order of 30–50 GB.

Alignment Statistics
Total reads: 178,010

Mapped reads: 177,290 (~99.60% mapped)

Primary alignments: 140,120

Variant Totals
Clair3: ~606,100 variants reported

DeepVariant: ~579,000 variants reported

Troubleshooting
Common Problems & Fixes
Problem	Cause	Fix
Alignment fails with "query name too long"	Some tools choke on long PacBio read identifiers	Use a newer samtools release (v1.17 or later)
Nextflow errors with "Permission denied"	Nextflow can't write to default cache/temp locations	Export writable directories: export NXF_HOME="$HOME/.nextflow" export NXF_TEMP="$HOME/.nextflow/tmp"
Pipeline runs out of disk space	Intermediate files (Nextflow work directory) grow quickly	Clean up: rm -rf work/ or route to scratch storage
SLURM job gets killed unexpectedly	Memory or time limits exceeded	Inspect logs and increase memory: #SBATCH --mem=64G
Citations
Software References
minimap2 — Li, H. minimap2 (software). GitHub repository: https://github.com/lh3/minimap2

Clair3 — HKU-BAL. Clair3 (software). GitHub repository: https://github.com/HKU-BAL/Clair3

DeepVariant — Google. DeepVariant (software). GitHub repository: https://github.com/google/deepvariant

hap.py — Illumina / GA4GH. hap.py (software). GitHub repository: https://github.com/Illumina/hap.py

SAMtools — SAMtools developers. SAMtools (software). GitHub repository: https://github.com/samtools/samtools

Data Sources
Reference Genome (GRCh38) — Genome Reference Consortium. GRCh38 assembly resources: https://www.ncbi.nlm.nih.gov/grc/human

Truth Set (GIAB HG002 v4.2.1) — Genome in a Bottle Consortium (NIST). GIAB releases: https://github.com/genome-in-a-bottle/giab

Sample Data (PacBio HiFi HG002 / HPRC) — Human Pangenome Reference Consortium. Data portal: https://humanpangenome.org/





