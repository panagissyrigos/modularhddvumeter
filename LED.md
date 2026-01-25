# LED Options and Power Considerations
Low-Power Alternatives for Addressable LED Systems

This document explains realistic power requirements for addressable LEDs, why WS2812B LEDs appear power-hungry, and what lower-power alternatives exist. It also provides practical guidance for building a homelab-friendly HDD activity display without requiring a large dedicated power supply.

---

# 1. Why WS2812B LEDs Are Rated at 60 mA Per Pixel

A WS2812B pixel contains:
- One red LED
- One green LED
- One blue LED
- A constant-current driver IC
- PWM logic and latch circuitry

Each color channel is designed for up to 20 mA at full duty cycle.

Therefore:
20 mA (R) + 20 mA (G) + 20 mA (B) = 60 mA worst-case per pixel

This is a datasheet ceiling, not typical usage.

---

# 2. Why Old LEDs Were 10 mA or Less

Older indicator LEDs:
- Were single-color
- Used a simple resistor
- Typically ran at 2 to 5 mA
- 10 mA was considered bright
- 20 mA was absolute maximum

A full RGB LED back then would also be 60 mA worst-case if all channels were driven at max.

WS2812B is not unusual; it is simply three LEDs plus a driver.

---

# 3. Realistic Current Usage for WS2812B

In real-world patterns:
- Brightness is rarely 100 percent
- Full-white is rarely used
- Only a subset of LEDs are lit at once
- Colors like green or red use only one channel

Typical real usage:
- 5 to 15 mA per pixel average
- 25 percent brightness cap reduces current by 75 percent
- VU meter patterns rarely exceed 2 to 4 A for an 8x16 board

---

# 4. Power Requirements Per Board Type

Assuming WS2812B LEDs:

## Board A (8x8 LED only)
- 64 LEDs
- Worst case: 64 x 60 mA = 3.84 A
- Realistic: 1 to 2 A

## Board B (8x8 + MCP23017)
- Same as Board A plus a few mA for logic
- Worst case: ~3.9 A
- Realistic: 1 to 2 A

## Board C (8x16 + MCP23017)
- 128 LEDs
- Worst case: 128 x 60 mA = 7.68 A
- Realistic: 2 to 4 A

## Board D (8x16 + MCP23017 + ESP32-PICO-D4)
- Same as Board C plus ~0.1 A for ESP32
- Worst case: ~7.8 to 8.0 A
- Realistic: 2 to 4 A

---

# 5. Lower-Power Addressable LED Options

There are no true "low-power WS2812B equivalents" that magically break physics, but several alternatives reduce current significantly.

## 5.1 SK6812 Mini / Mini-E / 2020 Packages
- Smaller LEDs
- Lower forward current (8 to 12 mA per channel)
- Worst case: 24 to 36 mA per pixel
- About half the power of WS2812B

Downside: tiny SMD packages.

---

## 5.2 APA102 / SK9822 (DotStar)
- Not lower power by default
- But have hardware global brightness control
- You can enforce a safe current ceiling
- Predictable maximum current

---

## 5.3 RGBW LEDs (SK6812 RGBW)
- White channel is a single LED
- If you use only white, max is ~20 mA
- Much lower than 60 mA RGB full white

Good for clean activity bars.

---

## 5.4 Single-Color Addressable LEDs
Some addressable LEDs are single-color:
- White
- Amber
- Red
- Blue

These draw:
- ~20 mA max total per pixel
- Not 60 mA

Perfect for NAS activity indicators.

---

## 5.5 Multiplexed LED Matrices (HT16K33, IS31FL3731)
These are the true low-power solution.

Instead of:
- 128 WS2812B LEDs = 7.7 A worst case

You get:
- A matrix driver
- Shared current
- Total draw: 100 to 300 mA for the entire panel

Still supports per-pixel control.

Downside: not individually addressable in the WS2812 sense.

---

# 6. Reducing Power Without Changing LEDs

## 6.1 Brightness Cap
Set global brightness to:
- 32/255 (12 percent)
- 64/255 (25 percent)

This reduces worst-case current proportionally.

Example:
128 LEDs x 60 mA x 0.25 = 1.9 A

---

## 6.2 Avoid Full-White
Use:
- Green
- Yellow
- Red

These use 1 or 2 channels, not 3.

---

## 6.3 Use Fewer LEDs Per Drive
Instead of 8 rows per drive:
- Use 4 rows
- Use 2 rows
- Use 1 pixel per drive

This reduces power by 50 to 87 percent.

---

# 7. Recommended Low-Power Approaches for a NAS Display

## Option A: WS2812B with Brightness Cap
- Easiest
- No hardware changes
- 1.5 to 3 A per 8x16 board

## Option B: SK6812 Mini
- Half the current
- Same protocol

## Option C: Matrix Driver (HT16K33 or IS31FL3731)
- 0.1 to 0.3 A total
- Best for homelab use

## Option D: One Addressable LED Per Drive
- 16 LEDs total
- Worst case: 16 x 60 mA = 960 mA
- Realistic: 200 to 400 mA

---

# 8. Summary

- WS2812B worst-case numbers are scary but unrealistic.
- Real-world usage is far lower.
- Several LED families offer lower power consumption.
- Brightness caps and color choices dramatically reduce current.
- A single 8x16 board can be powered safely from a small 5 V, 3 to 5 A supply.
- For ultra-low power, use matrix drivers or single-color addressable LEDs.

