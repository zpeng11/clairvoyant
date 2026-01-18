# Clairvoyant Architecture

## 1. System Overview

### 1.1 Project Goal
Clairvoyant is a modular edge-AI surveillance system designed for SoC platforms with hardware acceleration capabilities. It provides real-time video stream processing, AI-powered object detection, and flexible remote management through a microservices architecture.

### 1.2 Core Features
- **Zero-Copy Architecture**: Utilizes DMA-BUF and shared memory for efficient data exchange between services without CPU intervention
- **Hardware Acceleration**: Leverages platform-specific decoders (V4L2 Request API, VA-API) and NPUs (RKNPU, ACL, OpenVINO, TensorRT)
- **Modular Design**: Four independent Docker containers enabling independent deployment, scaling, and maintenance
- **Unified Management**: RESTful API and WebSocket for centralized stream and configuration management
- **Hardware Compositing**: DRM/KMS atomic API for seamless video and UI layer composition

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
          │                    │                     │
          ▼                    ▼                     ▼
┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐     
│   Remote UI     │   │    Display      │   │  Stream Engine  │               
│   (Browser)     │   │   (Chromium)    │   │   (GStreamer)   │───────────────┐ 
└─────────────────┘   └─────────────────┘   └─────────────────┘               |
                                │                     │                       |
                                │                     │ DMA-BUF (Zero-Copy)   |
                                │                     ▼                       |
                                │             ┌─────────────────┐             |
                                │             │  AI Inference   │             |
                                │             │   (ONNX Runtime)│             |
                                │             └─────────────────┘             |
                                │                     │                       |
                                │                     │ UDS (JSON)            |
                                │                     ▼                       |
                                │             ┌─────────────────┐             |
                                │             │  Stream Engine  │             |
                                │             │  (Persistence)  │             |  
                                │             └─────────────────┘             |
                                │                  (SQLite)                   |
                                │                                             |
                                └───────UI Plane────────┐                     |
                                                        |                     |
                                                        ┼── Video Planes──────┘ 
                                                        ▼
                                                    ┌─────────────────┐
                                                    │   DRM/KMS       │
                                                    │   Display       │
                                                    │   Controller    │
                                                    └─────────────────┘
```

### 2.2 Data Flow
1. **RTSP Ingestion**: External camera streams → Gateway (MediaMTX) → Stream Engine
2. **Video Decoding**: Stream Engine → Hardware Decoder (V4L2/VA-API) → DRM/KMS Video Planes
3. **AI Processing**: Decoded frames (DMA-BUF) → AI Inference → Detection Metadata (UDS)
4. **Metadata Distribution**: AI Inference(UDS) → Gateway (WebSocket) → Display/Remote UI
5. **Persistence**: AI Inference(UDS) → Stream Engine → SQLite Database
6. **Display Composition**: DRM/KMS Controller merges Video Planes (Stream Engine) + UI Plane (Display)

---

## 3. Service Details

### 3.1 Stream Engine
**Role**: Data & Display Hub

**Data Flow**:
- Input: RTSP streams from Gateway's MediaMTX
- Output: DRM/KMS video planes, DMA-BUF to AI, SQLite persistence

**Technology Stack**:
- Language: Rust
- Framework: GStreamer with V4L2 Request API (ARM) / VA-API (x86)
- Hardware: DRM/KMS Atomic API
- IPC: UDS (Inference results), DMA-BUF (video frames)

**Key Interfaces**:
- `services/stream-engine/src/rtsp/`: RTSP stream consumption logic
- `services/stream-engine/src/decoder/`: Hardware decoder integration
- `services/stream-engine/src/drm/`: DRM/KMS plane management
- `services/stream-engine/src/ipc/dma_buf.rs`: DMA-BUF sharing with AI Inference
- `services/stream-engine/src/ipc/uds_server.rs`: Subscribe AI metadata for persistence
- `services/stream-engine/src/storage/sqlite.rs`: SQLite database operations

### 3.2 AI Inference
**Role**: Metadata Generator

**Data Flow**:
- Input: DMA-BUF video frames from Stream Engine
- Output: Detection metadata via UDS

**Technology Stack**:
- Language: Rust
- Framework: ONNX Runtime
- Hardware Acceleration: Platform-specific Execution Providers
  - RKNPU (Rockchip NPU)
  - ACL (Huawei Ascend)
  - OpenVINO (Intel)
  - TensorRT (NVIDIA)

**Key Interfaces**:
- `services/ai-inference/src/ipc/dma_buf.rs`: Receive DMA-BUF frames
- `services/ai-inference/src/inference/onnx_runtime.rs`: ONNX Runtime wrapper
- `services/ai-inference/src/inference/ep/`: Execution Provider adapters
- `services/ai-inference/src/ipc/uds_publisher.rs`: Publish JSON metadata
- `services/ai-inference/configs/ai-inference.yaml`: Model path and EP configuration

### 3.3 Display
**Role**: Transparent UI Layer

**Data Flow**:
- Input: UI assets from Gateway, AI metadata via WebSocket (optional)
- Output: Chromium rendering on DRM/KMS UI Plane

**Technology Stack**:
- Runtime: Chromium Kiosk Mode
- Backend: Ozone/GBM (No X11/Wayland)
- Rendering: Hardware-accelerated GPU compositing
- Communication: WebSocket (from Gateway), HTTP/HTTPS (UI assets)

**Key Interfaces**:
- `services/display/src/main.sh`: Chromium startup script
- `services/display/src/browser/chromium.conf`: Chromium parameters (Ozone/GBM, transparent background)
- `services/display/static/`: Local fallback assets when Gateway is unavailable
- `services/display/configs/display.yaml`: Gateway URL, display parameters

**Fallback Mechanism**:
- If Gateway is unreachable, Display serves local static HTML from `services/display/static/`

### 3.4 Network Gateway
**Role**: Remote & API Bridge

**Data Flow**:
- Input: External RTSP sources, REST API requests, WebSocket connections, UDS publish.
- Output: Proxied RTSP streams, REST API responses, WebSocket metadata, UI assets

**Technology Stack**:
- Language: Node.js TypeScript
- RTSP Proxy: MediaMTX (bluenviron/mediamtx)
- API Framework: Express or Fastify
- WebSocket: ws or socket.io
- Frontend: SPA (static assets served via HTTP)

**Key Interfaces**:
- `services/network-gateway/src/api/`: REST API endpoints (streams, recordings, config)
- `services/network-gateway/src/mediamtx/client.ts`: MediaMTX API client
- `services/network-gateway/src/websocket/server.ts`: WebSocket server for AI metadata
- `services/network-gateway/src/static/`: Unified frontend assets (SPA)
- `services/network-gateway/configs/gateway.yaml`: API configuration, MediaMTX settings

---

## 4. IPC Communication Mechanisms

### 4.1 DMA-BUF Zero-Copy Sharing
**Purpose**: Share decoded video frames between Stream Engine and AI Inference without CPU memory copying

**Mechanism**:
- GStreamer decoder produces DMA-BUF file descriptors via V4L2/VA-API plugins.
- DMA-BUF handles are shared through Unix Domain Socket SCM_RIGHTS and SOCK_SEQPACKET
- AI Inference imports DMA-BUF from UDS and use CPU/GPU/NPU for inference processing
- UDS message back to notify GStreamer to reuse DMA-BUF

**Reference Files**:
- Protocol: `dma-buf-protocol.md`
- Data Structure: `shared/schema/detection-schema.json`

### 4.2 UDS Detection Publisher
**Purpose**: Publish AI inference metadata to Gateway and Stream Engine

**Transport**: Unix Domain Socket (SOCK_SEQPACKET), Inference results JSON.

**Connections**:
1. **AI Inference → Gateway**: Real-time detection results for UI overlay (WebSocket relay)
2. **AI Inference → Stream Engine**: Detection results for SQLite persistence and notify GStreamer memory reuse.

**Message Schema**: Return schema (defined in `shared/schema/detection-schema.json`)

**Reference**: `UDS-protocol.md`

### 4.3 WebSocket Real-Time Push
**Purpose**: Deliver AI metadata from Gateway to Display and remote browsers

**Connection Flow**:
- AI Inference publishes to Gateway via UDS
- Gateway forwards metadata via WebSocket to connected clients
- Display establishes WebSocket connection to Gateway on startup

**Message Format**: Same as DMA-BUF return message schema (JSON)

---

## 5. Hardware Abstraction Layer

### 5.1 Decoder Adapters
**V4L2 Request API** (ARM platforms - Rockchip, Amlogic, NXP)
- Used by Stream Engine for hardware-accelerated H.264/H.265 decoding
- Outputs DMA-BUF frames directly to DRM/KMS
- Supported platforms: RK3588, RK3568, etc.

**VA-API** (x86 platforms - Intel, AMD)
- Used by Stream Engine for hardware-accelerated decoding
- Outputs DMA-BUF or VA surface to DRM/KMS
- Supported platforms: Intel NUC, embedded x86 SoCs

### 5.2 Inference Accelerator Adapters
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

### 5.3 Display Controller Integration
**DRM/KMS Atomic API**
- Used by Stream Engine for direct video plane output
- Hardware compositing by SoC Display Controller (VOP/DC)
- Multi-plane support:
  - Overlay Planes: Video frames from Stream Engine
  - Primary/Top Plane: UI from Chromium (Display service)

**Plane Configuration**:
- Defined in `services/stream-engine/configs/stream-engine.yaml`
- Includes coordinates, Z-order, scaling factors

---

## 6. Directory Structure

### 6.1 Project Root
```
clairvoyant/
├── services/              # Microservices (4 containers)
├── shared/                # Cross-service resources (IPC protocols, configs)
├── data/                  # Data volumes (Docker mounts)
├── docs/                  # Documentation
├── scripts/               # Build and deployment scripts
└── README.md              # Project overview
```

### 6.2 Service Responsibilities
| Service | Primary Responsibility | Key Directories |
|:---|:---|:---|
| **Stream Engine** | Video decoding, DRM/KMS output, SQLite persistence | `services/stream-engine/src/rtsp/`, `decoder/`, `drm/`, `storage/` |
| **AI Inference** | Object detection, metadata publishing | `services/ai-inference/src/inference/`, `models/` |
| **Display** | UI rendering on Chromium | `services/display/src/`, `static/` |
| **Network Gateway** | RTSP proxy, REST API, WebSocket, frontend hosting | `services/network-gateway/src/api/`, `mediamtx/`, `websocket/`, `static/` |

### 6.3 Shared Resources
- `docs/`: IPC mechanism definitions and protocols (DMA-BUF, UDS, REST API)
- `shared/schema/`: JSON schemas for IPC messages
- `shared/configs/`: Common configuration templates (logging, environment)

### 6.4 Data Volumes
- `data/recordings/`: Video recording files
- `data/database/`: SQLite database (WAL logs)

### 6.5 Documentation & Scripts
- `docs/`: Architecture, deployment, API documentation
- `scripts/`: Build, deployment, Docker Compose orchestration

---

## 7. Runtime Flow

### 7.1 Startup Sequence
1. **Network Gateway** starts first:
    - Initializes MediaMTX service
    - Establishes connections to configured RTSP sources
    - Starts REST API and WebSocket server
    - Starts UDS server for AI metadata
2. **Stream Engine** starts:
    - Connects to Gateway's MediaMTX for RTSP streams
    - Initializes DRM/KMS display planes
    - Starts UDS server for AI metadata persistence
3. **AI Inference** starts:
    - Connects to Stream Engine's DMA-BUF sharing
    - Loads ONNX model and initializes Execution Provider
    - Starts UDS publishers to Gateway and Stream Engine
4. **Display** starts:
    - Attempts to connect to Gateway for UI assets
    - Falls back to local static assets if Gateway unreachable
    - Launches Chromium in Kiosk mode (Ozone/GBM)

### 7.2 Normal Operation Flow
1. **Video Streaming**: RTSP sources → MediaMTX → Stream Engine → Hardware Decoder → DRM/KMS Video Planes
2. **AI Inference**: DMA-BUF frames → AI Inference → ONNX Runtime → Detection Results → UDS Publisher
3. **Metadata Distribution**: UDS → Gateway → WebSocket → Display/Remote UI
4. **Persistence**: UDS → Stream Engine → SQLite Database
5. **Display Composition**: SoC Display Controller composites Video Planes + UI Plane

### 7.3 Exception Handling
- **Gateway Unavailable**: Display serves local fallback UI from `services/display/static/`
- **RTSP Source Lost**: Stream Engine reports to Gateway via REST API; Gateway attempts reconnection
- **AI Inference Failure**: Stream Engine continues video display; Gateway notifies UI of detection unavailability
- **DRM/KMS Failure**: System logs error; Display service continues with UI overlay only
