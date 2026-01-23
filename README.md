# Clairvoyant

> Modular Edge-AI Surveillance System for SoC Platforms

Clairvoyant is a modular edge-AI surveillance system designed for SoC platforms with hardware acceleration capabilities. It provides real-time video stream processing, AI-powered object detection, and flexible remote management through a microservices architecture.

---

## Features

- **Zero-Copy Architecture**: DMA-BUF, PipeWire, and Wayland linux-dmabuf for efficient data exchange without CPU intervention
- **Hardware Acceleration**: Platform-specific decoders (V4L2 Request API, VA-API) and NPUs (RKNPU, ACL, OpenVINO, TensorRT)
- **Modular Design**: Five independent Docker containers enabling independent deployment, scaling, and maintenance
- **Unified Management**: RESTful API and WebSocket for centralized stream and configuration management
- **Wayland Compositing**: Smithay-based Wayland compositor with linux-dmabuf protocol for seamless video and UI layer composition

---

## Architecture

### Core Pipeline: Wayland Zero-Copy Display

- **Producer (Stream Engine)**: Decodes RTSP to DMA-BUF; exports frames via PipeWire for AI and Wayland for display
- **Consumer (AI Inference)**: Receives zero-copy frames via PipeWire for NPU processing
- **UI Overlay (Chromium)**: Renders transparent UI via Wayland; submits surfaces to the compositor
- **Compositor (Smithay)**: Performs zero-copy composition of video and UI layers via linux-dmabuf; outputs to DRM/KMS

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
│   Remote UI     │   │    Display      │     │     │  Stream Engine  │               
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

For detailed architecture documentation, see [docs/architecture.md](docs/architecture.md).

---

## Services

| Service | Role | Technology | IPC |
|:--------|:-----|:-----------|:----|
| **Stream Engine** | Video Processing Hub | Rust / GStreamer | PipeWire, Wayland, UDS |
| **AI Inference** | Detections Generator | Rust / ONNX Runtime | PipeWire input, UDS output |
| **Display** | Transparent UI Layer | Chromium (Ozone/Wayland) | WebSocket, HTTP, Wayland |
| **Network Gateway** | Remote & API Bridge | Node.js TS / MediaMTX | REST, WebSocket, UDS |
| **Compositor** | Wayland Display Server | Rust / Smithay | Wayland, DRM/KMS |

### Zero-Copy Data Path

| Path | Mechanism | Advantage |
|:-----|:----------|:----------|
| Decoder → Compositor | Wayland linux-dmabuf | Zero-copy via DMA-BUF surfaces |
| Decoder → AI | PipeWire (pipewiresink) | Zero-copy frame sharing via PipeWire |
| UI → Compositor | Wayland linux-dmabuf | Hardware-composited transparent overlay |
| Compositor → Display | DRM/KMS Atomic | Direct hardware scanout |

---

## Quick Start

### Prerequisites

- Linux with DRM/KMS, DMA-BUF, Wayland, and PipeWire support
- Docker and Docker Compose
- Hardware with GPU/NPU acceleration and multiple DRM Planes
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
- `services/stream-engine/configs/stream-engine.yaml` - Decoder and PipeWire settings
- `services/ai-inference/configs/ai-inference.yaml` - Model path and EP configuration
- `services/display/configs/display.yaml` - Gateway URL and display parameters
- `services/compositor/configs/compositor.yaml` - Compositor and display layout settings

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
- IPC Protocols:
  - `docs/pipewire-protocol.md` - Zero-copy frame sharing (PipeWire)
  - `docs/wayland-dmabuf-protocol.md` - Wayland buffer sharing
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
