# SELFish
SELF **i**mage **s**haring **h**ub

This repository defines a suite of common container images for developing [SELF](https://github.com/fluidnumerics/self) to target different platforms. Each image recipe, defined here as a combination of a [`spack.yaml`](https://spack.readthedocs.io/en/latest/containers.html) and a generated `Dockerfile`, contains all the instructions necessary to install SELF's dependencies. The intention is to provide container images wherein someone can easily get set up to develop SELF in a common environment.

While SELF does support bare-metal builds and those are regularly tested, the core SELF team at Fluid Numerics is working on a standardized develop, build, and deploy workflow that leverages container environments. We are opting to build Docker container images as these are easily shareable broadly through Dockerhub. Additionally, Docker images can be easily converted for use on shared traditional HPC platforms as singularity/apptainer or enroot images.

The core SELF team at Fluid Numerics has adopted enroot+pyxis with Slurm for our deployment model due to positive experience with this approach.

See [Repository Guidelines](CLAUDE.md) for contributor expectations, build commands, and review checklists.


More docs coming soon


## Organization

The `envs/` subdirectory defines all of the base environments that are aimed at providing base images with all the dependencies required for developing SELF. The subdirectory structure is as `envs/{cpu_platform}/{gpu_backend}`. When `{gpu_platform}=none`, that environment is an environment for working with non-gpu accelerated implementations of SELF.

## Container Images

SELFish provides pre-built container images with all dependencies for GPU-accelerated spectral element computations. Images are tagged using a **version-architecture** naming scheme to support multiple GPU targets.

### Image Tagging Scheme

Images follow the pattern: `higherordermethods/selfish:<version>-<cpu_platform>-<gpu_backend>-<gpu_arch>`

- **`<version>`**: Semantic version (e.g., `v1.2.3`) or release channel (`latest`, `dev`)
- **`<cpu_platform>`** : Target cpu architecture (e.g. `x86`, `arm` )
- **`<gpu_backend>`** : GPU backend provider with version (e.g. `rocm643`, `cuda112`)
- **`<gpu_arch>`**: Target GPU architecture (e.g., `gfx90a`, `gfx906`, `gfx942`)

#### Examples:
```bash
# Stable release for MI210/MI250 (gfx90a)
docker pull higherordermethods/selfish:v1.2.3-gfx90a

# Latest stable for Radeon Instinct MI100 (gfx908)
docker pull higherordermethods/selfish:latest-gfx908

# Development build for MI300A (gfx942)
docker pull higherordermethods/selfish:dev-gfx942
```

### Supported GPU Architectures

| Architecture | GPU Models | Tag Suffix |
|--------------|------------|------------|
| gfx90a | MI210, MI250, MI250X | `-gfx90a` |
| gfx908 | MI100 | `-gfx908` |
| gfx906 | MI50, MI60, Radeon VII | `-gfx906` |
| gfx942 | MI300A, MI300X | `-gfx942` |
| sm_72  | V100 | -sm72 |

### Determining Your GPU Architecture

If you're unsure which image to use, check your GPU architecture.

For AMD GPUs,
```bash
# Using rocminfo
rocminfo | grep "Name:" | grep "gfx"

# Using rocm-smi
rocm-smi --showproductname
```

### Using with Slurm

Specify the architecture-specific image in your job script:
```bash
#!/bin/bash
#SBATCH --gpus=1
#SBATCH --container-image=higherordermethods/selfish:v1.2.3-gfx90a

./run_simulation.sh
```

### Version Pinning Recommendations

- **Production**: Pin to specific versions (e.g., `v1.2.3-gfx90a`) for reproducibility
- **Development**: Use `latest-<arch>` for convenience (auto-updates with new releases)
- **Testing CI**: Use `dev-<arch>` to test against bleeding-edge builds

### Image Metadata

All images include OCI labels for programmatic inspection:
```bash
docker inspect higherordermethods/selfish:v1.2.3-gfx90a | grep -A5 Labels
```

Key labels:
- `com.fluidnumerics.rocm.target`: GPU architecture target
- `com.fluidnumerics.selfish.version`: SELFish version
- `org.opencontainers.image.version`: Container image version
