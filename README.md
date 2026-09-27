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

## Progress log

- [x] Installed Miniconda on WSL
- [x] Configured bioconda/conda-forge channels, channel_priority strict
- [x] Created `ngs-pipeline` environment with fastqc, fastp, bowtie2, samtools, multiqc
- [x] Downloaded test dataset
- [ ] Initial QC (FastQC)
- [ ] Trimming (fastp)
- [ ] Post-trimming QC
- [ ] Reference genome download
- [ ] Alignment (Bowtie2)
- [ ] SAM → BAM conversion, sort and index (SAMtools)
- [ ] Mapping statistics (samtools flagstat)
- [ ] Final report (MultiQC)
