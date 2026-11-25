---
title: Output files
description: >-
    What NextITS writes to the results folder in Step-1, Step-2 and seqstats.
---

# Output files

All paths are relative to `--outdir`. `<S>` stands for a sample name.
Some folders appear only when the step that creates them is enabled; this is noted in the tables.

!!! warning "Results are links by default"
    With the default `--storagemode rellink`, files in `--outdir` are links into Nextflow's `work/` directory.
    See [Where results are stored](usage.md#where-results-are-stored) before deleting `work/`.

## Step-1

!!! tip "Start here"
    - `08_RunSummary/Step1_report.html`: interactive report with read counts at every step, per sample and per run
    - `07_SeqTable/Seqs.parquet`: the final sequence table, which is the input to Step-2

???+ abstract "Folder structure"

    ```
    Step1_Results/Run01/
    ├── 00_BAM2FASTQ/            # BAM input only
    ├── 01_Demux/
    ├── 02_PrimerCheck/
    ├── 03_ITSx/                 # or 03_PrimerTrim/ with --its_region none
    ├── 03_ITSx_PooledParts/
    ├── 03_LongChimera/          # --long_chimera true only
    ├── 04_Homopolymer/
    ├── 05_Chimera/
    ├── 06_TagJumpFiltration/
    ├── 07_SeqTable/
    ├── 08_RunSummary/
    ├── 09_DB/
    └── pipeline_info/
    ```

| Folder | Contents |
| --- | --- |
| `00_BAM2FASTQ/` | Reads converted from BAM to FASTQ (BAM input only) |
| `01_Demux/` | **PacBio:** per-sample reads (`LIMA/<S>.fq.gz`) and lima reports (`lima.lima.summary`, `.counts`, `.report.gz`). `tag_names_renamed.tsv` appears if sample names were changed.<br>**Illumina:** per-sample read pairs (`Demux/`), merged reads (`Merged/`), and tables with demultiplexing, reorientation and merging statistics |
| `02_PrimerCheck/` | Reads with both primers, reoriented (`<S>_PrimerChecked.fq.gz`), and multi-primer artefacts (`<S>_PrimerArtefacts.fq.gz`) |
| `03_ITSx/` | Dereplicated, primer-trimmed reads (`<S>_derep.fasta.gz`) and the extracted regions: `<S>.full.fasta.gz`, `.ITS1.`, `.5_8S.`, `.ITS2.`, `.SSU.`, `.LSU.`. Also ITSx summaries and positions, and `<S>_no_detections.txt`. Sequence IDs are SHA1 hashes of the sequences. |
| `03_PrimerTrim/` | Replaces `03_ITSx/` with `--its_region none`: primer-trimmed, dereplicated reads |
| `03_ITSx_PooledParts/` | `rRNA_parts.parquet`: the rRNA regions (SSU, ITS1, 5.8S, ITS2, LSU, full ITS) of every unique sequence in the run. Step-2 uses it to export the parts of OTU representatives. |
| `03_LongChimera/` | Removed long chimeras (`<S>_LongChimera.fa.gz`) and their rRNA hits (`.tsv.gz`) |
| `04_Homopolymer/` | Homopolymer-corrected sequences per sample and their cluster membership (`.uc.gz`) |
| `05_Chimera/` | Per sample: non-chimeric (`_NoChimera.fa.gz`), reference-based chimeras (`_Chimera.fa.gz`), rescued chimeras (`_RescuedChimera.fa.gz`), and *de novo* chimera scores (`_DeNovoChim.txt`) |
| `06_TagJumpFiltration/` | Sequence table after tag-jump removal (`Seq_tab_TagJumpFiltered.txt.gz`), UNCROSS scores (`TagJump_scores.qs`) and a diagnostic plot (`TagJump_plot.pdf`) |
| `07_SeqTable/` | **Final Step-1 table:** `Seqs.parquet`, the same data as a long tab-separated table (`Seqs.txt.gz`), and all sequences in FASTA (`Seqs.fa.gz`) |
| `08_RunSummary/` | `Step1_report.html`, `Run_summary.xlsx`, `per_sample.tsv`, `per_run.tsv`, read counts after each process (`PerProcessStats/`), and MultiQC-compatible tables (`MultiQC/`) |
| `09_DB/` | `SeqQualities.parquet`: sequences and quality scores of all reads. Step-2 uses it to export full-length sequences of OTU representatives. |
| `pipeline_info/` | See [below](#pipeline_info) |

**Sequence table columns.** Each row of `07_SeqTable/Seqs.parquet` is one unique sequence in one sample:

| Column | Description |
| --- | --- |
| `SampleID` | Sample name |
| `SeqID` | SHA1 hash of the sequence |
| `Abundance` | Number of reads |
| `SeqLen` | Sequence length |
| `PhredScore`, `MaxEE`, `MEEP` | Quality of the representative read: average Phred score, expected number of errors, and expected errors as a percentage of length |
| `DeNovo_Chimera`, `DeNovo_Chimera_Score` | *De novo* chimera flag and score. These sequences are removed in Step-2 (`--max_ChimeraScore`). |
| `Sequence` | The sequence |

## Step-2

!!! tip "Start here"
    - `06.Report/Step2_report.html`: summary of filtering and clustering
    - `05.LULU/OTU_table_LULU.txt.gz` and `05.LULU/OTUs_LULU.fa.gz`: the final OTU table and OTU sequences, after LULU curation
    - `04.PooledResults/`: the same before LULU (and the final results when `--lulu false`)

???+ abstract "Folder structure"

    ```
    Step2_Results/
    ├── 01.Dereplicated/
    ├── 02.UNOISE/                    # --preclustering unoise
    ├── 02.DADA2/                     # --preclustering dada2
    ├── 02.Preclustered_SWARM_d1/     # --preclustering swarm_d1
    ├── 02.Homopolymer/               # --preclustering homopolymer
    ├── 03.Clustered_VSEARCH/         # --clustering vsearch
    ├── 03.Clustered_SWARM/           # --clustering swarm
    ├── 03.Clustered_OptimOTU/        # --clustering optimotu
    ├── 04.PooledResults/
    │   └── rRNA_parts/
    ├── 05.LULU/                      # --lulu true
    │   └── rRNA_parts/
    ├── 06.Report/
    └── pipeline_info/
    ```

| Folder | Contents |
| --- | --- |
| `01.Dereplicated/` | Unique sequences pooled across all runs (`Dereplicated.fa.gz`) and their membership (`Dereplicated.uc.gz`) |
| `02.*/` | Output of the denoising or pre-clustering step: representative sequences (`.fa.gz`) and membership (`.uc.gz`). DADA2 also writes `DADA2_denoising_summary.txt`. |
| `03.Clustered_*/` | OTU representatives (`Clustered.fa.gz` or `SWARM_representatives.fa.gz`) and membership (`.uc.gz`). OptimOTU also writes `OptimOTU_otu_taxonomy.tsv.gz`. |
| `04.PooledResults/` | OTU table in long (`OTU_table_long.txt.gz`) and wide (`OTU_table_wide.txt.gz`) format, the same tables as R objects (`.RData`), OTU sequences (`OTUs.fa.gz`), and the membership of every sequence (`UC_Pooled.parquet`) |
| `04.PooledResults/rRNA_parts/` | OTU representatives split into rRNA regions: `SSU.fa.gz`, `ITS1.fa.gz`, `5_8S.fa.gz`, `ITS2.fa.gz`, `LSU.fa.gz`, `ITS_Full.fa.gz`, plus the full sequence (`OTUs_FullSeq.fa.gz`), a table (`rRNA_parts.parquet`) and a summary. Created when `--rrna_parts true`. |
| `05.LULU/` | OTU table after LULU, in wide format (`OTU_table_LULU.txt.gz`), OTU sequences (`OTUs_LULU.fa.gz`), which OTUs were merged (`LULU_merging_statistics.txt.gz`), and the match list used (`LULU_match_list.txt.gz`). `rRNA_parts/` holds the region files for the curated OTUs. |
| `06.Report/` | `Step2_report.html`, `Step2_summary.tsv`, and MultiQC-compatible tables |
| `pipeline_info/` | See [below](#pipeline_info) |

**OTU tables.** The long table has three columns: `OTU`, `SampleID`, `Abundance`.
The wide table has one row per OTU and one column per sample.
The `.RData` files are gzip-compressed RDS files, despite the extension; the wide one holds a sparse matrix.

=== "R"

    ```r
    otu_long <- readRDS("04.PooledResults/OTU_table_long.RData")   # data.table
    otu_wide <- readRDS("04.PooledResults/OTU_table_wide.RData")   # sparse matrix

    # Step-1 sequence table
    library(arrow)
    seqs <- read_parquet("Step1_Results/Run01/07_SeqTable/Seqs.parquet")
    ```

=== "Python"

    ```python
    import pandas as pd

    otu = pd.read_csv("05.LULU/OTU_table_LULU.txt.gz", sep="\t")
    seqs = pd.read_parquet("Step1_Results/Run01/07_SeqTable/Seqs.parquet")
    ```

=== "DuckDB"

    ```sql
    SELECT SampleID, COUNT(*) AS n_seqs, SUM(Abundance) AS n_reads
    FROM 'Step1_Results/*/07_SeqTable/Seqs.parquet'
    GROUP BY SampleID;
    ```

## seqstats

`--step seqstats` writes the same `00_BAM2FASTQ/`, `01_Demux/` and `02_PrimerCheck/` folders as Step-1, plus:

| Folder | Contents |
| --- | --- |
| `03_Stats/` | `Seqstats_report.html`, `Run_summary.xlsx`, `per_sample.tsv`, `per_run.tsv` and read counts per process (`PerProcessStats/`) |

## pipeline_info

Every step writes a `pipeline_info/` folder (set a different location with `--tracedir`).
Unlike the other outputs, these are regular files, not links.

| File | Contents |
| --- | --- |
| `README_Step1_Methods.txt` / `README_Step2_Methods.txt` | Methods description of the run, with the tools used and their references. A starting point for the methods section of a paper. |
| `software_versions.yml` | Versions of all tools used |
| `pipeline_params.tsv` | All parameter values. Step-2 reads this file from each Step-1 run. |
| `execution_command.txt` | The exact command used |
| `execution_report_<date>.html` | Nextflow report: run time, CPU and memory use per process |
| `execution_timeline_<date>.html` | Timeline of all tasks |
| `execution_trace_<date>.txt` | One line per task, with resources used |
| `pipeline_dag_<date>.svg` | Graph of the workflow |
