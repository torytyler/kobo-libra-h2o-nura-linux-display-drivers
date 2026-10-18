# Prebuilt artefacts

These are the tested binaries for the exact kernel:

```
Linux kobo-librah2o 7.1.10 ... armv7l
vermagic=7.1.10 mod_unload modversions ARMv7 p2v8
```

- `mxc_epdc_drm-7.1.10.ko.zst` — final source-built module, both patches,
  no `.BTF` section.
- `SHA256SUMS` — checksums for the module and the DTB.

Install the module to:

```
/lib/modules/7.1.10/kernel/drivers/gpu/drm/mxc-epdc/mxc_epdc_drm.ko.zst
```

and the DTB (`../dts/imx6sll-kobo-librah2o-epdc.dtb`) to:

```
/boot/imx6sll-kobo-librah2o.dtb
```

Back up the existing files first.  The module is kernel-version-specific; do
not install it on a different kernel.
