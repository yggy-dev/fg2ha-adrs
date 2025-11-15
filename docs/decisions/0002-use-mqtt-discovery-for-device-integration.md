# Use MQTT Discovery for Device Integration with Home Assistant

## Status

Proposed

## Context

With the decision to migrate to Home Assistant (see ADR-0001), we need to determine how ESP32 devices will integrate with the platform.

**Current Device Registration Approach:**
- Devices self-register via API (`POST /device/register`)
- User claims device via claim code in web app
- Device configuration stored in MongoDB
- Devices publish to custom MQTT topics (`/devices/{device_id}/*`)

**Home Assistant Integration Options:**

### Option 1: MQTT Discovery Protocol
Home Assistant supports automatic device discovery via MQTT messages published to special topics following the pattern:
```
homeassistant/{component}/{unique_id}/config
```

**Discovery Payload Example (Climate Entity):**
```json
{
  "name": "Fridge Climate Controller",
  "unique_id": "plantalytix_fridge_abc123",
  "device": {
    "identifiers": ["plantalytix_abc123"],
    "name": "Plantalytix Fridge #123",
    "model": "FridgeGrow 2.0",
    "manufacturer": "Plantalytix",
    "sw_version": "1.0.0"
  },
  "modes": ["off", "cool", "heat", "auto"],
  "temperature_state_topic": "plantalytix/abc123/state",
  "temperature_command_topic": "plantalytix/abc123/set/temperature",
  "mode_state_topic": "plantalytix/abc123/state",
  "mode_command_topic": "plantalytix/abc123/set/mode",
  "current_temperature_topic": "plantalytix/abc123/sensor/temperature",
  "availability_topic": "plantalytix/abc123/status",
  "payload_available": "online",
  "payload_not_available": "offline"
}
```

**Benefits:**
- Zero-configuration device addition
- Automatic entity creation in HA
- Device registry integration
- Standardized topic structure

**Challenges:**
- Firmware must implement discovery protocol
- More complex MQTT topic structure
- Must adhere to HA conventions

### Option 2: Manual YAML Configuration
Devices configured manually in Home Assistant's `configuration.yaml`:

```yaml
mqtt:
  climate:
    - name: "Fridge Climate Controller"
      unique_id: plantalytix_fridge_abc123
      modes: ["off", "cool", "heat", "auto"]
      temperature_state_topic: "plantalytix/abc123/state"
      # ... etc
```

**Benefits:**
- Simple firmware (no discovery messages)
- Full control over configuration
- Easier debugging

**Challenges:**
- Manual configuration for each device
- No automatic device discovery
- Difficult for end users
- Doesn't scale to many devices

### Option 3: ESPHome Native API
Migrate firmware to ESPHome, which uses native API instead of MQTT:

```yaml
# esphome/fridge-abc123.yaml
climate:
  - platform: thermostat
    name: "Fridge Climate"
    sensor: temperature_sensor
    # ...
```

**Benefits:**
- Automatic discovery via ESPHome API
- No MQTT required (direct connection)
- Simplified firmware development
- OTA updates handled by ESPHome

**Challenges:**
- Complete firmware rewrite
- Learning curve for ESPHome
- Different architecture than current MQTT approach
- Covered in ADR-0003

**Current Plantalytix Device Types and Required HA Components:**

| Device Type | HA Component(s) | MQTT Discovery Support |
|-------------|----------------|------------------------|
| Fridge/Climate Controller | `climate`, `sensor` (temp/humidity/CO2), `binary_sensor` (outputs) | ✅ Yes |
| Fan Controller | `fan`, `sensor` (temp/humidity/RPM) | ✅ Yes |
| Light Controller | `light`, `sensor` | ✅ Yes |
| Smart Socket | `switch`, `sensor` | ✅ Yes |
| Camera | `camera` | ✅ Yes |

All components support MQTT discovery.

## Decision

**We will implement MQTT Discovery protocol for Plantalytix devices to integrate with Home Assistant.**

### Implementation Approach

**1. Firmware Changes (Arduino-based):**
- Add MQTT discovery message publishing on boot and reconnection
- Restructure MQTT topics to follow HA conventions
- Implement HA-compatible state/command topic handling
- Subscribe to Home Assistant birth message (`homeassistant/status`)
- Republish discovery on birth message receipt

**2. Topic Structure Migration:**

**From (Current):**
```
/devices/{device_id}/status          → Device publishes all state
/devices/{device_id}/configuration   → Device receives config
/devices/{device_id}/firmware        → Device receives FW update
/devices/{device_id}/log             → Device publishes logs
```

**To (MQTT Discovery):**
```
# Discovery topics (retained)
homeassistant/climate/plantalytix_{device_id}/config
homeassistant/sensor/plantalytix_{device_id}_temp/config
homeassistant/sensor/plantalytix_{device_id}_humidity/config
homeassistant/sensor/plantalytix_{device_id}_co2/config

# State topics (not retained)
plantalytix/{device_id}/state                    → Climate state
plantalytix/{device_id}/sensor/temperature       → Temperature
plantalytix/{device_id}/sensor/humidity          → Humidity
plantalytix/{device_id}/sensor/co2               → CO2
plantalytix/{device_id}/status                   → Availability (online/offline)

# Command topics (not retained)
plantalytix/{device_id}/set/temperature          → Set target temp
plantalytix/{device_id}/set/mode                 → Set mode (auto/heat/cool/off)
plantalytix/{device_id}/set/fan_mode             → Set fan mode

# Legacy topics (for backward compatibility during migration)
plantalytix/{device_id}/firmware                 → OTA updates
plantalytix/{device_id}/log                      → Device logs (custom)
```

**3. Discovery Message Structure:**

**Climate Entity (Fridge/Fan Controllers):**
```json
{
  "name": "Plantalytix Climate",
  "unique_id": "plantalytix_climate_{device_id}",
  "device": {
    "identifiers": ["plantalytix_{device_id}"],
    "name": "Plantalytix {device_type} #{serial}",
    "model": "FridgeGrow 2.0 / FanControl / LightControl / Socket",
    "manufacturer": "Plantalytix",
    "sw_version": "{firmware_version}",
    "configuration_url": "http://{device_ip}"
  },
  "modes": ["off", "auto", "cool", "heat", "dry", "fan_only"],
  "fan_modes": ["auto", "low", "medium", "high"],
  "temperature_unit": "C",
  "precision": 0.1,
  "min_temp": 5,
  "max_temp": 35,
  "temp_step": 0.5,
  "current_temperature_topic": "plantalytix/{device_id}/sensor/temperature",
  "temperature_state_topic": "plantalytix/{device_id}/state",
  "temperature_state_template": "{{ value_json.target_temp }}",
  "temperature_command_topic": "plantalytix/{device_id}/set/temperature",
  "mode_state_topic": "plantalytix/{device_id}/state",
  "mode_state_template": "{{ value_json.mode }}",
  "mode_command_topic": "plantalytix/{device_id}/set/mode",
  "availability_topic": "plantalytix/{device_id}/status",
  "payload_available": "online",
  "payload_not_available": "offline",
  "qos": 1,
  "retain": false
}
```

**Sensor Entities:**
```json
{
  "name": "Temperature",
  "unique_id": "plantalytix_temp_{device_id}",
  "device": {
    "identifiers": ["plantalytix_{device_id}"]
  },
  "device_class": "temperature",
  "state_class": "measurement",
  "unit_of_measurement": "°C",
  "state_topic": "plantalytix/{device_id}/sensor/temperature",
  "value_template": "{{ value_json.temperature }}",
  "availability_topic": "plantalytix/{device_id}/status",
  "qos": 1,
  "expire_after": 300
}
```

**4. Firmware Publish Sequence:**

```cpp
void publishDiscovery() {
  // 1. Publish device discovery (climate)
  publishDiscoveryClimate();

  // 2. Publish sensor discoveries
  publishDiscoverySensor("temperature", "Temperature", "temperature", "°C");
  publishDiscoverySensor("humidity", "Humidity", "humidity", "%");
  publishDiscoverySensor("co2", "CO2", "carbon_dioxide", "ppm");

  // 3. Publish output state sensors (for debugging)
  publishDiscoverySensor("out_heater", "Heater Output", "power_factor", "%");
  publishDiscoverySensor("out_fan", "Fan Output", "power_factor", "%");

  // 4. Mark device as available
  publishAvailability("online");
}

void setup() {
  // ... WiFi connection
  // ... MQTT connection

  // Subscribe to HA birth message
  mqtt.subscribe("homeassistant/status");

  // Subscribe to command topics
  mqtt.subscribe("plantalytix/{device_id}/set/temperature");
  mqtt.subscribe("plantalytix/{device_id}/set/mode");

  // Publish discovery messages
  publishDiscovery();
}

void onMqttMessage(String topic, String payload) {
  if (topic == "homeassistant/status" && payload == "online") {
    // HA restarted, republish discovery
    publishDiscovery();
  }
  else if (topic.endsWith("/set/temperature")) {
    float temp = payload.toFloat();
    setTargetTemperature(temp);
    publishState(); // Acknowledge change
  }
  else if (topic.endsWith("/set/mode")) {
    setMode(payload); // "auto", "cool", "heat", "off"
    publishState();
  }
}
```

**5. State Publishing:**

```cpp
void publishState() {
  StaticJsonDocument<512> doc;
  doc["mode"] = currentMode; // "auto", "cool", "heat", "off"
  doc["target_temp"] = targetTemperature;
  doc["current_temp"] = currentTemperature;
  doc["fan_mode"] = currentFanMode;

  char buffer[512];
  serializeJson(doc, buffer);
  mqtt.publish("plantalytix/{device_id}/state", buffer);
}

void publishSensors() {
  // Individual sensor topics
  mqtt.publish("plantalytix/{device_id}/sensor/temperature",
               String(currentTemperature).c_str());
  mqtt.publish("plantalytix/{device_id}/sensor/humidity",
               String(currentHumidity).c_str());
  mqtt.publish("plantalytix/{device_id}/sensor/co2",
               String(currentCO2).c_str());
}
```

**6. Backward Compatibility:**

During migration, support both old and new topic structures:
- Publish to both `/devices/{device_id}/status` (old) and new topics
- Subscribe to both old config topics and new command topics
- Gradual deprecation of old topics

## Consequences

### Positive

✅ **Zero-Config Device Addition**: Users add devices by powering them on; HA discovers automatically

✅ **Device Registry Integration**: Devices appear in HA device registry with metadata

✅ **Standardized Integration**: Follows HA best practices and conventions

✅ **Rich Entity Support**: Automatic support for climate, sensors, switches, lights

✅ **Entity Customization**: Users can rename entities, change icons in HA UI

✅ **Availability Tracking**: HA knows when devices are online/offline

✅ **State Synchronization**: HA state reflects actual device state

✅ **Automation Ready**: Entities immediately available in HA automations

✅ **Multi-Device Support**: Easy to scale to many devices

✅ **Community Familiarity**: Other HA users understand the integration

### Negative

❌ **Firmware Complexity**: More complex than current simple status publishing

❌ **Topic Proliferation**: Many topics per device (discovery + state + commands)

❌ **JSON Parsing**: Firmware must parse JSON commands (ArduinoJson library required)

❌ **Discovery Message Size**: Large discovery payloads (512-1024 bytes each)

❌ **Retained Messages**: Discovery messages must be retained on broker

❌ **Topic Migration**: Existing devices require firmware update to new topic structure

❌ **Testing Overhead**: Must test all discovery scenarios (reconnect, HA restart, etc.)

❌ **No API Fallback**: Loses current API-based device management

### Risks

⚠️ **Breaking Changes**: Existing users must update firmware and reconfigure

⚠️ **Discovery Timing**: Race conditions if discovery sent before HA ready

⚠️ **Broker Load**: Many retained messages with multiple devices

⚠️ **Firmware Size**: Larger firmware due to JSON and discovery logic

### Mitigation Strategies

1. **Gradual Rollout**: Support both old and new protocols during transition
2. **Firmware Testing**: Extensive testing of discovery process and edge cases
3. **Documentation**: Clear migration guide for users
4. **Discovery Optimization**: Send minimal discovery payloads, use templates
5. **Broker Configuration**: Ensure MQTT broker configured for retained messages
6. **Fallback Mode**: Allow manual YAML config if discovery fails

## Alternatives Considered

### Alternative 1: Manual YAML Configuration Only

**Verdict**: Rejected - doesn't scale, poor user experience

### Alternative 2: Custom Home Assistant Integration Component

**Description**: Create custom Python integration for Plantalytix devices

**Pros**:
- More control over entity creation
- Can implement custom logic
- Python-based configuration UI

**Cons**:
- Requires Python development
- Must maintain custom component
- Devices still need MQTT or API communication
- Duplicates what MQTT discovery provides

**Verdict**: Rejected - unnecessary complexity when MQTT discovery sufficient

### Alternative 3: ESPHome Migration (No MQTT Discovery)

**Description**: Migrate to ESPHome, use native API instead of MQTT

**Pros**:
- Automatic discovery via ESPHome integration
- No MQTT discovery complexity
- ESPHome handles all HA communication

**Cons**:
- Complete firmware rewrite (covered in ADR-0003)
- Larger architectural change

**Verdict**: Deferred to ADR-0003 - MQTT discovery allows gradual migration

### Alternative 4: Hybrid API + MQTT Discovery

**Description**: Keep Node.js API for device management, use MQTT discovery for HA integration

**Pros**:
- Maintains current device management approach
- HA integration as add-on
- Flexibility for non-HA users

**Cons**:
- Duplicate systems (API and HA)
- Increased complexity
- Defeats purpose of HA migration

**Verdict**: Rejected - conflicts with ADR-0001 decision to fully migrate to HA

## Implementation Plan

### Phase 1: Prototype (1 week)

1. Set up MQTT broker with retention enabled
2. Create test firmware with discovery for one device type (fan controller)
3. Test discovery process with Home Assistant
4. Validate climate control and sensor entities

### Phase 2: Full Discovery Implementation (2 weeks)

1. Implement discovery for all device types:
   - Fridge/climate controller
   - Fan controller
   - Light controller
   - Smart socket
2. Add birth message handling
3. Implement state/command topic handling
4. Test availability tracking

### Phase 3: Backward Compatibility (1 week)

1. Add dual-publishing (old and new topics)
2. Test migration from old to new firmware
3. Create migration documentation

### Phase 4: Testing & Validation (1 week)

1. Test discovery edge cases (HA restart, device restart, network issues)
2. Load testing with multiple devices
3. Validate state synchronization
4. Test automation triggers

## Example Device Types

### Fridge/Climate Controller Discovery

**Components**: 1 climate, 3 sensors (temp/humidity/CO2), 5 binary_sensors (outputs)

**Discovery Topics**:
- `homeassistant/climate/plantalytix_{id}/config`
- `homeassistant/sensor/plantalytix_{id}_temp/config`
- `homeassistant/sensor/plantalytix_{id}_humidity/config`
- `homeassistant/sensor/plantalytix_{id}_co2/config`
- `homeassistant/binary_sensor/plantalytix_{id}_heater/config`
- `homeassistant/binary_sensor/plantalytix_{id}_dehumidifier/config`
- `homeassistant/binary_sensor/plantalytix_{id}_co2_output/config`

### Fan Controller Discovery

**Components**: 1 fan, 2 sensors (temp/humidity), 1 sensor (RPM)

**Discovery Topics**:
- `homeassistant/fan/plantalytix_{id}/config`
- `homeassistant/sensor/plantalytix_{id}_temp/config`
- `homeassistant/sensor/plantalytix_{id}_humidity/config`
- `homeassistant/sensor/plantalytix_{id}_rpm/config`

### Light Controller Discovery

**Components**: 1 light, 1 sensor (light level)

**Discovery Topics**:
- `homeassistant/light/plantalytix_{id}/config`
- `homeassistant/sensor/plantalytix_{id}_light/config`

## References

- [Home Assistant MQTT Discovery](https://www.home-assistant.io/integrations/mqtt/#mqtt-discovery)
- [MQTT Climate Platform](https://www.home-assistant.io/integrations/climate.mqtt/)
- [MQTT Sensor Platform](https://www.home-assistant.io/integrations/sensor.mqtt/)
- [MQTT Fan Platform](https://www.home-assistant.io/integrations/fan.mqtt/)
- [MQTT Light Platform](https://www.home-assistant.io/integrations/light.mqtt/)

## Notes

**Discovery Message Retention**: The MQTT broker MUST be configured to retain discovery messages. Otherwise, if Home Assistant restarts, it won't rediscover devices. Devices should republish discovery on receiving the HA birth message.

**Testing Checklist**:
- [ ] Device boots → HA discovers automatically
- [ ] HA restarts → Devices remain discovered
- [ ] Device restarts → HA shows unavailable then available
- [ ] Set temperature from HA → Device updates
- [ ] Device temperature changes → HA updates
- [ ] Network disconnects → HA shows unavailable
- [ ] Multiple devices → All discovered correctly

**Decision Date**: 2025-11-15
**Review Date**: After Phase 1 prototype completion
