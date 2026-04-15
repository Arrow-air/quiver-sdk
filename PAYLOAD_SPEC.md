# PAYLOAD_SPEC.md — Quiver Payload Contract

**Status:** Draft  
**Date:** 2026-03-31  
**Authors:** 21stCenturyAlex, Vector  
**Companion document:** [`SPEC.md`](SPEC.md) — quiver-sdk architecture  
**Reference:** [`quiver-payload-template`](https://github.com/Pan-Robotics/quiver-payload-template/tree/dev) — reference implementation

---

## 1. Purpose

This document defines the contract that Quiver payload attachments must implement to be compatible with `quiver.attachment` in the quiver-sdk.

There are two physical interfaces on every Quiver attachment port:
- **Ethernet** — high-bandwidth, for cameras, LiDAR, compute modules, custom sensors
- **CAN bus** — low-bandwidth, for actuators, simple sensors, battery monitors

Each has a different contract. CAN payloads use DroneCAN/UAVCAN (DSDL-typed messages — already standardized). Ethernet payloads need an explicit format contract, which this document defines.

A payload that implements this contract gets:
- Automatic typed discovery via `Attachment.scan()`
- Typed stream callbacks with schema-validated data
- Automatic QuiverHub App Builder integration (Hub knows which widget to render)
- mDNS-based name resolution (`sprayer.quiver.local`)
- Clean SDK API: `device.stream("scan")`, `device.command("start")`

A payload that does not implement this contract still works — but only as a raw byte pipe (`device.raw`).

---

## 2. Hardware Interface

### Attachment Interface PCB

Each Quiver attachment port uses a custom 2-layer PCB (v1.2, July 2025) designed for quick-release payload integration.

**PCB specs:**
- Dimensions: 15.8 mm × 23.5 mm
- Thickness: 1.2 mm (non-standard — thinner than FR4 default)
- Mounting: 4× M2 holes
- KiCad files: [`Arrow-air/project-quiver`](https://github.com/Arrow-air/project-quiver/tree/main/task-grant-bounty/pt3/electronics/0003-Attachment-Interface-PCB)

The PCB carries three connectors. The same board layout is used on both the drone side and the payload side — populated differently depending on role:

| Ref | Part | Type | Role |
|---|---|---|---|
| J1 | Molex 2077601281 | 12-pin locking | Cable harness to Main PCB |
| J2 | Mill-Max 813-22-010-30-000101 | 10-pin spring-loaded pogo | Board-to-board mating (drone side) |
| J3 | Mill-Max 419-10-210-30-054000 | 10-pin surface-mount target | Board-to-board mating (payload side) |

J2 and J3 are a mating pair — the drone's J2 (pogo) contacts the payload's J3 (target) when the quick-release plate clicks in.

---

### J1 — Main harness connector (12-pin Molex 2077601281)

This is the locking cable connector. On the drone side it connects via harness to the Main PCB. On the payload side it connects to the payload's internal electronics. Pins 2+4 (12VSW) and pins 6+8 (GND) are doubled for current capacity.

| Pin | Net | Description |
|---|---|---|
| 1 | ETH_RX+ | Ethernet receive+ (drone perspective) |
| 2 | 12VSW | Switched 12V |
| 3 | ETH_RX- | Ethernet receive− |
| 4 | 12VSW | Switched 12V (doubled for current) |
| 5 | ETH_TX+ | Ethernet transmit+ (drone perspective) |
| 6 | GND | Ground |
| 7 | ETH_TX- | Ethernet transmit− |
| 8 | GND | Ground (doubled for current) |
| 9 | CAN1_N | CAN bus negative (CAN_L) |
| 10 | +12V | Always-on 12V |
| 11 | CAN1_P | CAN bus positive (CAN_H) |
| 12 | FMU_CHx | FMU signal (channel varies by port — see table below) |

Mating connector (payload cable side): Molex 204523-1201

---

### J2 / J3 — Board-to-board interface (10-pin)

J2 and J3 carry the same signals as J1, condensed to 10 pins (single 12VSW and single GND):

| Pin | Net | Description |
|---|---|---|
| 1 | ETH_RX+ | Ethernet receive+ (drone perspective) |
| 2 | 12VSW | Switched 12V |
| 3 | ETH_RX- | Ethernet receive− |
| 4 | GND | Ground |
| 5 | ETH_TX+ | Ethernet transmit+ |
| 6 | +12V | Always-on 12V |
| 7 | ETH_TX- | Ethernet transmit− |
| 8 | FMU_CHx | FMU signal (channel varies by port) |
| 9 | CAN1_P | CAN bus positive (CAN_H) |
| 10 | CAN1_N | CAN bus negative (CAN_L) |

> **ETH TX/RX naming** is from the drone's perspective. From the payload's perspective, the directions are reversed (payload transmits on drone's RX pair, receives on drone's TX pair).

---

### Port-specific capabilities

| Port | Main PCB conn. | FMU signal | Switched 12V (12VSW) | Ethernet switch port |
|---|---|---|---|---|
| Bottom | J31 | FMU_CH1 | ✅ (relay 1 on Main PCB) | ETH1_2 (J39) |
| Side 1 | J29 | FMU_CH7 | ❌ (line present, unpowered) | ETH1_3 (J37) |
| Side 2 | J30 | FMU_CH8 | ❌ (line present, unpowered) | ETH2_1 (J38) |

> **12VSW is bottom-only.** The 12VSW net is physically present on all three ports via the attachment PCB, but the Main PCB only activates it for the bottom port (via SSR relay 1). Heavy actuators (pumps, motors, dispensers) must use the bottom port.

> **No analog I/O.** There is no analog signal pin on the Quiver attachment interface. Earlier documentation referencing an "Analog I/O (0–5V)" pin was incorrect.

### Network

All devices share `192.168.144.0/24`. The network is flat — no DHCP runs on the Pi (Siyi's firmware conflicts with DHCP servers). All IPs are static.

**Reserved addresses (do not use):**

| IP | Device |
|---|---|
| 192.168.144.11 | Siyi air unit |
| 192.168.144.12 | Siyi ground unit |
| 192.168.144.20 | Android GCS (Siyi reserved) |
| 192.168.144.25 | Siyi A8 Mini camera |
| 192.168.144.60 | Siyi camera reserved |
| 192.168.144.50 | Raspberry Pi |
| 192.168.144.51 | Flight controller |

**Attachment device range:** `192.168.144.100–.199` (developer-assigned static)

Recommended defaults to avoid collision:

| Port | Recommended IP |
|---|---|
| Bottom (J31) | 192.168.144.100 |
| Side 1 (J29) | 192.168.144.101 |
| Side 2 (J30) | 192.168.144.102 |

> ⚠️ **Correction from earlier docs:** The Integration Guide (v1) referenced `192.168.1.0/24` with DHCP. This is incorrect — Siyi hardware locks the subnet to `192.168.144.0/24` and DHCP conflicts with Siyi's own addressing. Use static IPs in `.100–.199`.

---

## 3. Ethernet Payload Contract

### 3.1 Manifest

Every Ethernet payload **must** serve a manifest at:

```
GET http://<device-ip>/quiver/manifest
Content-Type: application/json
```

The manifest describes everything the SDK needs to interact with the payload without prior knowledge.

#### Manifest schema

```json
{
  "name": "string — human-readable device name",
  "version": "string — semver, device firmware version",
  "quiver_sdk": "string — semver range, minimum compatible SDK version",
  "streams": [
    {
      "id": "string — unique stream identifier",
      "type": "string — quiver/* canonical type or quiver/custom/v1",
      "transport": "websocket | http",
      "port": "integer — TCP port (websocket only)",
      "path": "string — URL path (http only)",
      "rate_hz": "number — nominal publish rate (informational)"
    }
  ],
  "commands": [
    {
      "id": "string — command identifier",
      "method": "POST | GET",
      "path": "string — URL path",
      "body_schema": "object — JSON Schema for request body (optional)"
    }
  ]
}
```

#### Example: RPLidar C1

```json
{
  "name": "RPLidar C1",
  "version": "1.2.0",
  "quiver_sdk": ">=0.1.0",
  "streams": [
    {
      "id": "scan",
      "type": "quiver/pointcloud/v1",
      "transport": "websocket",
      "port": 8765,
      "rate_hz": 10
    },
    {
      "id": "status",
      "type": "quiver/device_status/v1",
      "transport": "http",
      "path": "/quiver/status"
    }
  ],
  "commands": [
    {"id": "start", "method": "POST", "path": "/quiver/cmd/start"},
    {"id": "stop",  "method": "POST", "path": "/quiver/cmd/stop"}
  ]
}
```

#### Example: Agricultural sprayer

```json
{
  "name": "JMRRC 6KG Sprayer",
  "version": "0.9.1",
  "quiver_sdk": ">=0.1.0",
  "streams": [
    {
      "id": "status",
      "type": "quiver/actuator_status/v1",
      "transport": "http",
      "path": "/quiver/status"
    }
  ],
  "commands": [
    {
      "id": "spray",
      "method": "POST",
      "path": "/quiver/cmd/spray",
      "body_schema": {
        "type": "object",
        "properties": {
          "rate_kg_per_min": {"type": "number", "minimum": 0, "maximum": 6}
        },
        "required": ["rate_kg_per_min"]
      }
    },
    {"id": "stop", "method": "POST", "path": "/quiver/cmd/stop"}
  ]
}
```

### 3.2 Transport

#### WebSocket streams

For high-rate data (≥ 5 Hz). Payload opens a WebSocket server on the declared port. SDK connects and receives a stream of newline-delimited JSON messages.

```
ws://192.168.144.100:<port>
```

Each message is one JSON object matching the declared stream type schema (see Section 4).

#### HTTP streams

For low-rate data (< 5 Hz) or status endpoints. Payload serves a JSON endpoint the SDK polls.

```
GET http://192.168.144.100<path>
Content-Type: application/json
```

Response is one JSON object matching the declared stream type schema.

### 3.3 Commands

Commands are HTTP POST (or GET) requests to the declared path. Request and response bodies are JSON.

```
POST http://192.168.144.100/quiver/cmd/spray
Content-Type: application/json

{"rate_kg_per_min": 2.5}
```

Standard response envelope:

```json
{
  "ok": true,
  "message": "spraying at 2.5 kg/min"
}
```

On error:

```json
{
  "ok": false,
  "error": "rate exceeds maximum (6 kg/min)"
}
```

---

## 4. Canonical Stream Types

These are the standard message schemas. Payload developers use these types in their manifest. The SDK provides typed Python objects for each. Field names and units are aligned with DroneCAN DSDL where applicable.

Schema registry lives in this repo at `schemas/`.

---

### `quiver/pointcloud/v1`

**Use case:** LiDAR, depth cameras, 2D/3D scanners  
**DroneCAN alignment:** `uavcan.equipment.range_sensor.Measurement` (single-point equivalent)

```json
{
  "timestamp": 1743440000.123,
  "scan_id": 4821,
  "frame": "sensor",
  "points": [
    {
      "angle_deg": 0.0,
      "distance_m": 2.341,
      "quality": 47,
      "x": 2.341,
      "y": 0.0,
      "z": 0.0
    }
  ]
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `timestamp` | float | ✅ | Unix epoch seconds |
| `scan_id` | integer | ✅ | Monotonically increasing scan counter |
| `frame` | string | ❌ | Reference frame (`"sensor"`, `"body"`, `"world"`) |
| `points[].angle_deg` | float | ✅ | Azimuth angle in degrees |
| `points[].distance_m` | float | ✅ | Distance in metres |
| `points[].quality` | integer 0–255 | ❌ | Signal quality / intensity |
| `points[].x/y/z` | float | ❌ | Cartesian coordinates in metres |

---

### `quiver/range/v1`

**Use case:** Single-beam rangefinders, ultrasonic sensors, time-of-flight  
**DroneCAN alignment:** `uavcan.equipment.range_sensor.Measurement`

```json
{
  "timestamp": 1743440000.456,
  "sensor_id": 1,
  "distance_m": 2.341,
  "min_range_m": 0.1,
  "max_range_m": 12.0,
  "field_of_view_deg": 27.0,
  "sensor_type": "laser"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `timestamp` | float | ✅ | Unix epoch seconds |
| `sensor_id` | integer | ✅ | Sensor instance ID |
| `distance_m` | float | ✅ | Measured distance in metres |
| `min_range_m` | float | ❌ | Sensor minimum range |
| `max_range_m` | float | ❌ | Sensor maximum range |
| `field_of_view_deg` | float | ❌ | Beam angle (half-angle) |
| `sensor_type` | string | ❌ | `"laser"`, `"ultrasonic"`, `"radar"`, `"infrared"` |

---

### `quiver/image/v1`

**Use case:** Cameras (non-Siyi), thermal imagers, multispectral sensors

```json
{
  "timestamp": 1743440000.789,
  "sequence": 1024,
  "width": 1920,
  "height": 1080,
  "encoding": "jpeg",
  "data": "<base64-encoded image bytes>"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `timestamp` | float | ✅ | Unix epoch seconds |
| `sequence` | integer | ✅ | Frame sequence number |
| `width` | integer | ✅ | Image width in pixels |
| `height` | integer | ✅ | Image height in pixels |
| `encoding` | string | ✅ | `"jpeg"`, `"png"`, `"raw_rgb8"`, `"raw_yuv420"` |
| `data` | string | ✅ | Base64-encoded image bytes |
| `bands` | array of string | ❌ | For multispectral: `["red", "green", "nir", "rededge"]` |

> **Note:** For high-framerate video, use RTSP stream URL in the manifest instead of per-frame WebSocket messages. See `quiver/video_stream/v1`.

---

### `quiver/video_stream/v1`

**Use case:** Live video (references an RTSP stream rather than sending frames)

```json
{
  "rtsp_url": "rtsp://192.168.144.102:8554/live",
  "width": 1920,
  "height": 1080,
  "fps": 30,
  "encoding": "h264"
}
```

This type is served as an HTTP stream (static until changed). SDK reads once and exposes `device.stream_url`.

---

### `quiver/environmental/v1`

**Use case:** Temperature, humidity, pressure, gas sensors  
**DroneCAN alignment:** `uavcan.equipment.air_data.StaticPressure`, `uavcan.equipment.air_data.StaticTemperature`

```json
{
  "timestamp": 1743440000.111,
  "temperature_c": 24.3,
  "humidity_pct": 61.2,
  "pressure_pa": 101325.0,
  "co2_ppm": 412.0,
  "pm2_5_ug_m3": 8.1
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `timestamp` | float | ✅ | Unix epoch seconds |
| `temperature_c` | float | ❌ | Temperature in °C |
| `humidity_pct` | float | ❌ | Relative humidity 0–100 |
| `pressure_pa` | float | ❌ | Atmospheric pressure in Pa |
| `co2_ppm` | float | ❌ | CO₂ concentration in ppm |
| `pm2_5_ug_m3` | float | ❌ | PM2.5 in µg/m³ |

At least one measurement field is required.

---

### `quiver/actuator_status/v1`

**Use case:** Sprayers, grippers, dispensers, servos  
**DroneCAN alignment:** `uavcan.equipment.actuator.Status`

```json
{
  "timestamp": 1743440000.222,
  "actuator_id": 0,
  "state": "running",
  "position_pct": 72.0,
  "power_w": 14.4,
  "error": null
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `timestamp` | float | ✅ | Unix epoch seconds |
| `actuator_id` | integer | ✅ | Actuator instance ID |
| `state` | string | ✅ | `"idle"`, `"running"`, `"error"`, `"unknown"` |
| `position_pct` | float | ❌ | Position 0–100% (servos, valves) |
| `power_w` | float | ❌ | Current power draw in W |
| `error` | string \| null | ❌ | Error description if state is `"error"` |

---

### `quiver/gnss/v1`

**Use case:** External GPS / GNSS receivers  
**DroneCAN alignment:** `uavcan.equipment.gnss.Fix`

```json
{
  "timestamp": 1743440000.333,
  "fix_type": "3d",
  "latitude_deg": 30.267153,
  "longitude_deg": -97.743057,
  "altitude_m": 152.4,
  "satellites": 14,
  "hdop": 0.8,
  "vdop": 1.2,
  "velocity_ned_m_s": [0.1, -0.2, 0.0]
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `timestamp` | float | ✅ | Unix epoch seconds |
| `fix_type` | string | ✅ | `"none"`, `"2d"`, `"3d"`, `"dgps"`, `"rtk_float"`, `"rtk_fixed"` |
| `latitude_deg` | float | ✅ | WGS84 latitude |
| `longitude_deg` | float | ✅ | WGS84 longitude |
| `altitude_m` | float | ❌ | Altitude MSL in metres |
| `satellites` | integer | ❌ | Satellites used in fix |
| `hdop` | float | ❌ | Horizontal dilution of precision |
| `vdop` | float | ❌ | Vertical dilution of precision |
| `velocity_ned_m_s` | [float, float, float] | ❌ | NED velocity in m/s |

---

### `quiver/device_status/v1`

**Use case:** Generic device health, used alongside any other stream type

```json
{
  "timestamp": 1743440000.444,
  "health": "ok",
  "uptime_s": 3600,
  "error": null,
  "info": {
    "motor_rpm": 600,
    "scans_sent": 36000
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `timestamp` | float | ✅ | Unix epoch seconds |
| `health` | string | ✅ | `"ok"`, `"warning"`, `"error"`, `"unknown"` |
| `uptime_s` | float | ❌ | Seconds since device boot |
| `error` | string \| null | ❌ | Error description |
| `info` | object | ❌ | Device-specific key/value pairs |

---

### `quiver/custom/v1`

**Use case:** Payloads that don't fit any canonical type

```json
{
  "timestamp": 1743440000.555,
  "schema_url": "https://example.com/schemas/my-payload/v1.json",
  "data": {
    "any": "payload-specific fields here"
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `timestamp` | float | ✅ | Unix epoch seconds |
| `schema_url` | string | ✅ | URL to a JSON Schema describing `data` |
| `data` | object | ✅ | Payload-specific data |

The `schema_url` is used by QuiverHub App Builder to understand the data structure for custom widgets.

---

## 5. CAN Payload Contract

CAN payloads use **DroneCAN v0 (UAVCAN v0)** protocol. Message types are defined by DSDL and are not duplicated here.

**Commonly used types:**

| DSDL type | Use case |
|---|---|
| `uavcan.equipment.range_sensor.Measurement` | Rangefinders, proximity sensors |
| `uavcan.equipment.power.BatteryInfo` | Battery monitors |
| `uavcan.equipment.gnss.Fix` | External GPS |
| `uavcan.equipment.air_data.StaticPressure` | Barometric pressure |
| `uavcan.equipment.air_data.StaticTemperature` | Air temperature |
| `uavcan.equipment.actuator.Command` | Actuator control |
| `uavcan.equipment.actuator.Status` | Actuator feedback |
| `uavcan.protocol.NodeStatus` | Heartbeat (required for all nodes) |

**Setup requirements:**
- CAN bitrate: 1 Mbps
- Each node must have a unique node ID (1–127)
- Termination: 120Ω at each end of the CAN bus
- CAN bus is shared across all three attachment ports + flight controller + companion

```bash
# Bring up CAN interface on companion computer
sudo ip link set can0 up type can bitrate 1000000
```

The SDK wraps DroneCAN messages as typed Python objects:

```python
sensor = Attachment(can_node=42)
sensor.on_can("uavcan.equipment.range_sensor.Measurement", callback)
# callback receives a typed RangeMeasurement, not raw CAN frames
```

---

## 6. mDNS Discovery

Payloads **should** advertise themselves via mDNS for name-based discovery.

**Service type:** `_quiver._tcp`

**TXT record fields:**

| Key | Value |
|---|---|
| `type` | Primary stream type (e.g., `quiver/pointcloud/v1`) |
| `version` | Device firmware version |
| `sdk` | Minimum compatible SDK version |

**Example:**
```
_quiver._tcp.local
  hostname: rplidar-c1.local
  address: 192.168.144.100
  port: 80
  txt: type=quiver/pointcloud/v1 version=1.2.0 sdk=>=0.1.0
```

SDK discovery:
```python
# By name
lidar = Attachment.find("rplidar-c1.local")

# Scan + mDNS combined
devices = Attachment.scan()
# → includes mDNS-advertised devices even if not in .100–.199 range
```

---

## 7. Payload SDK Reference (Python)

The [`quiver-payload-template`](https://github.com/Pan-Robotics/quiver-payload-template/tree/dev) provides a Python reference implementation. Fork it to build a new payload.

### Minimal implementation skeleton

```python
from quiver_payload import QuiverPayload, PointCloudMessage
import asyncio

class MyLidar(QuiverPayload):
    name = "My LiDAR"
    version = "1.0.0"

    streams = [
        {
            "id": "scan",
            "type": "quiver/pointcloud/v1",
            "transport": "websocket",
            "port": 8765,
            "rate_hz": 10,
        }
    ]

    commands = [
        {"id": "start", "method": "POST", "path": "/quiver/cmd/start"},
        {"id": "stop",  "method": "POST", "path": "/quiver/cmd/stop"},
    ]

    async def stream_scan(self):
        """Called by the SDK loop — yield messages to publish."""
        while True:
            points = self.read_sensor()   # your hardware read
            yield PointCloudMessage(points=points)
            await asyncio.sleep(0.1)

    async def cmd_start(self, body):
        self.sensor.start()
        return {"ok": True}

    async def cmd_stop(self, body):
        self.sensor.stop()
        return {"ok": True}

if __name__ == "__main__":
    MyLidar().run()  # starts manifest server, WS server, mDNS
```

### Constants (quiver_payload.py)

```python
# Network
QUIVER_SUBNET        = "192.168.144"
COMPANION_IP         = "192.168.144.50"
PAYLOAD_IP_RANGE     = (100, 199)
DEFAULT_IP_BOTTOM    = "192.168.144.100"
DEFAULT_IP_SIDE1     = "192.168.144.101"
DEFAULT_IP_SIDE2     = "192.168.144.102"

# CAN
CAN_INTERFACE        = "can0"
CAN_BAUDRATE         = 1_000_000

# Power
MAX_CURRENT_DRAW_A   = 2.0
VOLTAGE_NOMINAL_V    = 12.0

# J1 (Molex 2077601281) connector pin numbers — for reference
# Drone-side PCB: J1 → harness → Main PCB
# Payload-side PCB: J1 → payload electronics
J1_ETH_RX_P          = 1
J1_12VSW_A           = 2    # Switched 12V (pin A of doubled pair)
J1_ETH_RX_N          = 3
J1_12VSW_B           = 4    # Switched 12V (pin B of doubled pair)
J1_ETH_TX_P          = 5
J1_GND_A             = 6    # Ground (pin A of doubled pair)
J1_ETH_TX_N          = 7
J1_GND_B             = 8    # Ground (pin B of doubled pair)
J1_CAN_N             = 9    # CAN bus negative (CAN_L)
J1_12V               = 10   # Always-on 12V
J1_CAN_P             = 11   # CAN bus positive (CAN_H)
J1_FMU_CH            = 12   # FMU signal (CH1 bottom / CH7 side1 / CH8 side2)

# J2/J3 (10-pin board-to-board) connector pin numbers
J2_ETH_RX_P          = 1
J2_12VSW             = 2
J2_ETH_RX_N          = 3
J2_GND               = 4
J2_ETH_TX_P          = 5
J2_12V               = 6
J2_ETH_TX_N          = 7
J2_FMU_CH            = 8
J2_CAN_P             = 9
J2_CAN_N             = 10
```

---

## 8. Schema Registry

Canonical stream type schemas live in this repository at:

```
quiver-sdk/
└── schemas/
    ├── pointcloud/v1.json
    ├── range/v1.json
    ├── image/v1.json
    ├── video_stream/v1.json
    ├── environmental/v1.json
    ├── actuator_status/v1.json
    ├── gnss/v1.json
    ├── device_status/v1.json
    └── custom/v1.json
```

Each file is a [JSON Schema](https://json-schema.org/) (draft-07). The SDK uses these for runtime validation when `strict=True` (default: off).

Adding a new canonical type requires:
1. A JSON Schema file at `schemas/<type>/v<n>.json`
2. A typed Python class in `quiver/_types/<type>.py`
3. An entry in this document
4. A PR to this repo

---

## 9. Open Decisions

| # | Question | Status |
|---|---|---|
| 1 | **WebSocket vs HTTP threshold** — currently ≥5 Hz = WebSocket. Is this the right cutoff? | ❓ |
| 2 | **`quiver/custom/v1` schema_url** — required or optional? If no schema URL, Hub can't render a widget | ❓ |
| 3 | **Manifest caching** — SDK caches manifest after first fetch. How long? Forever until reconnect, or TTL? | ❓ |
| 4 | **`quiver/image/v1` for video** — base64 per frame vs separate RTSP URL (video_stream type). Current proposal: both exist. Is that right? | ❓ |
| 5 | **mDNS required or optional** — currently "should". Should it be "must" for certified payloads? | ❓ |
| 6 | **Strict schema validation** — on by default or opt-in? Adds latency, but catches bugs early | ❓ |
| 7 | **CAN payload manifest** — should CAN payloads also serve a manifest (over MAVLink STATUSTEXT or similar)? Or rely purely on DroneCAN node info? | ❓ |

---

## 10. Relationship to Existing Docs

| Document | Where it lives | What it defines |
|---|---|---|
| This document (PAYLOAD_SPEC.md) | `Arrow-air/quiver-sdk` | Data contract for payloads talking to the SDK |
| [SPEC.md](SPEC.md) | `Arrow-air/quiver-sdk` | SDK architecture and API surface |
| [Quiver Payload Integration Guide](https://github.com/Pan-Robotics/quiver-payload-template/tree/dev) | `Pan-Robotics/quiver-payload-template` | Hardware integration guide; should reference this doc for software contract |
| [QuiverHub Architecture](https://github.com/Pan-Robotics/Arrow-Quiver-Hub) | `Pan-Robotics/Arrow-Quiver-Hub` | Hub-side rendering and app model |
