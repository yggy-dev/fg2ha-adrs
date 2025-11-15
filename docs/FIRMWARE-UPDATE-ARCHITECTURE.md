# Plantalytix Firmware Update System Architecture

This document provides a visual overview of the firmware update management system described in [ADR-0009](./decisions/0009-firmware-update-management-system.md).

## System Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                    Firmware Sources                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │  GitHub Releases│  │ GitLab Releases │  │  Custom HTTP/S  │ │
│  │                 │  │                 │  │                 │ │
│  │ plantalytix/    │  │ community/      │  │ firmware.       │ │
│  │ firmware        │  │ beta-firmware   │  │ example.com     │ │
│  │                 │  │                 │  │                 │ │
│  │ • Official      │  │ • Community     │  │ • Custom ESPHome│ │
│  │ • Stable        │  │ • Beta          │  │ • Experimental  │ │
│  │ • Tagged        │  │ • Testing       │  │ • Private       │ │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘ │
│           │                    │                     │          │
└───────────┼────────────────────┼─────────────────────┼──────────┘
            │                    │                     │
            └────────────────────┴─────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │   Home Assistant        │
                    │   Custom Integration    │
                    │   plantalytix_firmware  │
                    └────────────┬────────────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
        │                        │                        │
┌───────┴────────┐  ┌───────────┴──────────┐  ┌─────────┴────────┐
│ Source Manager │  │  Update Coordinator  │  │  Update Entities │
│                │  │                      │  │                  │
│ • Fetch        │  │ • Version Tracking   │  │ Per Device:      │
│ • Parse        │  │ • Rollout Policy     │  │ - Fridge 001     │
│ • Cache        │  │ • Scheduling         │  │ - Fan 002        │
│ • Priority     │  │ • Failure Tracking   │  │ - Light 003      │
└────────────────┘  └──────────────────────┘  └──────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │  Home Assistant UI      │
                    │  Settings → Updates     │
                    └─────────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │  User Actions           │
                    │  • View Changelog       │
                    │  • Install Update       │
                    │  • Skip Version         │
                    │  • Rollback             │
                    └────────────┬────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │  OTA Update Delivery    │
                    │  MQTT or HTTP           │
                    └────────────┬────────────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
┌───────┴────────┐  ┌───────────┴──────────┐  ┌─────────┴────────┐
│ Arduino Devices│  │  ESPHome Devices     │  │  Offline Devices │
│                │  │                      │  │                  │
│ • MQTT OTA     │  │ • Native OTA         │  │ • Manual Flash   │
│ • Custom       │  │ • ESPHome API        │  │ • USB Upload     │
│   firmware     │  │ • Built-in           │  │ • Recovery       │
└────────────────┘  └──────────────────────┘  └──────────────────┘
```

## Data Flow

### 1. Source Monitoring (Every 1 hour)

```
GitHub API
    │
    └─→ GET /repos/plantalytix/firmware/releases
            │
            └─→ Response: [
                  {
                    "tag_name": "v2.1.0",
                    "name": "Fridge Controller v2.1.0",
                    "body": "## Changelog\n- Fixed PID bug\n...",
                    "assets": [
                      {
                        "name": "fridge-v2.1.0.bin",
                        "browser_download_url": "https://..."
                      }
                    ]
                  }
                ]
                    │
                    └─→ Source Manager
                            │
                            └─→ Parse & Match Device Types
                                    │
                                    └─→ Update Coordinator
                                            │
                                            └─→ Update Entities
```

### 2. Version Comparison

```
Update Entity (Fridge 001)
    │
    ├─→ Current Version: v2.0.0
    ├─→ Latest Version:  v2.1.0
    │
    └─→ Comparison: 2.0.0 < 2.1.0 → Update Available
            │
            └─→ Set State: "on"
                    │
                    └─→ Attributes:
                        - release_url: "https://github.com/..."
                        - changelog: "## Changelog..."
                        - download_url: "https://github.com/.../fridge-v2.1.0.bin"
```

### 3. User Update Flow

```
User sees notification: "Firmware update available for Fridge 001"
    │
    └─→ Opens Settings → Updates
            │
            └─→ Clicks "Fridge 001: v2.0.0 → v2.1.0"
                    │
                    ├─→ Reads changelog
                    │
                    └─→ Clicks "Install"
                            │
                            └─→ Service Call: update.install
                                    │
                                    └─→ Update Coordinator
                                            │
                                            ├─→ Check rollout policy
                                            ├─→ Download firmware
                                            ├─→ Backup current version
                                            └─→ Trigger OTA
                                                    │
                                                    └─→ Publish MQTT:
                                                        Topic: plantalytix/fridge-001/firmware
                                                        Payload: {
                                                          "version": "v2.1.0",
                                                          "url": "http://ha/firmware/...",
                                                          "checksum": "abc123..."
                                                        }
```

### 4. Device Update Flow (Arduino)

```
Device receives MQTT message
    │
    └─→ Parse firmware URL and version
            │
            └─→ Download firmware from URL
                    │
                    ├─→ Verify checksum
                    │
                    ├─→ Write to flash
                    │
                    ├─→ Reboot
                    │
                    └─→ Post-reboot:
                            │
                            └─→ Publish new version
                                    │
                                    └─→ MQTT: plantalytix/fridge-001/state
                                        Payload: {"firmware_version": "v2.1.0"}
                                                │
                                                └─→ Update Coordinator detects
                                                        │
                                                        └─→ Update Entity:
                                                            Current: v2.1.0
                                                            State: "off" (no update)
```

### 5. Auto-Update Flow

```
Automation Trigger:
    - New update available (state: "on")
    - Time: 02:00 AM
    - Auto-update enabled
    │
    └─→ Check Rollout Policy
            │
            ├─→ Immediate: Update all devices
            │
            ├─→ Staged: Update in batches
            │   │
            │   └─→ Stage 1 (10%, Day 1)
            │       Stage 2 (50%, Day 2)
            │       Stage 3 (100%, Day 3)
            │
            └─→ Canary: Update 1 device first
                │
                └─→ Wait 24 hours
                        │
                        ├─→ Success → Update rest
                        │
                        └─→ Failure → Pause & Alert
```

## Configuration Examples

### Source Configuration

```yaml
# configuration.yaml
plantalytix_firmware:
  sources:
    # Official releases
    - name: "Official Plantalytix"
      type: github
      repository: "plantalytix/firmware"
      enabled: true
      priority: 1
      auto_update: false

    # Beta testing
    - name: "Beta Channel"
      type: github
      repository: "plantalytix/firmware"
      branch: "beta"
      enabled: false
      priority: 2

    # Community firmware
    - name: "Community ESPHome"
      type: http
      url: "https://community.plantalytix.io/firmware/releases.json"
      enabled: true
      priority: 3
```

### Rollout Policies

```yaml
# Immediate rollout
update_policy:
  rollout:
    type: immediate

# Staged rollout
update_policy:
  rollout:
    type: staged
    stages:
      - percentage: 10   # 10% of devices
        duration: 24h    # Wait 24 hours
      - percentage: 50   # Next 40%
        duration: 48h    # Wait 48 hours
      - percentage: 100  # Remaining 50%

# Canary deployment
update_policy:
  rollout:
    type: canary
    canary_count: 1      # Update 1 device
    wait_time: 24h       # Wait 24 hours
    success_threshold: 1  # Must succeed
```

### Update Scheduling

```yaml
# Daily at 2 AM
update_policy:
  schedule: "02:00:00"
  max_concurrent: 5
  max_failures: 3
  rollback_on_failure: true
```

## Update Entity States

```yaml
# No update available
state: "off"
attributes:
  installed_version: "2.1.0"
  latest_version: "2.1.0"

# Update available
state: "on"
attributes:
  installed_version: "2.0.0"
  latest_version: "2.1.0"
  release_url: "https://github.com/plantalytix/firmware/releases/tag/v2.1.0"
  release_summary: |
    ## Changes in v2.1.0
    - Fixed PID controller bug
    - Improved WiFi stability
    - Added CO2 sensor support
  title: "Fridge Controller v2.1.0"
  source: "Official Plantalytix"
  download_url: "https://github.com/.../fridge-v2.1.0.bin"
  file_size: 524288
  checksum: "sha256:abc123..."

# Update in progress
state: "on"
attributes:
  installed_version: "2.0.0"
  latest_version: "2.1.0"
  in_progress: true
  progress: 45  # Percentage

# Update failed
state: "on"
attributes:
  installed_version: "2.0.0"
  latest_version: "2.1.0"
  error: "Download failed: Network timeout"
```

## Automation Examples

### Auto-Update at 2 AM

```yaml
automation:
  - alias: "Auto-update Plantalytix firmware"
    trigger:
      - platform: time
        at: "02:00:00"
    condition:
      - condition: state
        entity_id: input_boolean.auto_update_enabled
        state: "on"
    action:
      # Get all update entities that have updates available
      - service: update.install
        target:
          entity_id: >
            {{ states.update
               | selectattr('state', 'eq', 'on')
               | selectattr('entity_id', 'match', 'update.plantalytix_.*')
               | map(attribute='entity_id')
               | list }}
        data:
          backup: true
```

### Notification on Update Available

```yaml
automation:
  - alias: "Notify firmware update available"
    trigger:
      - platform: state
        entity_id:
          - update.plantalytix_fridge_001_firmware
          - update.plantalytix_fan_002_firmware
        to: "on"
    action:
      - service: notify.mobile_app
        data:
          title: "Firmware Update Available"
          message: >
            {{ state_attr(trigger.entity_id, 'title') }}

            {{ state_attr(trigger.entity_id, 'release_summary')[:200] }}...
          data:
            url: "{{ state_attr(trigger.entity_id, 'release_url') }}"
            actions:
              - action: "INSTALL_UPDATE"
                title: "Install Now"
              - action: "VIEW_CHANGELOG"
                title: "View Full Changelog"
```

### Staged Rollout

```yaml
automation:
  - alias: "Staged rollout - Stage 1 (10%)"
    trigger:
      - platform: state
        entity_id: update.plantalytix_fridge_001_firmware
        to: "on"
        for:
          hours: 1
    condition:
      - condition: template
        value_template: >
          {{ (now().timestamp() | int) % 10 == 0 }}
    action:
      - service: update.install
        target:
          entity_id: "{{ trigger.entity_id }}"

  - alias: "Staged rollout - Stage 2 (50%)"
    trigger:
      - platform: state
        entity_id: update.plantalytix_fridge_001_firmware
        to: "on"
        for:
          hours: 25  # 1 day after stage 1
    condition:
      - condition: template
        value_template: >
          {{ (now().timestamp() | int) % 2 == 0 }}
    action:
      - service: update.install
        target:
          entity_id: "{{ trigger.entity_id }}"
```

## Services

### Add Firmware Source

```yaml
service: plantalytix_firmware.add_source
data:
  name: "My Custom Firmware"
  type: github
  config:
    repository: "myuser/plantalytix-custom"
    auth_token: "ghp_xxx"
    branch: "main"
```

### Remove Firmware Source

```yaml
service: plantalytix_firmware.remove_source
data:
  source_id: "my_custom_firmware"
```

### Rollback Firmware

```yaml
service: plantalytix_firmware.rollback
data:
  device_id: update.plantalytix_fridge_001_firmware
  # version: "2.0.0"  # Optional, defaults to previous version
```

### Set Auto-Update

```yaml
service: plantalytix_firmware.set_auto_update
data:
  device_id: update.plantalytix_fridge_001_firmware
  enabled: true
  schedule: "02:00:00"
```

## Dashboard Card

```yaml
# Firmware updates dashboard card
type: custom:auto-entities
card:
  type: entities
  title: Firmware Updates
  show_header_toggle: false
filter:
  include:
    - entity_id: "update.plantalytix_*"
      state: "on"
      options:
        secondary_info: last-changed
        tap_action:
          action: more-info
sort:
  method: state
  reverse: true
```

## Benefits

✅ **Unified Management**: Single interface for all firmware sources
✅ **Automatic Discovery**: New updates automatically detected
✅ **Changelog Visibility**: See what's new before updating
✅ **Rollout Control**: Staged/canary deployments prevent mass failures
✅ **Auto-Update**: Set and forget with scheduled updates
✅ **Multi-Source**: Official, beta, and community firmware
✅ **Version History**: Track and rollback to previous versions
✅ **HA Native**: Uses standard update entities and UI
✅ **Automation Ready**: Full automation support
✅ **Mobile Notifications**: Push notifications with changelogs

## Implementation Priority

**Priority**: High (after Phase 2 of HA migration)
**Estimated Effort**: 5 weeks
**Dependencies**: ADR-0001, ADR-0002, ADR-0003

## Related Documentation

- [ADR-0009: Firmware Update Management System](./decisions/0009-firmware-update-management-system.md) - Full ADR
- [ADR-0008: MQTT Discovery Implementation](./decisions/0008-firmware-mqtt-discovery-implementation.md) - Firmware OTA implementation
- [ADR-0003: ESPHome vs Arduino](./decisions/0003-esphome-vs-arduino-firmware.md) - Firmware architecture

---

**Last Updated**: 2025-11-15
**Version**: 1.0
