# Experimental GitHub OptiX Build

This branch adds a Windows GitHub Actions build for `Denoiser.exe`.

Repository note: the working repository is `Ahmed-Hindy/NvidiaAIDenoiser`.
The user-facing link initially used `NvidiaAIDenoise`, but that repository did
not exist when checked with `gh repo view`.

## Branch

Experimental branch:

```text
ci/github-optix-build-experiment
```

The branch was created from the validated HDU multipart EXR branch:

```text
hdu/multilayer-exr-output @ 8893b605903f273512b750d45993bbe27a003362
```

Current experiment commits:

```text
b7f5a79 Pin CI OptiX headers to validated v9
c9fc1ee Use network CUDA installer in OptiX CI
6150c59 Add experimental GitHub OptiX build workflow
```

## Workflow

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

The experimental branch pins `contrib/optix` to OptiX dev `v9.0.0` (`fff65c2a7c592f1ea5f1661ad7d2381cf965f9bd`), matching the previously validated app-bundled denoiser build.

## CI Runs

Initial run:

```text
run: 26027524406
commit: 6150c59d6a8ca1645857309cb42c08ecfa8e572d
status: cancelled
reason: full CUDA installer path was still setting up after roughly 15 minutes
```

Second run:

```text
run: 26028256942
commit: c9fc1ee83594ff49cbaed693e2bfdb60733bc589
status: success
duration: about 26 minutes
artifact: optix-denoiser-windows-x64-c9fc1ee83594ff49cbaed693e2bfdb60733bc589
```

The second run built and uploaded an artifact, but local runtime sanity showed
that the executable found the RTX 3070 and then failed `optixInit()` with OptiX
error `7801`. The artifact manifest reported the `contrib/optix` submodule at
`f1f6dd803f3159992d248178f6e09421c6eb8b6d` (`OptiX 9.1.0`), which did not match
the previously validated app-bundled build.

Final validated run:

```text
run: 26029585690
url: https://github.com/Ahmed-Hindy/NvidiaAIDenoiser/actions/runs/26029585690
commit: b7f5a795f512816ca21be27af79a945fe4d8ea7d
status: success
duration: about 20 minutes
artifact: optix-denoiser-windows-x64-b7f5a795f512816ca21be27af79a945fe4d8ea7d
artifact api id: 7056464316
artifact size: 31500443 bytes
```

## Final Artifact Manifest

Downloaded and extracted artifact contents:

```text
Denoiser.exe   78138880 bytes
LICENSE            1092 bytes
manifest.json       642 bytes
```

Manifest:

```json
{
  "name": "NvidiaAIDenoiser",
  "executable": "Denoiser.exe",
  "source_repository": "https://github.com/Ahmed-Hindy/NvidiaAIDenoiser.git",
  "source_ref": "ci/github-optix-build-experiment",
  "source_commit": "b7f5a795f512816ca21be27af79a945fe4d8ea7d",
  "optix_dev_repository": "https://github.com/NVIDIA/optix-dev.git",
  "optix_dev_commit": "fff65c2a7c592f1ea5f1661ad7d2381cf965f9bd",
  "cuda_version": "12.9.0",
  "runner_os": "Windows",
  "runner_arch": "X64",
  "built_at_utc": "2026-05-18T11:24:55.8210819Z",
  "sha256": "8e0376a2aee8167c6a817a01382c247310c381ec00781a403a0077d93d45fa5e",
  "size_bytes": 78138880
}
```

## Local Runtime Validation

The final artifact was downloaded with:

```powershell
gh run download 26029585690 `
  --repo Ahmed-Hindy/NvidiaAIDenoiser `
  --name optix-denoiser-windows-x64-b7f5a795f512816ca21be27af79a945fe4d8ea7d `
  --dir $env:TEMP\nvidia-denoiser-ci-artifact-b7f5a79
```

Cheap runtime sanity:

```powershell
Denoiser.exe -h
```

Result:

```text
Found 1 CUDA device(s)
GPU 0: NVIDIA GeForce RTX 3070 (compute 8.6) with 8191MB memory
Command line parameters
```

The program exits with code `1` after `-h` because it continues argument
validation and reports `No input image could be loaded`. The important check is
that the final artifact initializes OptiX and reaches help output; the previous
OptiX 9.1.0 artifact failed before this point.

Manual Canyon Run smoke:

```powershell
Denoiser.exe `
  -v 1 `
  -multipart G:\Projects\AYON_PROJECTS\Canyon_Run\sq001\sh001\publish\render\renderFxMain\v001\CanRun_sh001_renderFxMain_v001.exr `
  -o $env:TEMP\nvidia-denoiser-ci-b7f5a79-canyon-output.exr `
  -beauty-name C `
  -albedo-name albedo `
  -normal-name N `
  -aov-name0 directdiffuse `
  -aov-name1 indirectdiffuse
```

Result:

```text
Denoising complete in 0.044 seconds
Written out: C:\Users\AHMEDH~1\AppData\Local\Temp\nvidia-denoiser-ci-b7f5a79-canyon-output.exr
```

Raw EXR header comparison against the Canyon Run source:

```json
{
  "source_part_count": 22,
  "output_part_count": 22,
  "first_parts": [
    "C",
    "albedo",
    "C_emission",
    "C_light_distant_1"
  ],
  "last_parts": [
    "indirectglossyreflection_light_distant_1",
    "indirectglossyreflection_light_dome_1",
    "N",
    "P"
  ],
  "raw_attribute_value_diff_count": 0,
  "attribute_order_diff_count": 0,
  "value_diffs": [],
  "order_diffs": []
}
```

## Notes For Future Agents

- Keep `contrib/optix` pinned to `fff65c2a7c592f1ea5f1661ad7d2381cf965f9bd`
  until a newer OptiX SDK is explicitly validated at runtime.
- Hosted GitHub runners can prove build and packaging, but runtime validation
  still needs a machine with an NVIDIA GPU.
- Git on this workstation needed `http.sslBackend = schannel` to avoid local
  issuer certificate failures while fetching GitHub repos/submodules.
- The current Actions artifact is temporary. If this experiment graduates, add
  a tag or release workflow that uploads the zip as a durable GitHub release
  asset.
