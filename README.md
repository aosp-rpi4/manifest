# RPi4 AAOS — init, sync & build

Android Automotive (AAOS) head-unit BSP for the **Raspberry Pi 4 Model B**.
`rpi4.xml` is a `repo` local manifest that layers the BSP from
[github.com/aosp-rpi4](https://github.com/aosp-rpi4) on top of an AOSP checkout:

| Path | Repo | Branch |
|------|------|--------|
| `device/rpi/rpi4`     | `aosp-rpi4/device_rpi_rpi4`     | `main` |
| `kernel/rpi/rpi4`     | `aosp-rpi4/kernel_rpi_rpi4`     | `rpi4-aaos` |
| `external/mesa3d-v3d` | `aosp-rpi4/external_mesa3d-v3d` | `rpi4-aaos` |
| `external/minigbm`    | `aosp-rpi4/external_minigbm`    | `rpi4-aaos` (replaces stock AOSP minigbm) |
| `vendor/rpi-firmware` | `raspberrypi/firmware`          | `master` |

Products: **`aosp_rpi4_car`** (Automotive) and **`aosp_rpi4`** (handheld).

---

## 0. Host prerequisites

```bash
sudo apt install -y \
    git curl python3 repo \
    gcc-aarch64-linux-gnu g++-aarch64-linux-gnu \
    bc bison flex libssl-dev make \
    android-tools-fsutils simg2img img2simg \
    parted dosfstools e2fsprogs picocom
```

---

## 1. Initialize the tree

Pick the AOSP base. **For a true Android 16 (API 36) build, use a finalized
release tag** (list them first):

```bash
git ls-remote --tags https://android.googlesource.com/platform/manifest | grep -E 'android-16\.0\.0_r[0-9]+$'

mkdir -p ~/android/aosp && cd ~/android/aosp
repo init -u https://android.googlesource.com/platform/manifest -b android-16.0.0_r1
```

> Note: building off `main` / `trunk_staging` instead produces a pre-finalization
> build that reports as "Baklava"/15 with SDK 35. The `android-16.0.0_rN` tag is
> SDK 36 and reports as **Android 16**.

## 2. Add this local manifest

```bash
mkdir -p .repo/local_manifests
cp /path/to/rpi4.xml .repo/local_manifests/rpi4.xml
# or fetch it from the manifest repo:
# curl -L https://raw.githubusercontent.com/aosp-rpi4/manifest/main/rpi4.xml -o .repo/local_manifests/rpi4.xml
```

## 3. Sync

```bash
repo sync -c -j"$(nproc)" --no-tags
```

This pulls AOSP **plus** the device tree, kernel, Mesa-v3d, minigbm fork, and Pi
firmware — everything needed to build.

---

## 4. Build the kernel

The device tree ships a prebuilt kernel under `device/rpi/rpi4/kernel/`, so you
can **skip this step** unless you changed the kernel. To (re)build it:

```bash
cd kernel/rpi/rpi4
export ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-

make bcm2711_defconfig                                              # full RPi4 HW config
scripts/kconfig/merge_config.sh -m .config arch/arm64/configs/bcm2711_android_defconfig
make olddefconfig
make -j"$(nproc)" Image.gz dtbs

cd ../../..
cp kernel/rpi/rpi4/arch/arm64/boot/Image.gz                         device/rpi/rpi4/kernel/Image.gz
cp kernel/rpi/rpi4/arch/arm64/boot/dts/broadcom/bcm2711-rpi-4-b.dtb device/rpi/rpi4/kernel/bcm2711-rpi-4-b.dtb
```

> The android defconfig is a **fragment** — you must merge it onto
> `bcm2711_defconfig`. Running the android defconfig alone loses the UART/GPU/SD
> drivers and gives a dead board.

---

## 5. Build Android

```bash
cd ~/android/aosp
source build/envsetup.sh

# Automotive head unit (the goal):
export TARGET_PRODUCT=aosp_rpi4_car
export TARGET_RELEASE=trunk_staging          # use the matching release for your base tag
export TARGET_BUILD_VARIANT=userdebug

or 

lunch aosp_rpi4_car-trunk_staging-userdebug
m

# (Handheld variant instead: TARGET_PRODUCT=aosp_rpi4)
```

Produces `out/target/product/rpi4/{system,vendor,product,userdata}.img`.

---

## 6. Package boot + flash the SD card

```bash
bash device/rpi/rpi4/rpi4_boot_package.sh        # assembles the FAT boot partition
sudo bash flash_rpi4.sh /dev/sdX                 # replace sdX with your card (ERASES it)
```

`flash_rpi4.sh` reformats **all** partitions including `/data`, so every flash is
a clean first boot. Always re-run **both** package + flash after a kernel/ramdisk
change.

---

## 7. First boot

```bash
picocom -b 115200 /dev/ttyUSB0                   # serial console (GPIO14/15 + GND)
```

First boot decompresses APEXes and runs dexopt — give it a few minutes. Then:

```bash
adb shell getprop sys.boot_completed             # -> 1
adb shell dumpsys activity activities | grep ResumedActivity   # -> CarLauncher
```

---

## More

Full bring-up history, the SD partition layout, kernel config rationale, and a
troubleshooting log (graphics, ashmem, audio, HSUM, Bluetooth/USB features,
RescueParty, serial-log muting) are in **`device/rpi/rpi4/README.md`**.
