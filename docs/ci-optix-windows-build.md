# Experimental GitHub OptiX Build

This branch adds a Windows GitHub Actions build for `Denoiser.exe`.

The workflow lives at `.github/workflows/optix-windows-build.yml` and runs on:

- pushes to `ci/github-optix-build-experiment`
- manual `workflow_dispatch`

What it does:

- checks out the repository with the `contrib/optix` submodule
- installs CUDA Toolkit 12.9.0 with the NVIDIA network installer
- installs Conan 2
- builds the C++ denoiser with CMake and Visual Studio 2022
- installs `Denoiser.exe` into `out/bin`
- packages a zip artifact containing:
  - `Denoiser.exe`
  - `LICENSE`
  - `manifest.json`

The hosted GitHub runner build only proves compilation and packaging. It does not run a real denoise smoke test because GitHub-hosted Windows runners do not provide an NVIDIA GPU for OptiX execution.

The artifact manifest records the app source SHA, OptiX submodule SHA, CUDA version, executable SHA256, and executable size so downstream apps can pin a known build.
