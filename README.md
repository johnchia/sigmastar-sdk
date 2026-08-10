# sigmastar-sdk

Kernel-side SigmaStar payload for thingino, kept out of the firmware tree and
fetched at build time by a pinned hash.

Companion to [`sigmastar-lib`](https://github.com/johnchia/sigmastar-lib), which
holds the prebuilt userspace MI libraries. As with `ingenic-sdk`, this repository
mixes prebuilt vendor artifacts with source that is compiled at build time.

This repository exists so `thingino-firmware` does not redistribute proprietary
vendor payload on every clone. The packages that consume it declare
`LICENSE = PROPRIETARY` and `REDISTRIBUTE = NO`, which is only truthful if the
binaries live outside that tree — the same arrangement `ingenic-lib` uses.

## Layout

Family first. Every artifact here is chip-coupled, so each family is
self-contained and a build indexes one directory with `$(SOC_FAMILY)`.

```
infinity6e/
  kmod-4.9.84-0607-glibc-9.1.0/    prebuilt vendor MI modules; see below
  sensor-src/                      sensor drivers, compiled here
  sensor-iq/                       per-sensor API bins
  iqfile/                          CUS3A iqfile + isp_api.xml
  venc_fw/                         chagall.bin — VENC firmware
  PROVENANCE
infinity6b0/
  sensor-src/                      drivers only — no vendor payload yet
  PROVENANCE
sensor-src-common/                 Makefile + sensor_config.c, symlinked
PROVENANCE                         layout rules and shared source
```

**Read `PROVENANCE` before changing anything here.** Payload provenance is
per family, in `<family>/PROVENANCE`; the top-level file covers the shared
sensor source and the rule that decides what belongs where.

Only families a thingino target can actually select are carried —
`soc/sigmastar/*.mk` defines exactly `infinity6e` and `infinity6b0`. Upstream's
sensor tree is not keyed the way this one is, so copy in the family you need,
not the directory above it.

### Why the kernel release is inside the kmod directory name

`kmod-<release>-<drop>-<libc>-<toolchain>` is not decoration. The vendor ships
its modules under `<libc>/<gcc>` trees that are *not* one source built several
ways: across a single release, `mi_sys.ko` differs by 27–29 imported kernel
symbols between trees, differs in `depends=`, and the module sets differ
outright. `vermagic` is byte-identical across all of them and
`CONFIG_MODVERSIONS` is off, so `insmod` accepts a foreign module and it fails
later — or misbehaves. The path is the only thing that records which build a
module belongs to, and the kernel release leads it because that is the component
`uname -r` must agree with.

The sensor drivers deliberately have *no* flavour key: `drv_sensor.h` is
byte-identical across vendor releases nine months apart, and their only coupling
to the vendor bundle is four `CamOs*` symbols from `mhal.ko`. Depth of nesting
encodes coupling — flavour in the path means bound to one vendor build, absence
means release-independent.

### Keeping the two repositories in step

The modules here and the libraries in `sigmastar-lib` are two halves of one
vendor build, and a mismatched pair loads without complaint. The drop is
therefore defined once, in `soc/sigmastar/<family>.mk`:

```make
SIGMASTAR_DROP := 0607
SIGMASTAR_LIBC := glibc
SIGMASTAR_GCC  := 9.1.0
SIGMASTAR_KREL := 4.9.84
```

Both packages read those, so the two pins move together or the path stops
resolving. Bumping one repository to a new vendor drop without the other is a
build failure rather than a runtime fault.

## Provenance summary

- **Modules** — harvested from a shipped camera firmware via OpenIPC's
  `sigmastar-osdrv-infinity6e`. Alkaid `release_0607`, built 2022-06-07. Not
  taken from a vendor SDK tarball; they match no public drop exactly. The same
  is true of `isp_api.xml` and `chagall.bin`, which were checked against every
  local vendor tree and match none of them.
- **Sensor drivers** — from `openipc/sensors`, sigmastar half only. These are
  *not* vendor dumps: OpenIPC ships 11 of the vendor's 68, and the overlap has
  diverged substantially. `gc4653` is retuned from 25fps to 30fps, which is what
  this board streams.

No source is available for the prebuilt artifacts and none is claimed here.

## Updating

Consumers pin a commit hash, never a branch — a branch would let the payload
change under a build with nothing in the image recording it. To move the pin,
push here and bump `SIGMASTAR_SDK_VERSION` in
`package/sigmastar-sdk/sigmastar-sdk.mk`.
