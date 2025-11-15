# Plantalytix Documentation

Welcome to the technical documentation for the Plantalytix to Home Assistant migration.

> **📍 New here?** Start with the main [README.md](../README.md) for an overview, then come back here for technical details.

---

## 📂 Quick Links

### Getting Started
- **[GETTING-STARTED-HA.md](./GETTING-STARTED-HA.md)** - 5-minute quickstart with Home Assistant
  - Docker Compose setup
  - MQTT Discovery testing
  - Creating your first dashboard
  - Troubleshooting tips

### Architecture & Design
- **[ARCHITECTURE-DIAGRAM.mmd](./ARCHITECTURE-DIAGRAM.mmd)** - Visual diagrams (8+ diagrams)
  - System overview
  - MQTT Discovery flow
  - Data architecture
  - Deployment options
  - Hardware configurations

- **[FIRMWARE-UPDATE-ARCHITECTURE.md](./FIRMWARE-UPDATE-ARCHITECTURE.md)** - OTA update system
  - Multi-source firmware management
  - Update flow and rollout strategies
  - Home Assistant integration

### Architecture Decision Records (ADRs)

**Complete index:** [decisions/README.md](./decisions/README.md)

| # | Title | Impact | Quick Link |
|---|-------|--------|------------|
| 0001 | Migrate to Home Assistant Platform | 🔴 Critical | [View](./decisions/0001-migrate-to-home-assistant-platform.md) |
| 0002 | MQTT Discovery Integration | 🟡 High | [View](./decisions/0002-use-mqtt-discovery-for-device-integration.md) |
| 0003 | ESPHome vs Arduino Firmware | 🟡 Medium | [View](./decisions/0003-esphome-vs-arduino-firmware.md) |
| 0004 | Data Storage Strategy | 🟡 Medium | [View](./decisions/0004-home-assistant-data-storage-strategy.md) |
| 0005 | Automation & Control Logic | 🟡 Medium | [View](./decisions/0005-home-assistant-automation-and-control-logic.md) |
| 0006 | Single-User Local Deployment | 🔴 High | [View](./decisions/0006-home-assistant-single-user-deployment.md) |
| 0007 | UI & Dashboards | 🔴 High | [View](./decisions/0007-home-assistant-ui-and-dashboards.md) |
| 0008 | MQTT Discovery Implementation | 🟢 Low | [View](./decisions/0008-firmware-mqtt-discovery-implementation.md) |
| 0009 | Firmware Update Management | 🔴 High | [View](./decisions/0009-firmware-update-management-system.md) |
| 0010 | Camera & AI Monitoring | 🔴 High | [View](./decisions/0010-fridge-cam-plant-monitoring.md) |

---

## 🎯 Navigation by Role

### 👨‍💻 For Developers
1. [GETTING-STARTED-HA.md](./GETTING-STARTED-HA.md) - Get HA running in 5 minutes
2. [ADR-0002: MQTT Discovery](./decisions/0002-use-mqtt-discovery-for-device-integration.md) - Device integration
3. [ADR-0008: Implementation Guide](./decisions/0008-firmware-mqtt-discovery-implementation.md) - Code examples
4. [ARCHITECTURE-DIAGRAM.mmd](./ARCHITECTURE-DIAGRAM.mmd) - System architecture

### 🏗️ For Architects
1. [decisions/README.md](./decisions/README.md) - All ADRs with decision flow
2. [ADR-0001: Platform Migration](./decisions/0001-migrate-to-home-assistant-platform.md) - Foundation
3. [ADR-0006: Deployment](./decisions/0006-home-assistant-single-user-deployment.md) - Local architecture
4. [ARCHITECTURE-DIAGRAM.mmd](./ARCHITECTURE-DIAGRAM.mmd) - Visual diagrams

### ⚙️ For Firmware Engineers
1. [ADR-0003: ESPHome vs Arduino](./decisions/0003-esphome-vs-arduino-firmware.md) - Firmware strategy
2. [ADR-0008: MQTT Discovery](./decisions/0008-firmware-mqtt-discovery-implementation.md) - Implementation
3. [ADR-0009: Firmware Updates](./decisions/0009-firmware-update-management-system.md) - OTA updates
4. [FIRMWARE-UPDATE-ARCHITECTURE.md](./FIRMWARE-UPDATE-ARCHITECTURE.md) - Update system

### 🚀 For DevOps
1. [ADR-0006: Deployment](./decisions/0006-home-assistant-single-user-deployment.md) - Single-user local setup
2. [GETTING-STARTED-HA.md](./GETTING-STARTED-HA.md) - Practical setup guide
3. [ADR-0004: Data Storage](./decisions/0004-home-assistant-data-storage-strategy.md) - Database strategy
4. [ARCHITECTURE-DIAGRAM.mmd](./ARCHITECTURE-DIAGRAM.mmd) - Deployment diagrams

---

## 📊 Viewing Mermaid Diagrams

### On GitHub
Mermaid diagrams render automatically when viewing `.mmd` files or markdown files on GitHub.

### Locally

**VS Code Extension** (Recommended)
```bash
# Install: Markdown Preview Mermaid Support
# Extension ID: bierner.markdown-mermaid
# Then: Open any .mmd file and use preview (Ctrl+Shift+V)
```

**Mermaid Live Editor**
- Visit [mermaid.live](https://mermaid.live/)
- Copy diagram code from `.mmd` files
- Paste for interactive viewing

**CLI Tool**
```bash
npm install -g @mermaid-js/mermaid-cli
mmdc -i docs/ARCHITECTURE-DIAGRAM.mmd -o architecture.png
```

---

## 🔍 Find Information by Topic

| Topic | Documents |
|-------|-----------|
| **MQTT** | ADR-0002, ADR-0008, Getting Started Guide |
| **Firmware** | ADR-0003, ADR-0008, ADR-0009, Firmware Update Architecture |
| **Deployment** | ADR-0006, Getting Started Guide |
| **Database** | ADR-0004 |
| **UI/UX** | ADR-0007 |
| **Automation** | ADR-0005 |
| **Camera/AI** | ADR-0010 |

## 📚 External Resources

### Home Assistant
- [Official Documentation](https://www.home-assistant.io/docs/)
- [MQTT Integration](https://www.home-assistant.io/integrations/mqtt/)
- [Developer Docs](https://developers.home-assistant.io/)
- [Community Forum](https://community.home-assistant.io/)

### ESPHome
- [Official Documentation](https://esphome.io/)
- [Components Reference](https://esphome.io/components/)
- [Device Database](https://www.esphome-devices.com/)

### MADR (ADR Format)
- [MADR Template](https://adr.github.io/madr/)
- [ADR Tools](https://github.com/npryce/adr-tools)
- [ADR GitHub Organization](https://adr.github.io/)

---

## 🤝 Contributing

See [CONTRIBUTING.md](../CONTRIBUTING.md) for:
- How to contribute to documentation
- ADR creation guidelines
- Mermaid diagram best practices
- Style guide and formatting rules

---

**Last Updated:** 2025-11-15 | **Version:** 1.0

For the main overview and getting started, see [../README.md](../README.md)
