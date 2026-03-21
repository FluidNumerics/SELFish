# SELFish
SELF **i**mage **s**haring **h**ub

This repository defines a suite of common container images for developing [SELF](https://github.com/fluidnumerics/self) to target different platforms. Each image recipe, defined here as a combination of a [`spack.yaml`](https://spack.readthedocs.io/en/latest/containers.html) and a generated `Dockerfile`, contains all the instructions necessary to install SELF's dependencies. The intention is to provide container images wherein someone can easily get set up to develop SELF in a common environment.

While SELF does support bare-metal builds and those are regularly tested, the core SELF team at Fluid Numerics is working on a standardized develop, build, and deploy workflow that leverages container environments. We are opting to build Docker container images as these are easily shareable broadly through Dockerhub. Additionally, Docker images can be easily converted for use on shared traditional HPC platforms as singularity/apptainer or enroot images.

The core SELF team at Fluid Numerics has adopted enroot+pyxis with Slurm for our deployment model due to positive experience with this approach.

See [Repository Guidelines](CLAUDE.md) for contributor expectations, build commands, and review checklists.


## Organization

The `envs/` subdirectory defines all of the base environments that are aimed at providing base images with all the dependencies required for developing SELF. The subdirectory structure is `envs/{cpu_platform}/{gpu_backend}`.

| Directory | GPU backend | Build args | MPI |
|-----------|-------------|------------|-----|
| `envs/x86/rocm/` | AMD ROCm | `GPU_ARCH` (e.g. `gfx90a`), `GPU_BACKEND_VERSION` (e.g. `6.4.3`) | OpenMPI |
| `envs/x86/cuda/` | NVIDIA CUDA | `CUDA_ARCH` (e.g. `70`, `100`), `CUDA_VERSION` (e.g. `12.4`, `13.0`) | OpenMPI |
| `envs/x86/none/` | None (CPU-only) | — | OpenMPI |

Each directory contains a `spack.yaml` manifest, a `Dockerfile`, and a `feq-parse.patch`.

## Container Images

SELFish provides pre-built container images with all dependencies for spectral element computations. Images are published to Docker Hub under `higherordermethods/selfish`.

### Image Tagging Scheme

Images follow the pattern: `higherordermethods/selfish:<version>-<cpu_platform>-<gpu_backend>-<gpu_arch>`

- **`<version>`**: `latest` or a commit SHA
- **`<cpu_platform>`**: Target CPU architecture (e.g. `x86`)
- **`<gpu_backend>`**: GPU backend with version (e.g. `rocm643`, `cuda124`) or `none` for CPU-only
- **`<gpu_arch>`**: Target GPU architecture (e.g. `gfx90a`, `sm70`); omitted for CPU-only images

#### Examples
```bash
# AMD MI210/MI250 (gfx90a) with ROCm 6.4.3
docker pull higherordermethods/selfish:latest-x86-rocm643-gfx90a

# NVIDIA V100 (sm70) with CUDA 12.4
docker pull higherordermethods/selfish:latest-x86-cuda124-sm70

# NVIDIA Blackwell (sm100) with CUDA 13.0
docker pull higherordermethods/selfish:latest-x86-cuda130-sm100

# CPU-only
docker pull higherordermethods/selfish:latest-x86-none
```

### Supported Architectures

#### AMD ROCm
| Architecture | GPU Models | Tag |
|--------------|------------|-----|
| gfx906 | MI50, MI60, Radeon VII | `latest-x86-rocm643-gfx906` |
| gfx90a | MI210, MI250, MI250X | `latest-x86-rocm643-gfx90a` |
| gfx942 | MI300A, MI300X | `latest-x86-rocm643-gfx942` |

#### NVIDIA CUDA
| Architecture | GPU Models | Tag |
|--------------|------------|-----|
| sm70 | V100 | `latest-x86-cuda124-sm70` |
| sm100 | B200, B300 | `latest-x86-cuda130-sm100` |

#### CPU-only
| Tag |
|-----|
| `latest-x86-none` |

### Determining Your GPU Architecture

For AMD GPUs:
```bash
rocminfo | grep "Name:" | grep "gfx"
```

For NVIDIA GPUs:
```bash
nvidia-smi --query-gpu=compute_cap --format=csv,noheader
```

### Using with Slurm

Specify the architecture-specific image in your job script:
```bash
#!/bin/bash
#SBATCH --gpus=1
#SBATCH --container-image=higherordermethods/selfish:latest-x86-rocm643-gfx90a

./run_simulation.sh
```

### Image Metadata

All images include OCI labels for programmatic inspection:
```bash
docker inspect higherordermethods/selfish:latest-x86-cuda124-sm70 | grep -A5 Labels
```

Key labels:
- `com.fluidnumerics.rocm.target` / `com.fluidnumerics.cuda.target`: GPU architecture target
- `com.fluidnumerics.rocm.version` / `com.fluidnumerics.cuda.version`: Backend version
- `org.opencontainers.image.source`: Source repository
- `org.opencontainers.image.revision`: Git commit SHA
