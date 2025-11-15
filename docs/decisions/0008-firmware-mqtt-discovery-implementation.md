# Firmware MQTT Discovery Implementation Guide

## Status

Proposed

## Context

Following ADR-0002's decision to use MQTT Discovery, we need a detailed implementation guide for modifying existing Arduino firmware to support the Home Assistant MQTT Discovery protocol.

**Current Firmware MQTT Topics**:
```
/devices/{device_id}/status          → All state in one message
/devices/{device_id}/configuration   → Receive config
/devices/{device_id}/firmware        → Receive FW updates
/devices/{device_id}/log             → Send logs
```

**Required HA MQTT Discovery Topics**:
```
# Discovery (retained)
homeassistant/climate/plantalytix_{device_id}/config
homeassistant/sensor/plantalytix_{device_id}_temp/config
homeassistant/sensor/plantalytix_{device_id}_humidity/config

# State
plantalytix/{device_id}/state
plantalytix/{device_id}/sensor/temperature
plantalytix/{device_id}/status

# Commands
plantalytix/{device_id}/set/temperature
plantalytix/{device_id}/set/mode
```

## Decision

**Implement MQTT Discovery in Arduino firmware using a phased approach with backward compatibility during transition.**

### Implementation Strategy

**Phase 1: Add Discovery Alongside Current Protocol**
- Keep existing `/devices/{device_id}/*` topics
- Add new `plantalytix/{device_id}/*` topics
- Publish to both (dual-publishing)
- Firmware version: v2.0.0

**Phase 2: Discovery as Primary**
- New devices use only discovery protocol
- Old devices continue dual-publishing
- Firmware version: v2.1.0

**Phase 3: Deprecate Old Protocol**
- All devices migrated to discovery
- Remove old topic code
- Firmware version: v3.0.0

## Implementation Details

### 1. Firmware Structure Changes

**New Files**:
```
firmware/src/
├── main.cpp                    # Existing
├── fridgecloud.cpp/.h          # Modified - add discovery
├── ha_discovery.cpp/.h         # NEW - Discovery message builder
└── ha_mqtt.cpp/.h              # NEW - HA-specific MQTT handlers
```

### 2. Discovery Message Builder

```cpp
// ha_discovery.h
#ifndef HA_DISCOVERY_H
#define HA_DISCOVERY_H

#include <Arduino.h>
#include <ArduinoJson.h>

class HADiscovery {
public:
    HADiscovery(const char* deviceId, const char* deviceType);

    // Publish all discovery messages
    void publishAll();

    // Individual discovery publishers
    void publishClimateDiscovery();
    void publishSensorDiscovery(const char* sensor, const char* name,
                                 const char* deviceClass, const char* unit);
    void publishBinarySensorDiscovery(const char* sensor, const char* name);

    // Device info
    void setDeviceInfo(const char* model, const char* manufacturer,
                       const char* swVersion);

private:
    String _deviceId;
    String _deviceType;
    String _model;
    String _manufacturer;
    String _swVersion;

    String getDeviceInfoJson();
    void publishDiscovery(const char* component, const char* objectId,
                         const char* config);
};

#endif
```

```cpp
// ha_discovery.cpp
#include "ha_discovery.h"
#include "mqtt_client.h"  // Your existing MQTT client

HADiscovery::HADiscovery(const char* deviceId, const char* deviceType)
    : _deviceId(deviceId), _deviceType(deviceType) {
}

void HADiscovery::setDeviceInfo(const char* model, const char* manufacturer,
                                 const char* swVersion) {
    _model = model;
    _manufacturer = manufacturer;
    _swVersion = swVersion;
}

String HADiscovery::getDeviceInfoJson() {
    StaticJsonDocument<512> doc;
    JsonArray identifiers = doc.createNestedArray("identifiers");
    identifiers.add("plantalytix_" + _deviceId);

    doc["name"] = "Plantalytix " + _deviceType + " " + _deviceId;
    doc["model"] = _model;
    doc["manufacturer"] = _manufacturer;
    doc["sw_version"] = _swVersion;
    doc["configuration_url"] = "http://" + WiFi.localIP().toString();

    String output;
    serializeJson(doc, output);
    return output;
}

void HADiscovery::publishClimateDiscovery() {
    StaticJsonDocument<1024> doc;

    doc["name"] = "Plantalytix Climate";
    doc["unique_id"] = "plantalytix_climate_" + _deviceId;

    // Device info
    JsonObject device = doc.createNestedObject("device");
    device["identifiers"][0] = "plantalytix_" + _deviceId;
    device["name"] = "Plantalytix " + _deviceType + " " + _deviceId;
    device["model"] = _model;
    device["manufacturer"] = _manufacturer;
    device["sw_version"] = _swVersion;

    // Climate capabilities
    JsonArray modes = doc.createNestedArray("modes");
    modes.add("off");
    modes.add("auto");
    modes.add("cool");
    modes.add("heat");
    modes.add("dry");

    doc["temperature_unit"] = "C";
    doc["precision"] = 0.1;
    doc["min_temp"] = 5;
    doc["max_temp"] = 35;
    doc["temp_step"] = 0.5;

    // Topics
    doc["current_temperature_topic"] = "plantalytix/" + _deviceId + "/sensor/temperature";
    doc["temperature_state_topic"] = "plantalytix/" + _deviceId + "/state";
    doc["temperature_state_template"] = "{{ value_json.target_temp }}";
    doc["temperature_command_topic"] = "plantalytix/" + _deviceId + "/set/temperature";
    doc["mode_state_topic"] = "plantalytix/" + _deviceId + "/state";
    doc["mode_state_template"] = "{{ value_json.mode }}";
    doc["mode_command_topic"] = "plantalytix/" + _deviceId + "/set/mode";
    doc["availability_topic"] = "plantalytix/" + _deviceId + "/status";
    doc["payload_available"] = "online";
    doc["payload_not_available"] = "offline";

    doc["qos"] = 1;
    doc["retain"] = false;

    String config;
    serializeJson(doc, config);

    String topic = "homeassistant/climate/plantalytix_" + _deviceId + "/config";
    mqttClient.publish(topic.c_str(), config.c_str(), true);  // Retained

    Serial.println("Published climate discovery: " + topic);
}

void HADiscovery::publishSensorDiscovery(const char* sensor, const char* name,
                                          const char* deviceClass, const char* unit) {
    StaticJsonDocument<768> doc;

    doc["name"] = name;
    doc["unique_id"] = "plantalytix_" + _deviceId + "_" + String(sensor);

    // Device info (link to same device)
    JsonObject device = doc.createNestedObject("device");
    device["identifiers"][0] = "plantalytix_" + _deviceId;

    doc["device_class"] = deviceClass;
    doc["state_class"] = "measurement";
    doc["unit_of_measurement"] = unit;
    doc["state_topic"] = "plantalytix/" + _deviceId + "/sensor/" + String(sensor);
    doc["availability_topic"] = "plantalytix/" + _deviceId + "/status";
    doc["qos"] = 1;
    doc["expire_after"] = 300;  // 5 minutes

    String config;
    serializeJson(doc, config);

    String topic = "homeassistant/sensor/plantalytix_" + _deviceId + "_" +
                   String(sensor) + "/config";
    mqttClient.publish(topic.c_str(), config.c_str(), true);  // Retained

    Serial.println("Published sensor discovery: " + topic);
}

void HADiscovery::publishAll() {
    Serial.println("Publishing HA discovery messages...");

    // Climate entity
    publishClimateDiscovery();
    delay(100);  // Avoid overwhelming broker

    // Sensors
    publishSensorDiscovery("temperature", "Temperature", "temperature", "°C");
    delay(100);
    publishSensorDiscovery("humidity", "Humidity", "humidity", "%");
    delay(100);
    publishSensorDiscovery("co2", "CO2", "carbon_dioxide", "ppm");
    delay(100);

    // Output state sensors (for debugging)
    publishSensorDiscovery("out_heater", "Heater Output", "power_factor", "%");
    delay(100);
    publishSensorDiscovery("out_fan", "Fan Output", "power_factor", "%");
    delay(100);

    // Availability
    mqttClient.publish(("plantalytix/" + _deviceId + "/status").c_str(),
                       "online", true);

    Serial.println("Discovery messages published!");
}
```

### 3. Main Firmware Integration

```cpp
// main.cpp - Modified setup()
#include "ha_discovery.h"

HADiscovery* haDiscovery = nullptr;
bool discoveryPublished = false;

void setup() {
    Serial.begin(115200);

    // Existing WiFi setup
    setupWiFi();

    // Existing MQTT setup
    setupMQTT();

    // NEW: Initialize HA Discovery
    haDiscovery = new HADiscovery(deviceId, deviceType);
    haDiscovery->setDeviceInfo(
        "FridgeGrow 2.0",
        "Plantalytix",
        FIRMWARE_VERSION
    );

    // Subscribe to HA birth message
    mqttClient.subscribe("homeassistant/status");

    // Subscribe to command topics
    String tempCmd = "plantalytix/" + String(deviceId) + "/set/temperature";
    String modeCmd = "plantalytix/" + String(deviceId) + "/set/mode";
    mqttClient.subscribe(tempCmd.c_str());
    mqttClient.subscribe(modeCmd.c_str());

    // Existing sensor setup
    setupSensors();
    setupOutputs();

    Serial.println("Setup complete!");
}

void loop() {
    // Existing MQTT loop
    mqttClient.loop();

    // Publish discovery on first connection
    if (mqttClient.connected() && !discoveryPublished) {
        delay(2000);  // Wait for connection to stabilize
        haDiscovery->publishAll();
        discoveryPublished = true;
    }

    // Existing sensor reading and control logic
    unsigned long now = millis();
    if (now - lastSensorRead > 10000) {  // Every 10 seconds
        readSensors();
        runAutomation();
        publishState();
        publishSensors();
        lastSensorRead = now;
    }

    // Existing UI update
    updateDisplay();

    delay(100);
}
```

### 4. MQTT Message Handlers

```cpp
// ha_mqtt.cpp
void onMqttMessage(String topic, String payload) {
    Serial.println("MQTT message: " + topic + " = " + payload);

    // HA birth message - republish discovery
    if (topic == "homeassistant/status" && payload == "online") {
        Serial.println("Home Assistant restarted - republishing discovery");
        discoveryPublished = false;  // Trigger republish in loop()
        return;
    }

    // Temperature setpoint command
    if (topic.endsWith("/set/temperature")) {
        float temp = payload.toFloat();
        if (temp >= 5.0 && temp <= 35.0) {
            targetTemperature = temp;
            saveSettings();
            publishState();  // Acknowledge
            Serial.println("Target temperature set to: " + String(temp));
        }
        return;
    }

    // Mode command
    if (topic.endsWith("/set/mode")) {
        if (payload == "off") {
            automationMode = MODE_OFF;
        } else if (payload == "auto") {
            automationMode = MODE_AUTO;
        } else if (payload == "cool") {
            automationMode = MODE_COOL;
        } else if (payload == "heat") {
            automationMode = MODE_HEAT;
        }
        saveSettings();
        publishState();
        Serial.println("Mode set to: " + payload);
        return;
    }

    // Legacy topics (backward compatibility)
    if (topic.startsWith("/devices/")) {
        handleLegacyMessage(topic, payload);
        return;
    }
}
```

### 5. State Publishing

```cpp
// fridgecloud.cpp - Modified state publishing
void publishState() {
    StaticJsonDocument<512> doc;

    // Climate state
    switch (automationMode) {
        case MODE_OFF:  doc["mode"] = "off"; break;
        case MODE_AUTO: doc["mode"] = "auto"; break;
        case MODE_COOL: doc["mode"] = "cool"; break;
        case MODE_HEAT: doc["mode"] = "heat"; break;
    }

    doc["target_temp"] = targetTemperature;
    doc["current_temp"] = currentTemperature;

    char buffer[512];
    serializeJson(doc, buffer);

    String topic = "plantalytix/" + String(deviceId) + "/state";
    mqttClient.publish(topic.c_str(), buffer);
}

void publishSensors() {
    // Individual sensor topics
    String baseTopic = "plantalytix/" + String(deviceId) + "/sensor/";

    mqttClient.publish((baseTopic + "temperature").c_str(),
                       String(currentTemperature, 1).c_str());
    mqttClient.publish((baseTopic + "humidity").c_str(),
                       String(currentHumidity, 1).c_str());
    mqttClient.publish((baseTopic + "co2").c_str(),
                       String(currentCO2, 0).c_str());

    // Output states
    mqttClient.publish((baseTopic + "out_heater").c_str(),
                       String(heaterOutput, 0).c_str());
    mqttClient.publish((baseTopic + "out_fan").c_str(),
                       String(fanOutput, 0).c_str());

    // Also publish to legacy topic for backward compatibility
    publishLegacyStatus();
}
```

### 6. Configuration via platformio.ini

```ini
[env:fridge-ha]
platform = espressif32
board = heltec_wifi_lora_32_V2
framework = arduino

build_flags =
    -DHWTYPE_FRIDGE
    -DFG_API_URL=\"${FG_API_URL}\"
    -DFG_MQTT_HOST=\"${FG_MQTT_HOST}\"
    -DFIRMWARE_VERSION=\"2.0.0\"
    -DHA_DISCOVERY_ENABLED=1

lib_deps =
    ArduinoJson@^6.21.3
    PubSubClient@^2.8
    Sensirion I2C SHT4x@^0.1.0
    Sensirion I2C SCD4x@^0.4.0

build_src_filter =
    +<*>
    +<../src_hwtype/fridge>
```

## Testing Procedure

### 1. Discovery Testing

**Test Case 1: Initial Discovery**
```
1. Flash firmware to device
2. Device connects to MQTT broker
3. Device publishes discovery messages
4. Verify in Home Assistant:
   - Configuration → Devices → MQTT
   - Device appears automatically
   - All entities created (climate + sensors)
```

**Test Case 2: Home Assistant Restart**
```
1. Restart Home Assistant
2. HA publishes birth message to homeassistant/status
3. Device receives birth message
4. Device republishes discovery
5. Verify device reappears in HA without manual intervention
```

**Test Case 3: Device Restart**
```
1. Restart device (power cycle)
2. Device reconnects to MQTT
3. Device publishes discovery and availability
4. Verify HA shows device as available
```

### 2. State Synchronization Testing

**Test Case 4: Temperature Control**
```
1. Set target temperature in HA UI: 22°C
2. Verify MQTT message published to plantalytix/{id}/set/temperature
3. Verify device receives command
4. Verify device publishes updated state
5. Verify HA UI updates to show 22°C target
```

**Test Case 5: Mode Changes**
```
1. Change mode in HA UI: auto → heat
2. Verify device receives mode command
3. Verify device automation changes behavior
4. Verify state published back to HA
```

### 3. Sensor Data Testing

**Test Case 6: Sensor Updates**
```
1. Device reads sensors every 10 seconds
2. Device publishes to individual sensor topics
3. Verify HA sensor entities update
4. Verify history graph shows data points
5. Verify no data loss or gaps
```

### 4. Availability Testing

**Test Case 7: Network Disconnection**
```
1. Disconnect device from network
2. Verify HA shows device as "unavailable" after timeout
3. Reconnect device
4. Verify HA shows device as "available"
5. Verify sensor data resumes
```

## Rollout Strategy

### Week 1: Development
- Implement HA discovery code
- Test with single device on bench
- Validate all test cases above

### Week 2: Beta Testing
- Deploy to 3-5 beta devices in production
- Monitor for issues
- Gather feedback

### Week 3: Gradual Rollout
- Deploy to 25% of devices
- Monitor stability
- Fix any issues

### Week 4: Full Deployment
- Deploy to remaining 75% of devices
- Monitor migration
- Provide support

## Backward Compatibility

During transition, firmware supports both protocols:

```cpp
#define LEGACY_SUPPORT 1  // Set to 0 to disable legacy topics

void publishSensors() {
    // New HA discovery topics
    publishHASensors();

    #if LEGACY_SUPPORT
    // Old topics (deprecated)
    publishLegacyStatus();
    #endif
}
```

**Migration Path**:
1. **v2.0.0**: Dual publishing (both old and new)
2. **v2.1.0**: New devices only use HA discovery
3. **v3.0.0**: Remove legacy support entirely

## Monitoring and Debugging

### MQTT Topic Monitor

Monitor topics to validate discovery:
```bash
# Subscribe to all Plantalytix topics
mosquitto_sub -h mqtt.plantalytix.com -t 'plantalytix/#' -v

# Subscribe to all discovery topics
mosquitto_sub -h mqtt.plantalytix.com -t 'homeassistant/#' -v

# Monitor specific device
mosquitto_sub -h mqtt.plantalytix.com -t 'plantalytix/device-001/#' -v
```

### Serial Debug Output

Add verbose logging:
```cpp
#define DEBUG_HA_DISCOVERY 1

void HADiscovery::publishClimateDiscovery() {
    #if DEBUG_HA_DISCOVERY
    Serial.println("=== Climate Discovery ===");
    Serial.println("Topic: " + topic);
    Serial.println("Config: " + config);
    Serial.println("========================");
    #endif

    mqttClient.publish(topic.c_str(), config.c_str(), true);
}
```

### Home Assistant Logs

Check HA logs for discovery issues:
```bash
# In Home Assistant
tail -f /config/home-assistant.log | grep mqtt

# Or via HA UI
Settings → System → Logs → Filter: mqtt
```

## Common Issues and Solutions

### Issue 1: Discovery Not Working

**Symptoms**: Device doesn't appear in HA

**Debug Steps**:
1. Verify MQTT broker allows retained messages
2. Check HA MQTT integration is enabled
3. Verify discovery prefix matches (default: `homeassistant`)
4. Check serial output for publish errors
5. Use MQTT client to verify messages published

**Solution**:
```yaml
# configuration.yaml - Verify MQTT config
mqtt:
  discovery: true
  discovery_prefix: homeassistant
```

### Issue 2: Entities Not Updating

**Symptoms**: HA shows old sensor values

**Debug Steps**:
1. Verify sensor topics match discovery config
2. Check QoS settings (should be 1)
3. Verify device is publishing regularly
4. Check HA logs for errors

**Solution**: Ensure topic names exactly match between discovery config and state publishing

### Issue 3: Device Shows "Unavailable"

**Symptoms**: Device appears but marked unavailable

**Debug Steps**:
1. Verify availability topic published
2. Check payload exactly matches: "online" / "offline"
3. Verify LWT (Last Will Testament) configured

**Solution**:
```cpp
// Set Last Will Testament
mqttClient.setWill(
    ("plantalytix/" + deviceId + "/status").c_str(),
    "offline",
    true,  // retained
    1      // QoS
);
```

## References

- [MQTT Discovery Documentation](https://www.home-assistant.io/integrations/mqtt/#mqtt-discovery)
- [MQTT Climate](https://www.home-assistant.io/integrations/climate.mqtt/)
- [MQTT Sensor](https://www.home-assistant.io/integrations/sensor.mqtt/)
- [ArduinoJson Library](https://arduinojson.org/)
- [PubSubClient Library](https://github.com/knolleary/pubsubclient)

## Appendix: Complete Example Firmware

See GitHub repository: `/firmware/examples/ha_discovery_fridge/` for complete working example.

---

**Decision Date**: 2025-11-15
**Implementation Status**: Ready for Development
**Target Firmware Version**: v2.0.0
