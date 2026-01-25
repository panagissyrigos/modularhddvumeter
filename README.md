# Modular HDD Activity VU Meter System
ESP32-PICO-D4 + MCP23017 + WS2812B Modular Tile Architecture

This project implements a modular, tile-based HDD activity visualization system using:

- ESP32-PICO-D4 as the master controller
- MCP23017 I/O expanders for HDD LED pulse inputs
- WS2812B or SK6812 addressable LEDs arranged in 8x8 or 8x16 tiles
- Daisy-chainable tiles that expand left and right
- Shared I2C bus and shared interrupt line
- Per-tile HDD LED headers for direct connection to backplanes or HBAs

Each tile represents 16 drives, displayed as 16 vertical VU-meter columns with 8 activity levels (rows).
Tiles can be mixed and matched to build a display of any width.

---

## 1. System Overview

The system is composed of four PCB types, all mechanically and electrically compatible:

| Board Type | LED Matrix | MCP23017 | HDD Headers | ESP32 | Purpose                 |
|-----------:|-----------:|---------:|------------:|------:|-------------------------|
| A          | 8x8        | No       | No          | No    | LED-only expansion tile |
| B          | 8x8        | Yes      | Yes (16)    | No    | Compact input+LED tile  |
| C          | 8x16       | Yes      | Yes (16)    | No    | Wide input+LED tile     |
| D          | 8x16       | Yes      | Yes (16)    | Yes   | Master controller tile  |

All tiles share:

- 5V power rail
- GND
- WS2812 LED data line (in to out)
- I2C bus (SDA and SCL)
- Shared interrupt line (INT)

Tiles connect left-to-right using a 6-pin JST-XH connector.

---

## 2. Electrical Architecture

### 2.1 Power Distribution

- 5V powers all WS2812 LEDs and feeds the 3.3V regulator.
- 3.3V powers:
  - ESP32-PICO-D4
  - MCP23017 expanders

Each tile includes:

- Wide 5V and GND pours
- Local 5V injection pads
- Decoupling capacitors near LED clusters
- 0.1 uF and 10 uF near MCP23017
- 0.1 uF and 10 uF near ESP32 (master tile only)

---

### 2.2 LED Data Chain

WS2812 LEDs are daisy-chained:

ESP32 -> Tile D LEDs -> Tile C LEDs -> Tile B LEDs -> Tile A LEDs -> ...

Each tile has:

- DATA_IN (from left)
- DATA_OUT (to right)
- 330 ohm series resistor on DATA_IN

LED addressing is linear:

- 8x8 tile = 64 LEDs
- 8x16 tile = 128 LEDs

Tile N LED indices:

base  = N * LEDs_per_tile  
pixel = base + (row * columns) + column

---

### 2.3 HDD LED Input Handling

Each input tile (B, C, D) supports 16 HDD LED inputs.

#### Signal characteristics

Motherboard or HBA HDD LED headers typically provide:

- LED+ = +5V (through onboard resistor)
- LED- = transistor sink (pulses low on activity)

#### Level shifting

Each input uses a resistor divider:

HDD_LED+ (5V) -> 10k -> MCP23017 input  
                     |  
                    15k  
                     |  
                    GND  

This produces approximately 3.0V logic-high.

#### MCP23017

- 16 inputs (GPA0 to GPA7, GPB0 to GPB7)
- I2C address set via A0 to A2 pins
- INTA and INTB tied together per tile
- All tile INT lines tied together (open-drain safe)

---

### 2.4 I2C Bus

All MCP23017 expanders share:

- SDA
- SCL
- Pull-ups (4.7k) on master tile only

Each tile sets a unique address:

| Tile   | A2 | A1 | A0 | Address |
|--------|----|----|----|---------|
| Master | 0  | 0  | 0  | 0x20    |
| Next   | 0  | 0  | 1  | 0x21    |
| Next   | 0  | 1  | 0  | 0x22    |
| ...    | .. | .. | .. | ...     |

Up to 8 tiles supported (128 drives).

---

### 2.5 Shared Interrupt Line

All MCP23017 INT pins are open-drain and tied together:

Tile 0 INT -----  
Tile 1 INT -----+---- ESP32 GPIO (INT input)  
Tile 2 INT -----  

- Any tile pulling INT low triggers the ESP32 interrupt.
- ESP32 ISR sets a flag.
- Main loop reads all MCP23017s to clear interrupts.

This guarantees:

- No missed pulses
- No polling
- Perfect scalability

---

## 3. Tile Designs

### 3.1 Board A: 8x8 LED Tile (LED Only)

Contains:

- 8x8 WS2812 LEDs (64 total)
- DATA_IN and DATA_OUT connectors
- 5V and GND rails
- No logic components

Purpose:  
Pure visual expansion.

---

### 3.2 Board B: 8x8 Input + LED Tile

Contains:

- 8x8 LED matrix (64 LEDs)
- 1x MCP23017
- 16 HDD LED headers
- I2C passthrough
- Shared INT line
- DATA_IN and DATA_OUT

Purpose:  
Compact 16-drive display tile.

---

### 3.3 Board C: 8x16 Input + LED Tile

Contains:

- 8x16 LED matrix (128 LEDs)
- 1x MCP23017
- 16 HDD LED headers
- I2C passthrough
- Shared INT line
- DATA_IN and DATA_OUT

Purpose:  
Wide 16-drive display tile.

---

### 3.4 Board D: Master 8x16 Tile (ESP32-PICO-D4)

Contains:

- Everything from Board C
- ESP32-PICO-D4
- 3.3V regulator
- USB-serial interface (CP2102 or CH340)
- Auto-reset circuitry
- Boot and Reset buttons
- I2C pull-ups
- INT pull-up
- LED_DATA_OUT to tile chain
- Auto-tile detection via I2C scanning

Purpose:  
System controller.

---

## 4. ESP32 Firmware Architecture

### 4.1 Interrupt Handling

ESP32 ISR:

void IRAM_ATTR handleInt() {  
    mcpFlag = true;  
}

Main loop:

if (mcpFlag) {  
    mcpFlag = false;  
    for (int i = 0; i < numTiles; i++) {  
        uint16_t state = readMCP(i);  
        processState(i, state);  
    }  
}

Reading each MCP23017 clears its interrupt.

---

### 4.2 Pulse Counting

Each channel has a counter:

uint16_t pulseCount[NUM_CHANNELS];

When a pin changes state, increment the counter.

---

### 4.3 LED Rendering

Every 50 ms:

- Convert pulse count to level (0 to 7).
- Light rows 0 to level-1 in that column.
- Clear pulse count.

Mapping:

int tile = channel / 16;  
int col  = channel % 16;  
int pixel = tileOffset + (row * 16) + col;

---

## 5. Tile-to-Tile Connector

A 6-pin JST-XH is used on both left and right edges.

### Left side (input)

1: GND  
2: 5V  
3: LED_DATA_IN  
4: SDA  
5: SCL  
6: INT  

### Right side (output)

1: GND  
2: 5V  
3: LED_DATA_OUT  
4: SDA  
5: SCL  
6: INT  

Tiles are keyed so they cannot be reversed.

---

## 6. Mechanical Layout

- Tiles mount side-by-side.
- HDD LED headers are on the bottom edge of B, C, and D tiles.
- LED matrices face forward.
- Connectors on left and right edges allow horizontal expansion.

---

## 7. Supported Configurations

### Single Tile

- 16 drives
- 8x16 display

### Master + 1 Tile

- 32 drives
- 8x32 display

### Master + 3 Tiles

- 64 drives
- 8x64 display

### Master + LED-only tiles

- Any width
- Still only 16 drives per input tile

---

## 8. Advantages of This Architecture

- Fully modular
- Scales to 128 drives
- No polling
- No missed pulses
- Clean wiring
- Easy to manufacture
- ESP32-PICO-D4 simplifies RF, flash, and crystal
- MCP23017 simplifies input handling
- WS2812B simplifies LED routing

---

## 9. Future Extensions

- Per-tile EEPROM for metadata (maybe)
- Color-coded drive groups
- Peak-hold bars
- USB or Wi-Fi configuration interface (maybe)
- Logging drive activity to SD card (maybe)

---

