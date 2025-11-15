# Getting Started with Home Assistant for Plantalytix

This guide walks through setting up your first Home Assistant instance for Plantalytix development and testing.

## Prerequisites

- Docker and Docker Compose installed
- Basic understanding of YAML
- MQTT broker (Mosquitto) running
- One Plantalytix device for testing

## Quick Start (5 minutes)

### 1. Create Project Structure

```bash
mkdir -p ~/plantalytix-ha-test
cd ~/plantalytix-ha-test

# Create directory structure
mkdir -p {homeassistant,postgres,influxdb,mosquitto/config,mosquitto/data}
```

### 2. Create Docker Compose File

```bash
cat > docker-compose.yml <<'EOF'
version: '3.8'

services:
  homeassistant:
    container_name: plantalytix-ha
    image: ghcr.io/home-assistant/home-assistant:stable
    restart: unless-stopped
    environment:
      - TZ=America/Los_Angeles
    volumes:
      - ./homeassistant:/config
    ports:
      - "8123:8123"
    networks:
      - plantalytix
    depends_on:
      - postgres
      - mosquitto
      - influxdb

  postgres:
    container_name: plantalytix-postgres
    image: postgres:15
    restart: unless-stopped
    environment:
      POSTGRES_DB: homeassistant
      POSTGRES_USER: homeassistant
      POSTGRES_PASSWORD: changeme123
    volumes:
      - ./postgres:/var/lib/postgresql/data
    networks:
      - plantalytix

  mosquitto:
    container_name: plantalytix-mosquitto
    image: eclipse-mosquitto:latest
    restart: unless-stopped
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - ./mosquitto/data:/mosquitto/data
    networks:
      - plantalytix

  influxdb:
    container_name: plantalytix-influxdb
    image: influxdb:2
    restart: unless-stopped
    environment:
      DOCKER_INFLUXDB_INIT_MODE: setup
      DOCKER_INFLUXDB_INIT_USERNAME: admin
      DOCKER_INFLUXDB_INIT_PASSWORD: changeme123
      DOCKER_INFLUXDB_INIT_ORG: plantalytix
      DOCKER_INFLUXDB_INIT_BUCKET: homeassistant
      DOCKER_INFLUXDB_INIT_ADMIN_TOKEN: super-secret-token-changeme
    ports:
      - "8086:8086"
    volumes:
      - ./influxdb:/var/lib/influxdb2
    networks:
      - plantalytix

networks:
  plantalytix:
    driver: bridge
EOF
```

### 3. Configure Mosquitto

```bash
cat > mosquitto/config/mosquitto.conf <<'EOF'
listener 1883
allow_anonymous true

# For production, use authentication:
# allow_anonymous false
# password_file /mosquitto/config/passwd

persistence true
persistence_location /mosquitto/data/

log_dest stdout
log_type all
EOF
```

### 4. Start Services

```bash
docker-compose up -d

# Check logs
docker-compose logs -f homeassistant
```

### 5. Access Home Assistant

1. Open browser: `http://localhost:8123`
2. Wait for initialization (1-2 minutes)
3. Create admin account when prompted
4. Complete onboarding wizard

**Congratulations!** Home Assistant is running.

---

## Configure Home Assistant for Plantalytix

### 1. Edit Configuration

```bash
# Stop HA to edit config safely
docker-compose stop homeassistant

# Edit main configuration
nano homeassistant/configuration.yaml
```

Add the following:

```yaml
# homeassistant/configuration.yaml

# Load default configuration
default_config:

# Recorder with PostgreSQL
recorder:
  db_url: postgresql://homeassistant:changeme123@postgres:5432/homeassistant
  purge_keep_days: 14
  commit_interval: 1

# MQTT Integration
mqtt:
  broker: mosquitto
  port: 1883
  discovery: true
  discovery_prefix: homeassistant
  birth_message:
    topic: 'homeassistant/status'
    payload: 'online'
  will_message:
    topic: 'homeassistant/status'
    payload: 'offline'

# InfluxDB Integration
influxdb:
  api_version: 2
  host: influxdb
  port: 8086
  token: super-secret-token-changeme
  organization: plantalytix
  bucket: homeassistant
  tags:
    source: home_assistant
  tags_attributes:
    - friendly_name
    - device_id
  include:
    entity_globs:
      - sensor.plantalytix_*
      - climate.plantalytix_*

# Enable frontend customization
frontend:
  themes: !include_dir_merge_named themes

# Logger for debugging
logger:
  default: info
  logs:
    homeassistant.components.mqtt: debug
    homeassistant.components.recorder: info

# System health
system_health:

# Mobile app support
mobile_app:

# History
history:

# Logbook
logbook:
```

### 2. Create Themes Directory

```bash
mkdir -p homeassistant/themes

cat > homeassistant/themes/plantalytix.yaml <<'EOF'
plantalytix:
  # Primary colors (green for agriculture)
  primary-color: "#4CAF50"
  accent-color: "#8BC34A"

  # Text colors
  primary-text-color: "#212121"
  secondary-text-color: "#757575"
  disabled-text-color: "#BDBDBD"

  # Background colors
  primary-background-color: "#FAFAFA"
  secondary-background-color: "#FFFFFF"

  # Card colors
  card-background-color: "#FFFFFF"
  paper-card-background-color: "#FFFFFF"

  # State colors
  state-icon-active-color: "#4CAF50"
  state-icon-unavailable-color: "#BDBDBD"
EOF
```

### 3. Restart Home Assistant

```bash
docker-compose up -d homeassistant
```

---

## Test MQTT Discovery

### 1. Install MQTT Client

```bash
# On Ubuntu/Debian
sudo apt-get install mosquitto-clients

# On macOS
brew install mosquitto
```

### 2. Publish Test Discovery Message

```bash
# Climate entity discovery
mosquitto_pub -h localhost -p 1883 \
  -t 'homeassistant/climate/plantalytix_test001/config' \
  -r \
  -m '{
  "name": "Test Fridge",
  "unique_id": "plantalytix_climate_test001",
  "device": {
    "identifiers": ["plantalytix_test001"],
    "name": "Plantalytix Test Fridge 001",
    "model": "FridgeGrow 2.0",
    "manufacturer": "Plantalytix",
    "sw_version": "2.0.0"
  },
  "modes": ["off", "auto", "cool", "heat"],
  "temperature_unit": "C",
  "current_temperature_topic": "plantalytix/test001/sensor/temperature",
  "temperature_state_topic": "plantalytix/test001/state",
  "temperature_command_topic": "plantalytix/test001/set/temperature",
  "mode_state_topic": "plantalytix/test001/state",
  "mode_command_topic": "plantalytix/test001/set/mode",
  "availability_topic": "plantalytix/test001/status",
  "payload_available": "online",
  "payload_not_available": "offline"
}'

# Temperature sensor discovery
mosquitto_pub -h localhost -p 1883 \
  -t 'homeassistant/sensor/plantalytix_test001_temp/config' \
  -r \
  -m '{
  "name": "Temperature",
  "unique_id": "plantalytix_test001_temp",
  "device": {
    "identifiers": ["plantalytix_test001"]
  },
  "device_class": "temperature",
  "state_class": "measurement",
  "unit_of_measurement": "°C",
  "state_topic": "plantalytix/test001/sensor/temperature",
  "availability_topic": "plantalytix/test001/status"
}'

# Mark device as available
mosquitto_pub -h localhost -p 1883 \
  -t 'plantalytix/test001/status' \
  -r \
  -m 'online'
```

### 3. Verify in Home Assistant

1. Go to **Settings → Devices & Services → MQTT**
2. Click on **1 device** (should show Plantalytix Test Fridge 001)
3. Verify climate and sensor entities are created

### 4. Publish Test Data

```bash
# Publish temperature
mosquitto_pub -h localhost -p 1883 \
  -t 'plantalytix/test001/sensor/temperature' \
  -m '22.5'

# Publish climate state
mosquitto_pub -h localhost -p 1883 \
  -t 'plantalytix/test001/state' \
  -m '{"mode":"auto","target_temp":20.0}'
```

### 5. Subscribe to Commands

```bash
# Listen for commands from HA
mosquitto_sub -h localhost -p 1883 \
  -t 'plantalytix/test001/set/#' \
  -v
```

Now in HA UI, change the temperature. You should see the command in the subscriber terminal!

---

## Create Your First Dashboard

### 1. Create Dashboard

1. Go to **Overview** (main page)
2. Click **⋮ Menu → Edit Dashboard**
3. Click **+ Add Card**

### 2. Add Thermostat Card

```yaml
type: thermostat
entity: climate.plantalytix_test_fridge
name: Test Fridge Climate
```

### 3. Add Sensor Cards

```yaml
type: horizontal-stack
cards:
  - type: gauge
    entity: sensor.plantalytix_test_fridge_temperature
    min: 0
    max: 40
    name: Temperature
    severity:
      green: 15
      yellow: 25
      red: 30
```

### 4. Add History Graph

Install ApexCharts card (optional, for better graphs):

1. **Settings → Add-ons → Add-on Store**
2. Search for **HACS** (Home Assistant Community Store)
3. Install HACS
4. Use HACS to install **ApexCharts Card**

Then add to dashboard:

```yaml
type: custom:apexcharts-card
header:
  show: true
  title: Temperature History
graph_span: 24h
series:
  - entity: sensor.plantalytix_test_fridge_temperature
    name: Temperature
    stroke_width: 2
```

---

## Troubleshooting

### HA Won't Start

```bash
# Check logs
docker-compose logs -f homeassistant

# Common issue: permissions
sudo chown -R 1000:1000 homeassistant/

# Restart
docker-compose restart homeassistant
```

### MQTT Discovery Not Working

```bash
# Verify MQTT integration
docker-compose exec homeassistant ha core check

# Check MQTT logs
docker-compose logs mosquitto

# Verify discovery prefix
mosquitto_sub -h localhost -p 1883 -t 'homeassistant/#' -v
```

### Database Errors

```bash
# Check PostgreSQL
docker-compose logs postgres

# Reset database (WARNING: deletes data)
docker-compose down
rm -rf postgres/*
docker-compose up -d
```

### Can't Access HA UI

```bash
# Verify port not in use
netstat -an | grep 8123

# Check HA is running
docker ps | grep homeassistant

# Check firewall
sudo ufw allow 8123
```

---

## Next Steps

### 1. Flash Test Device

Use the MQTT discovery firmware from ADR-0008 to flash a real device.

### 2. Install HACS

Install [Home Assistant Community Store](https://hacs.xyz/) for custom cards:

```bash
# Access HA container
docker-compose exec homeassistant bash

# Download HACS
wget -O - https://get.hacs.xyz | bash -
```

### 3. Create Advanced Dashboards

Explore custom cards:
- **ApexCharts**: Better time-series graphs
- **Mushroom Cards**: Modern minimalist UI
- **Auto-Entities**: Dynamic card generation
- **Button Card**: Highly customizable buttons

### 4. Set Up Automations

Create your first automation:

1. **Settings → Automations & Scenes → + Create Automation**
2. **Trigger**: Numeric state (temperature > 30°C)
3. **Action**: Send notification

### 5. Configure Mobile App

1. Install **Home Assistant Companion** app (iOS/Android)
2. Add server: `http://your-ip:8123`
3. Log in with your account
4. Enable notifications

---

## Production Deployment

### Security Hardening

```yaml
# homeassistant/configuration.yaml
http:
  server_port: 8123
  ssl_certificate: /ssl/fullchain.pem
  ssl_key: /ssl/privkey.pem
  ip_ban_enabled: true
  login_attempts_threshold: 5
```

### Mosquitto Authentication

```bash
# Create password file
docker-compose exec mosquitto mosquitto_passwd -c /mosquitto/config/passwd admin

# Update mosquitto.conf
cat > mosquitto/config/mosquitto.conf <<'EOF'
listener 1883
allow_anonymous false
password_file /mosquitto/config/passwd
EOF

# Restart
docker-compose restart mosquitto
```

### Backups

```bash
# Automated backup script
cat > backup.sh <<'EOF'
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="./backups/$DATE"

mkdir -p "$BACKUP_DIR"

# Backup HA config
tar -czf "$BACKUP_DIR/homeassistant.tar.gz" homeassistant/

# Backup PostgreSQL
docker-compose exec -T postgres pg_dump -U homeassistant homeassistant > "$BACKUP_DIR/postgres.sql"

# Backup InfluxDB
docker-compose exec -T influxdb influx backup /tmp/backup
docker cp plantalytix-influxdb:/tmp/backup "$BACKUP_DIR/influxdb"

echo "Backup completed: $BACKUP_DIR"
EOF

chmod +x backup.sh

# Schedule with cron
crontab -e
# Add: 0 2 * * * /path/to/backup.sh
```

---

## Resources

### Documentation
- [Home Assistant Documentation](https://www.home-assistant.io/docs/)
- [MQTT Integration](https://www.home-assistant.io/integrations/mqtt/)
- [Lovelace UI](https://www.home-assistant.io/dashboards/)

### Community
- [Home Assistant Community Forum](https://community.home-assistant.io/)
- [Home Assistant Discord](https://discord.gg/home-assistant)
- [Reddit r/homeassistant](https://www.reddit.com/r/homeassistant/)

### YouTube Channels
- [Everything Smart Home](https://www.youtube.com/c/EverythingSmartHome)
- [Home Automation Guy](https://www.youtube.com/c/HomeAutomationGuy)
- [Smart Home Solver](https://www.youtube.com/c/SmartHomeSolver)

---

**Need Help?**

Check the [Plantalytix Migration ADRs](./decisions/README.md) for architectural decisions and detailed implementation guides.
