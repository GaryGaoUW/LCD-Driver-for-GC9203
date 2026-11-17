# Bare-Metal GC9203 LCD Driver

A C++ driver for the GC9203 LCD controller, written from the datasheet with no graphics library and tested on an ESP8266. It provides pixel, line, rectangle and fill primitives on top of a compact, partially refreshed framebuffer.

| | |
|---|---|
| **MCU** | ESP8266 (hardware SPI, 26 MHz) |
| **Display** | GC9203, 128 x 220 panel |
| **Language** | C++ (Arduino) |
| **Hardware** | Custom ESP8266 breakout board (Altium Designer, fabricated by JLCPCB) |
| **Debug** | Logic analyzer on SCK, MOSI, CS, DC |

## Bring-up

Every SPI word is 16 bits. With DC low, the word selects a register. With DC high, it writes that register's value or pixel data.

The display stayed blank on the first attempt. A logic analyzer on SCK, MOSI, CS and DC showed that the command timing and framing did not match the datasheet. Comparing the captures with the datasheet timing diagrams fixed the initialization sequence.

Power-up order: reset low 10 ms, high 120 ms, vendor unlock, configure, power on, wait 120 ms, display on.

## Address window

The GC9203 addresses 176 columns, and the 128-pixel panel sits in the middle, so every column gets a base offset of 24. Rotation changes the entry mode (`R03h`) and swaps x and y in the window.

## Saving RAM

A full RGB565 framebuffer (128 x 220 x 2 B = 56,320 B) does not fit comfortably on the ESP8266. Instead:

- **1 byte per pixel:** a 7-bit 2:3:2 colour index plus a dirty flag (28,160 B).
- **256-entry LUT:** expands each index to RGB565 on the way out, replicating bits so full scale maps to full scale.
- **Dirty-row bitmap (28 B):** `display()` skips clean rows entirely.
- **Burst writes:** each dirty run in a row is sent as one address window and one continuous SPI burst. Runs closer than 12 pixels are merged to avoid extra window setup.

## Register map

| Reg | Name | Value | Purpose |
|---|---|---|---|
| `FEh`, `FFh` | Vendor unlock | none | Sent first, before any other register |
| `01h` | Driver output control | `0x001C` | Output setup, from the datasheet |
| `03h` | Entry mode | `0x1000` to `0x1030` | GRAM write direction, set per rotation |
| `05h` | SPI 2-data | `0x0000` | Single-line SPI |
| `07h` | Display control 1 | `0x0013` | Display on, written last |
| `08h` | Display control 2 | `0x0404` | Front and back porch |
| `0Fh` | Oscillator control | `0x0701` | Oscillator setup |
| `10h` | Power control 1 | `0x0000` | Exit standby |
| `11h` | Power control 2 | `0x1000` | Auto boost, then wait 120 ms |
| `30h` | Gate scan control | `0x0000` | Gate scan setup |
| `36h` to `39h` | Window EC, SC, EP, SP | per draw | GRAM address window |
| `22h` | GRAM write | none | Pixel data follows |

## Usage

Copy `GC9203_Driver.h` and `GC9203_Driver.cpp` into your sketch folder. See [example.ino](example.ino) for a full demo.

```cpp
#include "GC9203_Driver.h"

// ESP8266 hardware SPI: SCK = GPIO14, MOSI = GPIO13
GC9203_Driver lcd(14, 13, /*cs=*/15, /*dc=*/4, /*rst=*/5);

void setup() {
  lcd.begin();
}

void loop() {
  lcd.fillRect(20, 20, 50, 50, COLOR_RED);
  lcd.drawLine(0, 0, GC9203_Driver::WIDTH - 1, GC9203_Driver::HEIGHT - 1, COLOR_GREEN);
  lcd.display(); // push only the dirty pixels
}
```

Colours are 7-bit 2:3:2 R:G:B values; `COLOR_*` constants are defined in the header.

## API

| Function | Description |
|---|---|
| `begin()` | Reset and initialise the controller, then clear the screen |
| `setRotation(r)` | Orientation 0 to 3 |
| `drawPixel`, `drawLine`, `drawRect`, `fillRect`, `fillScreen`, `clearDisplay` | Draw into the framebuffer |
| `display()` | Flush dirty pixels to the panel |

