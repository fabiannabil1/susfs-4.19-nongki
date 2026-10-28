# Porting notes: SUSFS v2.3.0 (gki-android12-5.10) → 4.19

Source: `susfs4ksu` `gki-android12-5.10` @ `eb95da0`, `kernel_patches/50_add_susfs_in_gki-android12-5.10.patch`
(129 hunks, 25 files) plus `kernel_patches/fs/susfs.c` and `kernel_patches/include/linux/susfs*.h`.
Target: LineageOS `android_kernel_lge_sm8250` @ `29902cf733dc`.

## Don't trust fuzz

The 5.10 patch applies to this 4.19 tree with `patch -F3` except 9 hunks, but several "successful"
hunks landed in the wrong function and still compiled or nearly did:

- the post-exec hook inside `set_binfmt()`
- extern declarations inside a comment block of `do_faccessat()`
- `susfs_statfs_by_dentry()` defined inside `vfs_statfs()`
- `copy_flags |= CL_COPY_MNT_NS` inside `current_chrooted()`
- the `SUS_MAP` filter of `proc_map_files_readdir()` after its `out:` label

Check every hunk for the function it was written for (the `@@ ... @@ function` context of the 5.10
patch vs. where it landed), and port by hand from a clean file where they differ.

## ReSukiSU side

- `KSU_SUSFS` and `KSU_MANUAL_HOOK` are one Kconfig `choice`. With SUSFS, remove any manual hooks
  (`ksu_handle_execveat` & co. from a manual-hook patch): the SUSFS patch provides all of them.
- The build checks the hooks (`kernel/tools/inline_hook_check.mk`) for `ksu_handle_setresuid`
  (`kernel/sys.c`), `ksu_handle_execveat` (`fs/exec.c`), `ksu_handle_faccessat` (`fs/open.c`),
  `ksu_handle_sys_read` (`fs/read_write.c`), `ksu_handle_stat` (`fs/stat.c`),
  `ksu_handle_sys_reboot` (`kernel/reboot.c`), `ksu_handle_input_handle_event`
  (`drivers/input/input.c`), and warns on leftover `CONFIG_KSU_MANUAL_HOOK` guards.
- `ksu_selinux_hide_running` in `security/selinux/hooks.c` enables the selinux_hide feature.

## Per-file notes

| File | 5.10 | 4.19 port |
|---|---|---|
| `fs/exec.c` | hooks in `do_execveat_common()` | hooks in `__do_execve_file()`, post hook right after `exec_binprm()` (before the `retval < 0` check) |
| module preinit | PID 1 exec retry: `goto orig_flow` after `ksu_preinit_handle_post_execveat()` | **not hooked**. 4.19 would need to unwind `displaced`, `in_exec`, `pathbuf` and `bprm` before retrying, and a mistake there bootloops. Without the hook ReSukiSU simply never starts preinit; modules still run from `post-fs-data` |
| `fs/stat.c` | `vfs_fstat()`; `stat->mnt_id`, `STATX_MNT_ID` | `vfs_statx_fd()`; `struct kstat` has no `mnt_id` before 5.8, so the mnt_id lines are dropped and `STATX_SUS_KSTAT*` is cleared in `vfs_statx()` |
| `fs/open.c` | `do_faccessat(dfd, filename, mode, flags)` | `do_faccessat(dfd, filename, mode)`; same `getname_flags()` + `filename_lookup()` hook |
| `fs/namei.c` `nameidata` | `state` field upstream | add `unsigned int state` (under `SUS_PATH`), reset in `set_nameidata()` |
| `lookup_fast()` | returns a dentry / NULL, `try_to_unlazy()` | returns int: a hidden dentry is treated as "not in dcache" (`unlazy_walk()` + return 0 in RCU mode, `dput()` + return 0 otherwise), so `walk_component()` falls back to `lookup_slow()`, which substitutes the fake qstr |
| `walk_component()` / `lookup_last()` | `WALK_TRAILING` | `walk_component(nd, 0)`; set `ND_STATE_LOOKUP_LAST` in `lookup_last()` and `ND_FLAGS_LOOKUP_LAST` before `lookup_slow()` |
| `open_last_lookups()` | sets `ND_STATE_OPEN_LAST` | does not exist: `do_last()` is the equivalent, set it there |
| `lookup_open()` | returns a dentry | returns int; same fake-qstr substitution around `d_alloc_parallel()` |
| `OPEN_REDIRECT` | re-walks in `path_openat()` between `open_last_lookups()` and `do_open()` | 4.19 `do_last()` looks up **and** opens, so there is no point in between. Done in `do_filp_open()`: if the opened inode is flagged, `fput()` it and open the redirected path (the approach SUSFS 1.5.x used on 4.19). Not applied in `do_tmpfile()` |
| `vfs_readlink()` | same | same |
| `/proc/bootconfig` | `fs/proc/bootconfig.c` | NonGKI has no bootconfig: spoof `/proc/cmdline` instead. On this tree `fs/proc/cmdline.c` is LG's (`updated_command_line`, charger fixup); the spoof goes after the fixup and does not add `'\n'` (the fake buffer is a full copy including it) |
| `security/selinux/avc.c` | `tcontext` printed in `avc_audit_post_callback()` | printed in `avc_dump_query()`; spoof there and keep `kfree(scontext)` on the spoofed path |
| `fs/proc/fd.c` | fdinfo with `ino:` | 4.19 prints `pos/flags/mnt_id` only; the spoofed output must keep that format, so the `ino:` lines and the `kern_path()` lookup for it are dropped |
| `fs/proc_namespace.c` | `susfs_show_vfsmnt/mountinfo/vfsstat()` are copies of the 5.10 functions | re-sync the copies with the tree's own `show_*()`: Android 4.19 calls `->show_options2()` (sdcardfs) before `->show_options()`. Diff the copy against the original; only the `mnt_id >= DEFAULT_KSU_MNT_ID` check may differ |
| `fs/namespace.c` | `vfs_create_mount()`, `alloc_vfsmnt()` with `mnt_stuck_children` | this tree has the `fs_context` backport, so `vfs_create_mount()` exists (vanilla 4.19 would need `vfs_kern_mount()` instead). `susfs_alloc_unshare_ksu_vfsmnt()` / `susfs_alloc_non_unshare_ksu_vfsmnt()` must mirror the tree's `alloc_vfsmnt()`: `mnt.data = NULL`, `init_fs_pin(&mnt->mnt_umount, drop_mountpoint)`, `kstrdup_const(name, GFP_KERNEL)` |
| `fs/proc/base.c` `map_files` | one loop | counting loop and filling loop: filter `SUS_MAP` in **both**, otherwise the count and the entries disagree |
| `fs/proc/task_mmu.c` | `smaps_rollup` with mmap lock contention restart; `VMA_PAD_START()` | plain loop; `vma->vm_end` (no GKI VMA padding); `pagemap_read()` uses `walk_page_range(start, end, &pagemap_walk)` |
| `fs/readdir.c` | | 4.19 has `access_ok()` before the inserted declarations: move them up (`-Wdeclaration-after-statement`) |
| `fs/susfs.c` sdcard monitor | `fsnotify_ops.handle_inode_event(mark, mask, inode, dir, qstr, cookie)` | `->handle_event(group, inode, mask, data, data_type, const unsigned char *file_name, cookie, iter_info)`: compare with `strcmp(file_name, "Android")` |
| `include/linux/susfs_def.h` | | add `#include <linux/cred.h>` (`current_uid()` in inline helpers) |
| unchanged semantics | | `block/ioprio.c`, `drivers/input/input.c`, `fs/read_write.c`, `kernel/sys.c`, `kernel/reboot.c`, `kernel/kallsyms.c`, `mm/memory.c`, `fs/notify/fdinfo.c`, `fs/super.c`, `fs/statfs.c`, `security/selinux/{hooks,selinuxfs}.c`, `fs/Makefile` |

## Flags to check on your tree

| SUSFS | value | compare with |
|---|---|---|
| `ND_FLAGS_LOOKUP_LAST` | `0x2000000` | `LOOKUP_*` in `include/linux/namei.h` (max `0x8000` here) |
| `ND_STATE_LOOKUP_LAST` / `ND_STATE_OPEN_LAST` | `32` / `64` | own field, no conflict |
| `STATX_SUS_KSTAT` / `_FUSE` | `0x10000000` / `0x20000000` | `STATX_*` (max `0x00000fff` + `STATX__RESERVED`) |
| `VFSMOUNT_MNT_FLAGS_KSU_UNSHARED_MNT` | `0x80000000` | `MNT_*` in `include/linux/mount.h` (max `0x8000000` here) |
| `CL_COPY_MNT_NS` | `BIT(25)` | `CL_*` in `fs/pnode.h` (max `0x80` here) |
| `AS_FLAGS_*` | bits 33–39 of `mapping->flags` | `AS_*` in `include/linux/pagemap.h`; needs a 64-bit `unsigned long` |

## KABI

SUSFS v2.3.0 stores its markers in existing flag words (`mapping->flags`, `mnt_flags`, thread
flags) rather than new struct fields, so no struct used by modules changes layout. Compare
`out/Module.symvers` against a build without SUSFS: CRCs must be identical, or vendor modules
(WiFi, audio) will refuse to load.
