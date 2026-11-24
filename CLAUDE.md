# Boxing Robot Arm - 270-Degree Rebuild

Teleoperated robot arm: a webcam watches the user's real left arm via MediaPipe
Pose, and a servo arm mirrors it in real time. Built as a demo for a YC
application / robotics portfolio - it needs to look controlled on camera, not
janky, so tracking-quality bugs are treated as first-order problems.

**Mission**: the long-term goal is a humanoid fighting robot that mirrors the
user's full-body movement, punches included, in real time. The current
single-arm boxing rig is a step toward that, not the end state.

**This is the 270-degree rebuild (started 2026-10-20), a separate
folder/build from the original 180-degree `robot_arm/`** (kept intact
elsewhere as a clean, working fallback in case this build's parts/print
don't come together in time). New DS3218/Miuzei 270-degree servos are
physically installed; `robot_arm_270.ino` is the active firmware (0-270
range, not 180). Real centering confirmed on the physical build (not
assumed): pan=145, tilt=120, yaw=70, elbow=135 - see `positions_270.py`'s
header for the full direction-per-channel reference.

Everything below that also applied to the 180-degree build still applies
here too, unless noted otherwise: pan/roll mechanical coupling, yaw != palm
roll, the two parallel control approaches, the EFIGHT-lineage punch
detection gate, etc.

## Hardware

Elegoo Uno R3 + PCA9685 (I2C PWM driver) + 4x DS3218/Miuzei 270-degree
servos, arm structure built from Lego (fragile - has broken twice from wire
snags/mishandling).

PCA9685 channels: `0=pan  1=tilt  2=yaw  3=elbow`. Yaw is a 3rd shoulder DOF
added after the original pan/tilt-only build, hand-glued to the outside of
the pan/tilt bracket's metal U.

**Critical mechanical quirk - pan is coupled to roll.** The pan/tilt bracket
is mounted sideways (rotated 90° from a normal camera-mount orientation) to
fake shoulder-like range from just 2 servos. Because of this, and because
pan is the base joint with tilt/yaw/elbow/forearm all mounted rigidly on top
of it, **sweeping pan doesn't just point the arm left/right - it rolls the
entire downstream assembly with it**, flipping the hand's orientation past a
certain angle. This is a hardware design issue, not a software bug.

**Yaw is not palm/wrist roll.** MediaPipe's body-only landmarks can never see
forearm twist (no hand/finger tracking), regardless of camera count. `yaw`
is a proxy signal instead: how far the elbow swings out of the straight-
hanging plane (`yaw_angle()` in `track.py`) - real and useful for
distinguishing a hook from a jab, but a genuinely different physical
quantity from "which way is the palm facing."

## Two parallel control approaches

- **`track.py`** (shared with the 180-degree build) - per-channel formula +
  hand-tuned calibration constants. Elbow uses 2-link IK (law of cosines)
  instead of reading the elbow landmark's angle directly.
- **`interpolation_approach/track_interpolation_270.py`** - the live
  mirroring system for THIS hardware; loads
  `interpolation_implement/calibration_data_270.json` specifically (not the
  180-degree build's `calibration_data.json`). Purely additive on top of
  `track.py` (imports its geometry/filters rather than duplicating them).
  Has its own `SNAP_DISTANCE` (use the single nearest calibration point
  directly when close enough) and a real hold-time requirement
  (`HOLD_TIME_SEC`, lowered from 2.0 to 0.3 on 2026-10-31) on top of that -
  the same point now has to stay the confident nearest match continuously
  for this long before actually locking, not just for one frame, which was
  causing flicker on jittery readings near the snap boundary even while
  holding still.

`calibration_data_270.json` was built by reusing the ORIGINAL 180-degree
build's raw MediaPipe readings (those are about the user's body, unchanged
by the servo swap) paired with freshly hand-jogged servo values - caught
and fixed one real bug this way: `HAND_OUT_CENTER` (reaching forward) and
the T-pose had been silently merged into one entry during the first
rebuild - double check the pose NAME still matches the raw reading's actual
real-world pose, not just that a raw value exists.

**Open issue, user-flagged PRIORITY for next session (2026-11-03)**:
`HAND_DOWN` needs to lock in much faster than it currently does. Lowering
`HOLD_TIME_SEC` wasn't enough on its own. The deeper, not-yet-root-caused
issue: moving quickly from `HAND_OUT_SIDE` toward `HAND_DOWN` (a 77-degree
tilt swing) shows a real lag - the arm appears to pause partway through,
then catches up well after the real motion already happened. Checked the
IDW blend path with a straight-line simulation between the two points and
it transitions smoothly - probably NOT the interpolation math itself. Two
live theories, not yet distinguished: (1) the raw-signal OneEuroFilter's
lag on a large, fast tilt swing specifically, or (2)
`calibration_data_270.json`'s own noted issue - `HAND_DOWN`'s raw elbow
reading (115) doesn't match what a fully-straight hanging arm should read,
a MediaPipe z-depth limitation that no amount of hold-time tuning can fix.
Next diagnostic step, not yet done: log raw vs. smoothed tilt and the
chosen nearest-neighbor name every frame during a real `HAND_OUT_SIDE` ->
`HAND_DOWN` motion, to see directly which stage is actually lagging
instead of guessing further.

Punch recognition is now a startup prompt (`prompt_startup_config()`), not
a fixed constant - see "Wrist IMU punch detector" below for how
hook/uppercut routing actually works now.

## Punch classification + hook animation (`interpolation_approach/`)

`hook_detector.py`/`punch_classifier.py` (shared with the 180-degree build)
watch wrist speed + travel (a reliable "did a strike just happen" gate,
adapted from a separate webcam fighting-game project called EFIGHT) plus
elbow bend and yaw/direction to guess punch *type* (hook/uppercut/punch).
Vision-only type classification has a confirmed real information limit (see
`punch_dataset.py`'s original findings on the 180-degree build) - this is
exactly why the wrist IMU chapter below exists.

`hook_animation_270.py` is a standalone hook animation test (scripted,
stateless per-frame curve), not yet wired into live detection.

## Punch family + voice/text selection (`interpolation_approach/`)

`punch_angle_270.py` is this build's 4-punch family (tilt sets which
height/angle a punch comes from, yaw provides the fast strike sweep),
loading real confirmed values from `positions_270.POSITIONS`.
`voice_punch_270.py` lets the user *name* the punch instead of relying on
automatic classification - same design as the 180-degree build's
`voice_punch.py`, just pointed at `punch_angle_270.PUNCHES`.

## Wrist IMU punch detector (started 2026-10-29)

A GY-521/MPU-6050 taped to the user's left wrist brace, read over I2C on
the SAME Uno as the PCA9685 (shares A4/SDA, A5/SCL - two different I2C
addresses, 0x68 vs 0x40, coexist fine on one bus). Not a second Arduino -
its setup()/loop() is merged into `robot_arm_270.ino`'s own setup()/loop().
**Why this exists**: the camera/MediaPipe pipeline is too slow/laggy to
catch a fast punch specifically (not body movement generally), so the
wrist IMU exists as a faster trigger for that one input class.

Files, "keep the simpler known-good file standing" pattern:
- `imu_punch_test/imu_punch_test.ino` - first version, spike-only
  (magnitude above a threshold = "PUNCH"), no type classification. Has a
  known, not-fixed-there baseline bug: double-cancels gravity in the
  magnitude calc. Didn't block spike detection but don't trust "still
  reads 0" in this file literally.
- `imu_punch_classify_test/imu_punch_classify_test.ino` - confirmed-good
  fallback. Adds hook vs uppercut vs low-rotation "punch" (straight)
  classification plus a diagnostic swing angle, fixes the baseline bug
  above, and prints `PUNCH,<type>,<angle_deg>,<ms_to_recognize>,
  <verticality>,<peak_gyro_dps>` per punch - CSV. Type vocabulary
  deliberately matches `punch_classifier.py`'s own strings. This is the
  classifier that's actually merged into `robot_arm_270.ino`.
- `imu_punch_signed_angle_test/imu_punch_signed_angle_test.ino` - a full
  copy of the above plus a second reference axis and a signed, continuous
  angle. Kept in the repo as a record of a dead end (see below), NOT wired
  into anything live.

**Classifier model**: calibration (holding the wrist still) captures the
raw accel vector as `refUp`. `refUp` is continuously adaptive (not a
one-shot startup calibration) - while idle, it slowly drifts toward the
current accel reading each loop tick (`REFUP_ADAPT_RATE`), so each punch is
judged against whatever orientation the wrist is actually resting in right
before THAT throw. This fixed a real root cause found 2026-10-29: forearm
ROLL (not swing plane) was driving misclassification - palm-up vs.
palm-down on the identical swing flipped the verdict, because a one-shot
`refUp` goes stale the instant the wrist's rest orientation differs from
power-on time.

Classification uses `angleDeg` (accel-direction vs refUp): a hook's net
strike acceleration points roughly across the body (`ANGLE_HOOK_MIN_DEG`,
currently 60), an uppercut's roughly up. Retested with a real mixed batch
of throws (`imu_results_2026-10-29.md`) - uppercuts mostly 6-59 degrees,
hooks mostly 63-101, confirmed "identifies the punch properly."

**Continuous SIGNED angle, attempted 2026-10-30, SHELVED after extensive
real-hardware testing - a likely real information limit, not a tuning
gap.** The goal was a single continuous signed number describing where a
punch falls on a spectrum (uppercut ~= -90 degrees, hook ~= 0 degrees), to
drive tilt/pan continuously instead of snapping to discrete positions - see
`imu_punch_signed_angle_test.ino` for the full implementation history (kept
in the repo; NOT deleted, the dead ends are worth not re-discovering). Four
different feature-engineering attempts (two gyro-verticality variants,
peak-instant accel-direction vs a second axis, window-averaged
accel-direction) all converged on the same failure mode - hooks collapsing
toward the uppercut reading unless thrown in an exaggerated, atypical way -
even after fixing a real confound (fast non-punch movement crossing the
trigger threshold). **How to apply**: don't re-attempt "just try a
different axis/window/threshold" on this same peak-or-averaged-single-IMU-
accel-direction approach without new real-world reasoning - a genuinely
different signal source (second IMU/magnetometer, or full gyro-integration
orientation tracking) would be a real next idea, not a variant of what's
already been tried.

**What remains confirmed-good**: binary hook/uppercut/punch(straight) type
classification via the OLD, UNCHANGED `angleDeg` in
`imu_punch_classify_test.ino`. This binary classifier is the one merged
live into `robot_arm_270.ino` and consumed by
`interpolation_approach/imu_punch_listener.py`.

**IMU-vs-vision agreement test, confirmed AND LIVE (2026-11-03)**: built
`interpolation_approach/imu_vision_agreement_test.py` to pair the wrist
IMU's binary call against `punch_classifier.PunchClassifier`'s call on the
SAME real throws (keyboard-labeled ground truth). 55 real throws total
(`imu_vision_agreement_results_2026-11-03.md`):

| | IMU | vision |
|---|---|---|
| hooks (31 thrown) | 30% correct | 87% correct |
| uppercuts (24 thrown) | 95% correct | 22% correct |

Vision is reliably good at hooks, bad at uppercuts; the wrist IMU is the
mirror image. A naive "both must agree" combined rule scored WORSE than
either signal alone - don't reintroduce that.

**Now live in `track_interpolation_270.py`**: both detectors run every
session, and each punch type is routed to whichever detector is actually
good at it. IMU's `"uppercut"`/`"punch"` calls trigger immediately on their
own; IMU's `"hook"` calls are ignored, falling through to vision. Vision's
`"hook"` calls trigger; vision's `"uppercut"`/`"punch"` calls are ignored
whenever the brace is on and only used as a fallback with no brace at all.
See the trigger-priority block in `main()`'s loop. Confirmed working well
on real mixed hook/uppercut throws, 2026-11-03.

**Open issue, user-flagged PRIORITY for next session (2026-11-03)**: the
wrist brace's wiring is still fragile - each connection from the breadboard
to the MPU-6050 taped on the brace is currently 3 separate wire segments
joined with electrical tape (a "dead-flat-zero on every axis" wiring fault
has already happened once from this exact setup). Plan: extend each
connection by one more wire and tape it down tightly, then re-test
calibration/classification afterward - a mechanical fix, not a code one.

Phase B (`interpolation_implement/imu_angle_calibration.json`,
`interpolation_approach/imu_angle_map.py`,
`interpolation_approach/imu_angle_test.py`) and Phase C (live wiring into
`robot_arm_270.ino`/`track_interpolation_270.py`) were never started - left
in the repo as scaffolding from the original 3-phase plan, built on the
now-abandoned signed-angle approach, not deleted but not functional toward
any current goal either.

## Other standalone Arduino test sketches

- `circle_test/circle_test.ino` - no serial/Python involved. Traces a
  continuous pan/tilt circle to sanity-check the hardware
  (PCA9685/servos/wiring/power) in isolation. Only run on this 270-degree
  build so far.

## Running it

```
pip install -r requirements.txt
python track.py                                            # formula-based, shared with the 180-degree build
python interpolation_approach/track_interpolation_270.py   # interpolation-based, this hardware
```

Upload `robot_arm_270.ino` to the Arduino. `SERIAL_PORT` in `track.py`
(shared by everything above) needs to match the Arduino's actual COM port.
Don't upload a 180-degree `.ino` to this hardware or vice versa - uploading
the wrong one for the physically-connected servos is a real, easy-to-make
mistake, not just a config error.
