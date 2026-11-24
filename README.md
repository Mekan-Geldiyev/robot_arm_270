# Boxing Robot Arm — 270-Degree Rebuild

A webcam tracks your arm with MediaPipe Pose; a Lego + servo arm mirrors it
in real time and can throw jabs, hooks, uppercuts, and overhands - either by
watching you throw one, by you naming which one's coming next, or (new in
this build) by a wrist-worn IMU sensing the strike directly. Built as a demo
for a YC application / robotics portfolio.

**This is the 270-degree rebuild**, a separate build from the original
180-degree `robot_arm` (kept intact elsewhere as a working fallback). New
DS3218/Miuzei 270-degree servos are physically installed, giving real range
the 180-degree hardware's mechanical hard-stops couldn't reach. See
`CLAUDE.md` in this folder for the full build history, open issues, and the
wrist IMU chapter in detail - this README covers setup and day-to-day use.

## Hardware

- Elegoo Uno R3 (Arduino clone), USB to PC
- PCA9685 16-channel PWM servo driver
- 4x DS3218/Miuzei 270-degree servos: pan + tilt (shoulder gimbal), yaw (3rd
  shoulder DOF), elbow - real confirmed centering: pan=145, tilt=120,
  yaw=70, elbow=135
- GY-521/MPU-6050 IMU taped to a wrist brace, read over I2C on the SAME Uno
  as the PCA9685 (0x68 vs 0x40, coexist fine on one bus) - a faster trigger
  for punch detection than the camera pipeline can offer
- Breadboard as the Arduino <-> PCA9685/IMU wiring hub
- 5V 4A power supply -> barrel jack -> screw terminal -> PCA9685 `V+`
- Aluminum pan-tilt bracket, mounted **sideways** so the two servos give
  shoulder-like range for jab / hook / uppercut motion

- Channel 0 = **PAN** — horizontal rotation axis
- Channel 1 = **TILT** — vertical rotation axis — which "height/angle" a
  punch comes from; also reaches the uppercut position
- Channel 2 = **YAW** — rotates the shoulder around the upper-arm axis —
  provides the fast strike sweep
- Channel 3 = **ELBOW** — elbow flex

## Software

- **`track.py`** (shared with the 180-degree build) - the raw MediaPipe
  pose-to-angle geometry (pan/tilt/yaw/elbow formulas, filters). About the
  user's body, not the servo hardware, so it's identical across both
  builds.
- **`interpolation_approach/track_interpolation_270.py`** — the main entry
  point for this hardware. Loads hand-verified `(raw MediaPipe reading,
  correct servo output)` pairs from
  `interpolation_implement/calibration_data_270.json` and maps live input
  to servo output via inverse-distance-weighted interpolation.
- **`punch_classifier.py` + `hook_detector.py`** (shared) — vision-based
  "did a strike just happen + what type" gate.
- **`punch_angle_270.py`** — this build's 4-punch family (tilt+yaw
  combinations), built from `positions_270.POSITIONS`'s real confirmed
  values.
- **`hook_animation_270.py`** — standalone scripted hook-strike animation
  test for this build's yaw/tilt values.
- **`voice_punch_270.py`** — name a punch before throwing it instead of
  relying on automatic classification.
- **`imu_punch_listener.py`** — reads the wrist IMU's `PUNCH,<type>,...`
  serial lines (merged into `robot_arm_270.ino`'s firmware) and exposes the
  most recent call as a pending command, reacting faster than the vision
  gate can.
- **`imu_vision_angle_test.py`** — live-imported by
  `track_interpolation_270.py` for its trigger-routing helpers (which
  detector owns which punch type).
- **`imu_vision_agreement_test.py`** — standalone tool that paired the
  wrist IMU's calls against vision's calls on the same real throws; see
  `imu_vision_agreement_results_2026-11-03.md` for the numbers that
  decided the current trigger-routing split.
- **`imu_angle_map.py` / `imu_angle_test.py`** — Phase B/C scaffolding for
  an abandoned continuous signed-angle approach; not wired into anything
  live (see `CLAUDE.md`).
- **`robot_arm_270.ino`** — Arduino sketch: reads `"pan,tilt,yaw,elbow\n"`
  over serial, smooths motion toward the target angle, drives the PCA9685,
  AND runs the wrist IMU's punch classifier in the same loop.

### Standalone Arduino test sketches

- `circle_test/circle_test.ino` — continuous pan/tilt circle, sanity-checks
  wiring/power in isolation, no serial/Python involved.
- `imu_punch_test/imu_punch_test.ino` — first-pass spike-only punch
  detector (no type classification), superseded by the classify version
  below.
- `imu_punch_classify_test/imu_punch_classify_test.ino` — confirmed-good
  hook/uppercut/punch classifier; this is the logic actually merged into
  `robot_arm_270.ino`.
- `imu_punch_signed_angle_test/imu_punch_signed_angle_test.ino` — the
  shelved continuous signed-angle experiment, kept as a record of a dead
  end, not wired into anything live.

## Install

```
pip install -r requirements.txt
```

Also install the **Adafruit PWM Servo Driver Library** in the Arduino IDE
(Library Manager -> search "Adafruit PWM Servo Driver").

> Note: `mediapipe`'s own package init imports `matplotlib` transitively,
> which Windows **Smart App Control** can block as an unrecognized DLL,
> crashing on `import mediapipe`. `track.py` and `track_interpolation_270.py`
> work around this by stubbing `matplotlib`/`matplotlib.pyplot` in
> `sys.modules` before mediapipe ever gets a chance to import the real ones.

> Note: voice/text-cued punch selection additionally needs `vosk` and
> `PyAudio` (already in `requirements.txt`).

## Running it

1. Plug in the Arduino, upload `robot_arm_270.ino` from the Arduino IDE.
   **Don't upload the 180-degree build's `.ino` to this hardware or vice
   versa** - the servo ranges and the merged IMU firmware are specific to
   this build.
2. Open `track.py` and check `SERIAL_PORT` matches your Arduino's COM port.
3. Run:
   ```
   python interpolation_approach/track_interpolation_270.py
   ```
4. Stand in front of the webcam so your left shoulder, elbow, and hip are
   visible. Wear the wrist IMU brace for faster hook/uppercut detection -
   it calibrates against whatever pose the wrist is resting in right
   before each throw, so no fixed calibration pose is required.
5. A startup prompt (`prompt_startup_config()`) configures punch
   recognition for the session.
6. Press `q` in the video window to quit.

## Known limitations & open questions

See `CLAUDE.md` in this folder for the full build history. Headlines:

- **`HAND_DOWN` locks in slower than it should** when moving quickly from
  `HAND_OUT_SIDE` - not yet root-caused between a raw-signal filter lag and
  a known elbow-reading limitation in `calibration_data_270.json`. Flagged
  priority for the next session as of 2026-11-03.
- **Continuous signed punch angle (uppercut ↔ hook as one continuous
  number) is shelved** after four different signal-processing approaches
  all converged on the same failure mode. Believed to be a real information
  limit of a single wrist IMU's accel/gyro signal, not unfinished tuning -
  see `CLAUDE.md` before re-attempting a variant of what's already been
  tried.
- **The wrist brace's wiring is fragile** (spliced wire segments joined
  with electrical tape) - flagged priority for a mechanical fix.
- **Vision and the wrist IMU are mirror-image good/bad at hook vs.
  uppercut** (87%/22% vs. 30%/95% respectively) - the current system routes
  each punch type to whichever detector is actually good at it rather than
  requiring agreement between them.
