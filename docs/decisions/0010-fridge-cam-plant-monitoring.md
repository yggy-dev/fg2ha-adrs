# Fridge Cam Plant Monitoring with AI Analysis

## Status

Proposed

## Context

Plantalytix grow chambers (fridges) contain plants that grow over extended periods (typically 8 months). Users need to monitor plant growth visually and detect potential issues like diseases or pests early.

**User Requirements:**

1. **Visual Monitoring**: Capture plant growth over time
2. **Timelapse Creation**: Generate timelapse videos showing 8 months of growth
3. **AI Analysis**: Use LLMs to analyze plant health from images
4. **Automated Alerts**: Detect diseases/pests automatically
5. **Live Streaming**: View camera feed in real-time on dashboard
6. **Configurable**: All parameters should be adjustable per chamber

**Technical Challenges:**

1. **Camera Power Management**: Camera should only run when lights are on
2. **Storage Optimization**: 8 months of images requires smart storage strategy
3. **Timelapse Generation**: Convert thousands of images into 10-minute video
4. **AI Integration**: Connect camera to LLM services for image analysis
5. **User Experience**: Simple interface for complex functionality

**Home Assistant Capabilities:**

- **Camera Integration**: Native support for IP cameras, USB cameras, ESP32-CAM
- **Automations**: Trigger actions based on state changes (light on/off)
- **Media Storage**: Built-in media folder, external storage support
- **Custom Components**: Can create custom integrations for specialized features
- **Lovelace Cards**: Custom UI cards for camera display and controls
- **Services**: Call external APIs (OpenAI Vision, Anthropic Claude, local LLMs)
- **Notifications**: Send alerts with images attached

## Decision

**We will implement Fridge Cam as a custom Home Assistant integration (`plantalytix_fridge_cam`) that orchestrates HA's camera platform, automation engine, and external AI services to provide comprehensive plant monitoring.**

### Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│  Camera Layer                                           │
├─────────────────────────────────────────────────────────┤
│  - ESP32-CAM (recommended) or USB/IP Camera             │
│  - Powered via smart socket (linked to lights)          │
│  - MJPEG stream to HA                                   │
└─────────────────────────────────────────────────────────┘
                           │
┌─────────────────────────────────────────────────────────┐
│  Home Assistant Core                                    │
├─────────────────────────────────────────────────────────┤
│  - Camera Integration (native)                          │
│  - Automations (snapshot triggers)                      │
│  - Media Storage (/media/fridge_cam/)                   │
└─────────────────────────────────────────────────────────┘
                           │
┌─────────────────────────────────────────────────────────┐
│  Custom Integration: plantalytix_fridge_cam             │
├─────────────────────────────────────────────────────────┤
│  1. Snapshot Coordinator                                │
│     - Monitors light state                              │
│     - Captures images at configured intervals           │
│     - Manages storage and rotation                      │
│                                                          │
│  2. Timelapse Generator                                 │
│     - Converts images to video (FFmpeg)                 │
│     - Configurable output duration                      │
│     - Progress tracking                                 │
│                                                          │
│  3. AI Analysis Service                                 │
│     - Sends images to LLM APIs                          │
│     - Manages prompt templates                          │
│     - Parses AI responses                               │
│     - Triggers alerts based on analysis                 │
│                                                          │
│  4. Dashboard Components                                │
│     - Live stream card                                  │
│     - AI chat interface                                 │
│     - Timelapse viewer                                  │
│     - Health status display                             │
└─────────────────────────────────────────────────────────┘
                           │
┌─────────────────────────────────────────────────────────┐
│  External Services (User Choice)                        │
├─────────────────────────────────────────────────────────┤
│  - OpenAI GPT-4 Vision API                              │
│  - Anthropic Claude 3.5 Sonnet (vision)                 │
│  - Google Gemini Vision                                 │
│  - Local LLM (Ollama with LLaVA/BakLLaVA)              │
│  - Custom endpoint                                      │
└─────────────────────────────────────────────────────────┘
```

### Key Design Decisions

#### 1. Camera Hardware Recommendation: ESP32-CAM

**Rationale:**
- Low cost (~$10)
- Integrates with existing ESP32 ecosystem
- Can be powered by same smart socket as lights
- ESPHome integration available
- MJPEG streaming to HA

**Alternative:** Any HA-compatible camera (USB, IP camera)

#### 2. Power Management via Automation

**Strategy:** Link camera power to light state

```yaml
# Automation: Camera follows light state
automation:
  - alias: "Fridge 001: Camera Power Control"
    trigger:
      - platform: state
        entity_id: light.fridge_001_grow_light
    action:
      - service: switch.turn_{{ trigger.to_state.state }}
        target:
          entity_id: switch.fridge_001_camera_power
```

**Benefits:**
- Camera only runs when needed
- Reduces power consumption
- Extends camera lifespan
- Ensures photos only when plants are illuminated

#### 3. Snapshot Strategy: Event-Driven + Scheduled

**Two-Tier Approach:**

**Tier 1: Regular Timelapse Snapshots**
- Trigger: Every X minutes (configurable, default: 30 min) when light is on
- Purpose: Timelapse video creation
- Storage: Permanent (until timelapse generated)

**Tier 2: AI Analysis Snapshots**
- Trigger: Daily at specific time (configurable, default: 12:00 PM)
- Purpose: Health monitoring and disease detection
- Storage: Keep last 30 days for trend analysis

**Storage Structure:**
```
/media/plantalytix_fridge_cam/
├── fridge_001/
│   ├── timelapse/
│   │   ├── 2025-01-15_10-30-00.jpg
│   │   ├── 2025-01-15_11-00-00.jpg
│   │   └── ...
│   ├── analysis/
│   │   ├── 2025-01-15_daily.jpg
│   │   ├── 2025-01-16_daily.jpg
│   │   └── ...
│   ├── videos/
│   │   ├── cycle_001_2025-01-15_to_2025-08-15.mp4
│   │   └── ...
│   └── config.yaml
```

#### 4. Timelapse Generation: On-Demand + Scheduled

**FFmpeg Integration:**

```python
# Simplified timelapse generation logic
ffmpeg -framerate 30 -pattern_type glob -i 'timelapse/*.jpg' \
  -c:v libx264 -pix_fmt yuv420p \
  -vf "scale=1920:1080" \
  output.mp4

# For 10-minute video from 8 months (240 days):
# 240 days × 16 hours light/day × 2 photos/hour = 7,680 images
# 10 minutes × 60 sec × 30 fps = 18,000 frames
# Each image shown for: 18,000 / 7,680 ≈ 2.34 frames
```

**Configuration:**
```yaml
plantalytix_fridge_cam:
  fridge_001:
    timelapse:
      output_duration: 600  # seconds (10 minutes)
      framerate: 30  # fps
      resolution: "1920x1080"
      codec: "libx264"
      quality: "high"  # high, medium, low
      generate_on_cycle_complete: true
      auto_cleanup_images: true  # Delete after video creation
```

#### 5. AI Analysis: Multi-LLM Support with Prompt Templates

**Prompt Template System:**

```yaml
# Built-in prompt templates
plantalytix_fridge_cam:
  ai_prompts:
    health_check:
      name: "General Health Assessment"
      prompt: |
        Analyze this plant image and provide:
        1. Overall health status (Excellent/Good/Fair/Poor)
        2. Leaf color and appearance
        3. Growth stage assessment
        4. Any visible issues or concerns
        5. Recommendations for care

        Please be specific and actionable in your response.

    disease_detection:
      name: "Disease Detection"
      prompt: |
        Examine this plant image for signs of disease:
        1. Identify any visible diseases or infections
        2. Look for discoloration, spots, wilting, or mold
        3. Assess severity (None/Mild/Moderate/Severe)
        4. Provide treatment recommendations
        5. Indicate if immediate action is required (Yes/No)

        Format response as:
        DISEASE: [name or "None detected"]
        SEVERITY: [level]
        URGENT: [Yes/No]
        TREATMENT: [recommendations]

    pest_detection:
      name: "Pest Detection"
      prompt: |
        Scan this plant image for pest infestations:
        1. Identify any visible pests or pest damage
        2. Look for insects, webs, holes in leaves, etc.
        3. Assess infestation level (None/Minor/Moderate/Severe)
        4. Provide pest control recommendations
        5. Indicate if immediate action is required (Yes/No)

        Format response as:
        PEST: [name or "None detected"]
        SEVERITY: [level]
        URGENT: [Yes/No]
        TREATMENT: [recommendations]

    growth_analysis:
      name: "Growth Progress"
      prompt: |
        Analyze plant growth based on this image:
        1. Estimate plant height/size
        2. Assess vegetative growth
        3. Check for flowering/budding if applicable
        4. Compare to expected growth stage for {days_since_start} days
        5. Provide growth optimization suggestions

        Plant type: {plant_type}
        Days since start: {days_since_start}
```

**LLM Service Configuration:**

```yaml
plantalytix_fridge_cam:
  ai_service:
    provider: "openai"  # openai, anthropic, google, ollama, custom

    # OpenAI Configuration
    openai:
      api_key: !secret openai_api_key
      model: "gpt-4-vision-preview"
      max_tokens: 500
      temperature: 0.3

    # Anthropic Configuration
    anthropic:
      api_key: !secret anthropic_api_key
      model: "claude-3-5-sonnet-20241022"
      max_tokens: 500

    # Local Ollama Configuration
    ollama:
      endpoint: "http://localhost:11434"
      model: "llava:13b"  # or bakllava

    # Fallback chain (try in order)
    fallback:
      - ollama
      - openai
      - anthropic
```

#### 6. User Interface: Custom Lovelace Cards

**Card 1: Live Camera View with AI Chat**

```yaml
type: custom:plantalytix-fridge-cam-card
entity: camera.fridge_001_camera
fridge_id: fridge_001
features:
  - live_view
  - snapshot_button
  - ai_chat
  - timelapse_viewer
layout:
  live_view:
    show: true
    aspect_ratio: "16:9"
  ai_chat:
    show: true
    position: "right"  # or "bottom"
    prompt_templates:
      - health_check
      - disease_detection
      - pest_detection
      - growth_analysis
      - custom
```

**Card 2: Health Status Dashboard**

```yaml
type: custom:plantalytix-plant-health-card
entity: sensor.fridge_001_plant_health
fridge_id: fridge_001
features:
  - health_score
  - disease_alerts
  - pest_alerts
  - growth_chart
  - last_analysis_image
  - historical_trends
display:
  health_score:
    show_gauge: true
    thresholds:
      excellent: 90
      good: 70
      fair: 50
      poor: 0
  alerts:
    show_count: true
    max_display: 5
  timeline:
    show_graph: true
    days: 30
```

**Card 3: Timelapse Manager**

```yaml
type: custom:plantalytix-timelapse-card
fridge_id: fridge_001
features:
  - video_player
  - generation_controls
  - progress_indicator
  - download_button
  - share_button
settings:
  auto_play: false
  loop: true
  show_date_overlay: true
```

#### 7. Automated Analysis & Alerting

**Daily Health Check Automation:**

```yaml
automation:
  - alias: "Fridge 001: Daily Plant Health Check"
    trigger:
      - platform: time
        at: "12:00:00"
    condition:
      - condition: state
        entity_id: light.fridge_001_grow_light
        state: "on"
    action:
      # Take snapshot
      - service: camera.snapshot
        target:
          entity_id: camera.fridge_001_camera
        data:
          filename: "/media/plantalytix_fridge_cam/fridge_001/analysis/{{ now().strftime('%Y-%m-%d') }}_daily.jpg"

      # Wait for snapshot to save
      - delay: "00:00:03"

      # Run AI analysis (disease detection)
      - service: plantalytix_fridge_cam.analyze_image
        data:
          fridge_id: fridge_001
          image_path: "/media/plantalytix_fridge_cam/fridge_001/analysis/{{ now().strftime('%Y-%m-%d') }}_daily.jpg"
          prompt_template: "disease_detection"
        response_variable: disease_analysis

      # Check for urgent issues
      - choose:
          - conditions:
              - condition: template
                value_template: "{{ 'URGENT: Yes' in disease_analysis.response }}"
            sequence:
              # Send notification with image
              - service: notify.mobile_app
                data:
                  title: "🚨 Urgent: Plant Disease Detected!"
                  message: "{{ disease_analysis.disease }}: {{ disease_analysis.treatment }}"
                  data:
                    image: "/media/plantalytix_fridge_cam/fridge_001/analysis/{{ now().strftime('%Y-%m-%d') }}_daily.jpg"
                    actions:
                      - action: "VIEW_DETAILS"
                        title: "View Full Analysis"
                      - action: "ACKNOWLEDGE"
                        title: "Acknowledge"

              # Update sensor
              - service: sensor.set_value
                target:
                  entity_id: sensor.fridge_001_plant_health_alert
                data:
                  value: "urgent"
```

**Weekly Growth Analysis:**

```yaml
automation:
  - alias: "Fridge 001: Weekly Growth Analysis"
    trigger:
      - platform: time
        at: "10:00:00"
      - platform: template
        value_template: "{{ now().weekday() == 0 }}"  # Monday
    action:
      - service: plantalytix_fridge_cam.analyze_image
        data:
          fridge_id: fridge_001
          prompt_template: "growth_analysis"
          variables:
            plant_type: "{{ states('sensor.fridge_001_plant_type') }}"
            days_since_start: "{{ (now() - states('sensor.fridge_001_grow_cycle_start') | as_datetime).days }}"

      # Log results
      - service: logbook.log
        data:
          name: "Plant Growth Analysis"
          message: "{{ analysis_result }}"
          entity_id: sensor.fridge_001_plant_health
```

## Consequences

### Positive

✅ **Comprehensive Monitoring**: Visual + AI analysis for complete plant health tracking

✅ **Automated Alerts**: Early detection of diseases/pests without manual inspection

✅ **Growth Documentation**: Timelapse videos provide shareable proof of growth cycles

✅ **Flexible AI**: Support for multiple LLM providers including local options

✅ **Power Efficient**: Camera only runs when needed (lights on)

✅ **HA Native**: Leverages existing HA features (camera, automation, notifications)

✅ **User-Friendly**: Simple dashboard interface for complex functionality

✅ **Configurable**: All parameters adjustable per fridge

✅ **Scalable**: Works for any number of fridges

✅ **Cost-Effective**: Can use cheap ESP32-CAM hardware

✅ **Privacy-Aware**: Option for local LLM (no cloud)

✅ **Historical Analysis**: Keep 30 days of images for trend detection

### Negative

❌ **Storage Requirements**: 8 months of images requires significant storage (~50-100GB per fridge)

❌ **AI API Costs**: Cloud LLM services charge per image analysis

❌ **Processing Load**: FFmpeg timelapse generation is CPU-intensive

❌ **Camera Hardware**: Requires additional hardware (ESP32-CAM or USB camera)

❌ **Network Bandwidth**: MJPEG streaming uses bandwidth on local network

❌ **Complexity**: Custom integration adds maintenance burden

❌ **AI Accuracy**: LLMs may not always correctly identify plant issues

### Risks

⚠️ **Storage Exhaustion**: Long-term operation could fill disk

⚠️ **AI Hallucinations**: LLMs might provide incorrect diagnoses

⚠️ **Camera Failures**: Hardware failures could interrupt monitoring

⚠️ **Light Sync Issues**: If light/camera desync, photos might be dark

⚠️ **FFmpeg Dependencies**: Requires FFmpeg installed on HA host

### Mitigation Strategies

1. **Storage Management**:
   - Automatic cleanup after timelapse generation
   - Configurable retention policies
   - Warning alerts at 80% disk usage

2. **AI Validation**:
   - Multiple prompt runs for critical decisions
   - User confirmation for urgent alerts
   - Display confidence scores when available

3. **Hardware Redundancy**:
   - Support multiple camera types
   - Fallback to manual inspection if camera fails
   - Health monitoring for camera entity

4. **Sync Verification**:
   - Check light state before snapshot
   - Image brightness analysis (reject dark images)
   - Retry mechanism

5. **Dependency Management**:
   - Docker image with FFmpeg pre-installed
   - Graceful degradation if FFmpeg unavailable
   - Alternative timelapse methods (client-side)

## Implementation Plan

### Phase 1: Basic Camera Integration (2 weeks)

**Week 1: Camera Setup**
1. Document ESP32-CAM setup with ESPHome
2. Create camera power automation (linked to lights)
3. Test MJPEG streaming to HA
4. Implement manual snapshot service

**Week 2: Storage & Automation**
1. Set up media storage structure
2. Create timelapse snapshot automation
3. Test long-term image capture (24-hour test)
4. Document storage requirements

### Phase 2: Timelapse Generation (2 weeks)

**Week 1: FFmpeg Integration**
1. Develop timelapse generation service
2. Test with sample image sets
3. Implement configurable parameters
4. Add progress tracking

**Week 2: UI & Controls**
1. Create timelapse viewer card
2. Add generation controls to dashboard
3. Test end-to-end workflow
4. Document user guide

### Phase 3: AI Analysis (3 weeks)

**Week 1: LLM Integration**
1. Implement OpenAI Vision API integration
2. Add Anthropic Claude support
3. Add local Ollama support
4. Create prompt template system

**Week 2: Analysis Automation**
1. Develop AI analysis service
2. Create response parsing logic
3. Implement automated health checks
4. Test with real plant images

**Week 3: Alert System**
1. Create alert automation
2. Implement notification system (with images)
3. Add health status sensors
4. Test urgent alert workflow

### Phase 4: Dashboard UI (2 weeks)

**Week 1: Custom Cards**
1. Develop fridge cam card (live view + AI chat)
2. Create plant health card
3. Build timelapse manager card
4. Implement prompt template selector

**Week 2: Polish & Testing**
1. UI/UX improvements
2. Mobile responsiveness
3. User testing
4. Documentation

### Phase 5: Production Deployment (1 week)

1. Final testing with real grow cycle
2. Performance optimization
3. Documentation completion
4. Release v1.0

**Total Estimated Timeline**: 10 weeks

## Configuration Examples

### Fridge Cam Configuration

```yaml
# configuration.yaml
plantalytix_fridge_cam:
  fridges:
    - fridge_id: fridge_001
      name: "Main Growing Chamber"
      camera_entity: camera.fridge_001_camera
      light_entity: light.fridge_001_grow_light
      camera_power_entity: switch.fridge_001_camera_power

      # Timelapse settings
      timelapse:
        enabled: true
        interval: 1800  # 30 minutes
        output_duration: 600  # 10 minutes
        resolution: "1920x1080"
        framerate: 30
        codec: "libx264"
        quality: "high"
        storage_path: "/media/plantalytix_fridge_cam/fridge_001/timelapse"
        video_path: "/media/plantalytix_fridge_cam/fridge_001/videos"
        auto_cleanup: true
        generate_on_cycle_complete: true

      # AI Analysis settings
      ai_analysis:
        enabled: true
        provider: "openai"  # openai, anthropic, ollama
        daily_check_time: "12:00:00"
        weekly_check_day: "monday"
        weekly_check_time: "10:00:00"
        storage_path: "/media/plantalytix_fridge_cam/fridge_001/analysis"
        retention_days: 30

        # Alert thresholds
        alerts:
          disease_detected: true
          pest_detected: true
          poor_health: true
          urgent_issues_only: false

      # Plant information (for AI context)
      plant:
        type: "Cannabis Sativa"
        variety: "Northern Lights"
        expected_cycle_days: 240
        start_date: "2025-01-15"

# AI Service Configuration
plantalytix_ai:
  openai:
    api_key: !secret openai_api_key
    model: "gpt-4-vision-preview"
    max_tokens: 500
    temperature: 0.3

  anthropic:
    api_key: !secret anthropic_api_key
    model: "claude-3-5-sonnet-20241022"
    max_tokens: 500

  ollama:
    endpoint: "http://localhost:11434"
    model: "llava:13b"

  fallback_order:
    - ollama
    - openai
    - anthropic
```

### Dashboard Configuration

```yaml
# ui-lovelace.yaml
views:
  - title: Fridge Monitoring
    path: fridge-monitoring
    cards:
      # Live camera with AI chat
      - type: custom:plantalytix-fridge-cam-card
        entity: camera.fridge_001_camera
        fridge_id: fridge_001
        title: "Chamber 001 Live View"
        features:
          live_view:
            enabled: true
            aspect_ratio: "16:9"
            refresh_interval: 5
          ai_chat:
            enabled: true
            position: "right"
            width: "400px"
            prompt_templates:
              - health_check
              - disease_detection
              - pest_detection
              - growth_analysis
            allow_custom: true
          quick_actions:
            - snapshot
            - toggle_camera
            - fullscreen

      # Plant health status
      - type: custom:plantalytix-plant-health-card
        entity: sensor.fridge_001_plant_health
        fridge_id: fridge_001
        title: "Plant Health Status"
        display:
          health_score: true
          disease_alerts: true
          pest_alerts: true
          last_check: true
          growth_progress: true
          timeline_days: 30

      # Timelapse manager
      - type: custom:plantalytix-timelapse-card
        fridge_id: fridge_001
        title: "Growth Timelapse"
        features:
          video_player: true
          generation_controls: true
          download: true
          share: true
        settings:
          auto_play: false
          loop: true
          show_controls: true
          date_overlay: true

      # Quick stats
      - type: entities
        title: "Chamber Stats"
        entities:
          - entity: sensor.fridge_001_total_snapshots
            name: "Total Snapshots"
          - entity: sensor.fridge_001_storage_used
            name: "Storage Used"
          - entity: sensor.fridge_001_last_analysis
            name: "Last AI Analysis"
          - entity: sensor.fridge_001_days_in_cycle
            name: "Days in Cycle"
          - entity: binary_sensor.fridge_001_health_alert
            name: "Health Alert"
```

## Services

### Service: plantalytix_fridge_cam.take_snapshot

```yaml
service: plantalytix_fridge_cam.take_snapshot
data:
  fridge_id: fridge_001
  category: "timelapse"  # or "analysis" or "manual"
  filename_override: "special_moment.jpg"  # optional
```

### Service: plantalytix_fridge_cam.generate_timelapse

```yaml
service: plantalytix_fridge_cam.generate_timelapse
data:
  fridge_id: fridge_001
  start_date: "2025-01-15"  # optional
  end_date: "2025-08-15"    # optional
  output_duration: 600       # seconds
  framerate: 30
  resolution: "1920x1080"
  filename: "cycle_001.mp4"  # optional
```

### Service: plantalytix_fridge_cam.analyze_image

```yaml
service: plantalytix_fridge_cam.analyze_image
data:
  fridge_id: fridge_001
  image_path: "/media/plantalytix_fridge_cam/fridge_001/analysis/2025-01-15_daily.jpg"
  prompt_template: "disease_detection"  # or custom prompt
  prompt_custom: "Analyze this plant..."  # if template is "custom"
  variables:
    plant_type: "Cannabis"
    days_since_start: 45
  provider_override: "anthropic"  # optional
response_variable: analysis_result
```

### Service: plantalytix_fridge_cam.cleanup_storage

```yaml
service: plantalytix_fridge_cam.cleanup_storage
data:
  fridge_id: fridge_001
  cleanup_timelapse: false  # keep timelapse images
  cleanup_analysis: true    # delete old analysis images
  keep_days: 30
  cleanup_videos: false
```

## Hardware Recommendations

### Option 1: ESP32-CAM (Recommended)

**Specs:**
- Cost: ~$10
- Resolution: 2MP (1600x1200)
- Connection: WiFi
- Power: 5V via GPIO
- Integration: ESPHome

**ESPHome Configuration:**

```yaml
# fridge_001_camera.yaml
esphome:
  name: fridge-001-camera
  platform: ESP32
  board: esp32cam

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

logger:

api:
  encryption:
    key: !secret api_encryption_key

ota:
  password: !secret ota_password

esp32_camera:
  name: "Fridge 001 Camera"
  external_clock:
    pin: GPIO0
    frequency: 20MHz
  i2c_pins:
    sda: GPIO26
    scl: GPIO27
  data_pins: [GPIO5, GPIO18, GPIO19, GPIO21, GPIO36, GPIO39, GPIO34, GPIO35]
  vsync_pin: GPIO25
  href_pin: GPIO23
  pixel_clock_pin: GPIO22
  power_down_pin: GPIO32

  # Camera settings
  max_framerate: 10 fps
  idle_framerate: 0.1 fps
  resolution: 1600x1200
  jpeg_quality: 10

  # Enable when light is on (controlled externally)

# Binary sensor for camera status
binary_sensor:
  - platform: status
    name: "Fridge 001 Camera Status"
```

### Option 2: USB Webcam

**Specs:**
- Cost: ~$30-50
- Resolution: 1080p or higher
- Connection: USB to HA server
- Power: USB
- Integration: Generic Camera (motion, ffmpeg)

### Option 3: IP Camera

**Specs:**
- Cost: ~$50-100
- Resolution: 1080p or higher
- Connection: WiFi/Ethernet
- Power: PoE or adapter
- Integration: Generic Camera (RTSP, ONVIF)

## Alternatives Considered

### Alternative 1: HA Recorder for Continuous Video

**Description**: Use HA's built-in recorder to capture continuous video

**Pros**:
- Simple setup
- Built-in feature

**Cons**:
- Massive storage requirements (240 days × 24/7 video)
- Difficult to create timelapse from video
- No AI integration

**Verdict**: Rejected - storage prohibitive, doesn't meet timelapse requirements

### Alternative 2: External Timelapse Service

**Description**: Use external service like Cloudlapse or similar

**Pros**:
- Managed service
- No local processing

**Cons**:
- Monthly subscription costs
- Privacy concerns (cloud upload)
- Not integrated with HA
- No AI analysis

**Verdict**: Rejected - doesn't align with local-first philosophy

### Alternative 3: Camera with Built-in Timelapse

**Description**: Use camera with native timelapse feature

**Pros**:
- No HA integration needed
- Built-in feature

**Cons**:
- No HA dashboard integration
- No AI analysis
- Limited configurability
- No automation with grow lights

**Verdict**: Rejected - poor integration, missing key features

### Alternative 4: Frigate NVR

**Description**: Use Frigate add-on for camera management

**Pros**:
- Advanced camera features
- Object detection
- Event-based recording

**Cons**:
- Overkill for simple timelapse
- Continuous processing overhead
- Complex setup for simple use case

**Verdict**: Considered but deferred - may use Frigate for advanced users

## References

- [Home Assistant Camera Integration](https://www.home-assistant.io/integrations/camera/)
- [ESPHome Camera Component](https://esphome.io/components/esp32_camera.html)
- [FFmpeg Documentation](https://ffmpeg.org/documentation.html)
- [OpenAI Vision API](https://platform.openai.com/docs/guides/vision)
- [Anthropic Claude Vision](https://docs.anthropic.com/claude/docs/vision)
- [Ollama LLaVA Model](https://ollama.ai/library/llava)
- [HA Custom Cards Guide](https://developers.home-assistant.io/docs/frontend/custom-ui/lovelace-custom-card)

## Notes

**Storage Calculations:**

```
Assumptions:
- Image size: 500KB average (JPEG compressed)
- Light on: 16 hours/day
- Snapshot interval: 30 minutes
- Grow cycle: 240 days

Timelapse images:
240 days × 16 hours × 2 snapshots/hour = 7,680 images
7,680 × 500KB = 3.84 GB

Daily analysis images:
240 days × 1 image/day × 500KB = 120 MB

Total per fridge: ~4 GB for 8-month cycle

With 5 fridges: ~20 GB
With 10 fridges: ~40 GB

Recommendation: 100GB+ free storage for production
```

**AI Analysis Costs (Monthly):**

```
OpenAI GPT-4 Vision:
- Cost: $0.01 per image (high detail)
- Daily: 1 image/day × 30 days × $0.01 = $0.30/month/fridge
- 10 fridges: $3.00/month

Anthropic Claude 3.5 Sonnet:
- Cost: ~$0.008 per image
- Daily: 1 image/day × 30 days × $0.008 = $0.24/month/fridge
- 10 fridges: $2.40/month

Local Ollama (LLaVA):
- Cost: Free (hardware only)
- Requires: ~8GB VRAM for 13B model
```

**Performance Considerations:**

- Timelapse generation for 7,680 images takes ~5-10 minutes on modern CPU
- AI analysis per image: 2-10 seconds depending on provider
- MJPEG streaming: ~1-2 Mbps per camera
- Storage I/O: Should use SSD for media storage

**Decision Date**: 2025-11-15
**Review Date**: After Phase 1 completion (2 weeks)
