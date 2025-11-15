# Plantalytix Documentation

This directory contains all technical documentation for the Plantalytix to Home Assistant migration.

## 📂 Documentation Structure

### Quick Start Guides

- **[GETTING-STARTED-HA.md](./GETTING-STARTED-HA.md)** - Hands-on Home Assistant setup guide
  - Docker Compose configuration
  - MQTT Discovery testing
  - Creating your first dashboard
  - Troubleshooting common issues

### Architecture Documentation

- **[ARCHITECTURE-DIAGRAM.mmd](./ARCHITECTURE-DIAGRAM.mmd)** - Complete system architecture diagrams
  - System overview
  - MQTT Discovery flow
  - Data flow architecture
  - Deployment diagrams
  - Hardware options

- **[FIRMWARE-UPDATE-ARCHITECTURE.md](./FIRMWARE-UPDATE-ARCHITECTURE.md)** - Firmware update system
  - Multi-source firmware management
  - OTA update flow
  - Rollout strategies
  - Update entity configuration

### Architecture Decision Records (ADRs)

Location: **[decisions/](./decisions/)**

Complete index available at: **[decisions/README.md](./decisions/README.md)**

| # | Title | Status | Impact |
|---|-------|--------|--------|
| [0001](./decisions/0001-migrate-to-home-assistant-platform.md) | Migrate to Home Assistant Platform | Proposed | 🔴 Critical |
| [0002](./decisions/0002-use-mqtt-discovery-for-device-integration.md) | MQTT Discovery Integration | Proposed | 🟡 High |
| [0003](./decisions/0003-esphome-vs-arduino-firmware.md) | ESPHome vs Arduino Firmware | Proposed | 🟡 Medium |
| [0004](./decisions/0004-home-assistant-data-storage-strategy.md) | Data Storage Strategy | Proposed | 🟡 Medium |
| [0005](./decisions/0005-home-assistant-automation-and-control-logic.md) | Automation & Control Logic | Proposed | 🟡 Medium |
| [0006](./decisions/0006-home-assistant-single-user-deployment.md) | Single-User Local Deployment | Proposed | 🔴 High |
| [0007](./decisions/0007-home-assistant-ui-and-dashboards.md) | UI & Dashboards | Proposed | 🔴 High |
| [0008](./decisions/0008-firmware-mqtt-discovery-implementation.md) | MQTT Discovery Implementation | Proposed | 🟢 Low |
| [0009](./decisions/0009-firmware-update-management-system.md) | Firmware Update Management | Proposed | 🔴 High |
| [0010](./decisions/0010-fridge-cam-plant-monitoring.md) | Camera & AI Monitoring | Proposed | 🔴 High |

## 🎯 Documentation by Role

### For Developers

**Start here:** [GETTING-STARTED-HA.md](./GETTING-STARTED-HA.md)

Essential reading:
1. [ADR-0002: MQTT Discovery](./decisions/0002-use-mqtt-discovery-for-device-integration.md)
2. [ADR-0008: MQTT Discovery Implementation](./decisions/0008-firmware-mqtt-discovery-implementation.md)
3. [Architecture Diagrams](./ARCHITECTURE-DIAGRAM.mmd)

### For Architects

**Start here:** [decisions/README.md](./decisions/README.md)

Essential reading:
1. [ADR-0001: Platform Migration](./decisions/0001-migrate-to-home-assistant-platform.md)
2. [ADR-0006: Deployment Architecture](./decisions/0006-home-assistant-single-user-deployment.md)
3. [ADR-0004: Data Storage](./decisions/0004-home-assistant-data-storage-strategy.md)

### For Firmware Engineers

**Start here:** [ADR-0003: ESPHome vs Arduino](./decisions/0003-esphome-vs-arduino-firmware.md)

Essential reading:
1. [ADR-0008: MQTT Discovery Implementation](./decisions/0008-firmware-mqtt-discovery-implementation.md)
2. [ADR-0009: Firmware Update Management](./decisions/0009-firmware-update-management-system.md)
3. [Firmware Update Architecture](./FIRMWARE-UPDATE-ARCHITECTURE.md)

### For DevOps/Infrastructure

**Start here:** [ADR-0006: Single-User Deployment](./decisions/0006-home-assistant-single-user-deployment.md)

Essential reading:
1. [Getting Started Guide](./GETTING-STARTED-HA.md)
2. [Architecture Diagrams](./ARCHITECTURE-DIAGRAM.mmd)
3. [ADR-0004: Data Storage Strategy](./decisions/0004-home-assistant-data-storage-strategy.md)

## 📊 Viewing Mermaid Diagrams

### On GitHub
Mermaid diagrams render automatically when viewing `.mmd` files or markdown files with mermaid code blocks on GitHub.

### Locally

**Option 1: VS Code Extension**
- Install [Markdown Preview Mermaid Support](https://marketplace.visualstudio.com/items?itemName=bierner.markdown-mermaid)
- Open any `.mmd` or `.md` file
- Use preview pane (Ctrl+Shift+V)

**Option 2: Mermaid Live Editor**
- Visit [mermaid.live](https://mermaid.live/)
- Copy diagram code from `.mmd` files
- Paste into editor for interactive viewing

**Option 3: CLI Tool**
```bash
npm install -g @mermaid-js/mermaid-cli
mmdc -i docs/ARCHITECTURE-DIAGRAM.mmd -o architecture.png
```

## 🔄 Documentation Updates

### Lifecycle

```
Draft → Review → Approved → Published
           ↓
      Deprecated (if outdated)
```

### Versioning

Documentation follows the migration phases:

- **v1.0** (current): Initial comprehensive documentation
- **v1.1**: Updates from Phase 1 POC findings
- **v2.0**: Post-migration actual implementation documentation

### Contributing

See [CONTRIBUTING.md](../CONTRIBUTING.md) for:
- How to contribute
- ADR creation guidelines
- Mermaid diagram best practices
- Style guide

## 🔍 Finding Information

### By Topic

- **MQTT**: ADR-0002, ADR-0008, Getting Started Guide
- **Firmware**: ADR-0003, ADR-0008, ADR-0009
- **Deployment**: ADR-0006, Getting Started Guide
- **Database**: ADR-0004
- **UI/UX**: ADR-0007
- **Automation**: ADR-0005

### By Phase

- **Phase 1 (POC)**: Getting Started, ADR-0002, ADR-0008
- **Phase 2 (Backend)**: ADR-0003, ADR-0004, ADR-0005
- **Phase 3 (Frontend)**: ADR-0007
- **Phase 4 (Rollout)**: ADR-0006, ADR-0009

## 📞 Support

- **Questions about documentation**: Open an issue
- **Technical questions**: See main [README.md](../README.md)
- **ADR discussions**: See individual ADR files

## 📚 External Resources

### Home Assistant
- [Official Documentation](https://www.home-assistant.io/docs/)
- [MQTT Integration](https://www.home-assistant.io/integrations/mqtt/)
- [Developer Docs](https://developers.home-assistant.io/)

### ESPHome
- [Official Documentation](https://esphome.io/)
- [Components Reference](https://esphome.io/components/)

### MADR
- [MADR Template](https://adr.github.io/madr/)
- [ADR Tools](https://github.com/npryce/adr-tools)

---

**Last Updated:** 2025-11-15 | **Documentation Version:** 1.0
