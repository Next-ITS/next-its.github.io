---
title: Running the pipeline
description: >-
    Input formats, barcodes, primers, workflow steps, choice of clustering method, and run configuration.
---

# Running the pipeline

A typical NextITS command looks like this:

```bash
nextflow run vmikk/NextITS -r main \
  -profile docker \
  -resume \
  --step           Step1 \
  --input          "pacbio_ccs.fastq.gz" \
  --barcodes       "sample_barcodes.fasta" \
  --primer_forward "TACACACCGCCCGTCG" \
  --primer_reverse "CCTSCSCTTANTDATATGC" \
  --chimera_db     "Eukaryome_1.9.3_241222_FullITS_100-800.udb" \
  --outdir         "Step1_Results/Run01"
```

Options with **one dash** (`-profile`, `-resume`, `-work-dir`) belong to Nextflow.
Options with **two dashes** (`--input`, `--step`, …) are pipeline [parameters](parameters.md).

## Pipeline steps

`--step` selects what to run:

| `--step` | Input | What it does |
| --- | --- | --- |
| `Step1` (default) | Raw reads of one sequencing run | QC, demultiplexing, primer trimming, ITS extraction, homopolymer correction, chimera detection, tag-jump removal |
| `Step2` | Step-1 results of one or more runs | Pooling, filtering, optional denoising, clustering, LULU curation |
| `seqstats` | Same as `Step1` | QC, demultiplexing and primer check only, with a summary report. Use it for a quick look at a new run. |

### Workflow overview

=== "Step-1"

    ```mermaid
    flowchart TD
      A[Raw reads<br/>FASTQ / BAM] --> B[Quality filtering<br/>VSEARCH, max EE, homopolymers]
      B --> C[Demultiplexing<br/>lima · cutadapt for Illumina]
      C --> D[Primer check & reorientation<br/>cutadapt]
      D --> E[Primer trimming & dereplication]
      E -.->|--long_chimera| F[Long-chimera removal<br/>Infernal]
      E --> G[ITS extraction<br/>ITSx / ITSx2]
      F -.-> G
      G --> H[Homopolymer correction]
      H --> I[Chimera detection<br/>reference + de novo, with rescue]
      I --> J[Tag-jump removal<br/>UNCROSS2]
      J --> K[(Sequence table<br/>Seqs.parquet)]
    ```

=== "Step-2"

    ```mermaid
    flowchart TD
      A[(Step-1 tables<br/>of all runs)] --> B[Pooling & filtering<br/>de novo chimeras, length]
      B --> C[Dereplication]
      C -.->|--chunking_n| D[Split into chunks<br/>MMseqs2]
      C --> E[Denoising / pre-clustering<br/>UNOISE · DADA2 · SWARM d1]
      D -.-> E
      E --> F[Clustering<br/>VSEARCH · SWARM · OptimOTU]
      F --> G[OTU table<br/>low-quality singleton filtering]
      G --> H[Post-clustering curation<br/>LULU]
      H --> I[(OTU table + sequences<br/>rRNA parts)]
    ```

Dashed arrows mark optional steps that are off by default.
Most steps can be switched off, e.g. `--hp false`, `--chimera_methods none`, `--tj false`, `--lulu false`.

### Where results are stored

Nextflow runs every task in a sub-folder of the work directory (`work/` by default, or the path given by `-work-dir`).
By default (`--storagemode rellink`), the files in `--outdir` are relative links to files in `work/`.
This saves disk space, but **deleting `work/` breaks the results**.

If you plan to delete the work directory, run with `--storagemode copy`, or copy the results with dereferenced links first:

```bash
cp -rL Step1_Results Step1_Results_copy
```

`-resume` reuses the results of tasks that have not changed, so add it every time.
After a crash or a parameter change, only the affected steps run again.

## Input data

=== "PacBio, multiplexed"

    One file with all reads of a sequencing run and a FASTA file with the sample barcodes:

    ```bash
    --input    "Run01.fastq.gz" \
    --barcodes "Run01_barcodes.fasta"
    ```

    `--input` can also be an unaligned BAM file (e.g., `Run01.hifi_reads.bam`).
    The PacBio index (`Run01.hifi_reads.bam.pbi`) must be in the same folder.

=== "Pre-demultiplexed"

    If your reads are already split by sample, point `--input` to a folder of per-sample files and skip `--barcodes`:

    ```bash
    --demultiplexed true \
    --input "/path/to/demultiplexed_reads/"
    ```

    Files must end in `.fastq.gz`, `.fastq`, `.fq.gz` or `.fq`.
    The sample name is the file name up to the **first dot**, so `Run01__S1.fastq.gz` becomes `Run01__S1`.
    Don't use dots elsewhere in file names.

=== "Illumina paired-end"

    !!! warning "Experimental"
        Illumina support is new and its parameters may change between releases.

    Multiplexed reads, with tags on the reads:

    ```bash
    --seqplatform Illumina \
    --input_R1 "Run01_R1.fastq.gz" \
    --input_R2 "Run01_R2.fastq.gz" \
    --barcodes "Run01_barcodes.fasta" \
    --illumina_barcodetype dual_symmetric
    ```

    Reads already split by sample:

    ```bash
    --seqplatform Illumina \
    --demultiplexed true \
    --input "/path/to/demultiplexed_reads/" \
    --illumina_pe_pattern "*_R{1,2}*.{fastq,fq}.gz"
    ```

    For per-sample files, the sample name is taken from the file name.
    Illumina suffixes such as `_S1_L001` are removed, and names with dots are rejected.

    Read pairs are reoriented, then merged with USEARCH (`--pe_merger usearch`, default) or VSEARCH.
    USEARCH is not included in the current container (`vmikk/nextits:1.2.4`), so use `--pe_merger vsearch` with it.

    **Quality presets.** Use a profile that matches your instrument's quality scores.
    These presets set the QC and merging thresholds:

    | Profile | Instruments | Quality scores |
    | --- | --- | --- |
    | `-profile miseq,docker` | MiSeq and other four-colour chemistry | continuous |
    | `-profile novaseq,docker` | NovaSeq, NextSeq and other two-colour chemistry | binned; poly-G tails trimmed |

    **Unmerged pairs.** By default, read pairs that don't overlap are discarded.
    `--illumina_keep_notmerged true` joins them with a run of `N`s instead.
    It requires `--its_region none` and `--hp false`.

    See the [Illumina parameters](parameters.md#illumina-paired-end-experimental) for tag search and merging settings.

### Barcodes

Sample barcodes (also called tags or indices) are given in a FASTA file. The sequence name is the sample name:

```
>Run01__Sample_01
ACAACACTCCGA
>Run01__Sample_02
ACAAGTGCTGCT
>Run01__PosControl
ACACAGTCCTGA
```

For dual barcodes that differ between the two ends, write both barcodes separated by `...`.
Give both in the **same orientation, 5′ → 3′**, as they were ordered:

```
>Run01__Sample_01
ACAACACTCCGA...TGCTAGCTAGCT
>Run01__Sample_02
ACAAGTGCTGCT...GCATGCATGCAT
```

Set the barcoding scheme with `--lima_barcodetype` (PacBio) or `--illumina_barcodetype` (Illumina):

| Scheme | Description |
| --- | --- |
| `dual_symmetric` (default) | Same barcode at both ends |
| `dual_asymmetric` | Different barcodes at the two ends (`FWD...REV` format) |
| `dual` | A mix of symmetric and asymmetric pairs (PacBio only) |
| `single` | Barcode at one end only |

For PacBio data, the barcode score threshold (`--lima_minscore`, default 93) assumes 12-bp barcodes.
For shorter barcodes, lower it; see the [lima FAQ](https://lima.how/faq/barcode-score.html).

**Sample names.** We recommend `RunID__SampleID`, with a **double underscore** between the run and sample parts.
With this convention, you can trace every sequence back to its sequencing run.
It is also what lets `--merge_replicates` pool a sample that was sequenced in several runs.

Before demultiplexing, NextITS checks the barcode file and cleans up sample names:

- Non-ASCII letters are transliterated (`ö` → `o`).
- Any character that is not a letter or digit (including `-`, `.` and spaces) is replaced with `_`.
- Repeated underscores are collapsed, except for the first `__`.
- Duplicated names get a numeric suffix (`_1`, `_2`).

Renamed samples are listed in `01_Demux/tag_names_renamed.tsv`.
To keep the names you expect, use only letters, digits and `_`.

!!! warning "Barcodes must be unique"
    Two samples with the same barcode (or barcode combination) stop the pipeline with an error.
    For asymmetric pairs, `A...B` and `B...A` count as the same combination.

### Primers

```bash
--primer_forward "TACACACCGCCCGTCG" \
--primer_reverse "CCTSCSCTTANTDATATGC"
```

- Write both primers **5′ → 3′**, as ordered (do not reverse-complement the reverse primer).
- IUPAC ambiguity codes (`N`, `S`, `D`, …) are allowed.
- Reads may come in either orientation. After the primer check, all reads are turned to the forward-primer orientation.
- Reads without both primers are discarded. Reads with extra primer copies inside (multi-primer artefacts) are removed.

The defaults are the universal eukaryote primers ITS9mun (`TACACACCGCCCGTCG`) and ITS4ngsUni (`CCTSCSCTTANTDATATGC`)
([Tedersoo & Lindahl 2016](https://doi.org/10.1111/1758-2229.12438)).
To exclude plant sequences, the ITS1catta forward primer (`ACCWGCGGARGGATCATTA`) is a good alternative
([Tedersoo & Anslan 2019](https://doi.org/10.1111/1758-2229.12776)).
It sits at the very end of the SSU, so ITSx often can't find the SSU boundary;
use `--its_region ITS1_5.8S_ITS2` with it.

!!! tip "Low ITSx yield?"
    Because reads are already oriented by the primers, ITSx (v1.x) only searches the forward strand by default
    (`--ITSx_complement F`). If many reads have no ITS detected
    (see `03_ITSx/<sample>_no_detections.txt`), try `--ITSx_complement T`.
    Alternatively, use `--itsx_tool ITSx2`, which is faster and covers all eukaryotes.

## Denoising and clustering

Step-2 can denoise sequences, cluster them, or do both. You choose with two parameters:

- `--preclustering` (default `none`): denoising or pre-clustering applied first. Options: `unoise`, `dada2`, `swarm_d1`, `homopolymer`.
- `--clustering` (default `vsearch`): OTU clustering. Options: `vsearch`, `swarm`, `optimotu`, `none`.

| Method | Options | When to use |
| --- | --- | --- |
| VSEARCH | `--clustering vsearch --otu_id 0.98` | Classic OTUs at a fixed similarity threshold. A robust default. |
| SWARM | `--clustering swarm --swarm_d 1` | Clusters built by single-linkage of close neighbours, with no global threshold |
| UNOISE | `--preclustering unoise` | Denoising into zero-radius OTUs (zOTUs); combine with `--clustering vsearch` or `none` |
| DADA2 | `--preclustering dada2` | Error-model-based ASVs; the PAPA2 engine (`--dada2_engine papa2`) is the default |
| SWARM d1 | `--preclustering swarm_d1` | Fast lossless pre-clustering to shrink large datasets before OptimOTU or VSEARCH |
| None | `--preclustering none --clustering none` | Dereplicated sequences only (LULU must be disabled with `--lulu false`) |

!!! note "UNOISE and homopolymers"
    UNOISE was designed for Illumina reads. On PacBio data, homopolymer-length differences look like real variants to it,
    so keep homopolymer correction on in Step-1 (`--hp true`, the default) when using UNOISE.

**Large datasets.** `--chunking_n 10` splits the sequences into 10 chunks of similar sequences with MMseqs2,
and clusters each chunk separately. This reduces memory use and run time.
Sequences more similar than `--chunking_id` (default 0.9) always end up in the same chunk.

**Post-clustering curation.** By default, NextITS merges OTUs that look like errors of a more abundant,
co-occurring OTU with the LULU algorithm ([Frøslev et al. 2017](https://doi.org/10.1038/s41467-017-01312-x)),
using the [MUMU](https://github.com/frederic-mahe/mumu) implementation.
The default similarity threshold (`--lulu_match 97`) is stricter than the original LULU default (84%), which suits full-length ITS.

## Configuring runs

### Parameter files

Instead of a long command line, you can keep parameters in a YAML (or JSON) file:

```yaml title="params_step2.yml"
step: "Step2"
data_path: "/path/to/Step1_Results"
outdir: "Step2_Results"
preclustering: "unoise"
clustering: "vsearch"
otu_id: 0.97
lulu: true
```

```bash
nextflow run vmikk/NextITS -r main -profile docker -resume -params-file params_step2.yml
```

Use the parameter's type in the file: numbers without quotes, `true`/`false` for switches, and quoted strings.
Command-line options override values from the file.
See the [Nextflow docs](https://www.nextflow.io/docs/latest/cli.html#pipeline-parameters) for details.

### Built-in help

```bash
nextflow run vmikk/NextITS --help              # main parameters
nextflow run vmikk/NextITS --help chimera_db   # one parameter
nextflow run vmikk/NextITS --helpFull          # all parameters
nextflow run vmikk/NextITS --version
```

Parameter values are checked before the run starts. A misspelled parameter name produces a warning.

### Process resources

Every step has default CPU and memory settings. To change them, write a small config file and pass it with `-c`:

```groovy title="custom.config"
process {
    withName: 'S1:ITS_EXTRACTION:itsx' {
        cpus   = 16
        memory = 16.GB
    }
    withName: 'S2:CLUSTERING:cluster_vsearch' {
        cpus   = 32
    }
}
```

```bash
nextflow run vmikk/NextITS -r main -profile docker -c custom.config ...
```

Process names are listed in `pipeline_info/execution_trace_*.txt`.
For cluster setups, see [Running on HPC](HPC.md).
