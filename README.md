# susfs-4.19-nongki

**SUSFS v2.3.0 backported to a 4.19 NonGKI kernel**, for ReSukiSU / BakaSU.

[susfs4ksu](https://gitlab.com/simonpunk/susfs4ksu) no longer supports NonGKI kernels, and its
`kernel-4.19` branch is stuck at v1.5.5 (Feb 2025), which current ReSukiSU no longer builds
against. This repository is a manual port of the kernel side of the `gki-android12-5.10` branch
(v2.3.0) to 4.19.

## Scope: read this first

The patches are a **reference port for one tree**, not a drop-in patch for every 4.19 kernel:

| | Pinned to |
|---|---|
| Kernel | [LineageOS `android_kernel_lge_sm8250`](https://github.com/LineageOS/android_kernel_lge_sm8250) @ `29902cf733dc` (4.19.325-cip133-st17, LG V60 `timelm`, LineageOS 23.2-20261018) |
| SUSFS | `susfs4ksu` `gki-android12-5.10` @ `eb95da0` (v2.3.0) |
| KernelSU | [ReSukiSU](https://github.com/ReSukiSU/ReSukiSU) `d5e9e025` (BakaSU, KSU_VERSION 35252, UAPI 6) |
| Toolchain | AOSP clang `r563880c` (build 14054515) |

Parts of the port depend on that tree: LG's `fs/proc/cmdline.c`, the `fs_context` backport this
tree carries (`vfs_create_mount()`), and Android's `show_options2()` (sdcardfs). On another 4.19
tree, expect to re-port with [`docs/PORTING.md`](docs/PORTING.md); the per-file notes there are
the reusable part. ReSukiSU tracks only the latest SUSFS, so a newer ReSukiSU will likely need a
newer SUSFS port.

## Status

| | |
|---|---|
| Build | 0 errors, 0 warnings (`-Werror`), all ReSukiSU SUSFS hook checks found, `-- SUSFS_VERSION: v2.3.0` |
| KABI | `Module.symvers`: 0 changed CRCs, 0 removed symbols vs the KernelSU-only build |
| Device (LM-V600VM, flashed) | boots; `su` → `u:r:ksu:s0`; `ksud` `runtime_mode: built-in`; `ksu_susfs show version` → `v2.3.0`, `show variant` → `NON-GKI`, 9 features; ksu/susfs symbols hidden in `/proc/kallsyms`; 34 vendor modules (same as stock); modem, speaker tuning and model name like stock |
| **Not tested yet** | hiding rules with the `susfs4ksu` module and detector apps; long-term daily use (largest changes are in `fs/namei.c`, `fs/namespace.c`) |

Use at your own risk and keep a backup of your `boot` partition.

## Patches

`patches/lge-sm8250-29902cf733dc/`, applied in order on a clean checkout of the kernel:

| Patch | What |
|---|---|
| `0001-kernelsu-integration.patch` | `drivers/kernelsu` → `../ReSukiSU/kernel` link, `drivers/Kconfig` / `drivers/Makefile` lines |
| `0002-susfs-v2.3.0-4.19.patch` | SUSFS inline hooks (exec, stat, open, read, reboot, setresuid, input, selinux) and the SUSFS kernel side: `fs/susfs.c`, `include/linux/susfs.h`, `include/linux/susfs_def.h` and hooks in 25 files |
| `0003-setlocalversion-fixed.patch` | optional, keeps the stock `-g29902cf733dc` suffix so vermagic matches the vendor modules (from [`timelm-bakasu`](https://github.com/SkyLumi/timelm-bakasu)) |

In ReSukiSU `KSU_SUSFS` is a hooking method (a Kconfig `choice` with `KSU_MANUAL_HOOK`), so
these patches carry **no** manual hooks: SUSFS provides them.

## Quick start

```bash
git clone -b lineage-23.2 https://github.com/LineageOS/android_kernel_lge_sm8250 kernel
git -C kernel checkout 29902cf733dc
git clone https://github.com/ReSukiSU/ReSukiSU.git kernel/ReSukiSU
git -C kernel/ReSukiSU checkout d5e9e025

cd kernel
P=../susfs-4.19-nongki/patches/lge-sm8250-29902cf733dc
git apply $P/0001-kernelsu-integration.patch
git apply $P/0002-susfs-v2.3.0-4.19.patch
git apply $P/0003-setlocalversion-fixed.patch

mkdir -p out
scripts/extract-ikconfig ../boot-stock.img > out/.config
cat ../susfs-4.19-nongki/config/susfs.fragment >> out/.config
scripts/config --file out/.config -d LOCALVERSION_AUTO

export PATH=/path/to/clang-r563880c/bin:$PATH
MK="make O=out ARCH=arm64 CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm OBJCOPY=llvm-objcopy \
    OBJDUMP=llvm-objdump STRIP=llvm-strip CROSS_COMPILE=aarch64-linux-gnu- \
    CROSS_COMPILE_ARM32=arm-linux-gnueabi-"
$MK olddefconfig
$MK -j$(nproc) Image modules
```

Pass the full LLVM toolset to `olddefconfig` too, otherwise `CONFIG_RELR` (on in stock) is
silently dropped. The full build recipe for this device (toolchain fetch, stock `boot.img`,
repack) is in [SkyLumi/timelm-bakasu](https://github.com/SkyLumi/timelm-bakasu).

Checks after building and flashing: [`docs/VERIFY.md`](docs/VERIFY.md).

## Device notes (LG V60)

- Validate by **flashing** the image, not with `fastboot boot`: LG's bootloader truncates the
  kernel command line on a RAM boot (35 instead of 72 entries), which breaks the model name,
  speaker tuning and the modem, also with the stock kernel.
- SUSFS spoofs `/proc/cmdline` for every process once a fake buffer is set. Keep the
  `androidboot.vendor.lge.*` entries in the fake cmdline file.

## Credits

- [simonpunk/susfs4ksu](https://gitlab.com/simonpunk/susfs4ksu): SUSFS itself; this is only a port
- [ReSukiSU](https://github.com/ReSukiSU/ReSukiSU) / [BakaSU](https://github.com/Baka-SU/BakaSU): the KernelSU fork with NonGKI and SUSFS support
- [LineageOS](https://github.com/LineageOS): the `lge_sm8250` kernel
- [SkyLumi/timelm-bakasu](https://github.com/SkyLumi/timelm-bakasu): KernelSU build for the LG V60 this work started from

## License

See [`NOTICE.md`](NOTICE.md). In short: GPL-3.0 as in susfs4ksu (`LICENSE`), and the changes to
Linux kernel files are GPL-2.0 like the kernel.
