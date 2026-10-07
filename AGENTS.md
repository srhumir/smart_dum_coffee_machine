## Project
ESP32-S3 PID temperature controller for a Gaggia Classic and similar espresso machines.

Read `docs/software_spec_v1.md` before making architectural changes.

## Platform
- ESP-IDF
- ESP32-S3
- C preferred for firmware modules
- FreeRTOS

## Safety rules
- Heater GPIO must default OFF.
- No network/UI module may directly manipulate heater GPIO.
- Safety logic always overrides PID/heater requests.
- Invalid sensor data must result in heater OFF.
- Over-temperature protection must not depend on the web server or PID task.
- Do not weaken safety behavior without explicit instruction.

## Architecture
Keep these concerns separate:
- sensor acquisition
- filtering
- safety
- PID/control
- SSR timing
- persistence
- Wi-Fi
- HTTP API
- web UI

Avoid blocking calls in real-time tasks.

## Timing
- sensor: 4 Hz
- PID: 2 Hz
- safety: 10 Hz
- history: 1 Hz
- SSR window: 1 second

## Build
Before completing a task:
1. Run the ESP-IDF build.
2. Fix compiler warnings/errors introduced by the change.
3. Run relevant tests.
4. Summarize changed files and remaining limitations.

## Scope
Implement only the requested task.
Do not add:
- steam control
- pressure profiling
- shot detection
- cloud connectivity
unless explicitly requested.
