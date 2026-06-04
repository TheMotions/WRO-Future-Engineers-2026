<div align="center">

<img src="https://github.com/TheMotions.png" width="150" style="border-radius:50%" alt="The Motions"/>

# The Motions
### Autonomous Self-Driving Vehicle — WRO Future Engineers 2026
Kuwait National Qualifier

![Arduino](https://img.shields.io/badge/Arduino-Uno-00979D?style=flat-square&logo=arduino&logoColor=white)
![Language](https://img.shields.io/badge/Language-C%2FC%2B%2B-A97BFF?style=flat-square&logo=c%2B%2B&logoColor=white)
![WRO](https://img.shields.io/badge/WRO-Future%20Engineers-6A0DAD?style=flat-square)
![Status](https://img.shields.io/badge/Status-Competition%20Ready-success?style=flat-square)
![Kuwait](https://img.shields.io/badge/Country-Kuwait-007A3D?style=flat-square)

*Two ultrasonic eyes, one vision camera, and a tightly tuned PD loop — three laps, zero contact.*

</div>

---

## Table of Contents

| | | |
|---|---|---|
| [Overview](#project-overview) | [Hardware](#hardware--components) | [Electronics](#electrical-system) |
| [Software](#software-architecture) | [Wall Following](#wall-following-controller) | [Vision & Obstacles](#obstacle-detection--avoidance) |
| [3D Design](#3d-design) | [Photos](#robot-photos) | [Testing](#testing--calibration) |
| [Results](#results) | [Video](#video-demonstrations) | [Team](#team) |

---

## Project Overview

The Motions fields a compact, fully autonomous 1:28-scale vehicle for the **WRO Future Engineers 2026** category. On a closed track seeded with randomly placed colored pillars, the car must drive three uninterrupted laps — centering itself between the walls and obeying each pillar's pass-side rule — entirely on its own.

The design philosophy is deliberately lean: **two ultrasonic sensors** for wall geometry, **one Pixy2 camera** for color decisions, and a single Arduino Uno running a carefully damped PD controller. No excess sensors, no excess weight — just a clean, repeatable control loop.

```
┌──────────────────────────────────────────────────────────┐
│                 Arduino Uno  (ATmega328P)                 │
│                                                          │
│   HC-SR04 Left  ─┐                                       │
│                  ├─→  PD core  ──→  Mode manager ──┐     │
│   HC-SR04 Right ─┘                                  │     │
│                                                     ▼     │
│   Pixy2 camera ───────────────────────→  Servo (A0)      │
│                                          Cytron MD13S ──→ DC motor
└──────────────────────────────────────────────────────────┘
```

Two cooperating behaviors share that core:

- **PD mode** — ultrasonic wall-centering, active by default.
- **Pixy mode** — vision-driven pillar avoidance, engaged the moment a trained color is confidently seen.

---

## Hardware & Components

| # | Component | Specification | Function |
|---|-----------|---------------|----------|
| 1 | Microcontroller | Arduino Uno (ATmega328P, 16 MHz) | Central processing & I/O |
| 2 | Motor Driver | Cytron MD13S | Speed & direction control |
| 3 | Drive Motor | Brushed DC — 7.4 V | Rear-wheel propulsion |
| 4 | Steering Servo | Standard 180° servo | Front Ackermann steering |
| 5 | Distance Sensors | HC-SR04 × 2 | Left & right wall detection |
| 6 | Vision Sensor | Pixy2 (SPI) | Color pillar recognition |
| 7 | Battery | 7.4 V LiPo 2S | Full-system power |
| 8 | Chassis | WLtoys 284010 (1:28) | Compact RC platform |

> **Envelope** — ~17 × 9 × 7 cm and ~0.5 kg, comfortably inside the 30 × 20 × 30 cm / 1.5 kg limits.

<div align="center">
<img width="500" src="https://github.com/user-attachments/assets/9c7952be-69de-42f3-882a-6eecc17a7bac" alt="Component layout"/>
<br/><sub>Robot component layout</sub>
</div>

---

## Electrical System

```
LiPo 7.4 V
    ├──→ Cytron MD13S ──→ DC motor
    └──→ Arduino Vin
              └──→ 5 V rail ──→ Servo · HC-SR04 ×2 · Pixy2
```

<details>
<summary><b>Full pin mapping</b> (click to expand)</summary>

<br/>

| Component | Signal | Pin | Notes |
|-----------|--------|-----|-------|
| HC-SR04 Left | TRIG / ECHO | D4 / D5 | trigger pulse · echo timing |
| HC-SR04 Right | TRIG / ECHO | D2 / D9 | trigger pulse · echo timing |
| Steering Servo | SIG | A0 | PWM steering output |
| Cytron MD13S | PWM / DIR | D3 / D8 | speed · direction |
| Pixy2 Camera | CS · MOSI · MISO · SCK | D10 · D11 · D12 · D13 | SPI bus |

</details>

📄 **[Full wiring diagram (PDF)](Schemes/Schematic_Wiring_Diagram.pdf)**

<div align="center">
<img width="500" src="https://github.com/user-attachments/assets/6392126d-c2e6-4422-b1bd-10d1b5243850" alt="Lab setup"/>
<br/><sub>Development tools and lab setup</sub>
</div>

---

## Software Architecture

The firmware is written in **C/C++** on the Arduino IDE. Two sketches share one PD core:

| Module | File | Description |
|--------|------|-------------|
| Open Challenge | `Open_Challenge.ino` | pure wall-following — PD + anti-zigzag |
| Obstacle Challenge | `Obstacle_Challenge.ino` | wall-following + Pixy2 mode switching |

Each loop iteration runs the same nine steps:

```
1.  read HC-SR04 left + right  (3-sample average each)
2.  clamp readings to MAX_VALID_CM = 75 cm
3.  error = leftDist − rightDist
4.  if |error| < 0.8 cm  →  error = 0          (deadband)
5.  derivative = LPF(error − lastError),  α = 0.45
6.  output = KP·error + KD·derivative
7.  targetAngle = CENTER_ANGLE + output
8.  servoAngle = LPF(targetAngle),  α = 0.55
9.  if |servoAngle − CENTER| > 28°  →  ease off the throttle
```

<details>
<summary><b>Tuned parameters</b> (click to expand)</summary>

<br/>

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `KP` | 0.10 | proportional gain — centering strength |
| `KD` | 0.09 | derivative gain — oscillation damping |
| `MOTOR_SPEED` | 55 | cruise PWM (0–255) |
| `TURN_SPEED` | 45 | reduced speed on sharp turns |
| `CENTER_ANGLE` | 90° | servo straight-ahead |
| `ERROR_DEADBAND` | ±0.8 cm | noise suppression band |
| `DERIV_ALPHA` | 0.45 | derivative low-pass coefficient |
| `SERVO_ALPHA` | 0.55 | servo output smoother |
| `MAX_VALID_CM` | 75 cm | corner-spike rejection |
| `HARD_TURN_DEG` | 28° | speed-reduction trigger |

</details>

---

## Wall-Following Controller

A short straight is easy; a clean *lap* is not. Six anti-zigzag measures keep the car glued to the centerline:

| Measure | How | Why it matters |
|---------|-----|----------------|
| Error deadband | ignore ±0.8 cm | kills micro-jitter from sensor noise |
| Derivative LPF | α = 0.45 | smooths spike-driven corrections |
| Servo smoothing | α = 0.55 | removes mechanical oscillation |
| Corner clamping | cap at 75 cm | rejects false long-range echoes |
| Adaptive speed | slow past 28° | prevents overshoot in curves |
| Sensor fallback | mirror the good side | survives a single dropped sensor |

---

## Obstacle Detection & Avoidance

The Pixy2 is trained on two color signatures under arena lighting. Each pillar dictates which side to pass on:

| Pillar | Signature | WRO rule | Response |
|--------|-----------|----------|----------|
| 🟢 Green | Sig 1 | pass on the **left** | steer left, servo > 90° |
| 🔴 Red | Sig 2 | pass on the **right** | steer right, servo < 90° |

<details>
<summary><b>Steering lookup by pillar position</b> (click to expand)</summary>

<br/>

| Color | X &lt; 120 px | X 120–170 px | X &gt; 170 px |
|-------|-----------|--------------|-----------|
| 🟢 Green | hard left — 135° | medium left — 120° | soft left — 105° |
| 🔴 Red | soft right — 75° | medium right — 60° | hard right — 45° |

</details>

The handoff between wall-following and vision is governed by a hysteresis manager so the two never fight:

```
detection filter :  area ≥ 200 px²  ·  X within 20–300 px
enter Pixy mode  :  2 consecutive valid frames
exit to PD mode  :  3 consecutive misses
minimum hold     :  280 ms
servo slew limit :  6° per step
```

---

## 3D Design

The car rides on a WLtoys 284010 (1:28-scale) RC base, fitted with custom 3D-printed mounts for every board and sensor.

> **Print settings** — PLA · 20% infill · 0.2 mm layers.

<div align="center">
<img width="500" src="https://github.com/user-attachments/assets/450ecdc3-dc6c-4dc1-82c0-f9bb397f3465" alt="3D design"/>
<br/><sub>3D-printed electronics tray and sensor mounts</sub>
</div>

Printable files live in [`Models/`](Models/).

---

## Robot Photos

<div align="center">
<table>
  <tr>
    <td align="center"><img width="230" src="https://github.com/user-attachments/assets/39ed9b05-d331-49be-9855-8c95ed0cba1f"/><br/><sub><b>Front</b></sub></td>
    <td align="center"><img width="230" src="https://github.com/user-attachments/assets/7933ec02-1d54-4cbe-b85c-87f1342e2a5f"/><br/><sub><b>Back</b></sub></td>
    <td align="center"><img width="230" src="https://github.com/user-attachments/assets/9daa717b-47a3-42c6-bf30-d1f42c404ae9"/><br/><sub><b>Left</b></sub></td>
  </tr>
  <tr>
    <td align="center"><img width="230" src="https://github.com/user-attachments/assets/b40a195f-4771-49b5-b7de-987e301bb243"/><br/><sub><b>Right</b></sub></td>
    <td align="center"><img width="230" src="https://github.com/user-attachments/assets/e34e684f-ff2f-4c06-8861-a87f1982d041"/><br/><sub><b>Top</b></sub></td>
    <td align="center"><img width="230" src="https://github.com/user-attachments/assets/6788e1ae-b306-427e-af6d-16fb11b72219"/><br/><sub><b>Bottom</b></sub></td>
  </tr>
</table>
</div>

---

## Testing & Calibration

| Test | Method | Outcome |
|------|--------|---------|
| Ultrasonic accuracy | ruler comparison at known distances | `MAX_VALID_CM` set to 75 cm |
| Servo center | straight-line drive + visual check | `CENTER_ANGLE` = 90° |
| Motor dead zone | minimum-PWM sweep | `MOTOR_SPEED` = 55 |
| PD field tuning | iterative KP/KD on track | KP = 0.10, KD = 0.09 |
| Pixy2 training | capture signatures under arena light | Sigs 1 & 2 trained |
| Anti-zigzag | α-coefficient sweep | oscillation removed |
| 3-lap validation | full WRO-spec run | ✅ completed |
| Pillar avoidance | every color × position | ✅ no collisions |

---

## Results

| Metric | Result |
|--------|--------|
| Three-lap completion | ✅ consistent lap times |
| Wall-centering deviation | < 2 cm on straights |
| Pillar recognition | > 95% under arena lighting |
| Obstacle avoidance | ✅ smooth, zero contact |
| Mode switching | ✅ clean PD ↔ Pixy handoffs |

---

## Video Demonstrations

| Challenge | Description | File |
|-----------|-------------|------|
| Open Challenge | three-lap autonomous wall-following | [`videos/Open_Challenge.mp4`](videos/) |
| Obstacle Challenge | full pillar detection + avoidance | [`videos/Obstacle_Challenge.mp4`](videos/) |

---

## Team in Action

<div align="center">
<table>
  <tr>
    <td align="center"><img width="300" src="docs/team_photos/team_1.jpg"/></td>
    <td align="center"><img width="240" src="docs/team_photos/team_2.jpg"/></td>
    <td align="center"><img width="240" src="docs/team_photos/team_3.jpg"/></td>
  </tr>
  <tr><td colspan="3" align="center"><sub>The Motions at the AUM robotics lab — coding, tuning, and testing on the arena.</sub></td></tr>
</table>
</div>

---

## Team

<div align="center">

| Member | Responsibility |
|--------|----------------|
| **Abdullah Al-Otaibi** · عبدالله العتيبي | Hardware design · System integration |
| **Daoud Al-Aneizi** · داود العنيزي | Software development · PD controller |
| **Yousef Al-ostath** · يوسف الاستاد | Vision system · Testing & calibration |

</div>

---

<div align="center">

**The Motions · Digital Innovations**
*Kuwait · WRO Future Engineers 2026*

</div>
