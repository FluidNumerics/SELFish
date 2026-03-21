# Repository Guidelines

## Project Structure & Module Organization
Source files live under `envs/<cpu>/<gpu>/`, where each leaf directory owns a `spack.yaml` manifest and a `Dockerfile`. Current environments are `envs/x86/rocm` (AMD GPUs), `envs/x86/cuda` (NVIDIA GPUs, parameterised by `CUDA_ARCH` and `CUDA_VERSION` build args), and `envs/x86/none` (CPU-only). Keep CPU targets (`x86`, …) and accelerator targets (`rocm`, `cuda`, `none`) as separate directories so images stay purpose-built, and limit the root `README.md` to high-level context.

## Build, Test, and Development Commands
- `docker build -f envs/x86/rocm/Dockerfile --build-arg GPU_ARCH=gfx90a -t selfish:x86-rocm-gfx90a .` — builds the ROCm image for a specific AMD GPU arch.
- `docker build -f envs/x86/cuda/Dockerfile --build-arg CUDA_ARCH=70 --build-arg CUDA_VERSION=12.4 -t selfish:x86-cuda-sm70 .` — builds the CUDA image for a specific NVIDIA GPU arch.
- `docker build -f envs/x86/none/Dockerfile -t selfish:x86-none .` — builds the CPU-only image.
- `docker run --rm selfish:x86-none spack find hdf5` — smoke-tests that the expected view was installed inside the image.

## Coding Style & Naming Conventions
Spack YAML uses 2-space indentation, lowercase keys, and quoted constraint strings (`"target=x86_64_v3"`). Group `specs` alphabetically, keep `packages` overrides sorted by scope, and rely on multiline `RUN` blocks with trailing `\` alignment plus brief comments for non-obvious workarounds. Name new environments after the hardware tuple (`x86/cuda`, `x86/none`) so downstream scripts can glob predictably.

## Testing Guidelines
For each environment change, run `spack spec` followed by `spack install --fail-fast` inside a disposable builder container to verify concretization. Container builds must pass `docker build` locally before review; capture the last ~20 lines for the PR description. When adding MPI/HDF5 variants, run `docker run --rm <tag> mpichversion` (or another representative binary) to prove runtime availability. There is no coverage gate, but every new spec should ship with at least one build log, and GitHub Actions now double-checks gfx90a builds and publishes them to `higherordermethods/selfish`.

## Commit & Pull Request Guidelines
Existing history uses short, imperative subject lines (“Initial commit”); follow the same format and include the touched environment in parentheses when practical, e.g., `Add feq-parse 2.2.2 to gfx90a`. One logical change per commit keeps bisects clean. PRs should describe the motivation, list updated directories, attach the relevant `spack spec` or `docker build` excerpt, and link any upstream SELF issues. Paste terminal snippets when reviewing GPU-specific behavior.

## Security & Configuration Tips
Pin base images (`rockylinux:9`) and Spack refs in manifests, and run `dnf update -y` at build time to pick up CVEs. Never embed registry credentials or cluster hostnames in `spack.yaml`; rely on build-time secrets where required. Before publishing, scan the resulting image with `docker scout cves selfish:gfx90a` (or equivalent) to catch dependency vulnerabilities.
