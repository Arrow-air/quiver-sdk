# quiver-sdk

Python SDK for the [Quiver](https://github.com/Arrow-air/project-quiver) UAV platform.

> **Status: Pre-alpha — specification and design phase.**
> See [`SPEC.md`](SPEC.md) for the architecture and API design. Discussion and feedback welcome.

---

## What is this?

`quiver-sdk` is a Python library that lets developers build applications on the Quiver drone platform — onboard autonomy, payload control, ground station tools, and more.

```python
from quiver import Vehicle, Camera, Attachment

drone = Vehicle()
cam = Camera()

drone.arm()
drone.takeoff(10)
cam.set_gimbal(pitch=-45, yaw=0)
photo_path = cam.take_photo()
drone.land()
```

**Two packages:**

- **`quiver-sdk`** — Core hardware abstraction. Protocol-direct Python library. Offline-capable. Runs on the onboard Raspberry Pi or any machine on the Quiver network.
- **`quiver-hub`** — Platform integration layer. Connects `quiver-sdk` to [QuiverHub](https://github.com/Pan-Robotics/Quiver-Hub) for telemetry forwarding, remote job execution, and payload streaming.

---

## Architecture

```
Applications (any language)
        ↕ WebSocket / REST
  QuiverHub Server (Node.js)
        ↕ REST / WebSocket
  Companion Computer (Pi)
    quiver-hub  ←── TelemetryForwarder, JobRunner, PayloadStreamer
    quiver-sdk  ←── Vehicle, Camera, Attachment
        ↕ pymavlink / Siyi binary / CAN
  Hardware (ArduCopter FC / Siyi A8 / Attachment devices)
```

---

## Design Principles

- **Offline-first.** Never requires internet. Drones fly in fields.
- **ArduCopter-native.** pymavlink, not MAVSDK. Quiver runs ArduCopter.
- **One pip install.** No daemon, no subprocess, no arch-specific binary.
- **Simple by default, powerful when needed.** `Vehicle()` just works. `drone.mav` exposes raw pymavlink when you need it.
- **Hub integration is optional.** `quiver-hub` adds cloud connectivity. The core has no awareness of QuiverHub.

---

## Status

| Module | Status |
|---|---|
| `quiver.vehicle` | 📐 Spec |
| `quiver.camera` | 📐 Spec |
| `quiver.attachment` | 📐 Spec |
| `quiver-hub` telemetry | 📐 Spec |
| `quiver-hub` job runner | 📐 Spec |
| `quiver-hub` streaming | 📐 Spec |

---

## Contributing

Read [`SPEC.md`](SPEC.md) first. Then join the discussion in [GitHub Discussions](https://github.com/Arrow-air/quiver-sdk/discussions).

This project is part of [Arrow DAO](https://arrowair.com). Contributions are welcome — hardware access for testing helps, reach out on [Discord](https://discord.gg/arrowair).

---

## License

[Apache 2.0](LICENSE)
