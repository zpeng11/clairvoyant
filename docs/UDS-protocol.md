# UDS Protocols

## Overview
This file defines existing UDS protocols for local communication in the system.

## Usage Paths
1. Stream Engine -> AI Inference. Supporting DMA-BUF transfer, use detection schema.
2. AI Inference -> Stream Engine. Supporting DMA-BUF memory management, use detection schema.
3. AI Inference -> Network Gateway. Use detection schema.
4. Network Gateway -> Stream Engine. Sending streaming configs and video plane dispaly layout combination, use stream config schema, may embed display layout schema.
5. Network Gateway -> AI Inference. Sending AI Inference configs, use inference config schema.

## Detection Schema

### Unified Message Structure
See `shared/schema/detection-schema.json`, send and receive differentiated by the `direction` field.

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

### Return Message Example (AI Inference → Stream Engine/ Network Gateway)
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
