# Electrical Schematic — The Motions

📄 **[Schematic_Wiring_Diagram.pdf](Schematic_Wiring_Diagram.pdf)** — Full Fritzing wiring diagram

## Wiring Summary

| Component | Pin(s) | Notes |
|-----------|--------|-------|
| HC-SR04 Left | TRIG→D4, ECHO→D5 | 5V powered |
| HC-SR04 Right | TRIG→D2, ECHO→D9 | 5V powered |
| Servo | SIG→A0 | 5V powered |
| Cytron MD13S | PWM→D3, DIR→D8 | Motor driver |
| Pixy2 | CS→D10, MOSI→D11, MISO→D12, SCK→D13 | SPI, 5V powered |

## Power Distribution

```
LiPo 7.4V ──→ Cytron MD13S  (motor power)
          └──→ Arduino Vin
                   └──→ 5V pin ──→ Servo / Ultrasonics / Pixy2
```
