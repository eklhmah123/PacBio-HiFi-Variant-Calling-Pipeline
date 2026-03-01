PacBio HiFi Variant Calling Pipeline

This repository provides an automated variant calling and benchmarking workflow for PacBio HiFi data, intended for execution on a SLURM cluster with Singularity containers.

Table of Contents

•	Overview
•	Pipeline Workflow
•	Prerequisites
•	Installation
•	Usage
•	Results
•	Troubleshooting
•	Citations
•	License

Overview
The workflow runs a full short-variant analysis from alignment through benchmarking:

•	Alignment: minimap2 using the map-hifi preset

•	Variant calling: Clair3 and DeepVariant to generate independent callsets

•	Benchmarking: evaluation against GIAB truth variants using hap.py

Dataset: HG002 (Ashkenazi Trio Son)
Reference: GRCh38


Compute: SLURM-managed HPC with Singularity

Pipeline Workflow

1.	Start from PacBio FASTQ (~3.3 GB)
	
3.	Align reads using minimap2 (HiFi
  
5.	preset) and process BAM with samtools (~99.5% mapped)
	
7.	Run variant callers on GRCh38:
	
•	Clair3 (HiFi)

•	DeepVariant (PACBIO)

10.	Compare results to GIAB truth using hap.py to obtain precision and recall metrics
    
Prerequisites

Compute environment

This pipeline is designed for an HPC setup, ideally with SLURM for job scheduling.

•	Storage: ≥ 50 GB free
•	CPU: 8+ cores recommended
•	RAM: at least 32 GB
Software dependencies

•	Singularity/Apptainer (v1.3.3+)
•	A batch scheduler such as SLURM
•	Nextflow (v25.10.4+) to orchestrate the workflow (optional but recommended)
Containerized tools

All required tools are run via containers:
•	staphb/minimap2:2.24
•	staphb/samtools:1.17
•	hkubal/clair3:latest
•	google/deepvariant:1.6.1
•	pkrusche/hap.py:latest

Installation


Step 1 — Clone the repository
git clone https://github.com/<your-username>/pacbio-variant-calling-pipeline.git
cd pacbio-variant-calling-pipeline
Step 2 — Fetch container images
Create a directory for containers and pull the required images via Singularity/Apptainer:
mkdir -p containers

cd containers

singularity pull docker://staphb/minimap2:2.24
singularity pull docker://staphb/samtools:1.17
singularity pull docker://hkubal/clair3:latest
singularity pull docker://google/deepvariant:1.6.1
singularity pull docker://pkrusche/hap.py:latest
Step 3 — Download reference and benchmark resources

# GRCh38 reference (NCBI)



# Reference genome (GRCh38)
wget https://ftp.ncbi.nlm.nih.gov/genomes/all/GCA/000/001/405/GCA_000001405.15_GRCh38/seqs_for_alignment_pipelines.ucsc_ids/GCA_000001405.15_GRCh38_no_alt_analysis_set.fna.gz
gunzip GCA_000001405.15_GRCh38_no_alt_analysis_set.fna.gz
# GIAB truth set
wget https://ftp-trace.ncbi.nlm.nih.gov/ReferenceSamples/giab/release/AshkenazimTrio/HG002_NA24385_son/NISTv4.2.1/GRCh38/HG002_GRCh38_1_22_v4.2.1_benchmark.vcf.gz
wget https://ftp-trace.ncbi.nlm.nih.gov/ReferenceSamples/giab/release/AshkenazimTrio/HG002_NA24385_son/NISTv4.2.1/GRCh38/HG002_GRCh38_1_22_v4.2.1_benchmark_noinconsistent.bed
# Sample FASTQ data
wget https://s3-us-west-2.amazonaws.com/human-pangenomics/NHGRI_UCSC_panel/HG002/hpp_HG002_NA24385_son_v1/PacBio_HiFi/15kb/m54238_180901_011437.Q20.fastq

Step 4 — Index the reference FASTA

singularity exec containers/samtools_1.17.sif \
samtools faidx GCA_000001405.15_GRCh38_no_alt_analysis_set.fna



Usage
Running the Full Pipeline
1) Alignment
Submit the alignment job to SLURM:
sbatch scripts/run_alignment.sh
•	Output: results/alignment/aligned.bam
•	Estimated runtime: ~55 minutes
•	Requested resources: 8 CPUs, 30 GB RAM


3) Variant Calling — Clair3
Launch Clair3 variant calling:
sbatch scripts/run_clair3.sh
•	Output: results/clair3/merge_output.vcf.gz
•	Variants identified: ~606,100
•	Estimated runtime: ~35 minutes


5) Variant Calling — DeepVariant
Run DeepVariant using the PacBio model:
sbatch scripts/run_deepvariant.sh
•	Output: results/deepvariant/output.vcf.gz
•	Variants identified: ~578,900
•	Estimated runtime: ~65 minutes

6) Benchmarking
   
Compare callsets against the GIAB truth set:
sbatch scripts/run_benchmark.sh
•	Output: results/benchmarking/*_benchmark.summary.csv
•	Estimated runtime: ~45 minutes

Monitoring SLURM Jobs
Check running/queued jobs and follow logs:
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

•	Total reads: 178,010
•	Mapped reads: 177,290 (~99.60% mapped)
•	Primary alignments: 140,120

Variant Totals

•	Clair3: ~606,100 variants reported
•	DeepVariant: ~579,000 variants reported

Troubleshooting
Common Problems & Fixes
1) Alignment fails with “query name too long”
Cause: Some tools choke on long PacBio read identifiers.
Fix: Use a newer samtools release (recommended v1.17 or later) which handles long read names more reliably.
2) Nextflow errors with “Permission denied”
Cause: Nextflow can’t write to its default cache/temp locations on some systems.
Fix: Point Nextflow to writable directories in your home folder:
export NXF_HOME="$HOME/.nextflow"
export NXF_TEMP="$HOME/.nextflow/tmp"
3) Pipeline runs out of disk space
Cause: Intermediate files (especially Nextflow’s work directory) can grow quickly.
Fix: Clean up intermediates after major steps or route temporary output to scratch storage:
rm -rf work/   # removes the Nextflow work directory
4) SLURM job gets killed unexpectedly
Cause: Most often memory limits (or occasionally time limits) are exceeded.
Fix: Inspect SLURM stderr/stdout logs and request more memory in the job script, e.g.:
#SBATCH --mem=64G   # increase from ~32G

Citations (Software References)
Tools Used
•	minimap2 — Li, H. minimap2 (software). GitHub repository: https://github.com/lh3/minimap2 (accessed 2026-02-28).
•	Clair3 — HKU-BAL. Clair3 (software). GitHub repository: https://github.com/HKU-BAL/Clair3 (accessed 2026-02-28).
•	DeepVariant — Google. DeepVariant (software). GitHub repository: https://github.com/google/deepvariant (accessed 2026-02-28).
•	hap.py — Illumina / GA4GH. hap.py (software). GitHub repository: https://github.com/Illumina/hap.py (accessed 2026-02-28).
•	SAMtools — SAMtools developers. SAMtools (software). GitHub repository: https://github.com/samtools/samtools (accessed 2026-02-28).

Data Sources (References)
•	Reference Genome (GRCh38) — Genome Reference Consortium. GRCh38 assembly resources: https://www.ncbi.nlm.nih.gov/grc/human (accessed 2026-02-28).
•	Truth Set (GIAB HG002 v4.2.1) — Genome in a Bottle Consortium (NIST). GIAB releases: https://github.com/genome-in-a-bottle/giab (accessed 2026-02-28).
•	Sample Data (PacBio HiFi HG002 / HPRC) — Human Pangenome Reference Consortium. Data portal/resources: https://humanpangenome.org/ (accessed 2026-02-28).

License

This project is distributed under the MIT License. See LICENSE for details.

Author

Aqba Ejaz, Hafsah Batool and Nagina Naveed

Created for BS  Bioinformatics  - Special Topics in Bioinformatics

Date: February 2026



