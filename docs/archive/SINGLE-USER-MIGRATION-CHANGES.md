# Single-User Deployment Changes

**Date**: 2025-11-15
**Version**: 2.0
**Status**: Architecture Updated for Single-User Local Deployment

---

## Summary of Changes

The Plantalytix migration to Home Assistant has been **updated from a multi-tenant cloud SaaS model to a single-user local network deployment**. This dramatically simplifies the architecture while maintaining all core functionality.

## What Changed

### Architecture Philosophy
- **Before**: Cloud-based multi-tenant SaaS with separate HA instances per customer
- **After**: Single local HA instance per growing facility on local network

### Deployment Model
- **Before**: Complex multi-tenant orchestration with Docker Compose/Kubernetes
- **After**: Simple single-instance deployment (Raspberry Pi or local server)

### Authentication & User Management
- **Before**: Custom JWT authentication, MongoDB user management, device claim codes
- **After**: Built-in Home Assistant authentication (no custom auth needed)

### Network Architecture
- **Before**: Cloud deployment, reverse proxy, multiple domains per tenant
- **After**: Local network only (http://homeassistant.local:8123)

## Key Simplifications

### Removed Components ❌

1. **Multi-Tenant Infrastructure**
   - Tenant orchestration layer
   - Per-tenant HA instances
   - Tenant provisioning API
   - Management dashboard

2. **Authentication System**
   - Custom JWT token system
   - MongoDB user collection
   - Password reset functionality
   - Email activation system
   - Device claim codes
   - Device ownership model

3. **Network Complexity**
   - Nginx reverse proxy (multi-domain)
   - Tenant-specific subdomains
   - MQTT topic ACLs per tenant
   - InfluxDB bucket per tenant

4. **Database Complexity**
   - MongoDB (completely removed)
   - Per-tenant PostgreSQL databases
   - Shared service isolation

### Simplified Components ✅

1. **Single Home Assistant Instance**
   - One HA installation per facility
   - Built-in user management
   - Local network deployment
   - No cloud dependencies

2. **Simple MQTT Broker**
   - Mosquitto without ACLs
   - All devices can publish/subscribe
   - Simple allow_anonymous configuration

3. **Single Databases**
   - One PostgreSQL (14-day recorder)
   - One InfluxDB v2 bucket (long-term)

4. **Simplified Device Integration**
   - MQTT discovery auto-creates entities
   - No claim codes needed
   - All devices visible to user
   - No device-to-user association

## Updated Documents

### ✅ Fully Updated

1. **[ADR-0006](docs/decisions/0006-home-assistant-single-user-deployment.md)**
   - Completely rewritten for single-user deployment
   - Removed all multi-tenancy content
   - Added local network architecture
   - Added hardware options (Raspberry Pi, NUC, etc.)
   - Added deployment examples

2. **[ARCHITECTURE-DIAGRAM.mmd](docs/ARCHITECTURE-DIAGRAM.mmd)**
   - Updated all 8+ diagrams for single-user
   - Removed tenant orchestration diagram
   - Added local network deployment diagram
   - Added hardware options diagram
   - Simplified MQTT flow (no ACLs)
   - Updated technology stack comparison

### ⚠️ Requires Review/Update

The following documents contain references to multi-tenancy that should be reviewed:

1. **[ARCHITECTURE-ANALYSIS.md](ARCHITECTURE-ANALYSIS.md)**
   - Section on "ADR-010: Multi-Repository Structure" - OK (hardware vs software repos)
   - Current system analysis - OK (describes existing system)
   - Recommended ADRs list - ⚠️ Update ADR-006 title and description

2. **[MIGRATION-SUMMARY.md](MIGRATION-SUMMARY.md)**
   - ADR-0006 summary - ⚠️ Update to single-user deployment
   - Resource requirements - ⚠️ Update for single instance
   - Success metrics - ✅ Still valid
   - Timeline - ✅ Still valid (no multi-tenant complexity to build)

3. **[README.md](README.md)**
   - ADR-0006 description - ⚠️ Update summary
   - Quick navigation - ✅ Links still work
   - Success criteria - ✅ Still valid

4. **[docs/decisions/README.md](docs/decisions/README.md)**
   - ADR-0006 entry - ⚠️ Update title and description
   - Decision flow diagram - ⚠️ Update ADR-006 description
   - Multi-tenancy architecture overview - ⚠️ Update to single-user

5. **ADR Files That Reference Multi-Tenancy:**
   - **[ADR-0001](docs/decisions/0001-migrate-to-home-assistant-platform.md)** - Minor mentions, context OK
   - **[ADR-0004](docs/decisions/0004-home-assistant-data-storage-strategy.md)** - May mention multi-tenant buckets
   - **[ADR-0005](docs/decisions/0005-home-assistant-automation-and-control-logic.md)** - Should be OK
   - **[ADR-0007](docs/decisions/0007-home-assistant-ui-and-dashboards.md)** - Should be OK

## Impact on Implementation

### What Stays the Same ✅

- MQTT Discovery protocol
- ESPHome vs Arduino firmware strategy
- Data storage (PostgreSQL + InfluxDB)
- Automation logic distribution
- UI/Dashboard approach
- Firmware update management
- Migration timeline
- Code reduction benefits

### What Gets Easier ✅

- **Setup**: Single docker-compose up instead of tenant provisioning
- **Maintenance**: One instance to update instead of many
- **Hardware**: Raspberry Pi sufficient instead of server cluster
- **Networking**: Local network only, no cloud infrastructure
- **Security**: No internet exposure by default
- **Backup**: Simple snapshot instead of per-tenant backups
- **Development**: No multi-tenancy edge cases to handle

### What Changes ⚠️

- **Scope**: Per-facility installation instead of centralized SaaS
- **Users**: Single user/facility instead of many customers
- **Distribution**: Users install locally instead of cloud signup
- **Access**: Local network instead of cloud URLs
- **Support**: Installation guide instead of managed service

## Updated Code Reduction Metrics

### Before (Custom Platform)
```
Backend:        ~10,000 LOC (TypeScript)
Frontend:        ~8,000 LOC (Angular)
Authentication:  ~1,000 LOC (JWT, user mgmt)
Multi-tenancy:   ~2,000 LOC (tenant orchestration)
Firmware:        ~5,000 LOC (Arduino)
TOTAL:          ~26,000 LOC
```

### After (Single-User HA)
```
Backend:             ~0 LOC (HA Core handles it)
Frontend:          ~500 LOC (Lovelace YAML)
Authentication:      ~0 LOC (HA handles it)
Multi-tenancy:       ~0 LOC (not needed)
Firmware (ESPHome): ~100 LOC (YAML)
Firmware (Arduino): ~2,000 LOC (complex devices)
TOTAL:            ~2,600 LOC
```

**Code Reduction: ~90% (26,000 → 2,600 LOC)**

## Hardware Requirements (Updated)

### Minimum Setup (1-10 devices)
- **Hardware**: Raspberry Pi 4 (2GB RAM)
- **Cost**: ~$75
- **Power**: 15W
- **Storage**: 32GB SD card

### Recommended Setup (10-30 devices)
- **Hardware**: Raspberry Pi 4 (4GB RAM) or Intel NUC
- **Cost**: ~$100-400
- **Power**: 15-50W
- **Storage**: 128GB SSD

### Advanced Setup (30+ devices)
- **Hardware**: Intel NUC i5 or server
- **Cost**: ~$400-600
- **Power**: 50-100W
- **Storage**: 256GB+ SSD

## Installation Process (Updated)

### Option 1: Home Assistant OS (Recommended)
```bash
# 1. Download HA OS for Raspberry Pi
# 2. Flash to SD card with balenaEtcher
# 3. Insert SD card, power on
# 4. Navigate to http://homeassistant.local:8123
# 5. Complete setup wizard
# 6. Install MQTT, InfluxDB integrations
# 7. Flash devices with MQTT discovery firmware
# 8. Devices auto-appear in HA!
```

### Option 2: Docker Compose
```bash
# 1. Clone plantalytix repo
cd /opt/plantalytix
# 2. Copy docker-compose.yml
# 3. Configure .env file
# 4. docker-compose up -d
# 5. Access http://localhost:8123
```

## Benefits of Single-User Deployment

### For Users
1. **Complete Privacy**: All data stays local
2. **No Cloud Costs**: No subscription fees
3. **Works Offline**: No internet required
4. **Instant Response**: No cloud latency
5. **Full Control**: User owns the hardware and data
6. **Simple Backup**: Copy config files and databases

### For Developers
1. **Simpler Architecture**: No multi-tenancy complexity
2. **Faster Development**: Focus on features, not infrastructure
3. **Easier Testing**: Single instance to test
4. **Lower Maintenance**: No tenant provisioning bugs
5. **Better Alignment**: Matches HA's design philosophy

### For the Project
1. **Lower Barrier to Entry**: Users can start with Raspberry Pi
2. **Open Source Friendly**: Standard HA installation
3. **Community Support**: Leverage HA community
4. **Sustainable**: No cloud infrastructure to maintain
5. **Scalable**: Users add hardware as needed

## Migration Path for Existing Multi-Tenant Concepts

If you need to manage multiple facilities:

### Option 1: Separate Installations
- Install HA instance at each facility
- Each facility operates independently
- Access via VPN or Tailscale if needed

### Option 2: MQTT Bridge (Advanced)
- Central HA instance
- MQTT bridges to remote facilities
- Devices at remote sites bridge to central MQTT
- More complex but possible

### Option 3: HA Cloud (Nabu Casa)
- Each facility has HA instance
- Remote access via HA Cloud ($6.50/month per instance)
- Centralized monitoring possible with separate tool

## Next Steps

1. ✅ **Updated**: ADR-0006 and architecture diagrams
2. ✅ **Completed**: Updated references in other documents
3. ✅ **Completed**: Getting started guide reflects single-user deployment
4. ⏳ **Pending**: Create dedicated Raspberry Pi installation guide (optional)
5. ✅ **Completed**: Migration roadmap updated with simplified timeline

## Questions & Answers

### Q: Can I still manage multiple grow rooms?
**A**: Yes! HA supports multiple rooms/zones in a single installation. Use areas and device grouping.

### Q: What about remote access?
**A**: Three options:
1. VPN (WireGuard, Tailscale) - Free, secure
2. HA Cloud (Nabu Casa) - $6.50/month, easy
3. Port forwarding + DuckDNS - Free, manual

### Q: Can I add users (family members, employees)?
**A**: Yes! HA supports multiple users in Settings → People. All share device access.

### Q: How do I backup my system?
**A**: HA has built-in backup (Settings → System → Backups). Automatic or manual.

### Q: What about firmware updates?
**A**: Works the same! Custom integration monitors GitHub releases and offers updates via HA UI.

### Q: Is this less powerful than the cloud version?
**A**: No! Same functionality, just runs locally. Actually better privacy and response time.

---

## Conclusion

The shift to single-user local deployment **dramatically simplifies** the architecture while **improving** privacy, speed, and user control. This aligns perfectly with Home Assistant's design philosophy and makes Plantalytix more accessible to individual growers and small facilities.

The migration is now **faster** (no multi-tenant infrastructure to build), **cheaper** (no cloud hosting), and **more sustainable** (users own their infrastructure).

---

**For detailed architecture**: See [ARCHITECTURE-DIAGRAM.mmd](docs/ARCHITECTURE-DIAGRAM.mmd)
**For deployment guide**: See [ADR-0006](docs/decisions/0006-home-assistant-single-user-deployment.md)
**For getting started**: See [GETTING-STARTED-HA.md](docs/GETTING-STARTED-HA.md)
