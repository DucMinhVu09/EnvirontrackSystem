# Firmware

MicroPython firmware for the LILYGO T-A7670G (ESP32 + SIMCom A7670G). It reads the SDS011 and SHT30 sensors, uploads the data to ThingSpeak over LTE, and scrolls the readings on the 64x16 LED matrix.

## Requirements

- LILYGO TTGO T-A7670G board with a nano-SIM card (mobile data enabled)
- MicroPython firmware for ESP32
- [Thonny](https://thonny.org) or another tool for uploading files to the board
- The following modules on the board, in addition to the files in this folder:

| Module | Purpose | Source |
|---|---|---|
| `sim7600.py` | AT-command driver for the LTE modem: HTTP requests, network time, SMS and calls, FTP OTA | FAS-2.0 firmware (FASFARTECH), not included |
| `config.py` | settings storage (`Config` object): channel ID and key, upload interval, calibration, thresholds, phone numbers | FAS-2.0 firmware (FASFARTECH), not included |
| `sht30.py` | SHT30 driver, `SHT30().measure()` returns `(temperature, humidity)` | MicroPython SHT30 driver |
| `lcd_api.py`, `i2c_lcd.py` | HD44780 LCD over a PCF8574 I2C backpack (optional LCD) | [python_lcd](https://github.com/dhylands/python_lcd) |
| `sdcard.py` | SD card driver, only needed if SD logging is enabled | micropython-lib |

`neopixel`, `onewire` and `ds18x20` are built into MicroPython for ESP32.

## File structure

| File | Description | Source |
|---|---|---|
| [`main.py`](main.py) | entry point; imports the main program and falls back to an FTP OTA update loop if the import fails | FAS-2.0 firmware (FASFARTECH) |
| [`loop_sds011.py`](loop_sds011.py) | main program: initialization, sensor reading, ThingSpeak upload, remote commands, alerts, LCD and LED output | based on the FAS-2.0 data-logger firmware, configured for the SDS011 and extended with the LED matrix output |
| [`ledpm25.py`](ledpm25.py) | LED matrix driver: pixel mapping, character drawing, scrolling text, AQI face icons | EnviroTrack |
| [`font.py`](font.py) | 7-row bitmap font (A-Z, 0-9, space, `!`, `:`, `%`, `.`) | EnviroTrack |
| [`sds011.py`](sds011.py) | SDS011 UART driver | [micropython-sds011](https://github.com/alexmrqt/micropython-sds011) by Alexandre Marquet |
| [`boot.py`](boot.py) | default MicroPython boot file | MicroPython |

The ThingSpeak write key in a commented-out line of `loop_sds011.py` (line 272) has been masked.

## Installation

1. Flash MicroPython for ESP32 onto the board using Thonny or esptool.
2. In Thonny, select the "MicroPython (ESP32)" interpreter and the board's serial port.
3. Upload all `.py` files from this folder and the modules listed under [Requirements](#requirements) to the root of the board's file system.
4. Insert the nano-SIM card and reset the board.

On startup the shell shows the modem starting, `Connected GSM!`, the network time and then the first sensor readings.

## Configuration

Settings are stored by `config.py` and can be changed from the Thonny shell or remotely through ThingSpeak.

| Setting | Variable | Description |
|---|---|---|
| ThingSpeak channel ID | `Config.thingid` | channel that receives the data |
| ThingSpeak write key | `Config.thingkey` | 16-character Write API Key |
| Upload interval | `Config.tupload` | minutes between uploads |
| Calibration offsets | `Config.calibset` | added to PM2.5, PM10, temperature and humidity |
| Alarm thresholds | `Config.lowset`, `Config.highset` | lower and upper limits per field |
| Alert phone numbers | `Config.tel` | numbers that receive alarm calls and SMS |
| Sleep mode | `Config.sleepmode` | deep sleep between uploads |

To set the channel from the shell (as in the commented lines 271-273 of `loop_sds011.py`):

```python
Config.thingid = '<channel id>'
Config.thingkey = '<write api key>'
Config.Save_setting()
```

### Remote commands

Before each upload the device reads the last three status messages of its ThingSpeak channel (`/channels/<id>/status.json?results=3`). A status update containing one of the following commands changes the configuration remotely:

| Command | Effect |
|---|---|
| `Settimeupload=N` | set the upload interval to N minutes |
| `Setb=b1,b2,b3,b4` | set calibration offsets for the four fields |
| `Seta=a1,a2,a3,a4` | set calibration gains (used by the rain-gauge mode) |
| `Setlow=...` / `Sethigh=...` | set alarm thresholds per field |
| `Setphone=p1,p2,...` | set alert phone numbers |
| `Setinouts=N` | bit 0 enables alarms, bit 1 enables SMS |
| `+Alarm on` / `+Alarm off` | enable or disable alarms (`+Bat alarm` / `+Tat alarm` in Vietnamese) |
| `Key=<channel id>,<write key>` | switch to another ThingSpeak channel |
| `Setupdate=xxxxxx` | start an FTP OTA firmware update |

Example: `https://api.thingspeak.com/update?api_key=<write key>&status=Settimeupload%3D5`

The device confirms the new values (`Gettimeupload=...`, `Getb=...` and so on) in the status field of its next upload.

## Program flow

### Startup

1. Set the CPU clock to 80 MHz and open non-volatile storage.
2. Open UART2 (9600 baud, TX=GPIO14, RX=GPIO13), switch the SDS011 to query mode and put it to sleep.
3. Initialize the SHT30, the ADCs on GPIO36/35 and a 200-second watchdog timer.
4. Initialize the LCD if connected and load the settings from `config.py`.
5. Start the LTE modem and send `AT` until it responds (the modem restarts after 10 failed attempts and the ESP32 resets after more than 20).
6. Get the network time, check the SIM card, network registration and signal quality, and read the device and field names from the ThingSpeak channel.
7. Prepare a status message with the reset reason, current settings and firmware version for the first upload.

### Main loop

| Step | Action | Code |
|---|---|---|
| 1 | read supply and battery voltages | [L612-L614](loop_sds011.py#L612-L614) |
| 2 | wake the SDS011, wait 5 s, read PM2.5/PM10, put it to sleep | [L645-L663](loop_sds011.py#L645-L663) |
| 3 | read temperature and humidity from the SHT30 | [L664-L672](loop_sds011.py#L664-L672) |
| 4 | feed the watchdog | [L714](loop_sds011.py#L714) |
| 5 | once per minute: check thresholds; every `tupload` minutes read remote commands and upload to ThingSpeak | [L716-L775](loop_sds011.py#L716-L775) |
| 6 | update the LCD | [L777-L808](loop_sds011.py#L777-L808) |
| 7 | send alarm calls and SMS if a threshold was crossed | [L812-L828](loop_sds011.py#L812-L828) |
| 8 | scroll the readings on the LED matrix | [L837](loop_sds011.py#L837) |
| 9 | enter deep sleep if sleep mode is enabled | [L854-L858](loop_sds011.py#L854-L858) |

Any exception inside the loop is logged, the modem is powered down and the ESP32 resets after 10 seconds.

## SDS011 communication

The driver uses the sensor's query mode. Each reading is requested with a query command, and the sensor replies with a 10-byte frame:

```
AA C0 D1 D2 D3 D4 D5 D6 CS AB
```

| Bytes | Content |
|---|---|
| `AA C0` | header and command |
| D1, D2 | PM2.5, low and high byte |
| D3, D4 | PM10, low and high byte |
| D5, D6 | sensor ID |
| CS | checksum: (D1 + ... + D6) mod 256 |
| `AB` | tail |

PM2.5 = (256 x D2 + D1) / 10 and PM10 = (256 x D4 + D3) / 10, in µg/m³. Packets with a wrong checksum or tail are rejected. Derivations are in [docs/formulas.md](../docs/formulas.md#1-sds011-data-frame).

## LED matrix driver

`ledpm25.py` drives the 64x16 matrix (1024 WS2812B LEDs on GPIO32) through the MicroPython `neopixel` module.

- **`set_pixel(x, y, color)`** converts a column/row position to the LED index of the zigzag chain: 16x + y for even columns, 16x + (15 - y) for odd columns.
- **`draw_char(char, x, y, color)`** draws one character from `font.py`. Each character is 7 bytes, one per row; bit 6 is the leftmost pixel and bit 0 the rightmost. The first 6 rows are drawn.
- **`scroll_text(text)`** starts the text at the right edge (x = 64) and moves it one pixel to the left per frame, with an 8-pixel character spacing and a 20 ms delay between frames, until the whole text has left the display. The text is drawn on rows 5-10 in yellow.
- **`draw_face(aqi)`** draws a 16x16 face icon for AQI levels 1-5 (green, yellow, orange, red, purple) at columns 44-59. It is prepared for a future version of the display and not called in the main loop yet.

The main loop displays a message in this format:

```
PM25:30.4 PM10:31.2 TEMP:29.1 HUMI:58.2
```

## ThingSpeak integration

Data is sent as an HTTP GET request through the LTE modem:

```
http://api.thingspeak.com/update?api_key=<write key>&field1=<PM2.5>&field2=<PM10>&field3=<temperature>&field4=<humidity>
```

A `&status=` parameter is appended when there is a status message to report. ThingSpeak responds with the new entry ID, or `0` if the update was rejected; the firmware shows it on the LCD.

## Timing

| Stage | Duration |
|---|---|
| SDS011 fan warm-up | 5 s |
| LED scroll of a 39-character message (376 frames) | about 19 s or more |
| Delay at the end of the loop | 0.2 s |

One measurement cycle therefore takes roughly 25 seconds, and data is uploaded every `tupload` minutes. The scroll time is calculated in [docs/formulas.md](../docs/formulas.md#6-display-timing).
