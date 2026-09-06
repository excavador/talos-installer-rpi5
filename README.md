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

All inputs are pinned in [`versions.yaml`](versions.yaml). `talos:` must
equal `talosVersion` in the estate repo; a guard there asserts it, so a bump
in one place without the other fails that repo's `just check`.

## Building

Tag-driven. Tag `v1.14.0-1`, and CI publishes
`ghcr.io/excavador/installer-rpi5:v1.14.0-1`. The tag's version prefix must
match `talos:` in `versions.yaml`; the workflow refuses otherwise.
