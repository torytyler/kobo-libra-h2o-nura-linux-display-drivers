# Build and bring-up evidence

This document records how the display fix was found, the intermediate tests,
and the final build/install path.  It is intentionally more detailed than the
main README.

Tested device:

```
Linux kobo-librah2o 7.1.10 #1-postmarketOS ... armv7l
vermagic: 7.1.10 mod_unload modversions ARMv7 p2v8
```

Kernel source: `akemnade/linux`, commit
`8c68d420f2a30cfdd6673d971e8ae325e9835fe6` (`mxc-epdc` branch).

---

## 1. Hardware discovery

### 1.1 NTX hardware config

The stock eMMC backup was inspected read-only.  At physical offset `0x80000`
(512 KiB) there is an NTX config block with magic `"HW CONFIG v3.1"`.

Relevant decoded fields:

```
bDisplayResolution = 15      # index for 1264 x 1680
bDisplayBusWidth   = 1       # 16-bit SDDO
VCOM_10mV          = 0xff4e  # -1.78 V
```

The stock DTB and vendor kernel were decoded/extracted for comparison:

- `mx6sll-E70K00.dtb`
- `ntx_hwconfig`
- vendor `drivers/video/fbdev/mxc/mxc_epdc_v2_fb.c`

The vendor driver selects the source-driver bus width from
`bDisplayBusWidth`; mainline did not.

### 1.2 What the mainline driver assumed

`epdc_hw.c` in `epdc_init_settings()` contained:

```c
reg_val = ((priv->imx_mode.vscan_holdoff << ...) & MASK)
        | EPDC_TCE_CTRL_PIXELS_PER_SDCLK_4;
```

No SDDO width field was programmed, so the reset/default 8-bit mode was used.

Relevant register bits (`epdc_regs.h`):

```
EPDC_TCE_CTRL (0x200)
  EPDC_TCE_CTRL_SDDO_WIDTH_16BIT    = 0x4
  EPDC_TCE_CTRL_PIXELS_PER_SDCLK_4  = 0x2
  EPDC_TCE_CTRL_PIXELS_PER_SDCLK_8  = 0x3
```

### 1.3 Clock

The vendor timing is the 540 MHz branch.  Mainline left `pll3_pfd2` at
508.235294 MHz:

```
508235294 / 4 = 127058824 Hz
```

The vendor uses:

```
540000000 / 4 = 135000000 Hz
```

The DTS pins both the PFD and the EPDC pixel clock:

```dts
assigned-clocks = <&clks IMX6SLL_CLK_PLL3_PFD2>,
                  <&clks IMX6SLL_CLK_EPDC_PIX>;
assigned-clock-rates = <540000000>, <135000000>;
```

`clk_summary` after boot confirms the values.

---

## 2. The two-stage validation

### 2.1 16-bit SDDO test

To avoid a full kernel rebuild while testing the hypothesis, the installed
module was binary-patched:

- function `epdc_init_settings`
- instruction at `.text+0xb38`
- `orr r5, r0, #2` -> `orr r5, r0, #7`

This changes `PIXELS_PER_SDCLK_4` to
`SDDO_WIDTH_16BIT | PIXELS_PER_SDCLK_8`.

Result: the panel cleared fully on boot, no vertical banding, and text appeared
in the correct position.

A live write to `EPDC_TCE_CTRL` through `/dev/mem` was also attempted earlier,
but it hung the EPDC and dropped the device into fastboot; that path was
recovered with the project's fastboot tool.  The binary module patch was the
safe way to test.

### 2.2 Full-screen update test

Even with 16-bit SDDO, a full-screen framebuffer write did not refresh the
panel: the driver still used `WAVEFORM_MODE_AUTO` + `UPDATE_MODE_PARTIAL`.

A second test module forced:

```
waveform_mode = GC16 (2)
update_mode   = UPDATE_MODE_FULL (1)
```

for every update.  A 1264x1680 BGRX test image written to `/dev/fb0` then
rendered correctly, with accurate grays.  This proved:

- the EPDC data path and waveform are good;
- the remaining issue was update-mode selection, not the panel or framebuffer.

The final source patch is more conservative: only an exact full-screen clip is
promoted to GC16/FULL.

---

## 3. Final source patches

### 3.1 `0002-mxc-epdc-sddo-width.patch`

Files changed:

- `Documentation/devicetree/bindings/display/imx/fsl,mxc-epdc.yaml`
- `drivers/gpu/drm/mxc-epdc/epdc_hw.c`
- `drivers/gpu/drm/mxc-epdc/mxc_epdc_drv.c`
- `drivers/gpu/drm/mxc-epdc/mxc_epdc.h`

DT property:

```dts
sddo-width = <16>;
```

Driver logic:

```c
if (priv->imx_mode.sddo_width == 16)
        reg_val |= EPDC_TCE_CTRL_SDDO_WIDTH_16BIT |
                   EPDC_TCE_CTRL_PIXELS_PER_SDCLK_8;
else
        reg_val |= EPDC_TCE_CTRL_PIXELS_PER_SDCLK_4;
```

The property is optional and defaults to 0, so other panels (for example the
Clara HD e60k02) keep the original 8-bit behaviour.

### 3.2 `0003-mxc-epdc-fullscreen-gc16.patch`

In `mxc_epdc_send_single_update()`:

```c
if (clip->x1 == 0 && clip->y1 == 0 &&
    clip->x2 == priv->epdc_mem_width &&
    clip->y2 == priv->epdc_mem_height) {
        upd_desc->upd_data.waveform_mode = priv->wv_modes.mode_gc16;
        upd_desc->upd_data.update_mode = UPDATE_MODE_FULL;
} else {
        upd_desc->upd_data.waveform_mode = WAVEFORM_MODE_AUTO;
        upd_desc->upd_data.update_mode = UPDATE_MODE_PARTIAL;
}
```

`mode_gc16` is prepared by `epdc_waveform.c` from the panel waveform firmware
and is 2 for the ED070KH1 waveform.  `UPDATE_MODE_FULL` is 1; the previous
constant was `UPDATE_MODE_PARTIAL` (0).

---

## 4. DTB fragment

The upstream `imx6sll-kobo-librah2o.dts` has no `&epdc` node.  The fragment in
`dts/librah2o-epdc.dtsi` adds:

- pinctrl from the vendor DTS;
- EPDC supplies;
- clock assignments;
- vendor 540 MHz timing;
- `sddo-width = <16>`;
- `status = "okay"`.

The tested compiled DTB is `dts/imx6sll-kobo-librah2o-epdc.dtb`.

The kernel package's own DTB does **not** include this fragment.  The working
DTB is a separate asset and must not be overwritten by the package DTB.  Keep
`.stock`, `.preclk`, and `.pre-sddo` backups.

---

## 5. Source build and packaging

The kernel package was built with pmbootstrap in an isolated rootless container
(no host root password needed).  The build applied both patches successfully.

### 5.1 BTF problem

The first build loaded with:

```
failed to validate module [mxc_epdc_drm] BTF: -22
```

The current pahole/clang in the edge build environment generated BTF that the
7.1.10 kernel rejected.  The fix is in `packaging/APKBUILD`:

```
INSTALL_MOD_STRIP="--remove-section=.BTF --remove-section=.BTF.ext --strip-debug"
```

The packaged module now has no `.BTF` section and loads normally.

### 5.2 Final module checks

```
vermagic=7.1.10 mod_unload modversions ARMv7 p2v8
```

Disassembly checks:

- `epdc_init_settings`: the 16-bit branch writes `#7` when the DT property is
  16, otherwise `#2`.
- `mxc_epdc_send_single_update`: the full-screen branch loads `[priv + 64]`
  (`mode_gc16`) and sets update mode 1; the fallback keeps waveform 257 and
  update mode 0.

### 5.3 Final APK

```
linux-postmarketos-imx-ereader-7.1.10-r0.apk
```

The module from that APK is included in `prebuilt/` as
`mxc_epdc_drm-7.1.10.ko.zst`.

---

## 6. Verification

Expected boot log:

```
mxc_epdc 20f4000.epdc: EPDC pix format: 500
[drm] Initialized mxc_epdc 1.0.0 for 20f4000.epdc on minor 0
mxc_epdc 20f4000.epdc: Mode: 1264 x 1680
[drm] fb0: mxc_epdcdrmfb frame buffer device
```

Observed:

- no vertical banding;
- console text/position correct;
- cursor/console updates normal (partial path);
- full-screen grayscale image renders with GC16/FULL;
- no BTF validation error in dmesg.

The test image was a 1264x1680 BGRX buffer written to `/dev/fb0` with fbcon
unbound.  After the final source module, no special "force full refresh"
module was needed.

---

## 7. Historical binary patches (for reference only)

The temporary test modules are not part of the final source build.  The patches
used were:

```
epdc_init_settings, .text+0xb38:
    orr r5, r0, #2  ->  orr r5, r0, #7

mxc_epdc_send_single_update:
    movw ip, #257  ->  movw ip, #2   (AUTO -> GC16)
    mov lr, #0     ->  mov lr, #1    (PARTIAL -> FULL)
```

These were used only to falsify the hypotheses quickly.  The source patches in
`patches/` are the supported implementation.

---

## 8. Device backup layout used during bring-up

```
/lib/modules/7.1.10/kernel/drivers/gpu/drm/mxc-epdc/mxc_epdc_drm.ko.zst.orig
/lib/modules/7.1.10/kernel/drivers/gpu/drm/mxc-epdc/mxc_epdc_drm.ko.zst.sddo16
/lib/modules/7.1.10/kernel/drivers/gpu/drm/mxc-epdc/mxc_epdc_drm.ko.zst.gc16full
/boot/imx6sll-kobo-librah2o.dtb.stock
/boot/imx6sll-kobo-librah2o.dtb.preclk
/boot/imx6sll-kobo-librah2o.dtb.pre-sddo
```

The final installed module is the package-built source module; the old test
modules are kept only as fallbacks.

---

## 9. References

- Andreas Kemnade, `akemnade/linux`, `mxc-epdc` branch.
- Linux `drivers/gpu/drm/mxc-epdc/`.
- Kobo vendor kernel `drivers/video/fbdev/mxc/mxc_epdc_v2_fb.c`.
- EPDC register definitions in `epdc_regs.h`.
- postmarketOS pmaports, `linux-postmarketos-imx-ereader`.
