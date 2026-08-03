# Chimera PX4

UMD-UROC's PX4 firmware for the Chimera vehicle: **PX4 v1.16.2** plus
EchoMAV EchoPilot AI board support and a minimal set of local modifications.

This repo is a fork of [PX4/PX4-Autopilot](https://github.com/PX4/PX4-Autopilot).
The `chimera-v1.16.2` branch is the upstream `v1.16.2` tag plus only the
commits listed below. **All submodules are stock upstream pins** — no forked
submodules (the old personal fork's NuttX fork only contained an accidentally
committed build artifact and is not needed).

## Changes vs upstream v1.16.2

| Commit | Change |
|---|---|
| `3e48666a20` | `boards/echomav/echopilot-ai` imported from EchoMAV's official BSP ([EchoMAV/echopilot_ai_bsp](https://github.com/EchoMAV/echopilot_ai_bsp), branch `board_revision_1b`, commit `a77ae1d`), adapted for PX4 v1.15+ (`MICRODDS_CLIENT` → `UXRCE_DDS_CLIENT` rename; `FW_POS_CONTROL_L1` dropped, see note) |
| `dbe7614bf8` | TBS Crossfire: build the `crsf_rc` driver (`CONFIG_DRIVERS_RC_CRSF_RC=y`) |
| `5ce76b00c9` | DroneCAN rangefinder on **node ID 50** reported as forward-facing (workaround, see below) |
| `4c09efb8d9` | Reset safety: hold PWM pins low 100 ms so ESCs disarm on reboot (upstream `047578a844` swept in-tree boards only) |
| `88ae35d6b5` | Drop SIH simulator (~33 KB) — the v1.16.2 base overflowed the flash region by ~21 KB |

> **Note — flash budget:** this board has 1920 KB of usable flash and the
> v1.16.2 image uses **98.6%** of it (~27 KB free). Deliberately left out:
> fixed-wing position control (`FW_POS_CONTROL`, the BSP's old
> `FW_POS_CONTROL_L1`) and the SIH simulator (host-side SITL still provides
> SIH). Remaining freeing candidates if more space is ever needed (measured
> at v1.16.2): FrSky/HoTT/BST legacy telemetry ~14 KB, serial/I2C
> distance-sensor drivers ~27 KB, optical-flow drivers ~20 KB.

## Building

Set up the [PX4 v1.16 development environment](https://docs.px4.io/v1.16/en/dev_setup/dev_env.html), then:

```sh
git clone --recurse-submodules -b chimera-v1.16.2 git@github.com:UMD-UROC/chimera-px4.git
cd chimera-px4
make echomav_echopilot-ai_default
```

Flash over USB with `make echomav_echopilot-ai_default upload`, or load
`build/echomav_echopilot-ai_default/echomav_echopilot-ai_default.px4` as
custom firmware in QGroundControl.

## CRSF (Crossfire) setup

PX4 does not build CRSF support by default; this firmware does (the generic
`rc_input` driver stays disabled, as the PX4 docs recommend). On the vehicle:

- `RC_CRSF_PRT_CFG` — serial port wired to the CRSF receiver. If that port
  has a default MAVLink mapping (e.g. TELEM2 via `MAV_1_CONFIG`, set by this
  board's defaults), clear that mapping first.
- `RC_CRSF_TEL_EN` — set to 1 for CRSF telemetry back to the transmitter.

Reference: [PX4 CRSF telemetry docs](https://docs.px4.io/v1.16/en/telemetry/crsf_telemetry.html)

## Rangefinder orientation workaround

Upstream PX4 hardcodes **every** DroneCAN rangefinder as downward-facing and
has no per-node orientation config (still true at v1.16.2 and on upstream `main` as of
Aug 2026). The vehicle carries one downward- and one forward-facing DroneCAN
rangefinder, so the sensor on CAN node ID **50** is reported forward-facing
(`FORWARD_FACING_NODE_ID` in `src/drivers/uavcan/sensors/rangefinder.cpp`).
Verify with `listener distance_sensor` (orientation field).

## Porting to a newer PX4

1. Fetch upstream and branch from the new tag:
   `git checkout -b chimera-vX.Y.Z vX.Y.Z`
2. Cherry-pick the chimera commits from the table above (board, CRSF,
   rangefinder, docs).
3. Check [EchoMAV/echopilot_ai_bsp](https://github.com/EchoMAV/echopilot_ai_bsp)
   for a newer board revision branch and diff against `boards/echomav/`.
4. Check whether upstream added per-node DroneCAN rangefinder orientation —
   if so, drop the rangefinder commit and configure it properly instead.
5. Build, then bench-test: CRSF RC input, and both rangefinders' orientations.
