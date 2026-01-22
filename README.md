# Clairvoyant

> Modular Edge-AI Surveillance System for SoC Platforms

Clairvoyant is a modular edge-AI surveillance system designed for SoC platforms with hardware acceleration capabilities. It provides real-time video stream processing, AI-powered object detection, and flexible remote management through a microservices architecture.

---

## Features

- **Zero-Copy Architecture**: DMA-BUF and shared memory for efficient data exchange without CPU intervention
- **Hardware Acceleration**: Platform-specific decoders (V4L2 Request API, VA-API) and NPUs (RKNPU, ACL, OpenVINO, TensorRT)
- **Modular Design**: Four independent Docker containers enabling independent deployment, scaling, and maintenance
- **Unified Management**: RESTful API and WebSocket for centralized stream and configuration management
- **Hardware Compositing**: DRM/KMS atomic API for seamless video and UI layer composition

---

## Architecture

### Core Pipeline: Local Zero-Copy Display

- **Producer (Stream Engine)**: Decodes RTSP to DMA-BUF; pushes frames directly to DRM/KMS Overlay Planes
- **Consumer (AI Inference)**: Accesses same DMA-BUF handles via Shared Memory for NPU processing
- **UI Overlay (Chromium)**: Renders transparent UI on the DRM Primary/Top Plane; hardware-composited by Display Controller

```
┌─────────────────────────────────────────────────────────────────┐
│                      External RTSP Sources                      │
└─────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Network Gateway (MediaMTX)                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │  REST API    │  │  WebSocket   │  │  RTSP Proxy  │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
          │                    │                             │
          ▼                    ▼                             ▼
┌─────────────────┐   ┌─────────────────┐           ┌─────────────────┐
│   Remote UI     │   │    Display      │           │  Stream Engine  │
│   (Browser)     │   │ (Local Chromium)│           │   (GStreamer)   │
└─────────────────┘   └─────────────────┘           └─────────────────┘
                                │                           │
                                │                           │ DMA-BUF
                                │                           ▼
                                │                   ┌─────────────────┐
                                │                   │  AI Inference   │
                                │                   │ (ONNX Runtime)  │
                                │                   └─────────────────┘
                                │
                                └───── UI Plane ────┐
                                                    ▼
                                            ┌─────────────────┐
                                            │   DRM/KMS       │
                                            │   Display       │
                                            └─────────────────┘
```

For detailed architecture documentation, see [docs/architecture.md](docs/architecture.md).

---

## Services

| Service | Role | Technology | IPC |
|:--------|:-----|:-----------|:----|
| **Stream Engine** | Data & Display Hub | Rust / GStreamer | DMA-BUF → DRM, UDS |
| **AI Inference** | Detections Generator | Rust / ONNX Runtime | DMA-BUF input, UDS output |
| **Display** | Transparent UI Layer | Chromium Kiosk (Ozone/GBM) | WebSocket, HTTP |
| **Network Gateway** | Remote & API Bridge | Node.js TypeScript / MediaMTX | REST, WebSocket, UDS |

### Zero-Copy Data Path

| Path | Mechanism | Advantage |
|:-----|:----------|:----------|
| Decoder → DRM | DMA-BUF → DRM FB | Zero CPU copy; direct hardware scanout |
| Decoder → AI | DMA-BUF → ShM → NPU | Zero memory copy between containers |
| AI → Gateway | UDS (JSON) | Low-latency metadata delivery |
| Gateway → Display | WebSocket (JSON) | Real-time UI synchronization |

---

## Quick Start

### Prerequisites

- Linux with DRM/KMS and DMA-BUF support
- Docker and Docker Compose
- Hardware with multiple DRM Planes (video + UI)
- Supported platform (see [Hardware Support](#hardware-support))

### Installation

```bash
# Clone repository
git clone https://github.com/your-org/clairvoyant.git
cd clairvoyant

# Start services
docker compose up -d
```

### Configuration

Configuration files are located in each service's `configs/` directory:

- `services/network-gateway/configs/gateway.yaml` - API and MediaMTX settings
- `services/stream-engine/configs/stream-engine.yaml` - Decoder and DRM settings (TBD)
- `services/ai-inference/configs/ai-inference.yaml` - Model path and EP configuration
- `services/display/configs/display.yaml` - Gateway URL and display parameters

---

## Hardware Support

### Decoder Platforms

| Platform | API | SoCs |
|:---------|:----|:-----|
| ARM | V4L2 Request API | RK3588, RK3568, Amlogic, NXP |
| x86 | VA-API | Intel NUC, embedded x86 SoCs |

### Inference Accelerators

| Execution Provider | Target Platform |
|:-------------------|:----------------|
| RKNPU | Rockchip RK3588, RK3568, RK3399 Pro |
| ACL | Huawei Ascend / HiSilicon |
| OpenVINO | Intel CPUs / iGPUs |
| TensorRT | NVIDIA Jetson, discrete GPUs |

---

## Documentation

- [Architecture](docs/architecture.md) - System design and service details
- [API Reference](docs/api.md) - REST API and WebSocket specifications
- IPC Protocols (TBD):
  - `docs/dma-buf-protocol.md` - Zero-copy frame sharing
  - `docs/UDS-protocol.md` - Unix Domain Socket messaging

---

## Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/your-feature`)
3. Commit your changes (`git commit -m 'Add your feature'`)
4. Push to the branch (`git push origin feature/your-feature`)
5. Open a Pull Request

For major changes, please open an issue first to discuss the proposed changes.

---

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
