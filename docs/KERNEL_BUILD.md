# Kernel build: Android 17 x86-64

This repository now builds a patched Android Common Kernel (ACK) for the first
hardware-validation stage of a17PC. The build is a kernel artifact, not a
complete Android OS image and not yet proof of bootability on the Dell laptop.

## Source baseline

- Repository: `https://android.googlesource.com/kernel/common`
- Ref: `ASB-2026-10-05_17-6.18` (Android 17 / Linux 6.18 security baseline)
- The workflow records the resolved Git commit SHA in `BUILD_INFO.txt`.
- The checked-in starting config is `kernel_a17/kernel-build-output/.config-android17-6.18`.
- The build enables Android Binder IPC/BinderFS, OverlayFS, Intel i915, AHCI,
  EFI stub, ACPI battery/fan support, HDA audio and xHCI where supported by Kconfig.

The build workflow runs on pushes affecting the kernel patch, config, docs, or
workflow, and can also be manually started from GitHub Actions.

## BDPROCHOT patch

The patch is in `patches/bdprochot-android17-6.18.patch`. It intentionally
does **not** use bit 38 of `IA32_MISC_ENABLE` (`0x1a0`); that is the Turbo
Mode Disable bit. Instead it clears bit 0 of `IA32_POWER_CTL` (`0x1fc`),
the bidirectional PROCHOT# enable bit, on each Intel logical CPU during early
initialization.

The Kconfig option `CONFIG_DISABLE_BDPROCHOT` defaults to off in source.
This custom build explicitly enables it to match the intended charger/battery
compatibility use case. Internal CPU thermal control remains active, but
external components can no longer request CPU throttling through bidirectional
PROCHOT#. This can mask a genuine power or thermal fault in a charger, battery,
VRM, or other platform component; use only after diagnosing a false/latched
signal and validating temperatures and power behavior.

## Build output

A successful workflow publishes an artifact containing:

- `kernel-bzImage`: x86-64 kernel image
- `System.map`
- `kernel.config`: final resolved config
- `modules.tar.gz`: modules installed under the matching kernel release
- `BUILD_INFO.txt`: source ref/commit, kernel release, config and patch status
- `SHA256SUMS`: checksums

The workflow verifies that the patch applies, the critical config symbols are
enabled after `olddefconfig`, and the image/modules exist. This is compile-time
verification only. Before using it on hardware, package it with a suitable
Android boot image/initramfs, verify module compatibility and boot arguments,
and test recovery/rollback from external media. The existing `kernel-ranchu`
artifact is not treated as a physical-laptop kernel.

## Local reproduction

On a Linux x86-64 host with kernel build dependencies installed:

1. Clone the official ACK repository at `ASB-2026-10-05_17-6.18`.
2. Apply `patches/bdprochot-android17-6.18.patch`.
3. Copy the checked-in config to `.config`.
4. Use `scripts/config` to enable `DISABLE_BDPROCHOT`, Binder IPC/BinderFS,
   OverlayFS and the laptop drivers, then run `make olddefconfig`.
5. Build with `make -j2 ARCH=x86_64 bzImage modules`.

Do not use the resulting kernel as a drop-in replacement for an existing
kernel until its boot packaging, initramfs and modules have been matched.
