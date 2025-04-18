---
title: Quick start
description: >-
    Quick and easy NextITS instruction for impatient people
---

# Quick start  

Get started quickly with NextITS by following these simple instructions.  

The pipeline is divided into two main steps:  
1. The first step is executed separately for each sequencing run because it involves the removal of tag-jumps. This step primarily focuses on sequence quality control, ITS extraction, and chimera removal.  
2. The second step combines data from various runs, clusters sequences using either vsearch or swarm, and then performs post-clustering curation.  

##  Pull the latest pipeline version

``` bash
nextflow pull vmikk/NextITS
```


## Step 1

As a first step, each sequencing run must be processed individually.  

!!! info "DRY principle"
    For clarity, we're providing separate commands below. 
    Note, however, that this approach doesn't adhere to the DRY (Don't Repeat Yourself) principle and may lead to errors. 
    To minimize potential errors, you can encapsulate the command within a script or function (for details, see below).


Sequencing run #1:  
``` bash
nextflow run vmikk/NextITS -r main \
  -profile singularity \
  -resume \
  --step "Step1" \
  --input          "$(pwd)/Run01/Run01.fastq.gz" \
  --barcodes       "$(pwd)/Run01/Run01_barcodes.fasta" \
  --primer_forward "GTACACACCGCCCGTCG" \
  --primer_reverse "CCTSCSCTTANTDATATGC" \
  --outdir         "Step1_Results/Run01"
```

Sequencing run #2:  
``` bash
nextflow run vmikk/NextITS -r main \
  -profile singularity \
  -resume \
  --step "Step1" \
  --input          "$(pwd)/Run02/Run02.fastq.gz" \
  --barcodes       "$(pwd)/Run02/Run02_barcodes.fasta" \
  --primer_forward "GTACACACCGCCCGTCG" \
  --primer_reverse "CCTSCSCTTANTDATATGC" \
  --outdir         "Step1_Results/Run02"
```

Sequencing run #3:  
``` bash
nextflow run vmikk/NextITS -r main \
  -profile singularity \
  -resume \
  --step "Step1" \
  --input          "$(pwd)/Run03/Run03.fastq.gz" \
  --barcodes       "$(pwd)/Run03/Run03_barcodes.fasta" \
  --primer_forward "GTACACACCGCCCGTCG" \
  --primer_reverse "CCTSCSCTTANTDATATGC" \
  --outdir         "Step1_Results/Run03"
```

!!! info "Container engines"
    In the provided examples, we've assumed that you're utilizing **Singularity** as your container engine.  
    However, if you prefer to use **Docker**, it's a straightforward switch. 
    Simply modify the setting from `-profile singularity` to `-profile docker` in the command.  

## Step 2

Step-2 combines the results from Step-1.  
Then, we can use a greedy algorithm to cluster the sequences, setting a similarity threshold of 98%.  

``` bash
nextflow run vmikk/NextITS -r main \
  -resume \
  -profile     singularity \
  --step       "Step2" \
  --data_path  "$(pwd)/Step1_Results/" \
  --outdir     "Step2_Results" \
  --clustering_method "vsearch" \
  --otu_id 0.98
```

!!! info "The other options"
    Instead of using greedy clustering, you can opt for SWARM clustering, 
    which employs a dynamic sequence similarity threshold 
    that aligns with the natural limits of OTUs.  
    Additionally, before clustering or as an alternative to it, 
    you can denoise the data using the UNOISE algorithm.  
    For more details, see the `Usage instructions`.


## Wrapping a command into a script

To follow the [DRY principle](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself), 
it is possible to wrap Step-1 commands into a simple script.

``` bash
## Create the script
cat > run_Step1.sh <<'EOT'
#!/bin/bash

  ## Positional arguments to the script:
  # $1 = name of the directory containing FASTQ files

  echo -e "\n"
  echo -e "Input data: " $1
  echo -e "Output: " Step1_Results/"$1"
  echo -e "Temporary workdirs: " Step1_WorkDirs/"$1"

  BASEDIR=$(pwd)

  ## Create output directories
  mkdir -p Step1_Results/"$1"
  mkdir -p Step1_WorkDirs/"$1"
  cd Step1_WorkDirs/"$1"

  ## Run Step-1 of the pipeline
  nextflow run "$NEXTITS_PATH"/main.nf \
    -resume \
    -profile     docker \
    --step       "Step1" \
    --input      "$BASEDIR"/Input/"$1"/*.fastq.gz \
    --barcodes   "$BASEDIR"/Input/"$1"/*.fasta \
    --outdir     "$BASEDIR"/Step1_Results/"$1" \
    -work-dir    "$BASEDIR"/Step1_WorkDirs/"$1" \
  | tee Nextflow_"$1".log

EOT

## Make the script executable
chmod +x run_Step1.sh
```

The script assumes that the data is structured such that 
the `Input` directory houses multiple sub-directories corresponding to different sequencing runs 
(e.g., `Run01`, `Run02`, `Run03`), and within each of these sub-directories, 
there two files: a `FASTQ` file containing high-throughput data and a `FASTA` file with barcodes.  
For example:  

```
Input/
├── Run01/
│   ├── Run01.fastq.gz
│   └── Run01_barcodes.fasta
├── Run02/
│   ├── Run02.fastq.gz
│   └── Run02_barcodes.fasta
└── Run03/
    ├── Run03.fastq.gz
    └── Run03_barcodes.fasta
```

To initiate the analysis for individual sequencing runs, execute the following commands:  
``` bash
./run_Step1.sh "Run01"
./run_Step1.sh "Run02"
./run_Step1.sh "Run03"
```

If you have a large number of sequencing runs, 
you can automate the process by combining the `find` and 
[GNU `parallel`](https://www.gnu.org/software/parallel/) 
utilities:  
``` bash
find $(pwd)/Input/ -type d -not -path $(pwd)/Input/ \
  | sort \
  | parallel -j1 "./run_Step1.sh {/}"
```
This command will locate all sub-directories within the `Input` directory and run the analysis script for each one.

## Configuring the pipeline

While we have endeavored to choose the most sensible default options for the NextITS pipeline, 
it's worth noting that there are numerous parameters available for configuration and fine-tuning, 
allowing users to adapt the pipeline to best fit their data.  

Detailed descriptions of these parameters can be found in the [`Parameters`](parameters.md) section of the documentation.

