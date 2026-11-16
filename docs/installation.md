---
title: Installation
description: >-
    Install Nextflow and a container engine, download the reference database, and run a test.
---

# Installation

NextITS runs on Linux. On Windows, use the [Windows Subsystem for Linux (WSL)](https://learn.microsoft.com/en-us/windows/wsl/install).
macOS works with Docker, but it is not tested regularly.

You only need to install two things yourself:

| Requirement | Version |
| --- | --- |
| [Nextflow](https://www.nextflow.io/) (needs Java 17–26) | **26.04.0 or later** |
| A container engine: [Docker](https://www.docker.com/), [Podman](https://podman.io/), [Singularity](https://sylabs.io/singularity/) or [Apptainer](https://apptainer.org/) | any recent version |

Nextflow downloads the pipeline and its container image on the first run.
Every bioinformatics tool the pipeline uses is inside that image.

## Nextflow

=== "Manual"

    Install Java with [SDKMAN!](https://sdkman.io/), then Nextflow:

    ```bash
    curl -s https://get.sdkman.io | bash
    # open a new terminal, then:
    sdk install java 21-tem

    curl -s https://get.nextflow.io | bash
    chmod +x nextflow
    mkdir -p ~/bin && mv nextflow ~/bin/
    ```

=== "Conda / Mamba"

    ```bash
    mamba create -n nextflow -c conda-forge -c bioconda nextflow
    mamba activate nextflow
    ```

Check the installation:

```bash
nextflow -version
```

If you already have Nextflow, update it with `nextflow self-update`.
For details, see the [Nextflow installation guide](https://www.nextflow.io/docs/latest/install.html).

## Container engine

Pick one. Docker or Podman is usually the simplest choice on a workstation.
On a shared HPC cluster, Singularity or Apptainer is usually the only one available.
Select the engine at run time with `-profile docker`, `-profile podman`, `-profile singularity`
or `-profile apptainer`.

=== "Docker"

    Follow the [Docker Engine installation guide](https://docs.docker.com/engine/install/) for your distribution.
    Then let your user run Docker without `sudo`:

    ```bash
    sudo groupadd docker
    sudo usermod -aG docker $USER
    newgrp docker
    ```

=== "Podman"

    Podman is available in the package repositories of most distributions:

    ```bash
    sudo apt install podman      # Debian/Ubuntu
    ```

=== "Singularity / Apptainer"

    See the [Apptainer](https://apptainer.org/docs/admin/main/installation.html) or
    [SingularityCE](https://docs.sylabs.io/guides/latest/admin-guide/installation.html) installation guides.
    On Ubuntu, Apptainer is available from a PPA:

    ```bash
    sudo add-apt-repository -y ppa:apptainer/ppa
    sudo apt update && sudo apt install -y apptainer
    ```

    !!! tip "On an HPC cluster"
        The engine is usually installed as an environment module. Find and load it with:
        ```bash
        module avail singularity apptainer
        module load apptainer
        ```

### Container image

All profiles use one image from Docker Hub, [`vmikk/nextits`](https://hub.docker.com/r/vmikk/nextits).
Nextflow pulls it on the first run. Singularity and Apptainer convert it to a `.sif` file and cache it.

On a cluster, put the cache on a shared file system so the image is converted only once:

```bash
export NXF_SINGULARITY_CACHEDIR="/shared/path/singularity_cache"   # -profile singularity
export NXF_APPTAINER_CACHEDIR="/shared/path/apptainer_cache"       # -profile apptainer
```

??? note "Pull the image manually (e.g., for offline nodes)"

    ```bash
    # Docker
    docker pull vmikk/nextits:1.2.4

    # Singularity / Apptainer
    singularity pull nextits-1.2.4.sif docker://vmikk/nextits:1.2.4
    ```

    To use a local `.sif` file, point Nextflow to it in a custom config (see [Configuring runs](usage.md#configuring-runs)):

    ```groovy
    // three slashes are required for an absolute path
    process.container = 'file:///path/to/nextits-1.2.4.sif'
    ```

??? note "Build the image yourself"

    ```bash
    git clone https://github.com/vmikk/NextITS
    cd NextITS/containerfiles

    # Docker
    docker build --tag nextits:local --file NextITS.dockerfile .

    # Singularity / Apptainer
    singularity build --fakeroot nextits.sif NextITS.def
    ```

## Databases

### Chimera database

Reference-based chimera detection (enabled by default) needs a database in VSEARCH's UDB format.
We provide one built from [EUKARYOME](https://eukaryome.org/) v1.9.3:
full-length ITS sequences of 100–800 bp, extracted with ITSx.

[Download the database](https://owncloud.ut.ee/owncloud/s/PkasGDNDimNssm3){ .md-button }

From the command line:

```bash
wget -O Eukaryome_1.9.3_241222_FullITS_100-800.udb \
  "https://owncloud.ut.ee/owncloud/public.php/dav/files/PkasGDNDimNssm3"
```

Pass the path to the file with `--chimera_db`. The default value is a bare file name,
`Eukaryome_1.9.3_241222_FullITS_100-800.udb`, which Nextflow looks for in the directory you launch the pipeline from.
Using an absolute path avoids surprises:

```bash
--chimera_db "/path/to/Eukaryome_1.9.3_241222_FullITS_100-800.udb"
```

If you don't need reference-based chimera detection, use `--chimera_methods denovo`
(or `none`, which disables chimera detection completely).

