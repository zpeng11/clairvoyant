# PipeWire Video Frame Protocol

## 1. Overview
Defines zero-copy video frame sharing between Stream Engine and AI Inference using PipeWire.

## 2. Architecture
- **Producer**: Stream Engine hosts PipeWire daemon, uses GStreamer `pipewiresink`
- **Consumer**: AI Inference connects via `pipewire-rs` native API
- **Transport**: DMA-BUF buffers negotiated through PipeWire streams

## 3. Data Flow
1. Stream Engine decodes RTSP → DMA-BUF frames
2. GStreamer `pipewiresink` publishes frames to PipeWire
3. AI Inference receives frames via PipeWire client
4. AI Inference maps DMA-BUF for zero-copy inference

## 4. Frame Metadata
| Field | Type | Description |
|:---|:---|:---|
| width | u32 | Frame width in pixels |
| height | u32 | Frame height in pixels |
| format | FourCC | Pixel format (NV12, YUV420P) |
| timestamp | u64 | PTS in nanoseconds |
| stream_id | string | Source stream identifier |

## 5. Error Handling
- **Consumer disconnect**: Producer continues, buffers recycled by GStreamer
- **Producer unavailable**: Consumer waits with reconnection logic
