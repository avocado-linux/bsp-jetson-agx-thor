# Changelog

All notable changes to avocado-bsp-jetson-agx-thor are documented in this file.
The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.1]

### Fixed
- CUDA could not initialize on the AGX Thor devkit: every call returned 801 /
  `cudaErrorNotSupported`, `/dev/nvidia0` was absent, and TensorRT died with
  "CUDA initialization failure with error: 801". Root cause was that this BSP
  hand-lists what upstream pulls via `MACHINE_EXTRA_RDEPENDS` -- which Avocado's
  minimal rootfs drops -- and the GPU half of that list was never transcribed.
  Three separate gaps, all needed for CUDA:
  - **`tegra-drm` was missing.** `libcuda` probes host1x for its DRM child
    (`.../tegra-host1x/*.host1x/*.host1x.drm/drm`) during `cuInit`; without
    `tegra-drm` that node never appears and `cuInit` fails with
    `CUDA_ERROR_NOT_SUPPORTED` and no kernel-side error. This was the actual
    blocker. Added `nv-kernel-module-tegra-drm` (upstream ships it in
    `nvidia-kernel-oot-display`); dnf pulls cec / drm-display-helper /
    drm-dp-aux-bus behind it. Two traps: the bare `kernel-module-tegra-drm`
    name resolves to the *in-tree* tegra124..194 driver, which has no tegra264
    support and leaves CUDA broken exactly as if tegra-drm were absent, so the
    `nv-` spelling is required; and tegra-drm has no tegra264 modalias (it
    attaches as a host1x driver), so it never autoloads and is listed under
    `modprobe:` to be loaded at extension merge.
  - **No device nodes.** Avocado has no `nvidia-modprobe` and openrm does not
    create `/dev/nvidiaN` itself, so nothing made the nodes. Added
    `tegra-configs-udev`, whose `99-tegra-devices.rules` creates
    `/dev/nvidia0`, `/dev/nvidiactl`, `/dev/nvidia-modeset` and
    `/dev/nvidia-uvm{,-tools}`. Added `tegra-nvpower` for the `nvpower.sh` those
    rules invoke on GPU bind.
  - **Both GPU driver stacks installed, racing for the PCI bind.** `nvgpu.ko`
    carries a PCI modalias for Thor's Blackwell GPU (10DE:2B00), the same device
    `nvidia.ko` claims; when nvgpu won, NVRM never ran its per-GPU probe ("GPU
    0000:01:00.0 is already bound to nvgpu"). Dropped `kernel-module-nvgpu` and
    added `tegra-configs-display-driver`, which is upstream's own arbitration for
    the openrm SoC variant: `install nvgpu /bin/false`, plus the `install nvidia`
    chain that loads `nv-gpu-static-pg` with `gpu_pg_mask` before `nvidia`.
    `tegra-nvpmodel-base` comes along for the `gpu_pg_mask` file that chain
    reads. The Orin BSP's nvgpu entry is correct for tegra234, where the iGPU is
    an OF platform device and `nvidia.ko` drives display only -- that list must
    not be copied to Thor.

  Also added `tegra-configs-sysctl` and `tegra-nvsciipc` to complete the upstream
  `MACHINE_EXTRA_RDEPENDS` set. Verified on hardware: `cuInit` succeeds, the GPU
  enumerates as `NVIDIA Thor sm_110` with 122.8 GiB, and a CUDA kernel launch
  returns correct results both on the host and inside
  `nvcr.io/nvidia/cuda:13.2.1-devel-ubuntu24.04` via `docker run --runtime nvidia`.

## [0.1.0]

### Added
- Initial release: Board support for the Nvidia Jetson AGX Thor.
- CI via the shared `avocado-linux/actions` reusable workflows: PR build check
  (`test.yml`) and tag-driven package + publish to the Avocado feed (`release.yml`).
