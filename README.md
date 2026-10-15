# OrangeFox device tree for alioth (Xiaomi Mi 11X / Redmi K40 / POCO F3)
This tree is for building OrangeFox Recovery for the _alioth_ devices

The Xiaomi Mi 11X (codenamed _"aliothin"_), Redmi K40 / POCO F3 (codenamed _"alioth"_) is a mid range smartphone from Xiaomi.

# Device specifications

| Device       | Xiaomi Mi 11X / Redmi K40 / POCO F3                        |
| -----------: | :----------------------------------------------------------|
| SoC          | Qualcomm SM8250 Snapdragon 870 5G                          |
| CPU          | 8x Qualcomm® Kryo™ 585 up to 2.84GHz                       |
| GPU          | Adreno 650                                                 |
| Memory       | 8GB / 6GB  (LPDDR 5)                                       |
| Shipped Android version | 11                    			    |
| Storage      | 128GB  (UFS 3.1)                                           |
| Battery      | Li-Po 4520 mAh, non-removable                              |
| Dimensions   | 163.7 x 76.4 x 7.8 mm                                      |
| Display      | 1080 x 2400 (20:9), 6.67 inches                            |

## Features

**Works**

- Booting
- **Decryption** (Android 12+)
- ADB
- MTP
- OTG
- vA/B partition functions
- Vibration

The Mi 11X / Redmi K40 / POCO F3 devices use a Virtual A/B Partition Scheme!

## Building

You can find a full compile guide for OrangeFox [Here](https://wiki.orangefox.tech/en/dev/building)

_Lunch_ command :
```
lunch twrp_alioth-eng && mka adbd bootimage
```

## High-Speed Flash (BETA)

This tree ships the High-Speed Flash feature ported from
[khargosxh18's peridot tree](https://github.com/khargosxh18/twrrp_device_tree_peridot)
(engine lives in the recovery repo; this tree provides the patched
`payload-dumper-go` at `recovery/root/system/bin/payload-dumper-go`).

When the Install-page checkbox **"High-speed flash (BETA)"** is ticked, full
OTA payloads (MIUI / HyperOS ROMs) are streamed **directly to the block
devices of the inactive slot** by the dumper itself — no intermediate
extraction to RAM/disk — followed by an atomic rebuild of the Super group.
Slot-select boot partitions are written directly; firmware partitions
(`xbl`, `abl`, `modem`, `bluetooth`, `dsp`, ...) and alioth's `logo`/`cust`
are excluded from the direct-write path.

* The checkbox is **off by default** (safety first).
* Any probe or write failure falls back to the **normal A/B install** —
  worst case is the target slot needs re-flashing; the bootloader chain is
  never touched.
* Requires a full (non-delta) OTA payload zip with a `payload.bin`.

### Taking logs when something goes wrong

The main log is `/tmp/recovery.log`. It lives in RAM — **pull it before
rebooting**, since a reboot (or re-`fastboot boot`) wipes it.

```bash
# watch live while flashing (second terminal)
adb shell tail -f /tmp/recovery.log

# after the flash — success or failure
adb pull /tmp/recovery.log
adb shell dmesg > dmesg.log        # kernel-side, if block writes failed

# quick filter for the relevant lines
grep -E "high-speed|hsf_links|logical|Super|payload" recovery.log
```

**Healthy flow** looks like:

```
high-speed probe: payload declares group 'qti_dynamic_partitions' with size ...
high-speed: booted slot X, writing slot Y
high-speed: '<partition>' -> /tmp/hsf_links/...
```

**Failure lines and what they mean:** any of these means the safety guards
worked and the install fell back to the normal A/B path:

| Log line | Meaning |
|---|---|
| `payload.bin not found in zip` | Not an A/B OTA zip (e.g. recovery-flashable zip) |
| `delta payload` | Delta OTAs unsupported — full payloads only |
| `Refusing to modify a logical partition while a snapshot update is active` | Virtual A/B merge pending — safety guard |
| `not a slot-select partition / refusing to guess` | Ambiguous partition mapping — probe refused |
| `target node resolves to the booted slot's node, refusing` | Guard against writing the slot booted from |
| `Not enough free space in logical group / reclaim` | Super resize path ran — check commit vs restore |

**Actual write failures** (the interesting ones — the dumper's own error
line appears just above these in the log):

```
high-speed: dumper CRASHED (killed by signal...)
high-speed: dumper failed (wait=...)
high-speed: dumper printed error keywords but exited with 0. Aborting.
high-speed: partition 'X' only reached NN% before the dumper exited
high-speed: couldn't flush '...' (...)
```

If the ROM won't boot after a failed flash: just boot the recovery again
with `fastboot boot` and re-flash normally with the toggle off. The
bootloader chain is never touched by any of this.

## Credits
- [Original alioth tree by Nebrassy](https://github.com/TeamWin/android_device_xiaomi_alioth)
- [Kernel](https://github.com/AOSPA/android_kernel_xiaomi_sm8250.git)

### Copyright
 ```
  /*
  *  Copyright (C) 2022-2024 The OrangeFox Recovery Project
  *
  * This program is free software: you can redistribute it and/or modify
  * it under the terms of the GNU General Public License as published by
  * the Free Software Foundation, either version 3 of the License, or
  * (at your option) any later version.
  *
  * This program is distributed in the hope that it will be useful,
  * but WITHOUT ANY WARRANTY; without even the implied warranty of
  * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  * GNU General Public License for more details.
  *
  * You should have received a copy of the GNU General Public License
  * along with this program.  If not, see <http://www.gnu.org/licenses/>.
  *
  */
  ```
