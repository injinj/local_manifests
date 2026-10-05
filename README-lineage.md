# LineageOS 23.2 + AOHP (Pixel 6 "oriole", OnePlus 13 "dodge")

Branch `lineage-23.2-aohp` of this repo holds the local manifest that reproduces the tree built on chex
(`/home/chris/lineage/android`, 2026-10-03/04; reports in `/home/chris/lineage/logs/REPORT-*.md`).
The seven AOHP-patched projects are the `lineage-23.2-aohp` branches of the injinj forks:
`android_build`, `platform_system_core` (branch on the existing aohp-os fork, see comment in aohp.xml), `android_system_sepolicy`,
`android_frameworks_base`, `android_device_google_raviole`, `android_device_oneplus_dodge` and
`android_vendor_aohp` (private). Each is LineageOS `lineage-23.2` + the aohp-os upstream squash + the injinj
commits (sepolicy: `aohp_container_socket` moved to private policy for `sepolicy_freeze_test`; patch 0003
"drop policycap functionfs_seclabel" intentionally NOT applied - Lineage/Pixel keep the policycap).

## Reproduce

```bash
mkdir -p ~/lineage/android && cd ~/lineage/android
repo init -u https://github.com/LineageOS/android.git -b lineage-23.2 --git-lfs --no-clone-bundle --partial-clone --clone-filter=blob:limit=10M
mkdir -p .repo/local_manifests
curl -fsSL https://raw.githubusercontent.com/injinj/local_manifests/lineage-23.2-aohp/aohp.xml -o .repo/local_manifests/aohp.xml
repo sync -c -j8 --no-tags --no-clone-bundle --optimized-fetch --prune --retry-fetches=3
```

`vendor/aohp` is a private repo: `repo sync` needs a GitHub credential with access to injinj
(e.g. `gh auth login` + `gh auth setup-git`, or a `~/.git-credentials` entry).

### AOHP binaries that are not in any repo project

`packages/apps/AOHPAgentDriver` is synced from `injinj/AOHPAgentDriver` (`injinj-main`), but its gitignored
binaries must be put in place by hand (build/make's `generic/Android.bp` and the `aohp-rootfs-*` modules reference them):
`rootfs/debian.tar.gz` (617670439 B, sha256 f81067ab…590; built by `prepare_rootfs.sh` from the aohp-os tree,
chex copy `/aosp/templates/debian-arm64.tar.gz`), `rootfs/alpine.tar.gz`, `AOHPAgentDriver.apk`.
Also copy `packages/apps/AOHPDriver/` (`Android.bp`, `AOHPDriver.apk` 45361061 B sha256 c4e16ce6…,
`initial-package-stopped-states-aohp.xml`) and `packages/apps/OpenClawAndroid/` (`Android.bp`, `OpenClawAndroid.apk`)
from the chex tree (or rebuild them from github.com/injinj/aohp-driver and the OpenClaw Android app; OpenClawAndroid
is `preprocessed: true`, presigned with `/aosp/keys/aohp-apps.jks`). Lineage's default dev platform key is
byte-identical to the AOHP tree's, so AOHPDriver keeps its platform signature without any key override.

## Build (exact commands from logs/build-3.sh and logs/build-dodge-1.sh)

```bash
cd ~/lineage/android
export WITH_ADB_INSECURE=true      # ro.adb.secure=0, ro.debuggable=1 (dev phone: adb without RSA prompt, also in recovery)
source build/envsetup.sh
breakfast oriole                   # Pixel 6: = lunch lineage_oriole-bp4a-userdebug + build_kernel (repo-inits/syncs android_kernel_google_gs-6.1_manifest into out-kernel/, Kleaf build, ~10 min)
m -j10 bacon                       # -> out/target/product/oriole/lineage-23.2-*-UNOFFICIAL-oriole.zip, boot.img, dtbo.img, vendor_boot.img
# OnePlus 13 (kernel built in-tree from kernel/oneplus/sm8750, INLINE_KERNEL_BUILDING):
breakfast dodge                    # lineage_dodge-bp4a-userdebug
m -j10 bacon                       # -> out/target/product/dodge/lineage-23.2-*-UNOFFICIAL-dodge.zip, boot/dtbo/init_boot/vbmeta/vendor_boot/recovery.img
```

`brunch oriole` / `brunch dodge` is the one-step equivalent of breakfast + `m bacon`. Flashing: fastboot the
boot images, Lineage recovery → Factory reset → Apply update from ADB → `adb sideload <zip>` (REPORT-flash-2.md,
REPORT-dodge-build.md §6). Build time on chex (32 threads, -j10, CPUQuota 1000 %): oriole ~2.5 h cold, dodge 1 h 53 min.
