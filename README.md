# W1700K U-Boot (AN7581 HTTP Recovery)

> **Based on** [naoki66](https://github.com/naoki66/XR1710G-http-uboot) / [YYH2913](https://github.com/YYH2913/http-uboot)

## W1700K support

The same AN7581 platform family also includes the Gemtek W1700K (CenturyLink /
Lumen / Quantum Fiber). This repository builds HTTP-recovery U-Boot for both
boards from the same tree:

- `make an7581_w1700k_defconfig` — Gemtek W1700K (default)
- `make xr1710g_defconfig` — Brightspeed XR1710G

The W1700K build shares the recovery stack (lwIP HTTP server, RTL8261 10G
training, chainloader packaging) and only swaps the board device tree, default
environment and the recovery LAN LED GPIO mapping. CI (`build.yml`) accepts a
`board` input (`xr1710g` or `w1700k`) and names the artifacts
`<board>-uboot-<date>-g<commit>-flash-slot.bin` / `-chainloader.itb`.

Companion firmware: [ImmortalWrt-for-Gemtek-XR1710G](https://github.com/naoki66/ImmortalWrt-for-Gemtek-XR1710G)
(currently configured to build the `gemtek_w1700k-ubi` image).

## 捐赠 / Donate

如果您觉得对您有帮助，请我喝杯咖啡吧，让我更有动力持续更新。

<img src="./c6ea388c976395326514814f80d512d5.png" width="220" alt="Donate / 捐赠二维码">

## Fork Changes

This fork is based on YYH2913's XR1710G http-uboot and adds the following modifications:

### 1. DHCP Server Fix — Windows 11 Compatibility

**Problem**: The built-in DHCP server did not include the Router (Option 3, default gateway) and DNS Server (Option 6) options in DHCP Offer/ACK responses. Windows 11's DHCP client has stricter validation and would ignore the incomplete response, leaving the PC without an IP address. Users had to manually set `192.168.255.x` to access the recovery page.

**Fix**: Added Router and DNS options to `recovery_dhcp_send_reply()` in `net/lwip/httpd_recovery.c`. Both values default to the server IP (`192.168.255.1`).

### 2. GitHub Actions CI — Automated Build & Release

Added `.github/workflows/build.yml` for on-demand compilation and release publishing:
- Manual trigger only (`workflow_dispatch`)
- Cross-compiles with `aarch64-linux-gnu-` toolchain
- Automatically packages the chainloader slot image (`*-flash-slot.bin`)
- Publishes build artifacts to GitHub Releases with SHA256 checksums
- Releases are available at [Actions](https://github.com/naoki66/XR1710G-http-uboot/actions) page

### 3. Local Debug Mock Server

Added `mock_server.py` for previewing and debugging the HTTP recovery UI locally without hardware.

### 4. Recovery Entry Window with LED Cue

**Problem**: recovery could only be entered by holding `reset` while the
chainloader started. Nothing told the user when to press the button, and
`board_late_init()` sampled the button only once before the (slow) NAND/UBI
work, so the usable window was a few hundred milliseconds wide.

**Design**: the chainloader now opens a visible recovery window before any
NAND/UBI work:

1. The status LEDs chase through their colour channels - one channel lit at a
   time, 200 ms per step (W1700K RGBW status LED: red → green → blue → white).
2. The reset button is polled every 20 ms for the whole window (default
   3000 ms, see `recovery_window_ms` below).
3. Pressing it once starts the web recovery immediately, and the status LED
   turns blue at that very moment - before the web server is reachable.
   While recovery is idle it stays a steady blue, and it switches to the
   colour sweep while an image is being erased or written.
4. No press → the LEDs flash three times ("window missed") and the normal
   boot continues.

**Tuning and fallbacks**

- `recovery_window_ms` (U-Boot env): window length in ms, default `3000`.
  Setting `0` disables the cue and restores the previous single-sample
  behaviour.
- `recovery_trigger` (one-shot, set by the LuCI recovery app) still enters
  recovery directly, without the window.
- Boards where no status LEDs can be claimed fall back to the legacy
  single button sample.

**Implementation**: `xr1710g_recovery_button_window()` in
`board/airoha/an7581/an7581_rfb.c`; LED helpers
`recovery_status_leds_cue_begin()/poll()/end()` in `net/lwip/httpd_recovery.c`.

### 5. Auto-mode Recovery TX Egress Fallback

**Problem**: with `recovery_port` unset (`auto` / dual-service) the recovery
always used the internal 1G switch port (fport 1) as the default TX egress,
even when no 1G port had link. On the W1700K a client attached to the 2.5G LAN
port (GDM4, fport 4) therefore received nothing - RX worked (DHCP DISCOVER was
even answered), but every unicast reply (ARP, HTTP) was transmitted into the
idle switch port, so the recovery page stayed unreachable.

**Fix**: in `airoha_pick_tx_fport()`, when dual-service is enabled and no 1G
port has link, fall back to the external 10G/2.5G port when it is a candidate.
The dedicated 1G ports keep priority whenever they have link, so the original
behaviour is unchanged for XR1710G-style 1G-first setups.

## 中文说明（Fork Changes 中文对照）

### 4. 恢复入口窗口与 LED 提示

**问题**：以前只能"启动时按住 reset"进恢复模式，没有任何视觉提示，用户不知道
什么时候按、有没有按上；而且 `board_late_init()` 只在耗时的 NAND/UBI 操作之前
采样一次按键，真正可用的窗口只有几百毫秒。

**设计**：chainloader 在做任何 NAND/UBI 操作之前先开一个看得见的恢复窗口：

1. 状态灯按颜色通道依次跑马灯，每 200 ms 换一个通道（W1700K 是 RGBW 状态灯：
   红 → 绿 → 蓝 → 白）。
2. 整个窗口内每 20 ms 采样一次 reset 键（窗口默认 3000 ms，可用
   `recovery_window_ms` 调整，单位毫秒、十进制）。
3. 窗口内按一下 reset 就立刻进入网页恢复，**按下瞬间状态灯就变蓝**（此时网页
   服务还没起来）；空闲时维持常亮蓝，擦写阶段切回扫动进度动画。
4. 窗口内没按 → 状态灯快速闪 3 次（提示"错过了窗口"），然后继续正常启动。

操作方式：上电 → 看到状态灯开始 RGB 跑马灯 → 按一次 `reset` → 灯变成常亮蓝
＝ 已进入恢复模式，浏览器打开 `http://192.168.255.1/`。

**其他入口与回退**

- `recovery_trigger`（一次性变量，由 LuCI 恢复插件写入）仍然可以直接进入恢复，
  不走这个窗口。
- 采不到状态灯的板子自动回退到原来的"单次采样"行为。

**实现位置**：`xr1710g_recovery_button_window()`
（`board/airoha/an7581/an7581_rfb.c`）、LED 辅助函数
`recovery_status_leds_cue_begin()/poll()/end()`（`net/lwip/httpd_recovery.c`）。

### 5. auto 模式恢复发送出口回退

**问题**：`recovery_port` 未设置（auto / dual-service）时，恢复模式总是把内部
1G 交换机口（fport 1）当作默认发送出口，哪怕 1G 口根本没有链路。W1700K 上如果
客户端接在 2.5G LAN 口（GDM4，fport 4），收包是正常的（DHCP DISCOVER 都能收到
并回应），但所有单播回应（ARP、HTTP）都被发进空闲的交换机口，恢复页面因此
始终打不开。

**修复**：在 `airoha_pick_tx_fport()` 中，dual-service 且 1G 口无链路时，回退到
外部 10G/2.5G 口（前提是它被判定为候选）。1G 口有链路时仍然优先，所以
XR1710G 那种"1G 优先"的行为保持不变。

---

## Original README

The following is the original README by YYH2913, preserved in full.

---

# XR1710G U-Boot

This is a customized U-Boot port for the XR1710G. Its main features are:

- 10GbE support
- HTTP Recovery for convenient web-based firmware flashing
- Built-in DHCP server so a connected PC can obtain an address automatically in recovery mode
- Recovery page address: `http://192.168.255.1`

## Flashing Summary

Use this section as the short operational checklist. The detailed background
and rationale are folded below.

### 1. HTTP Recovery Web Flash

Use this after the custom U-Boot is installed and you only want to update the
main OpenWrt firmware.

1. Connect the PC to the 10GbE port and leave the PC NIC in DHCP mode.
2. Press and hold the `reset` button, wait a few seconds for the button to
   settle, and only then apply power (keep holding). This is the reliable way
   to enter recovery - pressing the button at the exact moment of power-on is
   timing-sensitive and may leave the vendor first-stage bootloader waiting
   with all LEDs off.
3. Release the button after the status LED changes from solid red to the
   flowing recovery pattern.
4. Open `http://192.168.255.1`.
5. Select the upload target:
   - `firmware` writes the OpenWrt-generated `*-sysupgrade.itb` to `ubi:fit`.
   - `uboot` writes `xr1710g-chainloader-slot.bin` to the raw chainloader slot.
6. For a firmware upload, select the layout encoded in that image's embedded
   device tree: `UBI 2.0`, `UBI 1.5`, or `UBI 1.0`. The selector controls the
   erase/rebuild boundary; it does not convert or patch the uploaded image.
7. Full rebuild recreates `ubootenv`, `ubootenv2`, `fit`, and `rootfs_data`.
   Factory EEPROM and MAC data are restored from the vendor DSD region, while
   saved U-Boot environment values are reset.
8. Do not select a layout based only on the currently detected flash layout.
    The selection must match the new image. A mismatch can let U-Boot load the
    kernel but leave Linux waiting indefinitely for `/dev/fit0`.


> Alternative timing: power on normally first, wait until the 10GbE port LED
> starts blinking (the chainloader has handed off to this U-Boot), then press
> and hold `reset`. This also works; the key is not to press the button at the
> instant of power-on.

Current HTTP Recovery page with the image-layout selector:

![Current HTTP Recovery page with UBI layout selector](./image.png)

To inspect an OpenWrt FIT before uploading it, extract its FDT and read the
`ubi` partition size:

```sh
u-boot/tools/dumpimage -T flat_dt -p 1 -o /tmp/xr1710g.dtb firmware.itb
dtc -I dtb -O dts /tmp/xr1710g.dtb | grep -A2 'label = "ubi"'
```

Choose the WebUI value that matches the second `reg` cell:

| Embedded FDT `ubi` reg | WebUI selection |
|---|---|
| `<0x00700000 0x1b700000>` | `UBI 2.0` |
| `<0x00700000 0x1d9c0000>` | `UBI 1.5` |
| `<0x00700000 0x1f700000>` | `UBI 1.0` |

For example, if Linux prints an `ubi` end address of `0x1e0c0000`, the image
is `UBI 1.5`; rebuild it with `UBI 1.5`, even if U-Boot previously detected or
created a 1.0 layout. The characteristic failure is `not enough PEBs` followed
by `Waiting for root device /dev/fit0`.

Latest verified multi-layout recovery build (`59060dde`, 2026-07-12):

- Flash this file through the `uboot` target:
  `../build-artifacts/xr1710g-20260712/xr1710g-uboot-v2026.07-59060dde-flash-slot.bin`
  (`908778` bytes,
  SHA256 `0d2277d578dcb05c004475c1f653d3078c1672182994eeb5ad17d37b0dc4023c`)
- Bare chainloader FIT for RAM-only validation:
  `../build-artifacts/xr1710g-20260712/xr1710g-uboot-v2026.07-59060dde-chainloader.itb`
  (`900330` bytes,
  SHA256 `c2cfa064ed631cd4494e8a9f150923eddd1950bb74bfbdd7d7a9bb4fcd28cf34`)

The artifact name contains the source commit used by the embedded U-Boot:
`U-Boot 2026.07-00762-g59060dde7b91`.

Do not upload `u-boot.bin`, `u-boot.img`, or `xr1710g-ubi.img` through HTTP
Recovery. Use the named `*-flash-slot.bin` artifact (or the generic build
output `xr1710g-chainloader-slot.bin`) only with the `uboot` target.

> [!WARNING]
> - Do not flash `u-boot.bin` directly.
> - Do not treat `u-boot.img` as the final flash image.
> - The latest verified U-Boot/chainloader image is
>   `../build-artifacts/xr1710g-20260712/xr1710g-uboot-v2026.07-59060dde-flash-slot.bin`.
> - For main OpenWrt firmware, HTTP Recovery uses `*-sysupgrade.itb`, while
>   low-level raw flashing uses `out/xr1710g-ubi.img`.

### 2. OpenWrt / Linux `nandwrite` on the Legacy Layout

Use this when the device is still running Linux with the original vendor
layout and the active slot is still named `tclinux`.

Copy the image to the router:

```sh
scp out/xr1710g-chainloader-slot.bin root@<router-ip>:/tmp/
```

On the router:

```sh
grep -E 'tclinux|tclinux_slave' /proc/mtd

# If the primary slot is /dev/mtd5, back up and replace only its first 1 MiB.
nanddump -l 0x100000 -f /tmp/tclinux-head-1m.bin /dev/mtd5
flash_erase /dev/mtd5 0 8
nandwrite -p /dev/mtd5 /tmp/xr1710g-chainloader-slot.bin

sync
reboot
```

Do not use `sysupgrade` for this U-Boot/chainloader write, and do not overwrite
the whole `64 MiB` `tclinux` slot.

### 3. ECNT Stock U-Boot Prompt

Use the stock ECNT / AXON prompt to verify environment and RAM boot behavior.

Expected boot-critical environment:

```sh
printenv bootcmd loadaddr fdt_high

setenv loadaddr 0x81800000
setenv fdt_high 0xac000000
setenv bootcmd 'flash read 0x602100 0x4000000 $loadaddr; bootm'
saveenv
```

The current slot image is dual-entry. The ECNT stock command above enters the
FIT at `0x602100`; older environments that read from `0x600000` and run
no-argument `bootm` enter a small legacy prefix shim instead.

RAM-only TFTP validation with the slot image:

```sh
setenv ipaddr 192.168.0.1
setenv serverip 192.168.0.205
tftpboot 0x81800000 xr1710g-chainloader-slot.bin
bootm 0x81802100
```

RAM-only TFTP validation with the bare FIT artifact:

```sh
setenv ipaddr 192.168.0.1
setenv serverip 192.168.0.205
tftpboot 0x81800000 xr1710g-chainloader.itb
bootm 0x81800000
```

### 4. Migration from `w1700k-ubi-installer`

If the first-stage boot command was changed by the older installer, fix it
before expecting the current `xr1710g-chainloader-slot.bin` to boot
automatically.

Check the current command:

```sh
printenv bootcmd
```

Use one of these compatible forms:

```sh
setenv bootcmd 'flash read 0x602100 0x100000 $loadaddr; bootm 0x81800000'
saveenv
```

or:

```sh
setenv bootcmd 'flash read 0x600000 0x100000 $loadaddr; bootm'
saveenv
```

or:

```sh
setenv bootcmd 'flash read 0x600000 0x100000 $loadaddr; bootm 0x81802100'
saveenv
```

Then use the OpenWrt/Linux `nandwrite` flow above or a known-good vendor update
path to write the actual image.

<details>
<summary>Flash image notes</summary>

## Flash Image Notes

- `out/u-boot.bin`
  The raw secondary U-Boot payload produced by the build.
- `u-boot.img`
  One of the standard U-Boot build artifacts, but not the final flash image for the XR1710G boot chain.
- `out/xr1710g-chainloader.itb`
  An intermediate FIT-packaged artifact. It is usually not written to flash directly.
- `out/xr1710g-chainloader-slot.bin`
  The image that is actually written to the `chainloader` partition.
- `../build-artifacts/xr1710g-YYYYMMDD/*-flash-slot.bin`
  Archived, commit-named copies of the verified flashable slot image. Prefer
  the exact latest path listed in the flashing summary above.

Upstream reference:

- OpenWrt PR `#22397`: <https://github.com/openwrt/openwrt/pull/22397>

> [!WARNING]
> - Do not flash `u-boot.bin` directly
> - Do not treat `u-boot.img` as the final flash image
> - The final image to flash is the packaged `xr1710g-chainloader-slot.bin`

</details>

<details>
<summary>ECNT stock U-Boot details</summary>

## Flash From ECNT Stock U-Boot

This section applies to units still using the original ECNT / AXON first-stage
U-Boot. The known stock environment reports:

```sh
ver=U-Boot 2014.04-rc1 (Mar 15 2024 - 15:13:21) AXON 1.6
version=1.6
vendor=ecnt
board=an7581_evb
loadaddr=0x81800000
fdt_high=0xac000000
bootcmd=flash read 0x602100 0x4000000 $loadaddr; bootm
ipaddr=192.168.0.1
serverip=192.168.0.205
bootfile=tclinux.bin
kernel_filename=tclinux.bin
uboot_filename=tcboot.bin
```

For this stock environment, keep the default `bootcmd` unchanged. The current
slot image is dual-entry:

- flash offset `0x600000` contains a small legacy bootm prefix shim
- flash offset `0x602100` contains the full chainloader FIT

The ECNT default command enters the FIT directly:

```sh
flash read 0x602100 0x4000000 $loadaddr
bootm
```

If the boot-critical variables do not match the values above, check only the
minimal set first:

```sh
printenv bootcmd loadaddr fdt_high
```

For the current chainloader slot image, the expected values are:

```sh
bootcmd=flash read 0x602100 0x4000000 $loadaddr; bootm
loadaddr=0x81800000
fdt_high=0xac000000
```

To change a mismatched ECNT first-stage environment back to these values:

```sh
setenv loadaddr 0x81800000
setenv fdt_high 0xac000000
setenv bootcmd 'flash read 0x602100 0x4000000 $loadaddr; bootm'
saveenv
reset
```

Do not copy a full `printenv` from another device. Board-specific values such
as `ethaddr`, `bootargs`, GPIO settings, serial data, and vendor/product fields
should be preserved unless there is a separate reason to change them.

The image to write is still:

```text
out/xr1710g-chainloader-slot.bin
```

The older `0x600000` no-argument form is also supported by the current
dual-entry slot image:

```sh
flash read 0x600000 0x100000 $loadaddr
bootm
```

In this mode, `bootm` starts the legacy prefix shim at `$loadaddr`; that shim
then locates the FIT at `$loadaddr + 0x2100` and jumps to the embedded
second-stage U-Boot payload.

After flashing the slot image, the ECNT prompt can be used to validate the
first-stage handoff manually:

```sh
printenv ver version vendor board bootcmd loadaddr fdt_high
flash read 0x600000 0x100 0x81800000
md.b 0x81800000 0x20
flash read 0x602100 0x4000000 $loadaddr
bootm
```

The expected magic bytes for a correctly flashed current slot image are:

- at flash offset `0x600000`: `27 05 19 56` (`IH_MAGIC`, legacy prefix shim)
- at flash offset `0x602100`: `d0 0d fe ed` (`FDT_MAGIC`, chainloader FIT)

If `bootcmd=flash read 0x602100 ...; bootm` prints `Wrong Image Format`, the
first thing to check is the second magic value above. If `0x602100` is not
`d0 0d fe ed`, the ECNT environment is not the cause; the chainloader slot was
not written with the current `out/xr1710g-chainloader-slot.bin`, or it was
written at the wrong offset.

If variable expansion is inconvenient, use the default address explicitly:

```sh
flash read 0x602100 0x4000000 0x81800000
bootm 0x81800000
```

The earlier bug where the log reached `Starting kernel ...` and then stopped
was observed on this same AXON 1.6 stock U-Boot line. In that failure mode the
shim could be entered from a FIT image reported at `0x89000000`, while the
control DTB still used the default `fit-base = 0x81800000`. The current shim
keeps `0x81800000` as the default and adds a fallback search for the AXON 1.6
handoff address, so the ECNT environment should not be changed just to work
around that bug.

The ECNT `flash` command is vendor-specific. This README intentionally does
not provide an unverified raw `flash write` command for the stock prompt; use a
known-good vendor update path or the OpenWrt `nandwrite` flow below for the
persistent write.

</details>

<details>
<summary>Migration from w1700k-ubi-installer details</summary>

## Migration From `w1700k-ubi-installer`

If the device is being switched from the older `w1700k-ubi-installer` path to
the current XR1710G chainloader layout, the first-stage vendor U-Boot
environment must be checked and usually updated.

The older installer commonly left the first-stage environment with:

```sh
bootcmd=flash read 0x600000 0x100000 $loadaddr; bootm
```

That command matched the older `openwrt-airoha-an7581-gemtek_w1700k-ubi-chainload-uboot.itb`
flow where the FIT image started directly at `0x600000`.

Older single-entry `xr1710g-chainloader-slot.bin` builds kept a vendor
`0x2100`-byte slot prefix and placed the FIT at `0x602100`, so the old
no-argument `bootm` form was not compatible. Current builds are dual-entry:
the slot starts with a legacy bootm prefix shim and still keeps the FIT at
`0x602100`.

Before expecting automatic boot to work, inspect the current setting in the
first-stage prompt:

```sh
printenv bootcmd
```

The following first-stage forms are confirmed to work with the current
`xr1710g-chainloader-slot.bin`:

```sh
flash read 0x600000 0x100000 0x81800000
bootm 0x81800000
```

or:

```sh
flash read 0x602100 0x100000 0x81800000
bootm 0x81800000
```

or:

```sh
flash read 0x600000 0x100000 0x81800000
bootm 0x81802100
```

To update the persistent first-stage environment, use one of these `bootcmd`
values:

```sh
setenv bootcmd 'flash read 0x600000 0x100000 $loadaddr; bootm'
saveenv
```

or:

```sh
setenv bootcmd 'flash read 0x602100 0x100000 $loadaddr; bootm 0x81800000'
saveenv
```

or:

```sh
setenv bootcmd 'flash read 0x600000 0x100000 $loadaddr; bootm 0x81802100'
saveenv
```

</details>

<details>
<summary>Current partition layout details</summary>

## Current Partition Layout

The three supported UBI layouts share the same `vendor` and `chainloader`
partitions. Only the UBI end boundary differs:

| Version | U-Boot MTD view | UBI start | UBI size | Reserved tail |
|---|---|---:|---:|---:|
| `UBI 2.0` | `ubi` | `0x00700000` | `0x1B700000` | `0x04200000` (528 PEB) |
| `UBI 1.5` | `ubi1.5` | `0x00700000` | `0x1D9C0000` | `0x01F40000` (250 PEB) |
| `UBI 1.0` | `ubi1.0` | `0x00700000` | `0x1F700000` | `0x00200000` (16 PEB) |

`vendor` is `0x00000000 + 0x00600000`; `chainloader` is
`0x00600000 + 0x00100000`. New builds default to `UBI 2.0`.

Inside `vendor`, the original vendor layout is still preserved:

- `bootloader`: `0x00000000-0x001FFFFF`
- `uenv`: `0x00200000-0x003FFFFF`
- `dsd`: `0x00400000-0x005FFFFF`

The `UBI 2.0` boundary follows the conservative 528-PEB reservation used by
the local OP integration. Migration between versions must fully rebuild UBI;
flashing only the `fit`/sysupgrade volume is not sufficient or safe. OpenWrt
PR `#22397` is still unmerged and currently carries the older 16-PEB boundary;
it is not evidence for the local `UBI 2.0` size.

The WebUI layout choice must match the partition size in the uploaded FIT's
embedded FDT, not merely this U-Boot's current MTD view. U-Boot can read a
`fit` volume from a larger layout while the Linux DTS exposes a smaller MTD
partition; Linux then rejects the UBI volume table because its reserved PEB
count exceeds the DTS-defined partition.

</details>

<details>
<summary>OpenWrt legacy tclinux flashing details</summary>

## Flashing the Primary `tclinux` Slot from OpenWrt

The following commands are intended for the case where:

- the device is still using the original legacy vendor partition layout
- there is not yet a dedicated `chainloader` partition
- you want to write `out/xr1710g-chainloader-slot.bin` into the primary `tclinux` slot

First, confirm that the partition names still match the old layout:

```sh
grep -E 'tclinux|tclinux_slave' /proc/mtd
```

If the primary slot is still `tclinux`, it is typically `/dev/mtd5`. It is recommended to overwrite only the first `1 MiB` of that slot. Do not use `sysupgrade`, and do not directly `mtd write ... tclinux` over the entire `64 MiB` slot.

Assuming the image has already been copied to `/tmp/xr1710g-chainloader-slot.bin` on the device, run:

```sh
# Optional: back up the first 1 MiB of the primary slot
nanddump -l 0x100000 -f /tmp/tclinux-head-1m.bin /dev/mtd5

# Erase the first 1 MiB (8 erase blocks of 128 KiB each)
flash_erase /dev/mtd5 0 8

# Write the chainloader slot image to the start of the primary slot
nandwrite -p /dev/mtd5 /tmp/xr1710g-chainloader-slot.bin

sync
reboot
```

Additional notes:

- The image written here is `out/xr1710g-chainloader-slot.bin`, not `u-boot.bin`
- These commands apply to the old layout where the primary slot is `tclinux`
- If the device has already been migrated to the new `vendor + chainloader + ubi` layout, write the `chainloader` partition directly instead of writing `tclinux`

</details>

<details>
<summary>Why the raw build artifact cannot be flashed directly</summary>

## Why the Raw Build Artifact Cannot Be Flashed Directly

The vendor boot chain follows a `bootm` path and cannot boot a bare `u-boot.bin` directly.

Because of that, the U-Boot in this project is used as a secondary payload. An outer wrapper with a Linux `Image`-style header and a chainloader package must be added so that the vendor `bootm` path will accept it. That outer wrapper first boots a shim, and the shim then jumps to the real `u-boot.bin`.

In other words, the U-Boot artifact produced by the build cannot be flashed directly. It must first be packaged into a chainloader image with the required Linux-style header.

</details>

<details>
<summary>Packaging files and build commands</summary>

## Files Required for Packaging

To package the raw U-Boot payload into a flashable chainloader image, prepare:

- Raw secondary payload: `u-boot.bin`
- Vendor slot image, for example `mtd5_tclinux.bin`
  Used as a reference input when building the slot image
- Prefix shim: `chainloader-prefix-shim.uImage`
  Provides compatibility with legacy `flash read 0x600000 ...; bootm`
- Shim: `chainloader-shim.bin`
- Control DTB: `chainloader-control.dtb`
- `u-boot/tools/mkimage`
- `u-boot/tools/dumpimage`

The resulting packaged artifacts are typically:

- `out/xr1710g-chainloader.itb`
- `out/xr1710g-chainloader-slot.bin`

Reference: simplified `build-chainloader-fit.sh` script contents, keeping only the core packaging logic:

```sh
#!/usr/bin/env sh
set -eu

stock_slot=xr1710g-backup/mtd5_tclinux.bin
prefix_shim=out/chainloader-prefix-shim.uImage
shim=out/chainloader-shim.bin
payload=out/u-boot.bin
dtb=out/chainloader-control.dtb
output_fit=out/xr1710g-chainloader.itb
output_slot=out/xr1710g-chainloader-slot.bin
mkimage=u-boot/tools/mkimage
dumpimage=u-boot/tools/dumpimage
template=xr1710g-chainloader.its.in
tmpdir=$(mktemp -d)
its="$tmpdir/xr1710g-chainloader.its"

sed \
  -e "s|__DTB__|$dtb|g" \
  -e "s|__SHIM__|$shim|g" \
  -e "s|__PAYLOAD__|$payload|g" \
  -e "s|__KCOMP__|none|g" \
  "$template" > "$its"

"$mkimage" -f "$its" "$output_fit"
"$dumpimage" -l "$output_fit"

# Place a legacy bootm prefix shim at slot offset 0, pad it to 0x2100 bytes,
# then append the full chainloader FIT image.
prefix_size=$(wc -c < "$prefix_shim")
cp "$prefix_shim" "$output_slot"
dd if=/dev/zero bs=1 count=$((0x2100 - prefix_size)) >> "$output_slot" 2>/dev/null
cat "$output_fit" >> "$output_slot"

```

</details>

<details>
<summary>Reference projects</summary>

## Reference Projects

- U-Boot official homepage: <https://u-boot.org/>
- U-Boot official documentation: <https://docs.u-boot.org/>
- U-Boot official source repository: <https://source.denx.de/u-boot/u-boot>

</details>
