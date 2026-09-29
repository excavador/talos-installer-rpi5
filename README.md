# talos-installer-rpi5

The Talos installer image for the Raspberry Pi 5 that runs this estate's
control plane. Built by GitHub Actions, published to
`ghcr.io/excavador/installer-rpi5`, public and anonymously pullable.

## Why it exists

The Image Factory cannot produce a bootable Pi 5 image for **D0 silicon**.
The `rpi_5` overlay ships `rpi_generic`'s u-boot binary
([sbc-raspberrypi#96](https://github.com/siderolabs/sbc-raspberrypi/issues/96)),
which on a D0 board either fails to boot or comes up without advertising
`SetVariableRT` — and without that, `talosctl upgrade` dies at the
bootloader step:

```
failed to install bootloader: failed to create efivarfs reader/writer: invalid argument
```

So the image is the factory recipe with our u-boot laid over the top. The
u-boot itself is built and published separately by
[`excavador/u-boot-rpi5`](https://github.com/excavador/u-boot-rpi5); this
repo consumes its release and verifies the checksum.

**Exit condition:** when #96 is fixed upstream, this repo and the u-boot
fork both retire.

## Why it is public

Nothing in the image is private. It is the public Talos installer, the
public `sbc-raspberrypi` overlay, the public `siderolabs/tailscale`
extension binary, and a GPL u-boot we already publish. The tailscale
**auth key is not in the image** — it arrives at runtime through
`ExtensionServiceConfig` in machine config.

Public also removes a dependency loop. A private registry would be reached
through an in-cluster pull-through cache authenticating with in-cluster
credentials, so the installer would be fetchable only while the cluster's
identity and registry planes are both healthy — thin precisely during the
partial outage where you most want it.

## Versions

All inputs are pinned in [`versions.yaml`](versions.yaml), by digest as well
as tag. Extension and overlay tags are re-pointed as Siderolabs rebuild them
for newer Talos releases — the bare `tailscale:1.102.2` tag resolves to a
different digest than the one Talos v1.14.0's own release manifests name for
it — so CI checks the pinned digests against
`ghcr.io/siderolabs/extensions:v1.14.0` and
`ghcr.io/siderolabs/overlays:v1.14.0` directly and refuses to build on a
mismatch. `talos:` must equal `talosVersion` in the estate repo; a guard
there asserts it, so a bump in one place without the other fails that
repo's `just check`.

## Building

Tag-driven. Tag `v1.14.0-1`, and CI publishes
`ghcr.io/excavador/installer-rpi5:v1.14.0-1`, the flashable SD image, and a
GitHub Release carrying both plus `provenance.json` and `installer.digest`.
The tag's version prefix must match `talos:` in `versions.yaml`; the
workflow refuses otherwise.

## The SD image needs an extra step

The Talos imager's `metal` output for `rpi_5` is not bootable as the imager
produces it. Measured 2026-09-29 with the pinned imager (v1.14.0): the
imager never runs the overlay installer in image mode, so the resulting
image's EFI System Partition holds only `EFI/` and `loader/` — no
`config.txt`, no `u-boot.bin`, no device trees. The Pi 5's firmware reads
`config.txt` from the FAT root at boot and finds nothing there.

So the build runs the step the imager skips: the *same* overlay installer
binary that ships inside the installer image just published (pulled back by
digest), invoked exactly as a node's own install invokes it, writing into a
staging directory that is then copied onto the disk image's ESP. That makes
the SD image and the installer carry the same u-boot, `config.txt` and
device trees by construction, and the build reads the bytes back off the
disk image and compares them byte-for-byte rather than assuming the copy
took.

## Consuming the installer: by digest, never `repo:tag@sha256:`

Point nodes and machine config at
**`ghcr.io/excavador/installer-rpi5@sha256:<digest>`** — the digest recorded
in a release's `installer.digest` or `provenance.json`. Do not use the bare
tag, and never use the `repo:tag@sha256:` form.

Talos v1.14's image pull parses the reference with
`reference.ParseDockerRef` (`github.com/distribution/reference` v0.6.0),
which **drops the tag whenever a digest is present**. An image referenced as
`repo:tag@sha256:…` is therefore stored under `repo@sha256:…` — the tag is
gone — while `talosctl upgrade` looks the image up afterwards by the exact
string it was given, tag included, and fails with "not found in containerd
store". The digest-only form is stored and looked up under the same name,
so it is the only form that survives a pull. This applies to anything else
that hands a node an installer reference, such as a `talconfig.yaml`: the
tag can stay in a comment or commit message, but the reference itself must
be digest-only.

No pre-pull-and-verify dance is needed to reach this installer: it is a
public, anonymously pullable GHCR image, so a node's own registry mirror, if
it has one, can fall back to pulling it directly from GHCR. That dance only
earns its keep when the only path to an installer runs through
infrastructure that can itself be down.

## Licensing

The `u-boot.bin` embedded in this image — both in the installer and in the
flashable SD image — is GPL-2.0. Its complete corresponding source is
[`excavador/u-boot-rpi5`](https://github.com/excavador/u-boot-rpi5), at the
tag recorded as `uboot.tag` in each release's `provenance.json`.
