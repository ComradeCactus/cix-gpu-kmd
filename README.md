# cix-gpu-kmd

Out-of-tree Mali kernel driver (`mali_kbase`) for the CIX Sky1 / CD8180 SoC
with Mali-G720-Immortalis MC10 GPU (Valhall/Titan architecture).

Based on ARM Mali DDK r54p1 sources from the CIX cix-go-2025q4 SDK, with
patches to build against mainline kernels 6.12 through 7.0.

Part of the [Sky1 Linux](https://github.com/Sky1-Linux) project.

## Modules

The build produces three kernel modules:

| Module | Description |
|--------|-------------|
| `mali_kbase` | Main GPU kernel driver |
| `memory_group_manager` | Memory group manager |
| `protected_memory_allocator` | Protected memory allocator |

## Kernel Compatibility

| Kernel | Status | Notes |
|--------|--------|-------|
| 6.12 LTS | Builds | Vendor baseline |
| 6.15+ | Builds | hrtimer_setup, MALI_CFLAGS rename |
| 6.16+ | Builds | fence_value_str removal |
| 6.17+ | Builds | Page migration disabled without PGTY_mali_gpu patch |
| 6.18-6.19 | Builds | mm_get_unmapped_area signature change |
| 7.0 | Builds | Tested with linux-next |

The driver builds on both Sky1-patched kernels (`CONFIG_ARCH_CIX=y`) and
generic ARM64 kernels. Features that require Sky1-specific kernel patches
(RCSU register access, GPU core harvesting, page migration on 6.17+)
degrade gracefully when the patches are absent.

### PGTY_mali_gpu kernel patch (6.17+)

Kernel 6.17 replaced the `__SetPageMovable` API with a page-type-based scheme.
For full page migration support, the kernel needs a `PGTY_mali_gpu` page type
defined in `include/linux/page-flags.h`. Sky1 Linux kernels carry this patch.

Without the patch, the driver still builds and runs — page migration is
automatically disabled at init with an informational message.

### CONFIG_ARCH_CIX

The Sky1 platform code uses RCSU registers and GPU core harvesting masks that
are defined in the `kbase_device` struct behind `#if IS_ENABLED(CONFIG_ARCH_CIX)`.
On kernels without `CONFIG_ARCH_CIX`, these features are compiled out and the
driver falls back to using `shader_present` for the core mask.

## Building with DKMS

Install the source tree into the DKMS directory and build:

```bash
sudo cp -r . /usr/src/cix-gpu-kmd-1.0.0
sudo dkms add cix-gpu-kmd/1.0.0
sudo dkms build cix-gpu-kmd/1.0.0
sudo dkms install cix-gpu-kmd/1.0.0
```

Or install the pre-built DKMS deb package:

```bash
sudo apt install cix-gpu-kmd-dkms
```

After installation, load the driver:

```bash
sudo modprobe mali_kbase
```

## Manual Build

```bash
make KDIR=/lib/modules/$(uname -r)/build all
sudo make KDIR=/lib/modules/$(uname -r)/build modules_install
```

## Patches

The following patches (on top of the vendor import) adapt the driver for
mainline kernels and fix hardware-specific issues:

### Mainline Porting
1. Fix DKMS build: pass KDIR to make invocations
2. Replace EXTRA_CFLAGS with MALI_CFLAGS (removed in kernel 6.15)
3. Fix Kbuild conditionals for out-of-tree module builds
4. Port to kernel 6.15+: hrtimer_setup and timer_delete_sync
5. Port to kernel 6.16+: remove fence_value_str callback
6. Port to kernel 6.17+: page migration movable_ops API
7. Port to kernel 6.19+: mm_get_unmapped_area signature change
8. Guard CONFIG_ARCH_CIX-specific code for generic kernel builds

### Sky1 Platform Fixes
9. Remove vendor SCMI ifdefs, fix dev_pm_domain_detach args
10. Make IPA thermal initialization failure non-fatal (ACPI boot)
11. Register Energy Model via SCMI perf domain for thermal throttling
12. Force non-cacheable GPU access for DMA-BUF imports without IOMMU
13. Fix infinite hang in GPU cache clean wait (2s timeout + reset recovery)

### DMA-BUF Non-Cacheable Fix (#12)

On Sky1, the GPU lacks an IOMMU entry in the firmware's IORT table. Without
SMMU-mediated coherency, GPU write-back cache lines are not visible to the
display controller (DPU), causing screen corruption. The fix forces
`BASE_MEM_UNCACHED_GPU` on DMA-BUF imports when the device has no IOMMU,
ensuring the DPU always sees up-to-date framebuffer contents.

### Cache Clean Timeout Fix (#13)

The vendor driver's `kbase_gpu_wait_cache_clean()` uses
`wait_event_interruptible()` with no timeout, causing an infinite hang if
the GPU fails to complete a cache flush (e.g., after a fault). Replaced with
a 2-second `wait_event_timeout()` followed by GPU reset recovery if the
wait expires.

## Related Repositories

- [sky1-gpu-support](https://github.com/Sky1-Linux/sky1-gpu-support) — GPU stack switcher, Mali UMD, and Vulkan compat layer
- [vulkan-wsi-layer](https://github.com/Sky1-Linux/vulkan-wsi-layer) — Vulkan WSI with DRI3 and Xwayland bypass presenters
- [linux](https://github.com/Sky1-Linux/linux) — Kernel with Sky1 patches (includes PGTY_mali_gpu)
- [apt](https://github.com/Sky1-Linux/apt) — APT repository with pre-built packages

## License

GPLv2. See `license.txt`.
