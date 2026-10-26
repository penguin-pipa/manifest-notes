# PenguinOS for Xiaomi Pad 6 (`pipa`)

Local manifest overlay for Project PenguinOS `celerity` (AOSPA `calcite` / Android 17).

## Repositories

This overlay replaces the repositories that contain pipa-specific changes:

| Path | Repository | Base |
| --- | --- | --- |
| `vendor/aospa` | `penguin-pipa/android_vendor_aospa` | PenguinOS `celerity` |
| `hardware/interfaces` | `penguin-pipa/android_hardware_interfaces` | `Project-PenguinOS/hardware_interfaces` |
| `system/core` | `penguin-pipa/system_core` | `Project-PenguinOS/system_core` |
| `frameworks/base` | `penguin-pipa/frameworks_base` | PenguinOS `celerity` |
| `frameworks/av` | `penguin-pipa/frameworks_av` | `Project-PenguinOS/frameworks_av` |
| `packages/apps/ThemePicker` | `penguin-pipa/packages_apps_ThemePicker` | `Project-PenguinOS/packages_apps_ThemePicker` |

The forks carry only the pipa-relevant deltas from YumeMichi's `aospa-pipa`
tree: lid state API and telephony null guards (`frameworks/base`), OGG
metadata with attached streams (`frameworks/av`), `host_init_verifier` fix
for overriding services (`system/core`), `kernel_config_t_4.19` in FCM 7
(`hardware/interfaces`), font refresh after overlay (`ThemePicker`), lid/UVC
sepolicy, sysconfig and `qti-telephony-utils` (`vendor/aospa`). Since 2026-10-26 each upstream commit is replayed
individually, so author and date match `aospa-pipa` and every imported
commit keeps its `(cherry picked from commit …)` line.

Current state of the `Project-PenguinOS` repos (checked 2026-10-26):
`android_vendor_aospa` is public again and the `vendor/aospa` fork has been
rebased on its new head `1c51484a`. The only repo still unreachable is
`packages_apps_ParanoidSense`; `vendor_qcom_opensource_power` was renamed to
`android_vendor_qcom-opensource_power`, so the overlay removes the stale
`vendor/qcom/opensource/power` path and barista re-declares it from
`aospa-pipa` at lunch. `ParanoidSense` is listed in `PRODUCT_PACKAGES` of
`aospa-target.mk` and has no public mirror, so it blocks a from-scratch sync
and build until it is published again. The other forks are aligned to the
current upstream heads (`frameworks/base` `de30bab9`, `frameworks/av`
`8607414`, `system/core` `666c84ed`, `hardware/interfaces` `da9d966`,
`packages/apps/ThemePicker` `e1fb80e`).

Everything else is synced by `vendor/aospa/products/pipa/beans.xml` directly
from YumeMichi's trees:

- `YumeMichi/device_xiaomi_pipa`, `YumeMichi/vendor_xiaomi_pipa`,
  `YumeMichi/kernel_xiaomi_pipa`
- `aospa-pipa/android_kernel_configs`
- `aospa-pipa/android_hardware_xiaomi`
- `aospa-pipa/android_hardware_dolby`
- `aospa-pipa/android_device_qcom_common`
- `aospa-pipa/android_device_qcom_sepolicy_vndr`
- `aospa-pipa/android_kernel_build`
- `aospa-pipa/android_vendor_qcom-opensource_power`
- `aospa-pipa/android_vendor_qcom-opensource_usb`
- `aospa-pipa/android_packages_apps_FlipFlap`
- `aospa-pipa/android_hardware_qcom_audio` (`calcite-865`)
- `aospa-pipa/android_hardware_qcom_display` (`calcite-865`)
- `aospa-pipa/android_hardware_qcom_media` (`calcite-865`)

`packages/apps/LMOFreeform` is replaced in this overlay by
`aospa-pipa/android_packages_apps_LMOFreeform` (`calcite`), cloned directly
rather than forked.

The three `calcite-865` HALs are mapped by
`vendor/aospa/products/pipa/kona.xml`; they are not forked into this org.

## Build

```bash
repo init -u https://github.com/Project-PenguinOS/manifest -b celerity
mkdir -p .repo/local_manifests
curl -L https://raw.githubusercontent.com/penguin-pipa/manifest/celerity/pipa.xml \
    -o .repo/local_manifests/pipa.xml
repo sync -j"$(nproc)"

./rom-build.sh pipa
```

In this setup the synced AOSP/PenguinOS tree and the build outputs live on
the project's build host under `~/penguinos`; this repository only carries
the manifest overlay. Run the build commands above from that tree.

`rom-build.sh` runs `lunch aospa_pipa-userdebug` and `m otapackage`, which
produces only the OTA package (`$OUT/PenguinOS-celerity-<date>[-gms]-pipa.zip`,
a `payload.bin` OTA that is not flashable with `fastboot`). Pass
`-z`/`--imgzip` to also generate the fastboot package. `lunch` automatically
invokes `vendor/aospa/build/tools/barista.py`, which creates
`.repo/local_manifests/baristablend.xml` and syncs the pipa device tree,
YumeMichi's device-common forks and the `calcite-865` HALs. Do not invoke
barista manually in the normal build flow.

Barista needs `vendor/aospa` at commit `46819e00` or newer: older copies
filter the sync list with `repo list --path-only`, which on repo v2.x drops
every project that is not checked out yet, so the device tree is never
downloaded.

### Variants

```bash
./rom-build.sh pipa              # GMS by default
./rom-build.sh pipa -l           # vanilla, without GMS
./rom-build.sh pipa -g           # GMS, explicit
./rom-build.sh pipa -z           # OTA + fastboot image zip
./rom-build.sh pipa -j"$(nproc)" # explicit job count
./rom-build.sh pipa -r           # repo sync before building
```

### Signing keys (release builds)

The default build uses the AOSP test keys. For a release build, generate a full
4096-bit key set (platform + one key per APEX) on the build host **after a first
build**, because the APEX list comes from `out/`:

```bash
mkdir -p ~/.android-certs
cp development/tools/make_key ~/.android-certs/
sed -i 's|2048|4096|g' ~/.android-certs/make_key

subject='/C=US/ST=State/L=City/O=Org/OU=Org/CN=Org/emailAddress=you@example.com'
for key in releasekey platform shared media networkstack sdk_sandbox bluetooth verity; do
    ~/.android-certs/make_key ~/.android-certs/$key "$subject"
done

for apex in $(find out/target/product/pipa -name "*.apex" \
        -exec basename {} .apex \; | sort -u); do
    ~/.android-certs/make_key ~/.android-certs/$apex "$subject"
    openssl pkcs8 -in ~/.android-certs/$apex.pk8 -inform DER -nocrypt \
        -out ~/.android-certs/$apex.pem
done
```

Then set `PRODUCT_DEFAULT_DEV_CERTIFICATE` to the platform key, e.g.
`PRODUCT_DEFAULT_DEV_CERTIFICATE := $(HOME)/.android-certs/releasekey`, locally in
the product makefile or in the environment (do not commit it). Soong looks for
the APEX `.pem` files in the same directory as that certificate
(`ApexKeyDir`). Rebuild the product cleanly afterwards (`rm -rf
out/target/product/pipa` or `make clobber`): Soong does not always notice a key
replacement.

Keep `~/.android-certs` offline and out of the repository. Losing these keys
means you cannot sign updates compatible with already distributed builds.

### Packages

Without flags, the build produces only the OTA package:

```text
out/target/product/pipa/PenguinOS-celerity-<date>[-gms]-pipa.zip
```

With `-z` the script also runs `img_from_target_files` and writes both
packages in the repository root:

```text
PenguinOS-celerity-<date>[-gms]-pipa.zip        # OTA
PenguinOS-celerity-<date>[-gms]-pipa-image.zip  # fastboot
```

The fastboot package contains `boot`, `dtbo`, `vendor_boot`, `vbmeta`,
`vbmeta_system`, `super_empty` and the logical partition images. Flash it
with:

```bash
fastboot -w update PenguinOS-celerity-<date>[-gms]-pipa-image.zip
```

`-w` runs the `if-wipe` erases declared in `fastboot-info.txt` and is
required when coming from another ROM.

### Updates

For updates, run `repo sync` before the build (or use
`./rom-build.sh pipa -r`) so direct repositories that are already checked out
are refreshed. Barista only syncs new or changed manifest entries:

```bash
repo sync -j"$(nproc)"
./rom-build.sh pipa
```

If YumeMichi force-pushes, use `repo sync --force-sync -j"$(nproc)"` instead.

### Validation

A manual barista dry-run is only useful for validation, never in the normal
build flow:

```bash
python3 vendor/aospa/build/tools/barista.py aospa_pipa true
cat .repo/local_manifests/baristablend.xml
```

After a sync, these checks must all hold:

```bash
ls device/xiaomi/pipa/device.mk
ls vendor/qcom/build/tasks/generate_extra_images.mk \
   vendor/qcom/build/tasks/kernel_definitions.mk
repo list --path-only --all | grep -x "device/xiaomi/pipa"
grep -n -A5 'framework_compatibility_matrix.7.xml' \
    hardware/interfaces/compatibility_matrices/Android.bp
```

The last check must show `kernel_config_t_4.19` next to `t_5.10` and `t_5.15`.

The default launcher is PenguinOS `Launcher3` (`Launcher3QuickStep`). Pixel
Launcher is not part of this product.
