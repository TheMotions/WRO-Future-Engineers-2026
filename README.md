<div align="center">

<img src="https://github.com/TheMotions.png" width="340" style="border-radius:50%" alt="The Motions Logo"/>

# The Motions — WRO Future Engineers 2026

### Kuwait National Qualifier &nbsp;·&nbsp; May 31, 2026

![Arduino](https://img.shields.io/badge/Arduino-Uno-00979D?style=flat-square&logo=arduino&logoColor=white)
![Language](https://img.shields.io/badge/Language-C%2FC%2B%2B-A97BFF?style=flat-square&logo=c%2B%2B&logoColor=white)
![WRO](https://img.shields.io/badge/WRO-Future%20Engineers-6A0DAD?style=flat-square)
![Status](https://img.shields.io/badge/Status-Competition%20Ready-success?style=flat-square)
![Kuwait](https://img.shields.io/badge/Country-Kuwait-007A3D?style=flat-square)

</div>

---

## Table of Contents

- [Project Overview](#project-overview)
- [Repository Structure](#repository-structure)
- [Hardware & Components](#hardware--components)
- [Electrical System](#electrical-system)
- [Software Architecture](#software-architecture)
- [Wall-Following Controller](#wall-following-controller)
- [Obstacle Detection & Avoidance](#obstacle-detection--avoidance)
- [3D Design](#3d-design)
- [Robot Photos](#robot-photos)
- [Testing & Calibration](#testing--calibration)
- [Results](#results)
- [Video Demonstrations](#video-demonstrations)
- [Team](#team)

---

## Project Overview

**The Motions** is a Kuwaiti robotics team competing in the **WRO Future Engineers 2026** challenge. Our autonomous vehicle completes three laps on a closed track with randomly placed colored traffic pillars, using real-time sensor fusion between ultrasonic distance sensors and a Pixy2 color vision system.

### System Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Arduino Uno (ATmega328P)            │
│                                                     │
│  HC-SR04 Left ──→ PD Controller ──→ Servo (A0)      │
│  HC-SR04 Right ─┘      ↑                            │
│                   Mode Manager                       │
│  Pixy2 Camera ─────────┘         Cytron MD13S        │
│                                   └──→ DC Motor      │
└─────────────────────────────────────────────────────┘
```

The system runs two interleaved control modes:
- **PID Mode** — Ultrasonic-driven wall centering
- **Pixy Mode** — Vision-driven pillar avoidance

---

## Repository Structure

```
WRO-Future-Engineers-2026/
│
├── src/
│   ├── Open_Challenge/
│   │   └── Open_Challenge.ino         ← Wall-following PD controller
│   └── Obstacle_Challenge/
│       └── Obstacle_Challenge.ino     ← PD + Pixy2 vision avoidance
│
├── Vehicle_Photos/                    ← 6-angle robot documentation
├── Models/                            ← 3D printed parts (.3mf files)
├── Schemes/                           ← Wiring diagram & schematic
├── videos/                            ← Demo videos
└── docs/                              ← Assets & diagrams
```

---

## Hardware & Components

| # | Component | Specification | Function |
|---|-----------|---------------|----------|
| 1 | Microcontroller | Arduino Uno (ATmega328P, 16 MHz) | Central processing & I/O |
| 2 | Motor Driver | Cytron MD13S | Speed & direction control |
| 3 | Drive Motor | Brushed DC — 7.4V | Rear-wheel propulsion |
| 4 | Steering Servo | Standard 180° servo | Front Ackermann steering |
| 5 | Distance Sensors | HC-SR04 × 2 | Left & right wall detection |
| 6 | Vision Sensor | Pixy2 Camera (SPI mode) | Color pillar recognition |
| 7 | Battery | 7.4V LiPo 2S | Full system power supply |
| 8 | Chassis | WLtoys 284010 (1:28 scale) | Compact RC platform |

**Physical Dimensions:**
- 📐 Size: ~17 × 9 × 7 cm *(within WRO 30×20×30 cm limit ✅)*
- ⚖️ Mass: ~0.5 kg *(within 1.5 kg limit ✅)*

<div align="center">
<img width="500" src="https://github.com/user-attachments/assets/9c7952be-69de-42f3-882a-6eecc17a7bac" />
" alt="Component Layout"/>
<br/><sub>Robot component layout</sub>
</div>

---

## Electrical System

### Power Distribution

```
LiPo 7.4V
    │
    ├──→ Cytron MD13S  ──→  DC Motor
    │
    └──→ Arduino Vin
              │
              └──→ 5V Rail ──→ Servo
                           ──→ HC-SR04 (×2)
                           ──→ Pixy2 Camera
```

### Pin Mapping

| Component | Signal | Arduino Pin | Notes |
|-----------|--------|-------------|-------|
| HC-SR04 Left | TRIG | D4 | 10µs pulse trigger |
| HC-SR04 Left | ECHO | D5 | Return pulse timing |
| HC-SR04 Right | TRIG | D2 | 10µs pulse trigger |
| HC-SR04 Right | ECHO | D9 | Return pulse timing |
| Servo Motor | SIG | A0 | PWM steering output |
| Cytron MD13S | PWM | D3 | Motor speed control |
| Cytron MD13S | DIR | D8 | Motor direction |
| Pixy2 Camera | CS | D10 | SPI chip select |
| Pixy2 Camera | MOSI | D11 | SPI data out |
| Pixy2 Camera | MISO | D12 | SPI data in |
| Pixy2 Camera | SCK | D13 | SPI clock |

📄 **[Full Wiring Diagram (PDF)](Schemes/Schematic_Wiring_Diagram.pdf)**

<div align="center">
<img width="500" src="https://github.com/user-attachments/assets/b750eb5a-1c5d-4c75-9160-fdc2dd91ea35" alt="Lab Setup"/>
<br/><sub>Development tools and lab setup</sub>
</div>

---

## Software Architecture

All code is written in **C/C++** using the Arduino IDE. Two sketch modules share a common PD core:

| Module | File | Description |
|--------|------|-------------|
| Open Challenge | `Open_Challenge.ino` | Pure wall-following with PD + anti-zigzag |
| Obstacle Challenge | `Obstacle_Challenge.ino` | Wall-following + Pixy2 mode switching |

### PD Control Loop

```
Every loop iteration:
  1. Read HC-SR04 Left + Right (3-sample average each)
  2. Clamp readings to MAX_VALID_CM (75 cm)
  3. error = leftDist - rightDist
  4. If |error| < DEADBAND (0.8 cm) → error = 0
  5. derivative = LPF( error - lastError )   α = 0.45
  6. output = KP × error + KD × derivative
  7. targetAngle = CENTER_ANGLE + output
  8. servoAngle = LPF( targetAngle )         α = 0.55
  9. If |servoAngle - CENTER| > 28° → reduce motor speed
```

### Tuned Parameters

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `KP` | 0.10 | Proportional gain — wall centering strength |
| `KD` | 0.09 | Derivative gain — damping oscillation |
| `MOTOR_SPEED` | 55 | Cruise PWM (0–255 scale) |
| `TURN_SPEED` | 45 | Reduced speed on sharp turns |
| `CENTER_ANGLE` | 90° | Servo straight-ahead position |
| `ERROR_DEADBAND` | ±0.8 cm | Noise suppression band |
| `DERIV_ALPHA` | 0.45 | Derivative low-pass filter coefficient |
| `SERVO_ALPHA` | 0.55 | Servo output smoother coefficient |
| `MAX_VALID_CM` | 75 cm | Corner spike rejection threshold |
| `HARD_TURN_DEG` | 28° | Speed reduction trigger angle |

---

## Wall-Following Controller

Anti-zigzag techniques implemented to ensure smooth, stable navigation:

| Technique | Implementation | Effect |
|-----------|---------------|--------|
| **Error deadband** | Ignore errors ±0.8 cm | Eliminates sensor noise jitter |
| **Derivative LPF** | α = 0.45 exponential filter | Smooths spike-driven corrections |
| **Servo smoothing** | α = 0.55 exponential filter | Removes mechanical oscillation |
| **Corner clamping** | Max 75 cm reading | Rejects false corner wall echoes |
| **Adaptive speed** | Slow at >28° turn angle | Prevents overshoot on curves |
| **Sensor fallback** | Mirror opposite side | Handles single-sensor failure |

---

## Obstacle Detection & Avoidance

The Pixy2 camera is trained on two color signatures under WRO arena lighting:

| Pillar | Signature | WRO Rule | Response |
|--------|-----------|----------|----------|
| 🟢 Green | Sig 1 | Pass on LEFT | Steer left (>90°) |
| 🔴 Red | Sig 2 | Pass on RIGHT | Steer right (<90°) |

### Steering Lookup Table

| Color | X < 120 px | X 120–170 px | X > 170 px |
|-------|-----------|--------------|-----------|
| 🟢 Green | Hard Left — 135° | Medium Left — 120° | Soft Left — 105° |
| 🔴 Red | Soft Right — 75° | Medium Right — 60° | Hard Right — 45° |

### Hysteresis Mode Manager

```
Detection filter:   min area 200 px²  ·  X range 20–300 px
Enter Pixy Mode:    2 consecutive valid frames
Exit to PID Mode:   3 consecutive missed frames
Minimum hold time:  280 ms  (prevents rapid switching)
Slew rate limit:    6°/step (smooth servo transitions)
```

---

## 3D Design

The robot is built on the WLtoys 284010 (1:28 scale) RC platform with custom 3D-printed mounts for all electronics and sensors.

**Print Settings:** PLA · 20% infill · 0.2 mm layer height

<div align="center">
<img width="500" src="https://github.com/user-attachments/assets/450ecdc3-dc6c-4dc1-82c0-f9bb397f3465" />

<br/><sub>3D printed electronics tray and sensor mounts</sub>
</div>

3D files: [`<img width="1195" height="896" alt="image" src="https://github.com/user-attachments/assets/deec2a5c-afaf-428c-babc-f4ac97a8620d" />
`](Models/)

---

## Robot Photos

<div align="center">
<table>
  <tr>
    <td align="center"><img width="230" src="https://github.com/user-attachments/assets/39ed9b05-d331-49be-9855-8c95ed0cba1f" 
"/><br/><sub><b>Front</b></sub></td>
    <td align="center"><img width="230" src="https://github.com/user-attachments/assets/7933ec02-1d54-4cbe-b85c-87f1342e2a5f" 
                           
<br/><sub><b>Back</b></sub></td>
    <td align="center"><img width="230" src="https://github.com/user-attachments/assets/9daa717b-47a3-42c6-bf30-d1f42c404ae9" 
    />
    
<br/><sub><b>Left</b></sub></td>
  </tr>
  <tr>
    <td align="center"><img width="230" src="https://github.com/user-attachments/assets/b40a195f-4771-49b5-b7de-987e301bb243" />
<br/><sub><b>Right</b></sub></td>
    <td align="center"><img width="230" src="https://github.com/user-attachments/assets/e34e684f-ff2f-4c06-8861-a87f1982d041" />
<br/><sub><b>Top</b></sub></td>
    <td align="center"><img width="230" src="https://github.com/user-attachments/assets/6788e1ae-b306-427e-af6d-16fb11b72219" />
<br/><sub><b>Bottom</b></sub></td>
  </tr>
</table>
</div>

---

## Testing & Calibration

| Test | Method | Outcome |
|------|--------|---------|
| Ultrasonic accuracy | Ruler comparison at known distances | Tuned MAX_VALID_CM = 75 cm |
| Servo center | Straight-line drive & visual alignment | CENTER_ANGLE = 90° |
| Motor dead zone | Minimum effective PWM sweep | Confirmed MOTOR_SPEED = 55 |
| PD field tuning | Iterative KP/KD adjustment on track | KP=0.10, KD=0.09 |
| Pixy2 training | Arena lighting color signature capture | Sigs 1 & 2 trained |
| Anti-zigzag | DERIV_ALPHA + SERVO_ALPHA sweep | Oscillation eliminated |
| 3-lap validation | Full run on WRO-spec track | ✅ Completed |
| Pillar avoidance | All color × position combinations | ✅ No collisions |

---

## Results

| Metric | Result |
|--------|--------|
| 3-lap completion | ✅ Consistent lap times |
| Wall-centering deviation | < 2 cm on straight segments |
| Pillar recognition accuracy | > 95% under arena lighting |
| Obstacle avoidance | ✅ Smooth — zero collisions |
| Mode switching | ✅ Clean PID ↔ Pixy transitions |

---

## Video Demonstrations

| Challenge | Description | File |
|-----------|-------------|------|
| Open Challenge | 3-lap autonomous wall-following | [`videos/Open_Challenge.mp4`](videos/) |
| Obstacle Challenge | Full pillar detection + avoidance | [`videos/Obstacle_Challenge.mp4`](videos/) |

---

## Team

<div align="center">

| Name | Role |
|------|------|
| **Abdullah Al-Otaibi** — عبدالله العتيبي | Hardware design · System integration |
| **Daoud Al-Aneizi** — داود العنيزي | Software development · PD controller |
| **Yousef Al-ostath** — يوسف الاستاد | Vision system · Testing & calibration |

</div>

---

<div align="center">

**The Motions — Digital Innovations**

*Kuwait · WRO Future Engineers 2026*

</div>
