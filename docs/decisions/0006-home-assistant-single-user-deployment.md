# Home Assistant Single-User Local Deployment for Plantalytix

## Status

Proposed

## Context

With the decision to migrate to Home Assistant (ADR-0001), we must define the deployment model for Plantalytix.

**Current Plantalytix Architecture:**
- **Architecture**: Custom Node.js backend with JWT authentication
- **User Management**: Custom user system with MongoDB
- **Multi-tenancy**: Support for multiple users with isolated devices
- **Deployment**: Cloud-based SaaS model

**New Requirements:**
Plantalytix will be deployed as a **single-user local network system** for individual growing facilities:

1. **Single User**: One growing facility or home user per installation
2. **Local Network**: No cloud dependency, operates entirely on local network
3. **Simplified Authentication**: Home Assistant handles all user authentication
4. **Direct Device Access**: All devices visible and accessible to the user
5. **No Multi-Tenancy**: Each installation is independent

**Home Assistant Architecture:**

Home Assistant is designed as a **single-user** smart home platform:
- One HA instance per home/facility
- Built-in user management for household members
- All users share access to all devices
- Local-first architecture (works offline)
- Web UI and mobile app included

This aligns perfectly with the new Plantalytix requirements.

## Decision

**We will deploy Plantalytix as a single Home Assistant instance per growing facility, running on the local network with no cloud dependencies.**

### Rationale

**Perfect Alignment with HA Design**:
1. **Single-User Model**: HA is designed for single household/facility use
2. **No Multi-Tenancy Needed**: Each installation is independent
3. **Local Control**: All processing happens locally, no internet required
4. **Built-in Authentication**: HA provides user management out-of-the-box
5. **Simplified Architecture**: No complex tenant isolation needed
6. **Lower Resource Requirements**: Single HA instance uses minimal resources
7. **Easier Maintenance**: Simple backup and update procedures

**Benefits Over Cloud/Multi-Tenant**:
- No cloud infrastructure costs
- Complete data privacy (all data stays local)
- Works without internet connection
- Instant response times (no cloud latency)
- No subscription fees
- Full control over updates and configuration

### Architecture

**Single-Instance Deployment Model**

```
Local Network (192.168.1.0/24)
├── Home Assistant Server (Raspberry Pi / Mini PC / Server)
│   ├── Home Assistant Core (Port 8123)
│   ├── PostgreSQL (14-day recorder)
│   ├── InfluxDB v2 (long-term time-series)
│   └── Mosquitto MQTT Broker (Port 1883)
│
├── Plantalytix Devices
│   ├── Fridge Controller 001 (192.168.1.101)
│   ├── Fan Controller 001 (192.168.1.102)
│   ├── Light Controller 001 (192.168.1.103)
│   └── Smart Socket 001 (192.168.1.104)
│
└── User Devices
    ├── Desktop/Laptop (Web browser → http://homeassistant.local:8123)
    └── Mobile Phone (HA Companion App)
```

**Directory Structure**:
```
/home/user/plantalytix/  (or /opt/plantalytix/)
├── docker-compose.yml
├── .env
├── homeassistant/
│   ├── configuration.yaml
│   ├── automations.yaml
│   ├── scripts.yaml
│   ├── dashboards/
│   │   ├── main.yaml
│   │   └── device-details.yaml
│   └── custom_components/
│       └── plantalytix_firmware/  (custom integration)
├── postgres/
│   └── data/
├── influxdb/
│   └── data/
├── mosquitto/
│   ├── config/
│   │   └── mosquitto.conf
│   └── data/
└── backups/
```

### Authentication & User Management

**Home Assistant Handles Everything**:
- Users created in HA Settings → People
- No custom authentication needed
- Built-in support for:
  - Local user accounts
  - OAuth (Google, GitHub, etc.)
  - Two-factor authentication
  - Trusted networks (auto-login on local network)
  - API tokens for external access

**User Types**:
1. **Owner/Admin**: Full access to all settings and devices
2. **Family Members**: Optional additional users with shared access
3. **API Access**: Tokens for external integrations (if needed)

**No Custom User Management Needed**:
- Remove MongoDB user collection
- Remove JWT token system
- Remove custom login/signup API
- Remove password reset functionality
- HA provides all of this out-of-the-box

### Network & Access

**Local Network Access**:
```
Primary Access:
- http://homeassistant.local:8123 (mDNS)
- http://192.168.1.50:8123 (direct IP)

Mobile Access (Same Network):
- HA Companion App → Auto-discovers HA instance
- Uses local network for communication
```

**Optional Remote Access** (if user wants):
```
Option 1: Nabu Casa Cloud ($6.50/month)
- Secure remote access
- Encrypted tunnel
- No port forwarding needed

Option 2: Tailscale/WireGuard VPN (Free)
- Self-hosted VPN
- Secure remote access
- No cloud service needed

Option 3: Reverse Proxy + DuckDNS (Free)
- Port forwarding required
- Let's Encrypt SSL
- Manual setup
```

### Device Integration

**Simplified Device Flow**:

1. **Device Powers On**:
   - Connects to local WiFi
   - Discovers MQTT broker via mDNS or static IP
   - Publishes MQTT discovery messages

2. **MQTT Discovery**:
   ```
   Topic: homeassistant/climate/fridge001/config
   Payload: {device info, topics, capabilities}
   ```

3. **HA Auto-Creates Entity**:
   - Device appears in HA immediately
   - No claim codes needed
   - No user assignment needed
   - All devices belong to the facility

4. **User Access**:
   - Opens HA dashboard
   - Sees all devices automatically
   - Can control and monitor immediately

**No Device Claiming Process**:
- Remove claim code system
- Remove device ownership concept
- Remove device-to-user association
- All devices are facility devices

### Data Storage

**All Data Stays Local**:

1. **PostgreSQL (Recorder)**:
   - 14 days of entity history
   - Automations, users, device registry
   - ~500MB-2GB depending on device count

2. **InfluxDB v2**:
   - Long-term time-series (1 year+)
   - Temperature, humidity, CO2 trends
   - Configurable retention policy

3. **HA Configuration**:
   - YAML files for automations, dashboards
   - Version controlled (optional git repo)

**No Cloud Storage**:
- No MongoDB in the cloud
- No cloud API required
- All data on local disks
- Full control over backups

## Consequences

### Positive

✅ **Simplified Architecture**: No multi-tenancy complexity

✅ **Lower Resource Usage**: Single HA instance, minimal hardware

✅ **Complete Privacy**: All data stays on local network

✅ **No Internet Dependency**: Works completely offline

✅ **Instant Response**: No cloud latency

✅ **No Subscription Costs**: Free (except optional Nabu Casa)

✅ **HA Best Practices**: Aligns perfectly with HA design philosophy

✅ **Easy Backup**: Simple backup of HA config and databases

✅ **Easy Setup**: Standard HA installation process

✅ **Built-in User Management**: HA handles authentication

✅ **Mobile App Included**: HA Companion app works out-of-box

✅ **Lower Maintenance**: No tenant provisioning, no user management

### Negative

❌ **One Installation Per Facility**: Can't manage multiple facilities from one HA

❌ **No SaaS Model**: Can't offer cloud-hosted service

❌ **User Must Host**: Requires local hardware (Raspberry Pi, NUC, server)

❌ **Local Network Only**: Remote access requires additional setup

❌ **No Central Management**: Can't manage multiple installations centrally

❌ **Initial Setup**: User must set up HA instance

### Neutral

⚪ **Scalability**: Limited to one facility, but that's the goal

⚪ **Hardware Requirements**: Minimal (Raspberry Pi 4 sufficient for 20-30 devices)

⚪ **Updates**: User controls update schedule

## Deployment Options

### Option 1: Raspberry Pi (Recommended for Home Users)

**Hardware**:
- Raspberry Pi 4 (4GB RAM minimum, 8GB recommended)
- 32GB+ SD card or SSD
- Power supply
- Case with cooling

**Installation**:
```bash
# Flash Home Assistant OS to SD card
# Boot Raspberry Pi
# Access http://homeassistant.local:8123
# Run installation wizard
```

**Cost**: ~$100-150

### Option 2: Mini PC / Intel NUC (Recommended for Commercial)

**Hardware**:
- Intel NUC or equivalent (i3/i5, 8GB+ RAM)
- 256GB+ SSD
- Runs Home Assistant OS or Docker

**Installation**:
```bash
# Option A: Home Assistant OS (recommended)
# Flash to USB, install on NUC

# Option B: Docker on Linux
docker-compose up -d
```

**Cost**: ~$300-600

### Option 3: Existing Server / VM

**Hardware**:
- Existing Linux server
- 2+ CPU cores, 4GB+ RAM
- 50GB+ disk space

**Installation**:
```bash
# Docker Compose deployment
cd /opt/plantalytix
docker-compose up -d
```

**Cost**: $0 (using existing hardware)

### Option 4: Home Assistant Blue/Yellow (Pre-built)

**Hardware**:
- Official Home Assistant appliance
- Plug-and-play solution
- Odroid N2+ based

**Cost**: ~$150-200

## Configuration Examples

### Docker Compose (Single-Instance)

```yaml
version: '3.8'

services:
  homeassistant:
    container_name: plantalytix-ha
    image: ghcr.io/home-assistant/home-assistant:stable
    restart: unless-stopped
    privileged: true
    network_mode: host
    environment:
      - TZ=America/Los_Angeles
    volumes:
      - ./homeassistant:/config
      - /run/dbus:/run/dbus:ro

  postgres:
    container_name: plantalytix-postgres
    image: postgres:15
    restart: unless-stopped
    environment:
      POSTGRES_DB: homeassistant
      POSTGRES_USER: homeassistant
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - ./postgres:/var/lib/postgresql/data
    ports:
      - "127.0.0.1:5432:5432"

  mosquitto:
    container_name: plantalytix-mqtt
    image: eclipse-mosquitto:latest
    restart: unless-stopped
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - ./mosquitto/data:/mosquitto/data

  influxdb:
    container_name: plantalytix-influxdb
    image: influxdb:2
    restart: unless-stopped
    environment:
      DOCKER_INFLUXDB_INIT_MODE: setup
      DOCKER_INFLUXDB_INIT_USERNAME: admin
      DOCKER_INFLUXDB_INIT_PASSWORD: ${INFLUXDB_PASSWORD}
      DOCKER_INFLUXDB_INIT_ORG: plantalytix
      DOCKER_INFLUXDB_INIT_BUCKET: sensors
      DOCKER_INFLUXDB_INIT_ADMIN_TOKEN: ${INFLUXDB_TOKEN}
    ports:
      - "8086:8086"
    volumes:
      - ./influxdb:/var/lib/influxdb2
```

### Home Assistant Configuration

```yaml
# homeassistant/configuration.yaml

# Basic configuration
homeassistant:
  name: Plantalytix Grow Facility
  latitude: !secret latitude
  longitude: !secret longitude
  elevation: !secret elevation
  unit_system: metric
  time_zone: America/Los_Angeles

# Enable default integrations
default_config:

# PostgreSQL Recorder
recorder:
  db_url: postgresql://homeassistant:password@postgres:5432/homeassistant
  purge_keep_days: 14
  commit_interval: 1

# MQTT (auto-discovery enabled)
mqtt:
  broker: mosquitto
  port: 1883
  discovery: true
  discovery_prefix: homeassistant

# InfluxDB (long-term storage)
influxdb:
  api_version: 2
  host: influxdb
  port: 8086
  token: !secret influxdb_token
  organization: plantalytix
  bucket: sensors
  tags:
    source: plantalytix
  include:
    entity_globs:
      - sensor.plantalytix_*
      - climate.plantalytix_*

# Frontend
frontend:
  themes: !include_dir_merge_named themes

# Lovelace dashboards
lovelace:
  mode: yaml
  resources:
    - url: /hacsfiles/apexcharts-card/apexcharts-card.js
      type: module
  dashboards:
    plantalytix-main:
      mode: yaml
      title: Grow Facility
      icon: mdi:leaf
      show_in_sidebar: true
      filename: dashboards/main.yaml

# Mobile app
mobile_app:

# History
history:

# Logbook
logbook:

# System health
system_health:
```

### Mosquitto Configuration

```conf
# mosquitto/config/mosquitto.conf

# Listener
listener 1883
protocol mqtt

# WebSocket (for browser-based MQTT)
listener 9001
protocol websockets

# Authentication (optional for local network)
allow_anonymous true

# For production, enable authentication:
# allow_anonymous false
# password_file /mosquitto/config/passwd

# Persistence
persistence true
persistence_location /mosquitto/data/

# Logging
log_dest stdout
log_type error
log_type warning
log_type notice
log_type information

# Performance
max_queued_messages 1000
max_inflight_messages 100
```

## Hardware Requirements

### Minimum (1-10 devices)
- **CPU**: 2 cores
- **RAM**: 2GB
- **Disk**: 32GB
- **Network**: 100 Mbps Ethernet or WiFi
- **Example**: Raspberry Pi 4 (2GB)

### Recommended (10-30 devices)
- **CPU**: 4 cores
- **RAM**: 4GB
- **Disk**: 128GB SSD
- **Network**: 1 Gbps Ethernet
- **Example**: Raspberry Pi 4 (4GB) or Intel NUC

### Advanced (30+ devices)
- **CPU**: 4+ cores
- **RAM**: 8GB+
- **Disk**: 256GB+ SSD
- **Network**: 1 Gbps Ethernet
- **Example**: Intel NUC i5 or dedicated server

## Installation Steps

### 1. Install Home Assistant

```bash
# Option A: Home Assistant OS (Recommended)
# - Download HA OS image for your hardware
# - Flash to SD card/USB
# - Boot device
# - Access http://homeassistant.local:8123

# Option B: Docker (Linux server)
cd /opt/plantalytix
# Create docker-compose.yml (see above)
docker-compose up -d
```

### 2. Initial Setup

1. Open http://homeassistant.local:8123
2. Create admin account
3. Set location and timezone
4. Configure integrations (MQTT, InfluxDB)

### 3. Configure MQTT

```bash
# In HA UI:
Settings → Devices & Services → Add Integration → MQTT
- Broker: mosquitto (or localhost)
- Port: 1883
- Enable Discovery: Yes
```

### 4. Configure InfluxDB

```bash
# In HA configuration.yaml:
influxdb:
  api_version: 2
  host: influxdb
  token: <your_token>
  organization: plantalytix
  bucket: sensors
```

### 5. Deploy Devices

- Flash device firmware (ESPHome or Arduino)
- Configure WiFi credentials
- Devices auto-discover via MQTT

### 6. Create Dashboards

- Use Lovelace UI editor
- Add cards for devices
- Configure ApexCharts for historical data

## Backup Strategy

### Automatic Backups

```yaml
# homeassistant/automations.yaml
- alias: "Daily Backup"
  trigger:
    - platform: time
      at: "03:00:00"
  action:
    - service: backup.create
    - service: shell_command.backup_databases

# homeassistant/configuration.yaml
shell_command:
  backup_databases: >
    bash /config/scripts/backup.sh
```

### Backup Script

```bash
#!/bin/bash
# homeassistant/scripts/backup.sh

BACKUP_DIR="/config/backups"
DATE=$(date +%Y%m%d_%H%M%S)

# Backup PostgreSQL
docker exec plantalytix-postgres pg_dump -U homeassistant homeassistant > \
    "${BACKUP_DIR}/postgres_${DATE}.sql"

# Backup InfluxDB
docker exec plantalytix-influxdb influx backup \
    /tmp/influx_backup_${DATE}
docker cp plantalytix-influxdb:/tmp/influx_backup_${DATE} \
    "${BACKUP_DIR}/influxdb_${DATE}"

# Compress and cleanup old backups
tar -czf "${BACKUP_DIR}/backup_${DATE}.tar.gz" \
    "${BACKUP_DIR}/postgres_${DATE}.sql" \
    "${BACKUP_DIR}/influxdb_${DATE}"
find "${BACKUP_DIR}" -name "*.tar.gz" -mtime +30 -delete
```

## Migration from Current System

### For Existing Users

1. **Export Data**: Export device configurations and historical data
2. **Install HA**: Set up new HA instance locally
3. **Import Data**: Import historical data to InfluxDB
4. **Reconfigure Devices**: Flash devices with new firmware (MQTT discovery)
5. **Validate**: Ensure all devices working
6. **Decommission**: Shut down old cloud system

### Data Migration

- Historical InfluxDB data can be exported and imported
- Device configurations converted to HA entities
- User settings migrated to HA user profile

## Alternatives Considered

### Alternative 1: Keep Cloud Multi-Tenant System

**Pros**:
- Centralized management
- Can manage multiple facilities
- No user hardware required

**Cons**:
- Higher infrastructure costs
- Complex multi-tenant architecture
- Cloud dependency
- Subscription model required
- Privacy concerns

**Verdict**: Rejected - local control and privacy preferred

### Alternative 2: Hybrid Cloud + Local

**Pros**:
- Local control with cloud backup
- Remote monitoring

**Cons**:
- Complexity of both systems
- Higher maintenance

**Verdict**: Deferred - can add Nabu Casa cloud later if needed

## References

- [Home Assistant Installation](https://www.home-assistant.io/installation/)
- [Home Assistant Docker](https://www.home-assistant.io/installation/generic-x86-64/)
- [Home Assistant Users](https://www.home-assistant.io/docs/authentication/)
- [PostgreSQL Recorder](https://www.home-assistant.io/integrations/recorder/)
- [InfluxDB Integration](https://www.home-assistant.io/integrations/influxdb/)

## Notes

**Security Considerations**:
- Local network only (no internet exposure by default)
- Optional remote access via VPN or Nabu Casa
- HA handles all authentication
- Regular backups essential
- Keep HA updated for security patches

**Maintenance**:
- Monthly HA updates (manual or auto)
- Database cleanup via HA recorder purge
- Backup verification
- Monitor disk space

**Support**:
- Home Assistant community forum
- Documentation at home-assistant.io
- GitHub issues for bugs

**Decision Date**: 2025-11-15
**Review Date**: After Phase 1 POC completion
