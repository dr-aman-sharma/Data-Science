# Measuring the Moments of Inertia of Our Hexacopter
## And Updating the Gazebo / ArduPilot SDF Model

---

## 1. What We're Doing and Why

The body inertia in our SDF model (`base_link`) is just a rough hand-estimate
(a hollow-cylinder guess). It doesn't properly capture the mass sitting far out
along the arms and the heavy items inside the frame.

We're going to **measure the real moments of inertia** of the main body with a
simple two-rope pendulum test, and drop those measured numbers into `base_link`.
A more accurate inertia means the simulated drone behaves like the real one, which
means we can tune the flight controller in simulation and actually trust it.

We measure three values — **Ixx (roll), Iyy (pitch), Izz (yaw)**.

---

## 2. What is On the Drone

Our fully-built hexacopter contains:

- Carbon frame, six arms, landing skids and legs
- Flight **battery** (the heaviest single item, ~2.0 kg)
- **Jetson** onboard companion computer
- All the **wiring and electronics** — ESCs, power board, GPS, telemetry, receiver, mounts
- **Cube Orange** flight controller (with the IMU inside)
- **Two cameras:**
  - a **downward-facing ZR10** camera fixed to the body frame
  - a **camera on a stabilised gimbal**
- **Six propellers** (Tarot 1655, 16-inch)

How the SDF is built (this is the whole key to the procedure): the drone isn't one
solid block. It is split into separate "links" — body, each rotor, the flight
controller, each camera, etc. Every link carries its own mass and its own inertia,
and Gazebo automatically adds them all together to get the inertia of the whole
drone. So our job is just to make sure **each part is counted once, in the right
place.**

---

## 3. The One Rule That Decides Everything

Every part's inertia must be counted **exactly once**. Some parts already have a
correct, separate link in the SDF. Others only live as a rough lump inside the body.

The deciding question for each part is simple:

> **Does anything in the SDF besides its own inertia refer to this part?**
> (a motor plugin, an IMU sensor, a camera sensor, etc.)

- **If YES** → its link must stay in the file, so we **take the physical part OFF
  the drone** during the test (otherwise we'd count it twice).
- **If NO** → it is just dead weight in the file, so we **leave it ON the drone**,
  measure it, and fold it into `base_link`.

Here is how every part falls out:

### Take OFF the drone for the test (their links stay in the SDF)

| Part | Why its link must stay |
|------|------------------------|
| 6 propellers | Used by the thrust (lift-drag) plugins and the motor controls |
| Cube Orange | Holds the IMU that ArduPilot reads |
| ZR10 downward camera | Has its own camera sensor/plugin |
| Gimbal camera | Has its own camera sensor/plugin |

### Keep ON the drone and measure (these fold into `base_link`)

| Part | Why it folds into base_link |
|------|-----------------------------|
| Frame, arms, skids | The body itself |
| Battery | No plugin refers to it — pure mass |
| Jetson companion computer | No plugin refers to it — pure mass |
| Wiring / electronics | Already lumped into the body |

So the thing we actually hang and measure is:
**frame + battery + Jetson + wiring — with the props, the Cube, and BOTH cameras
removed.**

(Note: this is a change from an earlier plan where the camera was folded into the
body. Because the cameras have their own sensors/plugins, they must stay separate,
so they come OFF for the test instead.)

---

## 4. The Theory (Short Version)

We hang the drone from two parallel ropes of length `h`, separated by distance `D`,
and run two tests.

**Test 1 — Gravity swing (a sanity check).** Swing it gently and time the swing.

    g = 4π²h / T²

If this comes out near **9.81 m/s²**, the setup is good and we can trust it. If not,
fix the leveling/ropes and try again. This measures nothing new — it just proves the
rig works.

**Test 2 — Torsional twist (the real measurement).** Twist the drone about the
vertical axis, release, and time the rotation.

    I = (m · g · D² · T²) / (16 · π² · h)

The answer is in **kg·m²**. Here `m` is the mass of the thing we hung, and `T` is
the torsional period.

The test always measures inertia about the **vertical axis through the middle of the
two ropes**, so to get each axis we just re-hang the drone with that axis pointing up:

- **Izz (yaw):** drone upright (normal flying attitude)
- **Iyy (pitch):** tipped 90° so the pitch axis is vertical
- **Ixx (roll):** tipped 90° the other way so the roll axis is vertical

Run **both** tests in **each** of the three orientations.

---

## 5. What we Need

- A strong overhead beam or frame to hang from
- Two ropes of equal length
- Measuring tape (for `h` and `D`)
- Digital spirit level (hang it level — aim for under 1°)
- A stopwatch — or better, log the **Cube's onboard gyro** and read the period off
  that (much more accurate)
- A weighing scale
- Tools to remove the props, the Cube, and both cameras

---

## 6. Procedure

### Step A — Prep the drone

1. Remove all **six propellers**.
2. Remove the **Cube Orange**.
3. Remove the **ZR10 downward camera** and the **gimbal camera**.
4. Leave **on**: frame, arms, skids, **battery** (in its bay), **Jetson**, and all
   wiring/electronics.
5. Power the drone **off** — nothing should spin or move.
6. **Weigh this exact configuration.** That scale reading is the new `base_link`
   mass (frame + battery + Jetson + wiring). Write it down.

### Step B — Hang and level

7. Hang the drone from two ropes in the first orientation.
8. Attach the ropes at two well-separated points, balanced about the centre of
   gravity so the drone hangs level.
9. Check level with the spirit level (aim under 1°).
10. Measure and record `h` (rope length) and `D` (rope separation).

### Step C — Gravity swing test

11. Swing gently, time at least 10 full swings, average to get `T`.
12. Compute `g = 4π²h / T²`. It should be near 9.81.
    - Good → continue. Off → fix leveling/ropes and redo.

### Step D — Torsional test

13. Wind the drone about the vertical axis until the ropes spiral together.
14. Release, time at least 10 full rotations, average to get `T`.
15. Compute `I = (m · g · D² · T²) / (16 · π² · h)`.

### Step E — Repeat

16. Repeat B–D for all three orientations (yaw, pitch, roll).
17. Also note the **centre-of-gravity height** of this configuration (roughly where
    the mass balances along the vertical axis).

We'll end with three numbers — **Ixx, Iyy, Izz** — plus the body mass and CG.

---

## 7. Updating the SDF File

### Edit 1 — Delete the battery link and joint (it folds into the body)

    <link name="battery_link"> … </link>
    <joint name="battery_joint" type="fixed">…</joint>

### Edit 2 — Make sure BOTH cameras are their own links (do NOT fold them in)

The current SDF has only one `camera_link`. Since we have **two** cameras, the second
one needs its own link too. Each camera link should keep:
- its own **mass** and **inertia** (estimate each as a simple box, or measure each
  one on its own — they're small and easy)
- its **camera sensor / plugin**
- a **fixed joint** to `base_link`
- its correct **mount position** (the ZR10 looking down on the body, the gimbal
  camera where it actually sits)

Do **not** delete the camera links and do **not** put the cameras' mass into
`base_link`.

### Edit 3 — The Jetson folds into the body (no separate link)

The Jetson's mass is already included in the `base_link` mass we weighed in Step A6,
so it does **not** get its own link. Nothing to add — just we need to make sure it was on the
drone when we weighed and measured.

### Edit 4 — Overwrite the `base_link` inertial block

    <inertial>
      <pose>0 0 CGz 0 0 0</pose>     <!-- measured CG height on the vertical axis -->
      <mass>MEASURED_MASS</mass>     <!-- the scale reading from Step A6 -->
      <inertia>
        <ixx>MEASURED_Ixx</ixx>
        <iyy>MEASURED_Iyy</iyy>
        <izz>MEASURED_Izz</izz>
      </inertia>
    </inertial>

### Do NOT touch

- The six rotor links and joints
- `fc_cube_link` (it holds the IMU)
- Either camera link, sensor, or plugin
- The lift-drag plugins
- The ArduPilot plugin block

### Why the `<pose>` line matters

In SDF, the inertia numbers are taken about the link's inertial frame, which sits at
the link origin by default. Our test measures inertia about the **centre of gravity**.
Setting `<pose>` to the measured CG tells Gazebo to place the mass at the right point.
The offset is small, but set it correctly.

### Quick mass check

After the edits, the parts should still add up to the real drone's total weight:

    base_link (measured)  +  Cube (0.075)  +  6 rotors (0.150)  +  ZR10  +  gimbal camera  =  full drone mass

If that matches what the whole drone weighs on a scale, our bookkeeping is right.

---

## 8. What to Expect

- The measured body values should land in the rough range of **a few hundredths of a
  kg·m²** (somewhere around 0.05–0.08 for roll/pitch). The full vehicle, once Gazebo
  re-adds the rotors, Cube and cameras, should come out around **0.08 kg·m²** per
  axis. Exact targets shift a bit depending on where the Jetson and cameras sit, so
  treat these as ballpark, not gospel.
- **Ixx and Iyy should be close to each other** (within a couple of percent). The
  frame is symmetric, so roll and pitch inertia should be nearly equal. If they're
  far apart, suspect a leveling or air-resistance problem in that orientation — not a
  real difference.
- On the full vehicle, **Izz (yaw) should be the largest** of the three. That is
  normal for something wider than it is tall.

If our numbers respect those checks and sit in the right ballpark, we're done.

---

## 9. Conclusion

We hang the body (frame + battery + Jetson + wiring), validate the rig against known
gravity, and twist it to measure Ixx, Iyy and Izz. Those three numbers plus the body
mass and CG go into `base_link`. The props, the Cube and both cameras stay off the
drone during the test and untouched in the file — because they already have their own
links and plugins, and we don't want to count them twice. The result is a simulation
model that matches the real aircraft's mass distribution, so the flight controller we
design in simulation will actually work on the real drone.

---

## 10. Limitations and Things to Watch

- **Hand timing** adds human reaction error. Using the Cube's gyro to read the period
  removes most of it — strongly recommended.
- **Air resistance** matters most in the tipped-over (roll/pitch) orientations where
  the drone has a big surface facing the swing. Test indoors, away from drafts, with
  small swing amplitudes.
- **Leveling** is the big one. If it doesn't hang level, the gravity check drifts off
  9.81 and the result can't be trusted. Level carefully.
- **Static props only.** This measures the still (non-spinning) inertia, which is
  exactly what the rigid-body model needs. The spinning-prop gyroscopic effect is a
  separate thing the simulator already handles — don't mix the two up.
- Camera inertia is small, so a simple box estimate per camera is fine if measuring
  each one individually is a hassle.

---

## 11. Remarks and Recommendations

- **Use the onboard gyro for timing** if we can, it is the single biggest accuracy
  win and fixes the main weakness of hand-stopwatch methods.
- **Repeat each measurement three times** and average. Throw out any run where the
  drone wobbled or a rope slipped.
- **Always do the gravity check first** in each orientation, and never accept an
  inertia value from an orientation whose gravity check failed.
- **Measure in true flight config** — real flight battery, Jetson powered and mounted
  as it flies — so the inertia matches reality.
- Keep one small results table per orientation: `h`, `D`, swing period, computed `g`,
  torsional period, computed `I`. Easy to review, easy to spot a bad run.

---

## Values to Fill In After Testing

- [ ] Body mass (scale reading: frame + battery + Jetson + wiring)
- [ ] Measured Ixx, Iyy, Izz
- [ ] Body CG height
- [ ] Mass + inertia + mount position for the ZR10 camera link
- [ ] Mass + inertia + mount position for the gimbal camera link

