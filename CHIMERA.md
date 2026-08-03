# Chimera PX4

UMD-UROC's PX4 firmware for the Chimera vehicle: **PX4 v1.18.0-beta1** plus
EchoMAV EchoPilot AI board support and a minimal set of local modifications.

> **This is a beta-preview branch.** The flying release lines are
> `chimera-v1.15.4` (current), `chimera-v1.16.2`, and `chimera-v1.17.0`.
> Expect to re-verify when v1.18.0 final is tagged.

This repo is a fork of [PX4/PX4-Autopilot](https://github.com/PX4/PX4-Autopilot).
The `chimera-v1.18.0-beta1` branch is the upstream `v1.18.0-beta1` tag plus
only the commits listed below. **All submodules are stock upstream pins.**

## Changes vs upstream v1.18.0-beta1

| Commit | Change |
|---|---|
| `066873d559` | `boards/echomav/echopilot-ai` imported from EchoMAV's official BSP ([EchoMAV/echopilot_ai_bsp](https://github.com/EchoMAV/echopilot_ai_bsp), branch `board_revision_1b`, commit `a77ae1d`), adapted for PX4 v1.15+ (`MICRODDS_CLIENT` → `UXRCE_DDS_CLIENT` rename; `FW_POS_CONTROL_L1` dropped, see note) |
| `f6c071b954` | TBS Crossfire: build the `crsf_rc` driver (`CONFIG_DRIVERS_RC_CRSF_RC=y`) |
| `3fd773e6fa` | DroneCAN rangefinder on **node ID 50** reported as forward-facing (workaround, see below; re-based onto v1.18's `make_uavcan_device_id()` refactor) |
| `8dd0a6a647` | Reset safety: hold PWM pins low 100 ms so ESCs disarm on reboot (upstream `047578a844` swept in-tree boards only) |
| `114637f972` | Drop `rover_pos_control` (module removed upstream in v1.17; this vehicle is a multirotor) |
| `2b59474751` | NuttX config: `CONFIG_PTHREAD_MUTEX_TYPES=y` — required by v1.18's mavlink (build fails without it) |
| `02fdf41e52` | Heater defines adapted to v1.18's multi-heater scheme (`HEATER_NUM`/`GPIO_HEATER1_OUTPUT`; same PA7 pin) |
| `c6ee5d0c07` | Flash: drop SIH (~37 KB), FrSky/HoTT/BST legacy telemetry (~14 KB), Roboclaw (~3 KB) |
| `ee3aea4d14` | Flash: drop fixed-wing/VTOL attitude modules (~44 KB) — FW position control never fit on this board, so FW/VTOL flight was already impossible |

> **Note — flash budget:** this board has 1920 KB of usable flash and the
> v1.18.0-beta1 image uses **97.5%** of it (~48 KB free, headroom reserved
> for beta→final growth). Deliberately left out on this branch: fixed-wing
> position control (never fit, since v1.15), SIH simulator, FrSky/HoTT/BST
> legacy telemetry, Roboclaw, rover_pos_control (removed upstream), and the
> fixed-wing/VTOL attitude modules. **Kept by explicit decision:** optical
> flow, serial/I2C distance sensors, camera capture/trigger/feedback, ROS 2
> bridge (`uxrce_dds_client`), Septentrio GNSS, airspeed stack, SMBus
> batteries. Remaining freeing candidates (measured at v1.18.0-beta1):
> ROS 2 bridge ~82 KB, serial/I2C distance sensors ~22.5 KB, optical flow
> ~20.5 KB, airspeed stack ~17.3 KB, Septentrio ~12.6 KB, cameras ~12.2 KB.

## Building

Set up the [PX4 development environment](https://docs.px4.io/main/en/dev_setup/dev_env.html), then:

```sh
git clone --recurse-submodules -b chimera-v1.18.0-beta1 git@github.com:UMD-UROC/chimera-px4.git
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

Reference: [PX4 CRSF telemetry docs](https://docs.px4.io/main/en/telemetry/crsf_telemetry.html)

## Rangefinder orientation workaround

Upstream PX4 hardcodes **every** DroneCAN rangefinder as downward-facing and
has no per-node orientation config (still true at v1.18.0-beta1 and on
upstream `main` as of Aug 2026). The vehicle carries one downward- and one
forward-facing DroneCAN rangefinder, so the sensor on CAN node ID **50** is
reported forward-facing (`FORWARD_FACING_NODE_ID` in
`src/drivers/uavcan/sensors/rangefinder.cpp`).
Verify with `listener distance_sensor` (orientation field).

## Porting to a newer PX4

1. Fetch upstream and branch from the new tag:
   `git checkout -b chimera-vX.Y.Z vX.Y.Z`
2. Cherry-pick the chimera commits from the table above (board, CRSF,
   rangefinder, ESC-disarm, compat fixes, flash trims, docs).
3. Check [EchoMAV/echopilot_ai_bsp](https://github.com/EchoMAV/echopilot_ai_bsp)
   for a newer board revision branch and diff against `boards/echomav/`.
4. Check whether upstream added per-node DroneCAN rangefinder orientation —
   if so, drop the rangefinder commit and configure it properly instead.
5. Diff a comparable in-tree STM32H743 board (e.g. `boards/matek/h743`)
   across the version jump to catch swept board fixes and new mandatory
   NuttX config options.
6. Watch the flash budget — the base image grows every release; expect to
   trim more modules (candidates in the note above).
7. Build, then bench-test: CRSF RC input, both rangefinders' orientations,
   optical flow, and camera triggering.
