# Deskset ESB Dongle - ZMK Config

This repository holds the ZMK firmware for a USB dongle setup. One XIAO nRF52840 dongle connects to the Mac with USB.
The Glove80 sidecar trackpad and the two halves of the Aurora Corne connect to the dongle with Nordic ESB.
The ESB link sends data at intervals of approximately 1 ms.

macOS Bluetooth gives third-party mice connection intervals of approximately 11.25–15 ms.
macOS also delivers the Bluetooth reports at irregular times. This caused visible steps in the cursor movement.
The dongle gives the Mac regular USB reports.

**Status:** All builds pass CI. Nobody has tested the firmware on hardware yet.

## Physical Setup

### Overview

```
 Corne left  (keys)            --ESB--\
 Corne right (keys + trackpad) --ESB---+--> XIAO dongle --USB--> Mac
 Sidecar     (trackpad + key)  --ESB--/
```

The dongle is the only device with a connection to the Mac.
The sidecar and the two Corne halves send their data to the dongle.

### Parts List

| Device | Component | Purpose |
| :--- | :--- | :--- |
| Dongle | **Seeed Studio XIAO nRF52840 (XIAO BLE)** | USB connection to the Mac, ESB receiver |
| Sidecar | **Seeed Studio XIAO nRF52840 (XIAO BLE)** | Trackpad controller, ESB transmitter |
| Sidecar | **Cirque GlidePoint trackpad** | Pointer input |
| Corne left | **nice!nano v2** | Key matrix controller, ESB transmitter |
| Corne right | **nice!nano v2** | Key matrix and trackpad controller, ESB transmitter |
| Corne right | **Cirque GlidePoint trackpad** | Pointer input |

### Dongle

The dongle is a XIAO nRF52840 without wiring. It has no keys.

1. Connect the dongle to the Mac with a USB-C cable.

### Sidecar Wiring

The sidecar trackpad uses the SPI3 bus. The pins come from `deskset_sidecar.overlay`.

| XIAO pin | nRF52840 GPIO | Signal | Trackpad pad |
| :--- | :--- | :--- | :--- |
| D10 | P1.15 | SCK | SCK |
| D9 | P1.14 | MOSI | SI |
| D8 | P1.13 | MISO | SO |
| D7 | P1.12 | CS | SS |
| D0 | P0.02 | DR (data ready) | DR |
| 3V3 | — | Power | VDD |
| GND | — | Ground | GND |

The sidecar also has one key. Connect the key between GPIO P0.03 and GND.
The firmware uses an internal pull-up on this pin, and the key is active low.

### Corne Right Wiring

The Corne right trackpad uses the SPI1 bus. The pins come from `deskset_corne_right.overlay`.

| nice!nano pin | nRF52840 GPIO | Signal | Trackpad pad |
| :--- | :--- | :--- | :--- |
| D3 | P0.20 | SCK | SCK |
| D2 | P0.17 | MOSI | SI |
| D20 | P0.29 | MISO | SO |
| D19 | P0.02 | CS | SS |
| D0 | P0.08 | DR (data ready) | DR |
| VCC | — | Power | VDD |
| GND | — | Ground | GND |

> **CAUTION:** Do not install the OLED or the right encoder on the Corne right half.
> They use the same pins as the trackpad. The firmware disables the I2C bus (`&pro_micro_i2c`).

### Corne Left

The Corne left half has keys only. It needs no additional wiring.
The firmware also disables the I2C bus on this half.

## Flashing

> **NOTE:** With this ESB firmware, the sidecar and the Corne halves do not pair with Bluetooth.
> A USB connection from these devices to the Mac only charges them.
> To return to Bluetooth, flash the firmware from
> [zmk-config-glove80-sidecar](https://github.com/mcadams93/zmk-config-glove80-sidecar) and
> [zmk-config-aurora-corne](https://github.com/mcadams93/zmk-config-aurora-corne).

CI builds these UF2 files (from `build.yaml`):

| UF2 file | Device |
| :--- | :--- |
| `deskset_dongle-xiao_ble_zmk.uf2` | Dongle (XIAO) |
| `deskset_sidecar-xiao_ble_zmk.uf2` | Sidecar (XIAO) |
| `settings_reset-xiao_ble_zmk.uf2` | Settings reset for the sidecar (XIAO) |
| `deskset_corne_left-nice_nano_zmk.uf2` | Corne left (nice!nano v2) |
| `deskset_corne_right-nice_nano_zmk.uf2` | Corne right (nice!nano v2) |

The exact file names can have small differences between ZMK versions. Use the shield name to find the correct file.

To enter the bootloader, double-tap the reset button. The device then shows as a USB drive on the Mac.

1. Open the latest run in the GitHub Actions tab of this repository.
2. Download the `firmware` artifact.
3. Extract the ZIP file.
4. Put the sidecar into the bootloader.
5. Copy the settings reset UF2 file to the sidecar drive.
6. Put the sidecar into the bootloader again.
7. Copy the sidecar UF2 file to the sidecar drive.
8. Do steps 4 to 7 again for each Corne half with its own firmware file.
9. Put the dongle into the bootloader.
10. Copy the dongle UF2 file to the dongle drive.
11. Connect the dongle to the Mac with the USB-C cable.

NOTE: `build.yaml` builds `settings_reset` only for the XIAO board.
For the Corne halves, use a `settings_reset` UF2 for the nice!nano v2 from another ZMK build.

## How It Works

### Topology

The dongle is the split central (`ZMK_SPLIT_ROLE_CENTRAL`). It holds the keymap.
The dongle uses only USB HID to send reports to the Mac.

Bluetooth is off on all devices (`CONFIG_ZMK_BLE=n`, `CONFIG_ZMK_SPLIT_BLE=n`). All split traffic uses ESB (`CONFIG_ZMK_SPLIT_ESB=y`).
The dongle accepts three peripherals (`CONFIG_ZMK_SPLIT_ESB_PERIPHERAL_COUNT=3`).

| Device | Role | ESB peripheral ID |
| :--- | :--- | :--- |
| Dongle | Central | — |
| Corne left | Peripheral | 1 |
| Corne right | Peripheral | 2 |
| Sidecar | Peripheral | 3 |

The ESB addresses are in `deskset_esb_addr.dtsi`. These are our own values for this set of devices.
All devices must use the same file. A device with a different address does not connect to the dongle.

### Dependencies

`config/west.yml` pins all revisions to a commit SHA.

| Project | Remote | Revision | Purpose |
| :--- | :--- | :--- | :--- |
| `zmk` | badjeff | `641514a` | ZMK fork on Zephyr 4.1 and NCS 3.1 |
| `zmk-feature-split-esb` | badjeff | `314c7cb` | Split transport over Nordic ESB |
| `sdk-nrf` | badjeff | `9b3d262` | Patched nRF Connect SDK v3.1 (`v3.1-branch+zmk-fixes`) |
| `nrfxlib` | nrfconnect | `dfadf17` | Nordic libraries for `v3.1-branch` |
| `zmk-kscan-blackout` | badjeff | `1ccc6b1` | Empty key scan for the dongle |
| `zmk-input-processor-report-rate-limit` | badjeff | `b056cb1` | Limit of the cursor report rate |
| `cirque-input-module` | nxtkb | `8af03a4` | Cirque Pinnacle trackpad driver |

The ESB split feature needs the ESB library from the nRF Connect SDK.
The badjeff ZMK fork and the patched `sdk-nrf` v3.1 supply this library on Zephyr 4.1.
The nxtkb driver gives the trackpad gestures in absolute mode.
ZMK needs a key scan on each device. The dongle has no keys, so it uses `zmk,kscan-blackout`.
The report rate limit module controls how often the dongle sends cursor movement to the Mac.

### Trackpad Event Path

1. The Cirque driver reads the trackpad in absolute mode. It converts the position to relative movement with the scale 12/40.
2. The driver sends the events to a `zmk,input-split` node on the peripheral.
   The sidecar uses `reg = <0>` (`sidecar_split`). The Corne right uses `reg = <1>` (`corne_tpad_split`).
3. The peripheral sends the events to the dongle over ESB.
4. The dongle receives the events on the `zmk,input-split` node with the same `reg` value.
5. A `zmk,input-listener` on the dongle processes the events with `<&zip_report_rate_limit 1>`.
   This limit applies only to the cursor X and Y movement. Buttons and the wheel go through without delay.
6. The dongle sends a USB HID mouse report to the Mac.

The sidecar listener also inverts X with `zip_xy_transform INPUT_TRANSFORM_X_INVERT`.
`deskset_zplit.dtsi` defines the two `input-split` nodes. Each device includes this file.

### Keymap

The keymap is on the dongle in `deskset.keymap`. It has 43 positions.
Positions 0 to 41 are the Corne keys. Position 42 is the sidecar key.
Position 42 has the binding `&none` on all layers.
The keymap has no `&bt` bindings, because the dongle does not use Bluetooth.

### Trackpad Functions

Both trackpads use the same settings, except for the inversion.
The overlays `deskset_sidecar.overlay` and `deskset_corne_right.overlay` set these properties.

| Function | Value | Property |
| :--- | :--- | :--- |
| Tap = left click | Maximum 200 ms, maximum movement 80 | `primary-tap-enable`, `absolute-tap-max-ms`, `absolute-tap-max-movement` |
| Click pulse length | 30 ms | `absolute-tap-click-ms` |
| Tap-drag | Timeout 300 ms | `absolute-tap-drag-enable`, `absolute-tap-drag-timeout-ms` |
| Right click | Tap in a zone of 520 x 420 | `absolute-secondary-tap-area-width`, `absolute-secondary-tap-area-height` |
| Scroll | Right edge zone of 240, divisor 40 | `absolute-right-edge-scroll-enable`, `absolute-scroll-zone`, `absolute-scroll-divisor` |
| Movement scale | 12/40 | `absolute-relative-multiplier`, `absolute-relative-divisor` |
| Inversion (sidecar) | X | `invert-x` |
| Inversion (Corne right) | X and Y | `invert-x`, `invert-y` |

The driver sets the location of the right-click zone. This repository does not set the location.

### Reliability Settings

The trackpad peripherals use 8 ESB hardware retransmits (`CONFIG_ZMK_SPLIT_ESB_PROTO_TX_RETRANSMIT_COUNT=8`).
The radio sends the same packet again in the same position. This keeps the order of the packets.

The application retry (`RETRY_INPUT_EVENT`) stays at 0 on purpose.
This retry puts a failed packet at the end of the queue. It can change the order of a press and its release.

## Build and Customize

### Build

GitHub Actions builds the firmware on each push and each pull request.
The workflow uses the ZMK `build-user-config.yml` workflow. `build.yaml` lists the board and shield for each build.

### Change the Keymap

1. Edit `boards/shields/deskset/deskset.keymap`.
2. Keep 43 bindings in each layer.
3. Push the change.
4. Flash only the dongle with the new firmware.

### Add a Device

1. Give the new peripheral a new ID with `CONFIG_ZMK_SPLIT_ESB_PERIPHERAL_ID`.
2. Increase `CONFIG_ZMK_SPLIT_ESB_PERIPHERAL_COUNT` in `deskset_dongle.conf`.
3. For a trackpad, add a new `zmk,input-split` node with a new `reg` value in `deskset_zplit.dtsi`.
4. Add a new `zmk,input-listener` for this node in `deskset.keymap`.
5. Include `deskset_esb_addr.dtsi` in the overlay of the new device.
6. Add the new shield to `build.yaml`.

CAUTION: All devices must use the same ESB address file. Do not change the addresses on only one device.

## Known Limits and Items to Test on Hardware

- Cursor smoothness on the Mac with the 1 ms ESB link.
- Double click on macOS with the 30 ms tap pulse.
- Packet loss on the ESB link, and the effect of 8 retransmits.
- The Corne pin mapping for the keys and the trackpad.
- The trackpad direction on the Corne right half.
- The X inversion on the sidecar.
- The battery use of the peripherals with the 1 ms link.

## References

- [badjeff/zmk-feature-split-esb](https://github.com/badjeff/zmk-feature-split-esb)
- [badjeff/zmk-config, branch esb-shield-only](https://github.com/badjeff/zmk-config/tree/esb-shield-only)
- [nxtkb/cirque-input-module](https://github.com/nxtkb/cirque-input-module)
- [nxtkb/Sweep-Pro](https://github.com/nxtkb/Sweep-Pro)
