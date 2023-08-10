---
title: Parameters
description: >-
    Pipeline parameters and configuration.
---

# Parameters

The NextITS pipeline is made up of multiple connected processes. 
Each of these processes is amenable to configuration, 
empowering users to tailor not just the process parameters 
but also the resources allocated for each one (e.g., the number of CPUs). 
When setting the pipeline parameter values, users have two choices: 
they can specify them directly from the command line using double dash arguments (for example, `--its_region "full"`) 
or opt for a simple parameters file. Utilizing a parameters file is especially advantageous 
when dealing with numerous settings, as it offers a cleaner command line interface. 
In addition to this, individual processes within the pipeline can be fine-tuned using a dedicated config file.  
## Step-1

### Required parameters

| Parameter    | Description                                                                                                            |
| ------------ | ---------------------------------------------------------------------------------------------------------------------- |
| `--input`    | Path to the input file containing single-end sequences (in FASTQ format) or the directory with pre-demultiplexed files |
| `--barcodes` | Path to the file with barcodes (in FASTA format) used for demultiplexing the input data                                |
| `--outdir`   | Path to the directory where the analysis results will be saved.                                                        |


### Demultiplexing

| Parameter            | Description                                                                                                       | Default Value |
| -------------------- | ----------------------------------------------------------------------------------------------------------------- | ------------- |
| `--demultiplexed`    | Whether input is multiplexed (`true`, single FASTQ file) or pre-demultiplexed (`false`, multiple FASTQ files) ^1^ | `true`        |
| `--lima_minscore`    | Barcode score for demultiplexing                                                                                  | 93            |
| `--lima_dualbarcode` | Dual-barcoding scheme with identical barcodes at both ends ^2^                                                    | `true`        |
| `--lima_W`           | Window size for barcode lookup                                                                                    | 70            |
| `--lima_minlen`      | Minimum sequence length after clipping barcodes                                                                   | 40            |


### Quality filtering

| Parameter                | Description                                                                          | Default Value |
| ------------------------ | ------------------------------------------------------------------------------------ | ------------- |
| `--qc_maxee`             | Maximum number of expected errors                                                    | false         |
| `--qc_maxeerate`         | Maximum number of expected errors per base                                           | 0.01          |
| `--qc_maxhomopolymerlen` | Threshold for a homopolymer region length in a sequence                              | 25            |
| `--qc_maxn`              | Discard sequences with more than the specified number of ambiguous nucleotides (N's) | 4             |


### Removal of multiprimer-artifacts and reorienting of reads

| Parameter             | Description                                | Default Value       |
| --------------------- | ------------------------------------------ | ------------------- |
| `--primer_forward`    | Sequence of the forward primer ^1^         | GTACACACCGCCCGTCG   |
| `--primer_reverse`    | Sequence of the reverse primer ^1^         | CCTSCSCTTANTDATATGC |
| `--primer_mismatches` | Allowed number of mismatches for primers   | 2                   |
| `--primer_foverlap`   | Minimum overlap for the forward primer ^2^ | F primer length - 2 |
| `--primer_roverlap`   | Minimum overlap for the reverse primer ^2^ | R primer length - 2 |

### ITS extraction

| Parameter        | Description                                                                     | Default Value |
| ---------------- | ------------------------------------------------------------------------------- | ------------- |
| `--its_region`   | ITS part selector ^2^                                                           | "full"        |
| `--ITSx_tax`     | Taxonomy profile for ITSx ^1^                                                   | "all"         |
| `--ITSx_evalue`  | E-value cutoff threshold for ITSx                                               | 1e-1          |
| `--ITSx_partial` | Keep partial ITS sequences (specify a minimum length cutoff), default = off (0) | 0             |