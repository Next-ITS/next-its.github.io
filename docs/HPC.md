---
title: Running on HPC
description: >-
    Run NextITS on an HPC cluster with Slurm or another job scheduler.
---

# Running on HPC

Nextflow supports most HPC schedulers, including [Slurm](https://slurm.schedmd.com/), PBS/Torque, SGE and LSF,
so NextITS runs on a university cluster without changes to the pipeline.
This page uses Slurm in its examples. If you don't know which scheduler your cluster uses, ask your HPC support team.

Most clusters provide Singularity or Apptainer rather than Docker, so use `-profile singularity` or `-profile apptainer`.
Set the image cache to a shared folder, so the image is downloaded and converted only once:

```bash
export NXF_SINGULARITY_CACHEDIR="$HOME/singularity_cache"   # or NXF_APPTAINER_CACHEDIR
```

## Two ways to run

**Single job.** You request one node (or part of one) and run the whole pipeline inside that job.
Nextflow runs all tasks locally on the allocated CPUs.
This is the simplest setup and works well for most datasets.

**Scheduler mode.** Nextflow runs as a small, long-lived job and submits each task as a separate Slurm job with its own resources.
Use it for very large datasets, or when a single node is not enough.
The run takes longer when the queue is busy.

!!! info "Further reading"
    [Best practices for deploying pipelines with HPC workload managers](https://www.nextflow.io/blog/2023/best-practices-deploying-pipelines-with-hpc-workload-managers.html)
    on the Nextflow blog.

## Single job

Request the resources in the batch script, and tell Nextflow not to give any single task more than that.
Nextflow's `resourceLimits` does this:

```groovy title="limits.config"
process.resourceLimits = [ cpus: 32, memory: 128.GB, time: 48.h ]
```

```bash title="nextits_step1.sh"
#!/bin/bash
#SBATCH --job-name=NextITS_S1
#SBATCH --cpus-per-task=32
#SBATCH --mem=128G
#SBATCH --time=48:00:00

module load apptainer          # if needed; name depends on your cluster
export NXF_APPTAINER_CACHEDIR="$HOME/apptainer_cache"
export NXF_OPTS="-Xms1g -Xmx4g"  # memory for Nextflow itself

nextflow run vmikk/NextITS -r main \
  -profile apptainer \
  -c limits.config \
  -resume \
  --step       Step1 \
  --input      "$PWD/Input/Run01/Run01.fastq.gz" \
  --barcodes   "$PWD/Input/Run01/Run01_barcodes.fasta" \
  --chimera_db "$PWD/DB/Eukaryome_1.9.3_241222_FullITS_100-800.udb" \
  --outdir     "$PWD/Step1_Results/Run01"
```

```bash
sbatch nextits_step1.sh
```

!!! tip "The `hpc` profile"
    The default settings suit a workstation. On a large node, `-profile hpc,apptainer` gives the heavy steps
    (demultiplexing, ITS extraction, clustering) more CPUs. Combine it with `limits.config` so that no task asks for more than your allocation.

## Scheduler mode

Create a config file that sets the executor and the resources of individual processes.
The values below are a starting point; adjust them to your data and your cluster's limits.

```groovy title="slurm.config"
executor {
    name            = 'slurm'
    queueSize       = 100          // max jobs in the queue at once
    submitRateLimit = '10 sec'     // be gentle with the scheduler
}

process {
    clusterOptions = '--partition=main'   // partition, account, etc.

    // defaults for all processes
    cpus   = 1
    memory = 4.GB
    time   = 4.h

    // retry tasks that ran out of memory or time
    errorStrategy = { task.exitStatus in [137,140,143] ? 'retry' : 'finish' }
    maxRetries    = 2

    // heavy steps
    withName: '(S1|seqstats):demux'               { cpus = 12; memory = 8.GB  }
    withName: 'S1:ITS_EXTRACTION:itsx'            { cpus = 16; memory = 16.GB }
    withName: 'S1:CHIMERA_REMOVAL:chimera_ref'    { cpus = 8;  memory = 8.GB  }
    withName: 'S2:dereplication'                  { cpus = 8;  memory = 32.GB }
    withName: 'S2:CLUSTERING:cluster_vsearch'     { cpus = 32; memory = 64.GB; time = 24.h }
    withName: 'S2:CLUSTERING:cluster_swarm'       { cpus = 32; memory = 64.GB; time = 24.h }
    withName: 'S2:lulu'                           { cpus = 16; memory = 32.GB }
}
```

Then start Nextflow itself as a small, long-running job:

```bash
sbatch --cpus-per-task=2 --mem=8G --time=7-00:00:00 --wrap \
  "nextflow run vmikk/NextITS -r main -profile apptainer -c slurm.config -resume --step Step2 --data_path $PWD/Step1_Results --outdir $PWD/Step2_Results"
```

The work directory (`-work-dir`, default `./work`) must be on a file system that all compute nodes can access.

`-profile hpc_utslurm` is the configuration we use on the University of Tartu cluster.
You can use [its config file](https://github.com/vmikk/NextITS/blob/main/conf/hpc_utslurm.config)
as a template: it lists resources for every process.

## Choosing resources

- **Run a pilot.** Process a few typical samples first, then look at `pipeline_info/execution_report_*.html`.
  It shows the CPU, memory and time each process actually used.
- **Heavy steps.** ITS extraction (ITSx) takes most of the time in Step-1.
  In Step-2, clustering and dereplication of large pooled datasets need the most memory.
- **Large Step-2 datasets.** Use `--chunking_n` to split the data and cluster the chunks in parallel. Each task then needs less memory.
- **Don't over-request.** Asking for more than the job needs makes it wait longer in the queue and takes resources from other users.
- **Clean up.** The work directory can grow large. Once you have the results, copy them (see [Where results are stored](usage.md#where-results-are-stored)) and delete `work/`.
