# Clairvoyant: Modular Edge-AI Surveillance System

## 1. Project Architecture
System comprises 4 specialized microservices leveraging Docker for lifecycle management and hardware abstraction.

### 1.1 Core Pipeline: Local Zero-Copy Display
- **Producer (Stream Engine)**: Decodes RTSP to DMA-BUF; pushes frames directly to DRM/KMS Overlay Planes.
- **Consumer (AI Inference)**: Accesses same DMA-BUF handles via Shared Memory for NPU processing.
- **UI Overlay (Chromium)**: Renders transparent UI on the DRM Primary/Top Plane; hardware-composited by Display Controller (VOP/DC).

---

## 2. Service Specifications

### 2.1 Stream Engine (Rust/GStreamer)
**The Data & Display Hub.**
- **Ingestion**: Consumes RTSP streams from Gateway's MediaMTX.
- **Hardware Decoding**: V4L2 Request API (ARM) or VA-API (x86).
- **Local Output**: 
    - Direct DRM/KMS integration via Atomic API.
    - Rule-based pixel mapping to hardware planes (coordinates, Z-order, scaling).
- **Inter-Process Communication**:
    - **DMA-BUF Sharing**: Zero-copy frame delivery to AI container.
    - **ZeroMQ Subscription**: Subscribes to AI inference metadata for SQLite persistence.

### 2.2 AI Inference (ONNX Runtime, C++/Rust)
**The Metadata Generator.**
- **Processing**: Consumes DMA-BUF from Stream Engine.
- **Acceleration**: Platform-specific Execution Providers (RKNPU, ACL, OpenVINO, TensorRT).
- **Output**: Real-time JSON metadata via ZeroMQ PUB.

### 2.3 Display (Chromium Kiosk)
**The Transparent UI Layer.**
- **Runtime**: Chromium with Ozone/GBM backend (No X11/Wayland).
- **Visuals**: Configured with transparent background and hardware-accelerated UI rendering.
- **Integration**: Fetches SPA from Gateway; overlays AI detections/UI elements via WebSocket data.

### 2.4 Network Gateway (Nodejs TypeScript)
**The Remote & API Bridge.**
- **RTSP Proxy**: Maintains persistent connections to registered RTSP sources via MediaMTX; provides proxied RTSP streams for remote browsers.
- **API**: RESTful management of streams, recordings, and system configuration.
- **Hosting**: Serves unified frontend assets for both local Kiosk and remote browsers.

---

## 3. Technical Implementation Standards

### 3.1 Zero-Copy Data Path
| Path | Mechanism | Advantage |
|:---|:---|:---|
| **Decoder → DRM** | DMA-BUF → DRM FB | Zero CPU copy; direct hardware scanout |
| **Decoder → AI** | DMA-BUF → ShM → NPU | Zero memory copy between containers |
| **AI → Display** | ZeroMQ (JSON) | Low-latency metadata synchronization |

### 3.2 Hardware Requirements
- **Kernel**: Mainline Linux with DRM/KMS and DMA-BUF support.
- **Graphics**: Hardware supporting multiple DRM Planes (at least one for video, one for UI).
- **Storage**: Dedicated volume for recording and SQLite WAL logs.

---

## 4. Operational Logic
1. **Startup**: Gateway establishes RTSP connections via MediaMTX; Display container launches Chromium pointing to Gateway.
2. **Streaming**: Stream Engine consumes Gateway-proxied RTSP streams; decodes and occupies specific DRM planes for video.
3. **Inference**: AI analyzes frames; sends bounding boxes to UI via Gateway/WebSocket.
4. **Composition**: SoC Display Controller merges video planes (Stream Engine) and UI plane (Chromium) in real-time.
