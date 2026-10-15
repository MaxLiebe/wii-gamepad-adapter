# Wii Gamepad Adapter

Play the Wii with a modern Bluetooth gamepad. One original ESP32, no wires to
the console: the gamepad connects to the ESP32, and the ESP32 shows up on the
Wii as a **Wii Remote with a Classic Controller** plugged in.

- Gamepads: anything [Bluepad32](https://github.com/ricardoquesada/bluepad32)
  supports, over Bluetooth Classic or BLE (DualSense, DualShock 4, Switch Pro,
  8BitDo, Xbox Wireless, Stadia, Steam, ...).
- Wii side: [WiimoteAnything](https://github.com/MaxLiebe/wiimote-anything)
  emulates the remote and its Classic Controller extension.
- Both share one BTstack instance on one radio.

Rumble from the Wii drives the gamepad's motors. The Wii's player LEDs show on
the gamepad's player LEDs (or as a light bar colour).

## Hardware

An **original ESP32** dev board (ESP32-WROOM-32 / ESP32-WROVER). The ESP32-S3,
C3, C6 and H2 have no Bluetooth Classic radio and cannot talk to a Wii.

## Build and flash

You need ESP-IDF 5.5 or 6.x.

```bash
git clone --recursive https://github.com/MaxLiebe/wii-gamepad-adapter.git
cd wii-gamepad-adapter
idf.py set-target esp32
idf.py build flash monitor
```

Already cloned without `--recursive`? Run
`git submodule update --init --recursive`.

## First-time pairing

Pair the gamepad first, then the Wii.

1. Boot the ESP32 and put the gamepad in pairing mode (table below). The log
   shows `Gamepad ready`.
2. With no Wii remembered yet, the sync window opens by itself for 20 s.
   Press the red **SYNC** button on the Wii (behind the SD card door). The log
   shows `Wii connected`.

Both are remembered across reboots.

| Gamepad | Pairing mode |
| --- | --- |
| DualSense / DualShock 4 | Hold **Create/Share + PS** until the light bar flashes |
| Switch Pro | Hold the small **sync** button on top |
| Xbox Wireless (BLE firmware) | Hold the **pair** button on top until the logo flashes fast |
| 8BitDo (SN30 Pro, Pro 2, ...) | Switch it on in a mode (below), then hold **Pair** for 3 s |

8BitDo pads pair separately in each mode, so pick one and stay with it:

| Power-on combo | Mode |
| --- | --- |
| **Start + Y** | Switch |
| **Start + B** | Android (D-Input) |
| **Start + X** | Windows (X-Input) |
| **Start + A** | macOS |

If a pad that was paired before refuses to connect, hold **BOOT** for 5 s to
forget all gamepads (and the Wii), then pair again.

## Every day

- Turn the gamepad on (PS / Home). It reconnects to the ESP32, and the ESP32
  connects to the Wii. If the Wii was not ready, any button press retries.
- When the Wii drops the remote (a game or homebrew app starting, radio
  trouble), the ESP32 pages it again back to back, like a real remote, for up
  to 60 s (15 s when the Wii reported "powered off", which some reloads do).
- Wii switched off for good (or HOME menu → disconnect): press a button to
  reconnect.
- Gamepad switched off or out of range: the Wii link stays up with all inputs
  released, so the game sees an idle remote until the gamepad is back.

## Controls

| Gamepad (position) | Wii |
| --- | --- |
| Left / right stick | Classic Controller left / right stick |
| D-pad | Classic D-pad (and Wii Remote D-pad) |
| East / South / North / West face buttons | Classic **a / b / x / y** (Nintendo layout, by position) |
| L1 / R1 | Classic **ZL / ZR** |
| L2 / R2 | Classic **L / R** (analog where the gamepad has it; click at 85 %) |
| Select / Start | **− / +** |
| PS / Home / Guide | **HOME** |

The Wii Remote's A / B / 1 / 2 / + / − / D-pad are pressed too
(`ADAPTER_MIRROR_CORE_BUTTONS`), so "Press A" screens and the Wii Menu work
from the gamepad. Set `ADAPTER_LABEL_LAYOUT 1` in `main/main.c` to map by
label (Xbox A → Wii a) instead of by position.

## Buttons on the adapter

| Action | Effect |
| --- | --- |
| **BOOT** short press | Sync window (pair with a Wii, or re-pair) |
| Hold **Select + Start** 3 s | Same, from the gamepad |
| Hold **BOOT** 5 s | Forget the Wii and all gamepads |

The Bluepad32 console on the serial port also works (`list_bluetooth_keys`,
`del_bluetooth_keys`, ...).

## Options (`main/main.c`)

| Option | Default | |
| --- | --- | --- |
| `ADAPTER_ENABLE_BLE` | 1 | BLE gamepads. Scanning only runs while no gamepad is connected. |
| `ADAPTER_MIRROR_CORE_BUTTONS` | 1 | Also press the matching Wii Remote buttons. |
| `ADAPTER_LABEL_LAYOUT` | 0 | Map face buttons by label instead of position. |
| `ADAPTER_SYNC_GPIO` | 0 | Sync / forget button (BOOT). -1 disables it. |

## How the sharing works

- `wiimote_init()` runs after `btstack_init()` and before `uni_init()`.
  WiimoteAnything owns L2CAP PSM 0x11 / 0x13 and forwards every channel that
  is not the console to Bluepad32. A gamepad's incoming channels reach
  Bluepad32 only once the link is authenticated and encrypted, as Bluepad32's
  own services would require (Switch-mode pads depend on that).
- The console is recognised by being the Wii synced before, by the Wii's class
  of device (`0x000448`), or by connecting during the sync window. Its
  connection and PIN requests are hidden from Bluepad32.
- The ESP32 radio stalls when it is slave on one link and master on another,
  so the ESP32 stays master on both. It pages the Wii itself with role
  switching refused. Once a gamepad is ready (never while it is still
  pairing), the adapter asks to be its master too, with sniff mode off and a
  short poll interval.
- Gamepad first, then the Wii. Without a gamepad the Wii link is released
  (the pairing is kept).

## Troubleshooting

- **Gamepad never shows `Gamepad ready`:** forget everything (hold BOOT 5 s),
  then put the pad in pairing mode again. A `Device found: ... COD` log line
  shows that the ESP32 sees it at all.
- **Wii never connects during sync:** press SYNC on the Wii while the log says
  the sync window is open, and keep the ESP32 within a few metres.
- If you had an `sdkconfig` from an older build, delete it so
  `sdkconfig.defaults` applies.

## License

MIT for this project (see `LICENSE`). Bluepad32 is Apache 2.0. BTstack, which
Bluepad32 uses, is free for non-commercial use only; read
[BTstack's license](https://github.com/bluekitchen/btstack/blob/master/LICENSE)
before you sell anything built on it.
