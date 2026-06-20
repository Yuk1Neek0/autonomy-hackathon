# Red Team Hack Sim — Project Plan

## Overview

Autonomous drone that reads visual clues (arrows + spheres), decides the correct route, and
delivers to the right vehicle — all from the FPV camera with no human input.

---

## Part 1 — Vision Engine

The core intelligence. Everything else depends on reading the camera correctly.

**Sub-tasks:**
- **Arrow detector** — HSV mask for green vs red, determine tip direction (left/right) from contour shape or centroid position
- **Sphere counter** — HSV mask for blue, blob detection + area filtering to count 1–5 spheres
- **Vehicle identifier** *(optional)* — visually confirm the target instead of purely trusting the turn legend

**Input:** `read_frame(drone)` → BGR numpy array
**Output:** `turn1: "left"|"right"`, `turn2: "left"|"right"`, `target_vehicle: str`

```python
import cv2, numpy as np

def detect_green_arrow_direction(frame):
    hsv = cv2.cvtColor(frame, cv2.COLOR_BGR2HSV)
    mask = cv2.inRange(hsv, np.array([40, 80, 80]), np.array([80, 255, 255]))
    contours, _ = cv2.findContours(mask, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
    if not contours:
        return None
    m = cv2.moments(max(contours, key=cv2.contourArea))
    cx = m["m10"] / m["m00"]
    return "left" if cx < frame.shape[1] / 2 else "right"

def count_blue_spheres(frame):
    hsv = cv2.cvtColor(frame, cv2.COLOR_BGR2HSV)
    mask = cv2.inRange(hsv, np.array([100, 100, 50]), np.array([130, 255, 255]))
    contours, _ = cv2.findContours(mask, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
    spheres = [c for c in contours if cv2.contourArea(c) > 200]
    count = len(spheres)
    return "left" if count % 2 == 0 else "right"
```

---

## Part 2 — Decision Engine

Pure logic — no vision. Takes vision outputs and decides the target vehicle and delivery method.

| Turn 1 | Turn 2 | Vehicle | Delivery |
|--------|--------|---------|----------|
| Left   | Left   | Tank | Fly into it |
| Left   | Right  | Boat | Fly into it |
| Right  | Left   | Jet  | Fly into it |
| Right  | Right  | Ice-Cream Truck | **Land beside it** (within 50 m) |

Also decides **when to look** — vision only matters at specific waypoints (arrow junction, sphere junction).

---

## Part 3 — Navigation Controller

Flies the drone through the course using the ProjectAirSim API.

**Flight phases:**
1. `takeoff` → fly forward to the arrow room
2. Hover + scan until arrow detected → execute `turn1`
3. Fly forward to the sphere room
4. Hover + scan until spheres detected → execute `turn2`
5. Fly to the vehicle room → approach the correct vehicle
6. If **Ice-Cream Truck**: `land_async()` within 50 m. Otherwise: fly into it.

**Key API calls:**

| Call | Used for |
|------|----------|
| `move_to_position_async(n, e, d, speed)` | Main waypoint navigation (NED coords) |
| `move_by_velocity_body_frame_async(vx, vy, vz, duration)` | Slow forward scan |
| `rotate_by_yaw_rate_async(rate, duration)` | Spin to look around at a junction |
| `hover_async()` | Hold position while vision processes a frame |
| `land_async()` | Ice-Cream Truck delivery |

> **NED reminder:** X = North, Y = East, Z = Down. Climbing = negative Z.
> Drone spawns at origin `(0, -35, -0.1)`.

---

## Part 4 — State Machine

Ties navigation and vision together. Prevents acting on a single bad frame.

```
INIT
  └─► TAKEOFF
        └─► FLYING_TO_ARROWS
              └─► SCANNING_ARROWS        ← hover + read frames
                    └─► TURN_1           ← confident detection (N consecutive frames)
                          └─► FLYING_TO_SPHERES
                                └─► SCANNING_SPHERES     ← hover + read frames
                                      └─► TURN_2
                                            └─► FLYING_TO_TARGET
                                                  └─► DELIVERING
                                                        └─► DONE
```

Each state transitions only when the vision condition is met with **confidence** (e.g. same
result across 5 consecutive frames) — a single noisy frame won't cause a wrong turn.

---

## Part 5 — Dashboard

A live debug overlay on the FPV feed — useful during development and for the demo.

```python
def draw_hud(frame, state, turn1, turn2, target, sphere_count):
    overlay = frame.copy()
    cv2.putText(overlay, f"State:   {state}",        (10, 30),  cv2.FONT_HERSHEY_SIMPLEX, 0.7, (255,255,255), 2)
    cv2.putText(overlay, f"Turn 1:  {turn1}",        (10, 60),  cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0,255,0),     2)
    cv2.putText(overlay, f"Turn 2:  {turn2}",        (10, 90),  cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0,255,0),     2)
    cv2.putText(overlay, f"Target:  {target}",       (10, 120), cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0,200,255),   2)
    cv2.putText(overlay, f"Spheres: {sphere_count}", (10, 150), cv2.FONT_HERSHEY_SIMPLEX, 0.7, (255,100,0),   2)
    cv2.imshow("Red Team HUD", overlay)
    cv2.waitKey(1)
```

Optionally draw contours of detected arrows/spheres on the frame for visual debugging.

---

## Part 6 — Reset & Retry Loop

The puzzle re-randomises on every reset, so the bot must handle multiple attempts.

```python
while True:
    result = await run_mission()
    if result == "PASSED":
        print("Mission complete!")
        break
    print("Failed — retrying...")
    reset(drone)          # teleport back to start, puzzle re-randomises
    await asyncio.sleep(2)  # wait for sim to reinitialise
```

---

## Recommended Build Order

| # | Part | Reason |
|---|------|--------|
| 1 | Vision Engine | Everything else is blocked on this |
| 2 | Decision Engine | Pure logic, trivial once vision works |
| 3 | Navigation (hardcoded waypoints) | Get flying first, tune coords manually |
| 4 | State Machine | Wrap navigation in robust state tracking |
| 5 | Reset Loop | Makes testing fast — one command reruns everything |
| 6 | Dashboard | Nice-to-have for demo / debugging |

---

## File Structure

```
autonomy-hackathon/
├── fly.py                  # main autonomous agent (build here)
├── vision.py               # Part 1 — arrow detector, sphere counter
├── decision.py             # Part 2 — turn logic → target vehicle
├── navigation.py           # Part 3 — waypoint sequences per route
├── redteam_sim.py          # helper: connect(), reset(), read_frame()
├── view_camera.py          # live FPV preview (dev tool)
├── smoke_test.py           # end-to-end setup check
├── sim_config/             # sf_scene.jsonc + sf_robot.jsonc (use as-is)
├── wheels/                 # offline pip packages
├── requirements.txt
└── docs/
    ├── README.md            # briefing (rules)
    ├── ProjectAirSim_API_Guide.md  # API reference
    └── ProjectPlan.md       # this file
```
