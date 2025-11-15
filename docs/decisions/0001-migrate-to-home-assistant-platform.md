# Migrate Plantalytix to Home Assistant Platform

## Status

Proposed

## Context

Plantalytix currently operates as a custom-built IoT platform with:
- Custom Node.js/TypeScript backend (Express API)
- Custom Angular/Ionic frontend
- MongoDB for metadata storage
- InfluxDB for time-series sensor data
- RabbitMQ for MQTT message brokering
- ESP32 firmware with custom MQTT implementation
- Custom user authentication and device management
- Custom automation logic

**Challenges with Current Architecture:**
1. **Maintenance Burden**: Maintaining custom backend, frontend, authentication, and infrastructure
2. **Limited Ecosystem**: No integration with existing smart home devices or services
3. **Feature Development Overhead**: Every feature (notifications, automation, graphs) requires custom development
4. **Scalability Concerns**: MQTT client embedded in API server, single-instance limitations
5. **User Experience**: Custom UI lacks polish and features of mature platforms
6. **Mobile App Maintenance**: Ionic/Capacitor requires separate mobile app maintenance
7. **No Multi-Protocol Support**: Limited to MQTT only, no Zigbee/Z-Wave/Thread support

**Home Assistant as a Platform:**
- **Mature Open-Source Platform**: 10+ years of development, large community
- **Rich Ecosystem**: 2000+ integrations (devices, services, protocols)
- **Built-in Features**: Authentication, user management, automation engine, notifications
- **Modern UI**: Lovelace dashboard system with mobile app (companion app)
- **Extensibility**: Custom components, integrations, dashboards
- **Active Development**: Monthly releases with new features
- **Multi-Protocol Support**: MQTT, Zigbee, Z-Wave, Thread, Matter, Bluetooth
- **Local-First**: Can operate entirely offline/air-gapped
- **Add-ons**: Built-in supervisor for running complementary services (InfluxDB, Grafana, Node-RED)

**Migration Opportunity:**
Home Assistant's architecture aligns well with Plantalytix requirements:
- MQTT discovery protocol for automatic device registration
- ESPHome framework for ESP32 firmware (alternative to Arduino)
- Climate component for temperature/humidity control
- Native InfluxDB integration for time-series data
- Automation engine for control logic (PID can be implemented via AppDaemon or Node-RED)
- RESTful API and WebSocket API for external integrations

## Decision

**We will migrate Plantalytix from a custom IoT platform to a Home Assistant-based architecture.**

### Migration Strategy: Phased Approach

**Phase 1: Parallel Deployment (Proof of Concept)**
- Deploy Home Assistant alongside existing Plantalytix system
- Migrate one device type (e.g., fan controller) to MQTT discovery
- Validate climate control, data logging, and automation
- Compare user experience and feature parity

**Phase 2: Backend Migration**
- Replace custom Node.js API with Home Assistant core
- Migrate authentication to Home Assistant user system
- Replace Angular frontend with Home Assistant Lovelace UI
- Configure InfluxDB integration for long-term data retention

**Phase 3: Firmware Migration**
- Option A: Keep Arduino firmware, add MQTT discovery support
- Option B: Migrate to ESPHome (recommended for long-term)
- Implement automatic device discovery
- Maintain backward compatibility during transition

**Phase 4: Data Migration**
- Export historical data from existing InfluxDB
- Import into Home Assistant recorder or InfluxDB instance
- Migrate device configurations to Home Assistant entities

**Phase 5: Decommission Custom Platform**
- Sunset custom backend and frontend
- Maintain only firmware and Home Assistant configuration

### Architecture Changes

**From:**
```
ESP32 Firmware → RabbitMQ → Custom API → MongoDB/InfluxDB → Angular UI
```

**To:**
```
ESP32 Firmware → MQTT Broker → Home Assistant → PostgreSQL/InfluxDB → Lovelace UI
                 (ESPHome)       (Core + Integrations)   (Recorder)
```

## Consequences

### Positive

✅ **Reduced Maintenance Burden**: Eliminate ~15,000 lines of custom backend/frontend code

✅ **Rich Feature Set**: Instant access to notifications, dashboards, mobile app, voice control

✅ **Better User Experience**: Polished UI with active development and improvements

✅ **Ecosystem Integration**: Can integrate with other smart home devices (lights, sensors, cameras)

✅ **Community Support**: Large community for troubleshooting and feature requests

✅ **Extensibility**: Can create custom components only for unique Plantalytix features

✅ **Multi-Tenancy via HA Instances**: Each customer/grow facility runs own HA instance

✅ **Automation Engine**: Built-in automation editor, Node-RED integration, AppDaemon for Python

✅ **Data Visualization**: Built-in history graphs, energy dashboard, Grafana integration

✅ **Security**: Regular security updates, established authentication system

✅ **Mobile App**: Official Home Assistant companion app (iOS/Android) with push notifications

✅ **Backup & Restore**: Built-in snapshot system for configuration and data

### Negative

❌ **Multi-Tenancy Complexity**: Home Assistant designed for single-tenant; SaaS model requires multiple HA instances

❌ **Learning Curve**: Team must learn Home Assistant architecture, YAML configuration, and integration patterns

❌ **Customization Constraints**: Limited to Home Assistant's extension points and patterns

❌ **Migration Effort**: Significant effort to migrate existing devices and users

❌ **Firmware Changes**: ESP32 firmware must implement MQTT discovery or migrate to ESPHome

❌ **Less Control**: Platform decisions made by Home Assistant community, not Plantalytix team

❌ **Configuration as Code**: Heavy reliance on YAML configuration files

❌ **Resource Requirements**: Home Assistant requires more resources than lightweight custom API

❌ **Database Migration**: Need to migrate or maintain dual databases during transition

❌ **API Limitations**: Home Assistant API less flexible than custom REST API for external integrations

### Risks

⚠️ **Feature Parity**: Some Plantalytix-specific features may not map directly to HA

⚠️ **Performance**: Need to validate HA performance with many devices/sensors

⚠️ **Lock-In**: Once migrated, difficult to move away from Home Assistant

⚠️ **Breaking Changes**: HA releases may introduce breaking changes requiring configuration updates

⚠️ **Multi-Tenancy**: Difficult to provide true SaaS with isolated HA instances (consider HA Cloud or separate VMs)

### Mitigation Strategies

1. **Proof of Concept**: Validate with small subset of devices before full migration
2. **Custom Components**: Develop custom HA components for Plantalytix-specific features
3. **ESPHome Migration**: Use ESPHome for firmware to reduce firmware maintenance
4. **Instance Management**: Develop orchestration for managing multiple HA instances (Docker, Kubernetes)
5. **Gradual Rollout**: Support both platforms during transition period
6. **Documentation**: Comprehensive migration guide for existing users

## Alternatives Considered

### Alternative 1: Keep Custom Platform, Add HA Integration

**Description**: Maintain Plantalytix backend, create HA integration component

**Pros**:
- Keep existing architecture and investments
- HA as optional add-on for users who want it
- Full control over core platform

**Cons**:
- Doesn't reduce maintenance burden
- Requires maintaining both systems
- Integration layer adds complexity

**Verdict**: Rejected - doesn't solve core maintenance problems

### Alternative 2: Migrate to ESPHome Only (No Full HA)

**Description**: Migrate firmware to ESPHome, keep custom backend/frontend

**Pros**:
- Reduces firmware maintenance
- ESPHome handles OTA updates
- Keeps control of backend/frontend

**Cons**:
- Still requires custom backend/frontend maintenance
- Loses HA ecosystem benefits
- ESPHome tightly coupled to HA anyway

**Verdict**: Rejected - partial solution that doesn't leverage full ecosystem

### Alternative 3: Commercial IoT Platform (AWS IoT, Azure IoT Hub)

**Description**: Migrate to cloud-based commercial IoT platform

**Pros**:
- Enterprise scalability and reliability
- Managed infrastructure
- Global deployment

**Cons**:
- Vendor lock-in
- Ongoing costs per device
- Less control over features
- Requires internet connectivity
- Doesn't solve UI/automation problems

**Verdict**: Rejected - high costs, vendor lock-in, doesn't align with local-first philosophy

### Alternative 4: Node-RED + InfluxDB + Grafana (Current Local Stack)

**Description**: Enhance existing local stack (fridgegrow2.0) as primary platform

**Pros**:
- Already deployed for local/edge use cases
- Lightweight and flexible
- Good for single-tenant deployments

**Cons**:
- No user authentication/management
- No mobile app
- Limited UI capabilities compared to HA
- Requires custom integration work

**Verdict**: Rejected - suitable for edge only, not a replacement for cloud platform

### Alternative 5: OpenHAB

**Description**: Migrate to OpenHAB instead of Home Assistant

**Pros**:
- Similar to HA (open-source smart home platform)
- Java-based (may suit some developers)
- Good integration ecosystem

**Cons**:
- Smaller community than Home Assistant
- Less active development
- Fewer integrations
- Steeper learning curve (Java, Xtend)

**Verdict**: Rejected - Home Assistant has larger community and more momentum

## Implementation Plan

### Phase 1: Proof of Concept (4 weeks)

1. **Week 1-2: HA Setup & Learning**
   - Install Home Assistant OS in test environment
   - Configure MQTT integration (broker: Mosquitto add-on)
   - Set up InfluxDB v2 add-on
   - Configure Grafana add-on
   - Learn Lovelace dashboard system

2. **Week 3: Device Migration**
   - Select one fan controller for testing
   - Implement MQTT discovery in firmware
   - Create climate entity in HA
   - Configure automation for fan control

3. **Week 4: Validation**
   - Test climate control functionality
   - Validate data logging to InfluxDB
   - Test automation triggers
   - Compare with existing platform
   - Document findings and gaps

### Phase 2: Full Migration Planning (2 weeks)

1. **Gap Analysis**: Identify features not available in HA
2. **Custom Component Design**: Plan custom components for unique features
3. **Data Migration Strategy**: Design historical data migration process
4. **Deployment Architecture**: Design multi-tenant instance management
5. **User Migration Plan**: Plan user transition process

### Phase 3: Implementation (12 weeks)

See subsequent ADRs for detailed implementation decisions:
- ADR-0002: MQTT Discovery Integration
- ADR-0003: ESPHome vs Arduino Firmware Decision
- ADR-0004: Data Storage Strategy
- ADR-0005: Multi-Tenancy Architecture
- ADR-0006: Automation and Control Logic
- ADR-0007: Frontend Customization
- ADR-0008: Migration and Deployment Strategy

## References

- [Home Assistant Architecture](https://developers.home-assistant.io/docs/architecture_index)
- [Home Assistant MQTT Integration](https://www.home-assistant.io/integrations/mqtt/)
- [ESPHome Documentation](https://esphome.io/)
- [Home Assistant InfluxDB Integration](https://www.home-assistant.io/integrations/influxdb/)
- [Home Assistant Climate Component](https://www.home-assistant.io/integrations/climate/)

## Notes

This decision represents a fundamental shift in Plantalytix architecture. The team should treat this as a potential rewrite rather than a simple migration. Success depends on:

1. **Executive buy-in**: Management must support the investment in migration
2. **Team training**: Developers need time to learn HA architecture
3. **User communication**: Existing users must understand benefits and transition plan
4. **Incremental rollout**: Avoid "big bang" migration
5. **Backward compatibility**: Support existing devices during transition period

**Decision Date**: 2025-11-15
**Decision Makers**: [To be filled]
**Review Date**: After Phase 1 POC completion
