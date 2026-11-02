# LCD-TAP for Lilka (007_LCD-TAP)

[Українська](LILKA.uk.md) | **English**

LCD-TAP mirrors the [Lilka v2](https://docs.lilka.dev/) screen to an HDMI monitor. It listens to
the display bus between the ESP32-S3 and the ST7789 panel and rebuilds the 240×280 picture as a
DVI-D (HDMI) signal: 640×480 @ 60 Hz or 1280×720 @ 30 Hz. It only receives, so the Lilka keeps
working normally.

Schematics and assembly drawing: [`007_LCD-TAP_DRAWINGS.PDF`](007_LCD-TAP_DRAWINGS.PDF).

## How it works

- **Front end:** a 74VHC4040 counter, a 74AHC1G04 inverter and a 74HC595 shift register turn the
  Lilka's display SPI into an 8-bit parallel bus with a write strobe on every 8th clock. This
  lets the RP2350 keep up with SPI speeds its own SPI input can't handle (limit about 50–62.5 MHz).
- **Main chip:** an RP2350A running the [`pico2_universal`](example/pico2_universal/) firmware.
  It takes the `Parallel8` input and generates the DVI output with HSTX.
- **Lilka preset:** sets up the ST7789 controller with a 240×320 GRAM, cropped to the visible
  window (x 0, y 20, 240×280) and rotated to landscape.

## Connections

| J2 pin | Signal | Lilka |
|---|---|---|
| 1 | GND | GND |
| 2 | 3V3 | 3V3 |
| 3 | LCD_SCK | display SPI SCK |
| 4 | LCD_MOSI | display SPI MOSI |
| 5 | LCD_RST | display RST |
| 6 | LCD_DC | display DC |
| 7 | LCD_CS | display CS |
| 8 | 3V3 | 3V3 (BLK) |

J2 has the same order as the common 8-pin ST7789 module (GND, VCC, SCL, SDA, RES, DC, CS, BLK).

> The REV01 silkscreen labels pin 4 as `GND`. The schematic and netlist say pin 4 is **LCD_MOSI**.
> Wire it by the table above.

| Board connector / control | Purpose |
|---|---|
| HDMI (P1) | Video output |
| USB-C (P2) | 5 V power and USB (CDC serial for configuration and framebuffer readout) |
| UP / DOWN / LEFT / RIGHT / ENTER | OSD menu |
| RUN | Reset |
| FLASH | BOOTSEL, for flashing firmware |
| 720P switch (GPIO20) | Output resolution, read at boot: 1280×720 @ 30 Hz or 640×480 @ 60 Hz |
| PROG (3V3, CLK, DIO, GND) | SWD debug |

## Setup

### 1. Build the firmware

`lilka-dev/lcdtap` has no binary releases yet, and the upstream
[shapoco/lcdtap](https://github.com/shapoco/lcdtap/releases) releases do not include the Lilka
preset, so build from source:

```bash
git clone --recursive https://github.com/lilka-dev/lcdtap.git
cd lcdtap/example/pico2_universal
export PICO_SDK_PATH=/path/to/pico-sdk
./build.sh
```

The result is `build/lcdtap_pico2_universal.uf2`.

### 2. Flash the LCD-TAP

1. Connect the LCD-TAP to your computer over USB-C.
2. Hold **FLASH**, press and release **RUN**, then release **FLASH**. A drive called `RP2350`
   appears.
3. Copy `lcdtap_pico2_universal.uf2` onto that drive. The board reboots into the new firmware.

### 3. Select the Lilka preset

1. Connect a monitor to the HDMI port.
2. Press **ENTER** to open the OSD and choose **Load Preset → Lilka**.
3. Choose **Apply**. The settings are saved to flash and restored at the next boot.

### 4. Set up the Lilka display bus: 40 MHz, SPI mode 3

The Lilka drives its display at **80 MHz** by default. That's too fast for the REV01 front end:
the 74HC4040/74HC595 latch can't line up with byte boundaries at that speed, and the image comes
out garbled (see [`issues/0015`](issues/0015_lilka_lcdtap_latch_race.md)). Lower the clock in
Keira:

1. **Settings → Display → Display frequency → 40 MHz**
2. **Settings → Display → Display SPI mode → SPI_MODE3**
3. Reboot the Lilka. Both values are stored in NVS (`spi` namespace, keys `dispFrequency` and
   `dispMode`) and take effect at the next boot.

SPI mode 3 is the ST7789 default and the mode the LCD-TAP latch timing is designed for. Only
change it if you set a different mode earlier.

### 5. Connect and power up

1. Connect J2 to the Lilka display bus (table above).
2. Power the LCD-TAP from USB-C.
3. Turn on the Lilka. The Keira screen should appear on the monitor in landscape.

## Troubleshooting

| Symptom | What to check |
|---|---|
| Garbled colours or shifted pixels | The Lilka display must be at 40 MHz / SPI_MODE3. Reboot the Lilka after changing either. |
| Picture is rotated, offset or cut off | Load the **Lilka** preset again and press Apply. |
| LCD-TAP doesn't boot after a bad setting | Hold **LEFT** while pressing **RUN** to boot without the saved settings. |
| No picture at all | Open **Command Dump** in the OSD. If no commands show up, check the CS, SCK, MOSI and DC wiring. |
| Monitor shows "no signal" | Check the 720P switch, then reset the board. Some monitors don't accept 1280×720 @ 30 Hz. |
