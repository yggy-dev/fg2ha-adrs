# ESPHome vs Arduino Firmware for Device Implementation

## Status

Proposed

## Context

With the decision to migrate to Home Assistant (ADR-0001) and use MQTT discovery (ADR-0002), we must decide on the firmware architecture for ESP32 devices.

**Current Firmware Architecture:**
- **Framework**: Arduino (via PlatformIO)
- **Language**: C++
- **Structure**: Hardware abstraction layer with device-specific implementations
- **Communication**: Custom MQTT protocol
- **OTA Updates**: Custom implementation via HTTP download
- **Configuration**: EEPROM-stored settings
- **UI**: Custom OLED display with rotary encoder
- **LOC**: ~5,000 lines of C++ code across all device types

**Firmware Components:**
- WiFi management
- MQTT client
- Sensor reading (SHT, SCD4x, etc.)
- Control logic (PID, hysteresis, scheduling)
- Display/UI management
- Settings persistence
- OTA update handling

**ESPHome Overview:**

ESPHome is a firmware framework that:
- Uses YAML configuration instead of C++ code
- Automatically generates C++ firmware from YAML
- Integrates natively with Home Assistant (no MQTT required)
- Provides built-in components for common sensors/actuators
- Handles OTA updates automatically
- Includes web interface for diagnostics
- Supports custom C++ components when needed

**ESPHome Example (Climate Controller):**
```yaml
# fridge-controller.yaml
esphome:
  name: plantalytix-fridge-001
  platform: ESP32
  board: heltec_wifi_lora_32_V2

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

api:
  encryption:
    key: !secret api_key

ota:
  password: !secret ota_password

logger:

i2c:
  sda: 4
  scl: 15

sensor:
  - platform: sht3xd
    temperature:
      name: "Temperature"
      id: current_temp
    humidity:
      name: "Humidity"
      id: current_humidity
    update_interval: 10s

  - platform: scd4x
    co2:
      name: "CO2"
      id: current_co2
    update_interval: 30s

climate:
  - platform: thermostat
    name: "Fridge Climate"
    sensor: current_temp
    default_target_temperature_low: 18°C
    default_target_temperature_high: 24°C

    heat_action:
      - switch.turn_on: heater_relay
    idle_action:
      - switch.turn_off: heater_relay

switch:
  - platform: gpio
    pin: 12
    name: "Heater Relay"
    id: heater_relay

  - platform: gpio
    pin: 13
    name: "Dehumidifier Relay"
    id: dehumidifier_relay
```

**Complexity Comparison:**

| Feature | Arduino (Current) | ESPHome |
|---------|-------------------|---------|
| WiFi Setup | ~200 LOC | 3 lines YAML |
| MQTT Client | ~150 LOC | 0 (native API) or ~5 lines (MQTT) |
| OTA Updates | ~100 LOC | 2 lines YAML |
| Sensor Reading | ~300 LOC | ~10 lines YAML per sensor |
| Climate Control | ~500 LOC | ~20 lines YAML |
| Display/UI | ~800 LOC | ~30 lines YAML (if using ESPHome display) |
| Total | ~5,000 LOC C++ | ~200 lines YAML + custom components |

## Decision

**We will keep all existing hardware with Arduino firmware and add MQTT Discovery support. ESPHome is documented as an optional future alternative for new device development, but is NOT required for the migration.**

### Rationale

**ESPHome Advantages:**
1. **Reduced Maintenance**: ~95% less code to maintain for simple devices
2. **Faster Development**: New devices configured in minutes, not days
3. **Built-in HA Integration**: Native API, automatic discovery
4. **Automatic OTA**: ESPHome dashboard handles updates
5. **Community Support**: Large library of component examples
6. **Configuration as Code**: YAML easier to review and modify than C++
7. **Built-in Web Server**: Diagnostics and logging via web UI
8. **Rapid Prototyping**: Test new sensors without C++ compilation

**Arduino Advantages:**
1. **Full Control**: Complete flexibility for complex logic
2. **Custom UI**: OLED display with rotary encoder menu system
3. **Existing Investment**: ~5,000 LOC already written and tested
4. **Performance**: Direct C++ optimization for critical paths
5. **Offline Capability**: Can operate independently of HA
6. **Custom Protocols**: Can implement proprietary communication if needed

**Migration Approach:**

| Device Type | Framework | Status |
|-------------|-----------|--------|
| **Fan Controller** | Arduino + MQTT Discovery | ✅ Keep existing hardware/firmware |
| **Light Controller** | Arduino + MQTT Discovery | ✅ Keep existing hardware/firmware |
| **Smart Socket** | Arduino + MQTT Discovery | ✅ Keep existing hardware/firmware |
| **Fridge Controller** | Arduino + MQTT Discovery | ✅ Keep existing hardware/firmware |
| **All Devices** | Add MQTT Discovery | 🔄 Update firmware to support HA discovery |

**Future Option** (for new device types only):
| **New Devices** | ESPHome (optional) | 📋 Can use ESPHome for rapid prototyping |
| **ESP32-CAM** | Arduino | Camera requires custom implementation |

### Migration Path

**Phase 1: Add MQTT Discovery to Existing Arduino Firmware** (REQUIRED)
- Update all existing Arduino firmware to support MQTT Discovery
- Keep all existing features and hardware as-is
- No device replacement needed
- Firmware update only (OTA)

**Phase 2: Future New Devices** (OPTIONAL)
- For completely new device types, ESPHome can be considered
- Existing devices continue with Arduino
- No migration of existing devices required

**Phase 3: Long-term** (OPTIONAL)
- ESPHome remains available for rapid prototyping
- Arduino firmware continues for production devices
- Hybrid approach is permanent, not transitional

## Consequences

### Positive

✅ **No Hardware Replacement**: All existing devices stay as-is

✅ **Firmware Update Only**: Simple OTA update to add MQTT Discovery

✅ **Proven Hardware**: Keep existing tested ESP32 devices

✅ **Existing Investment Protected**: ~5,000 LOC Arduino code remains useful

✅ **ESPHome Available for Future**: Option available for new device types

✅ **Full Feature Set**: All existing features remain (OLED UI, PID, etc.)

✅ **No Learning Curve Required**: Continue using familiar Arduino/C++

✅ **Flexibility**: Can choose Arduino or ESPHome for future devices

✅ **Risk Minimized**: No device replacement reduces migration risk

### Negative

❌ **Firmware Update Required**: All devices need OTA update for MQTT Discovery

❌ **MQTT Discovery Development**: Must implement discovery protocol in Arduino

❌ **ESPHome Learning Optional**: Team can learn ESPHome for future devices

❌ **Potential Dual Codebase**: If ESPHome adopted for new devices later

### Risks

⚠️ **MQTT Discovery Implementation**: Must correctly implement HA discovery protocol

⚠️ **Firmware Update Deployment**: Must update all deployed devices via OTA

⚠️ **Testing Coverage**: Must test MQTT discovery with all device types

### Mitigation Strategies

1. **Reference Implementation**: Use ADR-0008 as detailed implementation guide
2. **Incremental Rollout**: Test with one device type first, then expand
3. **Backward Compatibility**: Devices continue working even if discovery fails
4. **OTA Testing**: Thorough testing of OTA update process
5. **Rollback Plan**: Keep previous firmware version for emergency rollback

## Alternatives Considered

### Alternative 1: Replace All Devices with ESPHome

**Description**: Replace all existing hardware with new ESPHome-based devices

**Pros**:
- Clean slate approach
- Maximum code reduction
- Unified firmware framework

**Cons**:
- Must replace all working hardware (expensive)
- Loss of existing investment in Arduino firmware
- Complex PID logic difficult in ESPHome
- Custom OLED UI requires significant custom components
- High migration risk

**Verdict**: Rejected - unnecessarily expensive and risky when firmware update suffices

### Alternative 2: Keep Arduino Only, No ESPHome

**Description**: Maintain Arduino firmware exclusively, never use ESPHome

**Pros**:
- No learning curve
- Single firmware framework
- Full control

**Cons**:
- Slower development for future simple devices
- No option for rapid prototyping

**Verdict**: Partially adopted - Arduino primary, ESPHome available as future option

### Alternative 3: Tasmota Firmware

**Description**: Use Tasmota instead of ESPHome

**Pros**:
- Alternative to ESPHome
- Good HA integration
- Active community

**Cons**:
- Less flexible than ESPHome for custom devices
- LUA scripting vs. YAML configuration
- Smaller sensor library than ESPHome

**Verdict**: Rejected - ESPHome better fit for custom hardware

### Alternative 4: Custom Components in ESPHome Only

**Description**: Develop all logic as custom ESPHome C++ components

**Pros**:
- Uses ESPHome infrastructure
- Custom logic when needed
- Single framework

**Cons**:
- Defeats purpose of ESPHome (configuration simplicity)
- Still requires C++ development
- Custom components harder to maintain than standalone firmware

**Verdict**: Rejected - loses ESPHome benefits

## Implementation Plan

### Phase 1: MQTT Discovery Implementation (3 weeks) - REQUIRED

**Week 1: MQTT Discovery Protocol**
1. Study HA MQTT Discovery specification
2. Implement discovery message generation in Arduino
3. Test with single device type (fridge controller)
4. Validate entity creation in HA

**Week 2: All Device Types**
1. Add MQTT Discovery to fan controller firmware
2. Add MQTT Discovery to light controller firmware
3. Add MQTT Discovery to socket controller firmware
4. Test each device type

**Week 3: Deployment**
1. Test OTA update process
2. Deploy firmware updates to all devices
3. Verify all devices discovered by HA
4. Document any issues

### Phase 2: ESPHome Evaluation (Future, Optional)

**Only if new device types needed:**
1. Set up ESPHome development environment
2. Create example configuration for simple device
3. Evaluate for future use
4. **No migration of existing devices**
4. Gather user feedback

### Phase 3: Arduino MQTT Discovery (2 weeks)

**Week 1: Implementation**
1. Add MQTT discovery to Arduino fridge controller firmware
2. Test discovery process with HA
3. Ensure feature parity with current system

**Week 2: Deployment**
1. OTA update existing Arduino devices
2. Validate HA integration
3. Monitor for issues

### Phase 4: Evaluation (Ongoing)

1. Track maintenance time: ESPHome vs. Arduino
2. Collect user feedback on both approaches
3. Evaluate fridge controller migration feasibility
4. Decision point (6 months): Full ESPHome migration or long-term hybrid

## ESPHome Configuration Examples

### Fan Controller (Complete Example)

```yaml
substitutions:
  device_name: plantalytix-fan-001
  friendly_name: "Plantalytix Fan Controller"

esphome:
  name: ${device_name}
  platform: ESP32
  board: esp32dev

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  ap:
    ssid: "${device_name} Fallback"
    password: !secret ap_password

captive_portal:

api:
  encryption:
    key: !secret api_key

ota:
  password: !secret ota_password

logger:

web_server:
  port: 80

i2c:
  sda: 21
  scl: 22

sensor:
  - platform: sht3xd
    temperature:
      name: "${friendly_name} Temperature"
      id: temp_sensor
    humidity:
      name: "${friendly_name} Humidity"
      id: humidity_sensor
    update_interval: 10s

  - platform: pulse_counter
    pin: 25
    name: "${friendly_name} Fan RPM"
    unit_of_measurement: 'RPM'
    filters:
      - multiply: 0.5  # Convert pulses to RPM

output:
  - platform: ledc
    pin: 26
    id: fan_pwm
    frequency: 25000 Hz

fan:
  - platform: speed
    output: fan_pwm
    name: "${friendly_name} Fan"
    id: main_fan

# Temperature-based fan speed automation
interval:
  - interval: 10s
    then:
      - lambda: |-
          float temp = id(temp_sensor).state;
          if (temp > 28.0) {
            id(main_fan).turn_on().set_speed(100).perform();
          } else if (temp > 25.0) {
            id(main_fan).turn_on().set_speed(75).perform();
          } else if (temp > 22.0) {
            id(main_fan).turn_on().set_speed(50).perform();
          } else {
            id(main_fan).turn_on().set_speed(25).perform();
          }
```

### Light Controller (Complete Example)

```yaml
substitutions:
  device_name: plantalytix-light-001
  friendly_name: "Plantalytix Light"

esphome:
  name: ${device_name}
  platform: ESP32
  board: esp32dev

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

api:
  encryption:
    key: !secret api_key

ota:
  password: !secret ota_password

logger:

output:
  - platform: ledc
    pin: 23
    id: led_output
    frequency: 1000 Hz

light:
  - platform: monochromatic
    output: led_output
    name: "${friendly_name}"
    id: main_light
    default_transition_length: 1s

# Day/Night scheduling
time:
  - platform: homeassistant
    id: ha_time

sun:
  latitude: !secret latitude
  longitude: !secret longitude

# Automation: Lights on at sunset
automation:
  - platform: sun
    event: sunset
    then:
      - light.turn_on:
          id: main_light
          brightness: 100%

  - platform: sun
    event: sunrise
    then:
      - light.turn_off: main_light
```

### Fridge Controller (Custom Component Example)

For complex PID logic, use custom component:

```yaml
# In fridge-controller.yaml
esphome:
  name: plantalytix-fridge-001
  includes:
    - custom_components/pid_climate.h

external_components:
  - source:
      type: local
      path: custom_components
    components: [ pid_climate ]

climate:
  - platform: pid_climate
    name: "Fridge Climate"
    sensor: current_temp
    default_target_temperature: 20°C
    heat_output: heater_relay
    cool_output: dehumidifier_relay
    kp: 1.0
    ki: 0.5
    kd: 0.1
```

```cpp
// custom_components/pid_climate.h
#include "esphome.h"

class PIDClimate : public Component, public Climate {
 public:
  void setup() override {
    // Initialize PID controller
  }

  void loop() override {
    // Run PID algorithm
    float error = target_temperature - current_temperature;
    // ... PID calculation
  }

  climate::ClimateTraits traits() override {
    auto traits = climate::ClimateTraits();
    traits.set_supports_current_temperature(true);
    traits.set_supports_two_point_target_temperature(false);
    return traits;
  }

  void control(const climate::ClimateCall &call) override {
    if (call.get_target_temperature().has_value()) {
      target_temperature = *call.get_target_temperature();
    }
  }

 private:
  float target_temperature;
  float current_temperature;
};
```

## References

- [ESPHome Documentation](https://esphome.io/)
- [ESPHome Climate Component](https://esphome.io/components/climate/index.html)
- [ESPHome Custom Components](https://esphome.io/custom/custom_component.html)
- [ESPHome vs Tasmota Comparison](https://esphome.io/guides/faq.html#esphome-vs-tasmota)
- [Arduino PlatformIO Documentation](https://docs.platformio.org/en/latest/frameworks/arduino.html)

## Notes

**Testing Checklist (ESPHome Migration)**:
- [ ] WiFi connection and reconnection
- [ ] Sensor accuracy vs. Arduino version
- [ ] Control responsiveness
- [ ] OTA update process
- [ ] Web interface accessibility
- [ ] HA native API discovery
- [ ] Logs and diagnostics
- [ ] Power consumption comparison
- [ ] Memory usage
- [ ] Boot time

**Decision Criteria for Complex Device Migration**:
After 6 months of ESPHome usage, evaluate:
1. Can PID logic be implemented reliably in ESPHome custom component?
2. Is OLED UI still required, or can HA UI replace it?
3. What is the maintenance burden: Arduino vs. ESPHome?
4. Are users satisfied with ESPHome device reliability?

If answers favor ESPHome, proceed with full migration. Otherwise, maintain hybrid approach long-term.

**Decision Date**: 2025-11-15
**Review Date**: 6 months after ESPHome Phase 2 completion
