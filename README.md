# SELFish
SELF **i**mage **s**haring **h**ub

This repository defines a suite of common container images for developing [SELF](https://github.com/fluidnumerics/self) to target different platforms. Each image recipe, defined here as a combination of a [`spack.yaml`](https://spack.readthedocs.io/en/latest/containers.html) and a generated `Dockerfile`, contains all the instructions necessary to install SELF's dependencies. The intention is to provide container images wherein someone can easily get set up to develop SELF in a common environment.

While SELF does support bare-metal builds and those are regularly tested, the core SELF team at Fluid Numerics is working on a standardized develop, build, and deploy workflow that leverages container environments. We are opting to build Docker container images as these are easily shareable broadly through Dockerhub. Additionally, Docker images can be easily converted for use on shared traditional HPC platforms as singularity/apptainer or enroot images.

The core SELF team at Fluid Numerics has adopted enroot+pyxis with Slurm for our deployment model due to positive experience with this approach.

See [Repository Guidelines](AGENTS.md) for contributor expectations, build commands, and review checklists.


More docs coming soon


## Organization

The `envs/` subdirectory defines all of the base environments that are aimed at providing base images with all the dependencies required for developing SELF. The subdirectory structure is as `envs/{cpu_platform}/{gpu_platform}`. When `{gpu_platform}=none`, that environment is an environment for working with non-gpu accelerated implementations of SELF.
