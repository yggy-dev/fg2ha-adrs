# Plantalytix Architecture Analysis

## Document Information

- **Date**: 2025-11-15
- **Project**: Plantalytix IoT Platform
- **Purpose**: Comprehensive architectural analysis to inform Architecture Decision Records (ADRs)
- **Analysis Depth**: Deep codebase exploration including cloud services, firmware, and infrastructure

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [System Overview](#system-overview)
3. [Project Structure](#project-structure)
4. [Technology Stack](#technology-stack)
5. [Service Architecture](#service-architecture)
6. [Key Architectural Decisions](#key-architectural-decisions)
7. [Data Architecture](#data-architecture)
8. [Communication Patterns](#communication-patterns)
9. [Security Architecture](#security-architecture)
10. [Deployment Architecture](#deployment-architecture)
11. [Frontend Architecture](#frontend-architecture)
12. [Firmware Architecture](#firmware-architecture)
13. [Recommended ADRs](#recommended-adrs)

---

## Executive Summary

**Plantalytix** is an IoT platform for smart environmental control devices used in agriculture and horticulture, particularly indoor growing environments. The system consists of:

- **Cloud Platform**: TypeScript/Node.js backend with Angular/Ionic frontend
- **IoT Firmware**: ESP32-based devices running Arduino framework via PlatformIO
- **Hardware**: Custom PCB designs for climate controllers, fan controllers, light controllers, and smart sockets
- **Data Management**: Dual-database strategy using MongoDB for metadata and InfluxDB for time-series sensor data

**Architecture Style**: Microservices-based IoT platform with MQTT message broker, RESTful API, and containerized deployment.

**Primary Use Case**: Real-time monitoring and control of environmental conditions (temperature, humidity, CO2) in indoor growing facilities with remote management capabilities.

---

## System Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                              │
│  ┌───────────────────┐              ┌───────────────────┐       │
│  │   Web Application │              │   IoT Devices     │       │
│  │  (Angular/Ionic)  │              │    (ESP32)        │       │
│  │   Port: 8080      │              │   MQTT Client     │       │
│  └─────────┬─────────┘              └─────────┬─────────┘       │
└────────────┼──────────────────────────────────┼─────────────────┘
             │ HTTP/HTTPS                       │ MQTT/WiFi
             │                                  │
┌────────────┴──────────────────────────────────┴─────────────────┐
│                   Application Layer                              │
│  ┌────────────┐  ┌────────────┐  ┌──────────────────────┐      │
│  │   Nginx    │  │ RabbitMQ + │  │   Node.js API        │      │
│  │   Reverse  │  │   MQTT     │  │   Express/TypeScript │      │
│  │   Proxy    │  │  Plugin    │  │   Port: 8081         │      │
│  └────────────┘  └────────────┘  └──────────────────────┘      │
└──────────────────────────────────────┬──────────────────────────┘
                                       │
┌──────────────────────────────────────┴──────────────────────────┐
│                     Data Layer                                   │
│  ┌─────────────────────┐       ┌──────────────────────┐        │
│  │     MongoDB         │       │    InfluxDB v2       │        │
│  │  (Metadata/Config)  │       │  (Time-Series Data)  │        │
│  └─────────────────────┘       └──────────────────────┘        │
└──────────────────────────────────────────────────────────────────┘
```

### System Context

- **Multi-tenant SaaS Platform**: Users create accounts and claim devices
- **Real-time Monitoring**: 10-second polling interval for sensor data
- **Remote Control**: Configure and test device outputs remotely
- **OTA Updates**: Over-the-air firmware updates managed from cloud
- **Alternative Deployment**: Local/edge mode with Mosquitto, Node-RED, Grafana

---

## Project Structure

### Multi-Repository Organization

The codebase is organized as **four separate Git repositories** within a single parent directory:

```
/home/eo/plantalytix/
├── cloud/              [Git Repo] - Main cloud platform
│   ├── server/         - Node.js/TypeScript backend
│   ├── webapp/         - Angular/Ionic frontend
│   ├── firmware/       - ESP32 PlatformIO firmware
│   ├── rabbitmq/       - RabbitMQ MQTT broker config
│   ├── config/         - Infrastructure configuration
│   └── fw-buildcontainer/ - Docker firmware builder
│
├── hardware/           [Git Repo] - KiCAD PCB designs
│   ├── plantalytix-programmer/   - USB-C programmer
│   ├── plantalytix-fridgegrow2/  - Climate controller
│   ├── plantalytix-fan/          - Fan controller
│   ├── plantalytix-light/        - Light controller
│   └── plantalytix-socket/       - Smart socket
│
├── fridgegrow2.0/      [Git Repo] - Local deployment stack
│   └── docker-compose.yaml       - Mosquitto/InfluxDB/Grafana/Node-RED
│
└── test/               [Git Repo] - Test environment
    └── docker-compose.yaml       - Test stack
```

**Key Insight**: The multi-repo structure allows independent versioning of software (cloud), hardware designs, and deployment configurations.

---

## Technology Stack

### Backend (Server)

| Component | Technology | Version | Purpose |
|-----------|-----------|---------|---------|
| Runtime | Node.js | 14.x | JavaScript runtime |
| Language | TypeScript | ^4.9.5 | Type-safe development |
| Framework | Express.js | ^4.18.2 | HTTP server & routing |
| Database | MongoDB | Latest | Metadata & configuration storage |
| Time-Series DB | InfluxDB | v2 | Sensor data storage |
| Message Broker | RabbitMQ | Latest | MQTT broker with HTTP auth |
| ORM | Mongoose | ^7.0.1 | MongoDB object modeling |
| Authentication | JWT | jsonwebtoken ^9.0.0 | Stateless authentication |
| Validation | Joi | ^17.8.3 | Schema validation |
| Logging | Winston | ^3.8.2 | Structured logging |
| Email | Nodemailer | ^6.9.1 | Account activation emails |
| HTTP Client | Axios | ^1.3.4 | Internal HTTP requests |
| Build Tool | SWC | @swc/cli ^0.1.62 | Fast TypeScript compilation |

**Notable Dependencies**:
- `@influxdata/influxdb-client` - InfluxDB v2 API
- `mqtt` - MQTT client for message processing
- `bcrypt` - Password hashing
- `helmet` - Security headers
- `hpp` - HTTP Parameter Pollution protection
- `cors` - Cross-Origin Resource Sharing

### Frontend (Web Application)

| Component | Technology | Version | Purpose |
|-----------|-----------|---------|---------|
| Framework | Angular | 15.2.10 | SPA framework |
| UI Library | Ionic | 6.20.0 | Hybrid mobile/web UI |
| Language | TypeScript | ~4.9.4 | Type-safe development |
| Charts | Highcharts | ^10.3.3 | Time-series visualization |
| Charts | ng2-charts | ^4.1.1 | Chart.js wrapper |
| HTTP | Angular HttpClient | 15.2.10 | REST API communication |
| Routing | Angular Router | 15.2.10 | SPA navigation |
| i18n | ngx-translate | ^14.0.0 | Internationalization |
| Build Tool | Angular CLI | 15.2.10 | Build & development |
| Mobile Runtime | Capacitor | 4.7.3 | Native mobile wrapper |

### IoT Firmware

| Component | Technology | Version | Purpose |
|-----------|-----------|---------|---------|
| Platform | ESP32 | Various | Microcontroller |
| Framework | Arduino | Latest | Firmware framework |
| Build System | PlatformIO | 6.1.11 | Cross-platform build |
| MQTT | PubSubClient | ^2.8 | MQTT client |
| JSON | ArduinoJson | ^6.21.3 | JSON serialization |
| Sensors | Sensirion I2C | Latest | Temperature/humidity/CO2 |
| Display | Adafruit GFX | Latest | OLED display |
| RTC | MCP7940 | Latest | Real-time clock |

**Supported Hardware**:
- Heltec WiFi LoRa 32 V2
- ESP32-CAM
- Generic ESP32 modules

### Infrastructure

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Containerization | Docker | Service isolation |
| Orchestration | Docker Compose | Multi-container deployment |
| Web Server | Nginx | Reverse proxy & static hosting |
| Database | MongoDB 7.0 | NoSQL document store |
| Time-Series DB | InfluxDB 2.0 | Optimized time-series storage |
| Message Broker | RabbitMQ 3-management | MQTT + AMQP broker |
| DB Admin | Mongo-Express | Optional DB UI |

---

## Service Architecture

### Microservices Components

#### 1. MongoDB Service

**Purpose**: Persistent storage for application metadata

**Data Stored**:
- User accounts and credentials
- Device registry and configuration
- Device logs
- Firmware versions and metadata
- Device classes/types
- Claim codes for device pairing
- Password reset tokens

**Configuration**:
- Authentication enabled
- Admin credentials via environment variables
- Persistent volume: `./data/mongodb`
- Internal access only (no external port)

#### 2. InfluxDB v2 Service

**Purpose**: Time-series database for sensor measurements

**Data Stored**:
- Temperature, humidity, CO2 readings
- Output states (heater, dehumidifier, fan, light, relay)
- PID controller values (avg, p, i, d)
- RPM and power measurements
- Day/night mode indicators

**Configuration**:
- HTTP API on port 8086
- Admin token authentication
- Organization: plantalytix
- Bucket: plantalytix
- Persistent volume: `./data/influxdb`

**Retention**: Configurable per bucket

#### 3. RabbitMQ + MQTT Plugin Service

**Purpose**: Message broker for device communication

**Features**:
- MQTT protocol support on port 1883
- HTTP authentication backend
- Management UI on port 15672
- Topic-based access control
- WebSTOMP support (port 15674)

**Authentication Flow**:
```
Device/Client → RabbitMQ MQTT
                    ↓
            HTTP Auth Backend
                    ↓
    POST /mqttauth/user (validate credentials)
    POST /mqttauth/vhost (validate access)
    POST /mqttauth/topic (validate topic access)
                    ↓
            Node.js API Server
                    ↓
            MongoDB (device credentials)
```

**Configuration**:
- Custom `rabbitmq.conf` with HTTP auth backend
- Topic exchange for flexible routing
- Internal server credentials (UUID-based)

#### 4. Node.js API Server

**Purpose**: Core business logic and orchestration

**Responsibilities**:
- RESTful API for web application
- MQTT client (subscribes to `/devices/#`)
- Device message processing
- Data persistence to MongoDB/InfluxDB
- User authentication and authorization
- Firmware OTA update orchestration
- Email notifications
- MQTT authentication for RabbitMQ

**Architecture Pattern**: MVC with Service Layer

```
HTTP Request → Routes → Middlewares → Controllers → Services → Models
                            ↓
                    (Auth, Validation, Error)
```

**Key Services**:
- `AuthService` - JWT token management
- `DeviceService` - Device lifecycle management
- `DataService` - Time-series data queries
- `FirmwareService` - OTA update orchestration
- `MqttService` - MQTT client and message processing

**Configuration**:
- Port 8081 (internal)
- MQTT client to RabbitMQ
- MongoDB and InfluxDB clients
- Environment-based configuration

#### 5. Nginx Service

**Purpose**: Reverse proxy and static file serving

**Configuration**:
- Port 8080 (external)
- Serves Angular SPA static files
- Proxies API requests to Node.js server
- Multi-stage Docker build (Node.js builder + Nginx runtime)

**Routes**:
- `/` → Static Angular application
- API requests proxied to backend (via Angular `proxy.conf.json`)

#### 6. Mongo-Express (Optional)

**Purpose**: Database administration UI

**Configuration**:
- Port 8088
- Read-only recommended for production
- Admin credentials via environment variables

### Service Communication

```
Web App (Angular)
    │
    ├─→ HTTP:8080 → Nginx → Static Files
    └─→ HTTP API → Node.js:8081
                        │
                        ├─→ MongoDB (metadata CRUD)
                        ├─→ InfluxDB (time-series queries)
                        └─→ RabbitMQ (MQTT publish)

IoT Device (ESP32)
    │
    └─→ MQTT:1883 → RabbitMQ
                        │
                        ├─→ HTTP Auth → Node.js:8081/mqttauth/*
                        └─→ MQTT Subscribe → Node.js MQTT Client
                                                │
                                                ├─→ MongoDB (logs, status)
                                                └─→ InfluxDB (measurements)
```

---

## Key Architectural Decisions

Below are the major architectural decisions identified in the codebase. Each represents a candidate for a formal ADR.

### ADR-001: Dual Database Strategy (MongoDB + InfluxDB)

**Decision**: Use MongoDB for metadata/configuration and InfluxDB for time-series sensor data.

**Context**:
- IoT devices generate high-frequency sensor readings (every 10 seconds)
- Configuration and user data are document-oriented
- Time-series data requires efficient querying and aggregation
- Different access patterns for metadata vs. measurements

**Rationale**:
- **MongoDB**: Excellent for flexible schemas, relationships, user management
- **InfluxDB**: Optimized for time-series data with downsampling, retention policies
- **Separation of concerns**: Configuration changes don't impact measurement storage
- **Performance**: Each database optimized for its specific use case

**Consequences**:
- ✅ Optimal performance for both metadata and time-series queries
- ✅ Independent scaling of metadata and measurement storage
- ✅ InfluxDB Flux language for powerful time-series analytics
- ❌ Increased operational complexity (two databases to manage)
- ❌ No ACID transactions across both databases
- ❌ Additional learning curve for team (Flux query language)

**Alternatives Considered**:
1. MongoDB only with time-series collections (MongoDB 5.0+)
2. PostgreSQL with TimescaleDB extension
3. Single InfluxDB for all data

---

### ADR-002: RabbitMQ with MQTT Plugin vs. Standalone MQTT Broker

**Decision**: Use RabbitMQ with MQTT plugin instead of dedicated MQTT broker (Mosquitto).

**Context**:
- Need MQTT protocol for IoT device communication
- Require custom authentication against user database
- Future extensibility for message routing and processing

**Rationale**:
- **HTTP Auth Backend**: RabbitMQ supports HTTP-based authentication plugin
- **Flexible Routing**: Topic exchanges allow complex message routing
- **AMQP Support**: Future ability to use AMQP for internal services
- **Management UI**: Built-in monitoring and management interface
- **Enterprise Features**: Dead letter queues, message persistence, clustering

**Consequences**:
- ✅ Custom authentication via API server HTTP endpoint
- ✅ Fine-grained topic-based access control
- ✅ Management UI for debugging and monitoring
- ✅ Potential for future AMQP-based microservices
- ❌ Higher memory footprint than Mosquitto
- ❌ More complex configuration
- ❌ Slightly higher latency than pure MQTT broker

**Alternatives Considered**:
1. Mosquitto with custom auth plugin (C/C++)
2. HiveMQ (commercial)
3. EMQX (Erlang-based)

---

### ADR-003: MQTT Client Embedded in API Server

**Decision**: Run MQTT client inside the Node.js API server process.

**Context**:
- Need to process MQTT messages from devices
- Must persist data to MongoDB and InfluxDB
- Require access to device metadata for message validation

**Rationale**:
- **Simplicity**: Single service handles HTTP API and MQTT processing
- **Shared Context**: Access to database clients and business logic
- **Authentication**: Can validate device credentials directly
- **No Additional Service**: Reduces deployment complexity

**Consequences**:
- ✅ Simpler deployment (one less service)
- ✅ Shared database connections and models
- ✅ Direct access to authentication logic
- ✅ Lower latency for message processing
- ❌ Single point of failure (API and MQTT coupled)
- ❌ Difficult to scale horizontally (MQTT client state)
- ❌ Memory usage for MQTT client in API process
- ❌ Blocking MQTT processing could impact HTTP API

**Alternatives Considered**:
1. Separate MQTT consumer service
2. Serverless functions triggered by MQTT messages
3. Apache Kafka for message buffering

**Recommendation**: Consider extracting MQTT processing to separate service for production scaling.

---

### ADR-004: JWT-Based Stateless Authentication

**Decision**: Use JWT (JSON Web Tokens) for user authentication.

**Context**:
- Need authentication for web application and API
- Mobile app support planned (Ionic/Capacitor)
- Microservices architecture requires stateless auth

**Rationale**:
- **Stateless**: No session storage required in API server
- **Scalable**: Multiple API instances without session sharing
- **Mobile-Friendly**: Token storage in mobile apps
- **Expiration**: Built-in token expiration and refresh
- **Claims**: Embed user_id and is_admin in token

**Consequences**:
- ✅ Horizontal scaling without session store
- ✅ Mobile and web apps use same auth mechanism
- ✅ No database lookup for authentication (only authorization)
- ✅ Refresh token support for long-lived sessions
- ❌ Token revocation requires additional infrastructure
- ❌ Tokens can't be invalidated before expiration
- ❌ Larger request size (token in every request)

**Alternatives Considered**:
1. Session-based authentication with Redis
2. OAuth2 with third-party providers
3. API keys for devices

---

### ADR-005: Device Claim Code Pairing System

**Decision**: Use claim codes to pair devices with user accounts.

**Context**:
- Devices manufactured without user assignment
- Need secure device-to-user pairing
- Prevent unauthorized device access

**Rationale**:
- **Security**: Device generates unique claim code
- **User Experience**: Simple code entry in web app
- **Temporary**: Claim codes can expire
- **Automatic**: Device self-registration via API

**Flow**:
```
1. Device boots → POST /device/register (gets device_id, credentials)
2. Device requests → POST /device/claimcode (gets claim_code)
3. User enters claim code → POST /device (claim device)
4. Device now owned by user
```

**Consequences**:
- ✅ Secure pairing without pre-configuration
- ✅ Simple user experience (6-digit code)
- ✅ Prevents unauthorized device access
- ✅ Supports self-registration for automated provisioning
- ❌ Requires device to have display or output mechanism
- ❌ Claim codes could be brute-forced (mitigated by expiration)
- ❌ Additional database collection to manage

**Alternatives Considered**:
1. QR code scanning
2. Bluetooth pairing
3. Pre-configured device credentials

---

### ADR-006: Firmware OTA Update Strategy

**Decision**: Implement over-the-air firmware updates with class-based rollout and concurrency limiting.

**Context**:
- Devices deployed in remote locations
- Firmware bugs and feature updates required
- Need controlled rollout to prevent mass failures

**Rationale**:
- **Device Classes**: Group devices by type (fridge, fan, light, plug)
- **Concurrency Limiting**: Max concurrent updates per class
- **Failure Tracking**: Max failures before rollout pause
- **Automatic Rollout**: Devices poll for updates on status publish
- **Timeout Retry**: 10-minute timeout for failed updates

**Implementation**:
- Device publishes status → API checks `pending_firmware` vs. `current_firmware`
- If update pending → Publish firmware URL to MQTT
- Device downloads and flashes firmware
- Device publishes new version on next boot
- API marks update complete or failed

**Consequences**:
- ✅ Remote firmware updates without physical access
- ✅ Controlled rollout prevents mass failures
- ✅ Automatic retry for failed updates
- ✅ Class-based targeting (update all fridge controllers)
- ❌ Devices must have internet connection
- ❌ Bricked devices require manual recovery
- ❌ No rollback mechanism (must flash new firmware)
- ❌ Update status only known when device reconnects

**Alternatives Considered**:
1. Manual firmware upload via USB
2. Local network updates (mDNS discovery)
3. Staged rollout with canary deployments

---

### ADR-007: Hardware Abstraction Layer in Firmware

**Decision**: Use PlatformIO build environments with hardware-specific source directories.

**Context**:
- Multiple device types (fridge, fan, light, plug, dryer, cam)
- Shared core functionality (WiFi, MQTT, UI)
- Hardware-specific peripherals and control logic

**Rationale**:
- **Common Core**: Shared code in `/src/` directory
- **Hardware-Specific**: Per-type code in `/src_hwtype/` directories
- **Build Environments**: PlatformIO env for each device type
- **Compile-Time Selection**: Hardware type via build flags

**Structure**:
```
firmware/
├── src/                    # Common core
│   ├── main.cpp
│   ├── fridgecloud.cpp     # MQTT communication
│   ├── wifi.cpp
│   ├── ui.cpp
│   └── settings.cpp
├── src_hwtype/             # Hardware-specific
│   ├── fridge/
│   │   ├── automation.h    # Climate control logic
│   │   ├── hardware.h
│   │   └── sensors.h
│   ├── fan/
│   ├── light/
│   └── plug/
└── platformio.ini          # Build environments
```

**Consequences**:
- ✅ Code reuse across device types
- ✅ Single repository for all firmware
- ✅ Hardware-specific optimizations
- ✅ Easier testing (dummy hardware type)
- ❌ Requires careful abstraction design
- ❌ Build time increases (multiple variants)
- ❌ Larger repository size

**Alternatives Considered**:
1. Separate repositories per device type
2. Runtime hardware detection
3. Preprocessor macros only

---

### ADR-008: Angular/Ionic Hybrid Application

**Decision**: Use Angular with Ionic framework for web and mobile application.

**Context**:
- Need web application for desktop users
- Mobile app desired for on-the-go monitoring
- Limited resources for separate iOS/Android development

**Rationale**:
- **Single Codebase**: Web and mobile from same code
- **Ionic UI**: Mobile-first components that work on web
- **Capacitor**: Native mobile builds (iOS/Android)
- **Angular Ecosystem**: TypeScript, RxJS, dependency injection

**Consequences**:
- ✅ Reduced development effort (one codebase)
- ✅ Consistent UX across platforms
- ✅ Native mobile features via Capacitor plugins
- ✅ Web-based development and debugging
- ❌ Larger bundle size than native apps
- ❌ Performance not as good as native
- ❌ Platform-specific bugs require Capacitor workarounds
- ❌ "Hybrid" feel may impact UX

**Alternatives Considered**:
1. React Native
2. Flutter
3. Separate native iOS/Android apps
4. Progressive Web App (PWA) only

---

### ADR-009: Polling-Based Data Updates

**Decision**: Frontend polls API every 10 seconds for latest sensor data.

**Context**:
- Need real-time device monitoring
- Display current temperature, humidity, CO2
- Update charts with latest data

**Implementation**:
```typescript
// In Angular component
ngOnInit() {
  this.updateInterval = setInterval(() => {
    this.loadLatestData();
  }, 10000); // 10 seconds
}
```

**Consequences**:
- ✅ Simple implementation (no WebSocket infrastructure)
- ✅ Works with standard HTTP load balancers
- ✅ No persistent connections to manage
- ✅ Automatic reconnection on network issues
- ❌ 10-second latency for data updates
- ❌ Unnecessary requests when data hasn't changed
- ❌ Higher server load (polling vs. push)
- ❌ Mobile battery drain

**Alternatives Considered**:
1. WebSocket connection for real-time push
2. Server-Sent Events (SSE)
3. Long polling
4. GraphQL subscriptions

**Recommendation**: Consider WebSocket or SSE for real-time updates in future.

---

### ADR-010: Multi-Repository Structure

**Decision**: Organize project as four separate Git repositories under single parent directory.

**Context**:
- Cloud software (backend, frontend, firmware)
- Hardware designs (KiCAD PCBs)
- Alternative deployment configurations
- Different release cycles and versioning

**Repositories**:
1. **cloud/** - Main cloud platform (software)
2. **hardware/** - KiCAD PCB designs
3. **fridgegrow2.0/** - Local deployment stack
4. **test/** - Test environment

**Consequences**:
- ✅ Independent versioning (software vs. hardware)
- ✅ Smaller repository sizes
- ✅ Separate access control per repo
- ✅ Hardware changes don't pollute software history
- ❌ No atomic commits across repositories
- ❌ Dependency management between repos
- ❌ Requires manual synchronization
- ❌ More complex CI/CD pipelines

**Alternatives Considered**:
1. Monorepo with all components
2. Git submodules
3. Separate repositories without shared parent

---

### ADR-011: TypeScript for Backend Development

**Decision**: Use TypeScript instead of JavaScript for Node.js backend.

**Context**:
- Backend handles complex business logic
- Multiple developers working on codebase
- Need type safety for API contracts and database models

**Rationale**:
- **Type Safety**: Catch errors at compile time
- **IDE Support**: Better autocomplete and refactoring
- **Documentation**: Types serve as inline documentation
- **Maintainability**: Easier refactoring with confidence
- **Modern Features**: Async/await, decorators, etc.

**Build Process**:
- Development: `ts-node-dev` with watch mode
- Production: SWC compilation (faster than tsc)

**Consequences**:
- ✅ Fewer runtime type errors
- ✅ Better developer experience
- ✅ Easier onboarding for new developers
- ✅ Refactoring safety
- ❌ Compilation step required
- ❌ Additional tooling complexity
- ❌ Slightly slower development iteration (though mitigated by ts-node-dev)

**Alternatives Considered**:
1. Plain JavaScript (ES6+)
2. Flow type checker
3. JSDoc for type annotations

---

### ADR-012: Express.js MVC Architecture

**Decision**: Implement MVC pattern with additional service layer.

**Architecture**:
```
Routes → Middlewares → Controllers → Services → Models → Database
```

**Responsibilities**:
- **Routes**: URL mapping and parameter extraction
- **Middlewares**: Cross-cutting concerns (auth, validation, logging)
- **Controllers**: HTTP request/response handling
- **Services**: Business logic and orchestration
- **Models**: Data validation and database interaction

**Example Flow**:
```typescript
// Route
router.get('/device/:device_id',
  authMiddleware,
  deviceController.getDevice
);

// Controller
async getDevice(req, res, next) {
  const device = await deviceService.getDeviceById(req.params.device_id);
  res.json(device);
}

// Service
async getDeviceById(deviceId: string) {
  return Device.findOne({ device_id: deviceId });
}

// Model (Mongoose)
const deviceSchema = new Schema({ device_id: String, ... });
```

**Consequences**:
- ✅ Clear separation of concerns
- ✅ Testable business logic (services)
- ✅ Reusable services across controllers
- ✅ Easier to understand for new developers
- ❌ More files and boilerplate
- ❌ Potential over-engineering for simple CRUD

**Alternatives Considered**:
1. Thin controllers with direct model access
2. Domain-Driven Design (DDD) with repositories
3. Functional programming approach

---

## Data Architecture

### MongoDB Schema Design

#### Users Collection

```typescript
{
  username: String,           // Email address (unique index)
  password: String,           // bcrypt hashed
  user_id: String,            // UUID (unique index)
  is_admin: Boolean,          // Admin flag
  is_active: Boolean,         // Account activation status
  activation_code: String,    // Email activation code (optional)
  createdAt: Date,           // Timestamp
  updatedAt: Date            // Timestamp
}
```

**Indexes**: `username`, `user_id`

#### Devices Collection

```typescript
{
  device_id: String,          // UUID (unique index)
  username: String,           // MQTT username (unique index)
  password: String,           // MQTT password (plain text for MQTT auth)
  class_id: String,           // Reference to DeviceClass
  device_type: String,        // fridge | fan | light | plug | dryer | cam
  client_id: String,          // MQTT client ID
  owner_id: String,           // Reference to User.user_id (null if unclaimed)
  configuration: String,      // JSON stringified configuration
  serialnumber: Number,       // Auto-increment serial
  name: String,               // User-defined name
  lastseen: Number,           // Unix timestamp (updated on MQTT publish)
  current_firmware: String,   // firmware_id
  pending_firmware: String,   // firmware_id (null if no update pending)
  fwupdate_start: Number,     // Unix timestamp
  fwupdate_end: Number,       // Unix timestamp
  createdAt: Date,
  updatedAt: Date
}
```

**Indexes**: `device_id`, `username`, `owner_id`, `class_id`, `serialnumber`

**Device Configuration Structure** (JSON string):
```json
{
  "type": "fridge",
  "target_temp": 20.0,
  "target_humidity": 60.0,
  "target_co2": 1000,
  "automation_mode": "pid",
  "pid_p": 1.0,
  "pid_i": 0.5,
  "pid_d": 0.1,
  "day_start": "06:00",
  "night_start": "22:00",
  "light_day": 100,
  "light_night": 0,
  // ... device-specific settings
}
```

#### DeviceClass Collection

```typescript
{
  class_id: String,           // Unique identifier
  name: String,               // fridge | fan | light | plug
  description: String,        // Human-readable description
  firmware_id: String,        // Current firmware for this class
  concurrent: Number,         // Max concurrent OTA updates
  maxfails: Number,           // Max failed updates before pause
  createdAt: Date,
  updatedAt: Date
}
```

**Purpose**: Group devices for firmware management and configuration templates.

#### DeviceFirmware Collection

```typescript
{
  firmware_id: String,        // Unique identifier
  name: String,               // Human-readable name
  version: String,            // Semantic version (e.g., "1.0.0")
  binaries: [String],         // Array of firmware_id references (variants)
  createdAt: Date,
  updatedAt: Date
}
```

**File Storage**: Firmware binaries stored in `/firmware/` directory, referenced by `firmware_id`.

#### DeviceLog Collection

```typescript
{
  device_id: String,          // Reference to Device
  message: String,            // Log message
  severity: Number,           // 0=debug, 1=info, 2=warning, 3=error
  time: Date,                 // Timestamp
}
```

**Indexes**: `device_id`, `time`

**Usage**: Device debug logs accessible via API for troubleshooting.

#### ClaimCode Collection

```typescript
{
  claim_code: String,         // 6-digit code (unique index)
  device_id: String,          // Reference to Device
  created_at: Date,           // Timestamp
}
```

**TTL**: Codes can be configured to expire after N minutes.

#### PasswordToken Collection

```typescript
{
  user_id: String,            // Reference to User
  token: String,              // Reset token (unique)
  created_at: Date,           // Timestamp
}
```

**TTL**: Tokens expire after 1 hour.

### InfluxDB Schema Design

**Organization**: plantalytix
**Bucket**: plantalytix
**Measurement**: status

#### Tags (Indexed)

- `device_id` (String, UUID)
- `user_id` (String, UUID)

#### Fields (Not Indexed)

**Sensor Readings**:
- `temperature` (Float) - Degrees Celsius
- `humidity` (Float) - Relative humidity %
- `co2` (Float) - CO2 ppm
- `sensor_type` (String) - Sensor model identifier

**Control Outputs** (0-100%):
- `out_heater` (Float)
- `out_dehumidifier` (Float)
- `out_co2` (Float)
- `out_light` (Float)
- `out_fan` (Float)
- `out_relais` (Float)

**PID Controller Values**:
- `avg` (Float) - Process variable
- `p` (Float) - Proportional term
- `i` (Float) - Integral term
- `d` (Float) - Derivative term

**Additional Metrics**:
- `rpm` (Float) - Fan speed
- `day` (Float) - Day/night mode (1/0)

**Timestamp**: High-precision Unix timestamp (nanoseconds)

#### Sample Query (Flux)

```flux
from(bucket: "plantalytix")
  |> range(start: -1h)
  |> filter(fn: (r) => r["_measurement"] == "status")
  |> filter(fn: (r) => r["device_id"] == "device-uuid")
  |> filter(fn: (r) => r["_field"] == "temperature")
  |> aggregateWindow(every: 1m, fn: mean)
```

---

## Communication Patterns

### Device-to-Cloud Message Flow

#### MQTT Topic Structure

```
/devices/{device_id}/status          - Device publishes current state
/devices/{device_id}/bulk            - Device publishes batch data
/devices/{device_id}/configuration   - Device receives configuration
/devices/{device_id}/firmware        - Device receives firmware update URL
/devices/{device_id}/log             - Device publishes log messages
/devices/{device_id}/test            - Device receives test commands
```

#### Device Publishes (QoS 0)

**Status Message** (Every 10 seconds or on change):
```json
{
  "device_id": "uuid",
  "temperature": 20.5,
  "humidity": 60.2,
  "co2": 1000,
  "out_heater": 50.0,
  "out_dehumidifier": 0,
  "out_fan": 75.0,
  "out_light": 100.0,
  "rpm": 1200,
  "day": 1,
  "firmware_version": "1.0.0"
}
```

**Bulk Data Message** (Batched measurements):
```json
{
  "device_id": "uuid",
  "measurements": [
    {"time": 1638360000, "temp": 20.5, "hum": 60.2},
    {"time": 1638360010, "temp": 20.6, "hum": 60.1},
    ...
  ]
}
```

**Log Message**:
```json
{
  "device_id": "uuid",
  "message": "WiFi connected",
  "severity": 1,
  "timestamp": 1638360000
}
```

#### Device Subscribes (QoS 1)

**Configuration** (Receives on change):
```json
{
  "target_temp": 20.0,
  "target_humidity": 60.0,
  "automation_mode": "pid",
  "pid_p": 1.0,
  ...
}
```

**Firmware Update** (Receives when update pending):
```json
{
  "firmware_url": "http://api:8081/device/firmware/{firmware_id}/{binary}",
  "version": "1.0.1",
  "md5": "checksum"
}
```

**Test Command** (Receives during test mode):
```json
{
  "out_heater": 50,
  "out_fan": 100,
  "duration": 300
}
```

### API Request/Response Patterns

#### Authentication Flow

```
POST /login
Content-Type: application/json

{
  "username": "user@example.com",
  "password": "password123"
}

Response 200 OK:
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "user_id": "uuid",
    "username": "user@example.com",
    "is_admin": false
  }
}
```

#### Device Claiming Flow

```
1. Device Registration (Device):
POST /device/register
Authorization: Bearer {AUTOMATION_TOKEN}
{
  "device_type": "fridge",
  "mac_address": "AA:BB:CC:DD:EE:FF"
}

Response:
{
  "device_id": "uuid",
  "username": "device-unique-username",
  "password": "mqtt-password"
}

2. Claim Code Request (Device):
POST /device/claimcode
Authorization: Basic {device credentials}
{
  "device_id": "uuid"
}

Response:
{
  "claim_code": "123456"
}

3. Device Claiming (User via Web App):
POST /device
Authorization: Bearer {user JWT}
{
  "claim_code": "123456"
}

Response:
{
  "device_id": "uuid",
  "name": "My Device",
  "device_type": "fridge",
  "configuration": { ... }
}
```

#### Time-Series Data Query

```
GET /data/series/{device_id}/temperature?from=-1h&to=now()&interval=1m
Authorization: Bearer {user JWT}

Response:
{
  "data": [
    {"time": "2025-11-15T10:00:00Z", "value": 20.5},
    {"time": "2025-11-15T10:01:00Z", "value": 20.6},
    ...
  ]
}
```

### MQTT Authentication Flow

```
1. Device connects to RabbitMQ MQTT (port 1883)
   Client ID: device-uuid
   Username: device-unique-username
   Password: mqtt-password

2. RabbitMQ HTTP Auth Plugin:
   POST http://api:8081/mqttauth/user
   username=device-unique-username&password=mqtt-password

3. API Server validates:
   - Query MongoDB for device credentials
   - Compare password (plain text)
   - Return "allow" or "deny"

4. RabbitMQ HTTP Auth Plugin:
   POST http://api:8081/mqttauth/vhost
   username=device-unique-username&vhost=/

5. API Server: Return "allow"

6. RabbitMQ HTTP Auth Plugin (Topic Access):
   POST http://api:8081/mqttauth/topic
   username=device-unique-username
   &vhost=/
   &resource=topic
   &name=/devices/{device_id}/status
   &permission=write

7. API Server validates:
   - Ensure username matches device_id
   - Devices can only access their own topics
   - Return "allow" or "deny"
```

---

## Security Architecture

### Authentication Mechanisms

#### 1. User Authentication (JWT)

**Token Generation**:
```typescript
// At login
const accessToken = jwt.sign(
  { user_id, is_admin },
  process.env.JWT_SECRET,
  { expiresIn: '15m' }
);

const refreshToken = jwt.sign(
  { user_id },
  process.env.JWT_REFRESH_SECRET,
  { expiresIn: '7d' }
);
```

**Token Verification** (Middleware):
```typescript
const authMiddleware = (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];
  const decoded = jwt.verify(token, process.env.JWT_SECRET);
  req.user_id = decoded.user_id;
  req.is_admin = decoded.is_admin;
  next();
};
```

#### 2. Device Authentication (MQTT)

- **Unique Credentials**: Each device has unique MQTT username/password
- **Stored in MongoDB**: Plain text (required for RabbitMQ auth)
- **Generated at Registration**: UUID-based username, random password
- **No Token Expiration**: Long-lived credentials

#### 3. Internal Service Authentication

- **RabbitMQ → API**: HTTP Basic Auth with UUID credentials
- **API → MongoDB**: Username/password from environment
- **API → InfluxDB**: Admin token from environment

### Authorization Model

#### Role-Based Access Control (RBAC)

**Roles**:
1. **Unauthenticated** - Login, signup, password reset
2. **Authenticated User** - Own devices only
3. **Admin** - All devices, firmware, classes

**Middleware Chain**:
```typescript
// User must be authenticated
router.get('/device', authMiddleware, deviceController.getDevices);

// User must be admin
router.get('/device/all', authMiddleware, isAdminMiddleware, ...);
```

**Resource Authorization**:
```typescript
// Verify device ownership
const device = await Device.findOne({ device_id, owner_id: req.user_id });
if (!device && !req.is_admin) {
  return res.status(403).json({ message: 'Forbidden' });
}
```

#### MQTT Topic Authorization

**Rules**:
- Devices can only publish to `/devices/{their_device_id}/*`
- Devices can only subscribe to `/devices/{their_device_id}/*`
- API server (internal client) can subscribe to `/devices/#`

**Enforcement**: RabbitMQ HTTP auth plugin validates topic access

### Security Measures

#### 1. Password Security

- **Hashing**: bcrypt with 10 rounds
- **Password Reset**: Time-limited tokens (1 hour)
- **Email Activation**: Optional account activation

#### 2. API Security

- **Helmet.js**: Security headers (XSS, CSP, etc.)
- **CORS**: Configurable allowed origins
- **HPP**: HTTP Parameter Pollution protection
- **Rate Limiting**: Not implemented (recommendation)
- **Input Validation**: Joi schemas for all endpoints

#### 3. MQTT Security

- **Authentication Required**: No anonymous access
- **Topic ACLs**: Strict topic access control
- **TLS/SSL**: Not configured (recommendation for production)

#### 4. Database Security

- **MongoDB**: Authentication enabled
- **InfluxDB**: Token-based authentication
- **No External Access**: Databases not exposed externally

### Security Gaps & Recommendations

1. **MQTT over TLS**: Currently using plain MQTT (port 1883)
   - **Risk**: Credentials and data in plain text
   - **Recommendation**: Enable TLS on port 8883

2. **Device Password Storage**: Plain text in MongoDB
   - **Risk**: Database compromise exposes MQTT passwords
   - **Recommendation**: Hash passwords with bcrypt (requires custom RabbitMQ auth)

3. **Rate Limiting**: Not implemented
   - **Risk**: Brute force attacks, DoS
   - **Recommendation**: Add express-rate-limit

4. **API HTTPS**: Not configured in docker-compose
   - **Risk**: Credentials transmitted in plain text
   - **Recommendation**: Terminate TLS at Nginx

5. **Secret Management**: Secrets in `.env` file
   - **Risk**: Accidental commit to Git
   - **Recommendation**: Use secrets management (Vault, AWS Secrets Manager)

6. **CSRF Protection**: Not implemented
   - **Risk**: Cross-Site Request Forgery attacks
   - **Recommendation**: Add CSRF tokens for state-changing operations

---

## Deployment Architecture

### Docker Compose Stack

**Network**: Bridge network named `main`

**Services**:
1. MongoDB (internal only)
2. InfluxDB (port 8086)
3. RabbitMQ (ports 1883, 5672, 15672, 15674)
4. API Server (port 8081)
5. Nginx (port 8080)
6. Mongo-Express (port 8088, optional)

**Persistent Volumes**:
- `./data/mongodb:/data/db`
- `./data/influxdb:/var/lib/influxdb2`
- `./firmware:/firmware`

### Environment Configuration

**.env File** (Required):
```bash
# Database
MONGO_USER=admin
MONGO_PASS=password
INFLUX_ADMIN_TOKEN=token
INFLUX_URL=http://influxdb:8086

# MQTT
MQTT_SERVER=rabbitmq
MQTT_PORT=1883
MQTT_INT_USER=internal-uuid
MQTT_INT_PASS=internal-password

# API
API_URL_EXT=http://api.example.com
API_URL_INT=http://server:8081
JWT_SECRET=secret
JWT_REFRESH_SECRET=refresh-secret

# Email (Optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=user@gmail.com
SMTP_PASS=app-password

# Admin User
ADMIN_USERNAME=admin@example.com
ADMIN_PASSWORD=admin123

# Features
AUTOMATION_TOKEN=uuid (for device self-registration)
SELF_REGISTRATION_PASSWORD=password (optional)
```

### Build Process

#### Backend Build

```bash
cd cloud/server
npm install
npm run build  # SWC compilation to dist/
```

**Docker Multi-Stage**:
```dockerfile
FROM node:14 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM node:14-slim
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/index.js"]
```

#### Frontend Build

```bash
cd cloud/webapp
npm install
npm run build  # Angular production build to dist/
```

**Docker Multi-Stage**:
```dockerfile
FROM node:14 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
```

#### Firmware Build

```bash
cd cloud/firmware
./build-fw.sh fridge  # Builds fridge firmware
```

**Docker Build Container**:
```bash
docker build -t fw-builder fw-buildcontainer/
docker run --rm \
  -v $(pwd):/workspace \
  -e FG_API_URL=http://api.example.com \
  -e FG_MQTT_HOST=mqtt.example.com \
  fw-builder \
  pio run -e fridge
```

**Output**: `firmware/.pio/build/{env}/firmware.bin`

### Deployment Modes

#### 1. Cloud Deployment (Production)

**Infrastructure**:
- Docker Compose on VM or cloud instance
- External access to ports 8080 (web), 1883 (MQTT)
- Domain names for API and MQTT
- TLS certificates (recommended)

**Use Case**: Multi-tenant SaaS platform

#### 2. Local/Edge Deployment

**Stack** (fridgegrow2.0/ or test/):
- Mosquitto MQTT broker
- InfluxDB
- Grafana
- Node-RED

**Use Case**: Single-tenant, air-gapped environments

**Differences**:
- No user authentication
- Local MQTT broker
- Direct InfluxDB access from Grafana
- Node-RED for automation instead of cloud logic

#### 3. Development Environment

```bash
# Terminal 1: MongoDB, InfluxDB, RabbitMQ
docker-compose up mongodb influxdb rabbitmq

# Terminal 2: API Server
cd cloud/server
npm run dev  # ts-node-dev with watch

# Terminal 3: Web App
cd cloud/webapp
npm start  # Angular dev server on port 4200

# Terminal 4: Firmware (PlatformIO)
cd cloud/firmware
pio run -e fridge-test -t upload && pio device monitor
```

### Scaling Considerations

**Current Limitations**:
- Single API server instance (no load balancing)
- MQTT client state in API server (not horizontally scalable)
- No database replication or clustering

**Scaling Recommendations**:

1. **API Server Horizontal Scaling**:
   - Extract MQTT client to separate service
   - Use load balancer (Nginx, HAProxy)
   - Session storage in Redis (if needed)

2. **Database Scaling**:
   - MongoDB replica set (3+ nodes)
   - InfluxDB clustering or InfluxDB Cloud
   - Read replicas for API queries

3. **MQTT Broker Scaling**:
   - RabbitMQ clustering
   - Separate MQTT message processor (consumer group)

4. **Caching Layer**:
   - Redis for frequently accessed data
   - Cache device configurations
   - Cache latest sensor readings

---

## Frontend Architecture

### Angular Application Structure

```
webapp/src/app/
├── app.component.ts           # Root component
├── app-routing.module.ts      # Main routing
├── login/                     # Login module
├── device-list/               # Device list module
├── device-charts/             # Device monitoring charts
├── device-settings/           # Device configuration
├── device-test/               # Device test mode
├── diagnostics/               # Admin diagnostics
├── classes/                   # Device class management (admin)
├── account/                   # User account settings
├── services/
│   ├── auth.service.ts        # Authentication state
│   ├── device.service.ts      # Device CRUD operations
│   ├── data.service.ts        # Time-series data queries
│   └── device-admin.service.ts # Admin-only operations
└── guards/
    ├── auth.guard.ts          # Authenticated route protection
    └── is-admin.guard.ts      # Admin route protection
```

### State Management

**Approach**: Service-based state (no NgRx or Akita)

**Auth State** (AuthService):
```typescript
@Injectable({ providedIn: 'root' })
export class AuthService {
  private currentUserSubject = new BehaviorSubject<User | null>(null);
  public currentUser$ = this.currentUserSubject.asObservable();

  login(username: string, password: string) {
    return this.http.post<AuthResponse>('/login', { username, password })
      .pipe(tap(response => {
        localStorage.setItem('accessToken', response.accessToken);
        this.currentUserSubject.next(response.user);
      }));
  }

  logout() {
    localStorage.removeItem('accessToken');
    this.currentUserSubject.next(null);
  }

  get isAuthenticated(): boolean {
    return !!localStorage.getItem('accessToken');
  }

  get isAdmin(): boolean {
    return this.currentUserSubject.value?.is_admin || false;
  }
}
```

### Routing & Navigation

**Main Routes**:
```typescript
const routes: Routes = [
  { path: 'login', component: LoginComponent },
  {
    path: 'devices',
    component: DeviceListComponent,
    canActivate: [AuthGuard]
  },
  {
    path: 'device/:id/charts',
    component: DeviceChartsComponent,
    canActivate: [AuthGuard]
  },
  {
    path: 'device/:id/settings',
    component: DeviceSettingsComponent,
    canActivate: [AuthGuard]
  },
  {
    path: 'diagnostics',
    component: DiagnosticsComponent,
    canActivate: [AuthGuard, IsAdminGuard]
  },
  { path: '', redirectTo: '/devices', pathMatch: 'full' }
];
```

### Data Flow

**Polling Pattern**:
```typescript
@Component({ ... })
export class DeviceChartsComponent implements OnInit, OnDestroy {
  private updateInterval: any;

  ngOnInit() {
    this.loadLatestData();
    this.updateInterval = setInterval(() => {
      this.loadLatestData();
    }, 10000); // Poll every 10 seconds
  }

  ngOnDestroy() {
    clearInterval(this.updateInterval);
  }

  async loadLatestData() {
    const data = await this.dataService.getLatest(this.deviceId, 'temperature');
    this.updateChart(data);
  }
}
```

### Chart Visualization

**Libraries**:
- Highcharts for time-series charts
- ng2-charts (Chart.js) for simple visualizations

**Example**:
```typescript
this.chartOptions = {
  chart: { type: 'line', zoomType: 'x' },
  title: { text: 'Temperature History' },
  xAxis: { type: 'datetime' },
  yAxis: { title: { text: 'Temperature (°C)' } },
  series: [{
    name: 'Temperature',
    data: this.temperatureData // [[timestamp, value], ...]
  }]
};
```

### UI Components (Ionic)

**Key Components**:
- `<ion-header>`, `<ion-toolbar>`, `<ion-title>` - Navigation
- `<ion-content>` - Scrollable content
- `<ion-card>` - Device cards
- `<ion-list>`, `<ion-item>` - Lists
- `<ion-button>` - Actions
- `<ion-input>`, `<ion-select>` - Forms
- `<ion-toggle>` - Switches
- `<ion-loading>`, `<ion-toast>` - Feedback

**Mobile-First Design**: Responsive layouts that work on phones, tablets, desktops.

---

## Firmware Architecture

### ESP32 Firmware Structure

```
firmware/
├── platformio.ini              # Build environments
├── src/                        # Common core
│   ├── main.cpp                # Setup, loop, ISR
│   ├── fridgecloud.cpp/.h      # MQTT communication
│   ├── wifi.cpp/.h             # WiFi management
│   ├── ui.cpp/.h               # Display & menu system
│   ├── settings.cpp/.h         # Persistent settings (EEPROM)
│   └── automation.h            # Automation interface
├── src_hwtype/                 # Hardware-specific implementations
│   ├── fridge/
│   │   ├── automation.h        # Climate control (PID)
│   │   ├── hardware.h          # Pin definitions, peripherals
│   │   └── sensors.h           # Sensor initialization
│   ├── fan/
│   ├── light/
│   └── plug/
└── lib/                        # External libraries
```

### Build Environments (PlatformIO)

**platformio.ini**:
```ini
[env:fridge]
platform = espressif32
board = heltec_wifi_lora_32_V2
framework = arduino
build_flags =
  -DHWTYPE_FRIDGE
  -DFG_API_URL=\"${FG_API_URL}\"
  -DFG_MQTT_HOST=\"${FG_MQTT_HOST}\"
build_src_filter =
  +<*>
  +<../src_hwtype/fridge>

[env:fan]
build_flags = -DHWTYPE_FAN
build_src_filter = +<*> +<../src_hwtype/fan>

[env:light]
build_flags = -DHWTYPE_LIGHT
build_src_filter = +<*> +<../src_hwtype/light>

[env:plug]
build_flags = -DHWTYPE_PLUG
build_src_filter = +<*> +<../src_hwtype/plug>
```

### Firmware Lifecycle

**1. First Boot**:
```
Setup WiFi → Register with API → Get device_id/credentials → Request claim code → Show code on display
```

**2. Normal Operation**:
```
Connect WiFi → Connect MQTT → Subscribe to config/firmware topics → Start sensor loop → Publish status every 10s
```

**3. Configuration Update**:
```
Receive config on MQTT → Parse JSON → Update settings → Save to EEPROM → Apply automation changes
```

**4. OTA Update**:
```
Receive firmware URL on MQTT → Download binary → Verify checksum → Flash firmware → Reboot → Publish new version
```

### Main Loop

```cpp
void loop() {
  // MQTT client
  mqtt.loop();

  // WiFi keepalive
  wifi.checkConnection();

  // Read sensors (every 10 seconds)
  if (millis() - lastSensorRead > 10000) {
    readSensors();
    runAutomation();
    publishStatus();
    lastSensorRead = millis();
  }

  // Update display
  ui.update();

  // Handle rotary encoder (ISR)
  handleEncoder();

  // Watchdog reset
  esp_task_wdt_reset();
}
```

### Automation Modes

**Fridge Controller**:
1. **PID Mode**: PID controller for temperature
2. **Hysteresis Mode**: Simple on/off with deadband
3. **Manual Mode**: User-controlled outputs
4. **Schedule Mode**: Time-based control

**Fan Controller**:
1. **RPM Control**: Target fan speed
2. **Temperature-based**: Fan speed based on temperature
3. **Manual Mode**: User-controlled speed

**Light Controller**:
1. **Schedule Mode**: Day/night dimming
2. **Manual Mode**: User-controlled brightness

### Sensor Abstraction

```cpp
// sensors.h (hardware-specific)
class Sensors {
public:
  void init();
  float readTemperature();
  float readHumidity();
  float readCO2();
  int readRPM();
};

// main.cpp (common core)
Sensors sensors;
sensors.init();
float temp = sensors.readTemperature();
```

### Settings Persistence

**EEPROM Structure**:
```cpp
struct Settings {
  char device_id[37];
  char mqtt_username[65];
  char mqtt_password[65];
  char wifi_ssid[33];
  char wifi_password[65];
  Configuration config;  // Device-specific settings
  uint32_t checksum;
};
```

**Save/Load**:
```cpp
void saveSettings() {
  EEPROM.put(0, settings);
  EEPROM.commit();
}

void loadSettings() {
  EEPROM.get(0, settings);
  if (validateChecksum()) {
    applySettings();
  }
}
```

### OTA Update Implementation

```cpp
void handleFirmwareUpdate(String url) {
  HTTPClient http;
  http.begin(url);
  int httpCode = http.GET();

  if (httpCode == HTTP_CODE_OK) {
    int contentLength = http.getSize();
    bool canBegin = Update.begin(contentLength);

    if (canBegin) {
      WiFiClient * stream = http.getStreamPtr();
      size_t written = Update.writeStream(*stream);

      if (Update.end()) {
        if (Update.isFinished()) {
          Serial.println("Update success. Rebooting...");
          ESP.restart();
        }
      }
    }
  }
  http.end();
}
```

---

## Recommended ADRs

Based on this analysis, the following ADRs should be created (following MADR template):

### High Priority (Foundational Decisions)

1. **ADR-001: Use Dual Database Strategy (MongoDB + InfluxDB)**
   - Status: Accepted
   - Context: IoT time-series data vs. metadata
   - Decision: MongoDB for metadata, InfluxDB for time-series

2. **ADR-002: Use RabbitMQ with MQTT Plugin for Message Broker**
   - Status: Accepted
   - Context: IoT device communication
   - Decision: RabbitMQ + MQTT plugin vs. Mosquitto

3. **ADR-003: Embed MQTT Client in API Server**
   - Status: Accepted (Consider deprecating)
   - Context: Message processing architecture
   - Decision: Single service vs. separate consumer

4. **ADR-004: Use JWT for Stateless Authentication**
   - Status: Accepted
   - Context: Multi-platform authentication
   - Decision: JWT vs. session-based auth

5. **ADR-005: Implement Device Claim Code Pairing System**
   - Status: Accepted
   - Context: Secure device-to-user association
   - Decision: Claim codes vs. QR/Bluetooth

### Medium Priority (Implementation Decisions)

6. **ADR-006: Implement Class-Based Firmware OTA Updates**
   - Status: Accepted
   - Context: Remote firmware management
   - Decision: OTA update strategy with rollout controls

7. **ADR-007: Use Hardware Abstraction Layer in Firmware**
   - Status: Accepted
   - Context: Multi-device firmware codebase
   - Decision: Shared core vs. separate repos

8. **ADR-008: Use Angular/Ionic for Hybrid Web/Mobile App**
   - Status: Accepted
   - Context: Multi-platform frontend
   - Decision: Ionic vs. React Native vs. separate apps

9. **ADR-009: Use Polling for Real-Time Data Updates**
   - Status: Accepted (Consider deprecating)
   - Context: Frontend real-time updates
   - Decision: Polling vs. WebSocket vs. SSE

10. **ADR-010: Organize Project as Multi-Repository Structure**
    - Status: Accepted
    - Context: Software vs. hardware versioning
    - Decision: Multi-repo vs. monorepo

### Lower Priority (Technology Choices)

11. **ADR-011: Use TypeScript for Backend Development**
    - Status: Accepted
    - Context: Type safety and maintainability
    - Decision: TypeScript vs. JavaScript

12. **ADR-012: Implement MVC Architecture with Service Layer**
    - Status: Accepted
    - Context: Backend code organization
    - Decision: MVC + services vs. alternatives

13. **ADR-013: Use Docker Compose for Service Orchestration**
    - Status: Accepted
    - Context: Multi-service deployment
    - Decision: Docker Compose vs. Kubernetes

14. **ADR-014: Use PlatformIO for Firmware Build System**
    - Status: Accepted
    - Context: Multi-platform embedded builds
    - Decision: PlatformIO vs. Arduino IDE

15. **ADR-015: Implement Service-Based State Management (No NgRx)**
    - Status: Accepted
    - Context: Frontend state management
    - Decision: Services vs. NgRx/Akita

### Future Decisions (Proposed)

16. **ADR-016: Migrate to WebSocket for Real-Time Updates** [Proposed]
    - Status: Proposed
    - Context: Replace polling with push-based updates
    - Decision: WebSocket vs. SSE vs. keep polling

17. **ADR-017: Extract MQTT Consumer to Separate Service** [Proposed]
    - Status: Proposed
    - Context: API server horizontal scaling
    - Decision: Separate service vs. keep embedded

18. **ADR-018: Implement TLS/SSL for MQTT** [Proposed]
    - Status: Proposed
    - Context: Security for device communication
    - Decision: MQTTS vs. VPN vs. plain MQTT

19. **ADR-019: Add Rate Limiting and API Security Enhancements** [Proposed]
    - Status: Proposed
    - Context: Production security hardening
    - Decision: Rate limiting strategy and tools

20. **ADR-020: Implement Database Replication and Backups** [Proposed]
    - Status: Proposed
    - Context: High availability and disaster recovery
    - Decision: Replication strategy and backup approach

---

## Appendix

### Repository URLs

- Cloud Platform: `/home/eo/plantalytix/cloud/`
- Hardware Designs: `/home/eo/plantalytix/hardware/`
- Local Stack: `/home/eo/plantalytix/fridgegrow2.0/`
- Test Environment: `/home/eo/plantalytix/test/`

### Key Configuration Files

- `cloud/docker-compose.yaml` - Service orchestration
- `cloud/server/src/index.ts` - API server entry point
- `cloud/webapp/src/app/app-routing.module.ts` - Frontend routing
- `cloud/firmware/platformio.ini` - Firmware build config

### Technology Documentation

- **MongoDB**: https://docs.mongodb.com/
- **InfluxDB v2**: https://docs.influxdata.com/influxdb/v2/
- **RabbitMQ**: https://www.rabbitmq.com/documentation.html
- **Angular**: https://angular.io/docs
- **Ionic**: https://ionicframework.com/docs
- **PlatformIO**: https://docs.platformio.org/
- **ESP32**: https://docs.espressif.com/projects/esp-idf/

### MADR Template Reference

This document follows the structure recommended by [MADR (Markdown Architectural Decision Records)](https://github.com/adr/madr).

To create individual ADRs from this analysis:
1. Create directory: `docs/decisions/`
2. Use template: `adr-template.md` or `adr-template-minimal.md`
3. Name format: `nnnn-title-with-dashes.md`
4. Include sections: Title, Status, Context, Decision, Consequences

---

**Document Version**: 1.0
**Last Updated**: 2025-11-15
**Analyzed By**: Claude Code (Sonnet 4.5)
**Analysis Method**: Deep codebase exploration with specialized agents
