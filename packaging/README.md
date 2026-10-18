# Packaging notes

`APKBUILD` is the modified `linux-postmarketos-imx-ereader` recipe.

It:

- lists `0002-mxc-epdc-sddo-width.patch` and
  `0003-mxc-epdc-fullscreen-gc16.patch` in `source`;
- strips `.BTF`/`.BTF.ext` from modules during `modules_install`:

```
INSTALL_MOD_STRIP="--remove-section=.BTF --remove-section=.BTF.ext --strip-debug"
```

The BTF strip is needed because the current pahole/clang in edge generates BTF
that the 7.1.10 kernel rejects (`failed to validate module BTF: -22`).

To build it, copy this `APKBUILD` and the two patches into the pmaports
directory `device/testing/linux-postmarketos-imx-ereader/`, then:

```sh
pmbootstrap build --arch armv7 linux-postmarketos-imx-ereader
```

The resulting module is under
`usr/lib/modules/7.1.10/kernel/drivers/gpu/drm/mxc-epdc/`.
