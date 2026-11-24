---
title: NextITS
description: >-
    NextITS is a Nextflow pipeline for metabarcoding fungi and other eukaryotes
    with full-length ITS (PacBio) and short-read (Illumina) data.
hide:
  - navigation
  - toc
---

<div class="nx-hero" markdown>
<div class="nx-hero__text" markdown>

<h1>NextITS</h1>

<p class="nx-hero__tagline">
A Nextflow pipeline for metabarcoding fungi and other eukaryotes with full-length ITS from PacBio (Illumina paired-end reads are also supported).
Raw reads to a curated OTU table in two steps, with every tool in a container.
</p>

[Get started](quickstart.md){ .md-button .md-button--primary }
[Installation](installation.md){ .md-button }

<p class="nx-links" markdown>
[GitHub](https://github.com/vmikk/NextITS)
[Docker Hub](https://hub.docker.com/r/vmikk/nextits)
[AI agents](agents.md)
</p>

</div>
<div class="nx-hero__logo" markdown>
![NextITS logo](assets/NextITS_logo_square.webp){ .off-glb }
</div>
</div>

<div class="nx-features" markdown>
<div markdown>
<p class="nx-features__title">Long reads</p>
<p>Full-length ITS from PacBio HiFi, as BAM or FASTQ, with symmetric, asymmetric or single barcodes.</p>
</div>
<div markdown>
<p class="nx-features__title">Homopolymer correction</p>
<p>Fixes indels in homopolymer runs, the main error type in HiFi reads and common in fungal ITS.</p>
</div>
<div markdown>
<p class="nx-features__title">Tag-jump removal</p>
<p>The UNCROSS2 algorithm removes cross-sample contamination within each sequencing run.</p>
</div>
<div markdown>
<p class="nx-features__title">Chimera detection</p>
<p>Reference-based and <em>de novo</em> detection, with rescue of false positives that occur as non-chimeric sequences elsewhere.</p>
</div>
<div markdown>
<p class="nx-features__title">Denoising and clustering</p>
<p>UNOISE, DADA2, SWARM, VSEARCH and taxon-aware OptimOTU, followed by LULU-style curation.</p>
</div>
<div markdown>
<p class="nx-features__title">Reproducible</p>
<p>Docker, Podman, Singularity or Apptainer, from a laptop to HPC. Every run writes reports, a methods summary and software versions.</p>
</div>
</div>

## How it works

<div class="nx-steps" markdown>
<div markdown>
<p class="nx-steps__label">Step 1 · per sequencing run</p>

Quality control, demultiplexing, primer trimming, ITS extraction,
homopolymer correction, chimera detection and tag-jump removal.

<figure class="nx-figure" markdown>
![NextITS workflow, Step-1](assets/NextITS_Workflow_Step1.webp)
</figure>
</div>
<div markdown>
<p class="nx-steps__label">Step 2 · across runs</p>

Pooling of Step-1 results from all runs, filtering, optional denoising,
clustering into OTUs and curation of the OTU table.

<figure class="nx-figure" markdown>
![NextITS workflow, Step-2](assets/NextITS_Workflow_Step2.webp)
</figure>
</div>
</div>

Tag switching happens within a sequencing run, so that is where it is removed.
New runs can be added to a project without reprocessing the old ones.
Click a figure to enlarge it.

## Citation

<div class="nx-cite" markdown>
Mikryukov V., Anslan S., Tedersoo L. NextITS: a pipeline for metabarcoding fungi and other eukaryotes
with full-length ITS sequenced with PacBio. Zenodo.
DOI: [10.5281/zenodo.15074881](https://doi.org/10.5281/zenodo.15074881)
</div>

Please also cite the tools NextITS uses. They are listed, with references, in
`pipeline_info/README_Step1_Methods.txt` and `README_Step2_Methods.txt` in your output folder.
NextITS is released under the Apache-2.0 license.
