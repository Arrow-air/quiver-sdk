# SPEC.md — Arrow/Quiver SDK

**Status:** Draft  
**Date:** 2026-03-30  
**Authors:** 21stCenturyAlex, Vector  
**Input documents:** Thomas's SDK planning gist (2026-03-22), QuiverHub Architecture Reference (2026-02-21), QuiverHub Developer Guide (2026-02-12), RPLidar MVP note (2025-12-03), SDK Information Note (2026-01-25)

---

## 1. Purpose

The Arrow/Quiver SDK is a Python software stack that lets developers build applications on the Quiver UAV platform. It consists of two layers:

- **`quiver-sdk`** — Core hardware abstraction. Protocol-direct Python library for controlling the vehicle, camera, and attachment devices. Runs on the onboard Raspberry Pi or any machine on the Quiver network. No internet required.

- **`quiver-hub`** — Platform integration layer. Connects `quiver-sdk` to QuiverHub for telemetry forwarding, remote job execution, and payload data streaming. Depends on `quiver-sdk`; requires network connectivity to Hub.

These are two separate packages. A developer building onboard autonomy only needs `quiver-sdk`. A developer building Hub-integrated applications installs both.

---

## 2. Design Principles

1. **Offline-first.** `quiver-sdk` never requires internet. Drones fly in fields.
2. **ArduCopter-native.** pymavlink is the MAVLink layer. Not MAVSDK. Quiver runs ArduCopter; pymavlink is maintained by the same team and exposes all ArduPilot-specific messages.
3. **One pip install.** `pip install quiver-sdk` is the entire onboarding for core use. No daemon, no subprocess, no arch-specific binary.
4. **Device-oriented, not port-oriented.** The SDK addresses devices by IP or CAN node ID, not by physical attachment port. The developer knows what they plugged in and where.
5. **Simple by default, powerful when needed.** `Vehicle()` just works. `drone.mav` exposes the raw pymavlink connection when you need it.
6. **Hub integration is optional.** `quiver-hub` adds Hub connectivity on top of the SDK. The core library has no awareness of QuiverHub.
7. **Separation of concerns.** `quiver-sdk` owns protocol implementation. `quiver-hub` owns data transport to the cloud. Neither bleeds into the other.
8. **Escape hatches everywhere.** Every abstraction exposes the underlying object for developers who need to go deeper.

---

## 3. System Architecture

```
┌─────────────────────────────────────────┐
│         Applications (any language)      │  ← Web, mobile, scripts
├─────────────────────────────────────────┤
│              QuiverHub                   │
│  ┌──────────┐  ┌───────┐  ┌──────────┐ │
│  │ REST API │  │  WS   │  │ App Store│ │  ← Node.js server
│  └──────────┘  └───────┘  └──────────┘ │
├─────────────────────────────────────────┤
│           quiver-hub (Python)            │  ← Companion layer
│  telemetry_forwarder  job_runner        │    imports quiver-sdk
│  payload_streamer     file_sync         │
├─────────────────────────────────────────┤
│           quiver-sdk (Python)            │  ← Core layer
│  quiver.vehicle  quiver.camera          │    protocol-direct
│  quiver.attachment                      │    offline-capable
├─────────────────────────────────────────┤
│  pymavlink │ Siyi binary │ python-can   │  ← Protocol libs
├─────────────────────────────────────────┤
│  ArduCopter FC │ Siyi A8 │ CAN devices  │  ← Hardware
└─────────────────────────────────────────┘
```

---

## 4. Network Topology

All devices share a flat `192.168.144.0/24` network, dictated by Siyi's hardcoded addressing. Two Gigablox Nano switches (unmanaged L2) connect everything.

### Reserved addresses (do not use)

| IP | Device |
|---|---|
| 192.168.144.11 | Siyi air unit |
| 192.168.144.12 | Siyi ground unit |
| 192.168.144.20 | Android GCS (Siyi reserved) |
| 192.168.144.25 | Siyi A8 Mini camera |
| 192.168.144.60 | Siyi camera reserved |

### Quiver conventions

| Range | Purpose |
|---|---|
| .50 | Raspberry Pi (companion computer) |
| .51 | Flight controller (if on ETH) |
| .100–.199 | Attachment devices (developer-assigned static) |
| .200–.254 | Ground station laptops / dev machines |

### Attachment port mapping

Each physical port has its own Ethernet switch port. IP assignment is static, developer-managed in the `.100–.199` range. Recommended defaults to minimize collisions:

| Port | Connector | Signal pin | Recommended IP |
|---|---|---|---|
| Bottom | J31 | FMU_CH1 | 192.168.144.100 |
| Side 1 | J29 | FMU_CH7 | 192.168.144.101 |
| Side 2 | J30 | FMU_CH8 | 192.168.144.102 |

> ⚠️ **Note:** Earlier QuiverHub docs assigned C1=.11, C2=.12, C3=.13. Those addresses conflict with Siyi hardware. The correct range for attachment devices is `.100–.199`.

---

## 5. Package: `quiver-sdk`

### Installation

```bash
pip install quiver-sdk
# Optional: OpenCV for cam.get_frame()
pip install quiver-sdk[cv]
```

### Dependencies

| Package | Purpose |
|---|---|
| pymavlink | MAVLink / ArduPilot communication |
| python-can | CAN bus (SocketCAN) |
| numpy | Frame data, math |
| opencv-python | Video frame capture (optional) |

### Repository layout

```
quiver-sdk/
├── quiver/
│   ├── __init__.py          # Public API: Vehicle, Camera, Attachment
│   ├── vehicle.py           # Flight controller (pymavlink)
│   ├── camera.py            # Siyi camera/gimbal control
│   ├── attachment.py        # Device communication (ETH + CAN)
│   └── _internal/
│       ├── mavlink.py       # pymavlink connection + async bridge
│       ├── siyi.py          # Siyi binary protocol implementation
│       └── can.py           # CAN bus helpers
├── examples/
│   ├── hello.py
│   ├── takeoff_land.py
│   ├── take_photo.py
│   ├── spray_mission.py
│   └── monitor_battery.py
├── tests/
├── pyproject.toml
├── README.md
└── SPEC.md
```

---

## 6. Module: `quiver.vehicle`

Wraps pymavlink. Provides a clean Pythonic interface to the ArduPilot flight controller.

### Connection

```python
from quiver import Vehicle

drone = Vehicle()                             # auto-connect (udp:127.0.0.1:14550)
drone = Vehicle("udp:192.168.144.50:14550")  # explicit
```

### State (read-only)

```python
drone.position           # (lat, lon, alt_msl)
drone.position_relative  # (north, east, down) from home
drone.altitude           # meters AGL
drone.heading            # degrees
drone.groundspeed        # m/s
drone.airspeed           # m/s
drone.attitude           # (roll, pitch, yaw) degrees
drone.battery            # Battery(voltage, current, remaining_pct)
drone.gps                # GPS(fix_type, satellites, hdop)
drone.mode               # "STABILIZE", "GUIDED", "AUTO", etc.
drone.armed              # bool
drone.is_flying          # bool
drone.ekf_ok             # bool
drone.home               # (lat, lon, alt)
drone.uptime             # seconds since boot
drone.firmware_version
drone.board_type         # "CubeOrange", etc.
drone.vehicle_type       # "ArduCopter"
```

### Commands

```python
drone.arm()
drone.disarm()
drone.takeoff(altitude_m)
drone.land()
drone.rtl()
drone.set_mode("GUIDED")
drone.goto(lat, lon, alt)
drone.goto_relative(north, east, down)
drone.set_groundspeed(5.0)
drone.set_yaw(180)
drone.reboot()
```

### Mission

```python
drone.upload_mission(waypoints)
drone.start_mission()
drone.pause_mission()
drone.resume_mission()
drone.clear_mission()
drone.mission_progress   # (current_wp, total_wp)
```

### Params

```python
drone.params["ARMING_CHECK"]      # read
drone.params["ARMING_CHECK"] = 1  # write
```

### Logs

```python
drone.logs.list()
drone.logs.download(log_id, path)
drone.logs.download_latest(path)
drone.logs.delete(log_id)
drone.logs.delete_all()
```

### Preflight

```python
report = drone.preflight()
# → GPS fix: OK (12 sats, HDOP 0.8)
# → Battery: OK (25.2V, 98%)
# → EKF: OK
# → Compass: OK
# → RC: OK
# → Params: WARNING (ARMING_CHECK = 0)
drone.preflight_ok  # bool
```

### Attachment port control

```python
# Switched 12V — bottom port only, via SSR on relay 1
drone.relay.set(1, on=True)
drone.relay.get(1)

# Signal pins — FMU channels
drone.servos.set(1, 1500)   # bottom (FMU_CH1)
drone.servos.set(7, 2000)   # side 1 (FMU_CH7)
drone.servos.set(8, 1000)   # side 2 (FMU_CH8)
```

### Sensors

```python
drone.imu           # (accel_xyz, gyro_xyz)
drone.baro          # (pressure, temperature, altitude)
drone.rangefinder   # altitude AGL
drone.vibration     # (x, y, z, clip_count)
drone.rc.channels   # {1: 1500, 2: 1500, ...}
drone.rc.connected  # bool
drone.messages      # recent statustext
drone.errors        # filtered to errors/warnings
```

### Events / callbacks

```python
drone.on("armed", callback)
drone.on("mode_changed", callback)
drone.on("waypoint_reached", callback)
drone.on("low_battery", callback)
drone.on("failsafe", callback)
drone.on("statustext", callback)
drone.on("connected", callback)
drone.on("disconnected", callback)
```

### Escape hatch

```python
drone.mav  # raw pymavlink connection
drone.mav.command_long_send(...)
```

---

## 7. Module: `quiver.camera`

Implements Siyi's binary SDK protocol directly. ~20 command IDs, binary framing (STX 0x6655 + CRC16). Developers never see raw bytes.

### Connection

```python
from quiver import Camera

cam = Camera()                    # defaults to 192.168.144.25
cam = Camera("192.168.144.25")    # explicit
```

### Gimbal

```python
cam.set_gimbal(pitch=-45, yaw=0)   # degrees
cam.set_pitch(pitch)
cam.set_yaw(yaw)
cam.rotate(yaw_speed, pitch_speed) # -100 to 100
cam.center()
cam.look_down()
cam.gimbal_attitude   # GimbalAttitude(yaw, pitch, roll)
cam.gimbal_velocity   # (yaw_vel, pitch_vel, roll_vel)
cam.set_mode("lock" | "follow" | "fpv")
cam.mode
```

### Capture

```python
cam.take_photo()
cam.start_recording()
cam.stop_recording()
cam.is_recording
cam.tf_card_ok
```

### Zoom

```python
cam.zoom_in()
cam.zoom_out()
cam.zoom_stop()
cam.set_zoom(4.5)
cam.zoom_level
cam.zoom_max
```

### Stream

```python
cam.stream_url         # RTSP URL
cam.get_frame()        # numpy array (requires opencv)
```

### Info

```python
cam.model
cam.firmware_version
cam.hardware_id
cam.hdr_enabled
cam.mounting_direction
```

### Events

```python
cam.on("photo_taken", callback)
cam.on("recording_started", callback)
cam.on("tf_card_error", callback)
```

**Implementation notes:**
- Angle values in protocol are 10× actual degrees; SDK converts internally
- Camera takes ~30s to boot; `firmware_version` returns `0.0.0` during startup
- `get_frame()` requires OpenCV (optional dependency)
- Supports UDP (primary), TCP with heartbeat, and TTL serial at 115200 baud

---

## 8. Module: `quiver.attachment`

Device-oriented, not port-oriented. SDK cannot auto-detect physical port from CAN ID or IP (shared bus, unmanaged switch). Developer knows what they plugged in and where.

### Ethernet device

```python
from quiver import Attachment

# Raw bytes
device = Attachment("192.168.144.100")
device.send(b"start")
device.on_receive(callback)
device.connected

# JSON protocol
sprayer = Attachment("192.168.144.100", protocol="json")
sprayer.send({"command": "start", "rate": 2.5})
sprayer.on("status", callback)
```

### CAN device

```python
sensor = Attachment(can_node=42)
sensor.send_can(msg_id, data)
sensor.on_can(msg_id, callback)
```

### Discovery

```python
devices = Attachment.scan()                  # ping sweep .100–.199 + CAN node scan
sprayer = Attachment.find("sprayer.local")   # mDNS
```

**Payload developer contract:**
- Assign a static IP in `192.168.144.100–.199`
- Or claim a unique CAN node ID on CAN2
- Optionally implement mDNS for name-based discovery
- Document your device's IP/node and protocol

---

## 9. Package: `quiver-hub`

Separate package. Imports `quiver-sdk`. Provides the companion-side integration with QuiverHub.

```bash
pip install quiver-hub  # pulls quiver-sdk as dependency
```

### Components

**Telemetry forwarder** — collects `Vehicle` state at configurable rate, POSTs to Hub REST endpoint.

```python
from quiver_hub import TelemetryForwarder

fwd = TelemetryForwarder(
    vehicle=drone,
    hub_url="https://hub.example.com",
    drone_id="quiver_001",
    api_key="...",
    rate_hz=10,
)
fwd.start()  # non-blocking, runs in background thread
```

**Job runner** — polls Hub for pending jobs, executes locally, reports status.

```python
from quiver_hub import JobRunner

runner = JobRunner(
    hub_url="https://hub.example.com",
    drone_id="quiver_001",
    api_key="...",
    poll_interval=5,
)
runner.register("restart_service", my_handler)
runner.start()
```

**Payload streamer** — forwards high-bandwidth sensor data to Hub via WebSocket.

```python
from quiver_hub import PayloadStreamer

streamer = PayloadStreamer(hub_url=..., drone_id=..., api_key=...)
streamer.stream("pointcloud", source=lidar_generator, rate_hz=10)
streamer.stream("camera_status", source=cam_status_generator, rate_hz=1)
streamer.start()
```

### Repository layout

```
quiver-hub/
├── quiver_hub/
│   ├── __init__.py
│   ├── telemetry.py     # TelemetryForwarder
│   ├── jobs.py          # JobRunner + built-in job handlers
│   ├── streaming.py     # PayloadStreamer (WebSocket)
│   └── client.py        # Hub HTTP/WS client (auth, retry, backoff)
├── systemd/
│   ├── quiver-telemetry.service
│   ├── quiver-jobs.service
│   └── quiver-streamer.service
├── examples/
└── pyproject.toml
```

---

## 10. Open Decisions

The following require explicit agreement before implementation begins:

| # | Decision | Options | Status |
|---|---|---|---|
| 1 | **MAVLink connection string default** | `udp:127.0.0.1:14550` vs `udpin://0.0.0.0:14540` | ❓ |
| 2 | **pymavlink async model** | Background thread + queue vs `asyncio.run_in_executor` | ❓ |
| 3 | **`quiver-hub` repo home** | `Arrow-air/quiver-hub` vs `Pan-Robotics/quiver-hub` | ❓ |
| 4 | **Attachment port recommended IPs** | Fixed .100/.101/.102 defaults vs fully open .100–.199 | ❓ |
| 5 | **Hub API versioning** | Path-based (`/v1/`) vs header-based | ❓ |
| 6 | **Offline telemetry buffering** | Drop frames vs persist to disk vs in-memory queue with cap | ❓ |
| 7 | **Camera module scope** | Siyi A8 only (now) vs multi-camera abstraction (future) | ❓ |
| 8 | **ROS2 compatibility layer** | Out of scope v1 vs optional `quiver.ros` submodule | ❓ |

---

## 11. Out of Scope (v1)

- Mission planning UI (leave to QGC / Mission Planner)
- Comms abstraction (Siyi bridge / WiFi / cellular are standard IP — use Python networking directly)
- Multi-vehicle coordination
- ROS2 integration
- Fleet management (Hub concern, not SDK concern)
- Windows support (Pi / Linux only)

---

## 12. Milestones

| Milestone | Deliverable |
|---|---|
| M1 | `quiver.vehicle` working on real hardware — connect, telemetry, arm/disarm, preflight |
| M2 | `quiver.camera` working on A8 Mini — gimbal, photo, stream URL |
| M3 | `quiver.attachment` — ETH device, CAN device, scan |
| M4 | `quiver-hub` telemetry forwarder replacing existing MAVSDK-based forwarder |
| M5 | `quiver-hub` job runner parity with existing `raspberry_pi_client.py` |
| M6 | Payload streamer (LiDAR reference pipeline ported) |
| M7 | PyPI publish, systemd templates, developer docs |
