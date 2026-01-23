# Clairvoyant Architecture

## 1. System Overview

### 1.1 Project Goal
Clairvoyant is a modular edge-AI surveillance system designed for SoC platforms with hardware acceleration capabilities. It provides real-time video stream processing, AI-powered object detection, and flexible remote management through a microservices architecture.

### 1.2 Core Features
- **Zero-Copy Architecture**: Utilizes DMA-BUF, PipeWire, and Wayland linux-dmabuf for efficient data exchange between services without CPU intervention
- **Hardware Acceleration**: Leverages platform-specific decoders (V4L2 Request API, VA-API) and NPUs (RKNPU, ACL, OpenVINO, TensorRT)
- **Modular Design**: Five independent Docker containers enabling independent deployment, scaling, and maintenance
- **Unified Management**: RESTful API and WebSocket for centralized stream and configuration management
- **Wayland Compositing**: Smithay-based Wayland compositor with linux-dmabuf protocol for seamless video and UI layer composition

---

## 2. Overall Architecture

### 2.1 Service Relationship
```
┌─────────────────────────────────────────────────────────────────┐
│                          External RTSP Sources                  │
└─────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Network Gateway (MediaMTX)                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │  REST API    │  │  WebSocket   │  │  RTSP Proxy  │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
          │                    │              ▴              │
          │                    │              │              │
          ▼                    ▼              │              ▼
┌─────────────────┐   ┌─────────────────┐     │     ┌─────────────────┐     
│   Remote UI     │   │       UI        │     │     │  Stream Engine  │     
│   (Browser)     │   │   (Chromium)    │     │     │   (GStreamer)   │
└─────────────────┘   └─────────────────┘     │     └─────────────────┘
                              │               │          │         │
                              │ Wayland       │          │         │ PipeWire
                              │ linux-dmabuf  │          │         │ (frames)
                              │               │          │         ▼
                              │               │          │  ┌─────────────────┐
                              │               └──────────│──│  AI Inference   │
                              │                          │  │ (ONNX Runtime)  │
                              │                          │  └─────────────────┘
                              │                          │  Wayland
                              │                          │  linux-dmabuf
                              ▼                          ▼
                         ┌─────────────────────────────────────┐
                         │            Compositor               │
                         │            (Smithay)                │
                         └─────────────────────────────────────┘
                                          │
                                          ▼
                                   ┌─────────────────┐
                                   │   DRM/KMS       │
                                   │   Display       │
                                   └─────────────────┘
```

### 2.2 Data Flow
1. **RTSP Ingestion**: External camera streams → Gateway (MediaMTX) → Stream Engine/Remote UI
2. **Video Decoding**: Stream Engine → Hardware Decoder (V4L2/VA-API) → Wayland surface (linux-dmabuf) → Compositor
3. **AI Processing**: Decoded frames → PipeWire (pipewiresink) → AI Inference (pipewire-rs) → Gateway
4. **Detections Distribution**: Gateway (WebSocket) → UI/Remote UI
5. **UI Rendering**: UI (Chromium Ozone/Wayland) → Wayland surface (linux-dmabuf) → Compositor
6. **Display Composition**: Compositor (Smithay) merges video surfaces + UI surfaces → DRM/KMS output

---

## 3. Service Details

### 3.1 Stream Engine
**Role**: Video Processing & Distribution Hub

**Data Flow**:
- Input: RTSP streams from Gateway's MediaMTX, UDS message of video layout and config
- Output: Wayland surfaces (linux-dmabuf) to Compositor, PipeWire frames to AI Inference, Recording persistence

**Technology Stack**:
- Language: Rust
- Framework: GStreamer with V4L2 Request API (ARM) / VA-API (x86)
- Display: Wayland client (wayland-client, linux-dmabuf-v1 protocol)
- AI IPC: PipeWire daemon with GStreamer pipewiresink
- Config IPC: UDS (video layouts, configs from Gateway)

**Key Interfaces**:
- `services/stream-engine/src/rtsp/`: RTSP stream consumption logic
- `services/stream-engine/src/decoder/`: Hardware decoder integration
- `services/stream-engine/src/wayland/client.rs`: Wayland client for Compositor connection
- `services/stream-engine/src/wayland/dmabuf.rs`: linux-dmabuf surface submission
- `services/stream-engine/src/ipc/pipewire_sink.rs`: PipeWire daemon and GStreamer pipewiresink for AI frames
- `services/stream-engine/src/ipc/uds_client.rs`: Receiving video layout from Gateway
- `services/stream-engine/src/storage/sqlite.rs`: SQLite database operations

### 3.2 AI Inference
**Role**: Detections Generator

**Data Flow**:
- Input: Video frames from Stream Engine via PipeWire, config from Gateway
- Output: Detections distributed to Gateway 

**Technology Stack**:
- Language: Rust
- Framework: ONNX Runtime
- Frame Input: Native PipeWire API (pipewire-rs)
- Hardware Acceleration: Platform-specific Execution Providers
  - RKNPU (Rockchip NPU)
  - ACL (Huawei Ascend)
  - OpenVINO (Intel)
  - TensorRT (NVIDIA)

**Key Interfaces**:
- `services/ai-inference/src/ipc/pipewire_source.rs`: Native PipeWire client for receiving frames
- `services/ai-inference/src/inference/onnx_runtime.rs`: ONNX Runtime wrapper
- `services/ai-inference/src/inference/ep/`: Execution Provider adapters
- `services/ai-inference/src/ipc/uds_publisher.rs`: Publish JSON detections
- `services/ai-inference/configs/ai-inference.yaml`: Model path and EP configuration

### 3.3 UI
**Role**: Transparent UI Layer

**Data Flow**:
- Input: UI assets and backend from Gateway
- Output: Wayland surface (linux-dmabuf) to Compositor

**Technology Stack**:
- Runtime: Chromium Kiosk Mode
- Backend: Ozone/Wayland (connects to Compositor)
- Rendering: Hardware-accelerated GPU compositing via linux-dmabuf
- Communication: WebSocket (from Gateway), HTTP/HTTPS (UI assets)

**Key Interfaces**:
- `services/ui/src/main.sh`: Chromium startup script
- `services/ui/src/browser/chromium.conf`: Chromium parameters (Ozone/Wayland, transparent background)
- `services/ui/static/`: Local fallback assets when Gateway is unavailable
- `services/ui/configs/ui.yaml`: Gateway URL, Compositor socket, display parameters

**Fallback Mechanism**:
- If Gateway is unreachable, UI serves local static HTML from `services/ui/static/`

### 3.4 Network Gateway
**Role**: Remote & API Bridge

**Data Flow**:
- Input: External RTSP sources, REST API requests, WebSocket connections, UDS connections.
- Output: Proxied RTSP streams, REST API responses, WebSocket detections, UI assets, UDS video layout and configs.

**Technology Stack**:
- Language: Node.js TypeScript
- RTSP Proxy: MediaMTX (bluenviron/mediamtx)
- API Framework: Express or Fastify
- WebSocket: ws or socket.io
- Frontend: SPA (static assets served via HTTP)

**Key Interfaces**:
- `services/network-gateway/src/api/`: REST API endpoints (streams, recordings, config)
- `services/network-gateway/src/uds/`: UDS communications
- `services/network-gateway/src/mediamtx/client.ts`: MediaMTX API client
- `services/network-gateway/src/websocket/server.ts`: WebSocket server for AI detections
- `services/network-gateway/src/static/`: Unified frontend assets (SPA)
- `services/network-gateway/configs/gateway.yaml`: API configuration, MediaMTX settings

### 3.5 Compositor
**Role**: Wayland Display Compositor

**Data Flow**:
- Input: Wayland surfaces with linux-dmabuf from Stream Engine (video) and UI
- Output: Composed frames to DRM/KMS display

**Technology Stack**:
- Language: Rust
- Framework: Smithay (Wayland compositor library)
- Protocols: wl_compositor, linux-dmabuf-v1, xdg-shell
- Output: DRM/KMS Atomic API

**Key Interfaces**:
- `services/compositor/src/compositor/mod.rs`: Smithay compositor state and event loop
- `services/compositor/src/dmabuf/handler.rs`: linux-dmabuf-v1 protocol handler
- `services/compositor/src/drm/backend.rs`: DRM/KMS backend for display output
- `services/compositor/src/shell/`: Surface management and layer ordering
- `services/compositor/configs/compositor.yaml`: Display configuration, layer priorities

**Layer Management**:
- Video surfaces from Stream Engine are placed on lower z-order (background)
- UI surfaces from UI service are placed on higher z-order (foreground/overlay)

---

## 4. Hardware Abstraction Layer

### 4.1 Decoder Adapters
**V4L2 Request API** (ARM platforms - Rockchip, Amlogic, NXP)
- Used by Stream Engine for hardware-accelerated H.264/H.265 decoding
- Outputs DMA-BUF frames directly to DRM/KMS
- Supported platforms: RK3588, RK3568, etc.

**VA-API** (x86 platforms - Intel, AMD)
- Used by Stream Engine for hardware-accelerated decoding
- Outputs DMA-BUF or VA surface to DRM/KMS
- Supported platforms: Intel NUC, embedded x86 SoCs

### 4.2 Inference Accelerator Adapters
**RKNPU** (Rockchip NPU)
- Execution Provider: `rknn-rt` or RKNPU ONNX EP
- Target platforms: RK3588, RK3568, RK3399 Pro
- Configuration: `services/ai-inference/src/inference/ep/rknpu.rs`

**ACL** (Huawei Ascend NPU)
- Execution Provider: Ascend ONNX EP
- Target platforms: HiSilicon Ascend SoCs
- Configuration: `services/ai-inference/src/inference/ep/acl.rs`

**OpenVINO** (Intel CPUs/iGPUs)
- Execution Provider: OpenVINO Runtime
- Target platforms: Intel x86 platforms with integrated graphics
- Configuration: `services/ai-inference/src/inference/ep/openvino.rs`

**TensorRT** (NVIDIA GPUs)
- Execution Provider: TensorRT ONNX EP
- Target platforms: NVIDIA Jetson, discrete GPUs
- Configuration: `services/ai-inference/src/inference/ep/tensorrt.rs`

---

## 5. Directory Structure

### 5.1 Project Root
```
clairvoyant/
├── services/              # Microservices (5 containers)
├── data/                  # Data volumes (Docker mounts)
├── docs/                  # Documentation
├── scripts/               # Build and deployment scripts
└── README.md              # Project overview
```

### 5.2 Service Responsibilities
| Service | Primary Responsibility | Key Directories |
|:---|:---|:---|
| **Stream Engine** | Video decoding, Wayland client, PipeWire producer | `services/stream-engine/src/rtsp/`, `decoder/`, `wayland/`, `storage/` |
| **AI Inference** | Object detection via PipeWire frames | `services/ai-inference/src/inference/`, `ipc/`, `models/` |
| **UI** | UI rendering via Wayland | `services/ui/src/`, `static/` |
| **Network Gateway** | RTSP proxy, REST API, WebSocket, full stack hosting | `services/network-gateway/src/api/`, `mediamtx/`, `websocket/`, `static/` |
| **Compositor** | Wayland compositor, DRM/KMS output | `services/compositor/src/compositor/`, `dmabuf/`, `drm/` |

### 5.3 Shared Resources
- `docs/`: IPC mechanism definitions and protocols (PipeWire, Wayland, UDS, REST API)
- `services/common/schema/`: JSON schemas for IPC messages
- `services/common/configs/`: Common configuration templates (logging, environment)

### 5.4 Data Volumes
- `data/recordings/`: Video recording files
- `data/database/`: SQLite database (WAL logs)

### 5.5 Documentation & Scripts
- `docs/`: Architecture, deployment, API documentation
- `scripts/`: Build, deployment, Docker Compose orchestration

---

## 6. Runtime Flow

### 6.1 Startup Sequence
1. **Network Gateway** starts first:
    - Initializes MediaMTX service
    - Establishes connections to configured RTSP sources
    - Starts REST API and WebSocket server
    - Starts UDS server for connecting Stream Engine
    - Starts UDS server for connecting AI Inference
2. **Compositor** starts second:
    - Initializes DRM/KMS backend
- Creates Wayland socket for clients
- Waits for client connections (Stream Engine, UI)
3. **Stream Engine** starts:
    - Connects to Gateway's UDS to receive video layout and configs
    - Connects to Gateway's MediaMTX to consume RTSP streams
    - Connects to Compositor as Wayland client
    - Starts PipeWire daemon for AI frame sharing
4. **AI Inference** starts:
    - Connects to Stream Engine's PipeWire to receive video frames
    - Connects to Gateway's UDS to receive configs and send AI detections
    - Loads ONNX model and initializes Execution Provider
5. **UI** starts:
    - Attempts to connect to Gateway for UI assets
    - Connects to Compositor as Wayland client
    - Launches Chromium in Kiosk mode (Ozone/Wayland)

### 6.2 Exception Handling
- **Gateway Unavailable**: UI serves local fallback UI from `services/ui/static/`, remote unreachable 
- **Compositor Unavailable**: Stream Engine and UI cannot render locally, fall back to Remote UI only
- **Stream Engine Unavailable/PipeWire Failure**: AI Inference cannot receive frames, Gateway notifies UI of detection unavailability
- **AI Inference Unavailable/Failure**: Gateway notifies UI of detection unavailability
- **UI Chromium Unavailable**: Gateway remote UI keep working
- **RTSP Source Lost**: Gateway attempts reconnection and keep proxying a signal lost word display 
