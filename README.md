# NGS QC + Alignment Mini-Pipeline

Practice project to learn the basic NGS workflow: from raw reads (FASTQ) to an indexed alignment (BAM), covering quality control and trimming along the way.

## Goal
Understand and execute the full flow end to end: QC of raw reads → trimming → alignment → mapping statistics.

## Environment

Set up with Conda (Miniconda) on WSL (Ubuntu on Windows).

Tools and environment created with:
\`\`\`bash
conda create -n ngs-pipeline fastqc fastp bowtie2 samtools multiqc -y
\`\`\`

Activate with:
\`\`\`bash
conda activate ngs-pipeline
\`\`\`

### SRA Toolkit (for downloading data from SRA)

The `sra-tools` package installed via Bioconda was outdated (old TLS certificates, failing against NCBI). The official NCBI binary is used instead:

\`\`\`bash
cd ~
wget https://ftp-trace.ncbi.nlm.nih.gov/sra/sdk/current/sratoolkit.current-ubuntu64.tar.gz
tar -xzf sratoolkit.current-ubuntu64.tar.gz
echo 'export PATH=$PATH:$HOME/sratoolkit.3.4.1-ubuntu64/bin' >> ~/.bashrc
source ~/.bashrc
\`\`\`

See `NOTES.md` for details on the issue and why it was solved this way.

## Dataset

SRA accession: `SRR2584863` (E. coli REL606, paired-end). Downloaded into `raw_data/` as `SRR2584863_1.fastq` and `SRR2584863_2.fastq`.

## Folder structure

- `raw_data/` — original FASTQ files, unmodified
- `trimmed_data/` — FASTQ files after trimming with fastp
- `reference/` — reference genome for alignment
- `alignments/` — resulting SAM/BAM files
- `qc_reports/` — FastQC and MultiQC reports
- `scripts/` — pipeline scripts

## Usage

### 1. Initial quality control

\`\`\`bash
fastqc raw_data/SRR2584863_1.fastq raw_data/SRR2584863_2.fastq -o qc_reports/
\`\`\`

Reports available in `qc_reports/`. Key findings: overall good quality data, 
with expected quality drop and Nextera adapter content toward the read tail (more pronounced in R2, consistent with typical
Illumina paired-end behavior). See `NOTES.md` for full interpretation.

### 2. Trimming and adapter removal

\`\`\`bash
fastp \
  -i raw_data/SRR2584863_1.fastq \
  -I raw_data/SRR2584863_2.fastq \
  -o trimmed_data/SRR2584863_1.trimmed.fastq \
  -O trimmed_data/SRR2584863_2.trimmed.fastq \
  --html qc_reports/fastp_report.html \
  --json qc_reports/fastp_report.json
\`\`\`

fastp automatically detects and trims adapters (Nextera Transposase Sequence was found near read tails) and low-quality tails. Q30 improved from 89.4%→93.6% (R1) and 73.9%→84.7% (R2). ~16% of reads were discarded for low quality, still leaving ample coverage given the ~50x starting depth. See `NOTES.md` for full details.

### 3. Post-trimming quality check

\`\`\`bash
fastqc trimmed_data/SRR2584863_1.trimmed.fastq trimmed_data/SRR2584863_2.trimmed.fastq -o qc_reports/
\`\`\`

Confirms adapter content warning is resolved and the quality tail no longer drops into the red zone.

## Progress log

- [x] Installed Miniconda on WSL
- [x] Configured bioconda/conda-forge channels, channel_priority strict
- [x] Created `ngs-pipeline` environment with fastqc, fastp, bowtie2, samtools, multiqc
- [x] Downloaded test dataset
- [X] Initial QC (FastQC)
- [X] Trimming (fastp)
- [X] Post-trimming QC
- [ ] Reference genome download
- [ ] Alignment (Bowtie2)
- [ ] SAM → BAM conversion, sort and index (SAMtools)
- [ ] Mapping statistics (samtools flagstat)
- [ ] Final report (MultiQC)
