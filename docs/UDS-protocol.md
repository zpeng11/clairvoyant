# UDS Protocols

## Overview
This file defines existing UDS protocols for local communication in the system.

## Usage Paths
1. Stream Engine -> AI Inference. Supporting DMA-BUF transfer, use detection schema.
2. AI Inference -> Stream Engine. Supporting DMA-BUF memory management, use detection schema.
3. AI Inference -> Network Gateway. Use detection schema.
4. Network Gateway -> Stream Engine. Sending streaming configs including plane dispaly layout, use stream config schema.
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

## Stream Config Schema

### Message Structure
See `services/common/schema/video-config-schema.json`. This is a map where each key is a `stream_id` and the value is the stream's configuration.

#### Fields (per stream)
| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `stream_url` | string | RTSP source URL | yes |
| `x` | number | Normalized X position (0-1) | yes |
| `y` | number | Normalized Y position (0-1) | yes |
| `width` | number | Normalized width (0-1) | yes |
| `height` | number | Normalized height (0-1) | yes |
| `rotation` | number | Rotation in degrees | no (default: 0) |
| `visibility` | boolean | Toggle display visibility | no (default: true) |
| `color` | boolean | Enable color | no (default: true) |
| `crop` | object | Source crop region {x, y, width, height} | no |
| `region_of_interest`| object | AI ROI region {x, y, width, height} | no |

### Example (Network Gateway → Stream Engine)
```json
{
  "camera-001": {
    "stream_url": "rtsp://192.168.1.100:554/live",
    "x": 0.0,
    "y": 0.0,
    "width": 0.5,
    "height": 0.5,
    "visibility": true,
    "color": true
  }
}
```

## Inference Config Schema

### Message Structure
See `services/common/schema/inference-config-schema.json`. This hierarchical object defines global model and runtime parameters.

#### model_config Fields
| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `model_name` | string | Name of the ONNX model file | yes |
| `provider` | string | Execution Provider (`cpu`, `rknn`, `acl`, `openvino`, `tensorrt`) | no |

#### inference_params Fields
| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `confidence_threshold` | number | Min detection confidence (0-1) | no |
| `nms_threshold` | number | IoU threshold for NMS (0-1) | no |

### Example (Network Gateway → AI Inference)
```json
{
  "model_config": {
    "model_name": "yolov8n.onnx",
    "provider": "rknn"
  },
  "inference_params": {
    "confidence_threshold": 0.5,
    "nms_threshold": 0.45
  }
}
```

