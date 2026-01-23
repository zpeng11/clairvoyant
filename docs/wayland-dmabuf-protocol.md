# Wayland linux-dmabuf Protocol

## 1. Overview
Defines zero-copy buffer sharing between Wayland clients and the Compositor via the `linux-dmabuf-v1` protocol extension.

## 2. Architecture
- **Server**: Compositor (wlroots) manages DRM/KMS output
- **Clients**: Stream Engine (video), UI
- **Protocol**: `zwp_linux_dmabuf_v1` for buffer import

## 3. Client Roles

### Stream Engine (Video Client)
- Submits decoded video frames as Wayland surfaces
- Uses `linux-dmabuf-v1` to create `wl_buffer` from DMA-BUF
- Lower z-order (background layer)

### UI (Client)
- Chromium with Ozone/Wayland backend
- Submits UI overlay as transparent Wayland surface
- Higher z-order (foreground layer)

## 4. Compositor Responsibilities
- Receives surfaces from clients
- Composites layers by z-order
- Outputs to DRM/KMS display

## 5. Supported Formats
| Format | Usage |
|:---|:---|
| DRM_FORMAT_NV12 | Video frames (YUV) |
| DRM_FORMAT_ARGB8888 | UI overlay (RGBA) |

## 6. Error Handling
- **Client disconnect**: Compositor removes surface, continues operation
- **Compositor unavailable**: Clients fall back to Remote UI only
