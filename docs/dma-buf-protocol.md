# DMA-BUF Zero-Copy Protocol

## Overview
This protocol defines how Stream Engine shares decoded video frames with AI Inference using DMA-BUF zero-copy mechanism.

## Communication Model
- **Direction**: Stream Engine → AI Inference (send), AI Inference → Stream Engine (return)
- **Inference**: Consume the file descriptor in UDS SCM_RIGHTS to get mmaped's DMA-BUF as inference input.
- **Message Format**: JSON + file descriptor

## Message Schema

### Unified Message Structure
All messages use the same JSON format, differentiated by the `direction` field.

#### Fields
| Field | Type | Description | Applicable Direction |
|-------|------|-------------|---------------------|
| `direction` | string | `"send"` for Stream Engine → AI Inference, `"return"` for AI Inference → Stream Engine | both |
| `stream_id` | string | RTSP stream identifier | both |
| `sequence` | number | Frame sequence number | both |
| `width` | number | Frame width in pixels | both |
| `height` | number | Frame height in pixels | both |
| `format` | string | Pixel format (e.g., `"NV12"`, `"YUV420P"`) | send |
| `timestamp` | number | Frame timestamp in nanoseconds | send |
| `inference_time` | number | AI inference duration in nanoseconds | return |
| `status` | string | `"success"` or `"error"` | return |
| `model_name` | string | Model name used for inference | return |
| `timestamp` | number | Detection timestamp in nanoseconds | return |
| `detections` | array | Detection results array | return (optional) |

### Send Message Example (Stream Engine → AI Inference)
```json
{
  "direction": "send",
  "stream_id": "camera-001",
  "sequence": 12345,
  "width": 1920,
  "height": 1080,
  "format": "NV12",
  "timestamp": 1737072000000000000
}
```

### Return Message Example (AI Inference → Stream Engine)
```json
{
  "direction": "return",
  "stream_id": "camera-001",
  "sequence": 12345,
  "width": 1920,
  "height": 1080,
  "inference_time": 15600000,
  "status": "success",
  "model_name": "yolov8n.onnx",
  "timestamp": 1737072000000000000,
  "detections": [
    {
      "class_id": 0,
      "class_name": "person",
      "confidence": 0.95,
      "bbox": {
        "x": 100,
        "y": 200,
        "width": 50,
        "height": 100
      }
    }
  ]
}
```

## Protocol Flow
1. Stream Engine sends message with `direction: "send"` and DMA-BUF FD via SCM_RIGHTS
2. AI Inference imports DMA-BUF and processes frame
3. AI Inference sends return message with `direction: "return"` and `status`
4. Stream Engine reuses or releases DMA-BUF based on status

## Error Handling
- Invalid message format: Return message with `status: "error"`
- FD import failure: Return message with `status: "error"`
- Timeout: Stream Engine may forcibly reclaim DMA-BUF after timeout threshold
