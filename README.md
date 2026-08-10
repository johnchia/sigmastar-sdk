# sigmastar-sdk

Kernel-side SigmaStar payload for thingino, kept out of the firmware tree and
fetched at build time by a pinned hash.

Companion to [`sigmastar-lib`](https://github.com/johnchia/sigmastar-lib), which
holds the prebuilt userspace MI libraries. The split follows thingino's Ingenic
convention: `-lib` is userspace, `-sdk` is kernel-side. As with `ingenic-sdk`,
this repository mixes prebuilt vendor artifacts with source that is compiled at
build time.

This repository exists so `thingino-firmware` does not redistribute proprietary
vendor payload on every clone. The packages that consume it declare
`LICENSE = PROPRIETARY` and `REDISTRIBUTE = NO`, which is only truthful if the
binaries live outside that tree — the same arrangement `ingenic-lib` uses.

## Layout

```
4.9.84/                            kernel release, as ingenic-sdk keys its tree
  kmod-0607-glibc-9.1.0/           prebuilt vendor MI modules; see below
  sensor-src/
    infinity6e/ infinity6b0/       sensor drivers, compiled here
                                   (from openipc/sensors)
sensor-iq/<family>/                per-sensor API bins
iqfile/                            CUS3A iqfile + isp_api.xml
venc_fw/<family>/                  chagall.bin — VENC firmware
PROVENANCE
```

**Read `PROVENANCE` before changing anything here.** It records which vendor
build each artifact came from and why the paths are shaped this way.

Only families a thingino target can actually select are carried, and only ones
whose kernel matches the level they sit under. Upstream's sensor tree is not
keyed by kernel release, so a wholesale mirror of it lands families that fail
both tests — `infinity6c` is a 5.10.61 family and has no business under `4.9.84/`.
Copy in the family you need, not the directory above it.

### Why the modules carry a flavour in the directory name

`kmod-<release>-<libc>-<toolchain>` is not decoration. The vendor ships its
modules under `<libc>/<gcc>` trees that are *not* one source built several ways:
across a single release, `mi_sys.ko` differs by 27–29 imported kernel symbols
between trees, differs in `depends=`, and the module sets differ outright.
`vermagic` is byte-identical across all of them and `CONFIG_MODVERSIONS` is off,
so `insmod` accepts a foreign module and it fails later — or misbehaves. The
path is the only thing that records which build a module belongs to.

The sensor drivers deliberately have *no* flavour key: `drv_sensor.h` is
byte-identical across vendor releases nine months apart, and their only coupling
to the vendor bundle is four `CamOs*` symbols from `mhal.ko`. Depth of nesting
encodes coupling — flavour in the path means bound to one vendor build, absence
means release-independent.

### Case

`<family>` is lowercase here and uppercase in `sigmastar-lib`. That mirrors
thingino's own inconsistency (`ingenic-lib/T31` versus
`ingenic-sdk/sensor-iq/t31`) and is deliberate — matching each repository's
local convention beats inventing a third rule.

## Provenance summary

- **Modules** — harvested from a shipped camera firmware via OpenIPC's
  `sigmastar-osdrv-infinity6e`. Alkaid `release_0607`, built 2022-06-07. Not
  taken from a vendor SDK tarball; they match no public drop exactly.
- **Sensor drivers** — mirror of `openipc/sensors`, sigmastar half only. These
  are *not* vendor dumps: OpenIPC ships 11 of the vendor's 68, and the overlap
  has diverged substantially. `gc4653` is retuned from 25fps to 30fps, which is
  what this board streams.

No source is available for the modules and none is claimed here.

## Updating

Consumers pin a commit hash, never a branch — a branch would let the payload
change under a build with nothing in the image recording it. To move the pin,
push here and bump `SIGMASTAR_SDK_VERSION` in
`package/sigmastar-sdk/sigmastar-sdk.mk`.
