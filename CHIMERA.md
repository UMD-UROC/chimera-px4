# Chimera PX4

UMD-UROC's PX4 firmware for the Chimera vehicle: **PX4 v1.15.4** plus
EchoMAV EchoPilot AI board support and a minimal set of local modifications.

This repo is a fork of [PX4/PX4-Autopilot](https://github.com/PX4/PX4-Autopilot).
The `chimera-v1.15.4` branch is the upstream `v1.15.4` tag plus only the
commits listed below. **All submodules are stock upstream pins** — no forked
submodules (the old personal fork's NuttX fork only contained an accidentally
committed build artifact and is not needed).

## Changes vs upstream v1.15.4

| Commit | Change |
|---|---|
| `2e8138163a` | `boards/echomav/echopilot-ai` imported from EchoMAV's official BSP ([EchoMAV/echopilot_ai_bsp](https://github.com/EchoMAV/echopilot_ai_bsp), branch `board_revision_1b`, commit `a77ae1d`), adapted for PX4 v1.15 (`MICRODDS_CLIENT` → `UXRCE_DDS_CLIENT` rename; `FW_POS_CONTROL_L1` dropped, see note) |
| `53267f1b8f` | TBS Crossfire: build the `crsf_rc` driver (`CONFIG_DRIVERS_RC_CRSF_RC=y`) |
| `ebb08b2484` | DroneCAN rangefinder on **node ID 50** reported as forward-facing (workaround, see below) |

> **Note — no fixed-wing position control:** the BSP enabled `FW_POS_CONTROL_L1`,
> which v1.15 renamed to `FW_POS_CONTROL`. Re-enabling it under the new name
> overflows this board's 1920 KB flash region by ~31 KB, so it is intentionally
> left out (this firmware targets multirotor use). To restore it, free ~31 KB
> first — measured candidates: SIH simulator ~30 KB, FrSky/HoTT/BST legacy RC
> telemetry ~14 KB, serial/I2C distance-sensor drivers ~35 KB.

## Building

Set up the [PX4 v1.15 development environment](https://docs.px4.io/v1.15/en/dev_setup/dev_env.html), then:

```sh
git clone --recurse-submodules -b chimera-v1.15.4 git@github.com:UMD-UROC/chimera-px4.git
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

Reference: [PX4 CRSF telemetry docs](https://docs.px4.io/v1.15/en/telemetry/crsf_telemetry.html)

## Rangefinder orientation workaround

Upstream PX4 hardcodes **every** DroneCAN rangefinder as downward-facing and
has no per-node orientation config (still true on upstream `main` as of
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
