# degas-kernel — prebuilt kernel for the Xiaomi 14T (degas)

Prebuilt kernel, loadable modules and base device tree for the Xiaomi 14T
(`degas`, MediaTek MT6897 / Dimensity 8300-Ultra), used to build LineageOS 23
for this device.

**Nothing is compiled here.** This repository holds binaries taken from the
device's own stock firmware, plus one device-tree change of our own that is
also provided in source form (see [Modifications](#modifications-made-here)).

## Contents

| Path | What it is |
|---|---|
| `Image.lz4` | Kernel `6.1.138-android14-11-g44bda9e8f6e9-ab13792638`, built 2025-07-16 by Google's GKI build `ab13792638`. `sha256 618d7723…6191` |
| `modules/` | 492 loadable modules (`.ko`) |
| `dtb/mt6897.dtb` | Generic MT6897 SoC device tree — **modified here**. `sha256 d53811e3…15ba` |
| `dts/` | Device-tree source: the full source form of the shipped `.dtb`, plus our changes as patches |
| `kernel-uapi-headers.tar.gz` | UAPI headers matching that kernel |

The device-specific device tree is **not** here: on MediaTek platforms it lives
in the `dtbo` partition, which this tree does not build or replace, so the
device keeps its stock `dtbo` and it is overlaid onto `mt6897.dtb` at boot.

## Provenance

Everything binary in this repository comes from the stock HyperOS firmware for
degas, build **OS3.0.4.0.WNEMIXM** (global, Android 16). This is verifiable
against Xiaomi's own fastboot package:

* `Image.lz4` is **byte-identical** to the kernel inside the stock `boot.img`:

  ```sh
  # kernel size is a LE u32 at offset 8; payload starts at page 1 (4096)
  ksize=$(od -An -tu4 -j8 -N4 boot.img | tr -d ' ')
  dd if=boot.img of=stock-kernel.bin bs=4096 skip=1 count=$(( (ksize+4095)/4096 )) 2>/dev/null
  head -c "$ksize" stock-kernel.bin | cmp - Image.lz4   # no output = identical
  ```

* Of the 492 modules, **301 are byte-identical** to the ones in the stock
  `vendor_dlkm` and `system_dlkm` images of that build, and the remaining
  **191 are byte-identical** to the modules in the stock `vendor_boot`
  ramdisk (they are exactly the `vendor_ramdisk.modules.load` set). **None was
  built, patched or re-signed here** — there is no kernel build infrastructure
  in this repository at all.

* `dtb/mt6897.dtb` began as the device tree from the stock `vendor_boot` image
  and carries two changes of ours, described below.

## License

The kernel, the modules and the device tree are licensed under the **GNU
General Public License, version 2** (see `LICENSE`). Copyright is held by the
Linux kernel contributors, MediaTek Inc., Xiaomi Inc. and Google LLC. Nothing
in this repository is licensed by us; we redistribute unmodified vendor
binaries and one modified device tree, all under the GPL-2.0 they already
carry.

### Where the corresponding source is

Xiaomi publishes part of it, on branch `bsp-degas-u-oss`:

* [`MiCode/Xiaomi_Kernel_OpenSource`](https://github.com/MiCode/Xiaomi_Kernel_OpenSource) — the GKI core
* [`MiCode/MTK_kernel_modules`](https://github.com/MiCode/MTK_kernel_modules) — connectivity (wlan `gen4m`, GPS, BT)
* [`MiCode/kernel_build`](https://github.com/MiCode/kernel_build) — the kleaf/bazel build harness

The GKI binary itself is Google's: kernel `android14-6.1`, build
`ab13792638`, source at
[android.googlesource.com/kernel/common](https://android.googlesource.com/kernel/common/+/refs/heads/android14-6.1).

### What Xiaomi has not published

`MiCode/MTK_kernel_device_modules` has **no degas branch**. The MediaTek
`mediatek_v2` DRM stack, `mi_disp` and the degas panel drivers therefore have
no published source anywhere. Those modules are redistributed here exactly as
Xiaomi shipped them on the device; we hold no source for them and cannot
provide what the copyright holder has not released. If you need it, it has to
be requested from Xiaomi Inc., who distributed the binaries first.

### Written offer

For any binary in this repository that we hold corresponding source for, we
will provide that source — open an issue and ask. For the only file we have
modified, the source is already in this repository, in `dts/`.

## Modifications made here

Exactly one file has been modified: `dtb/mt6897.dtb`, in two commits, both done
as a `dtc` round-trip on the stock device tree:

1. **GPU thermal backstop** — thermal zones `gpu1`/`gpu2` shipped with a single
   critical trip at 119 °C and no cooling maps, because Xiaomi delegates GPU
   thermal policy to `mi_thermald`, which does not run on LineageOS. Adds a
   passive trip at 105 °C mapped onto the Mali cooling device.
2. **CPU thermal backstop** — adds `#cooling-cells` to the three cluster
   leaders (`cpu0`, `cpu4`, `cpu7`) so `mediatek-cpufreq-hw` registers cpufreq
   cooling devices, and a passive trip at 100 °C per hot-core zone. Without it
   nothing regulated the CPU between 88 °C under load and the 119 °C emergency
   shutdown.

Both keep `polling-delay = 0`, leave `__symbols__` byte-identical and move no
pre-existing phandle — which matters, because the stock `dtbo` is overlaid onto
this device tree at boot and resolves against them.

### Source form, and how to verify it

`dts/mt6897.dts` is the complete device-tree source of the shipped `.dtb`, and
`dts/patches/` holds our two changes as patches against the stock base.

```sh
# the shipped binary is reproducible from the source in this repo, byte for byte
dtc -I dts -O dtb -f -o /tmp/mt6897.dtb dts/mt6897.dts   # dtc 1.8.1
cmp /tmp/mt6897.dtb dtb/mt6897.dtb                       # no output = identical

# and reverse-applying both patches gives back the unmodified stock device tree
cp dts/mt6897.dts /tmp/stock.dts
patch -R -p1 /tmp/stock.dts < dts/patches/0002-cpu-thermal-backstop.patch
patch -R -p1 /tmp/stock.dts < dts/patches/0001-gpu-thermal-backstop.patch
```

## No warranty

This is an unofficial, community repository. It is not endorsed by, and carries
no warranty from, Xiaomi Inc., MediaTek Inc., Google LLC or the LineageOS
project. Flashing software built with it can render a device unbootable; you do
so at your own risk. See sections 11 and 12 of the GPL-2.0 in `LICENSE`.

If you are a rights holder and believe something here should not be
redistributed, open an issue and it will be removed.
