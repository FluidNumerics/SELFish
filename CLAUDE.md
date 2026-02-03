# Repository Guidelines

## Project Structure & Module Organization
Source files live under `envs/<cpu>/<gpu>/`, where each leaf directory owns a `spack.yaml` manifest and (optionally) a generated `Dockerfile`. Keep CPU targets (`x86`, …) and accelerator targets (`gfx90a`, `sm72`, `none`) granular so images stay purpose-built, and limit the root `README.md` to high-level context.

## Build, Test, and Development Commands
- `spack spec -e envs/x86/gfx90a/spack.yaml` — concretizes the manifest locally; run this before opening a PR so dependency drift is caught early.
- `spack containerize envs/x86/gfx90a/spack.yaml > envs/x86/gfx90a/Dockerfile` — regenerates the Dockerfile after manifest edits (avoid hand-tuning output).
- `docker build -f envs/x86/gfx90a/Dockerfile -t selfish:gfx90a .` — builds the shareable runtime image; tag images `<cpu>-<gpu>` for clarity.
- `docker run --rm selfish:gfx90a spack find hdf5` — smoke-tests that the expected view was installed inside the image.

## Coding Style & Naming Conventions
Spack YAML uses 2-space indentation, lowercase keys, and quoted constraint strings (`"target=x86_64_v3"`). Group `specs` alphabetically, keep `packages` overrides sorted by scope, and rely on multiline `RUN` blocks with trailing `\` alignment plus brief comments for non-obvious workarounds. Name new environments after the hardware tuple (`x86/gfx942`, `x86/none`) so downstream scripts can glob predictably.

## Testing Guidelines
For each environment change, run `spack spec` followed by `spack install --fail-fast` inside a disposable builder container to verify concretization. Container builds must pass `docker build` locally before review; capture the last ~20 lines for the PR description. When adding MPI/HDF5 variants, run `docker run --rm <tag> mpichversion` (or another representative binary) to prove runtime availability. There is no coverage gate, but every new spec should ship with at least one build log, and GitHub Actions now double-checks gfx90a builds and publishes them to `higherordermethods/selfish`.

## Commit & Pull Request Guidelines
Existing history uses short, imperative subject lines (“Initial commit”); follow the same format and include the touched environment in parentheses when practical, e.g., `Add feq-parse 2.2.2 to gfx90a`. One logical change per commit keeps bisects clean. PRs should describe the motivation, list updated directories, attach the relevant `spack spec` or `docker build` excerpt, and link any upstream SELF issues. Paste terminal snippets when reviewing GPU-specific behavior.

## Security & Configuration Tips
Pin base images (`rockylinux:9`) and Spack refs in manifests, and run `dnf update -y` at build time to pick up CVEs. Never embed registry credentials or cluster hostnames in `spack.yaml`; rely on build-time secrets where required. Before publishing, scan the resulting image with `docker scout cves selfish:gfx90a` (or equivalent) to catch dependency vulnerabilities.
