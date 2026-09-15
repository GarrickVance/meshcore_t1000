# MeshCore T1000-E iOS stale Bluetooth bond recovery

A narrowly scoped MeshCore firmware variant for the Seeed Studio SenseCAP T1000-E that recovers from an inaccessible stale iOS Bluetooth bond.

## Symptom

The MeshCore app or nRF Connect reports `Peer removed pairing information`, but the T1000-E does not appear in **Settings → Bluetooth**, so iOS provides no **Forget This Device** action. Reinstalling the app, resetting network settings, erasing the tracker, and reflashing normal firmware may not clear the stale bond.

## Tested result

This build was field-tested on a SenseCAP T1000-E and iPhone. After flashing by USB serial DFU, the tracker advertised under a new Bluetooth identity, iOS requested the standard MeshCore PIN, the app connected successfully, and the Public channel appeared.

## What changes

The firmware reads the factory Bluetooth address at startup, derives a stable per-device random-static address, and sets it before advertising and security initialization. This makes iOS treat the tracker as a new Bluetooth peripheral.

It does **not** change:

- MeshCore/LoRa identity or keys
- Channels or radio configuration
- T1000-E hardware target
- The standard upstream T1000-E build environment

## Baseline

- Upstream: [meshcore-dev/MeshCore](https://github.com/meshcore-dev/MeshCore)
- Commit: [`d92964352441e53b93e8667b802e04f6e072b39e`](https://github.com/meshcore-dev/MeshCore/commit/d92964352441e53b93e8667b802e04f6e072b39e)
- Release: `v1.17.1`
- Reference image: `t1000e_companion_radio_ble-v1.17.1-d929643.uf2`

## Ready-to-flash files

The `firmware/` directory contains:

- `t1000e_companion_radio_ble-v1.17.1-d929643-alt-ble-identity.zip` — serial DFU package; this is the method field-tested successfully.
- `t1000e_companion_radio_ble-v1.17.1-d929643-alt-ble-identity.uf2` — drag-and-drop UF2 image.

Verify downloads against `SHA256SUMS`.

## Serial DFU—the tested method

Install Adafruit's nRF utility in an isolated environment:

```sh
python3 -m venv ~/adafruit-nrfutil
~/adafruit-nrfutil/bin/pip install adafruit-nrfutil
```

Enter DFU mode, identify the port with `ls /dev/cu.usbmodem*`, and flash:

```sh
~/adafruit-nrfutil/bin/adafruit-nrfutil dfu serial \
  --package ~/Downloads/t1000e_companion_radio_ble-v1.17.1-d929643-alt-ble-identity.zip \
  --port /dev/cu.usbmodemXXXX \
  --baudrate 115200 \
  --singlebank
```

Wait for `Activating new firmware` and `Device programmed.`, then allow the tracker to reboot.

## UF2 method

If the T1000-E mounts as a drive in UF2 bootloader mode, copy the UF2 file from `firmware/` onto that drive. A separate erase is not required when replacing the v1.17.1 application firmware.

## Pairing

1. Turn off Bluetooth on other nearby computers or phones previously connected to the tracker.
2. Let the T1000-E boot completely.
3. Open MeshCore on iOS and select the newly advertised `MeshCore-<address>` device.
4. Enter the standard MeshCore PIN when prompted.
5. Confirm that the device connects and its channels appear.

## Source and rebuilding

Apply [the patch](patches/t1000e-alt-ble-identity.patch) to the exact upstream commit, then build the `t1000e_companion_radio_ble_alt_identity` PlatformIO environment. Detailed commands and build measurements are in [the recovery notes](docs/ios-stale-ble-bond-recovery.md).

To return to the factory Bluetooth identity, flash the official upstream T1000-E Companion Bluetooth firmware.

## Privacy

The source, documentation, and binaries were checked before publication. They contain no user-specific BLE address, device name, channel data, local filesystem path, access token, private key, or personal email. The alternate BLE address is derived at runtime on the device.

## License and attribution

This patch is based on [MeshCore](https://github.com/meshcore-dev/MeshCore), distributed under the MIT License. The upstream project and its contributors retain their applicable copyright.
