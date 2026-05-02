# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Super-Builders is a CI-driven kernel build pipeline for [ZeroMount](https://github.com/Enginex0/zeromount). It applies SUSFS hiding infrastructure + ZeroMount VFS patches to stock GKI kernel sources and builds them across all supported Android/kernel versions and KSU variants via GitHub Actions.

This repo does **not** contain kernel source code. It contains patches, build scripts, CI workflows, and device profiles that together produce pre-patched kernel images.

## Repository Structure

```
android{ver}-{kver}/          # Per-version patch set (e.g. android14-6.1)
  {KSU-variant}/patches/      # Variant-specific patch files
  build-helpers/              # Runtime build scripts
  samsung-fixes/              # Samsung KDP conflict patches (not all versions)
  defconfig.fragment          # Kernel config flags ([base],[susfs],[zram],[overlayfs],[kpm])
zram/                         # LZ4 v1.10.0 upgrade patches per kernel version
manifests/                    # OEM kernel manifests (bbk/)
.github/workflows/            # CI pipeline (entry point: main.yml)
.github/inputs/               # Device registries (bbk-devices.json, samsung-devices.json)
device-profiles.json          # Stock kernel identity spoofing profiles
```

## Patch System

Patches use a numbered prefix system. Application order is critical:

1. **Samsung fixes** (pre-50, device-specific KDP patches) — only for Samsung builds
2. **`50_add_susfs_in_gki`** — Base SUSFS integration into GKI kernel
3. **`51_enhanced_susfs`** — Enhanced SUSFS features (hiding, spoofing, hooks)
4. **`70_ksu_safety`** — Applied to the KSU variant source tree (not kernel tree)
5. **`60_zeromount`** — ZeroMount VFS engine
6. **Runtime fixes** — `fix-susfs-compat.sh` handles sublevel-dependent API differences

`50_`, `51_`, and `60_` patches are variant-independent (same file across all KSU variants). `70_` is variant-specific.

## KSU Variants

Four variants, each in its own subdirectory under each `android{ver}-{kver}/`:
- **SukiSU Ultra** — `SukiSU-Ultra/`
- **ReSukiSU** — `ReSukiSU/`
- **KernelSU-Next** — `KernelSU-Next/`
- **WKSU (WildKSU)** — `WildKSU/`

Android 12 (5.4) only supports SukiSU and ReSukiSU — KernelSU-Next/WKSU lack pre-5.7 compatibility.

## Build Helpers

Scripts in `build-helpers/` run during CI to handle build-time adaptation:

| Script | Purpose |
|--------|---------|
| `assemble-defconfig.sh` | Merges defconfig.fragment sections based on feature flags, deduplicates entries |
| `fix-susfs-compat.sh` | Patches source for sublevel-dependent API differences (6 major fixes) |
| `fix-old-kernel-compat.sh` | Compatibility fixes for older kernel versions |
| `bypass-abi-check.sh` | Disables ABI symbol list enforcement for modified kernels |
| `clean-build-flags.sh` | Spoofs kernel version strings via setlocalversion/mkcompile_h |
| `clean-module-list.sh` | Module list cleanup |
| `setup-bbg.sh` | Baseband Guard integration |
| `report-config.sh` | Final config reporting |

## CI Pipeline

**Entry point:** `main.yml` dispatches to version-specific `kernel-a{ver}-{kver}.yml` workflows, which call reusable `build-{variant}.yml` workflows.

**Trigger a build:**
```bash
gh workflow run main.yml --ref main \
  -f kernel_build_version=a14-6-1 \
  -f ksu_variant=SukiSU \
  -f add_susfs=true \
  -f add_zeromount=true \
  -f add_zram=true \
  -f device_codename=generic
```

**OEM builds** use separate workflows (`build-samsung-{variant}.yml`, `build-bbk-{variant}.yml`) that reference device registries in `.github/inputs/`.

**Build systems:**
- Kernel 5.10/5.15/6.1: Legacy `build/build.sh` with `GKI_DEFCONFIG_FRAGMENT`
- Kernel 6.6+: Kleaf/Bazel with `--defconfig_fragment`
- Kernel 6.12: Requires Rust support, LTO=none

**Feature flags** (workflow inputs): `add_susfs`, `add_zeromount`, `add_zram`, `add_bbg`, `add_overlayfs_support`, `add_kpm`

## Kernel Versions

| Directory | Android | Kernel | Note |
|-----------|---------|--------|------|
| `android12-5.4` | 12 | 5.4 | Pre-GKI, limited variant support |
| `android12-5.10` | 12 | 5.10 | |
| `android13-5.10` | 13 | 5.10 | |
| `android13-5.15` | 13 | 5.15 | |
| `android14-5.15` | 14 | 5.15 | |
| `android14-6.1` | 14 | 6.1 | |
| `android15-6.6` | 15 | 6.6 | |
| `android16-6.12` | 16 | 6.12 | |

## Device Profiles

`device-profiles.json` stores stock kernel metadata (version string, build user/host, compiler info) for identity spoofing. The `clean-build-flags.sh` script rewrites `scripts/setlocalversion` and `scripts/mkcompile_h` so the built kernel reports matching stock identity.

OEM device registries (`.github/inputs/bbk-devices.json`, `samsung-devices.json`) define per-device manifest repos, build commands, image paths, and fix patches.

## Samsung Fixes

`samsung-fixes/` directories contain device-specific patches that resolve KDP (Kernel Data Protection) conflicts between Samsung's kernel modifications and SUSFS hooks. Applied **before** `50_` SUSFS patches. Organized by device model (e.g., `SM-S928B/`).

## ZRAM LZ4 Upgrade

`zram/` contains LZ4 v1.10.0 source and per-kernel-version integration patches that upgrade the kernel's bundled LZ4 for better ZRAM compression, enabling LZ4K/LZ4KD algorithms.
