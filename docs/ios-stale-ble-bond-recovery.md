# T1000-E iOS stale BLE bond recovery

This is a narrowly scoped recovery variant of MeshCore Companion Bluetooth v1.17.1 for the Seeed Studio SenseCAP T1000-E. It targets the iOS CoreBluetooth failure `Peer removed pairing information` when the bonded device is absent from Settings and cannot be forgotten.

## Baseline

- Upstream repository: `meshcore-dev/MeshCore`
- Upstream commit: `d92964352441e53b93e8667b802e04f6e072b39e`
- Upstream version: `v1.17.1`
- Reference firmware: `t1000e_companion_radio_ble-v1.17.1-d929643.uf2`

## Implementation

After `Bluefruit.begin()`, the recovery build:

1. Reads the factory Bluetooth address with `sd_ble_gap_addr_get`.
2. Changes the low address byte using XOR `0xA5`.
3. Sets the two high bits required for a Bluetooth random-static address.
4. Assigns the derived address with `sd_ble_gap_addr_set` before advertising and security initialization.

The derivation is stable but device-specific. It changes only the BLE identity presented to the phone; it does not alter the MeshCore/LoRa identity, keys, channels, radio configuration, or target hardware. The normal `t1000e_companion_radio_ble` environment remains unchanged.

## Build

```sh
git clone https://github.com/meshcore-dev/MeshCore.git
cd MeshCore
git checkout d92964352441e53b93e8667b802e04f6e072b39e
git apply /path/to/t1000e-alt-ble-identity.patch

python3 -m venv .venv
.venv/bin/pip install platformio
.venv/bin/pio run -e t1000e_companion_radio_ble_alt_identity

.venv/bin/python bin/uf2conv/uf2conv.py \
  .pio/build/t1000e_companion_radio_ble_alt_identity/firmware.hex \
  -c \
  -o t1000e_companion_radio_ble-v1.17.1-d929643-alt-ble-identity.uf2 \
  -f 0xADA52840
```

## Verified build

- Alternate build RAM: 144,952 / 235,520 bytes (61.5%)
- Alternate build flash: 345,268 / 708,608 bytes (48.7%)
- Standard build flash: 345,220 / 708,608 bytes
- Difference from standard: 48 bytes
- UF2 size: 690,688 bytes
- UF2 family: nRF52840
- UF2 target address: `0x27000`

## Flashing by serial DFU

The serial DFU ZIP is the method validated in the field:

```sh
python3 -m venv ~/adafruit-nrfutil
~/adafruit-nrfutil/bin/pip install adafruit-nrfutil
~/adafruit-nrfutil/bin/adafruit-nrfutil dfu serial \
  --package /path/to/t1000e_companion_radio_ble-v1.17.1-d929643-alt-ble-identity.zip \
  --port /dev/cu.usbmodemXXXX \
  --baudrate 115200 \
  --singlebank
```

Use `ls /dev/cu.usbmodem*` to identify the current port. Do not use `--baud-rate`; the Adafruit utility's option is `--baudrate`.

## Flashing by UF2

When the bootloader mounts as a drive, copy the alternate-identity UF2 onto it. A separate erase is not required when replacing the v1.17.1 application firmware.

## Field validation

Validated on a SenseCAP T1000-E and iPhone on 15 September 2026. Before the change, iOS repeatedly returned `Peer removed pairing information` and exposed no Bluetooth Settings entry that could be forgotten. After flashing by USB serial DFU, iOS requested the standard MeshCore PIN, the app connected successfully, and the Public channel was available.

## Reversal

Flash the official upstream T1000-E Companion Bluetooth firmware to restore the factory BLE identity. If iOS still retains the prior factory bond, the original symptom may return under that identity.
