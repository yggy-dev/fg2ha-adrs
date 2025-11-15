# Architecture Decision Records (ADRs)

This directory contains Architecture Decision Records (ADRs) for the Plantalytix platform migration to Home Assistant.

## About ADRs

Architecture Decision Records document important architectural decisions made during the project lifecycle. They capture the context, decision, consequences, and alternatives considered.

This project follows the [MADR (Markdown Architectural Decision Records)](https://github.com/adr/madr) format.

## Index of Decisions

### Migration Strategy

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [0001](./0001-migrate-to-home-assistant-platform.md) | Migrate Plantalytix to Home Assistant Platform | Proposed | 2025-11-15 |

**Summary**: Fundamental decision to migrate from custom IoT platform to Home Assistant-based architecture. Replaces custom Node.js backend, Angular frontend, and device management with HA ecosystem.

**Impact**: High - Affects entire architecture

---

### Device Integration

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [0002](./0002-use-mqtt-discovery-for-device-integration.md) | Use MQTT Discovery for Device Integration with Home Assistant | Proposed | 2025-11-15 |

**Summary**: Implement MQTT Discovery protocol for automatic device registration with Home Assistant. Devices publish discovery messages following HA conventions, enabling zero-configuration device addition.

**Impact**: High - Requires firmware changes for all devices

---

### Firmware Architecture

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [0003](./0003-esphome-vs-arduino-firmware.md) | ESPHome vs Arduino Firmware for Device Implementation | Proposed | 2025-11-15 |

**Summary**: Hybrid approach using ESPHome for simple devices (fan, light, socket) and Arduino+MQTT discovery for complex devices (fridge controller). Balances code reduction with control requirements.

**Impact**: Medium - Changes firmware development approach for new devices

---

### Data Storage

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [0004](./0004-home-assistant-data-storage-strategy.md) | Home Assistant Data Storage Strategy | Proposed | 2025-11-15 |

**Summary**: Dual-database strategy using PostgreSQL for HA Recorder (14-day retention) and InfluxDB v2 for long-term time-series data (1 year+). Includes data migration plan from existing InfluxDB.

**Impact**: Medium - Maintains existing InfluxDB investment with new HA Recorder

---

### Automation & Control

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [0005](./0005-home-assistant-automation-and-control-logic.md) | Home Assistant Automation and Control Logic Implementation | Proposed | 2025-11-15 |

**Summary**: Layered approach with critical PID control in firmware (autonomous operation) and non-critical automation in HA (scheduling, notifications, multi-device coordination). Optional AppDaemon for complex logic.

**Impact**: Medium - Distributes control logic between firmware and HA

---

### Deployment Architecture

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [0006](./0006-home-assistant-single-user-deployment.md) | Home Assistant Single-User Local Deployment | Proposed | 2025-11-15 |

**Summary**: Deploy Plantalytix as a single Home Assistant instance per growing facility on local network. All authentication handled by HA, no cloud dependencies, simple local deployment.

**Impact**: High - Defines deployment model for single-user local installations

---

### User Interface

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [0007](./0007-home-assistant-ui-and-dashboards.md) | Home Assistant UI and Dashboard Strategy | Proposed | 2025-11-15 |

**Summary**: Replace Angular/Ionic frontend with HA Lovelace dashboards using ApexCharts for time-series, Mushroom cards for modern UI, and custom cards/integrations for Plantalytix-specific features (claim codes, firmware management).

**Impact**: High - Complete frontend replacement

---

### Firmware Management

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [0008](./0008-firmware-mqtt-discovery-implementation.md) | Firmware MQTT Discovery Implementation Guide | Proposed | 2025-11-15 |
| [0009](./0009-firmware-update-management-system.md) | Firmware Update Management System with Multi-Source Support | Proposed | 2025-11-15 |

**Summary ADR-0008**: Detailed implementation guide for adding MQTT Discovery to Arduino firmware, including working C++ code examples, testing procedures, and migration strategy.

**Impact**: Low - Implementation guide

**Summary ADR-0009**: Custom HA integration leveraging update entities to manage firmware from multiple sources (GitHub, GitLab, custom URLs) with auto-update policies, changelog display, and rollout control.

**Impact**: High - New firmware management system

---

### Advanced Features

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [0010](./0010-fridge-cam-plant-monitoring.md) | Fridge Cam Plant Monitoring with AI Analysis | Proposed | 2025-11-15 |

**Summary**: Camera integration for visual plant monitoring with automated timelapse generation (8-month cycles into 10-min videos) and AI-powered health analysis using LLMs (OpenAI/Claude/local). Includes live streaming dashboard, automated disease/pest detection, and configurable alerts.

**Impact**: High - New camera monitoring and AI analysis system

---

## Decision Flow

```
┌─────────────────────────────────────────────────────────┐
│ ADR-0001: Migrate to Home Assistant                     │
│ (Foundational Decision)                                 │
└─────────────┬───────────────────────────────────────────┘
              │
              ├─→ ADR-0002: MQTT Discovery
              │   (Device Integration)
              │       │
              │       └─→ ADR-0008: MQTT Discovery Implementation
              │           (Implementation Guide)
              │
              ├─→ ADR-0003: ESPHome vs Arduino
              │   (Firmware Architecture)
              │       │
              │       ├─→ ADR-0009: Firmware Update Management
              │       │   (Multi-Source Updates)
              │       │
              │       └─→ ADR-0010: Fridge Cam Monitoring
              │           (Camera + AI Analysis)
              │
              ├─→ ADR-0004: Data Storage
              │   (PostgreSQL + InfluxDB)
              │
              ├─→ ADR-0005: Automation & Control
              │   (Firmware + HA + AppDaemon)
              │
              ├─→ ADR-0006: Single-User Deployment
              │   (Local HA Instance)
              │
              └─→ ADR-0007: UI & Dashboards
                  (Lovelace + Custom Cards)
```

## Decision Dependencies

### Must Be Decided First
- **ADR-0001** (Migrate to HA) - All other decisions depend on this

### Can Be Decided Independently
- **ADR-0002** (MQTT Discovery)
- **ADR-0003** (ESPHome vs Arduino)
- **ADR-0004** (Data Storage)
- **ADR-0007** (UI & Dashboards)

### Dependent on Other Decisions
- **ADR-0005** (Automation) - Depends on ADR-0002 (device integration) and ADR-0003 (firmware platform)
- **ADR-0006** (Deployment) - Depends on ADR-0001 (migration to HA)

## Architecture Overview

### Current (Plantalytix)

```
┌──────────────────────────────────────────────────────────┐
│  Custom Platform                                         │
├──────────────────────────────────────────────────────────┤
│  Frontend: Angular + Ionic                               │
│  Backend: Node.js + Express + TypeScript                 │
│  Database: MongoDB (metadata) + InfluxDB (time-series)   │
│  Message Broker: RabbitMQ + MQTT Plugin                  │
│  Firmware: Arduino (ESP32) + Custom MQTT                 │
│  Auth: Custom JWT                                        │
│  Automation: Firmware PID + Custom API                   │
│  Deployment: Docker Compose (single instance)            │
└──────────────────────────────────────────────────────────┘
```

### Proposed (Home Assistant - Single-User Local)

```
┌──────────────────────────────────────────────────────────┐
│  Home Assistant Platform (Local Network)                 │
├──────────────────────────────────────────────────────────┤
│  Frontend: Lovelace + ApexCharts + Custom Cards          │
│  Backend: Home Assistant Core                            │
│  Database: PostgreSQL (Recorder) + InfluxDB (long-term)  │
│  Message Broker: Mosquitto (Local MQTT)                  │
│  Firmware: ESPHome (simple) + Arduino (complex) + MQTT   │
│           Discovery                                      │
│  Auth: Home Assistant Built-in (No Custom Auth)          │
│  Automation: Firmware PID + HA Automations + AppDaemon   │
│  Deployment: Single Local Instance (Raspberry Pi/Server) │
└──────────────────────────────────────────────────────────┘
```

## Migration Phases

### Phase 1: Proof of Concept (6 weeks)
- **ADR-0001**: Deploy HA alongside existing system
- **ADR-0002**: Implement MQTT discovery for one device type
- **ADR-0004**: Set up PostgreSQL Recorder + InfluxDB integration
- **ADR-0007**: Create prototype dashboards
- **Validation**: Compare feature parity and performance

### Phase 2: Backend Migration (8 weeks)
- **ADR-0002**: Add MQTT discovery to all device types
- **ADR-0003**: Migrate simple devices to ESPHome
- **ADR-0004**: Migrate historical data to InfluxDB
- **ADR-0005**: Implement HA automations for scheduling
- **ADR-0006**: Document local deployment options

### Phase 3: Frontend Migration (4 weeks)
- **ADR-0007**: Create production dashboards
- **ADR-0007**: Develop custom cards for Plantalytix features
- **ADR-0007**: Mobile app configuration and testing

### Phase 4: Production Rollout (4 weeks)
- Create installation guide
- Test installations on various hardware
- User documentation and training
- Release v1.0

**Total Estimated Timeline**: 19 weeks (4-5 months)

## Key Metrics

| Metric | Current (Custom) | Proposed (HA) | Impact |
|--------|------------------|---------------|--------|
| **Lines of Code (Backend)** | ~10,000 (TypeScript) | ~0 (HA Core) | -100% |
| **Lines of Code (Frontend)** | ~8,000 (Angular) | ~500 (Lovelace YAML) + Custom Cards | -90% |
| **Lines of Code (Auth/User Mgmt)** | ~1,000 (JWT, MongoDB) | ~0 (HA Built-in) | -100% |
| **Lines of Code (Firmware - Simple)** | ~2,000 (C++) | ~100 (ESPHome YAML) | -95% |
| **Services to Maintain** | 6 (API, MongoDB, InfluxDB, RabbitMQ, Nginx, Auth) | 3 (HA, PostgreSQL, InfluxDB) | -50% |
| **Features Out-of-Box** | 0 (all custom) | 100+ (HA integrations) | ∞% |
| **Mobile App Development** | Required (Ionic/Capacitor) | Free (HA Companion) | Eliminated |
| **Security Updates** | Manual (custom code) | Automatic (HA monthly) | Improved |
| **Hardware Requirements** | Server (cloud hosting) | Raspberry Pi ($75-100) | -95% cost |
| **Developer Onboarding** | 2-3 months | 2-3 weeks | -75% |

## Benefits Summary

### Quantitative Benefits
- **90% reduction in code maintenance** (backend + frontend)
- **95% reduction in firmware code** (for ESPHome devices)
- **Monthly security updates** instead of manual patches
- **100+ integrations** available immediately
- **$0 mobile app development** cost (use HA Companion)

### Qualitative Benefits
- **Proven platform** with 10+ years of development
- **Active community** with 1000+ contributors
- **Professional UI/UX** surpassing custom implementation
- **Ecosystem integration** (Zigbee, Z-Wave, Thread, Matter)
- **Local-first** architecture (works without internet)
- **Better developer experience** (YAML vs. TypeScript/C++)

## Risks and Mitigation

| Risk | Severity | Mitigation |
|------|----------|------------|
| **Feature Parity Gap** | High | Prototype phase validates critical features |
| **Multi-Tenancy Complexity** | Medium | Automated provisioning scripts and testing |
| **Firmware Migration Effort** | Medium | Hybrid approach (ESPHome + Arduino) |
| **Data Migration Errors** | High | Comprehensive testing and validation scripts |
| **User Adoption** | Medium | Training, documentation, gradual rollout |
| **Lock-in to HA** | Low | Firmware remains independent, data exportable |

## Resources

### Home Assistant Documentation
- [Architecture](https://developers.home-assistant.io/docs/architecture_index)
- [MQTT Integration](https://www.home-assistant.io/integrations/mqtt/)
- [Lovelace UI](https://www.home-assistant.io/dashboards/)
- [Custom Integrations](https://developers.home-assistant.io/docs/creating_component_index/)

### ESPHome Documentation
- [ESPHome](https://esphome.io/)
- [Climate Component](https://esphome.io/components/climate/index.html)
- [Custom Components](https://esphome.io/custom/custom_component.html)

### Community Resources
- [Home Assistant Community](https://community.home-assistant.io/)
- [HACS (Custom Cards & Integrations)](https://hacs.xyz/)
- [ESPHome Devices](https://www.esphome-devices.com/)

### Plantalytix Resources
- [Architecture Analysis](../../ARCHITECTURE-ANALYSIS.md)
- [Current System Documentation](../../README.md)

## Contributing to ADRs

### Creating a New ADR

1. Copy the MADR template (see [MADR repo](https://github.com/adr/madr))
2. Number sequentially (0008, 0009, etc.)
3. Fill in all sections:
   - **Status**: Proposed, Accepted, Deprecated, Superseded
   - **Context**: What is the issue we're seeing that is motivating this decision?
   - **Decision**: What is the change we're proposing and/or doing?
   - **Consequences**: What becomes easier or harder by this change?
   - **Alternatives Considered**: What other options were considered?
4. Submit for review

### ADR Lifecycle

```
Proposed → Accepted → Implemented
    ↓
Deprecated (if outdated)
    ↓
Superseded by ADR-XXXX (if replaced)
```

### Review Process

1. Author creates ADR with status "Proposed"
2. Team reviews (architecture review meeting)
3. Discussion and iteration
4. Decision: Accept, Reject, or Defer
5. If accepted, status changes to "Accepted"
6. After implementation, add "Implemented" date

## Questions?

For questions about these ADRs or the migration strategy, contact:

- Architecture Team: [team contact]
- GitHub Issues: [repository link]
- Documentation: See [ARCHITECTURE-ANALYSIS.md](../../ARCHITECTURE-ANALYSIS.md)

---

**Last Updated**: 2025-11-15
**Next Review**: After Phase 1 POC completion (estimated 6 weeks)
