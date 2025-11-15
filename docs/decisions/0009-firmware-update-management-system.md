# Firmware Update Management System with Multi-Source Support

## Status

Proposed

## Context

Plantalytix devices require firmware updates from various sources:
- Official Plantalytix firmware releases
- Community-contributed firmware variants
- Experimental/beta firmware branches
- Third-party ESPHome configurations
- Device-specific patches

**Current Firmware Update Approach (ADR-0002, ADR-0008)**:
- Arduino firmware: Custom OTA via MQTT
- ESPHome devices: ESPHome dashboard or HA integration
- Firmware stored in `/firmware/` directory
- Manual version tracking
- No centralized update management

**Requirements**:
1. **Multi-Source Support**: Subscribe to firmware from GitHub, GitLab, custom URLs
2. **Version Management**: Track available, installed, and pending versions
3. **Automatic Updates**: Optional auto-update with configurable policies
4. **Changelog Visibility**: Display release notes before updating
5. **Rollout Control**: Staged rollouts, canary deployments
6. **Rollback Support**: Ability to revert to previous versions
7. **Update Scheduling**: Update during maintenance windows
8. **Source Management**: Enable/disable sources, priority ordering

**Home Assistant Capabilities**:

### Update Entity (HA Core 2022.4+)
- Standard entity type for firmware/software updates
- Attributes: `installed_version`, `latest_version`, `release_url`, `release_summary`
- Services: `update.install`, `update.skip`
- Support for firmware device class
- Auto-discovery via integrations

### ESPHome Integration
- Built-in update entities for ESPHome devices
- OTA update via `update.install` service
- Automatic version detection
- Changelog from ESPHome dashboard

### GitHub Integration
- Monitor repository releases
- Sensor for latest release/tag
- Webhook support for push notifications

### Custom Update Entities Integration (HACS)
- Create update entities from external sources
- GitHub releases monitoring
- Custom version parsing
- Update action configuration

**Challenges**:
- Arduino firmware needs custom update entity (not ESPHome)
- Multiple firmware sources require custom integration
- Changelog extraction from different sources
- Version compatibility validation
- Device-specific firmware variants (fridge vs. fan vs. light)

## Decision

**We will implement a custom Home Assistant integration (`plantalytix_firmware`) that leverages HA's update entity system to manage firmware from multiple sources (GitHub releases, GitLab, custom URLs) with configurable auto-update policies and changelog display.**

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              Home Assistant Frontend                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Settings → Updates                                   │   │
│  │  - Plantalytix Fridge 001: v2.0.0 → v2.1.0 available│   │
│  │  - [View Changelog] [Update] [Skip]                  │   │
│  └──────────────────────────────────────────────────────┘   │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────┴────────────────────────────────────┐
│    Custom Integration: plantalytix_firmware                 │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Update Entity per Device                            │   │
│  │  - Tracks: installed_version, latest_version         │   │
│  │  - Attributes: changelog, release_url, source        │   │
│  │  - Services: install, skip, rollback                 │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Firmware Source Manager                             │   │
│  │  - GitHub releases monitor                           │   │
│  │  - GitLab releases monitor                           │   │
│  │  - Custom HTTP/S sources                             │   │
│  │  - Version comparison logic                          │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Update Coordinator                                  │   │
│  │  - Rollout policies (canary, staged, immediate)      │   │
│  │  - Update scheduling                                 │   │
│  │  - Concurrent update limits                          │   │
│  │  - Failure tracking                                  │   │
│  └──────────────────────────────────────────────────────┘   │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────┴────────────────────────────────────┐
│              Firmware Delivery                               │
│  ┌──────────────────┐  ┌──────────────────┐                │
│  │  GitHub Releases │  │  Custom HTTP/S   │                │
│  │  (Official)      │  │  (Community)     │                │
│  └──────────────────┘  └──────────────────┘                │
│                  ↓              ↓                            │
│            Device OTA Update (MQTT or HTTP)                 │
└──────────────────────────────────────────────────────────────┘
```

### Implementation Components

#### 1. Custom Integration Structure

```
custom_components/plantalytix_firmware/
├── __init__.py              # Integration setup
├── manifest.json            # Integration metadata
├── config_flow.py           # UI configuration
├── const.py                 # Constants
├── update.py                # Update entity platform
├── coordinator.py           # Data update coordinator
├── sources/
│   ├── __init__.py
│   ├── base.py             # Base source class
│   ├── github.py           # GitHub releases source
│   ├── gitlab.py           # GitLab releases source
│   └── http.py             # Generic HTTP source
├── services.yaml            # Service definitions
└── translations/
    └── en.json             # English translations
```

#### 2. Firmware Source Configuration

**Config Flow (UI)**:
```yaml
# User configures firmware sources via UI
# Settings → Devices & Services → Add Integration → Plantalytix Firmware

# Step 1: Source Type
source_type: github | gitlab | http

# Step 2: Source Details (GitHub example)
repository: "plantalytix/firmware"
branch: "main" | "beta" | "experimental"
filter_pattern: "fridge-*.bin"  # Filter releases by pattern
auth_token: "ghp_xxx"  # Optional, for private repos

# Step 3: Update Policy
auto_update: true | false
update_schedule: "02:00:00"  # Time of day
rollout_policy: immediate | staged | canary
max_concurrent: 3
```

**YAML Configuration (Advanced)**:
```yaml
# configuration.yaml
plantalytix_firmware:
  sources:
    - name: "Official Plantalytix"
      type: github
      repository: "plantalytix/firmware"
      enabled: true
      priority: 1
      auto_update: false
      filter:
        device_type: ["fridge", "fan", "light"]
        version_pattern: "v\\d+\\.\\d+\\.\\d+"

    - name: "Community Beta"
      type: github
      repository: "plantalytix-community/firmware-beta"
      enabled: false
      priority: 2
      auto_update: false

    - name: "Custom ESPHome"
      type: http
      url: "https://firmware.example.com/releases.json"
      enabled: true
      priority: 3
      auth:
        type: bearer
        token: !secret custom_firmware_token

  update_policy:
    schedule: "02:00:00"
    rollout:
      type: staged  # immediate, staged, canary
      stages:
        - percentage: 10
          duration: 24h
        - percentage: 50
          duration: 48h
        - percentage: 100
    max_concurrent: 5
    max_failures: 3
    rollback_on_failure: true
```

#### 3. Update Entity Implementation

```python
# custom_components/plantalytix_firmware/update.py
from homeassistant.components.update import UpdateEntity, UpdateEntityFeature
from homeassistant.helpers.update_coordinator import CoordinatorEntity

class PlantalytixFirmwareUpdate(CoordinatorEntity, UpdateEntity):
    """Firmware update entity for Plantalytix devices."""

    _attr_device_class = "firmware"
    _attr_supported_features = (
        UpdateEntityFeature.INSTALL
        | UpdateEntityFeature.RELEASE_NOTES
        | UpdateEntityFeature.SPECIFIC_VERSION
        | UpdateEntityFeature.BACKUP
    )

    def __init__(self, coordinator, device_id, device_type):
        """Initialize the update entity."""
        super().__init__(coordinator)
        self._device_id = device_id
        self._device_type = device_type
        self._attr_unique_id = f"plantalytix_firmware_{device_id}"
        self._attr_name = f"Plantalytix {device_type.title()} {device_id} Firmware"

    @property
    def installed_version(self) -> str | None:
        """Version currently installed."""
        return self.coordinator.data.get(self._device_id, {}).get("current_version")

    @property
    def latest_version(self) -> str | None:
        """Latest version available for install."""
        return self.coordinator.data.get(self._device_id, {}).get("latest_version")

    @property
    def release_url(self) -> str | None:
        """URL to release notes."""
        return self.coordinator.data.get(self._device_id, {}).get("release_url")

    @property
    def release_summary(self) -> str | None:
        """Summary of the release."""
        return self.coordinator.data.get(self._device_id, {}).get("changelog")

    @property
    def title(self) -> str | None:
        """Title of the release."""
        return self.coordinator.data.get(self._device_id, {}).get("release_title")

    @property
    def extra_state_attributes(self) -> dict:
        """Return additional state attributes."""
        device_data = self.coordinator.data.get(self._device_id, {})
        return {
            "source": device_data.get("source"),
            "download_url": device_data.get("download_url"),
            "file_size": device_data.get("file_size"),
            "checksum": device_data.get("checksum"),
            "compatibility": device_data.get("compatibility"),
            "previous_versions": device_data.get("previous_versions", []),
        }

    async def async_install(
        self, version: str | None, backup: bool, **kwargs
    ) -> None:
        """Install an update."""
        await self.coordinator.async_install_update(
            self._device_id,
            version or self.latest_version,
            backup
        )

    async def async_release_notes(self) -> str | None:
        """Return the release notes."""
        return await self.coordinator.async_get_release_notes(
            self._device_id,
            self.latest_version
        )
```

#### 4. GitHub Source Implementation

```python
# custom_components/plantalytix_firmware/sources/github.py
from typing import Dict, List
from datetime import datetime
import aiohttp
import re

class GitHubFirmwareSource:
    """Fetch firmware from GitHub releases."""

    def __init__(self, repository: str, auth_token: str = None):
        self.repository = repository
        self.auth_token = auth_token
        self.api_url = f"https://api.github.com/repos/{repository}/releases"

    async def async_get_releases(self) -> List[Dict]:
        """Fetch all releases from GitHub."""
        headers = {"Accept": "application/vnd.github+json"}
        if self.auth_token:
            headers["Authorization"] = f"Bearer {self.auth_token}"

        async with aiohttp.ClientSession() as session:
            async with session.get(self.api_url, headers=headers) as response:
                if response.status == 200:
                    return await response.json()
                else:
                    raise Exception(f"GitHub API error: {response.status}")

    async def async_get_latest_for_device(
        self, device_type: str, current_version: str = None
    ) -> Dict | None:
        """Get latest firmware for specific device type."""
        releases = await self.async_get_releases()

        for release in releases:
            # Skip pre-releases and drafts
            if release["prerelease"] or release["draft"]:
                continue

            # Find asset matching device type
            for asset in release["assets"]:
                if self._matches_device_type(asset["name"], device_type):
                    version = self._extract_version(release["tag_name"])

                    # Skip if same or older version
                    if current_version and not self._is_newer_version(
                        version, current_version
                    ):
                        continue

                    return {
                        "version": version,
                        "release_title": release["name"],
                        "changelog": release["body"],
                        "release_url": release["html_url"],
                        "download_url": asset["browser_download_url"],
                        "file_size": asset["size"],
                        "published_at": release["published_at"],
                        "author": release["author"]["login"],
                    }

        return None

    def _matches_device_type(self, filename: str, device_type: str) -> bool:
        """Check if filename matches device type."""
        # Example: fridge-v2.1.0.bin matches "fridge"
        pattern = rf"{device_type}[-_].*\.bin$"
        return bool(re.search(pattern, filename, re.IGNORECASE))

    def _extract_version(self, tag_name: str) -> str:
        """Extract version from tag name."""
        # Remove 'v' prefix if present: v2.1.0 → 2.1.0
        return tag_name.lstrip("v")

    def _is_newer_version(self, version: str, current: str) -> bool:
        """Compare versions (semantic versioning)."""
        def parse_version(v):
            return tuple(map(int, v.split(".")))

        try:
            return parse_version(version) > parse_version(current)
        except ValueError:
            # Fallback to string comparison
            return version > current
```

#### 5. Update Coordinator

```python
# custom_components/plantalytix_firmware/coordinator.py
from homeassistant.helpers.update_coordinator import DataUpdateCoordinator
from datetime import timedelta
import asyncio

class PlantalytixFirmwareCoordinator(DataUpdateCoordinator):
    """Coordinator to manage firmware updates."""

    def __init__(self, hass, sources, update_policy):
        """Initialize coordinator."""
        super().__init__(
            hass,
            _LOGGER,
            name="Plantalytix Firmware",
            update_interval=timedelta(hours=1),
        )
        self.sources = sources
        self.update_policy = update_policy
        self.updating_devices = {}

    async def _async_update_data(self):
        """Fetch latest firmware versions from all sources."""
        data = {}

        # Get all Plantalytix devices
        devices = await self._get_plantalytix_devices()

        for device in devices:
            device_id = device["device_id"]
            device_type = device["device_type"]
            current_version = device["firmware_version"]

            # Check each enabled source (by priority)
            latest_firmware = None
            for source in sorted(self.sources, key=lambda x: x.priority):
                if not source.enabled:
                    continue

                try:
                    firmware = await source.async_get_latest_for_device(
                        device_type, current_version
                    )
                    if firmware:
                        latest_firmware = firmware
                        latest_firmware["source"] = source.name
                        break  # Use highest priority source
                except Exception as e:
                    _LOGGER.error(f"Error fetching from {source.name}: {e}")

            if latest_firmware:
                data[device_id] = {
                    "current_version": current_version,
                    "latest_version": latest_firmware["version"],
                    "changelog": latest_firmware["changelog"],
                    "release_url": latest_firmware["release_url"],
                    "release_title": latest_firmware["release_title"],
                    "download_url": latest_firmware["download_url"],
                    "file_size": latest_firmware["file_size"],
                    "source": latest_firmware["source"],
                    "compatibility": self._check_compatibility(
                        device, latest_firmware
                    ),
                }

        return data

    async def async_install_update(
        self, device_id: str, version: str, backup: bool = True
    ):
        """Install firmware update to device."""
        device_data = self.data.get(device_id)
        if not device_data:
            raise ValueError(f"No update available for {device_id}")

        # Check rollout policy
        if not await self._can_start_update(device_id):
            raise Exception("Update blocked by rollout policy")

        # Download firmware
        firmware_path = await self._download_firmware(
            device_data["download_url"],
            device_id,
            version
        )

        # Backup current version (if requested)
        if backup:
            await self._backup_current_firmware(device_id)

        # Trigger OTA update via MQTT
        await self._trigger_ota_update(device_id, firmware_path, version)

        # Track update
        self.updating_devices[device_id] = {
            "version": version,
            "started_at": datetime.now(),
            "status": "in_progress"
        }

    async def _trigger_ota_update(
        self, device_id: str, firmware_path: str, version: str
    ):
        """Send OTA update command to device via MQTT."""
        # For Arduino devices
        firmware_url = f"{self.hass.config.api.base_url}/local/firmware/{device_id}/{version}.bin"

        await self.hass.services.async_call(
            "mqtt",
            "publish",
            {
                "topic": f"plantalytix/{device_id}/firmware",
                "payload": {
                    "version": version,
                    "url": firmware_url,
                    "checksum": await self._calculate_checksum(firmware_path),
                },
            },
        )

        # For ESPHome devices
        # Use built-in update.install service
        esphome_entity = f"update.plantalytix_{device_id}_firmware"
        if self.hass.states.get(esphome_entity):
            await self.hass.services.async_call(
                "update",
                "install",
                {"entity_id": esphome_entity},
            )
```

### 6. Configuration Flow (UI)

```python
# custom_components/plantalytix_firmware/config_flow.py
from homeassistant import config_entries
import voluptuous as vol

class PlantalytixFirmwareConfigFlow(config_entries.ConfigFlow, domain=DOMAIN):
    """Handle config flow for Plantalytix Firmware."""

    VERSION = 1

    async def async_step_user(self, user_input=None):
        """Handle initial step."""
        if user_input is not None:
            return await self.async_step_source_type()

        return self.async_show_form(
            step_id="user",
            data_schema=vol.Schema({
                vol.Required("setup_type"): vol.In({
                    "quick": "Quick Setup (Official sources only)",
                    "advanced": "Advanced Setup (Custom sources)",
                }),
            }),
        )

    async def async_step_source_type(self, user_input=None):
        """Select source type."""
        if user_input is not None:
            self.source_type = user_input["source_type"]
            if self.source_type == "github":
                return await self.async_step_github()
            elif self.source_type == "gitlab":
                return await self.async_step_gitlab()
            else:
                return await self.async_step_http()

        return self.async_show_form(
            step_id="source_type",
            data_schema=vol.Schema({
                vol.Required("source_type"): vol.In({
                    "github": "GitHub Releases",
                    "gitlab": "GitLab Releases",
                    "http": "Custom HTTP/HTTPS",
                }),
            }),
        )

    async def async_step_github(self, user_input=None):
        """Configure GitHub source."""
        errors = {}

        if user_input is not None:
            # Validate repository
            try:
                await self._validate_github_repo(
                    user_input["repository"],
                    user_input.get("auth_token")
                )
                return self.async_create_entry(
                    title=f"GitHub: {user_input['repository']}",
                    data=user_input,
                )
            except Exception:
                errors["base"] = "invalid_repo"

        return self.async_show_form(
            step_id="github",
            data_schema=vol.Schema({
                vol.Required("name"): str,
                vol.Required("repository"): str,  # e.g., "plantalytix/firmware"
                vol.Optional("auth_token"): str,
                vol.Optional("branch", default="main"): str,
                vol.Optional("filter_pattern", default="*.bin"): str,
                vol.Required("enabled", default=True): bool,
                vol.Required("auto_update", default=False): bool,
            }),
            errors=errors,
        )
```

### 7. Services Definition

```yaml
# custom_components/plantalytix_firmware/services.yaml
add_source:
  name: Add Firmware Source
  description: Add a new firmware source
  fields:
    name:
      name: Source Name
      description: Friendly name for the source
      required: true
      example: "Community Firmware"
      selector:
        text:
    type:
      name: Source Type
      description: Type of firmware source
      required: true
      selector:
        select:
          options:
            - github
            - gitlab
            - http
    config:
      name: Configuration
      description: Source-specific configuration
      required: true
      example: '{"repository": "plantalytix/firmware", "auth_token": "ghp_xxx"}'
      selector:
        object:

remove_source:
  name: Remove Firmware Source
  description: Remove a firmware source
  fields:
    source_id:
      name: Source ID
      description: ID of the source to remove
      required: true
      selector:
        text:

refresh_sources:
  name: Refresh All Sources
  description: Force refresh of all firmware sources

rollback:
  name: Rollback Firmware
  description: Rollback device to previous firmware version
  fields:
    device_id:
      name: Device ID
      description: Device to rollback
      required: true
      selector:
        entity:
          domain: update
    version:
      name: Version
      description: Version to rollback to (optional, defaults to previous)
      required: false
      selector:
        text:

set_auto_update:
  name: Set Auto Update
  description: Enable/disable automatic updates for a device
  fields:
    device_id:
      name: Device ID
      required: true
      selector:
        entity:
          domain: update
    enabled:
      name: Enabled
      description: Enable automatic updates
      required: true
      selector:
        boolean:
    schedule:
      name: Schedule
      description: Time to perform updates (HH:MM:SS)
      required: false
      default: "02:00:00"
      selector:
        time:
```

### 8. Frontend Dashboard Card

```yaml
# Lovelace card for firmware management
type: custom:auto-entities
card:
  type: entities
  title: Firmware Updates
  show_header_toggle: false
filter:
  include:
    - domain: update
      attributes:
        device_class: firmware
      options:
        secondary_info: last-changed
        tap_action:
          action: more-info
  exclude:
    - state: "off"
sort:
  method: attribute
  attribute: latest_version
  reverse: true

# Or use built-in update card (HA 2022.4+)
type: update-entities
title: Plantalytix Firmware Updates
show_release_notes: true
```

### 9. Automation Examples

```yaml
# Automatic update during maintenance window
automation:
  - alias: "Auto-update Plantalytix firmware"
    trigger:
      - platform: state
        entity_id: update.plantalytix_fridge_001_firmware
        to: "on"
    condition:
      - condition: time
        after: "02:00:00"
        before: "04:00:00"
      - condition: state
        entity_id: input_boolean.auto_update_enabled
        state: "on"
    action:
      - service: update.install
        target:
          entity_id: "{{ trigger.entity_id }}"
        data:
          backup: true

  - alias: "Notify on firmware update available"
    trigger:
      - platform: state
        entity_id: update.plantalytix_fridge_001_firmware
        to: "on"
    action:
      - service: notify.mobile_app
        data:
          title: "Firmware Update Available"
          message: >
            {{ state_attr(trigger.entity_id, 'title') }}
            Version: {{ state_attr(trigger.entity_id, 'latest_version') }}
          data:
            actions:
              - action: "VIEW_CHANGELOG"
                title: "View Changelog"
                uri: "{{ state_attr(trigger.entity_id, 'release_url') }}"
              - action: "INSTALL_UPDATE"
                title: "Install Now"

  - alias: "Staged rollout - 10% after 24h"
    trigger:
      - platform: state
        entity_id: update.plantalytix_fridge_001_firmware
        to: "on"
        for:
          hours: 24
    condition:
      - condition: template
        value_template: >
          {{ (now().timestamp() | int) % 10 == 0 }}  # 10% of devices
    action:
      - service: update.install
        target:
          entity_id: "{{ trigger.entity_id }}"
```

## Consequences

### Positive

✅ **Unified Update Management**: Single interface for all firmware sources

✅ **Leverages HA Features**: Built on standard update entity, native UI support

✅ **Multi-Source Flexibility**: GitHub, GitLab, custom HTTP sources

✅ **Automatic Discovery**: Update entities auto-discovered in HA

✅ **Changelog Visibility**: Release notes displayed before updating

✅ **Rollout Control**: Staged, canary, or immediate deployment

✅ **Auto-Update Support**: Configurable automatic updates with scheduling

✅ **Version History**: Track previous versions for rollback

✅ **Source Prioritization**: Use highest-priority source first

✅ **HA Native UI**: Uses built-in update dashboard

✅ **Automation Ready**: Trigger automations on update availability

✅ **Mobile Notifications**: Push notifications with changelog links

### Negative

❌ **Custom Integration Required**: Not available out-of-box in HA

❌ **Development Effort**: ~2-3 weeks to implement fully

❌ **Maintenance**: Must maintain integration as HA evolves

❌ **Testing Complexity**: Must test with multiple source types

❌ **GitHub API Rate Limits**: May hit limits with many sources (unauthenticated: 60/hour)

❌ **Storage Requirements**: Downloaded firmware stored locally

❌ **Network Dependency**: Requires internet for GitHub/GitLab sources

### Risks

⚠️ **Incompatible Firmware**: User might install wrong firmware variant

⚠️ **Bricked Devices**: Failed OTA update could brick device

⚠️ **Source Availability**: External sources might go offline

⚠️ **Version Parsing**: Different version schemes might break comparison

⚠️ **Security**: Malicious firmware from untrusted sources

### Mitigation Strategies

1. **Compatibility Checking**: Validate firmware compatible with device type
2. **Rollback Support**: Always backup previous version before updating
3. **Source Verification**: Signature verification for firmware binaries
4. **Rate Limiting**: Cache GitHub API responses, use authentication
5. **Graceful Degradation**: Fallback to local cache if source unavailable
6. **User Warnings**: Clear warnings when enabling non-official sources

## Alternatives Considered

### Alternative 1: ESPHome Dashboard Only

**Description**: Use only ESPHome dashboard for firmware updates

**Pros**:
- Built-in to ESPHome
- No custom development
- Well-tested

**Cons**:
- Doesn't work for Arduino devices
- No multi-source support
- Limited automation capabilities

**Verdict**: Rejected - doesn't meet multi-source and Arduino device requirements

### Alternative 2: Node-RED Flow

**Description**: Implement firmware management in Node-RED

**Pros**:
- Visual programming
- Flexible

**Cons**:
- Not integrated with HA UI
- No native update entities
- Requires Node-RED installation

**Verdict**: Rejected - poor integration with HA ecosystem

### Alternative 3: AppDaemon App

**Description**: Python AppDaemon app for firmware management

**Pros**:
- Python development
- HA integration

**Cons**:
- Doesn't create native update entities
- Requires AppDaemon
- Less discoverable than integration

**Verdict**: Rejected - lacks native HA update entity support

### Alternative 4: HACS Custom Repository Only

**Description**: Use HACS to manage firmware as if it were HA integrations

**Pros**:
- HACS already installed
- GitHub integration

**Cons**:
- Firmware isn't HA integration
- Confusing UX
- No device-specific tracking

**Verdict**: Rejected - misuse of HACS functionality

## Implementation Plan

### Phase 1: Core Integration (2 weeks)

**Week 1: Foundation**
1. Create integration skeleton
2. Implement update entity platform
3. Implement GitHub source
4. Basic config flow

**Week 2: Features**
5. Implement coordinator
6. Add OTA trigger logic
7. Version comparison
8. Changelog extraction

### Phase 2: Additional Sources (1 week)

1. Implement GitLab source
2. Implement HTTP source
3. Source priority system
4. Enable/disable sources

### Phase 3: Advanced Features (1 week)

1. Rollback support
2. Backup system
3. Rollout policies (staged, canary)
4. Update scheduling
5. Concurrent update limiting

### Phase 4: Testing & Documentation (1 week)

1. Integration tests
2. End-to-end testing with real devices
3. Documentation
4. Example automations
5. Troubleshooting guide

**Total**: 5 weeks development + testing

### Phase 5: Deployment

1. Publish as HACS custom integration
2. Add to Plantalytix HA setup guide
3. User training materials
4. Monitor for issues

## Configuration Examples

See sections 2-9 above for complete configuration examples.

## References

- [HA Update Entity](https://www.home-assistant.io/integrations/update/)
- [HA Update Entity Developer Docs](https://developers.home-assistant.io/docs/core/entity/update/)
- [GitHub API Releases](https://docs.github.com/en/rest/releases/releases)
- [ESPHome OTA Updates](https://esphome.io/components/ota/)
- [Custom Update Entities (HACS)](https://github.com/gadgetchnnel/custom_update_entities)

## Notes

**GitHub Rate Limiting**:
- Unauthenticated: 60 requests/hour
- Authenticated: 5,000 requests/hour
- Solution: Use authentication tokens, cache responses

**Firmware Signing** (Future Enhancement):
```python
# Verify firmware signature before installation
async def _verify_signature(firmware_path: str, signature: str) -> bool:
    """Verify firmware binary signature using GPG."""
    # Implementation using cryptography library
    pass
```

**Compatibility Matrix** (Future Enhancement):
```yaml
# firmware-compatibility.yaml
fridge-v2.1.0:
  compatible_hardware:
    - "ESP32-WROOM"
    - "ESP32-WROVER"
  min_previous_version: "v2.0.0"
  breaking_changes: true

fan-v1.5.0:
  compatible_hardware:
    - "ESP32-WROOM"
  min_previous_version: "v1.0.0"
  breaking_changes: false
```

---

**Decision Date**: 2025-11-15
**Implementation Priority**: High (after Phase 2 of HA migration)
**Estimated Effort**: 5 weeks development
**Dependencies**: ADR-0001 (HA Migration), ADR-0002 (MQTT Discovery), ADR-0003 (ESPHome)
