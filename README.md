# AR Cooperative Map Creation

A multiplayer Augmented Reality application where users collaboratively build a shared spatial map of their physical environment in real time. Each device detects surfaces independently, shares that data over a network, and all clients render a unified, synchronized play space.

Built as my BSc thesis at BME (Budapest University of Technology and Economics), Faculty of Electrical Engineering and Informatics, December 2021.

## The Problem

In AR, every device lives in its own coordinate system. There is no shared ground truth. This makes multiplayer AR fundamentally harder than multiplayer VR — you can't just sync object positions, because position (0,0,0) means something different on every device.

On top of that, each user sees the environment from a different angle, so their plane detections differ: one phone might detect a table surface as two overlapping polygons, another as one large plane, and a third might miss it entirely.

This project tackles both problems: **coordinate synchronization** and **surface deduplication**.
 ---

## How It Works
### Architecture

```
[Phone A]          [Phone B]
   |                   |
   └──── Mirror (TCP) ──┘
              |
           [Server/Host]
         (plane filtering +
          world map storage)
```

One device acts as host+client (no dedicated server required). All plane data flows through it.
### Surface Detection & Sharing

- **ARFoundation + ARCore** handle plane detection on each device
- The `ARPlaneManager` fires events (`planesChanged`, `boundaryChanged`) when surfaces are added, updated, or removed
- On each event, the device extracts the plane's boundary points, position, rotation, and ID, serializes them to JSON, and sends a `Command` to the server via Mirror

### Surface Deduplication (Server-side)

The server maintains a `Dictionary` of all known planes. When a new plane arrives, it runs a two-stage filter:

1. **Winding Number test** — checks whether the new plane's centroid falls inside any existing plane polygon (fast implementation by W. Randolph Franklin)
2. **Bounding box test** — checks whether the new plane's boundary points fall within the min/max extents of the existing plane

If the new plane is fully contained → discard it (it's a duplicate).  
If centroids are close but shapes don't overlap → merge, averaging positions (useful when the player who originally detected the surface has left the session).  
If neither → add as a new plane.

This avoids re-rendering the same surface multiple times and keeps the shared map clean.

### Coordinate Synchronization

Each player scans a **QR code** placed at a known physical location. This sets the shared world origin `(0,0,0)`.

All planes are stored as children of a `WorldMap` object. A `LocalPositionCalculator` helper object converts each plane's position from the device's local coordinate space into a position relative to the shared origin before it's sent to the server. This means all clients place planes in the same world — regardless of where they were standing when the app started.

If a device loses tracking and re-scans the QR code, it re-syncs automatically and the plane positions update.

### Joining Mid-Session

When a new player connects, they request the current world map from the server via a `TargetRpc` (sent only to them). The server iterates its plane dictionary and sends each entry directly to the new client, which reconstructs the full map locally.

---

## Features

- Real-time plane sharing between multiple AR devices
- Winding Number algorithm for surface overlap detection
- QR-based coordinate system synchronization
- Mid-session join with full world map handoff
- Plane mesh reconstruction from boundary point arrays
- Occlusion-aware rendering per platform

---

## Tech Stack

| Layer | Technology |
|---|---|
| Game engine | Unity |
| AR framework | ARFoundation + ARCore |
| Networking | Mirror (evolved from Unity UNet) |
| Platform | Android (primary), iOS/iPhone 12 Pro (tested) |
| Language | C# |
| Algorithms | Winding Number (W.R. Franklin), Raycast PIP, Barycentric |

##Showcase
<img width="321" height="551" alt="image" src="https://github.com/user-attachments/assets/8a850057-7884-475c-96f8-4026e4cd7142" />
Environment detection by an iPhone
<img width="562" height="394" alt="image" src="https://github.com/user-attachments/assets/1bf2bb4d-af15-484c-8f07-2af8a2087667" />
The detected planes on the server side



 
