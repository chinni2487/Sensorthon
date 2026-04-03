# CLAUDE.md — Sensorthon Codebase Guide

## Project Overview

Sensorthon is an educational IoT workshop repository providing standalone firmware examples for the **M5Stack Core 2** (ESP32-based microcontroller). Each subdirectory is an independent PlatformIO project demonstrating a specific hardware capability or sensor. The `Workshop/` folder contains step-by-step tutorials for learners.

---

## Repository Structure

```
Sensorthon/
├── README.md
├── CLAUDE.md
├── Buttons/           # Physical button input (A, B, C)
├── DHTTempSensor/     # DHT11 temperature & humidity sensor
├── IMU/               # 6-axis accelerometer + gyroscope
├── LCD/               # Screen graphics and text rendering
├── LED/               # Addressable RGB LEDs (FastLED/Neopixel)
├── MQTT/              # AWS IoT Core connectivity via MQTT over TLS
├── Microphone/        # I2S microphone audio capture
├── MotionSensor/      # PIR passive infrared motion detection
├── NyanCat/           # RGB LED animation with audio
├── Speaker/           # I2S audio output
├── Touch Screen/      # Capacitive touchscreen input
├── Vibration/         # Haptic vibration motor
├── WAV Player/        # WAV file playback over I2S
├── WaterSensor/       # Analog water detection sensor
└── Workshop/          # Tutorial modules (5 sections)
    ├── 1. Environment Setup/
    ├── 2. Device Registration/
    ├── 3. Hardware Basics/
    ├── 4. Rules/
    └── 5. Lambda/
```

### Standard Project Layout (per module)

```
ProjectName/
├── src/main.cpp           # Firmware entry point (Arduino sketch)
├── include/
│   ├── README
│   └── secrets_template.h # (MQTT only) AWS credentials template
├── lib/README
├── test/README
└── platformio.ini         # Build configuration
```

---

## Build System

**Tool:** [PlatformIO](https://platformio.org/)

**Target platform:** `espressif32` (ESP32)  
**Board:** `m5stack-core2`  
**Framework:** Arduino  
**Serial baud rate:** `115200`

Each project has its own `platformio.ini`. There is no root-level build file — projects are built independently.

### Common PlatformIO Commands

```bash
# Build firmware
pio run -d <ProjectDir>

# Build and upload to device (connect via USB first)
pio run -d <ProjectDir> --target upload

# Open serial monitor (115200 baud)
pio device monitor -d <ProjectDir>

# Upload and monitor in one step
pio run -d <ProjectDir> --target upload && pio device monitor -d <ProjectDir>

# Clean build artifacts
pio run -d <ProjectDir> --target clean
```

### VSCode Workflow

1. Open a project folder in VSCode with the PlatformIO extension installed.
2. Use "PROJECT TASKS" panel: Build → Upload → Monitor.
3. Device is auto-detected on the appropriate serial port.
4. Windows users may need the [Silicon Labs CP210x USB driver](https://www.silabs.com/developers/usb-to-uart-bridge-vcp-drivers).

### platformio.ini Version Notes

Some projects use `espressif32@3.4.0`, others use `espressif32@4.4.0`. The MQTT project explicitly requires `4.4.0`. When creating new projects, prefer `4.4.0` for consistency.

---

## Code Conventions

### Language

C++ in Arduino dialect. All firmware lives in `src/main.cpp`.

### Naming

| Element | Convention | Example |
|---------|-----------|---------|
| Variables | camelCase | `accX`, `lastMsg`, `temp` |
| Constants / macros | UPPER_SNAKE_CASE | `NUM_LEDS`, `SAMPLE_RATE`, `PORT` |
| Functions | camelCase | `connectWifi()`, `messageHandler()` |
| Classes/types | PascalCase (library types) | `WiFiClientSecure`, `MQTTClient` |

### Arduino Sketch Pattern

Every project follows the standard Arduino pattern:

```cpp
void setup() {
    Serial.begin(115200);
    M5.begin();           // Initialize M5Stack Core 2 hardware
    // Module-specific initialization
}

void loop() {
    M5.update();          // Refresh button/touch state each frame
    // Read sensors, update display, publish data
    delay(interval);
}
```

### Serial Debugging

All projects log to serial at 115200 baud using `Serial.println()` / `Serial.printf()`. This is the primary debugging interface — there is no logging framework.

### LCD Display

Use the `M5.Lcd` API:

```cpp
M5.Lcd.clear();
M5.Lcd.setCursor(x, y);
M5.Lcd.setTextColor(WHITE, BLACK);
M5.Lcd.printf("Value: %d", value);
// Color constants: RED, GREEN, BLUE, WHITE, BLACK, YELLOW, etc.
```

---

## Credentials and Security

**`secrets.h` is gitignored and must never be committed.**

The MQTT project requires a `secrets.h` file (not tracked in git). Use `MQTT/include/secrets_template.h` as the starting point:

```cpp
static const String TEAMNAME = "YourTeam";
const char WIFI_SSID[]     = "your_ssid";
const char WIFI_PASSWORD[] = "your_password";
// AWS IoT endpoint (Sydney region)
const char AWS_IOT_ENDPOINT[] = "a2fk4jnz30nap8-ats.iot.ap-southeast-2.amazonaws.com";
// Paste PEM-encoded certs into the PROGMEM string literals below
static const char AWS_CERT_CA[]      PROGMEM = R"EOF(-----BEGIN CERTIFICATE-----...EOF";
static const char AWS_CERT_CRT[]     PROGMEM = R"KEY(-----BEGIN CERTIFICATE-----...KEY";
static const char AWS_CERT_PRIVATE[] PROGMEM = R"KEY(-----BEGIN RSA PRIVATE KEY-----...KEY";
```

**Gitignored patterns to preserve:**
- `**/.pio` — PlatformIO build artifacts
- `**/*.cer`, `**/*.crt`, `**/*.key` — certificates
- `**/secrets.h` — credentials

Never remove these entries from `.gitignore`.

---

## AWS IoT / MQTT Integration

The MQTT project connects to AWS IoT Core using:

- **Protocol:** MQTT over TLS (port 8883)
- **Auth:** X.509 client certificates
- **Publish topic:** `EduKit_<TEAMNAME>/pub` (every 30 seconds)
- **Subscribe topic:** `EduKit_<TEAMNAME>/sub` (incoming LED control)
- **Payload format:** JSON via ArduinoJson `StaticJsonDocument<200>`

Incoming messages are expected to contain `{ "LED": true/false }` to toggle the M5Stack status LED.

### Reconnection Handling

The `loop()` function checks `mqttClient.connected()` and calls `connectAWSIoTCore()` again if the connection drops.

---

## Key Libraries

| Library | Version | Used In |
|---------|---------|---------|
| `m5stack/M5Core2` | `^0.1.2` | All projects |
| `bblanchon/ArduinoJson` | `^6.19.4` | MQTT |
| `256dpi/MQTT` | `^2.5.0` | MQTT |
| `adafruit/DHT sensor library` | `^1.4.3` | DHTTempSensor |
| `fastled/FastLED` | `^3.5.0` | LED, MotionSensor, NyanCat |
| `earlephilhower/ESP8266Audio` | `^1.9.6` | NyanCat, WAV Player |

---

## Testing

There are no automated tests in this repository. The `test/` folder in each project contains only a PlatformIO placeholder README. All validation is done manually:

- Observe serial monitor output at 115200 baud
- Verify visual output on LCD and LEDs
- Use the AWS IoT MQTT Test Client to inspect publish/subscribe messages

When modifying firmware, test by building and uploading to a physical M5Stack Core 2 device.

---

## Workshop Modules

The `Workshop/` directory contains markdown tutorials:

| Module | Content |
|--------|---------|
| `1. Environment Setup` | VSCode, PlatformIO, USB drivers, first build |
| `2. Device Registration` | AWS IoT Core setup, certificate generation, secrets.h config |
| `3. Hardware Basics` | LCD, buttons, touch screen, sensor reading patterns |
| `4. Rules` | AWS IoT Rules Engine configuration |
| `5. Lambda` | AWS Lambda function integration |

---

## Adding a New Example Project

1. Create a new directory at the repo root: `mkdir MyNewSensor`.
2. Inside it, create the standard PlatformIO layout (`src/`, `include/`, `lib/`, `test/`, `platformio.ini`).
3. Use this `platformio.ini` template:

```ini
[env:m5core2-MyNewSensor]
platform = espressif32@4.4.0
board = m5stack-core2
framework = arduino
lib_deps =
    m5stack/M5Core2@^0.1.2
    ; add other libs here
monitor_speed = 115200
```

4. Implement `src/main.cpp` following the `setup()`/`loop()` pattern above.
5. If credentials are needed, add an `include/secrets_template.h` and ensure `secrets.h` stays gitignored.

---

## Common Pitfalls

- **Missing `secrets.h`:** The MQTT project will not compile without this file. Copy and fill in `secrets_template.h`.
- **Wrong PlatformIO version:** If a project fails to build, check that `espressif32` version in `platformio.ini` matches the one tested (prefer `4.4.0`).
- **Device not detected:** On Windows, install the Silicon Labs CP210x driver. On Linux, add your user to the `dialout` group (`sudo usermod -aG dialout $USER`).
- **Stale `.pio` cache:** Run `pio run --target clean` to clear build artifacts if you see unexpected compilation errors.
- **Folder names with spaces:** `Touch Screen/` and `WAV Player/` contain spaces — quote these paths in shell commands.
