# sigmastar-osdrv

Prebuilt SigmaStar vendor binaries, kept out of the thingino firmware tree and
fetched at build time by a pinned hash.

This repository exists so `thingino-firmware` does not redistribute several
megabytes of proprietary vendor payload on every clone. The package that
consumes it declares `LICENSE = PROPRIETARY` and `REDISTRIBUTE = NO`, which is
only truthful if the binaries live outside that tree — the same arrangement
`ingenic-lib` uses on the Ingenic side.

## Layout

One directory per Infinity family. `sigmastar-osdrv-infinity6e.mk` selects it
with `$(SOC_FAMILY)`, so adding a family is a directory, not a code change.

```
infinity6e/
  kmod/     mi_* and mhal kernel modules, vendor-built against Linux 4.9.84.
            insmod verifies vermagic, so these load only on that kernel.
  lib/      MI userspace libraries. The Raptor HAL dlopens these rather than
            linking them, so they are not link-time dependencies — but an
            image without them has daemons that start and fail at the first
            HAL call.
  sensor/
    configs/    per-sensor ISP tuning blobs
    firmware/   ISP firmware (chagall.bin), IQ file, isp_api.xml
```

Only binaries live here. `load_sigmastar`, `zoom.sh` and `S20sigmastar` are
thingino's own and stay in the firmware tree where they can be reviewed and
diffed.

## Provenance

Ported from OpenIPC's `general/package/sigmastar-osdrv-infinity6e`. These are
vendor-supplied binaries redistributed by SigmaStar's SDK licensees; no source
is available and none is claimed here.

## Updating

Consumers pin a commit hash, never a branch — a branch would let the payload
change under a build with nothing in the image recording it. To move the pin,
push here and bump `SIGMASTAR_OSDRV_INFINITY6E_VERSION` in
`package/sigmastar-osdrv-infinity6e/sigmastar-osdrv-infinity6e.mk`.
