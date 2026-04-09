# SD Logger Node — Software Setup Guide

**Complete software setup instructions for I2S audio capture with SD card storage on ESP32.**

---

## Table of Contents
- [What This Does](#what-this-does)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Build & Upload](#build--upload)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)
- [Key Functions Reference](#key-functions-reference)

---

## What This Does

The SD Logger Node receives stereo audio via I2S protocol and simultaneously:
- **Records** 32-bit, 16 kHz stereo WAV files to microSD card
- **Streams** CSV samples to USB serial for real-time monitoring
- **Rotates** recording sessions into unique directories

**Data Format**: 32-bit signed integers, 16000 Hz, 2 channels (stereo)

---

## Prerequisites

### Software
- **PlatformIO** (IDE or CLI)  
- **Python** 3.6 or newer
- **Git** (optional, for cloning)

### Hardware (handled separately)
- ESP32-WROOM-32 board
- microSD card (FAT32 formatted)
- I2S audio source (sender node)

### Install PlatformIO

**Option A: PlatformIO IDE** (recommended for beginners)
```bash
# Download and install from:
# https://platformio.org/install/ide
```

**Option B: PlatformIO CLI** (if you prefer command line)
```bash
pip install platformio
pio --version  # verify installation
```

---

## Installation

### 1. Obtain Source Code

**From GitHub:**
```bash
git clone <your-repo-url>
cd sd_logger_node
```

**Or manually:**
- Extract zip file to your projects directory
- Open `sd_logger_node` folder

### 2. Identify Serial Port

Connect ESP32 via USB cable. Identify which port it appears on:

**Linux/macOS:**
```bash
ls -la /dev/tty*
# Look for /dev/ttyUSB0 or /dev/ttyACM0
```

**Windows:**
- Open Device Manager → Ports (COM & LPT)
- Note the COM port (e.g., COM3)

**PlatformIO:**
```bash
pio device list
```

---

## Configuration

### 1. Serial Port Setup

Edit `platformio.ini` and set your port:

```ini
[env:esp32dev]
platform = espressif32
board = esp32doit-devkit-v1
framework = arduino
monitor_speed = 921600
upload_port = /dev/ttyUSB0      ; ← Change to YOUR port
monitor_port = /dev/ttyUSB0     ; ← Same port
```

**Windows example:**
```ini
upload_port = COM3
monitor_port = COM3
```

### 2. Verify Library Dependencies

The following libraries auto-install via `platformio.ini`:

| Library | Purpose |
|---------|---------|
| `arduino-audio-tools` | I2S/WAV streaming |
| `SD` | SD card interface |

No manual installation needed — PlatformIO handles this.

### 3. SPI Pin Configuration (Optional)

If using non-standard pins, edit `include/sd_card.h`:

```cpp
// Default pins (VSPI) — change only if wiring differs
constexpr int SD_CS_PIN   = 5;    // Chip Select
constexpr int SD_MOSI_PIN = 23;   // Data out to card
constexpr int SD_MISO_PIN = 19;   // Data in from card
constexpr int SD_SCK_PIN  = 18;   // Clock
```

**Safe alternatives for CS pin:** 4, 13, 14, 15, 16, 17, 21, 22, 25, 26, 27, 32, 33  
(Avoid pins already used for I2S: 14, 15, 32, 34, 35)

---

## Build & Upload

### Quick Start (One Command)

```bash
cd sd_logger_node
platformio run -t upload
```

### Step-by-Step

**1. Clean build** (recommended first time)
```bash
pio run -t clean
```

**2. Build**
```bash
pio run
```

Expected output:
```
Building in release mode
...
Building .pio/build/esp32dev/firmware.elf
...
Memory Usage: [====   ] 42.5%
```

**3. Upload**
```bash
pio run -t upload
```

Device will reboot automatically when upload completes.

**4. Monitor** (watch serial output)
```bash
pio device monitor -b 921600
```

Press `Ctrl+C` to exit monitor.

### Complete Workflow (All-in-One)

```bash
platformio run -t clean && platformio run -t upload && platformio device monitor -b 921600
```

---

## Verification

### Expected Serial Output (in order)

After upload completes, the board resets. You should see:

```
Serial communication initialized
[I2S] I2S receiver started
Debug Info:
StreamInfo - Sample Rate: 16000, Channels: 2, Bits per Sample: 32
BCLK: 34
LRCLK: 35
DIN: 32

[SD] Card mounted — 32000 MB total
[SD] Free space: 31900 MB

Setup complete, waiting for I2S sender...
Waiting for I2S sender...
```

When I2S sender connects:
```
I2S sender detected
[SESSION] Created /AUDIO_LOGGER_SESSION_1
[MAIN] Entering CSV streaming mode.

Starting SD write cycle
[WAV] Recording started: /AUDIO_LOGGER_SESSION_1/recording_1_04-05-26.wav
...
```

### Test Checklist

- ✅ No garbage characters in serial output (if present, adjust monitor speed)
- ✅ "I2S sender detected" appears within 10 seconds
- ✅ SD card size is non-zero (indicates successful mount)
- ✅ LED blinks during recording (indicates active I2S transfer)

---

## Troubleshooting

### No Serial Output at All

**Issue:** Serial monitor shows nothing  
**Solutions:**
1. Verify baud rate is **921600** in monitor settings
2. Check USB cable is fully plugged in
3. Try different USB port
4. Restart serial monitor application
5. Check Device Manager / `ls /dev/tty*` for port existence

---

### Garbage Characters in Output

```
ΩΩ⎕☺³♪⎕Ω☺Ω
```

**Issue:** Baud rate mismatch  
**Solution:** Verify monitor speed is **921600**:
```bash
pio device monitor -b 921600
```

---

### "I2S Sender Not Detected" (Stays on "Waiting...")

**Issue:** No audio source or I2S wiring problem  
**Verify:**
1. Sender node is powered on and running first
2. I2S wires connected (BCLK, WS, DIN, GND)
3. Sender and receiver share common GND
4. No broken/loose connections

---

### SD Card Errors

```
[SD] ERROR: SD card mount failed!
```

**Solutions:**
1. Format SD card to **FAT32**
2. Verify card is fully inserted
3. Check SPI pins match wiring (see [Configuration](#configuration))
4. Try a different SD card (compatibility issue)

---

### Build Fails with Missing Libraries

```
undefined reference to 'AudioTools'
```

**Solution:**
```bash
rm -rf .pio/libdeps
pio run
```

PlatformIO will re-download all dependencies.

---

### Upload Hangs / Device Not Recognized

**Issue:** Serial port is in use  
**Solution:**
1. Close other programs using the port (including other serial monitors)
2. Unplug USB, wait 5 seconds, plug back in
3. Try different USB port

---

## Key Functions Reference

### Core Flow

```
setup()
  ├─ Initialize Serial (921600 baud)
  ├─ Configure I2S receiver
  ├─ Initialize SD card
  └─ Wait for I2S sender
  
loop() — repeats endlessly
  ├─ write_sd(20000)    — record 20 seconds to WAV
  └─ write_csv(20000)   — stream 20 seconds to serial
```

### Main Functions

#### `write_sd(unsigned long durationMs)`

**What it does:** Records I2S audio to microSD card as WAV file  
**Inputs:** Duration in milliseconds (e.g., `20000` = 20 seconds)  
**Output:** Creates `/AUDIO_LOGGER_SESSION_N/recording_X.wav`

**Example:**
```cpp
write_sd(30000);  // Record 30 seconds
```

**Key operations:**
- Generates unique filename via `getWavPath()`
- Opens file on SD card
- Streams I2S samples to WAV encoder
- Flushes and closes file
- Updates WAV header with correct file size

---

#### `write_csv(unsigned long durationMs)`

**What it does:** Streams I2S audio to USB serial as CSV  
**Inputs:** Duration in milliseconds  
**Output:** Serial CSV lines (left_sample, right_sample)

**Example:**
```cpp
write_csv(20000);  // Stream 20 seconds
```

**CSV Output Format:**
```
1234567, -1234567
-123456, 123456
...
```

Each line = one stereo sample (32-bit left, 32-bit right)

---

#### `sd_card_init()`

**What it does:** Initializes SPI bus and mounts SD card  
**Returns:** `true` if successful, `false` if error  
**Errors:** Missing card, wrong format (must be FAT32), wiring issue

**Called in:** `setup()` (with retry logic)

---

#### `wav_writer_begin(AudioInfo, const char* path)`

**What it does:** Opens file and starts WAV encoding pipeline  
**Inputs:**
- `AudioInfo` — sample rate, channels, bit depth
- `path` — file path (e.g., `/AUDIO_LOGGER_SESSION_1/recording_1.wav`)

**Returns:** `true` on success, `false` if file can't be created  
**Side Effect:** Writes WAV header automatically

---

#### `wav_writer_end()`

**What it does:** Finalizes WAV file and closes file handle  
**Important:** Must be called after recording to update file size in header  
**If skipped:** File will not be playable (truncated header)

---

#### `getWavPath()`

**What it does:** Generates unique filename by checking existing files  
**Logic:**
1. Lists all files in session directory
2. Finds first gap in `recording_1.wav`, `recording_2.wav`, etc.
3. Returns next available path (up to 100 files)

**Example outputs:**
```
/AUDIO_LOGGER_SESSION_1/recording_1_04-05-26.wav
/AUDIO_LOGGER_SESSION_1/recording_2_04-05-26.wav
/AUDIO_LOGGER_SESSION_2/recording_1_04-05-26.wav
```

---

#### `createSessionDir()`

**What it does:** Creates unique session folder at boot  
**Logic:**
1. Checks for existing `/AUDIO_LOGGER_SESSION_N` directories
2. Creates next available session (1, 2, 3, ... up to 999)
3. All recordings in that session go into this folder

**Called in:** `setup()` (after SD init)

---

### Configuration Constants

Edit values in `src/main.cpp`:

```cpp
// Audio format (must match sender node)
AudioInfo StreamInfo(16000, 2, 32);
// │         │       │      │  └─ Bits per sample (32-bit)
// │         │       │      └──── Channels (stereo)
// │         │       └─────────── Sample rate (16 kHz)
// └─────────┴─────────────────── Object name
```

**To change format** (e.g., 44.1 kHz instead of 16 kHz):
```cpp
AudioInfo StreamInfo(44100, 2, 32);
```

⚠️ **Must match sender node configuration exactly**

---

### Loop Timing

Adjust recording/streaming duration in `loop()`:

```cpp
void loop() {
    write_sd(20000);   // ← Change to 30000 for 30 sec, etc.
    delay(1000);
    write_csv(20000);  // ← Same here
    delay(1000);
}
```

Current cycle: 20 sec SD + 1 sec pause + 20 sec CSV + 1 sec pause = ~42 seconds per cycle

---

## Development Notes

### Code Structure

```
src/
├── main.cpp         — Setup, loop, I2S config, session management
├── sd_card.cpp      — SPI initialization
└── wav_writer.cpp   — WAV file encoding pipeline

include/
├── sd_card.h        — SPI pin definitions
└── wav_writer.h     — WAV encoder interface
```

### Dependent Libraries

- **AudioTools** by Phil Schatzmann
  - Handles I2S streaming, WAV encoding, format conversion
  - Auto-installed by platformio.ini

- **SD** (ESP32 built-in)
  - SD card file operations
  - Auto-installed by platformio.ini

---

## Advanced: Changing Recording Format

### 16-Bit WAV Instead of 32-Bit

If you want smaller files, add a format converter in `main.cpp`:

**Before:**
```cpp
StreamCopy wavCopier(wav_writer_stream(), i2sStream);
```

**After (with 32→16 bit conversion):**
```cpp
NumberFormatConverterStream converter(i2sStream);
auto convCfg = converter.defaultConfig();
convCfg.from_bits_per_sample = 32;
convCfg.to_bits_per_sample = 16;
convCfg.copyFrom(StreamInfo);
converter.begin(convCfg);

StreamCopy wavCopier(wav_writer_stream(), converter);
```

Also update WAV header info:
```cpp
AudioInfo wavInfo(16000, 2, 16);  // ← 16-bit instead of 32-bit
wav_writer_begin(wavInfo, wavPath.c_str());
```

---

## Support

**Issues during setup?**
1. Check all serial output matches [expected output](#expected-serial-output)
2. Review [troubleshooting](#troubleshooting) for your error
3. Verify board port and baud rate in `platformio.ini`

**Questions about code?**
- See [Key Functions Reference](#key-functions-reference)
- Check inline comments in `src/` files
- Review Arduino-Audio-Tools documentation: https://github.com/pschatzmann/arduino-audio-tools

---

**Last Updated:** April 2026  
**Compatible Boards:** ESP32-WROOM-32 (other ESP32 variants may work with pin adjustments)
