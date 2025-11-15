# Home Assistant UI and Dashboard Strategy

## Status

Proposed

## Context

With the migration to Home Assistant (ADR-0001), we need to replace the custom Angular/Ionic frontend with Home Assistant's UI system.

**Current Plantalytix Frontend:**
- **Framework**: Angular 15 + Ionic 6
- **Pages**: Login, Device List, Device Charts, Device Settings, Device Test, Diagnostics (Admin), Account
- **Charts**: Highcharts for time-series visualization
- **Mobile**: Ionic components, Capacitor for native builds
- **Features**: Real-time data polling (10s), device configuration, firmware management, claim codes

**User Workflows**:
1. User logs in → Views device list
2. Selects device → Views real-time charts (temperature, humidity, CO2)
3. Adjusts settings → Sends configuration to device
4. Admin manages firmware updates and device classes

**Home Assistant UI System:**

### Lovelace Dashboard System

HA's frontend is built on **Lovelace** (YAML or UI editor):
- **Cards**: Modular UI components (entity card, history graph, gauge, button)
- **Views**: Pages within a dashboard
- **Dashboards**: Separate dashboards per purpose
- **Themes**: Customizable CSS themes
- **Custom Cards**: Community-built cards via HACS

**Built-in Card Types**:
- **Entity Cards**: Display sensor values, controls
- **History Graph**: Time-series charts
- **Gauge**: Circular/linear gauges for sensors
- **Thermostat**: Climate entity control
- **Button**: Action triggers
- **Markdown**: Text content
- **Picture Elements**: Interactive image overlays
- **Conditional**: Show/hide based on state

**Custom Card Ecosystem (HACS)**:
- **Mini Graph Card**: Compact time-series graphs
- **ApexCharts Card**: Advanced charting (like Highcharts)
- **Mushroom Cards**: Modern, minimalist UI
- **Button Card**: Highly customizable buttons
- **Auto-Entities**: Dynamic card generation
- **Layout Cards**: Grid, masonry, vertical/horizontal layouts

### Home Assistant Companion App

Official mobile app (iOS/Android):
- Auto-syncs with HA instance
- Push notifications
- Location tracking
- Widgets
- Quick actions

## Decision

**We will use Home Assistant's Lovelace dashboard system with custom cards (ApexCharts, Mini Graph) to replace the Angular frontend, supplemented by custom integrations for Plantalytix-specific features.**

### Dashboard Structure

**1. Device Overview Dashboard**

Replaces current "Device List" page:

```yaml
# Dashboard: Device Overview
views:
  - title: My Devices
    path: devices
    type: masonry
    cards:
      # Auto-generate device cards using auto-entities
      - type: custom:auto-entities
        card:
          type: entities
          title: Climate Controllers
        filter:
          include:
            - domain: climate
              attributes:
                device_class: plantalytix
        sort:
          method: name

      # Status summary
      - type: custom:mushroom-chips-card
        chips:
          - type: entity
            entity: sensor.plantalytix_devices_online
            icon: mdi:check-network
          - type: entity
            entity: sensor.plantalytix_devices_total
            icon: mdi:devices
```

**2. Device Detail Dashboard**

Replaces "Device Charts" and "Device Settings" pages:

```yaml
# Dashboard: Fridge Controller 001 (template)
views:
  - title: Overview
    path: overview
    cards:
      # Climate control
      - type: thermostat
        entity: climate.plantalytix_fridge_001
        name: Climate Control

      # Current readings (replaces Angular component)
      - type: horizontal-stack
        cards:
          - type: gauge
            entity: sensor.plantalytix_fridge_001_temperature
            min: 0
            max: 40
            severity:
              green: 15
              yellow: 25
              red: 30
          - type: gauge
            entity: sensor.plantalytix_fridge_001_humidity
            min: 0
            max: 100
          - type: gauge
            entity: sensor.plantalytix_fridge_001_co2
            min: 0
            max: 2000

  - title: Charts
    path: charts
    cards:
      # Temperature history (replaces Highcharts)
      - type: custom:apexcharts-card
        header:
          show: true
          title: Temperature History
        graph_span: 24h
        series:
          - entity: sensor.plantalytix_fridge_001_temperature
            name: Temperature
            stroke_width: 2
            type: line
          - entity: climate.plantalytix_fridge_001
            attribute: temperature
            name: Target
            stroke_width: 1
            type: line
            curve: stepline

      # Multi-sensor chart
      - type: custom:apexcharts-card
        header:
          title: Environmental Conditions
        graph_span: 24h
        all_series_config:
          stroke_width: 2
        series:
          - entity: sensor.plantalytix_fridge_001_temperature
            name: Temp
            yaxis_id: temp
          - entity: sensor.plantalytix_fridge_001_humidity
            name: Humidity
            yaxis_id: humidity
          - entity: sensor.plantalytix_fridge_001_co2
            name: CO2
            yaxis_id: co2
        yaxis:
          - id: temp
            min: 0
            max: 40
          - id: humidity
            min: 0
            max: 100
            opposite: true
          - id: co2
            min: 0
            max: 2000
            opposite: true

  - title: Settings
    path: settings
    cards:
      # Device configuration
      - type: entities
        title: Automation Settings
        entities:
          - entity: input_select.plantalytix_fridge_001_mode
            name: Automation Mode
          - entity: input_number.plantalytix_fridge_001_pid_p
            name: PID - P Gain
          - entity: input_number.plantalytix_fridge_001_pid_i
            name: PID - I Gain
          - entity: input_number.plantalytix_fridge_001_pid_d
            name: PID - D Gain

      # Output controls
      - type: entities
        title: Output Status
        entities:
          - entity: binary_sensor.plantalytix_fridge_001_heater
            name: Heater
          - entity: binary_sensor.plantalytix_fridge_001_dehumidifier
            name: Dehumidifier
          - entity: binary_sensor.plantalytix_fridge_001_fan
            name: Fan

  - title: Diagnostics
    path: diagnostics
    cards:
      # Device info
      - type: entities
        title: Device Information
        entities:
          - entity: sensor.plantalytix_fridge_001_firmware_version
          - entity: sensor.plantalytix_fridge_001_wifi_signal
          - entity: sensor.plantalytix_fridge_001_uptime
          - entity: sensor.plantalytix_fridge_001_ip_address

      # Logs (custom integration)
      - type: custom:plantalytix-device-logs
        entity: sensor.plantalytix_fridge_001_logs
```

**3. Admin Dashboard**

Replaces "Diagnostics" and "Classes" admin pages:

```yaml
# Dashboard: Admin (restricted to admin users)
views:
  - title: Fleet Overview
    path: fleet
    cards:
      # All devices status
      - type: custom:auto-entities
        card:
          type: entities
        filter:
          include:
            - domain: climate
              attributes:
                integration: plantalytix
          sort:
            method: last_changed

      # Firmware management (custom card)
      - type: custom:plantalytix-firmware-manager
        title: Firmware Updates

  - title: Device Classes
    path: classes
    cards:
      # Custom integration for device class management
      - type: custom:plantalytix-device-classes
```

### Custom Integrations for Plantalytix Features

Some features don't map directly to HA, requiring custom integrations:

**1. Device Claiming (Custom Integration)**

```python
# custom_components/plantalytix/config_flow.py
class PlantalytixConfigFlow(config_entries.ConfigFlow, domain=DOMAIN):
    """Handle device claiming via claim code."""

    async def async_step_user(self, user_input=None):
        if user_input is not None:
            claim_code = user_input["claim_code"]
            # Validate claim code via MQTT or custom API
            device_id = await self.validate_claim_code(claim_code)

            if device_id:
                # Add device to HA
                return self.async_create_entry(
                    title=f"Plantalytix Device {device_id}",
                    data={"device_id": device_id}
                )

        return self.async_show_form(
            step_id="user",
            data_schema=vol.Schema({
                vol.Required("claim_code"): str
            })
        )
```

**UI**: HA Settings → Devices & Services → Add Integration → Plantalytix → Enter Claim Code

**2. Firmware Management (Custom Card)**

```javascript
// www/community/plantalytix-firmware-manager/plantalytix-firmware-manager.js
class PlantalytixFirmwareManager extends HTMLElement {
  setConfig(config) {
    this.config = config;
  }

  set hass(hass) {
    this._hass = hass;

    // Render firmware update UI
    this.innerHTML = `
      <ha-card>
        <div class="card-header">Firmware Updates</div>
        <div class="card-content">
          ${this.renderDeviceList()}
        </div>
      </ha-card>
    `;
  }

  renderDeviceList() {
    // Get all Plantalytix devices
    const devices = Object.keys(this._hass.states)
      .filter(e => e.startsWith('sensor.plantalytix_'))
      .filter(e => e.endsWith('_firmware_version'));

    return devices.map(entity => {
      const state = this._hass.states[entity];
      const deviceId = entity.split('_')[1];
      const currentVersion = state.state;
      const pendingVersion = state.attributes.pending_version;

      return `
        <div class="device-row">
          <span>${deviceId}</span>
          <span>Current: ${currentVersion}</span>
          ${pendingVersion ? `
            <span>Pending: ${pendingVersion}</span>
            <button onclick="this.startUpdate('${deviceId}')">Update</button>
          ` : ''}
        </div>
      `;
    }).join('');
  }

  startUpdate(deviceId) {
    // Call HA service to trigger firmware update
    this._hass.callService('plantalytix', 'update_firmware', {
      device_id: deviceId
    });
  }
}

customElements.define('plantalytix-firmware-manager', PlantalytixFirmwareManager);
```

**3. Device Test Mode (Custom Service)**

```python
# custom_components/plantalytix/services.py
async def async_setup(hass, config):
    """Set up Plantalytix component."""

    async def handle_test_mode(call):
        """Handle test mode service call."""
        device_id = call.data.get("device_id")
        outputs = call.data.get("outputs")  # {"heater": 50, "fan": 100}
        duration = call.data.get("duration", 300)  # seconds

        # Publish test command to MQTT
        await hass.async_add_executor_job(
            publish_test_command,
            device_id,
            outputs,
            duration
        )

    hass.services.async_register(
        DOMAIN,
        "test_mode",
        handle_test_mode,
        schema=vol.Schema({
            vol.Required("device_id"): cv.string,
            vol.Required("outputs"): dict,
            vol.Optional("duration", default=300): cv.positive_int
        })
    )

    return True
```

**Usage in Dashboard**:
```yaml
# Button to trigger test mode
- type: button
  name: Test Heater (50%)
  tap_action:
    action: call-service
    service: plantalytix.test_mode
    data:
      device_id: fridge_001
      outputs:
        heater: 50
      duration: 300
```

### Mobile App Experience

**Home Assistant Companion App**:
- Automatic sync with dashboards
- Push notifications for alerts
- Widgets for quick device status
- Location-based automations (optional)

**Configuration**:
```yaml
# Enable mobile app notifications
notify:
  - name: mobile_app
    platform: mobile_app

# Automation: Send notification on high temperature
automation:
  - alias: "Temperature Alert - Mobile"
    trigger:
      - platform: numeric_state
        entity_id: sensor.plantalytix_fridge_001_temperature
        above: 30
    action:
      - service: notify.mobile_app_user_phone
        data:
          title: "High Temperature Alert"
          message: "Fridge 001 temperature is {{ states('sensor.plantalytix_fridge_001_temperature') }}°C"
          data:
            tag: "temp_alert_fridge_001"
            url: "/lovelace/fridge-001"
            actions:
              - action: "VIEW_DEVICE"
                title: "View Device"
```

### Theme and Branding

**Custom Theme** (Plantalytix branding):
```yaml
# themes/plantalytix.yaml
plantalytix:
  primary-color: "#4CAF50"  # Green for agriculture
  accent-color: "#8BC34A"
  card-background-color: "#FFFFFF"
  primary-text-color: "#212121"
  secondary-text-color: "#757575"
  disabled-text-color: "#BDBDBD"
  paper-card-background-color: "#FFFFFF"
  # ... more theme variables
```

Applied in HA configuration:
```yaml
# configuration.yaml
frontend:
  themes: !include_dir_merge_named themes/
  default_theme: plantalytix
```

## Consequences

### Positive

✅ **Zero Frontend Maintenance**: No custom Angular code to maintain

✅ **Rich Card Ecosystem**: 100+ community cards via HACS

✅ **Mobile App**: Official iOS/Android app with push notifications

✅ **Customizable**: Users can customize their own dashboards

✅ **Responsive**: Works on desktop, tablet, mobile

✅ **Real-time Updates**: WebSocket-based updates (not polling)

✅ **Offline Support**: PWA can work offline

✅ **Accessibility**: Built-in accessibility features

✅ **Themes**: Customizable appearance

✅ **Advanced Charts**: ApexCharts rival Highcharts functionality

### Negative

❌ **Learning Curve**: Team must learn Lovelace YAML/UI editor

❌ **Custom Cards Required**: Some features need custom cards

❌ **Limited Customization**: More constrained than custom Angular

❌ **YAML Verbosity**: Complex dashboards become long YAML files

❌ **Custom Integration Work**: Claim codes, firmware management need custom integrations

❌ **UI Editor Limitations**: Some features only in YAML mode

❌ **Migration Effort**: Must recreate all Angular pages as Lovelace dashboards

### Risks

⚠️ **Feature Parity**: May not perfectly replicate Angular UI

⚠️ **Custom Card Maintenance**: Community cards may break with HA updates

⚠️ **User Confusion**: Different UI paradigm from current system

⚠️ **Performance**: Complex dashboards may be slow on low-end devices

### Mitigation Strategies

1. **Prototype Dashboards**: Create mockups before full migration
2. **User Testing**: Test with actual users during transition
3. **Documentation**: Comprehensive dashboard configuration guide
4. **Fallback Options**: Provide alternative card options if custom cards fail
5. **Incremental Migration**: Roll out new UI to users gradually

## Alternatives Considered

### Alternative 1: Keep Angular Frontend, Integrate with HA Backend

**Description**: Maintain Angular frontend, connect to HA via API

**Pros**:
- Keep familiar UI
- No user retraining
- Full control over UX

**Cons**:
- Continued maintenance of Angular code
- Duplicates HA frontend functionality
- Users don't get HA app benefits

**Verdict**: Rejected - defeats purpose of HA migration

### Alternative 2: Custom HA Frontend (Fork Lovelace)

**Description**: Fork HA frontend and customize heavily

**Pros**:
- Full customization
- HA integration

**Cons**:
- Massive maintenance burden
- Must merge upstream HA changes
- Fragile, likely to break

**Verdict**: Rejected - unsustainable

### Alternative 3: Iframe Angular into HA

**Description**: Embed Angular app in HA via iframe card

**Pros**:
- Reuse existing Angular code
- Quick migration

**Cons**:
- Poor UX (iframe within iframe)
- Authentication complexity
- Mobile app won't work
- Hacky solution

**Verdict**: Rejected - poor user experience

## Implementation Plan

### Phase 1: Dashboard Prototyping (2 weeks)

**Week 1: Core Dashboards**
1. Create device overview dashboard
2. Create single device detail dashboard
3. Test ApexCharts for time-series
4. Test Mushroom cards for modern UI

**Week 2: Custom Integrations**
1. Build device claiming integration (config flow)
2. Build firmware manager custom card
3. Test mobile app experience
4. User testing with stakeholders

### Phase 2: Custom Card Development (2 weeks)

**Week 1: Development**
1. Develop Plantalytix device logs card
2. Develop firmware manager card
3. Develop device class management card

**Week 2: Testing**
1. Test all custom cards
2. Integrate with HA dashboards
3. Test mobile app
4. Performance testing

### Phase 3: Migration (2 weeks)

**Week 1: Template Dashboards**
1. Create dashboard templates for each device type
2. Document dashboard configuration
3. Create theme (Plantalytix branding)

**Week 2: User Rollout**
1. Deploy to test users
2. Gather feedback
3. Iterate on dashboard designs
4. Full production rollout

## Dashboard Configuration Examples

See YAML configurations in Decision section above.

## References

- [Lovelace UI Documentation](https://www.home-assistant.io/dashboards/)
- [ApexCharts Card](https://github.com/RomRider/apexcharts-card)
- [Mushroom Cards](https://github.com/piitaya/lovelace-mushroom)
- [Auto-Entities Card](https://github.com/thomasloven/lovelace-auto-entities)
- [Custom Card Development](https://developers.home-assistant.io/docs/frontend/custom-ui/custom-card/)
- [HA Companion App](https://companion.home-assistant.io/)

## Notes

**User Training**:
- Create video tutorials for dashboard customization
- Provide pre-built dashboard templates
- Document common customizations
- Offer support for dashboard configuration

**Testing Checklist**:
- [ ] Device overview dashboard displays all devices
- [ ] Device detail charts match Angular version
- [ ] Settings can be changed via dashboard
- [ ] Mobile app displays dashboards correctly
- [ ] Custom cards load without errors
- [ ] Theme applied correctly
- [ ] Notifications work on mobile app
- [ ] Performance acceptable on mobile devices

**Decision Date**: 2025-11-15
**Review Date**: After Phase 2 custom card development
