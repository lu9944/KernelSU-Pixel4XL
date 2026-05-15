# KernelSU-Pixel4XL Build Guide

<!-- AI-CONTEXT
This document is a build guide for an Android kernel project.
Target: Pixel 4 XL (codename: coral, platform: floral, SoC: Snapdragon 855 / SM8150).
The kernel integrates KernelSU for root management.
All commands assume working directory is the project root: /home/ubuntu/code/KernelSU-Pixel4XL
-->

## Project Metadata

| Key | Value |
|-----|-------|
| Kernel version | `4.14.276` (see `Makefile`: VERSION=4, PATCHLEVEL=14, SUBLEVEL=276) |
| KernelSU version | `v0.9.5` (commit `b766b98513b5a7eb33bc1c4a76b5702bf1288f07`) |
| Target device | Pixel 4 XL (coral) / Pixel 4 (flame) |
| Device codename / platform | `floral` |
| SoC | Qualcomm Snapdragon 855 (SM8150) |
| Android version | 13 (TP1A.220624.014) |
| Architecture | `arm64` (aarch64) |
| Defconfig | `arch/arm64/configs/floral_defconfig` |
| Boot image format | Android Boot Image Header v2 |
| Required toolchain | Clang r416183b + aarch64 GCC 4.9 + arm GCC 4.9 |

## Known-Working Reference Image

A known-working boot.img built from this source code is available for comparison:

| Field | Value |
|-------|-------|
| File | `/home/ubuntu/code/pixel4xl_android13_4.14.276_v095.img` |
| Size | 37.4 MB (39,219,200 bytes) |
| Status | Fully functional — touchscreen, WiFi, all hardware working |

Reference image boot header (for comparison when debugging packing issues):

| Header Field | Value |
|--------------|-------|
| Magic | `ANDROID!` |
| Header version | 2 |
| Page size | 4096 |
| Kernel size | 15,599,872 bytes (14.9 MB) |
| Kernel addr | `0x00008000` |
| Ramdisk size | 22,138,014 bytes (21.1 MB) |
| Ramdisk addr | `0x01000000` |
| Tags addr | `0x00000100` |
| DTB size | 1,473,632 bytes (1.4 MB) |
| DTB addr | `0x01f00000` |
| cmdline | `console=ttyMSM0,115200n8 androidboot.console=ttyMSM0 ...` |

<!-- AI-NOTE: Use this reference image to diff against any newly packed boot.img.
     Compare header fields, kernel size, ramdisk size, DTB size and content.
     This is the ground truth for a correct boot.img produced from this source. -->

## Build Artifacts

| Artifact | Path | Description |
|----------|------|-------------|
| Kernel image | `out/arch/arm64/boot/Image.lz4` | LZ4-compressed kernel binary |
| DTB | `out/arch/arm64/boot/dts/google/qcom-base/sm8150-v2.dtb` | Device tree blob for SM8150 v2 |
| Final output | `boot-KernelSU-Pixel4XL.img` | Flashable boot image (kernel + ramdisk + DTB) |

## Network Proxy

All external downloads (GitHub repos, Google factory images) require proxy access.

```bash
export http_proxy=http://192.168.10.198:10808
export https_proxy=http://192.168.10.198:10808
export no_proxy=localhost,127.0.0.1,192.168.0.0/16,10.0.0.0/8
```

## Prerequisites

### System Packages (Ubuntu 22.04)

```bash
sudo apt-get update
sudo apt-get install -y build-essential bc bison flex libssl-dev libelf-dev \
  python3 device-tree-compiler lz4 dwarves wget unzip
```

Additional fix: GCC 4.9 cross-compiler requires `/usr/bin/python`:
```bash
sudo ln -sf /usr/bin/python3 /usr/bin/python
```

### Toolchain Setup

Three toolchains are required, cloned into `toolchains/`:

| Toolchain | Directory | Source Repo | Branch |
|-----------|-----------|-------------|--------|
| Clang r416183b | `toolchains/clang` | `LineageOS/android_prebuilts_clang_kernel_linux-x86_clang-r416183b` | `lineage-20.0` |
| aarch64 GCC 4.9 | `toolchains/gcc-64` | `LineageOS/android_prebuilts_gcc_linux-x86_aarch64_aarch64-linux-android-4.9` | `lineage-19.1` |
| arm GCC 4.9 | `toolchains/gcc-32` | `LineageOS/android_prebuilts_gcc_linux-x86_arm_arm-linux-androideabi-4.9` | `lineage-19.1` |

```bash
mkdir -p toolchains
git clone --depth=1 https://github.com/LineageOS/android_prebuilts_clang_kernel_linux-x86_clang-r416183b -b lineage-20.0 toolchains/clang
git clone --depth=1 https://github.com/LineageOS/android_prebuilts_gcc_linux-x86_aarch64_aarch64-linux-android-4.9 -b lineage-19.1 toolchains/gcc-64
git clone --depth=1 https://github.com/LineageOS/android_prebuilts_gcc_linux-x86_arm_arm-linux-androideabi-4.9 -b lineage-19.1 toolchains/gcc-32
```

### KernelSU Setup

KernelSU source lives in `KernelSU/` and is linked into the kernel driver tree via symlink `drivers/kernelsu -> ../KernelSU/kernel`.

```bash
git clone https://github.com/tiann/KernelSU.git KernelSU_tmp
cd KernelSU_tmp && git fetch origin b766b98513b5a7eb33bc1c4a76b5702bf1288f07 --depth=1 && git checkout FETCH_HEAD && cd ..
rm -rf KernelSU && mv KernelSU_tmp KernelSU
rm -f drivers/kernelsu && ln -s ../KernelSU/kernel drivers/kernelsu
```

Verify: `ls drivers/kernelsu/` should show `.c` and `.h` files (allowlist.c, core_hook.c, etc.)

## Build Steps

All `make` commands use the same set of cross-compilation flags. Define them once:

```bash
export PATH="$(pwd)/toolchains/clang/bin:$(pwd)/toolchains/gcc-64/bin:$(pwd)/toolchains/gcc-32/bin:$PATH"
export ARCH=arm64
export KBUILD_BUILD_HOST=local-build
export KBUILD_BUILD_USER=KernelSU

MAKE_FLAGS="O=out CROSS_COMPILE=aarch64-linux-android- CROSS_COMPILE_ARM32=arm-linux-androideabi- CC=clang CLANG_TRIPLE=aarch64-linux-gnu- LD=ld.lld"
```

### Step 1: Generate kernel config

```bash
make $MAKE_FLAGS floral_defconfig
```

### Step 2: Disable LTO/CFI

<!-- AI-NOTE: LTO and CFI must be disabled for standalone builds (outside AOSP tree).
     Without this step, the linker will fail due to missing CFI/LTO infrastructure. -->

```bash
scripts/config --file out/.config -d LTO -d LTO_CLANG -d CFI -d CFI_PERMISSIVE -d CFI_CLANG
make $MAKE_FLAGS olddefconfig
```

### Step 3: Build kernel image

```bash
make -j$(nproc) $MAKE_FLAGS Image.lz4
```

Output: `out/arch/arm64/boot/Image.lz4`

### Step 4: Build DTB

```bash
make -j$(nproc) $MAKE_FLAGS arch/arm64/boot/dts/google/qcom-base/sm8150-v2.dtb
```

Output: `out/arch/arm64/boot/dts/google/qcom-base/sm8150-v2.dtb`

## Pack boot.img

<!-- AI-WARNING: Do NOT reference .github/workflows/build.yml for boot.img packing.
     The CI workflow produces a boot.img that boots but has BROKEN touchscreen and WiFi.
     The exact root cause is still unknown. Other developers have successfully built
     working boot.img images from this same source code, which means:
       1. The kernel source itself is correct and complete.
       2. The problem is in the boot.img packing/creation process, not the kernel.
       3. The CI's Python repack script is likely flawed — possibly incorrect DTB handling,
          wrong kernel image format, or improper boot header reconstruction.
     If you are implementing boot.img packing, investigate the differences between the
     CI output and known-working images built by others. -->

### Known Issues with CI Workflow boot.img

| Symptom | Status |
|---------|--------|
| Touchscreen not working | CI-produced boot.img has this issue |
| WiFi not working | CI-produced boot.img has this issue |
| Root cause | Unknown. The same kernel source can produce working images when packed differently |
| Verified by | Other developers have built fully functional boot.img from this source code |

<!-- AI-NOTE: The floral platform defines external modules in build.config.floral.common:
     - private/msm-google-modules/wlan/qcacld-3.0  → wlan.ko (WiFi driver)
     - private/msm-google-modules/touch/fts/floral  → FTS touch panel driver
     These are listed for reference, but may or may not be the actual cause of the failures.
     Since others have built working images from this source, the kernel itself is functional. -->

### Recommended Approach (TODO)

Since the kernel source is known to produce working images, the focus should be on getting the boot.img packing correct:

**Option A: Compare with known-working image**
1. Obtain a known-working boot.img built by others from this source
2. Compare boot header fields (cmdline, page_size, header version, DTB offset)
3. Compare kernel image format (Image vs Image.lz4 vs Image.gz-dtb)
4. Identify the exact difference in packing process

**Option B: Use mkbootimg / unpackbootimg from AOSP**
1. Use `mkbootimg` / `unpackbootimg` tools from AOSP instead of hand-written Python
2. These tools handle Android boot image format correctly and robustly
3. May resolve subtle format issues in the CI's Python script

**Option C: Use Android Image Kitchen (AIK)**
1. Unpack stock boot.img with AIK
2. Replace kernel image
3. Repack with AIK
4. AIK preserves all original metadata and handles format edge cases

**Option D: Use Image.lz4-dtb instead of separate DTB**
1. The stock kernel uses `Image.lz4-dtb` (kernel + DTB concatenated) format
2. See `build.config.common` FILES list which includes `arch/arm64/boot/Image.lz4-dtb`
3. Try appending DTB directly to the kernel image instead of using boot header v2 DTB field

### Flash

```bash
fastboot flash boot boot-KernelSU-Pixel4XL.img
```

## Key Project Files

| File | Purpose |
|------|---------|
| `Makefile` | Kernel version: 4.14.276 |
| `arch/arm64/configs/floral_defconfig` | Kernel config for floral platform |
| `build.config.floral` | Build config entry point for Pixel 4 XL |
| `build.config.floral.common` | defconfig, output files, external modules |
| `build.config.floral.common.clang` | Clang compiler settings for floral |
| `build.config.common` | Global settings (toolchain paths, architecture) |
| `.github/workflows/build.yml` | CI/CD pipeline (full automated build + release) |
| `KernelSU/` | KernelSU source (git cloned, NOT part of this repo) |
| `drivers/kernelsu` | Symlink → `../KernelSU/kernel` (kernel module glue) |

## Critical Notes

<!-- AI-NOTE: These are hard requirements. Skipping any of them will cause build failure. -->

1. **KernelSU must be cloned before build** — `KernelSU/` is empty in the repo; the symlink `drivers/kernelsu` points to it.
2. **Toolchain versions are pinned** — Clang `r416183b`, GCC `4.9`. Other versions are untested.
3. **LTO/CFI must be disabled** — Standalone builds (outside AOSP) lack the LTO/CFI infrastructure; the linker will fail without this step.
4. **Stock boot.img required for packing** — The repack script extracts ramdisk + header (cmdline, page_size, etc.) from the factory image. These cannot be generated from kernel source alone.
5. **Only Image.lz4 and main DTB needed** — DTB overlays (dtbo.img) are not required for boot.img.
6. **GitHub Actions CI has broken boot.img** — The `.github/workflows/build.yml` produces a boot.img without working touchscreen and WiFi. Root cause unknown. The same source code can produce working images — the issue is in the packing process, not the kernel. Do NOT reference the CI script for boot.img creation. See "Pack boot.img" section for details.
7. **KernelSU version updated to v0.9.5** — Commit hash `b766b98513b5a7eb33bc1c4a76b5702bf1288f07`. If compatibility issues arise with kernel 4.14.276, revert to v0.8.1 (commit `3f341a4e3a80c651e3d8f3bbf1070e7768a44cb8`).
