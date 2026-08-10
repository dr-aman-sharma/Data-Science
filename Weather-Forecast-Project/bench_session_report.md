# Bench Testing and Gimbal Search Development — Session Report

**Aircraft:** TBXT tower homing hexacopter
**Companion computer:** Jetson, `drona@drona-desktop`
**Camera/gimbal:** SIYI ZR10 at 192.168.144.25
**Dates:** 9–10 August 2026

---

## Contents

1. Summary
2. Findings — flight-relevant
3. Findings — methodology
4. Bench test coverage
5. Gimbal measurements
6. Gimbal search development
7. Open questions
8. Recommendations

---

# 1. Summary

Two days of bench work produced **five flight-relevant findings**, one of which
prevented the homing software from communicating with the flight controller at
all. It also produced a working gimbal search capability, tested on hardware
through eight development iterations.

The single most important observation is a pattern rather than an individual
fault: **this system fails silently.** Three separate faults were found in
which the software continued running and displayed plausible but incorrect
values, with no error raised anywhere.

## Status at end of session

| Area | State |
|---|---|
| Bench tests passed | 5 of 30 |
| Bench tests partial | 2 |
| Bench tests blocked | 0 (the CH7 blocker was resolved) |
| Faults found | 5 |
| Faults fixed | 1 (serial port) |
| Gimbal search | Working on hardware, not flight-tested |

---

# 2. Findings — Flight-Relevant

## F1. The homing software could not reach the flight controller

**Severity: high. Fixed during the session.**

`config.yaml` lines 26 and 32 specified `/dev/ttyTHS1`. That is the telemetry
port. The telemetry radio was unserviceable, so the flight controller was
connected over USB, which enumerates as `/dev/ttyACM0`.

Evidence:

| Port | Heartbeat |
|---|---|
| `/dev/ttyTHS1` (config) | none — `wait_heartbeat` hung indefinitely |
| `/dev/ttyTHS2` | none |
| `/dev/ttyACM0` | connected, `sys=1` |

**The failure was silent.** The overlay displayed:

```
Y1495 P1495 R1495 T1495    ch4in:1495  ch7in:0
```

Every value a default. Nothing on screen indicated the link was absent. A pilot
reading that display had no reason to doubt it.

Fixed by changing both config lines to `/dev/ttyACM0`. After the change:

```
[MAV] Serial /dev/ttyACM0 @ 115200
[MAV] Connected sys=1 comp=0
[CH7] 1045 [LOW]
```

**Note for the team:** ACM device numbers can move if another USB device
enumerates first. A stable device path would be more robust than `ttyACM0`.

## F2. The CH7 middle position does nothing

**Severity: medium. Not fixed.**

CH7 is a three-position switch: 1045 / 1495 / 1945.

The software uses hysteresis, HIGH above 1600 and LOW below 1400, holding the
previous state in between. **1495 falls inside the dead zone**, so the middle
position never changes the control state.

Compounding this, two different thresholds exist in the same file:

| Location | Threshold | Reports at 1495 |
|---|---|---|
| Terminal print, line 207 | `ch7 > 1500` | "LOW" |
| State machine | `< 1400` for LOW | stays HIGH |

**The printed message and the actual behaviour disagree.** A pilot moving the
switch to the middle would see "[CH7] 1495 [LOW]" in the terminal while the
system remained autonomous.

Suggested remedies: assign a two-position switch, or move the middle detent
outside 1400–1600, and align the print threshold with the state logic.

## F3. The HUD overlay freezes while the system runs

**Severity: medium. Not fixed. Not fully diagnosed.**

After extended running, the overlay stopped updating. With CH7 moved to full
low, the terminal correctly printed `[CH7] LOW — pilot` while the overlay
continued to display `CH7:HIGH` and a frozen `ch7in:1945`. The raw value was
stale, not only the derived label.

A restart cleared it. Not reproduced since; a timed soak test is needed to
establish whether it recurs and after how long.

All five status paths use the same pattern — lines 146, 185, 189, 195 and 200:

```python
try: status_queue.put_nowait(...)
except: pass
```

`put_nowait` fails immediately when the queue is full, and the bare `except`
discards the failure. Altitude and attitude travel the same queue, so those
freeze too.

**Measured in simulation** at the system's real rates — 50 Hz producer,
22 Hz consumer:

| Design | Data age at start | After 6 s | Queue depth | Updates dropped |
|---|---|---|---|---|
| Queue, one item per loop | 413 ms | **3,537 ms** | full | 84 |
| Queue, drained every loop | 7.8 ms | 11.1 ms | 0 | 0 |
| Shared memory | 10.3 ms | 9.9 ms | — | 0 |

The safety concern is direction-dependent. An overlay showing HIGH while the
pilot has control is confusing. **An overlay showing LOW while the system is
still autonomous would be dangerous**, and nothing in the current design
prevents it.

Two changes suggested: drain the queue fully each loop rather than taking one
item, and add a staleness indicator so a frozen display is visible as frozen.

## F4. Detection fails at high screen brightness

**Severity: unknown. Partial test.**

Displaying the tower image on a screen and raising brightness caused detection
to fail. The direction is notable: the configuration guards against images
being too dark (`v_min: 80`) but has no guard against washout.

Likely mechanism: as brightness rises, red desaturates toward white and falls
below `s_min: 80`. Not confirmed.

**Relevance to flight:** a bright screen is a rough analogue of a tower face in
direct sunlight. Worth repeating with the printed matte target to separate
screen behaviour from detector behaviour.

**Caveat:** the aircraft runs YOLO (`detection.mode: tower`), not the HSV
colour detector these thresholds belong to.

## F5. The colour detector is not selective enough for a real environment

**Severity: low for flight (YOLO is used). High for any colour-mode fallback.**

During closed-loop tracking on the bench with production thresholds
(`confidence_threshold: 0.3`, minimum area 0.3% of frame), the detector
switched target between frames.

| Observed | Range |
|---|---|
| `error_x` between consecutive frames | +0.84 to −0.44 |
| `size_ratio` | 0.09 to 0.96 |
| `confidence` | 0.23 to 0.93 |

The camera drifted approximately 90° away from the intended target.

Measured separation between intended and stray objects:

| Object | Confidence |
|---|---|
| Intended target, clearly in view | 0.70 – 0.93 |
| Stray red objects in the room | 0.23 – 0.50 |

There is a clean gap. The production threshold of 0.30 accepts both.

**Structural observation:** the production detector selects
`max(contours, key=contourArea)` — a single blob — then applies
`max_position_jump` to reject it if it moved too far. With only one candidate
offered, the guard can reject a wrong object but cannot request the right one.
Measured in this session: 16 consecutive rejections, after which the tracker
timed out and adopted the distractor anyway.

Additionally, `max_position_jump` assumes a stationary camera. At 71.5° field
of view, 5° of gimbal movement between frames displaces a fixed object by 0.14
in `error_x` — most of the way to the 0.35 threshold on its own.

---

# 3. Findings — Methodology

## M1. The SIYI SDK serves one client at a time

Established by direct measurement, not inference.

| Test | Result |
|---|---|
| Second client sends `0x25` while the first is listening | First stops receiving after 2 s |
| Socket that listens without requesting | 0 packets in 5 s |
| Two clients polling simultaneously | Silent reply loss |

Demonstrated live and reproducibly: with `tower_homing_color.py` running,
`probe_gimbal_sdk.py` returned `zoom: NO REPLY`. With it stopped, the same
command returned `1.0x`.

**Consequence:** any gimbal measurement taken while the homing software is
running is unreliable. Early Test 9 data was affected — the spread of resting
readings was ~1.9° contaminated versus 0.3° clean, a factor of six.

**In flight this does not apply**, since only the homing software runs. It is a
bench-methodology constraint, not a flight fault. It should, however, be
verified whether the pilot's ground station also commands the gimbal in flight.

## M2. The homing software polls rather than using push mode

`grep 0x25` across `TowerHomingClient/` returns nothing.

`zr10_flow/inputs/gimbal_input.py` implements push mode and its notes record it
measured at 50 Hz on 7 August. Independently reproduced this session:

```
501 packets in 10.0 s -> 50.0 Hz  [PASS]
push interval: mean 20.0 ms, sd 1.8 ms
```

That module's own notes state the existing polling implementation blocks up to
200 ms waiting for an acknowledgement, which is unsuitable at frame cadence.

The flight code is dated 1–12 July; the push-mode work is dated 7 August. The
finding appears not to have been carried across.

**Important qualification:** push mode does not enable multiple readers. It
sends to the requesting socket only, as measured above. Sharing gimbal data
between processes requires one owner plus local distribution.

## M3. Thirty-six copies of the main program exist

`find ~ -name "tower_homing_color.py"` returns 36 paths, including one in the
Trash, spread across folders named `Working_Code`, `PRP`, `Bharadwaj`,
`old files`, `code_backp` and `jetson_code_backup`.

The live file was identified from `ps aux` — direct evidence, not inference:

```
/home/drona/jetson/TowerHomingClient/tower_homing_color.py
```

Not `IBVS_HOMING_2`, not `Working_Code`.

## M4. Documentation drift

Three mismatches between the file docstrings and the running configuration:

| Docstring says | Config / runtime says |
|---|---|
| Detector: HSV RED, `detection.mode = color` | `mode: tower`, runtime prints `TOWER (YOLO)` |
| Pitch driven by distance from geometry | `PITCH now drives from SIZE_RATIO` |
| Window title `TBXT Tower Homing (RED)` | Running YOLO |

Not necessarily faults, but on a system with three confirmed silent failures,
worth reconciling.

---

# 4. Bench Test Coverage

Against the 30-test plan.

## Passed

| Test | Result |
|---|---|
| **8** — gimbal direction | Positive yaw turns the camera LEFT. Confirmed with observer facing the camera; +45 appeared to observer's right. Matches the Gazebo convention documented in `tbxt_input_module_sitl1.py` |
| **10** — gimbal speed | ~78°/s sustained, clean deceleration, no overshoot |
| **11** — travel limits | −160.3° to +159.1°, total sweep 319.4°. The gimbal reported the stop rather than the commanded 200°. **Note: the SIYI manual specifies ±135° control range; measured travel exceeds it** |
| **18** — FC communication | Passed after the F1 fix. CH7 toggles cleanly, mode follows every time |
| **22** — emergency stop | Passed with conditions. Works from full-low (1045) only; see F2 |

## Partial

| Test | State |
|---|---|
| **2** — brightness | Failure point identified (see F4). Numbers not recorded per brightness level |
| **9** — angle accuracy | See section 5. Physical movement accurate; reported angle offset by ~2° |

## Not started

Tests 1, 3–7, 12–17, 19–21, 23–30.

Tests 19–21 require an arming session. All others are unblocked — the camera
feed was confirmed working at 1280×720 during this session, which was the
previous obstacle.

---

# 5. Gimbal Measurements

## Reported angle at rest

Six separate readings across two days, gimbal physically stationary:

```
-1.9, -1.8, -1.9, -2.0, -2.1, -2.0        spread ±0.15
```

Remarkably stable. Identical at the start and end of the session.

## Commanded versus reported versus actual movement

| Commanded | Reported | Actual movement |
|---|---|---|
| +10 | 7.7 | +9.6 |
| +20 | 17.3 | **+19.4** |
| +20 (day 1) | 17.9 | +18.7 |
| +45 | 43.8 | +44.8 |

**Physical movement tracks the command closely — within 0.6°.** The
discrepancy sits in the reported value.

## Return to zero

| Approaching from | Settled at |
|---|---|
| +45 | +1.0 |
| +20 | +0.4 |
| −160 | −0.9 |

Direction-dependent. Approaching from above stops positive, from below stops
negative. **That signature indicates a deadband, not a fixed calibration
offset** — a constant offset would read the same value every time.

Consistent with `gimbal_input.py`'s note that in Follow mode the camera yaw
tracks the airframe with a deadband and the residual offset is a small
differential quantity.

## Motion mode

The gimbal is in FOLLOW mode. **This is set by the software itself**, in
`GimbalStateProvider.start()`:

```python
mode = self._pm(self._send(0x0A, b''))
if mode != 'FOLLOW':
    self._send(0x0C, bytes([0x04])); time.sleep(0.5)
```

Not a pilot setting. Nobody needs to guess which mode the aircraft flies in.

## Push mode

```
501 packets in 10.0 s -> 50.0 Hz   requested 50 Hz  [PASS]
push interval: mean 20.0 ms, sd 1.8 ms
```

Independently reproduces the 7 August measurement (50.0 Hz, 20.0 ms,
1.9–2.3 ms jitter).

**Caveat on the velocity fields:** the probe tool reports them as usable, but
the measurement was taken with the gimbal stationary. Maxima of 0.3–0.6 °/s are
the noise floor, not evidence the fields track real motion. The 7 August test
made the distinction properly — hand-tilting gave 16.7 °/s against 0.3–0.6
stationary.

## Camera

```
rtsp://192.168.144.25:8554/main.264
1280x720 H265 2097 kbps @ 30 fps       (main stream)
2560x1440 H265 12288 kbps @ 30 fps     (recording stream)
```

**Resolution mismatch:** `config.yaml` specifies 1920×1080. The camera delivers
1280×720.

Measured throughput with no processing in the loop: 26.7 fps over 10 s.

---

# 6. Gimbal Search Development

## Context

The real system does not sweep. `tower_homing_color.py` line 678:

```python
state = 'SEARCHING'; phase = 'SEARCHING'; rc_cmd = NEUTRAL  # wait (no scan)
```

The aircraft waits when it has no target. The production `GimbalStateProvider`
is read-only — it sends `0x0D`, `0x0A`, `0x18`, plus `0x0C` and `0x08` at
startup. There is no `0x0E`, so no way to command an angle.

A full SEARCH → LOCKING → LOCKED → HOLD state machine exists, but only in
`tbxt_input_module_sitl1.py`, the Gazebo stand-in. The simulation has been
testing a capability the aircraft does not have.

## Approach

`gimbal_search.py` **subclasses** the production `GimbalStateProvider` rather
than replacing it. Everything the flight code depends on is inherited
unchanged: the polling thread, the 500 ms staleness gate, zoom validation, the
FOLLOW-mode startup sequence, `get()` and `stop()`.

Only two things are added: `set_angle()` using `0x0E`, and the `track()` state
machine ported from the SITL module.

This shape was chosen because the gimbal is accessed in exactly four places in
`tower_homing_color.py` — construction at 398, `gimbal.get()` at 603, a logging
read at 748, and `stop()` at 815. A replacement needs only `get()` returning
the same dictionary and `stop()`.

## Faults found and fixed during development

| # | Fault | Found by | Effect |
|---|---|---|---|
| 1 | `send_hz` default 20 against a 22 Hz loop | Simulation | **50% of commands dropped**, control rate halved to 11 Hz |
| 2 | Gain sensitivity | Simulation | Limit cycle above ~0.25 |
| 3 | Camera reader stalling | Hardware | `Stream timeout after 30051 ms`, fps collapsed to 0 |
| 4 | Command windup | Hardware | Command ran 8–15° ahead; continuous oscillation, target never centred |
| 5 | No object persistence | Hardware | Switched target frame to frame; camera drifted 90° away |
| 6 | No settling deadband | Hardware | Locked and followed correctly but never held still |

**Faults 1 and 2 were found only by simulation. Faults 3 to 6 were found only
on hardware.** Neither method would have found them all.

## Gain measurement

Same 50 randomised scenarios, gain held fixed, all else varying:

| kp | Passed | Oscillating | Residual error |
|---|---|---|---|
| 0.15 | 50/50 | 0 | 0 |
| 0.20 | 50/50 | 0 | 0 |
| 0.25 | 49/50 | 1 | 0 |
| 0.30 | 48/50 | 2 | 0 |
| 0.40 | 48/50 | 2 | 0 |
| 0.60 | 44/50 | 5 | 1 |
| 0.80 | 41/50 | 6 | 3 |
| 1.20 | 34/50 | 12 | 7 |

Degradation is a gradient, not a cliff. Default set to 0.2. Measured cost
against 0.4: median settling 1.38 s versus 1.03 s over 30 runs — 0.35 s slower.

`KP_MAX = 0.50` blocks the range where behaviour clearly degrades.

## Approaches tried and rejected

| Approach | Result |
|---|---|
| Deadband on `ex` (first attempt) | Introduced residual error at high zoom when noise pushed the measurement inside the band |
| Command from the reported angle | **0/50.** Re-applies the reporting offset each frame — drifts 1.9° per frame with zero error |
| Rate damping from `0x0D` | 0/50 at the gain chosen. Correct value not determined |
| Anti-windup lead clamp (in simulation) | 48/50 — no improvement |

The third row is instructive. The absolute-command law scores 48/50 with the
offset removed from the model, identical to the accumulating design. **The
accumulating design is immune to the offset and performs equally well**, so it
is the correct choice regardless of whether the offset is ever calibrated out.

The fourth row is a caution about simulation. The lead clamp showed no benefit
in the model because the modelled gimbal kept up with its commands. On hardware
it was the necessary fix.

## Final configuration

```
kp            0.2      measured: 50/50 across randomised conditions
KP_MAX        0.5      ceiling on misconfiguration
send_hz       50       above the 22 Hz loop rate
max_rate      30 deg/s halved after the hardware oscillation
MAX_LEAD_DEG  8.0      anti-windup, from the measured 8-15 deg lead
EX_DEADBAND   0.05     1.8 deg at 1x zoom
yaw limits    +/-155   inside the measured -160.3/+159.1 stops
```

## Verification

| Suite | Result |
|---|---|
| Deterministic, 8 tests | 8/8 |
| Randomised, 50 scenarios | 50/50 |
| Closed-loop with rendered camera frames | 5/5 |
| Hardware — import, selftest, single angle, sweep | 4/4 |
| Hardware — closed-loop tracking | Locks and follows; settling behaviour under test |

The closed-loop simulation renders frames from the gimbal's physical pointing
direction, so the detector processes real pixels. Detector geometry matched
true bearing to three decimal places at five positions.

## Not established

- Behaviour in flight. Never flown
- Interaction between sweeping and FOLLOW mode's own airframe tracking, beyond
  the observation that commanding works in FOLLOW
- Performance with the YOLO detector. All closed-loop work used HSV colour
- `build_inputs_from_config` — the integration path — has never been called

---

# 7. Open Questions

| # | Question | Why it matters |
|---|---|---|
| Q1 | Does the pilot's ground station command the gimbal in flight? | Determines whether M1 contention applies in the air |
| Q2 | Is the ~2° reporting offset a deadband or a calibration error? | `SignalProcessor` subtracts it every frame |
| Q3 | Why does `GimbalStateProvider.start()` open a second RTSP connection? | Undocumented; holds it until `stop()` |
| Q4 | Why does the homing software request sudo at startup? | Unusual for this class of application |
| Q5 | What does SDK command `0x08` with data `0x01` do? | Sent at every startup, followed by a 2 s sleep |
| Q6 | Should the aircraft sweep when it loses the target, or wait? | A design decision. The capability now exists and is tested |
| Q7 | Were the HSV thresholds ever validated against a real tower? | Comments say "default: red cloth" |

## The largest open question

**The simulation run supplied at the start of this session showed a flyaway.**
Starting gimbal yaw 20°, the aircraft flew a curved path away from the tower:
`ex` pinned near −0.5 for the full 60 s and never converged, heading swept
monotonically from +20° to −120°, and the state machine held HOMING throughout.

**This is unexplained.** None of the findings in this report account for it.

Adding a sweeping camera to an aircraft with an unexplained divergence would be
building on something nobody understands yet.

---

# 8. Recommendations

## Immediate

1. **Verify the F1 fix survives a reboot.** ACM device numbering can move
2. **Reconcile CH7** — assign a two-position switch, or move the middle detent
   outside 1400–1600, and align the print threshold with the state logic
3. **Add a staleness indicator to the HUD.** The display can freeze silently.
   This protects against every variant of that failure, diagnosed or not

## Short term

4. **Run the timed soak test** for the HUD freeze — check CH7 every five
   minutes for 45 minutes and record when it stops responding
5. **Complete Tests 1 to 7.** The camera feed is confirmed working, so these
   are unblocked
6. **Run Test 15** — the gimbal correction. Described in the source document as
   the most important calculation in the system, and still never checked on
   hardware
7. **Investigate the simulation flyaway** before any decision on search

## Design decisions for the team

8. **Should the aircraft sweep when it loses the target?** The capability is
   built and bench-tested. Enabling it is a behaviour change
9. **Bring push mode into the flight path?** Measured at 50 Hz; the existing
   polling blocks up to 200 ms per read
10. **Consolidate the 36 copies.** The live file is confirmed; the rest are a
    standing hazard

## Methodology

11. **One program on the gimbal at a time.** Check `ps aux` before every gimbal
    measurement. A lock file would make the rule enforceable rather than
    remembered
12. **Re-examine the 19 previous `bench_runs`** — if any were taken with two
    programs on the gimbal port, the same contamination applies

---

# Appendix — Reference Values

```
Live code        /home/drona/jetson/TowerHomingClient/tower_homing_color.py
Config           /home/drona/jetson/TowerHomingClient/config.yaml
Gimbal tooling   /home/drona/zr10_flow/tools/probe_gimbal_sdk.py
Camera           rtsp://192.168.144.25:8554/main.264   1280x720
Gimbal SDK       192.168.144.25:37260 UDP
Flight controller /dev/ttyACM0 @ 115200   (config previously said ttyTHS1)
CH7              1045 low / 1495 middle / 1945 high
Gimbal travel    -160.3 to +159.1 deg, measured
Slew rate        ~78 deg/s, measured
Resting report   -1.9 deg +/- 0.15, six readings
Loop rate        18.2 - 22.1 Hz, observed
```

## Confidence levels

**High** — supported by repeated direct measurement: travel limits, slew rate,
resting offset, push mode rate, the serial port fault, the contention
demonstration, CH7 switch values.

**Moderate** — single or few observations: the HUD freeze (one occurrence, not
reproduced), the brightness failure (direction clear, magnitude not recorded),
the deadband interpretation of the return-to-zero data.

**Low / inferred** — the mechanism behind the HUD freeze (queue saturation is a
hypothesis supported by simulation of the same rates, not by instrumenting the
running system), and the desaturation explanation for F4.
