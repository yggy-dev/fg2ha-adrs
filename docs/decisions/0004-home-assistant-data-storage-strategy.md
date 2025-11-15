# Home Assistant Data Storage Strategy

## Status

Proposed

## Context

With the migration to Home Assistant (ADR-0001), we need to determine how to handle data storage and time-series data management.

**Current Plantalytix Data Architecture:**
- **MongoDB**: User accounts, device registry, configurations, logs, firmware metadata
- **InfluxDB v2**: Time-series sensor data (temperature, humidity, CO2, output states)
- **Data Retention**: Configurable in InfluxDB, MongoDB keeps all historical data
- **Query Interface**: Custom REST API endpoints for data retrieval
- **Visualization**: Custom Angular charts (Highcharts)

**Home Assistant Data Options:**

### 1. Home Assistant Recorder (Default)

**Technology**: SQLite (default) or MariaDB/PostgreSQL
**Purpose**: Short-term state history (default 10 days)

**Schema**:
- `states` table: All entity state changes
- `events` table: System events, automations
- `statistics` table: Aggregated hourly/daily stats
- `statistics_short_term` table: 5-minute statistics

**Limitations**:
- Not optimized for time-series queries
- Default 10-day retention (configurable)
- Large database size with many sensors
- Slower queries for long-term analysis

### 2. InfluxDB Integration

**Technology**: InfluxDB v1.x or v2.x
**Purpose**: Long-term time-series storage

**Integration Options**:
- **Option A**: InfluxDB v1.x integration (legacy, stable)
- **Option B**: InfluxDB v2.x integration (newer, InfluxQL and Flux)

**Features**:
- Streams state changes to InfluxDB
- Configurable entity inclusion/exclusion
- Retention policies
- Downsampling and aggregation
- Grafana integration for visualization

**Limitations**:
- Runs alongside Recorder (not a replacement)
- Requires separate InfluxDB instance
- No built-in HA UI for InfluxDB queries

### 3. PostgreSQL + TimescaleDB

**Technology**: PostgreSQL with TimescaleDB extension
**Purpose**: Replace Recorder with time-series optimized DB

**Features**:
- SQL interface familiar to developers
- Time-series optimizations (hypertables)
- Compression and retention policies
- Single database for all data

**Limitations**:
- Requires custom add-on installation
- More complex setup than SQLite/MariaDB
- Less common in HA community

### 4. Prometheus + Grafana

**Technology**: Prometheus for metrics, Grafana for visualization
**Purpose**: Alternative time-series stack

**Features**:
- Purpose-built for metrics
- Powerful query language (PromQL)
- Industry-standard monitoring stack

**Limitations**:
- Requires separate services
- Not integrated with HA UI
- More complex setup

**Data Flow Comparison:**

**Current (Plantalytix):**
```
Device → MQTT → API Server → MongoDB (metadata)
                           → InfluxDB (measurements)

User → Angular UI → REST API → MongoDB/InfluxDB → Highcharts
```

**Proposed (Home Assistant):**
```
Device → MQTT → Home Assistant → Recorder (PostgreSQL) → 10-day history
                               → InfluxDB Integration → Long-term storage

User → Lovelace UI → HA Frontend → Recorder (short-term)
                                 → InfluxDB (long-term, via Grafana)
```

## Decision

**We will use a dual-database strategy: PostgreSQL for Home Assistant Recorder (short-term) + InfluxDB v2 for long-term time-series data.**

### Architecture

**PostgreSQL Recorder**:
- **Purpose**: HA state history, automations, short-term data
- **Retention**: 14 days (configurable)
- **Technology**: PostgreSQL 15+
- **Access**: Home Assistant UI (history graphs, logbook)
- **Backup**: Daily snapshots via HA backup system

**InfluxDB v2**:
- **Purpose**: Long-term sensor data, historical analysis
- **Retention**: 1 year (or longer based on storage)
- **Technology**: InfluxDB v2 (maintained from current setup)
- **Access**: Grafana dashboards, HA InfluxDB integration
- **Downsampling**: Aggregate old data (e.g., 10s → 1m → 1h)

### Configuration

**Home Assistant Recorder** (`configuration.yaml`):
```yaml
recorder:
  db_url: postgresql://homeassistant:password@postgres:5432/homeassistant
  purge_keep_days: 14
  commit_interval: 1

  # Include only necessary entities
  include:
    domains:
      - climate
      - sensor
      - light
      - fan
      - switch
    entity_globs:
      - sensor.plantalytix_*
      - climate.plantalytix_*

  # Exclude high-frequency sensors from recorder
  exclude:
    entities:
      - sensor.plantalytix_*_raw  # Exclude raw high-frequency data
      - sensor.uptime
      - sensor.time
```

**InfluxDB Integration** (`configuration.yaml`):
```yaml
influxdb:
  api_version: 2
  host: influxdb
  port: 8086
  token: !secret influxdb_token
  organization: plantalytix
  bucket: homeassistant

  # Include Plantalytix sensor data
  include:
    domains:
      - sensor
      - climate
    entity_globs:
      - sensor.plantalytix_*_temperature
      - sensor.plantalytix_*_humidity
      - sensor.plantalytix_*_co2
      - climate.plantalytix_*

  # Exclude non-essential data
  exclude:
    entities:
      - sensor.plantalytix_*_wifi_signal
      - sensor.plantalytix_*_uptime

  # Tags for better querying
  tags:
    source: home_assistant
  tags_attributes:
    - friendly_name
    - device_id
```

**InfluxDB Retention Policies** (InfluxDB UI or CLI):
```
# Raw data: 30 days
# Downsampled to 1-minute: 90 days
# Downsampled to 1-hour: 1 year
# Downsampled to 1-day: 5 years

# Continuous query example (Flux)
option task = {
  name: "downsample_to_1m",
  every: 1h,
}

from(bucket: "homeassistant")
  |> range(start: -1h)
  |> filter(fn: (r) => r["_measurement"] == "°C" or r["_measurement"] == "%")
  |> aggregateWindow(every: 1m, fn: mean)
  |> to(bucket: "homeassistant_1m")
```

### Data Mapping

**Plantalytix MongoDB → Home Assistant**:
- User accounts → HA user system
- Device registry → HA device registry (via MQTT discovery)
- Device configurations → HA entity attributes
- Device logs → HA logbook and system log
- Firmware metadata → File storage + custom integration

**Plantalytix InfluxDB → HA InfluxDB**:
- Migrate existing data using data transformation
- Map device_id tags to new HA entity IDs
- Maintain measurement names for continuity

### Migration Process

**Phase 1: Dual Write (Transition)**
- Devices publish to both old and new MQTT topics
- API server writes to Plantalytix InfluxDB
- HA writes to new InfluxDB bucket

**Phase 2: Data Migration**
- Export Plantalytix InfluxDB data
- Transform to HA entity format
- Import to new InfluxDB bucket

**Phase 3: Cutover**
- Disable old API server
- HA becomes primary data sink
- Grafana dashboards updated to new bucket

## Consequences

### Positive

✅ **Optimized Storage**: PostgreSQL for relational data, InfluxDB for time-series

✅ **Performance**: Fast queries for both short-term (Postgres) and long-term (InfluxDB)

✅ **Built-in Features**: HA recorder provides logbook, history, statistics

✅ **Long-term Analysis**: InfluxDB retains years of data for trending

✅ **Grafana Integration**: Advanced visualizations via Grafana

✅ **Resource Efficiency**: Short retention in Postgres keeps DB small

✅ **Familiar Tools**: Reuse existing Grafana dashboards with minimal changes

✅ **Downsampling**: Reduce storage with aggregated data over time

✅ **Backup Strategy**: HA snapshots for Postgres, InfluxDB backup for time-series

✅ **Query Flexibility**: SQL for Postgres, Flux for InfluxDB

### Negative

❌ **Dual Databases**: Must maintain both PostgreSQL and InfluxDB

❌ **Complexity**: More complex than single-database approach

❌ **Sync Issues**: State in Postgres and InfluxDB may diverge

❌ **Storage Overhead**: Some data duplicated between databases

❌ **Configuration Management**: Must configure both Recorder and InfluxDB integration

❌ **Migration Effort**: Must migrate existing InfluxDB data

❌ **Query Inconsistency**: Different query languages for different data sources

### Risks

⚠️ **Data Loss**: Migration errors could lose historical data

⚠️ **Storage Growth**: Without proper retention policies, InfluxDB can grow unbounded

⚠️ **Performance**: High sensor update frequency could overwhelm databases

⚠️ **Integration Breakage**: HA updates may break InfluxDB integration

### Mitigation Strategies

1. **Backup Before Migration**: Full backup of Plantalytix InfluxDB before migration
2. **Validation Scripts**: Compare data before/after migration
3. **Retention Policies**: Strict retention and downsampling rules
4. **Monitoring**: Alert on database size and query performance
5. **Gradual Migration**: Migrate device-by-device, validate each

## Alternatives Considered

### Alternative 1: Recorder Only (No InfluxDB)

**Description**: Use PostgreSQL Recorder with long retention (1+ year)

**Pros**:
- Simpler architecture (one database)
- All data accessible in HA UI
- Less configuration

**Cons**:
- Large database size (100GB+ for 1 year)
- Slow queries for historical data
- Not optimized for time-series analysis
- No downsampling (raw data only)

**Verdict**: Rejected - poor performance and storage efficiency

### Alternative 2: InfluxDB Only (No Recorder)

**Description**: Disable Recorder, use InfluxDB for everything

**Pros**:
- Single time-series database
- Optimized for sensor data

**Cons**:
- HA UI relies on Recorder (logbook, history)
- Can't disable Recorder without losing HA features
- InfluxDB not suitable for event data

**Verdict**: Rejected - HA requires Recorder for core features

### Alternative 3: TimescaleDB as Recorder Database

**Description**: Use PostgreSQL + TimescaleDB extension for Recorder

**Pros**:
- Single database for all data
- Time-series optimizations
- SQL interface

**Cons**:
- Requires custom add-on (not officially supported)
- Still need InfluxDB for Grafana compatibility
- More complex setup

**Verdict**: Rejected - adds complexity without eliminating InfluxDB

### Alternative 4: SQLite Recorder + InfluxDB

**Description**: Use default SQLite for Recorder

**Pros**:
- Simple setup
- No PostgreSQL installation

**Cons**:
- SQLite not suitable for high write loads
- Concurrent access issues
- Poor performance with many sensors

**Verdict**: Rejected - performance concerns with many devices

### Alternative 5: Prometheus + Grafana (No InfluxDB)

**Description**: Replace InfluxDB with Prometheus

**Pros**:
- Industry-standard metrics stack
- Powerful query language (PromQL)

**Cons**:
- Requires rewriting Grafana dashboards
- Migration from InfluxDB to Prometheus format
- HA Prometheus integration less mature

**Verdict**: Rejected - migration effort not justified

## Implementation Plan

### Phase 1: PostgreSQL Recorder Setup (1 week)

1. **Week 1: Infrastructure**
   - Add PostgreSQL service to docker-compose
   - Configure HA Recorder with PostgreSQL
   - Test state recording and history
   - Configure 14-day retention
   - Set up daily backups

### Phase 2: InfluxDB Integration (1 week)

1. **Week 1: Configuration**
   - Maintain existing InfluxDB v2 instance
   - Configure HA InfluxDB integration
   - Test data flow from HA to InfluxDB
   - Set up retention policies
   - Configure downsampling tasks

### Phase 3: Data Migration (2 weeks)

1. **Week 1: Export & Transform**
   - Export Plantalytix InfluxDB data
   - Write transformation scripts (device_id → entity_id)
   - Test transformation with sample data

2. **Week 2: Import & Validate**
   - Import transformed data to new InfluxDB bucket
   - Validate data integrity (count, ranges)
   - Test Grafana queries against new bucket

### Phase 4: Grafana Dashboard Migration (1 week)

1. **Week 1: Dashboard Updates**
   - Update Grafana datasource to new InfluxDB bucket
   - Update queries for new schema (entity_id vs device_id)
   - Create new dashboards for HA entities
   - Test all dashboard panels

## Database Sizing Estimates

### PostgreSQL Recorder (14 days)

**Assumptions**:
- 20 devices
- 5 sensors per device (100 sensors total)
- 1 climate entity per device (20 climate entities)
- 10-second update interval for sensors
- Total: 120 entities

**Calculations**:
- States per day: 120 entities × (86400s / 10s) = 1,036,800 states/day
- States in 14 days: 14,515,200 states
- Estimated size: ~150 bytes per state = ~2 GB

**With indexing and events**: ~5 GB total for 14 days

### InfluxDB v2 (1 year)

**Assumptions**:
- 100 sensor entities
- 10-second intervals (raw data: 30 days)
- 1-minute downsampled (90 days)
- 1-hour downsampled (1 year)

**Calculations**:

**Raw data (30 days)**:
- Points per day: 100 sensors × (86400s / 10s) = 864,000 points/day
- Points in 30 days: 25,920,000 points
- Size: ~50 bytes per point = ~1.3 GB

**1-minute downsampled (90 days)**:
- Points per day: 100 sensors × (86400s / 60s) = 144,000 points/day
- Points in 90 days: 12,960,000 points
- Size: ~50 bytes per point = ~650 MB

**1-hour downsampled (1 year)**:
- Points per day: 100 sensors × 24 = 2,400 points/day
- Points in 365 days: 876,000 points
- Size: ~50 bytes per point = ~44 MB

**Total**: ~2 GB for 1 year (with downsampling)

## Configuration Examples

### docker-compose.yaml

```yaml
version: '3'

services:
  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    depends_on:
      - postgres
      - influxdb
    environment:
      - TZ=America/Los_Angeles
    volumes:
      - ./homeassistant:/config
    network_mode: host

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: homeassistant
      POSTGRES_USER: homeassistant
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - ./postgres:/var/lib/postgresql/data

  influxdb:
    image: influxdb:2
    environment:
      DOCKER_INFLUXDB_INIT_MODE: setup
      DOCKER_INFLUXDB_INIT_USERNAME: admin
      DOCKER_INFLUXDB_INIT_PASSWORD: ${INFLUXDB_PASSWORD}
      DOCKER_INFLUXDB_INIT_ORG: plantalytix
      DOCKER_INFLUXDB_INIT_BUCKET: homeassistant
      DOCKER_INFLUXDB_INIT_ADMIN_TOKEN: ${INFLUXDB_TOKEN}
    volumes:
      - ./influxdb:/var/lib/influxdb2
    ports:
      - "8086:8086"

  grafana:
    image: grafana/grafana:latest
    depends_on:
      - influxdb
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
    volumes:
      - ./grafana:/var/lib/grafana
    ports:
      - "3000:3000"
```

### Data Migration Script (Python)

```python
#!/usr/bin/env python3
"""
Migrate Plantalytix InfluxDB data to Home Assistant format
"""

from influxdb_client import InfluxDBClient
from datetime import datetime, timedelta

# Old Plantalytix InfluxDB
old_client = InfluxDBClient(
    url="http://localhost:8086",
    token="old_token",
    org="plantalytix"
)
old_query_api = old_client.query_api()

# New HA InfluxDB
new_client = InfluxDBClient(
    url="http://localhost:8086",
    token="new_token",
    org="plantalytix"
)
new_write_api = new_client.write_api()

# Device ID to Entity ID mapping
device_mapping = {
    "device-uuid-001": "sensor.plantalytix_fridge_001_temperature",
    "device-uuid-002": "sensor.plantalytix_fan_001_temperature",
    # ... more mappings
}

# Query old data
query = '''
from(bucket: "plantalytix")
  |> range(start: -1y)
  |> filter(fn: (r) => r["_measurement"] == "status")
  |> filter(fn: (r) => r["_field"] == "temperature")
'''

result = old_query_api.query(query)

# Transform and write to new bucket
for table in result:
    for record in table.records:
        device_id = record.values.get("device_id")
        entity_id = device_mapping.get(device_id)

        if entity_id:
            point = {
                "measurement": "°C",
                "tags": {
                    "entity_id": entity_id,
                    "domain": "sensor",
                    "friendly_name": f"Plantalytix {device_id} Temperature"
                },
                "fields": {
                    "value": record.get_value()
                },
                "time": record.get_time()
            }

            new_write_api.write(bucket="homeassistant", record=point)

print("Migration complete!")
```

## References

- [Home Assistant Recorder](https://www.home-assistant.io/integrations/recorder/)
- [Home Assistant InfluxDB Integration](https://www.home-assistant.io/integrations/influxdb/)
- [PostgreSQL Home Assistant Database](https://www.home-assistant.io/integrations/recorder/#postgresql)
- [InfluxDB v2 Documentation](https://docs.influxdata.com/influxdb/v2/)
- [TimescaleDB Home Assistant Add-on](https://github.com/Expaso/hassos-addon-timescaledb)

## Notes

**Data Governance**:
- Implement automated backups for both databases
- Monitor storage usage and set alerts
- Document retention policies for compliance
- Provide data export functionality for users

**Performance Monitoring**:
- Track Recorder query response times
- Monitor InfluxDB write throughput
- Alert on database size thresholds
- Optimize queries based on usage patterns

**Testing Checklist**:
- [ ] PostgreSQL Recorder stores state changes
- [ ] InfluxDB receives sensor data
- [ ] Retention policies purge old data
- [ ] Grafana dashboards render correctly
- [ ] Data migration preserves historical data
- [ ] Backup and restore procedures work
- [ ] Query performance acceptable

**Decision Date**: 2025-11-15
**Review Date**: After Phase 3 data migration completion
