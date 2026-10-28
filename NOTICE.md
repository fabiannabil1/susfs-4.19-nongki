# License notice

This repository contains patches against the Linux kernel and a port of SUSFS.

- **SUSFS** (`fs/susfs.c`, `include/linux/susfs.h`, `include/linux/susfs_def.h` and the SUSFS
  hooks) comes from [simonpunk/susfs4ksu](https://gitlab.com/simonpunk/susfs4ksu), whose
  repository is licensed under the **GNU General Public License v3.0**. `LICENSE` is that text,
  copied unchanged from susfs4ksu.
- **Changes to existing Linux kernel files** (`fs/`, `kernel/`, `mm/`, `security/`, `drivers/`,
  `block/`, `scripts/`) are derived from the Linux kernel / LineageOS
  `android_kernel_lge_sm8250` and are distributed under the **GNU General Public License v2.0**,
  the license of those files.
- `0003-setlocalversion-fixed.patch` is from [SkyLumi/timelm-bakasu](https://github.com/SkyLumi/timelm-bakasu)
  (kernel patches there: GPL-2.0).

Copyright of the original code stays with its authors. The 4.19 port adds no new license terms.
