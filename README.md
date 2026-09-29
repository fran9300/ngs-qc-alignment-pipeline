# NGS QC + Alignment Mini-Pipeline

Practice project to learn the basic NGS workflow: from raw reads (FASTQ) to an indexed alignment (BAM), covering quality control and trimming along the way.

## Goal

Understand and execute the full flow end to end: QC of raw reads → trimming → alignment → mapping statistics.

## Environment

Set up with Conda (Miniconda) on WSL (Ubuntu on Windows).

Tools and environment created with:

```bash
conda create -n ngs-pipeline fastqc fastp bowtie2 samtools multiqc -y
```

Activate with:

```bash
conda activate ngs-pipeline
```

### SRA Toolkit (for downloading data from SRA)

The `sra-tools` package installed via Bioconda was outdated (old TLS certificates, failing against NCBI). The official NCBI binary is used instead:

```bash
cd ~
wget https://ftp-trace.ncbi.nlm.nih.gov/sra/sdk/current/sratoolkit.current-ubuntu64.tar.gz
tar -xzf sratoolkit.current-ubuntu64.tar.gz
echo 'export PATH=$PATH:$HOME/sratoolkit.3.4.1-ubuntu64/bin' >> ~/.bashrc
source ~/.bashrc
```

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

```bash
fastqc raw_data/SRR2584863_1.fastq raw_data/SRR2584863_2.fastq -o qc_reports/
```

Reports available in `qc_reports/`. Key findings: overall good quality data,
with expected quality drop and Nextera adapter content toward the read tail (more pronounced in R2, consistent with typical
Illumina paired-end behavior). See `NOTES.md` for full interpretation.

### 2. Trimming and adapter removal

```bash
fastp \
  -i raw_data/SRR2584863_1.fastq \
  -I raw_data/SRR2584863_2.fastq \
  -o trimmed_data/SRR2584863_1.trimmed.fastq \
  -O trimmed_data/SRR2584863_2.trimmed.fastq \
  --html qc_reports/fastp_report.html \
  --json qc_reports/fastp_report.json
```

fastp automatically detects and trims adapters (Nextera Transposase Sequence was found near read tails) and low-quality tails. Q30 improved from 89.4%→93.6% (R1) and 73.9%→84.7% (R2). ~16% of reads were discarded for low quality, still leaving ample coverage given the ~50x starting depth. See `NOTES.md` for full details.

### 3. Post-trimming quality check

```bash
fastqc trimmed_data/SRR2584863_1.trimmed.fastq trimmed_data/SRR2584863_2.trimmed.fastq -o qc_reports/
```

Confirms adapter content warning is resolved and the quality tail no longer drops into the red zone.

### 4. Reference genome

Downloaded the E. coli B REL606 reference genome (same strain as the sequenced sample) using NCBI datasets:

```bash
conda install -n ngs-pipeline -c conda-forge ncbi-datasets-cli -y
datasets download genome accession GCA_000017985.1 --include genome
```

The FASTA (`REL606.fasta`, accession CP000819, ~4.6 Mbp) is not tracked in git; re-download using the command above to reproduce.

### 5. Alignment

Index the reference, then align trimmed reads:

```bash
bowtie2-build reference/REL606.fasta reference/REL606_index

bowtie2 -x reference/REL606_index \
  -1 trimmed_data/SRR2584863_1.trimmed.fastq \
  -2 trimmed_data/SRR2584863_2.trimmed.fastq \
  -S alignments/SRR2584863.sam \
  --threads 4 \
  2> alignments/bowtie2_summary.txt
```

**Result: 99.46% overall alignment rate.** Only ~38.7% of pairs aligned "concordantly" — this is expected, not a data quality issue: fragment sizes in this library are broadly distributed (a plateau roughly ~50-135bp with a long tail, per fastp's insert size plot), with many fragments shorter than the 300bp needed to avoid mate overlap. This likely explains the low concordant-pair rate, though it was not directly verified against Bowtie2's `-I`/`-X` limits. See `NOTES.md` for the full explanation.

### 6. SAM to sorted, indexed BAM

```bash
samtools view -b alignments/SRR2584863.sam > alignments/SRR2584863.bam
samtools sort alignments/SRR2584863.bam -o alignments/SRR2584863.sorted.bam
samtools index alignments/SRR2584863.sorted.bam
```

The sorted BAM (217 MB) is ~5x smaller than the original SAM (1.1 GB). Intermediate SAM/BAM files can be deleted once the sorted BAM is verified.

### 7. Mapping statistics

```bash
samtools flagstat alignments/SRR2584863.sorted.bam > alignments/flagstat.txt
```

Results: 2,602,652 reads total, 99.46% mapped, 38.74% properly paired. These numbers match Bowtie2's summary exactly, confirming BAM integrity. Almost all reads (2,578,134) aligned together with their mate, so the low "properly paired" rate reflects the short library fragment size (~100bp), not poor data quality.

### 8. Aggregate report (MultiQC)

```bash
multiqc qc_reports/ alignments/ -o qc_reports/multiqc --force --fullnames
```

`--fullnames` is required: by default MultiQC strips common processing suffixes like `_trimmed` when deriving sample names, which caused raw and trimmed FastQC reports to collapse into the same sample and only 2 of 4 to show up. See `NOTES.md` for details.

Open the report at `qc_reports/multiqc/multiqc_report.html`.

## Progress log

- [x] Installed Miniconda on WSL
- [x] Configured bioconda/conda-forge channels, channel_priority strict
- [x] Created `ngs-pipeline` environment with fastqc, fastp, bowtie2, samtools, multiqc
- [x] Downloaded test dataset
- [x] Initial QC (FastQC)
- [x] Trimming (fastp)
- [x] Post-trimming QC
- [x] Reference genome download
- [x] Alignment (Bowtie2)
- [X] SAM → BAM conversion, sort and index (SAMtools)
- [X] Mapping statistics (samtools flagstat)
- [x] Final report (MultiQC)
