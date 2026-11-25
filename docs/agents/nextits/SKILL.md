---
name: nextits
description: >-
  Set up, run, troubleshoot and interpret the NextITS Nextflow pipeline for metabarcoding of fungi
  and other eukaryotes (full-length ITS from PacBio HiFi, and Illumina paired-end ITS).
  Use this skill whenever the user mentions NextITS, or wants to process ITS or rRNA amplicon reads
  into an OTU/ASV table with Nextflow: demultiplexing with sample barcodes, primer trimming, ITSx,
  chimera or tag-jump removal, VSEARCH/SWARM/UNOISE/DADA2/OptimOTU clustering, LULU curation,
  running on Slurm/HPC, reading Step1/Step2 output folders, debugging a failed NextITS run,
  or writing a methods section for such an analysis. Use it even if NextITS is not named but the task
  is PacBio ITS metabarcoding.
---

# NextITS

NextITS is a Nextflow pipeline (`vmikk/NextITS`) that turns raw amplicon reads into a curated OTU table.
All tools run inside a container; the user only needs Nextflow and Docker, Podman, Singularity or Apptainer.

It runs in two steps:

- **Step1** runs once per sequencing run: QC, demultiplexing, primer check, ITS extraction,
  homopolymer correction, chimera detection, tag-jump removal. Output: a sequence table per run.
- **Step2** pools the Step1 results of one or more runs: filtering, optional denoising, clustering,
  LULU curation. Output: OTU table and sequences.
- `--step seqstats` takes the same inputs as Step1 and runs only QC, demultiplexing and the primer check.
  It is a quick check of a new run.

Tag jumps happen within a sequencing run, so Step1 must process each run separately. Do not merge
FASTQ files of different runs before Step1.

## Where the facts are

This skill holds the workflow and the common mistakes. It does not list parameters or defaults,
because they change between releases. Look them up instead of relying on memory:

- Documentation as Markdown: <https://next-its.github.io/llms.txt> (index) and
  <https://next-its.github.io/llms-full.txt> (all pages in one file). Individual pages:
  [installation](https://next-its.github.io/installation/index.md),
  [quick start](https://next-its.github.io/quickstart/index.md),
  [running the pipeline](https://next-its.github.io/usage/index.md),
  [parameters](https://next-its.github.io/parameters/index.md),
  [output files](https://next-its.github.io/output/index.md),
  [HPC](https://next-its.github.io/HPC/index.md).
- Built-in help: `nextflow run vmikk/NextITS --help <parameter>`, `--helpFull` for all parameters.
- Changes between versions: <https://github.com/vmikk/NextITS/blob/main/CHANGELOG.md>.

Check every parameter name you write in a command or params file against `--help` or the parameters page.
A misspelled parameter only produces a warning, and the run then uses the default.
If this skill disagrees with the docs or `--help`, the docs and `--help` are correct.

## Setting up a run

1. **Ask the user for the facts only they know.** Do not guess them; a wrong guess runs for hours and
   produces a plausible but wrong table.
   - Platform (PacBio or Illumina) and whether reads are multiplexed or already split by sample.
   - Barcode file and barcoding scheme (symmetric, asymmetric, mixed, single).
   - Primer sequences and the amplified region (full ITS, ITS1-5.8S-ITS2 without SSU, longer rRNA amplicon).
   - Where it runs: workstation or HPC cluster, and which container engine is available.
2. **Check prerequisites:** `nextflow -version` (the version required is on the installation page)
   and the container engine. On a new installation, run `-profile test,<engine>` first.
3. **Check a new run with `--step seqstats`** if the user is unsure about barcodes or primers.
   Its report shows how many reads were demultiplexed and passed the primer check.
4. **Run Step1 once per sequencing run**, each with its own `--outdir` and its own launch directory
   (so `-resume` and the `work/` cache are kept per run). The quick start page has a script for many runs.
5. **Run Step2** with `--data_path` set to the folder that contains all Step1 output folders.
6. For reproducibility, put parameters in a YAML file (`-params-file params.yml`; numbers unquoted,
   `true`/`false` for switches) and pin a release with `-r <tag>` instead of `-r main`.

Skeletons (replace the angle brackets; use absolute paths):

```bash
nextflow run vmikk/NextITS -r <tag> -profile <docker|podman|singularity|apptainer> -resume \
  --step Step1 \
  --input <run.fastq.gz or run.hifi_reads.bam> \
  --barcodes <barcodes.fasta> \
  --primer_forward <SEQ> --primer_reverse <SEQ> \
  --chimera_db </abs/path/to/db.udb> \
  --outdir <Step1_Results/RunID>

nextflow run vmikk/NextITS -r <tag> -profile <engine> -resume \
  --step Step2 \
  --data_path <Step1_Results> \
  --clustering vsearch --otu_id 0.98 \
  --outdir <Step2_Results>
```

## Common mistakes

- **One dash vs two.** `-profile`, `-resume`, `-c`, `-work-dir`, `-params-file` are Nextflow options;
  `--step`, `--input` and the rest are pipeline parameters.
- **Deleting `work/` breaks the results.** By default (`--storagemode rellink`) files in `--outdir`
  are links into `work/`. Before deleting `work/`, copy results with `cp -rL`, or run with
  `--storagemode copy`. Warn the user before any cleanup of `work/`.
- **Barcode file.** FASTA; the sequence name is the sample name. Asymmetric pairs are written
  `FWD...REV`, both 5'→3' as ordered. Barcodes must be unique (`A...B` and `B...A` count as the same).
  Name samples `RunID__SampleID` (double underscore), with letters, digits and `_` only;
  other characters are replaced and the renames are listed in `01_Demux/tag_names_renamed.tsv`.
- **Pre-demultiplexed input** (`--demultiplexed true`): the sample name is the file name up to the first dot.
- **Primers** are written 5'→3' as ordered. Do not reverse-complement the reverse primer.
  With a forward primer at the very end of SSU (e.g. ITS1catta), use `--its_region ITS1_5.8S_ITS2`.
- **Chimera database.** Reference-based chimera detection needs the UDB file from the installation page.
  Pass an absolute path to `--chimera_db`; a bare file name is looked up in the launch directory.
  Without the database, use `--chimera_methods denovo`.
- **Few reads with ITS detected** (`03_ITSx/<sample>_no_detections.txt`): try `--ITSx_complement T`
  or `--itsx_tool ITSx2`.
- **Step2 combinations.** `--preclustering none --clustering none` needs `--lulu false`.
  UNOISE on PacBio data needs homopolymer correction in Step1 (`--hp true`, the default).
  OptimOTU needs full ITS from Step1, a reference set under `--db_dir`, and a CPU with AVX2.
- **Illumina support is experimental**; read the Illumina section of the usage page before use.
- **HPC.** Use `-profile singularity` or `apptainer` and set `NXF_SINGULARITY_CACHEDIR` or
  `NXF_APPTAINER_CACHEDIR` to a shared folder. Choose between one batch job and Nextflow's Slurm
  executor as described on the HPC page. Do not submit large jobs without the user's approval.

## Choosing a clustering method

VSEARCH at a fixed threshold (`--clustering vsearch --otu_id 0.98`) followed by LULU is a robust default.
For other options (SWARM, UNOISE, DADA2 ASVs, OptimOTU, chunking of large datasets with `--chunking_n`),
read the "Denoising and clustering" section of the usage page and explain the trade-off to the user
before choosing.

## Results and failures

- Step1: start with `08_RunSummary/Step1_report.html` (read counts per step and sample);
  the final table is `07_SeqTable/Seqs.parquet`.
- Step2: start with `06.Report/Step2_report.html`; final OTU table and sequences are in `05.LULU/`
  (or `04.PooledResults/` with `--lulu false`). The output page describes every folder and table.
- `pipeline_info/README_Step1_Methods.txt` and `README_Step2_Methods.txt` describe the run with the
  tools used and their references. Use them as the basis for a methods section; do not write
  methods from memory.
- When a run fails: read the error in the terminal or in `.nextflow.log` (launch directory).
  It names the failed process and its work directory; there, `.command.err`, `.command.log` and
  `.command.sh` show what ran. Fix the cause and rerun the same command with `-resume`.
