# Verify

## After building

```bash
grep -E "susfs_inline|susfs_feature_check|SUSFS_VERSION" build.log
# -- BakaSU/susfs_feature_check: selinux_hide manual hook found
# -- BakaSU/susfs_inline: ksu_handle_{execveat,faccessat,input_handle_event,setresuid,stat,sys_read,sys_reboot} found
# -- SUSFS_VERSION: v2.3.0

strings -a out/arch/arm64/boot/Image | grep -m1 "Linux version"   # stock vermagic suffix
grep -c " susfs_" out/System.map                                   # ~90
grep -c " ksu_"   out/System.map                                   # ~340

# KABI: no CRC may change compared to a build without SUSFS
join -j 2 <(awk '{print $1, $2}' ksu-only/Module.symvers | sort -k2) \
          <(awk '{print $1, $2}' out/Module.symvers      | sort -k2) | awk '$2 != $3'
# (no output)

grep -E "^CONFIG_(TOOLS_SUPPORT_)?RELR=" out/.config   # same as the stock ikconfig
```

## On the device

Flash the image (back up `boot_<slot>` first). On the LG V60 do not use `fastboot boot` for this:
the bootloader truncates the kernel command line on a RAM boot.

```bash
adb shell su -c id                                      # uid=0(root) ... context=u:r:ksu:s0
adb shell su -c "/data/adb/ksud debug info"             # runtime_mode: built-in

adb push ksu_susfs_arm64 /data/local/tmp/ksu_susfs      # from susfs4ksu ksu_module_susfs/tools
adb shell su -c "chmod 755 /data/local/tmp/ksu_susfs"
adb shell su -c "/data/local/tmp/ksu_susfs show version"            # v2.3.0
adb shell su -c "/data/local/tmp/ksu_susfs show variant"            # NON-GKI
adb shell su -c "/data/local/tmp/ksu_susfs show enabled_features"   # one line per enabled CONFIG_KSU_SUSFS_*

adb shell su -c "grep -c -E ' (ksu_|susfs_)' /proc/kallsyms"        # 0 with HIDE_KSU_SUSFS_SYMBOLS
adb shell 'cat /proc/modules | wc -l'                               # same as stock
adb logcat -d -b kernel | grep -E "Oops|BUG:|WARNING: CPU|disagrees about version|Unknown symbol"
```

LG V60 specific (must look like stock):

```bash
adb shell getprop ro.product.model                                   # LM-V600
adb shell 'cat /proc/cmdline | tr " " "\n" | wc -l'                  # 72
adb logcat -d -b kernel | grep -E "ESOC_BOOT_DONE|use cnt file"
# esoc-mdm:ESOC_BOOT_DONE ...
# tfa98xx ...: use cnt file rev_no 12, fw_name tfa98xx.cnt
```

Then install the `susfs4ksu` module (same `gki-android12-5.10` checkout) and test the hiding rules
with your detector of choice.
