# CV-Based Cycle Time Detection System

## Concept

Deploy fixed cameras at manufacturing stations to automatically measure cycle time, idle time, and throughput using computer vision. The system detects **state transitions** — part arrives, work begins, work ends, part leaves — rather than trying to understand complex human behavior. The math is simple: timestamp the transitions, calculate the deltas.

### Metrics Derived Per Station

| Metric | Derivation |
|--------|-----------|
| Cycle time | Timestamp delta: work start → work end |
| Idle time | Gap between consecutive cycles |
| Throughput | Completed cycles per hour/shift |
| Utilization % | Active time / total available time |
| Variability | Standard deviation of cycle times |
| Bottleneck ID | Station with highest average cycle time or most idle time |

---

## System Architecture

### Data Flow

```
Camera Stream (RTSP over LAN)
    │
    ▼
Frame Capture (OpenCV / GStreamer) — pull frames at 5-10 FPS
    │
    ▼
YOLO Inference — detect parts, operators, fixtures per frame
    │
    ▼
Object Tracking (ByteTrack) — maintain identity across frames
    │
    ▼
Zone Logic (Supervision PolygonZone) — is the detection inside the work area?
    │
    ▼
State Machine — per station: IDLE → PART_PRESENT → IN_CYCLE → CYCLE_COMPLETE
    │
    ▼
Event Logger — timestamp each transition, calculate cycle time
    │
    ▼
Time-Series Database (InfluxDB or TimescaleDB)
    │
    ▼
Dashboard (Grafana or custom web UI)
```

### Application Structure

```
cycle-time-system/
├── config/
│   ├── stations.yaml          # Station definitions, camera URLs, zone polygons
│   └── models/                # YOLO model weights per station type
├── src/
│   ├── main.py                # Entry point — orchestrates the pipeline
│   ├── capture/
│   │   └── stream.py          # Camera stream management (RTSP via OpenCV/GStreamer)
│   ├── detection/
│   │   └── detector.py        # YOLO inference wrapper (ultralytics)
│   ├── tracking/
│   │   └── tracker.py         # ByteTrack object tracking
│   ├── zones/
│   │   └── zone_manager.py    # ROI polygon definitions + point-in-polygon logic
│   ├── state/
│   │   └── state_machine.py   # Per-station state machine
│   ├── events/
│   │   └── event_logger.py    # Cycle event recording + DB writes
│   ├── api/
│   │   └── server.py          # REST API (FastAPI) for config, live status, metrics
│   └── dashboard/
│       └── web_ui.py          # Optional lightweight web dashboard
├── tools/
│   ├── zone_drawer.py         # GUI tool to draw ROI polygons on camera feed
│   ├── label_helper.py        # Auto-labeling pipeline (see companion doc)
│   └── model_trainer.py       # YOLO fine-tuning script
├── docker-compose.yaml
└── requirements.txt
```

---

## Module Details

### `stream.py` — Frame Capture

Connects to IP cameras via RTSP and pulls frames at a configurable rate.

**Key design decisions:**
- Pull at 5-10 FPS, not 30. Cycle times are measured in seconds or minutes — high frame rates waste compute for no gain.
- Downscale frames to 640x640 before inference (YOLO's native input size) to save memory and increase throughput.
- Handle reconnection on stream drop with exponential backoff.
- One thread per camera stream, feeding frames into a shared queue.

**Frame rate math:** At 5 FPS, with a 5-frame debounce, the minimum detectable state transition is 1 second. For cycle times measured in tens of seconds to minutes, this is more than sufficient.

### `detector.py` — YOLO Inference

Wraps the ultralytics library for inference.

**What to detect per station (choose the simplest set that works):**
- **Minimum:** just `part` — detect the workpiece arriving and leaving the zone
- **Better:** `part` + `empty_fixture` — distinguish between "fixture loaded" and "fixture empty"
- **Optional:** `operator_hands` — detect hands in the work zone for hand-activity-based cycle detection

**Model selection:**
- YOLOv8s (small) is the recommended starting point — good balance of speed and accuracy
- On an RTX 4090, YOLOv8s runs at ~200 FPS, meaning one GPU handles ~40 cameras at 5 FPS each
- Use pretrained COCO weights if detecting "person" is sufficient; fine-tune only if detecting specific parts

### `tracker.py` — Object Tracking

ByteTrack (built into ultralytics) maintains object identity across frames. This matters because:

- YOLO detects independently per frame — it doesn't know that the "part" in frame 100 is the same "part" in frame 101
- Tracking bridges brief detection gaps (e.g., operator's arm temporarily blocks the camera's view of the part)
- Tracking assigns a consistent ID, so you can count unique parts entering/leaving the zone

### `zone_manager.py` — Region of Interest Logic

Each station has one or more polygon zones defined in the station config. Zones are drawn once per camera position using a GUI tool (`zone_drawer.py`) and stored in `stations.yaml`.

**Zone types:**
- `WORK_ZONE`: the primary area where the part sits during the cycle — this is the only zone you strictly need
- `ENTRY_ZONE` (optional): where parts arrive — useful for detecting flow direction
- `EXIT_ZONE` (optional): where parts leave — useful for confirming cycle completion vs. part rejection

**Logic:** For each detection in each frame, check if the detection's center point falls inside the polygon. Supervision's `PolygonZone` class handles this efficiently.

**Configuration example (stations.yaml):**
```yaml
stations:
  - id: "station_04"
    name: "Final Assembly"
    camera_url: "rtsp://192.168.1.104:554/stream1"
    model: "models/assembly_v2.pt"
    fps: 5
    zones:
      work_zone:
        polygon: [[120, 80], [520, 80], [520, 400], [120, 400]]
      entry_zone:
        polygon: [[0, 200], [120, 200], [120, 400], [0, 400]]
    state_machine:
      debounce_frames: 5
      min_cycle_seconds: 10
      max_cycle_seconds: 300
```

### `state_machine.py` — The Core Logic

Each station runs an independent state machine that tracks the current phase of the work cycle.

**States:**
```
IDLE            No part detected in the work zone
PART_ARRIVED    Part detected, waiting for debounce confirmation
IN_CYCLE        Part confirmed present, cycle timer running
PART_DEPARTING  Part no longer detected, waiting for debounce confirmation
CYCLE_COMPLETE  Part confirmed gone, cycle time calculated
```

**Transitions:**
```
IDLE → PART_ARRIVED:
  Trigger: part detected in WORK_ZONE
  Action: start debounce counter

PART_ARRIVED → IN_CYCLE:
  Trigger: part detected for N consecutive frames (debounce satisfied)
  Action: record cycle_start timestamp

PART_ARRIVED → IDLE:
  Trigger: part disappears before debounce threshold
  Action: reset (was a false positive or transient detection)

IN_CYCLE → PART_DEPARTING:
  Trigger: part not detected in WORK_ZONE
  Action: start departure debounce counter

PART_DEPARTING → IN_CYCLE:
  Trigger: part re-detected before departure debounce threshold
  Action: reset departure counter (was a brief occlusion, not a real departure)

PART_DEPARTING → CYCLE_COMPLETE:
  Trigger: part absent for N consecutive frames (departure debounce satisfied)
  Action: record cycle_end timestamp, calculate cycle_time, log event

CYCLE_COMPLETE → IDLE:
  Trigger: automatic
  Action: record idle_start timestamp

CYCLE_COMPLETE → PART_ARRIVED:
  Trigger: new part detected immediately (back-to-back cycles)
  Action: record idle_time = 0, begin new cycle debounce
```

**Debouncing is critical.** Without it, a single frame where the part is occluded (operator's arm passes in front of camera) would falsely end the cycle. Recommended starting values:
- Arrival debounce: 5 frames at 5 FPS = 1 second of consistent detection
- Departure debounce: 5-10 frames = 1-2 seconds of consistent absence

**Guard rails:**
- `min_cycle_seconds`: If a "cycle" completes in less than this, discard it as noise (configurable per station)
- `max_cycle_seconds`: If a cycle exceeds this, flag it as an anomaly (part may be stuck, camera may be obstructed)

### `event_logger.py` — Recording Cycles

Each completed cycle generates an event record:

```json
{
  "station_id": "station_04",
  "cycle_number": 1247,
  "cycle_start": "2026-02-13T08:14:32.100Z",
  "cycle_end": "2026-02-13T08:15:47.300Z",
  "cycle_time_seconds": 75.2,
  "idle_before_seconds": 12.4,
  "avg_detection_confidence": 0.94,
  "part_tracking_id": "track_0042",
  "flagged": false,
  "flag_reason": null
}
```

Events are written to a time-series database (InfluxDB or TimescaleDB) for querying and visualization.

### `server.py` — REST API

FastAPI-based API for system management and data access.

**Endpoints:**
- `GET /stations` — list all stations and their current state
- `GET /stations/{id}/status` — live status (current state, current cycle elapsed time)
- `GET /stations/{id}/metrics` — cycle time stats (avg, min, max, std dev, throughput)
- `GET /stations/{id}/events` — recent cycle events with filtering
- `POST /stations/{id}/zones` — update zone polygons
- `GET /health` — system health (camera connectivity, GPU utilization, inference latency)

---

## Infrastructure: Centralized vs. Edge

### Recommended: Centralized On-Prem GPU Server

A single workstation with an NVIDIA RTX 4090 can handle 30-40 cameras at 5 FPS. This is the simplest architecture for a single-facility deployment.

**Why centralized works for this use case:**
- All cameras are on the same LAN — RTSP video traversal is straightforward
- One server, one model, one deployment — simplest to manage and update
- A single 1080p RTSP stream is ~4-8 Mbps; 20 cameras = ~80-160 Mbps, well within gigabit LAN
- One point of management vs. N edge devices to maintain and update

**Server spec:**
- CPU: Modern 8+ core (Intel i7/i9 or AMD Ryzen 7/9)
- GPU: NVIDIA RTX 4090 (24GB VRAM) — handles ~40 cameras at 5 FPS with YOLOv8s
- RAM: 32-64 GB
- Storage: 1TB SSD (system + models + event database)
- OS: Ubuntu 22.04 or 24.04
- Estimated cost: $4,000-6,000

**Scaling:**
- 40+ cameras: add a second GPU (RTX 4090) or upgrade to A4000/L40S
- Multiple facilities: deploy one server per facility, aggregate data to a central dashboard

### Alternative: Edge with Jetson Orin

Use this if cameras are spread across multiple buildings with no shared LAN, or if network bandwidth is genuinely constrained.

- Jetson Orin Nano ($250): handles ~6 cameras at 5 FPS with YOLOv8n
- Jetson Orin NX ($500-700): handles ~12 cameras at 5 FPS with YOLOv8s
- Tradeoff: more devices to manage, model updates must be pushed to each device

### Cloud Processing

Possible but generally not recommended for this use case. Sending 20+ video streams to the cloud introduces latency, bandwidth costs, and data residency concerns. Use cloud only for proof of concept or if existing infrastructure strongly favors it.

---

## Detection Strategies: Simplest That Works

Not every station requires a custom-trained YOLO model. Evaluate each station and pick the simplest approach that reliably detects cycle transitions.

### Strategy 1: Background Subtraction (No ML)

If the camera is fixed and the fixture is always visible, basic frame differencing can detect when "something new" appears in the work zone.

**How:** OpenCV's `BackgroundSubtractorMOG2` learns the static background and highlights changes. Threshold the change within the work zone polygon — significant change = part present.

**Pros:** Zero training, zero ML, runs on a CPU. **Cons:** Sensitive to lighting changes, can't distinguish part from operator's arm, no classification.

**Best for:** Stations where the part is large, always in the same spot, and clearly different from the background.

### Strategy 2: Pretrained YOLO Person Detection + Zone Logic (No Custom Training)

YOLOv8 pretrained on COCO already detects "person" with 90%+ accuracy. If the cycle is defined by "operator is present at the station," this works out of the box.

**How:** Run pretrained YOLOv8, filter for class "person," check if any person detection falls inside the work zone. Person in zone = cycle active.

**Pros:** Zero custom training, high accuracy for person detection, immediate deployment. **Cons:** Doesn't detect the actual part, can't distinguish "standing at station" from "actively working."

**Best for:** Stations where operator presence is a reliable proxy for active work.

### Strategy 3: Custom-Trained YOLO Part Detection + Zone Logic

Train YOLO to detect the specific parts, fixtures, or tool states at each station type.

**How:** Collect and label images (see companion auto-labeling document), fine-tune YOLOv8s, deploy the custom model.

**Pros:** Most accurate, detects the actual part, can distinguish multiple part types or states. **Cons:** Requires training data and labeling effort per station type.

**Best for:** Stations where part presence is the true indicator, or where multiple part types need to be distinguished.

### Strategy 4: Hybrid (Person + Part)

Combine pretrained person detection with custom part detection. Both must be true for cycle to be active: operator is present AND part is in the fixture.

**Pros:** Most robust — avoids false positives from either source alone. **Cons:** Slightly more complex logic, requires custom training for part detection.

**Best for:** Stations where you want high confidence and can invest in the training pipeline.

---

## The Core Processing Loop

This is what the main processing loop looks like using Supervision and ultralytics. The entire detection pipeline is concise:

```python
import supervision as sv
from ultralytics import YOLO
import cv2

# Load model and configure
model = YOLO("models/station_assembly_v2.pt")
tracker = sv.ByteTrack()

# Define work zone polygon (from stations.yaml config)
work_zone_polygon = np.array([[120, 80], [520, 80], [520, 400], [120, 400]])
zone = sv.PolygonZone(polygon=work_zone_polygon)

# State machine instance for this station
state_machine = CycleStateMachine(
    station_id="station_04",
    debounce_frames=5,
    min_cycle_seconds=10
)

# Main loop
cap = cv2.VideoCapture("rtsp://192.168.1.104:554/stream1")

while True:
    ret, frame = cap.read()
    if not ret:
        reconnect()
        continue

    # Detect objects
    results = model(frame, verbose=False)[0]
    detections = sv.Detections.from_ultralytics(results)

    # Track across frames
    detections = tracker.update_with_detections(detections)

    # Check which detections are inside the work zone
    in_zone = zone.trigger(detections)
    parts_in_zone = detections[in_zone]

    # Update state machine
    event = state_machine.update(
        parts_detected=len(parts_in_zone) > 0,
        timestamp=time.time(),
        avg_confidence=parts_in_zone.confidence.mean() if len(parts_in_zone) > 0 else 0
    )

    # If a cycle completed, log it
    if event:
        event_logger.log(event)
```

---

## Zone Configuration Tool

A one-time setup tool per camera. Displays the live camera feed and lets the user draw polygons to define zones.

**Implementation:** OpenCV window with mouse callbacks for polygon drawing. The user clicks to define vertices, and the tool saves the polygon coordinates to `stations.yaml`.

**Usage flow:**
1. Run `python tools/zone_drawer.py --camera rtsp://192.168.1.104:554/stream1`
2. Camera feed appears in a window
3. Click to draw the work zone polygon
4. Press Enter to confirm, Escape to redo
5. Coordinates are saved to config

This only needs to be done once per camera, and again if the camera is moved.

---

## Dashboard and Analytics

### Grafana + InfluxDB (Recommended for V1)

InfluxDB stores cycle events as time-series data. Grafana queries InfluxDB and displays dashboards.

**Dashboard panels:**
- **Cycle time trend** — line chart of cycle times over time per station, with target line overlaid
- **Station comparison** — bar chart of average cycle time across all stations (identifies bottlenecks)
- **Throughput** — parts per hour, rolling average
- **Utilization heatmap** — station utilization % by hour of day (identifies patterns)
- **Alerts panel** — stations currently exceeding cycle time thresholds or with camera issues
- **Idle time breakdown** — where time is being lost between cycles

### Alerting

Configure Grafana alerts for:
- Cycle time exceeding target by more than X%
- Extended idle time (no cycle detected for Y minutes)
- Camera stream down (no frames received)
- Detection confidence dropping below threshold (model degradation or environmental change)

---

## Execution Phases

### Phase 1: Proof of Concept (2-4 weeks)

**Goal:** Demonstrate cycle time detection on one station.

- Select one station with clear, repetitive cycles
- Mount a camera (USB webcam is fine for POC)
- Record 30 minutes of operation
- Determine detection strategy (pretrained person detection may be sufficient)
- If custom training needed: label ~200 images using auto-labeling pipeline, fine-tune YOLOv8s
- Build the core pipeline: capture → detect → zone → state machine → log to CSV
- Validate: compare CV-measured cycle times against manually stopwatch-timed cycles
- **Success criteria:** CV cycle times within ±5% of manual measurements

### Phase 2: Multi-Station Pilot (4-8 weeks)

**Goal:** Scale to 5-10 stations with a centralized server and live dashboard.

- Procure GPU server and IP cameras
- Deploy cameras at 5-10 stations
- Train/select models for each station type
- Build zone configuration tool
- Deploy InfluxDB + Grafana
- Build dashboard with cycle time trends, throughput, and bottleneck identification
- Run in parallel with existing measurement methods for 2+ weeks
- **Success criteria:** System runs reliably, data matches manual tracking

### Phase 3: Production Deployment (8-12 weeks)

**Goal:** Full floor coverage with alerting and system integration.

- Scale to all target stations
- Add alerting (cycle time anomalies, extended idle, camera health)
- Integrate with MES/ERP if applicable
- Build model retraining pipeline for handling new parts or station changes
- Harden: auto-reconnect, health monitoring, log rotation, backup
- Document and train production engineering team
- **Success criteria:** System is the primary source of truth for cycle time data

---

## Hardware Bill of Materials (Centralized Approach)

| Item | Specification | Est. Cost | Notes |
|------|--------------|-----------|-------|
| GPU Server | Workstation, RTX 4090 24GB | $4,000-6,000 | Handles ~40 cameras at 5 FPS |
| IP Cameras | PoE 1080p with RTSP (Hikvision, Reolink, or Axis) | $100-300 each | 1080p is sufficient; avoid 4K (wastes bandwidth and compute) |
| PoE Switch | Managed PoE+ switch, enough ports for cameras | $200-500 | Powers cameras over ethernet, no separate power runs |
| Camera Mounts | Overhead or angled mounts, ethernet cable runs | $50-100/station | Overhead preferred to minimize occlusion |
| Network | Gigabit LAN between cameras and server | Existing | 20 cameras at 1080p RTSP ≈ 80-160 Mbps |

**Total for a 10-station pilot: ~$6,000-10,000**

---

## Risk Mitigation

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Lighting variation across shifts | Detection accuracy drops | Include all lighting conditions in training data; add supplemental lighting if needed |
| Operator occludes part | False cycle-end detection | Overhead camera mount; departure debounce (5-10 frames); ByteTrack bridges short gaps |
| Model drift (new parts, fixture changes) | Missed detections over time | Monitor avg confidence per station; alert on drops; periodic retraining schedule |
| Camera failure | Station goes dark | Health check on stream; alert on loss; store last known state; prioritize reconnection |
| Network congestion (centralized) | Dropped frames | QoS for camera VLAN; local frame buffer; consider MJPEG fallback if bandwidth is tight |
| Similar-looking parts at same station | Misclassification between part types | Train with part-specific classes; or use zone-only logic if type doesn't matter for cycle time |
| Operator behavior varies widely | State machine thresholds don't generalize | Per-station config for debounce and min/max cycle time; tune based on observed data |

---

## Technology Stack Summary

| Component | Technology | Why |
|-----------|-----------|-----|
| Object Detection | ultralytics YOLOv8/v11 | Industry standard, fast, well-documented |
| Object Tracking | ByteTrack (via ultralytics) | Built-in, handles occlusion well |
| Zone Logic | Supervision (Roboflow) | Production-ready polygon zones, counting, annotation |
| Frame Capture | OpenCV + GStreamer | Reliable RTSP handling |
| State Machine | Custom Python | Simple, fully configurable per station |
| Time-Series DB | InfluxDB or TimescaleDB | Purpose-built for time-stamped event data |
| Dashboard | Grafana | Flexible, connects to InfluxDB natively, alerting built in |
| API Layer | FastAPI | Lightweight, async, auto-generates API docs |
| Containerization | Docker Compose | Bundles app + DB + dashboard into one deployment |
| Auto-Labeling | Grounding DINO + Roboflow (see companion doc) | Minimizes manual labeling effort |
