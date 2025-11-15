# Home Assistant Automation and Control Logic Implementation

## Status

Proposed

## Context

With the migration to Home Assistant (ADR-0001), we need to determine how to implement the automation and control logic currently in the custom backend and firmware.

**Current Plantalytix Automation:**

### Firmware-Based Control (ESP32)
- **PID Controllers**: Temperature, humidity, CO2 control
- **Hysteresis Mode**: Simple on/off with deadband
- **Schedule-Based**: Day/night cycles for lighting
- **Manual Mode**: Direct user control of outputs
- **Automation Modes**: Configurable per device type

### API-Based Control
- **OTA Firmware Updates**: Managed rollout with concurrency limits
- **Configuration Push**: Real-time config updates via MQTT
- **Test Mode**: Temporary output control for diagnostics

**Example Fridge Controller Logic (Current):**
```cpp
// PID temperature control
void runPIDControl() {
  float error = targetTemp - currentTemp;
  float pTerm = kp * error;
  float iTerm += ki * error * dt;
  float dTerm = kd * (error - lastError) / dt;

  float output = pTerm + iTerm + dTerm;
  output = constrain(output, 0, 100);

  if (mode == COOL) {
    setDehumidifier(output);
  } else if (mode == HEAT) {
    setHeater(output);
  }

  lastError = error;
}

// Schedule-based lighting
void runLightSchedule() {
  int hour = rtc.getHour();

  if (hour >= dayStart && hour < nightStart) {
    setLight(lightDayBrightness);
  } else {
    setLight(lightNightBrightness);
  }
}
```

**Home Assistant Automation Options:**

### 1. Built-in Automations (YAML/UI)

**Example**:
```yaml
automation:
  - alias: "Fridge Temperature Control"
    trigger:
      - platform: state
        entity_id: sensor.plantalytix_fridge_001_temperature
    condition:
      - condition: numeric_state
        entity_id: climate.plantalytix_fridge_001
        attribute: temperature
        above: 20
    action:
      - service: switch.turn_on
        target:
          entity_id: switch.plantalytix_fridge_001_heater
```

**Pros**: Simple, built-in, GUI editor
**Cons**: Limited for complex logic like PID

### 2. Node-RED

**Visual flow-based programming**:
- Nodes for triggers, logic, actions
- PID node available
- Dashboard for visualization

**Pros**: Visual programming, active community, PID support
**Cons**: Another service to run, learning curve

### 3. AppDaemon (Python)

**Python-based automation**:
```python
import hassapi as hass

class FridgeController(hass.Hass):
    def initialize(self):
        self.listen_state(self.temp_changed, "sensor.plantalytix_fridge_001_temperature")
        self.pid = PIDController(kp=1.0, ki=0.5, kd=0.1)

    def temp_changed(self, entity, attribute, old, new, kwargs):
        current = float(new)
        target = float(self.get_state("climate.plantalytix_fridge_001", attribute="temperature"))

        output = self.pid.update(current, target)

        if output > 0:
            self.call_service("switch/turn_on", entity_id="switch.plantalytix_fridge_001_heater")
        else:
            self.call_service("switch/turn_off", entity_id="switch.plantalytix_fridge_001_heater")
```

**Pros**: Full Python power, complex logic, testable
**Cons**: Requires Python knowledge, separate service

### 4. Firmware-Based (Keep Current)

**Keep automation in ESP32 firmware**:
- Devices operate autonomously
- HA monitors and configures
- No network dependency for critical control

**Pros**: Autonomous operation, low latency, proven code
**Cons**: Hard to modify without firmware update

## Decision

**We will use a layered approach combining firmware-based control for critical functions with Home Assistant automations for non-critical features.**

### Control Logic Distribution

| Function | Implementation | Rationale |
|----------|---------------|-----------|
| **PID Temperature Control** | ESP32 Firmware | Low-latency, autonomous, safety-critical |
| **PID Humidity Control** | ESP32 Firmware | Low-latency, autonomous |
| **CO2 Injection Control** | ESP32 Firmware | Low-latency, autonomous |
| **Day/Night Scheduling** | HA Automation + Firmware | HA sets mode, firmware executes |
| **Safety Shutoffs** | ESP32 Firmware | Must work without network |
| **User Notifications** | HA Automation | HA notification system |
| **Multi-Device Scenes** | HA Automation | Coordinate multiple devices |
| **Data Logging** | HA Automation | Trigger InfluxDB writes |
| **Firmware Updates** | HA Automation + Custom Integration | Orchestrated rollout |
| **Test Mode** | HA Automation | Temporary override |

### Architecture

```
┌─────────────────────────────────────────────────┐
│           Home Assistant Layer                   │
│  - Scheduling (sun-based, time-based)           │
│  - Multi-device coordination                    │
│  - Notifications & alerts                       │
│  - User interface & dashboards                  │
│  - Non-critical automations                     │
└──────────────┬──────────────────────────────────┘
               │ MQTT (set target, mode)
               ↓
┌─────────────────────────────────────────────────┐
│            ESP32 Firmware Layer                  │
│  - PID control loops (10s cycle)                │
│  - Safety logic (temp limits, failsafes)        │
│  - Sensor reading & filtering                   │
│  - Output control (PWM, relays)                 │
│  - Autonomous operation (no network)            │
└─────────────────────────────────────────────────┘
```

### Implementation Details

**1. Firmware PID Control (Unchanged)**

Keep existing PID implementation in firmware for:
- Fast response (10-second cycles)
- Autonomous operation
- Safety guarantees

**Configuration from HA**:
```yaml
# Home Assistant climate entity controls firmware
climate:
  - platform: mqtt
    name: "Fridge Climate"
    temperature_command_topic: "plantalytix/001/set/temperature"
    mode_command_topic: "plantalytix/001/set/mode"
```

User adjusts target temperature in HA → MQTT command → Firmware updates PID setpoint

**2. HA Automations for Scheduling**

Replace firmware RTC-based scheduling with HA automations:

```yaml
# Day/Night cycle based on sun
automation:
  - alias: "Fridge Grow - Day Cycle"
    trigger:
      - platform: sun
        event: sunrise
        offset: "+01:00"
    action:
      - service: light.turn_on
        target:
          entity_id: light.plantalytix_fridge_001
        data:
          brightness_pct: 100

  - alias: "Fridge Grow - Night Cycle"
    trigger:
      - platform: sun
        event: sunset
    action:
      - service: light.turn_on
        target:
          entity_id: light.plantalytix_fridge_001
        data:
          brightness_pct: 10
```

**Benefits**: Sun-based triggers, timezone handling, easier to modify

**3. AppDaemon for Complex Logic (Optional)**

For features that need complex logic beyond YAML:

```python
# apps/firmware_updater.py
import hassapi as hass
from datetime import datetime

class FirmwareUpdater(hass.Hass):
    def initialize(self):
        # Schedule firmware updates during off-hours
        self.run_daily(self.check_updates, "02:00:00")
        self.max_concurrent = 3
        self.updating_devices = []

    def check_updates(self, kwargs):
        # Get all Plantalytix devices
        devices = self.get_devices()

        for device in devices:
            pending_fw = self.get_state(f"sensor.{device}_pending_firmware")

            if pending_fw and len(self.updating_devices) < self.max_concurrent:
                self.start_firmware_update(device)

    def start_firmware_update(self, device):
        self.log(f"Starting firmware update for {device}")
        self.updating_devices.append(device)

        # Publish firmware URL to device
        self.call_service("mqtt/publish",
            topic=f"plantalytix/{device}/firmware",
            payload=self.get_firmware_url(device)
        )

        # Monitor for completion (10 minute timeout)
        self.run_in(self.check_update_complete, 600, device=device)
```

**4. Node-RED for Visual Workflows (Optional)**

For users who prefer visual programming:

```
[Sensor: Temperature] → [Function: Calculate Avg] → [Switch: Alert if > threshold]
                                                   → [Notify: Send notification]
```

**Use Cases**:
- Custom dashboards
- Complex data transformations
- Integration with external services

**5. Safety Logic (Firmware)**

Critical safety checks remain in firmware:

```cpp
void checkSafetyLimits() {
  // Emergency shutoff if temperature too high
  if (currentTemp > MAX_TEMP) {
    setHeater(0);
    setLight(0);
    publishAlert("CRITICAL: Temperature exceeded safe limit");
  }

  // Emergency shutoff if humidity too high
  if (currentHumidity > MAX_HUMIDITY) {
    setDehumidifier(100);
    publishAlert("WARNING: High humidity detected");
  }

  // Failsafe: If no MQTT connection for 1 hour, enter safe mode
  if (millis() - lastMqttMessage > 3600000) {
    enterSafeMode();
  }
}
```

**Rationale**: Safety-critical logic cannot depend on network connectivity

## Consequences

### Positive

✅ **Autonomous Operation**: Devices work without network/HA

✅ **Low Latency**: PID loops run on-device at 10-second intervals

✅ **Safety**: Critical shutoffs always active

✅ **Flexibility**: Non-critical automation easily modified in HA

✅ **User-Friendly**: HA GUI for most automation changes

✅ **Powerful**: AppDaemon for complex logic when needed

✅ **Visual Programming**: Node-RED option for less technical users

✅ **Reusable**: Existing PID tuning and firmware logic retained

✅ **Testable**: Firmware logic tested in isolation

✅ **Scalable**: HA automations coordinate multiple devices

### Negative

❌ **Complexity**: Logic split across firmware and HA

❌ **Debugging**: Harder to debug distributed logic

❌ **Learning Curve**: Team must learn HA automation syntax

❌ **Firmware Updates**: Still need OTA for control logic changes

❌ **State Synchronization**: HA and firmware state must stay in sync

❌ **AppDaemon Overhead**: Another service if complex logic needed

❌ **Testing**: Must test firmware and HA automations separately

### Risks

⚠️ **State Divergence**: HA thinks device is in mode A, firmware in mode B

⚠️ **Network Dependency**: Scheduling depends on HA being online

⚠️ **Automation Bugs**: Incorrect HA automation could damage crops

⚠️ **Firmware Complexity**: Firmware still contains significant logic

### Mitigation Strategies

1. **State Validation**: Firmware publishes state, HA reconciles
2. **Watchdog Timers**: Firmware reverts to safe defaults if HA offline
3. **Automation Testing**: Test harness for HA automations
4. **Gradual Migration**: Keep firmware scheduling as fallback initially
5. **Documentation**: Clear separation of firmware vs. HA responsibilities

## Alternatives Considered

### Alternative 1: Full HA Automation (Move PID to HA)

**Description**: Implement PID controllers in AppDaemon, firmware only reads sensors and controls outputs

**Pros**:
- All logic in HA (single source of truth)
- Easier to modify without firmware updates
- Centralized monitoring

**Cons**:
- Network dependency for critical control
- Higher latency (MQTT roundtrip)
- Single point of failure
- Not suitable for safety-critical applications

**Verdict**: Rejected - unacceptable for agriculture where network outages could kill crops

### Alternative 2: Full Firmware Automation (HA Monitoring Only)

**Description**: Keep all automation in firmware, HA just displays state

**Pros**:
- Fully autonomous devices
- Proven code
- No new automation logic needed

**Cons**:
- Firmware updates required for automation changes
- Can't coordinate multiple devices
- Misses HA ecosystem benefits (notifications, schedules)

**Verdict**: Rejected - defeats purpose of HA migration (ADR-0001)

### Alternative 3: Node-RED Only

**Description**: Use Node-RED for all automation, bypass HA automations

**Pros**:
- Visual programming for all logic
- Powerful data processing
- Active community

**Cons**:
- Another service to maintain
- HA automations unused
- Learning curve for team
- Not integrated with HA UI

**Verdict**: Rejected - prefer HA-native automations with Node-RED as optional

### Alternative 4: ESPHome Automation

**Description**: Use ESPHome's built-in automation features

**Pros**:
- YAML-based automation on-device
- Fast execution
- Autonomous operation

**Cons**:
- Limited to ESPHome devices (see ADR-0003)
- Less flexible than firmware C++
- Harder to debug than HA automations

**Verdict**: Considered for ESPHome devices, but fridge controllers will stay Arduino

## Implementation Plan

### Phase 1: HA Automation Foundation (2 weeks)

**Week 1: Basic Automations**
1. Create HA automations for day/night scheduling
2. Test sun-based triggers
3. Create notification automations (alerts, warnings)
4. Document automation patterns

**Week 2: Testing**
1. Test all automations with real devices
2. Validate state synchronization
3. Test network failure scenarios
4. Document edge cases

### Phase 2: AppDaemon Setup (1 week, if needed)

1. Install AppDaemon add-on
2. Create firmware updater app
3. Create advanced monitoring app
4. Test Python-based automations

### Phase 3: Node-RED Setup (1 week, optional)

1. Install Node-RED add-on
2. Create example flows
3. Integrate with InfluxDB for data processing
4. Create custom dashboard

### Phase 4: Firmware Safety Enhancements (1 week)

1. Add watchdog for HA connectivity
2. Implement safe mode (if HA offline > 1 hour)
3. Add state reconciliation logic
4. Test autonomous operation

### Phase 5: Migration & Testing (2 weeks)

1. Migrate scheduling from firmware to HA
2. Update firmware to remove deprecated scheduling
3. Test with real grow environment
4. Monitor for issues

## Automation Examples

### Basic HA Automations

**Temperature Alert**:
```yaml
automation:
  - alias: "Fridge Temperature Alert"
    trigger:
      - platform: numeric_state
        entity_id: sensor.plantalytix_fridge_001_temperature
        above: 30
    action:
      - service: notify.mobile_app
        data:
          title: "High Temperature Alert"
          message: "Fridge 001 temperature is {{ states('sensor.plantalytix_fridge_001_temperature') }}°C"
          data:
            priority: high
```

**Coordinated Multi-Device Scene**:
```yaml
scene:
  - name: "All Fridges Night Mode"
    entities:
      climate.plantalytix_fridge_001:
        temperature: 18
      climate.plantalytix_fridge_002:
        temperature: 18
      light.plantalytix_fridge_001:
        brightness: 10
      light.plantalytix_fridge_002:
        brightness: 10

automation:
  - alias: "Night Mode - All Fridges"
    trigger:
      - platform: time
        at: "22:00:00"
    action:
      - service: scene.turn_on
        target:
          entity_id: scene.all_fridges_night_mode
```

**Adaptive Climate Control**:
```yaml
automation:
  - alias: "Adaptive VPD Control"
    trigger:
      - platform: state
        entity_id:
          - sensor.plantalytix_fridge_001_temperature
          - sensor.plantalytix_fridge_001_humidity
    action:
      - service: climate.set_temperature
        target:
          entity_id: climate.plantalytix_fridge_001
        data:
          temperature: >
            {% set temp = states('sensor.plantalytix_fridge_001_temperature') | float %}
            {% set humidity = states('sensor.plantalytix_fridge_001_humidity') | float %}
            {# Calculate VPD and adjust target temperature #}
            {% set vpd = ((saturation_pressure(temp) - actual_pressure(temp, humidity)) / 1000) %}
            {% if vpd < 0.8 %}
              {{ temp + 1 }}
            {% elif vpd > 1.2 %}
              {{ temp - 1 }}
            {% else %}
              {{ temp }}
            {% endif %}
```

### AppDaemon Example

**Advanced Firmware Update Orchestration**:
```python
import hassapi as hass
import datetime

class FirmwareOrchestrator(hass.Hass):
    def initialize(self):
        self.max_concurrent = int(self.args.get("max_concurrent", 3))
        self.max_failures = int(self.args.get("max_failures", 5))

        self.updating = {}
        self.failed = {}

        # Monitor firmware status
        self.listen_state(self.on_firmware_status, "sensor.", attribute="all", namespace="mqtt")

        # Daily update check at 2 AM
        self.run_daily(self.check_for_updates, "02:00:00")

    def check_for_updates(self, kwargs):
        devices = self.get_devices_needing_update()
        class_failures = self.get_class_failures()

        for device_class, devices_list in devices.items():
            # Skip class if too many failures
            if class_failures.get(device_class, 0) >= self.max_failures:
                self.log(f"Skipping {device_class} - too many failures")
                continue

            # Update devices up to concurrent limit
            for device in devices_list:
                if len(self.updating) < self.max_concurrent:
                    self.start_update(device)

    def start_update(self, device_id):
        self.log(f"Starting firmware update for {device_id}")

        firmware_url = self.get_firmware_url(device_id)

        # Publish firmware URL
        self.call_service("mqtt/publish",
            topic=f"plantalytix/{device_id}/firmware",
            payload=firmware_url
        )

        # Track update start
        self.updating[device_id] = datetime.datetime.now()

        # Set timeout (10 minutes)
        self.run_in(self.update_timeout, 600, device_id=device_id)

    def on_firmware_status(self, entity, attribute, old, new, kwargs):
        device_id = self.extract_device_id(entity)

        if device_id in self.updating:
            current_version = new.get("state")
            expected_version = self.get_expected_version(device_id)

            if current_version == expected_version:
                self.update_success(device_id)
            elif (datetime.datetime.now() - self.updating[device_id]).seconds > 600:
                self.update_failed(device_id)

    def update_success(self, device_id):
        self.log(f"Firmware update succeeded for {device_id}")
        del self.updating[device_id]

        # Send notification
        self.call_service("notify/mobile_app",
            title="Firmware Update Success",
            message=f"Device {device_id} updated successfully"
        )

    def update_failed(self, device_id):
        self.log(f"Firmware update failed for {device_id}")
        del self.updating[device_id]

        device_class = self.get_device_class(device_id)
        self.failed[device_class] = self.failed.get(device_class, 0) + 1

        # Send alert
        self.call_service("notify/mobile_app",
            title="Firmware Update Failed",
            message=f"Device {device_id} update failed",
            data={"priority": "high"}
        )
```

### Node-RED Flow (JSON)

**VPD Monitoring and Alert**:
```json
[
    {
        "id": "vpd_calc",
        "type": "function",
        "name": "Calculate VPD",
        "func": "const temp = msg.payload.temperature;\nconst humidity = msg.payload.humidity;\n\n// Saturation vapor pressure (kPa)\nconst svp = 0.6108 * Math.exp((17.27 * temp) / (temp + 237.3));\n\n// Actual vapor pressure\nconst avp = (humidity / 100) * svp;\n\n// VPD\nconst vpd = svp - avp;\n\nmsg.payload.vpd = vpd;\n\nreturn msg;",
        "outputs": 1
    },
    {
        "id": "vpd_check",
        "type": "switch",
        "name": "VPD Range Check",
        "property": "payload.vpd",
        "rules": [
            {"t": "lt", "v": "0.8"},
            {"t": "btwn", "v": "0.8", "v2": "1.2"},
            {"t": "gt", "v": "1.2"}
        ],
        "outputs": 3
    },
    {
        "id": "alert_low",
        "type": "ha-api",
        "name": "Alert: VPD Low",
        "service": "notify.mobile_app",
        "data": "{\"message\": \"VPD too low: {{payload.vpd}}\"}"
    }
]
```

## References

- [Home Assistant Automations](https://www.home-assistant.io/docs/automation/)
- [AppDaemon Documentation](https://appdaemon.readthedocs.io/)
- [Node-RED Documentation](https://nodered.org/docs/)
- [ESPHome Automations](https://esphome.io/guides/automations.html)
- [PID Control Theory](https://en.wikipedia.org/wiki/PID_controller)

## Notes

**Critical Design Principle**: Safety-critical control logic (PID, emergency shutoffs) MUST remain in firmware. Home Assistant automations should handle convenience, coordination, and monitoring - never life-safety functions.

**Testing Requirements**:
- [ ] Devices operate autonomously without HA
- [ ] HA automations trigger correctly
- [ ] State synchronization between HA and devices
- [ ] Network failure scenarios
- [ ] Firmware safety logic activates appropriately
- [ ] AppDaemon apps (if used) handle errors gracefully
- [ ] Automation changes don't require firmware updates

**Decision Date**: 2025-11-15
**Review Date**: After Phase 5 migration completion
