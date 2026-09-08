---
layout: default
title: MGB ERIS Nucleus
nav_order: 3
parent: Platforms
---

# Running MCMICRO on MGB ERIS Nucleus

MGB ERIS Nucleus is a Slurm-based high-performance computing environment at Mass General Brigham. The `MGB` profile configures MCMICRO to use Slurm, Singularity/Apptainer, and the Nucleus GPU partition observed on `erishpc-login-001`.

## Setting up for MCMICRO on Nucleus

1. Load Nextflow and Singularity/Apptainer.

```
module purge
module load Nextflow/25.10.0
module load singularity/latest
```

If `singularity` is not available after loading `singularity/latest`, load Apptainer directly:

```
module load Apptainer/1.4.2-1.el9
```

1. Confirm the runtime commands are available.

```
which java
java -version
which nextflow
nextflow -version
which singularity || which apptainer
singularity --version || apptainer --version
```

1. Use scratch storage for the Nextflow work directory.

```
mkdir -p /scratch/$USER/mcmicro-work
mkdir -p $HOME/.mcmicro/singularity
```

## Checking available resources

The Nucleus partition and GPU names should be verified with Slurm before running a large job:

```
bash setup/MGB_resources.sh
```

To test GPU allocation:

```
bash setup/MGB_resources.sh --test-gpu
```

The observed GPU resource on `erishpc-login-001` is:

```
partition: gpu-l40s
gres:      gpu:nvidia_l40s:1
```

## Running MCMICRO

Submit the Nextflow launcher through Slurm instead of running a full pipeline directly on the login node. A minimal submission script is:

```
#!/bin/bash
#SBATCH -p normal
#SBATCH -J mcmicro
#SBATCH -o mcmicro-%j.log
#SBATCH -t 12:00:00
#SBATCH --mem=4G
#SBATCH -c 2

module purge
module load Nextflow/25.10.0
module load singularity/latest

mkdir -p /scratch/$USER/mcmicro-work
mkdir -p $HOME/.mcmicro/singularity

nextflow run labsyspharm/mcmicro \
  --in "$DATASETDIR" \
  -profile MGB \
  -w /scratch/$USER/mcmicro-work \
  -resume
```

Submit with:

```
sbatch submit_mcmicro_mgb.sh
```

## Customizing GPU requests

If the GPU partition or GRES name changes, override the defaults:

```
nextflow run labsyspharm/mcmicro \
  --in "$DATASETDIR" \
  -profile MGB \
  --mgb_gpu_queue gpu-l40s \
  --mgb_gpu_gres gpu:nvidia_l40s:1 \
  -w /scratch/$USER/mcmicro-work \
  -resume
```

## Dynamic resource requests

The `MGB` profile retries Slurm jobs that exit with statuses commonly associated with resource exhaustion and increases memory/time on retry. It also uses the bundled `setup/MGB_ome_tiff_gpx.py` helper script for OME-TIFF image-size-aware memory estimates.

To use a different gigapixel helper script, override the default:

```
--mgb_ome_tiff_gpx_script /path/to/ome-tiff-gpx.py
```

If the helper cannot read an image, the profile uses conservative fallback memory values.
