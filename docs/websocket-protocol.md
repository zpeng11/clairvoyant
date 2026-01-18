# WebSocket AI Detection Protocol

## Overview
This protocol defines how Network Gateway pushes real-time AI detection metadata to frontend clients (Display service, remote browsers) via WebSocket.

## Communication Model
- **Direction**: Network Gateway → Clients (unidirectional push)
- **Transport**: WebSocket
- **Message Format**: JSON
- **Purpose**: Real-time delivery of AI inference detection results

## Connection & Authentication

### Connection Endpoint
```
ws://gateway-host:port/ws
```

### Authentication Methods
Two authentication methods are supported. **Only one method is required to authenticate successfully.**

#### Method 1: URL Parameter
Include token as query parameter during connection establishment:
```
ws://gateway-host:port/ws?token=YOUR_ACCESS_TOKEN
```

#### Method 2: First Message
Send authentication message as the first message after connection established:
```json
{
  "type": "auth",
  "token": "YOUR_ACCESS_TOKEN"
}
```

#### Authentication Logic
- **Success**: If either URL token or first message token is valid, client enters authenticated state and Gateway starts pushing detection data.
- **Failure**: If neither method provides a valid token, Gateway immediately closes the connection.

## Message Format

### Detection Message Structure
Gateway pushes detection messages with the same schema as DMA-BUF return messages. See `shared/schema/detection-schema.json` for complete JSON Schema definition.

#### Required Fields (return direction)
| Field | Type | Description |
|-------|------|-------------|
| `direction` | string | Must be `"return"` |
| `stream_id` | string | RTSP stream identifier |
| `sequence` | number | Frame sequence number |
| `width` | number | Frame width in pixels |
| `height` | number | Frame height in pixels |
| `inference_time` | number | AI inference duration in nanoseconds |
| `status` | string | `"success"` or `"error"` |
| `model_name` | string | Model name used for inference |
| `timestamp` | number | Detection timestamp in nanoseconds |
| `detections` | array | Detection results array (optional) |

#### Detection Object Fields
| Field | Type | Description |
|-------|------|-------------|
| `class_id` | number | Class ID |
| `class_name` | string | Class name |
| `confidence` | number | Confidence score (0-1) |
| `bbox` | object | Bounding box with `x`, `y`, `width`, `height` |

### Example Message
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

## Heartbeat Mechanism

### Client Responsibility
- Client must send a heartbeat message every 30 seconds
- Heartbeat format (WebSocket ping frame preferred):
  - Option A: WebSocket PING frame (recommended, most efficient)
  - Option B: JSON message: `{"type": "ping"}`

### Server Behavior
- Gateway tracks last heartbeat timestamp for each client
- If no heartbeat received within 30 seconds, Gateway closes the connection
- Heartbeat messages do not affect message queue or detection push timing

## Error Handling

### Authentication Failure
- Gateway immediately closes WebSocket connection with close code (e.g., 4000)
- No error message sent

### Heartbeat Timeout
- Gateway closes connection due to inactivity
- Client should reconnect with fresh authentication

### Invalid Message Format
- Gateway logs error but continues connection
- Malformed messages are silently ignored

### Gateway Service Disruption
- Client should implement reconnection logic with exponential backoff
- Reuse existing authentication method (URL or first message)
