# ZMK Wireless Macropad + Micro-Scroller — Project Brief

This file is context for Claude Code. It describes a hobby hardware project, its
current state, the exact hardware in hand, a known-good Phase 1 firmware config,
and a Phase 2 roadmap. Read it fully before making changes.

---

## Goal

Build a tiny wireless (Bluetooth LE) input device that works with any
computer/tablet, including an iPad. Two phases:

1. **Phase 1 (now):** a 4-key macropad. Prove out firmware, flashing, wiring.
2. **Phase 2 (next):** add a **scroller** (rotary encoder → scroll events) and a
   **small no-touch pointer** (mini joystick or nav switch → mouse movement).

The owner is new to ZMK and building on a breadboard. Prioritise the simplest
thing that works, low idle power (battery build), and clear explanations.

---

## Hardware in hand (confirmed)

- **Controller:** "Straw ProMicro nRF52840" clone (seller: 偉恩電子材料 / WAYNE
  Electronics, Taiwan). Marketed as Meshtastic/ZMK compatible, BT 5.2.
- **It is pin-compatible with the nice!nano v2.** Build firmware for the ZMK
  board target **`nice_nano_v2`**.
- **Bootloader: confirmed working.** Adafruit nRF52 UF2 bootloader — shorting
  RST→GND twice quickly mounts a USB mass-storage drive. Drag-and-drop `.uf2`
  flashing works. (This was the main clone risk and it passed.)
- **No physical reset button** — there is an **RST pad**; reset = momentarily
  short **RST to GND** (they are adjacent pads on the right side).
- **Through-holes are bare**; header pins need soldering before breadboard use.
- Also on hand: 4-pin tactile switches. LiPo battery + optional EC11 encoder to
  come.

### IMPORTANT: pin labelling

This board's silkscreen uses **raw nRF52840 GPIO numbers** (e.g. `017` = P0.17,
`100` = P1.00, `104` = P1.04), NOT Arduino "D2/D3" numbers. ZMK configs for
nice!nano reference pins as `&pro_micro <N>`, which ZMK translates internally.
Use this table when telling the owner which physical hole to wire:

| ZMK reference   | Physical pad label | nRF52840 pin | Notes            |
|-----------------|--------------------|--------------|------------------|
| `&pro_micro 2`  | `017`              | P0.17        | key 0            |
| `&pro_micro 3`  | `020`              | P0.20        | key 1            |
| `&pro_micro 4`  | `022`              | P0.22        | key 2            |
| `&pro_micro 5`  | `024`              | P0.24        | key 3            |
| `&pro_micro 6`  | `100`              | P1.00        | free / encoder A |
| `&pro_micro 7`  | `011`              | P0.11        | free / encoder B |
| `&pro_micro 8`  | `104`              | P1.04        | free / enc. sw   |

**ADC-capable pads (for an analog joystick in Phase 2):** only `002` (AIN0),
`029` (AIN5), `031` (AIN7) are analog. Pad `115` (P1.15) is NOT analog despite
nice!nano's "A0" silk on some layouts — do not plan analog input on port-1 pins.

---

## Build & flash workflow (how the owner iterates)

Firmware is **built in the cloud via GitHub Actions**, not locally. Claude Code
should edit repo files; it should NOT attempt to compile ZMK locally.

1. Push repo to GitHub → the **Actions** tab builds automatically.
2. On success: Actions run → **Artifacts** → download zip → contains
   `macropad nice_nano_v2.uf2`.
3. Double-tap RST→GND so the USB drive mounts; drag the `.uf2` onto it.
4. Pair to the host over Bluetooth (device name "Macropad").

**Debug loop:** if an Actions build fails, the owner will paste the log. Errors
name the file and line; most are a missing `;`/`,` or a mismatch in the three
counts that MUST stay equal:
- number of entries in `kscan0` `input-gpios`
- number of `keys` in the physical layout
- number of `bindings` in each keymap layer

---

## Phase 1 firmware (known-good starting point)

Repo layout:

```
zmk-config/
├── .github/workflows/build.yml
├── build.yaml
└── config/
    ├── west.yml
    ├── macropad.conf
    └── boards/shields/macropad/
        ├── Kconfig.shield
        ├── Kconfig.defconfig
        ├── macropad.overlay
        └── macropad.keymap
```

### `.github/workflows/build.yml`
```yaml
on: [push, pull_request, workflow_dispatch]
jobs:
  build:
    uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@main
```

### `build.yaml`
```yaml
include:
  - board: nice_nano_v2
    shield: macropad
    snippet: studio-rpc-usb-uart   # enables ZMK Studio (live remapping)
```

### `config/west.yml`
```yaml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: main
      import: app/west.yml
  self:
    path: config
```

### `config/macropad.conf`
```
# Battery: deep sleep after 30 min idle
CONFIG_ZMK_SLEEP=y
CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=1800000
# Live remapping
CONFIG_ZMK_STUDIO=y
```

### `config/boards/shields/macropad/Kconfig.shield`
```
config SHIELD_MACROPAD
    def_bool $(shields_list_contains,macropad)
```

### `config/boards/shields/macropad/Kconfig.defconfig`
```
if SHIELD_MACROPAD
config ZMK_KEYBOARD_NAME
    default "Macropad"
endif
```

### `config/boards/shields/macropad/macropad.overlay`
```dts
#include <physical_layouts.dtsi>

/ {
    chosen {
        zmk,kscan = &kscan0;
        zmk,physical-layout = &physical_layout0;
    };

    kscan0: kscan_direct {
        compatible = "zmk,kscan-gpio-direct";
        wakeup-source;
        input-gpios
            = <&pro_micro 2 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>
            , <&pro_micro 3 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>
            , <&pro_micro 4 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>
            , <&pro_micro 5 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>
            ;
    };

    physical_layout0: physical_layout_0 {
        compatible = "zmk,physical-layout";
        display-name = "Macropad";
        keys
            = <&key_physical_attrs 100 100    0   0 0 0 0>
            , <&key_physical_attrs 100 100  100   0 0 0 0>
            , <&key_physical_attrs 100 100    0 100 0 0 0>
            , <&key_physical_attrs 100 100  100 100 0 0 0>
            ;
    };
};
```

### `config/boards/shields/macropad/macropad.keymap`
```dts
#include <behaviors.dtsi>
#include <dt-bindings/zmk/keys.h>
#include <dt-bindings/zmk/bt.h>

/ {
    keymap {
        compatible = "zmk,keymap";
        default_layer {
            display-name = "Base";
            // order = pin order above: key0 key1 / key2 key3
            bindings = <
                &kp LG(C)   &kp LG(V)
                &kp LG(Z)   &kp LG(LS(Z))
            >;
        };
    };
};
```

`LG(...)` = Left GUI = Cmd on iPad → Copy / Paste / Undo / Redo.

---

## Wiring notes (Phase 1)

- Each 4-pin tactile switch: one leg → its signal pad (`017`, `020`, `022`,
  `024`), the other leg → the GND rail.
- **4-pin tactile gotcha:** the 4 legs are two internally-shorted pairs. On a
  breadboard, straddle the center gap so the two legs used are on opposite sides
  (a valid pair). If a key reads permanently pressed, the legs are a shorted
  pair — rotate the switch 90°.
- Solder headers before trusting any results. Friction-fit pins cause phantom
  presses and dropouts; do not debug firmware on unsoldered contacts.
- Recommended: wire a spare tactile switch across **RST–GND** as a real reset
  button for easy reflashing.

**Bench test without soldering buttons:** after flashing + pairing, briefly short
`017`→GND with tweezers; it should send Cmd+C. Confirms firmware + BLE before any
build work.

---

## Phase 2 roadmap — scroller + pointer

The nRF52840 is correct for this; ZMK pointer/mouse support is first-party but
**evolving quickly**, so verify specifics against current docs at
`zmk.dev/docs/keymaps/behaviors/mouse-emulation` and
`zmk.dev/docs/features/pointing` before finalising. Config symbol for mouse/
pointing has changed across versions (`CONFIG_ZMK_POINTING` on recent mainline);
confirm the current name.

### Scroller (recommended first — simple, low power)
- **Hardware:** EC11 rotary encoder. Wire A→`100`, B→`011`, C(common)→GND; push
  switch → `104` + GND (becomes a 5th direct-GPIO key).
- Add EC11 to overlay (`compatible = "alps,ec11"`, `steps = <80>`), a
  `zmk,keymap-sensors` node, and `CONFIG_EC11=y` +
  `CONFIG_EC11_TRIGGER_GLOBAL_THREAD=y` in `.conf`.
- **Two scroll implementations, pick per current ZMK capability:**
  1. Simple/robust: `sensor-bindings = <&inc_dec_kp PG_UP PG_DN>` (page scroll,
     works everywhere, not true wheel).
  2. True mouse-wheel from the knob: bind mouse-scroll behavior (`&msc SCRL_*`)
     or use an input module. Verify the current stock way to drive
     `INPUT_REL_WHEEL` from an encoder; community modules
     (`badjeff/zmk-input-behavior-listener`, scaler behaviors) exist if stock
     support is insufficient.

### Pointer — three options, in order of recommendation for a wireless build

1. **Digital nav switch / mouse-keys (recommended).** A 5-way switch or 4 buttons
   bound to `&mmv MOVE_UP/DOWN/LEFT/RIGHT`, `&mkp LCLK/RCLK`. Pure first-party
   ZMK, **zero idle draw**, sleeps normally. Fixed-speed movement (acceleration
   configurable via the two-axis behavior; add a slow-mode layer).
2. **Analog thumbstick (what the owner originally described).** Driver:
   `badjeff/zmk-analog-input-driver` (add to `west.yml`). Use ADC pads `029`
   (AIN5) and `031` (AIN7) for X/Y. ⚠️ **Power warning from that driver's own
   README: poll-mode, high power consumption, NOT recommended for wireless
   builds.** A resistive stick also draws continuous current across 3.3V even
   idle. Mitigations: gate stick VCC via a GPIO cut during sleep, or use
   high-value pots — this is custom work, not a flag. Treat as a v2 experiment
   accepting reduced battery life or USB-tethered use.
3. **TrackPoint (ThinkPad nub).** Truest "tiny no-touch pointer." Driver:
   `infused-kim/kb_zmk_ps2_mouse_trackpoint_driver` (mainline fork by `badjeff`).
   Proportional, no moving parts. Downsides: PS/2 is bit-banged (fiddly, timing-
   sensitive), sourcing usually means harvesting from a dead ThinkPad or an
   AliExpress/Taobao module. Advanced.

---

## Instructions for Claude Code

- Do NOT compile locally; the owner builds via GitHub Actions. Edit files only.
- Target board is always `nice_nano_v2`. Reference pins as `&pro_micro N` and
  translate to silkscreen labels using the table above when giving wiring
  directions.
- After ANY change to key/encoder count, re-check the three counts match
  (input-gpios, physical-layout keys, keymap bindings). Call this out explicitly.
- Keep diffs small and explain the "why" of each change; the owner is learning.
- Pointer/mouse/scroll config in ZMK moves fast — when touching Phase 2, prefer
  first-party behaviors, and note anything the owner should verify against live
  docs rather than asserting a possibly-stale API.
- Preserve `CONFIG_ZMK_STUDIO=y` and the `studio-rpc-usb-uart` snippet so live
  remapping keeps working.
- When a build fails, ask for the Actions log; diagnose from the named file/line.

## Current status checklist

- [x] Board sourced, identified as nice!nano v2 compatible
- [x] UF2 bootloader confirmed (RST→GND double-tap mounts drive)
- [ ] Headers soldered
- [ ] Phase 1 config pushed + first successful Actions build
- [ ] Firmware flashed + paired to iPad
- [ ] Buttons wired + all 4 keys verified
- [ ] Phase 2: encoder scroll
- [ ] Phase 2: pointer
