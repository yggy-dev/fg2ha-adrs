---
title: Plantalytix Home Assistant Architecture
description: Final architecture after migration to Home Assistant platform (Single-User Local Deployment)
version: 2.0
date: 2025-11-15
---

# Plantalytix Home Assistant Architecture
## Single-User Local Network Deployment

## System Overview Diagram

```mermaid
graph TB
    subgraph "User Devices"
        WebUI[Web Browser<br/>http://homeassistant.local:8123]
        MobileApp[HA Companion App<br/>iOS/Android]
    end

    subgraph "Home Assistant Server (Local Network)"
        HA[Home Assistant Core<br/>All-in-one Platform]
        PG[(PostgreSQL<br/>Recorder<br/>14 days)]
        InfluxDB[(InfluxDB v2<br/>Long-term Storage<br/>1 year+)]
        MQTT[Mosquitto MQTT Broker<br/>Local Message Bus]
    end

    subgraph "Plantalytix IoT Devices (Local Network)"
        Fridge[Fridge Controller<br/>Arduino + MQTT Discovery<br/>192.168.1.101]
        Fan[Fan Controller<br/>Arduino + MQTT Discovery<br/>192.168.1.102]
        Light[Light Controller<br/>Arduino + MQTT Discovery<br/>192.168.1.103]
        Socket[Smart Socket<br/>Arduino + MQTT Discovery<br/>192.168.1.104]
    end

    %% User Connections
    WebUI -->|HTTP :8123| HA
    MobileApp -->|Local Network| HA

    %% HA to Databases
    HA -->|14-day history| PG
    HA -->|Long-term metrics| InfluxDB

    %% HA to MQTT
    HA <-->|Subscribe/Publish| MQTT

    %% Devices to MQTT
    Fridge <-->|MQTT :1883<br/>plantalytix/fridge001/*| MQTT
    Fan <-->|MQTT :1883<br/>plantalytix/fan001/*| MQTT
    Light <-->|MQTT :1883<br/>plantalytix/light001/*| MQTT
    Socket <-->|MQTT :1883<br/>plantalytix/socket001/*| MQTT

    %% Styling
    classDef userLayer fill:#E3F2FD,stroke:#1976D2,stroke-width:2px
    classDef haLayer fill:#C8E6C9,stroke:#388E3C,stroke-width:2px
    classDef deviceLayer fill:#F8BBD0,stroke:#C2185B,stroke-width:2px
    classDef dbLayer fill:#E1BEE7,stroke:#7B1FA2,stroke-width:2px

    class WebUI,MobileApp userLayer
    class HA haLayer
    class MQTT haLayer
    class Fridge,Fan,Light,Socket deviceLayer
    class PG,InfluxDB dbLayer
```

## MQTT Discovery Flow

```mermaid
sequenceDiagram
    participant Device as ESP32 Device<br/>(Fridge Controller)
    participant MQTT as MQTT Broker<br/>(Mosquitto)
    participant HA as Home Assistant
    participant UI as User Interface<br/>(Lovelace)

    Note over Device: Device boots up
    Device->>MQTT: Connect to broker
    MQTT-->>Device: Connected

    Note over Device: Publish discovery config
    Device->>MQTT: homeassistant/climate/fridge001/config<br/>{name, unique_id, device, topics...}
    Device->>MQTT: homeassistant/sensor/fridge001_temp/config<br/>{name, device_class, state_topic...}
    Device->>MQTT: homeassistant/sensor/fridge001_humidity/config
    Device->>MQTT: homeassistant/sensor/fridge001_co2/config

    MQTT->>HA: Forward discovery messages

    Note over HA: Auto-create entities
    HA->>HA: Create climate.fridge001
    HA->>HA: Create sensor.fridge001_temp
    HA->>HA: Create sensor.fridge001_humidity
    HA->>HA: Create sensor.fridge001_co2

    Note over Device: Mark as available
    Device->>MQTT: plantalytix/fridge001/status = "online"
    MQTT->>HA: Forward status
    HA->>HA: Mark entities available

    Note over Device: Publish sensor data
    Device->>MQTT: plantalytix/fridge001/sensor/temperature = 22.5
    MQTT->>HA: Forward data
    HA->>HA: Update sensor.fridge001_temp
    HA->>UI: Display in dashboard

    Note over UI: User sets temperature
    UI->>HA: Set climate.fridge001 to 20°C
    HA->>MQTT: plantalytix/fridge001/set/temperature = 20.0
    MQTT->>Device: Forward command
    Device->>Device: Update target temperature

    Note over Device: Publish state update
    Device->>MQTT: plantalytix/fridge001/state<br/>{mode: "auto", target_temp: 20.0}
    MQTT->>HA: Forward state
    HA->>UI: Update UI
```

## Data Flow Architecture

```mermaid
graph LR
    subgraph "Device Firmware"
        Sensors[Sensors<br/>SHT31, SCD30, etc.]
        PID[PID Controller<br/>Autonomous]
        Outputs[Outputs<br/>Heater, Fan, etc.]
        MQTT_Client[MQTT Client<br/>PubSubClient]
    end

    subgraph "Home Assistant"
        MQTT_Int[MQTT Integration<br/>Discovery]
        Climate[Climate Entity<br/>Thermostat]
        SensorEnts[Sensor Entities<br/>Temp, Humidity, CO2]
        Recorder[Recorder<br/>PostgreSQL]
        InfluxInt[InfluxDB Integration]
        Automations[Automations<br/>YAML/UI]
        AppDaemon[AppDaemon<br/>Python Logic]
    end

    subgraph "Storage"
        PG[(PostgreSQL<br/>14 days)]
        Influx[(InfluxDB v2<br/>1 year+)]
    end

    subgraph "User Interface"
        Lovelace[Lovelace Dashboard<br/>ApexCharts]
        Mobile[Mobile App<br/>Notifications]
    end

    %% Sensor flow
    Sensors -->|Read every 10s| PID
    PID -->|Control signal| Outputs
    Sensors -->|Temperature| MQTT_Client
    Outputs -->|State| MQTT_Client

    %% MQTT to HA
    MQTT_Client -->|Publish state| MQTT_Int
    MQTT_Int -->|Update| Climate
    MQTT_Int -->|Update| SensorEnts

    %% Storage
    Climate -->|State changes| Recorder
    SensorEnts -->|Measurements| Recorder
    Recorder -->|Write| PG
    Climate -->|Time-series| InfluxInt
    SensorEnts -->|Time-series| InfluxInt
    InfluxInt -->|Write| Influx

    %% Automations
    Climate -->|Trigger| Automations
    SensorEnts -->|Trigger| Automations
    Automations -->|Control| Climate
    SensorEnts -->|Trigger| AppDaemon
    AppDaemon -->|Advanced logic| Climate

    %% UI
    Climate -->|Display| Lovelace
    SensorEnts -->|Charts| Lovelace
    Influx -->|Historical data| Lovelace
    Climate -->|Alerts| Mobile
    Automations -->|Notifications| Mobile

    %% Styling
    classDef firmware fill:#FFE0B2,stroke:#E64A19
    classDef ha fill:#C8E6C9,stroke:#388E3C
    classDef storage fill:#E1BEE7,stroke:#7B1FA2
    classDef ui fill:#E3F2FD,stroke:#1976D2

    class Sensors,PID,Outputs,MQTT_Client firmware
    class MQTT_Int,Climate,SensorEnts,Recorder,InfluxInt,Automations,AppDaemon ha
    class PG,Influx storage
    class Lovelace,Mobile ui
```

## Firmware Architecture

```mermaid
graph TB
    subgraph "All Plantalytix Devices (Arduino)"
        Arduino[Arduino Framework<br/>PlatformIO]
        ArduinoCore[Core Libraries<br/>wifi, settings, ui]
        HWSpecific[Hardware Specific<br/>sensors, automation, PID]
        MQTTDisc[MQTT Discovery<br/>HA Integration]
        OTA[OTA Updates<br/>HTTP + ArduinoOTA]

        Arduino --> ArduinoCore
        Arduino --> HWSpecific
        ArduinoCore --> MQTTDisc
        ArduinoCore --> OTA
    end

    subgraph "ESPHome (Optional for Future)"
        ESPHome[ESPHome YAML<br/>For new devices only]
        ESPNote[Not used for<br/>existing hardware]

        ESPHome -.->|Optional| ESPNote
    end

    subgraph "Existing Plantalytix Devices"
        Fridge[Fridge Controller<br/>Complex PID logic<br/>OLED UI]
        Fan[Fan Controller<br/>PWM control<br/>Temp sensors]
        Light[Light Controller<br/>Dimming control]
        Socket[Smart Socket<br/>Relay + monitoring]
    end

    Arduino -->|All devices| Fridge
    Arduino -->|All devices| Fan
    Arduino -->|All devices| Light
    Arduino -->|All devices| Socket

    classDef arduinoStyle fill:#FFE0B2,stroke:#E64A19,stroke-width:2px
    classDef esphomeStyle fill:#E8F5E9,stroke:#4CAF50,stroke-width:1px,stroke-dasharray: 5 5
    classDef deviceStyle fill:#E1BEE7,stroke:#7B1FA2,stroke-width:2px

    class Arduino,ArduinoCore,HWSpecific,MQTTDisc,OTA arduinoStyle
    class ESPHome,ESPNote esphomeStyle
    class Fridge,Fan,Light,Socket deviceStyle
```

## Automation & Control Logic Distribution

```mermaid
graph TB
    subgraph "Firmware Layer (Autonomous)"
        FW_PID[PID Controller<br/>Temperature control]
        FW_Safety[Safety Limits<br/>Min/max enforcement]
        FW_Sensors[Sensor Reading<br/>10-second interval]
        FW_Outputs[Output Control<br/>PWM, relays]
        FW_DayNight[Day/Night Mode<br/>Local RTC]
    end

    subgraph "Home Assistant Layer (Coordination)"
        HA_Schedule[Scheduling<br/>Automations]
        HA_Notify[Notifications<br/>Alerts, warnings]
        HA_Multi[Multi-Device<br/>Coordination]
        HA_Scenes[Scenes<br/>Preset configurations]
        HA_Stats[Statistics<br/>Aggregation]
    end

    subgraph "AppDaemon Layer (Advanced)"
        AD_ML[Machine Learning<br/>Predictive control]
        AD_Complex[Complex Logic<br/>Python-based]
        AD_External[External Integrations<br/>Weather, electricity pricing]
    end

    subgraph "User Interface"
        UI_Manual[Manual Override<br/>User control]
        UI_Config[Configuration<br/>Settings]
        UI_Monitor[Monitoring<br/>Real-time display]
    end

    %% Firmware connections
    FW_Sensors -->|Readings| FW_PID
    FW_PID -->|Control signal| FW_Outputs
    FW_Safety -->|Limits| FW_PID
    FW_DayNight -->|Mode| FW_Outputs

    %% HA connections
    HA_Schedule -->|Target temp| FW_PID
    FW_Sensors -->|Alerts| HA_Notify
    HA_Multi -->|Coordination| FW_PID
    HA_Scenes -->|Preset| FW_PID
    FW_Sensors -->|Data| HA_Stats

    %% AppDaemon connections
    AD_ML -->|Predictions| HA_Schedule
    AD_Complex -->|Advanced control| FW_PID
    AD_External -->|Context| HA_Schedule

    %% UI connections
    UI_Manual -->|Override| FW_PID
    UI_Config -->|Settings| FW_PID
    FW_Sensors -->|Display| UI_Monitor
    FW_PID -->|Status| UI_Monitor

    classDef fwStyle fill:#FFE0B2,stroke:#E64A19,stroke-width:2px
    classDef haStyle fill:#C8E6C9,stroke:#388E3C,stroke-width:2px
    classDef adStyle fill:#FFF9C4,stroke:#F57C00,stroke-width:2px
    classDef uiStyle fill:#E3F2FD,stroke:#1976D2,stroke-width:2px

    class FW_PID,FW_Safety,FW_Sensors,FW_Outputs,FW_DayNight fwStyle
    class HA_Schedule,HA_Notify,HA_Multi,HA_Scenes,HA_Stats haStyle
    class AD_ML,AD_Complex,AD_External adStyle
    class UI_Manual,UI_Config,UI_Monitor uiStyle
```

## Local Network Deployment

```mermaid
graph TB
    subgraph "Local Network - 192.168.1.0/24"
        subgraph "HA Server (192.168.1.50)"
            HA[Home Assistant<br/>:8123]
            PG[(PostgreSQL<br/>:5432)]
            Influx[(InfluxDB<br/>:8086)]
            MQTT[Mosquitto<br/>:1883]
        end

        subgraph "Plantalytix Devices"
            Device1[Fridge 001<br/>192.168.1.101]
            Device2[Fan 001<br/>192.168.1.102]
            Device3[Light 001<br/>192.168.1.103]
            Device4[Socket 001<br/>192.168.1.104]
        end

        subgraph "User Devices"
            Laptop[Laptop/Desktop<br/>192.168.1.x]
            Phone[Smartphone<br/>192.168.1.y]
        end

        Router[WiFi Router<br/>192.168.1.1]
    end

    Internet[Internet<br/>Optional]

    %% Router connections
    Router --- HA
    Router --- Device1
    Router --- Device2
    Router --- Device3
    Router --- Device4
    Router --- Laptop
    Router --- Phone

    %% HA internal
    HA --- PG
    HA --- Influx
    HA --- MQTT

    %% Device to MQTT
    Device1 <-.->|MQTT| MQTT
    Device2 <-.->|MQTT| MQTT
    Device3 <-.->|MQTT| MQTT
    Device4 <-.->|MQTT| MQTT

    %% User to HA
    Laptop -.->|HTTP :8123| HA
    Phone -.->|HA App| HA

    %% Optional internet
    Router -.->|Optional<br/>Remote Access| Internet

    classDef serverStyle fill:#C8E6C9,stroke:#388E3C,stroke-width:2px
    classDef deviceStyle fill:#FFCCBC,stroke:#D84315,stroke-width:2px
    classDef userStyle fill:#E3F2FD,stroke:#1976D2,stroke-width:2px
    classDef networkStyle fill:#FFF9C4,stroke:#F57C00,stroke-width:2px

    class HA,PG,Influx,MQTT serverStyle
    class Device1,Device2,Device3,Device4 deviceStyle
    class Laptop,Phone userStyle
    class Router,Internet networkStyle
```

## Firmware Update Management

```mermaid
graph TB
    subgraph "Firmware Sources"
        GitHub[GitHub Releases<br/>Official stable]
        GitLab[GitLab Releases<br/>Community beta]
        Custom[Custom HTTP<br/>Experimental]
    end

    subgraph "Home Assistant Integration"
        SourceMgr[Source Manager<br/>Poll every 1h]
        UpdateCoord[Update Coordinator<br/>Version tracking]
        UpdateEntity[Update Entities<br/>Per device]
    end

    subgraph "Update UI"
        Settings[Settings → Updates]
        Changelog[Changelog Display]
        Controls[Install/Skip/Rollback]
    end

    subgraph "Device Update"
        MQTT_Notify[MQTT Notification<br/>firmware/update topic]
        Download[HTTP Download<br/>Firmware binary]
        Flash[Flash & Reboot<br/>OTA process]
        Verify[Version Verification<br/>Post-update]
    end

    %% Source to HA
    GitHub -->|API poll| SourceMgr
    GitLab -->|API poll| SourceMgr
    Custom -->|HTTP poll| SourceMgr
    SourceMgr -->|New version| UpdateCoord
    UpdateCoord -->|Create/Update| UpdateEntity

    %% UI flow
    UpdateEntity -->|Display| Settings
    UpdateEntity -->|Show| Changelog
    Settings -->|User action| Controls

    %% Update delivery
    Controls -->|Trigger| MQTT_Notify
    MQTT_Notify -->|URL + checksum| Download
    Download -->|Binary| Flash
    Flash -->|Success| Verify
    Verify -->|Report version| UpdateCoord

    classDef sourceStyle fill:#E1F5FE,stroke:#0277BD,stroke-width:2px
    classDef haStyle fill:#C8E6C9,stroke:#388E3C,stroke-width:2px
    classDef uiStyle fill:#F3E5F5,stroke:#6A1B9A,stroke-width:2px
    classDef deviceStyle fill:#FFCCBC,stroke:#D84315,stroke-width:2px

    class GitHub,GitLab,Custom sourceStyle
    class SourceMgr,UpdateCoord,UpdateEntity haStyle
    class Settings,Changelog,Controls uiStyle
    class MQTT_Notify,Download,Flash,Verify deviceStyle
```

## Technology Stack Comparison

```mermaid
graph LR
    subgraph "Current Platform"
        direction TB
        C_FE[Angular + Ionic<br/>~8000 LOC]
        C_BE[Node.js + Express<br/>~10000 LOC]
        C_DB[MongoDB +<br/>InfluxDB]
        C_MQ[RabbitMQ<br/>MQTT Plugin]
        C_FW[Arduino<br/>~5000 LOC]
        C_Auth[Custom JWT Auth<br/>~1000 LOC]

        C_FE -.-> C_BE
        C_BE -.-> C_DB
        C_BE -.-> C_MQ
        C_BE -.-> C_Auth
        C_FW -.-> C_MQ
    end

    subgraph "Home Assistant Platform"
        direction TB
        H_FE[Lovelace + Cards<br/>~500 LOC YAML]
        H_BE[HA Core<br/>Built-in]
        H_DB[PostgreSQL +<br/>InfluxDB v2]
        H_MQ[Mosquitto<br/>Standard MQTT]
        H_FW[ESPHome + Arduino<br/>~100-2000 LOC]
        H_Auth[HA Auth<br/>Built-in]

        H_FE -.-> H_BE
        H_BE -.-> H_DB
        H_BE -.-> H_MQ
        H_BE -.-> H_Auth
        H_FW -.-> H_MQ
    end

    Current[Current:<br/>~24000 LOC] --> C_FE
    Proposed[Proposed:<br/>~7500 LOC] --> H_FE

    classDef currentStyle fill:#FFCDD2,stroke:#C62828,stroke-width:2px
    classDef proposedStyle fill:#C8E6C9,stroke:#2E7D32,stroke-width:2px

    class C_FE,C_BE,C_DB,C_MQ,C_FW,C_Auth,Current currentStyle
    class H_FE,H_BE,H_DB,H_MQ,H_FW,H_Auth,Proposed proposedStyle
```

## Migration Phases Timeline

```mermaid
gantt
    title Plantalytix to Home Assistant Migration Timeline
    dateFormat YYYY-MM-DD
    section Phase 1: POC
    Deploy HA test environment           :p1a, 2025-11-18, 2w
    MQTT discovery (1 device type)       :p1b, after p1a, 2w
    Prototype dashboards                 :p1c, after p1b, 1w
    Validate & go/no-go decision        :milestone, after p1c, 1d

    section Phase 2: Backend
    MQTT discovery (all devices)         :p2a, after p1c, 3w
    Migrate simple devices to ESPHome    :p2b, after p2a, 2w
    Historical data migration            :p2c, after p2a, 2w
    HA automations implementation        :p2d, after p2b, 2w

    section Phase 3: Frontend
    Production Lovelace dashboards       :p3a, after p2d, 2w
    Custom cards development             :p3b, after p3a, 1w
    Mobile app configuration             :p3c, after p3b, 1w

    section Phase 4: Deployment
    User installation guide              :p4a, after p3c, 1w
    Test installations                   :p4b, after p4a, 1w
    Documentation & support              :p4c, after p4b, 1w
    Release v1.0                        :milestone, after p4c, 1d
```

## Deployment Hardware Options

```mermaid
graph LR
    subgraph "Hardware Options"
        RaspPi[Raspberry Pi 4<br/>4GB RAM<br/>~$100]
        NUC[Intel NUC<br/>i3/8GB<br/>~$400]
        HABlue[HA Blue/Yellow<br/>Official<br/>~$150]
        Server[Existing Server<br/>Docker<br/>$0]
    end

    subgraph "Suitable For"
        Home[Home Users<br/>1-10 devices]
        Small[Small Facility<br/>10-30 devices]
        Medium[Medium Facility<br/>30-50 devices]
        Large[Large Facility<br/>50+ devices]
    end

    RaspPi -->|Best for| Home
    HABlue -->|Best for| Home
    RaspPi -->|Works for| Small
    NUC -->|Best for| Small
    NUC -->|Best for| Medium
    Server -->|Best for| Large

    classDef hwStyle fill:#FFE0B2,stroke:#E64A19,stroke-width:2px
    classDef useStyle fill:#C8E6C9,stroke:#388E3C,stroke-width:2px

    class RaspPi,NUC,HABlue,Server hwStyle
    class Home,Small,Medium,Large useStyle
```

---

## Legend

### Color Coding
- 🔵 **Blue** - User interfaces and clients
- 🟢 **Green** - Home Assistant components and services
- 🟡 **Yellow** - Network infrastructure
- 🔴 **Red/Orange** - IoT devices and firmware
- 🟣 **Purple** - Databases and storage

### Component Types
- **Rectangle** - Service/Application
- **Cylinder** - Database
- **Parallelogram** - User interface
- **Diamond** - Decision point

### Connection Types
- **Solid line** - Data flow
- **Dashed line** - Optional/configuration
- **Dotted line** - Management

---

## Key Differences from Multi-Tenant Architecture

### Removed Components
- ❌ Tenant orchestration
- ❌ Multi-tenant MQTT ACLs
- ❌ Per-tenant InfluxDB buckets
- ❌ Nginx reverse proxy (multiple domains)
- ❌ Custom user management
- ❌ JWT authentication system
- ❌ Device claim code system
- ❌ Tenant provisioning API

### Simplified Components
- ✅ Single HA instance (not per-tenant)
- ✅ Single PostgreSQL database
- ✅ Single InfluxDB bucket
- ✅ Simple MQTT broker (no ACLs needed)
- ✅ Direct device access (no ownership model)
- ✅ HA built-in auth (no custom auth)

---

## Related Documentation

- [Architecture Analysis](../ARCHITECTURE-ANALYSIS.md) - Current system deep-dive
- [Migration Summary](../MIGRATION-SUMMARY.md) - Executive summary
- [ADR Index](./decisions/README.md) - All architecture decisions
- [Getting Started Guide](./GETTING-STARTED-HA.md) - Hands-on setup
- [ADR-0006: Single-User Deployment](./decisions/0006-home-assistant-single-user-deployment.md) - Deployment strategy

---

**Generated**: 2025-11-15
**Version**: 2.0 (Updated for single-user deployment)
**Format**: Mermaid.js
**View online**: Use [Mermaid Live Editor](https://mermaid.live/) or GitHub/GitLab rendering
