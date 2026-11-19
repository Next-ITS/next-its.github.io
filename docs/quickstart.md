---
title: Quick start
description: >-
    Run NextITS on your own PacBio data in two commands.
---

# Quick start

This page assumes you have [installed](installation.md) Nextflow and a container engine,
and downloaded the [chimera database](installation.md#databases).
The examples use Docker; swap `-profile docker` for `singularity`, `apptainer` or `podman` as needed.

NextITS runs in two steps:

1. **Step-1** processes each sequencing run separately: QC, demultiplexing, ITS extraction, chimera detection and tag-jump removal.
2. **Step-2** pools the Step-1 results of all runs and clusters the sequences into OTUs.

## Step-1: process a sequencing run

For a multiplexed PacBio run you need a FASTQ (or BAM) file, a FASTA file with sample barcodes,
and the primer sequences:

```bash
nextflow run vmikk/NextITS -r main \
  -profile docker \
  -resume \
  --step           Step1 \
  --input          "$(pwd)/Input/Run01/Run01.fastq.gz" \
  --barcodes       "$(pwd)/Input/Run01/Run01_barcodes.fasta" \
  --primer_forward "TACACACCGCCCGTCG" \
  --primer_reverse "CCTSCSCTTANTDATATGC" \
  --chimera_db     "/path/to/Eukaryome_1.9.3_241222_FullITS_100-800.udb" \
  --outdir         "Step1_Results/Run01"
```

Check your barcode file format and primer orientation in [Running the pipeline](usage.md#input-data) before the first run.
To process several runs, repeat the command for each one with its own `--input`, `--barcodes` and `--outdir`.
See [Processing many runs](#processing-many-runs) below for a script that does this.

!!! tip "Check the data first"
    `--step seqstats` takes the same input options. It runs only QC, demultiplexing and the primer check,
    and writes a short HTML report. It gives you a quick look at a new run before the full analysis.

## Step-2: pool runs and cluster

Point `--data_path` to the folder with all Step-1 results:

```bash
nextflow run vmikk/NextITS -r main \
  -profile docker \
  -resume \
  --step       Step2 \
  --data_path  "$(pwd)/Step1_Results" \
  --clustering vsearch \
  --otu_id     0.98 \
  --outdir     "Step2_Results"
```

This clusters sequences at 98% similarity with VSEARCH, then curates the OTU table with LULU.
The final tables are in `Step2_Results/05.LULU/`; see [Output files](output.md) for the full output.

Other clustering strategies only change the last few options:

=== "SWARM"

    Single-linkage clustering with a local threshold, instead of a global one:

    ```bash
    --clustering swarm --swarm_d 1
    ```

=== "UNOISE + VSEARCH"

    Denoise first, then cluster the denoised sequences:

    ```bash
    --preclustering unoise --clustering vsearch --otu_id 0.98
    ```

=== "DADA2 (ASVs only)"

    Exact sequence variants without OTU clustering:

    ```bash
    --preclustering dada2 --clustering none
    ```

See [Denoising and clustering](usage.md#denoising-and-clustering) for how to choose.

## Processing many runs

If your data are organised with one sub-directory per sequencing run,
each holding one FASTQ file and one barcode file:

```
Input/
├── Run01/
│   ├── Run01.fastq.gz
│   └── Run01_barcodes.fasta
├── Run02/
│   ├── Run02.fastq.gz
│   └── Run02_barcodes.fasta
└── Run03/
    ├── ...
```

then a small script can run Step-1 for each run.
Each run is launched from its own directory (`Step1_WorkDirs/<RunID>`), which holds that run's Nextflow cache and work files.
This keeps `-resume` working per run and lets several runs execute in parallel.

```bash title="run_step1.sh"
#!/bin/bash
# Usage: ./run_step1.sh <RunID>
set -euo pipefail

RUN="$1"
BASEDIR="$(pwd)"

mkdir -p "$BASEDIR"/Step1_WorkDirs/"$RUN"
cd "$BASEDIR"/Step1_WorkDirs/"$RUN"

nextflow run vmikk/NextITS -r main \
  -profile docker \
  -resume \
  --step           Step1 \
  --input          "$BASEDIR"/Input/"$RUN"/*.fastq.gz \
  --barcodes       "$BASEDIR"/Input/"$RUN"/*.fasta \
  --primer_forward "TACACACCGCCCGTCG" \
  --primer_reverse "CCTSCSCTTANTDATATGC" \
  --chimera_db     "/path/to/Eukaryome_1.9.3_241222_FullITS_100-800.udb" \
  --outdir         "$BASEDIR"/Step1_Results/"$RUN" \
  | tee "$BASEDIR"/Nextflow_"$RUN".log
```

Run it for each sub-directory of `Input/`:

```bash
chmod +x run_step1.sh

for run in Input/*/; do
  ./run_step1.sh "$(basename "$run")"
done
```

With [GNU parallel](https://www.gnu.org/software/parallel/) you can run several at once.
Each Nextflow run already uses all available CPUs, so keep `-j` small:

```bash
ls Input/ | parallel -j 2 ./run_step1.sh {}
```

Once a run has finished and you have checked its results, you can delete its `Step1_WorkDirs/<RunID>` folder.
Before you do, read the note on [storage mode](usage.md#where-results-are-stored).

## Next steps

- [Running the pipeline](usage.md): input formats, barcodes, primers and the choice of clustering method.
- [Parameters](parameters.md): every option with its default.
- [Running on HPC](HPC.md): Slurm and other schedulers.
