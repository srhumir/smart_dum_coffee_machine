DIY Gaggia ESP32-S3 PID Controller — Software Specification V1
1. Purpose
This document specifies Version 1 of the software for an ESP32-S3-based PID temperature controller for a Gaggia Classic espresso machine.
The firmware shall be implemented in C/C++ using ESP-IDF.
The system has two main responsibilities:
Real-time temperature control of the espresso boiler.
A mobile-friendly web user interface served directly by the ESP32 over its own Wi-Fi hotspot.
V1 controls brew temperature only. Steam control, brew-shot detection, pressure profiling, and a native mobile application are outside the scope of this version.
---
2. Hardware Assumptions
2.1 Microcontroller
ESP32-S3-R8
Dual-core
Wi-Fi enabled
PSRAM available
2.2 Temperature sensing
K-type thermocouple
MAX31855 thermocouple interface
SPI communication
2.3 Heater actuation
Zero-cross solid state relay
SSR controls AC power to the Gaggia boiler heating element
2.4 Safety hardware
The machine's physical thermal fuse remains present and independent of the ESP32.
---
3. High-Level Architecture
The firmware shall separate real-time control from networking and user-interface work.
Core / task grouping
Real-time control side
Responsible for:
Thermocouple sampling
EMA filtering
Safety monitoring
PID calculation
Warm-up logic
SSR command generation
Watchdog participation
Networking / UI side
Responsible for:
Wi-Fi access point
HTTP server
REST API
Web UI
History retrieval
Configuration
Diagnostics
OTA firmware update
Networking activity must never block or delay temperature control or safety functions.
---
4. Control Timing
4.1 Temperature sampling
Frequency: 4 Hz
Period: 250 ms
Each valid MAX31855 sample shall update the raw temperature value.
4.2 Temperature filtering
An exponential moving average shall be applied.
The EMA coefficient shall be fixed in firmware in V1.
The EMA coefficient shall not be user-adjustable from the UI.
The filtered temperature shall be the primary temperature used by the control algorithm and UI status logic.
4.3 PID update
Frequency: 2 Hz
Period: 500 ms
The PID loop shall operate on the filtered temperature.
4.4 Safety task
Frequency: 10 Hz
Period: 100 ms
The safety task shall be independent of the PID task.
It shall use the latest available sensor and controller state and shall have authority to force the SSR OFF regardless of PID output.
---
5. Temperature Control Strategy
5.1 Default brew target
Default target temperature: 93 °C
5.2 User-settable range
The target temperature shall be restricted to:
Minimum: 80 °C
Maximum: 110 °C
Any request outside this range shall be rejected.
5.3 Warm-up phase
On cold startup, after a valid temperature reading is available and no safety fault exists:
Heater output shall be 100%.
Full-power warm-up shall continue until the filtered temperature reaches:
`target_temperature - 10 °C`
Example:
For a 93 °C setpoint, full-power warm-up ends at 83 °C.
5.4 Transition to PID
The transition from full-power warm-up to PID control shall use a bumpless-transfer strategy.
The PID controller shall not introduce a sudden heater-output discontinuity when control changes from the warm-up phase to closed-loop PID control.
5.5 PID formulation
The controller shall support:
Kp
Ki
Kd
The derivative term shall use derivative on measurement rather than derivative on error.
This is intended to prevent derivative kick when the target temperature changes.
5.6 Integral anti-windup
The PID implementation shall include integral anti-windup.
Integral accumulation shall be constrained so that output saturation at 0% or 100% does not produce excessive stored integral action.
5.7 Output limits
PID heater output shall be limited to:
Minimum: 0%
Maximum: 100%
No negative output or active cooling exists.
---
6. SSR Control
The zero-cross SSR shall be controlled using time-proportional window control.
Window duration
1 second
Example:
If PID output is 30%, the heater shall be ON for approximately 300 ms during a 1-second control window and OFF for approximately 700 ms.
The SSR output implementation shall avoid unnecessarily high-frequency switching.
The heater output percentage reported to the UI shall represent the requested duty level.
---
7. Operating States
V1 shall expose three user-visible operating states:
Heating
Stable
Fault
7.1 Heating
The system is actively warming or regulating but has not yet satisfied the Stable criteria.
7.2 Stable
The system enters Stable when:
filtered temperature remains within ±1.0 °C of the target
continuously for at least 10 seconds
7.3 Stable-state hysteresis
Once Stable, the system shall not return to Heating until the filtered temperature deviates by more than ±1.5 °C from target.
This hysteresis prevents status flicker.
7.4 Fault
Fault state is entered when a safety condition requires heater shutdown.
---
8. Safety Requirements
Safety behavior takes priority over all control requests.
8.1 Heater default state
The SSR output shall fail OFF.
During:
Boot
Reset
Watchdog reset
OTA
Invalid sensor state
the heater must remain OFF unless explicitly permitted by valid control logic.
8.2 Startup sensor validation
After boot, the heater shall remain OFF until at least one valid thermocouple reading has been received.
8.3 Sensor fault behavior
If the MAX31855 reports:
Thermocouple disconnected
Invalid thermocouple reading
Sensor fault
Communication failure producing invalid temperature data
then:
Heater shall immediately be forced OFF.
System state shall become Fault.
Fault shall be reported to the UI.
Fault event shall be logged.
8.4 Sensor fault recovery
Sensor-related faults shall automatically recover when:
Valid thermocouple readings return.
All safety conditions are satisfied.
A reboot shall not be required.
8.5 Over-temperature cutoff
Software over-temperature limit:
120 °C
If measured temperature reaches or exceeds 120 °C:
Heater shall immediately be forced OFF.
System shall enter Fault.
Over-temperature fault shall be latched.
Reboot shall be required before normal heating resumes.
Fault shall be persistently logged.
8.6 Watchdog
A watchdog shall monitor the control system.
If the control task stalls:
ESP32 shall reset.
SSR must return to OFF during reset and startup.
---
9. Manual Heater Disable
The main or settings UI shall provide a manual controller disable / heater OFF control.
When disabled:
PID heating shall stop.
SSR shall remain OFF.
ESP32 shall continue running.
Wi-Fi hotspot shall remain active.
Web UI shall remain available.
This disable state shall not persist across reboot.
After reboot, normal control may resume once:
A valid temperature reading is available.
No safety fault exists.
---
10. Wi-Fi Architecture
V1 shall not connect to the user's home network.
The ESP32 shall operate permanently as a Wi-Fi access point.
10.1 Hotspot
ESP32 hosts its own Wi-Fi SSID.
Hotspot must be password-protected.
10.2 Configuration
The advanced/settings UI shall allow modification of:
Hotspot SSID
Hotspot password
These values shall:
Be stored in NVS.
Take effect after reboot.
10.3 Web access
The web UI shall be accessible by:
Fixed AP IP address, such as `192.168.4.1`
Friendly local hostname such as `gaggia.local`, if practical
The fixed IP shall remain the reliable fallback.
---
11. Web User Interface
The UI shall be mobile-friendly and optimized for use from a phone browser.
V1 shall contain two major sections.
11.1 Main dashboard
The main dashboard shall display:
Current filtered temperature
Target temperature
Current heater output percentage
Current system state:
Heating
Stable
Fault
Live temperature/time visualization
Manual heater/controller disable control
11.2 Advanced / settings page
The advanced page shall provide:
Raw thermocouple temperature
Filtered temperature
PID parameters:
Kp
Ki
Kd
Hotspot SSID
Hotspot password
OTA firmware update
Uptime
Last reset reason
Current fault code
Human-readable fault message
Persistent diagnostic event log
The advanced page shall not require a second password beyond access to the password-protected ESP32 hotspot.
The UI shall not display:
Individual P-term contribution
Individual I-term contribution
Individual D-term contribution
Wi-Fi client count/status
---
12. User-Editable Control Parameters
12.1 Target temperature
Editable from UI
Change shall take effect immediately
Value shall be constrained to 80–110 °C
Value shall be persisted in NVS
12.2 PID parameters
Editable:
Kp
Ki
Kd
PID edits shall not take effect while typing.
The user must press an Apply button.
After Apply:
Values are validated.
Valid values are stored in NVS.
Valid values become active.
V1 shall not provide a Restore Default PID Settings button.
---
13. Persistent Configuration
NVS shall be used to store at least:
Target temperature
Kp
Ki
Kd
Wi-Fi SSID
Wi-Fi password
13.1 Configuration validation
At boot, persisted values shall be validated.
If persisted PID values or setpoint values are invalid or corrupted:
They shall be rejected.
Firmware shall fall back to compiled safe defaults.
---
14. Live Data Transport
The web interface shall use HTTP polling rather than WebSocket.
Polling interval
1 second
The browser shall periodically request fresh controller status using HTTP.
This choice is intended to:
Reduce implementation complexity
Reduce persistent-connection failure modes
Keep the protocol simple for future native-app reuse
---
15. REST API
V1 shall expose a simple REST-style HTTP API using JSON.
The API should be versionable, for example under:
`/api/v1/`
Exact endpoint naming may evolve during implementation, but the API shall support the following logical operations.
15.1 Read current status
Example logical operation:
`GET /api/v1/status`
Response should include at least:
Raw temperature
Filtered temperature
Target temperature
Heater output percentage
Controller state
Heater enabled/disabled
Current fault code
Uptime
Last reset reason
15.2 Update target
Example:
`PUT /api/v1/target`
Payload:
```json
{
  "target_c": 93.0
}
```
15.3 Read PID parameters
Example:
`GET /api/v1/pid`
15.4 Update PID parameters
Example:
`PUT /api/v1/pid`
Payload:
```json
{
  "kp": 1.0,
  "ki": 0.1,
  "kd": 5.0
}
```
15.5 Read history
Example:
`GET /api/v1/history`
15.6 Heater enable/disable
Example:
`PUT /api/v1/control`
15.7 Diagnostics
API shall allow retrieval of:
Fault state
Event log
Reset reason
Uptime
15.8 Wi-Fi settings
API shall allow update of:
AP SSID
AP password
Changes shall become active after reboot.
15.9 OTA
API shall support firmware upload/update operations.
The REST API shall be designed so that a future native mobile application can reuse the same interface.
---
16. Graph and In-Memory History
The system shall maintain a rolling 20-minute history in RAM.
16.1 Sampling interval
Graph data shall be stored at:
1 sample per second
16.2 Capacity
At least:
1,200 records
16.3 Stored graph values
Each graph record shall contain at least:
Relative timestamp / uptime
Filtered temperature
Target temperature
Heater output percentage
16.4 Graph series
The UI shall display three series:
Filtered actual temperature
Target temperature
Heater output percentage
Raw thermocouple temperature shall not be plotted.
16.5 Persistence
Graph history shall be RAM-only.
It shall reset on reboot.
Continuous temperature history shall not be written to flash in V1.
---
17. Persistent Event Logging
V1 shall include lightweight persistent event logging.
Continuous temperature data shall not be persistently logged.
17.1 Log structure
The persistent log shall be a fixed-size circular log containing the most recent:
100 events
17.2 Event timestamp
Because the device operates as a standalone hotspot without guaranteed internet time synchronization, events shall use:
Uptime since boot
Real-world date/time is not required in V1.
17.3 Events to log
At minimum:
Boot
Reset
Watchdog reset indication
Thermocouple fault
Thermocouple recovery
Invalid sensor state
Over-temperature shutdown
PID parameter change
Target temperature change
Wi-Fi configuration change
OTA start
OTA success
OTA failure
---
18. OTA Firmware Update
V1 shall support OTA firmware update through the web UI over the ESP32 hotspot.
Safety rules
Before OTA begins:
PID heating shall be disabled.
SSR shall be forced OFF.
During OTA:
SSR shall remain OFF.
Heating shall not resume until:
OTA has completed successfully.
The device has rebooted.
A valid temperature reading is available.
All safety checks pass.
---
19. Boot Sequence
Recommended boot sequence:
Configure SSR GPIO immediately to safe OFF state.
Initialize logging.
Read reset reason.
Initialize NVS.
Load persisted configuration.
Validate configuration.
Replace invalid values with compiled safe defaults.
Initialize SPI and MAX31855.
Start safety task.
Start sensor task.
Wait for at least one valid thermocouple reading.
Initialize control task.
Start ESP32 Wi-Fi access point.
Start HTTP server.
Start UI/API services.
If no safety fault exists and heater control is enabled:
Enter full-power warm-up if more than 10 °C below target.
Otherwise begin PID control.
---
20. Concurrency Requirements
Implementation shall use ESP-IDF / FreeRTOS tasks.
Recommended logical tasks:
Sensor task
4 Hz
Reads MAX31855
Updates raw temperature
Updates EMA-filtered temperature
Safety task
10 Hz
Highest functional priority among application tasks
Checks sensor validity
Checks over-temperature
Controls global heater permission
Can override PID output
PID/control task
2 Hz
Handles:
Warm-up state
Bumpless transfer
PID calculation
Anti-windup
Requested heater percentage
SSR timing task or timer callback
Implements 1-second time-proportional heater window
Must always obey safety override
History task
1 Hz
Stores graph data in RAM circular buffer
Network/UI task group
Wi-Fi AP
HTTP requests
REST API
Static web interface
OTA
Shared controller state shall be protected using appropriate ESP-IDF synchronization primitives such as:
Mutexes
Critical sections
Queues
Atomics
Networking code shall never directly manipulate SSR GPIO.
---
21. Heater Authority Model
Heater output should follow a strict authority chain.
Conceptually:
`Safety permission`
→ `Manual controller enabled`
→ `Warm-up/PID requested output`
→ `SSR windowing`
→ `GPIO`
Any failure at a higher-priority safety layer shall force output to 0%.
The web server shall never directly command the SSR.
---
22. Functional Scope Excluded from V1
The following are explicitly out of scope:
Steam temperature control
Steam mode
Brew-switch detection
Automatic shot detection
Shot logging
Shot timer
Pressure sensor
Pressure profiling
Pump control
Flow profiling
Native Android app
Native iOS app
Connection to home Wi-Fi
Cloud services
Internet connectivity
Remote access
Real-world clock synchronization
Persistent continuous temperature logs
User-adjustable EMA parameter
Restore-default-PID UI button
The architecture should, where reasonable, avoid making future addition of native-app support or pressure profiling unnecessarily difficult.
---
23. Fault Codes
A simple internal fault enumeration is recommended.
Example:
```c
typedef enum {
    FAULT_NONE = 0,
    FAULT_THERMOCOUPLE_OPEN,
    FAULT_THERMOCOUPLE_SHORT_GND,
    FAULT_THERMOCOUPLE_SHORT_VCC,
    FAULT_SENSOR_INVALID,
    FAULT_SENSOR_TIMEOUT,
    FAULT_OVERTEMP,
    FAULT_INTERNAL
} fault_code_t;
```
The exact MAX31855 fault mapping shall follow the driver implementation.
The UI shall show:
Internal fault code
Human-readable description
---
24. Data Model
A central controller-state structure is recommended.
Example conceptual structure:
```c
typedef struct {
    float temp_raw_c;
    float temp_filtered_c;
    float target_c;

    float kp;
    float ki;
    float kd;

    float heater_output_pct;

    bool heater_enabled;
    bool sensor_valid;
    bool overtemp_latched;

    controller_state_t state;
    fault_code_t fault;

    uint64_t uptime_ms;
} controller_status_t;
```
Exact implementation may use separate structures for:
Configuration
Runtime state
Safety state
UI snapshot
to reduce lock contention.
---
25. Acceptance Criteria
V1 shall be considered functionally complete when all of the following are satisfied.
Temperature acquisition
MAX31855 is sampled at 4 Hz.
Raw and filtered values are available.
EMA filtering operates continuously.
PID control
PID updates at 2 Hz.
Derivative is based on measurement.
Integral anti-windup is implemented.
Output remains between 0% and 100%.
Warm-up
Heater runs at 100% when temperature is more than 10 °C below target.
Transition to PID occurs at target minus 10 °C.
Transition is bumpless.
SSR
1-second proportional control window is used.
Safety override always forces OFF.
Stability indication
Stable is entered only after 10 continuous seconds inside ±1.0 °C.
Stable exits when deviation exceeds ±1.5 °C.
Safety
Heater remains OFF before first valid temperature reading.
Sensor failure forces immediate heater shutdown.
Sensor-related fault automatically recovers after valid readings return.
Temperature at or above 120 °C forces immediate shutdown.
Over-temperature remains latched until reboot.
Watchdog reset results in heater OFF during restart.
Wi-Fi
ESP32 provides password-protected access point.
No home Wi-Fi is required.
SSID and password can be changed from settings.
Settings persist in NVS.
Web UI
Works from mobile browser.
Displays current temperature.
Displays target.
Displays heater output.
Displays system state.
Allows target change.
Allows PID editing with Apply button.
Provides manual heater disable.
Provides advanced diagnostics.
Graph
Stores 20 minutes of RAM history.
Stores one record per second.
Displays actual filtered temperature.
Displays target temperature.
Displays heater output percentage.
Persistence
Target and PID settings survive reboot.
Invalid persisted values fall back to safe defaults.
100-event persistent circular diagnostic log is maintained.
OTA
Firmware can be updated through web UI.
Heater is forced OFF before and during OTA.
---
26. Future V2 Considerations
The following should be considered during V1 design but not implemented unless required later:
Native mobile app using the same REST API
Home Wi-Fi client mode
BLE provisioning
Pressure sensing
Pump control
Pressure profiling
Shot detection
Shot history
Steam-mode control
Multiple temperature profiles
Real-time clock
Exportable logs
Automatic PID tuning
Additional physical display or controls
---
27. Summary of Fixed V1 Parameters
Parameter	V1 Value
Sensor sampling	4 Hz
PID update	2 Hz
Safety task	10 Hz
UI polling	1 Hz
Graph logging	1 Hz
Graph history	20 minutes
Graph records	~1,200
SSR window	1 second
Default target	93 °C
Allowed target range	80–110 °C
Full-power warm-up threshold	Until 10 °C below target
Stable band	±1.0 °C
Stable dwell	10 seconds
Stable exit threshold	±1.5 °C
Software over-temperature cutoff	120 °C
Event-log capacity	100 events
Persistent temperature history	No
Steam control	No
Brew detection	No
Wi-Fi mode	ESP32 Access Point only
UI transport	HTTP polling
Native app	Future version
