# Clairvoyant API Documentation

## 1. API Overview

### 1.1 Purpose and Scope
The Clairvoyant API provides a unified interface for managing surveillance streams, recordings, and system configuration. It enables remote clients (browsers, third-party applications) to interact with the system through RESTful endpoints and real-time WebSocket connections.

### 1.2 Authentication
All API endpoints require authentication via Bearer token. Tokens are issued by the Network Gateway and should be included in the `Authorization` header for every request.

### 1.3 Common Response Format
All REST API responses follow a consistent JSON structure:
- Success responses include HTTP status codes 2xx
- Error responses include HTTP status codes 4xx/5xx with error details
- Detailed specifications are documented in individual endpoint references

---

## 2. REST API Endpoints

### 2.1 Streams Management
Manages registration and configuration of RTSP video sources.

**Endpoints Summary:**
- `GET /api/streams` - List all registered streams
- `POST /api/streams` - Register a new RTSP source
- `GET /api/streams/:id` - Retrieve specific stream details
- `PUT /api/streams/:id` - Update stream configuration
- `DELETE /api/streams/:id` - Remove a stream

**Detailed Specification**: See `shared/protocols/rest-api/streams.yaml`

**Implementation**: `services/network-gateway/src/api/streams.ts`

### 2.2 Recordings Management
Manages access to recorded video files.

**Endpoints Summary:**
- `GET /api/recordings` - List all recordings
- `GET /api/recordings/:id` - Retrieve recording metadata
- `DELETE /api/recordings/:id` - Delete a recording file

**Detailed Specification**: See `shared/protocols/rest-api/recordings.yaml`

**Implementation**: `services/network-gateway/src/api/recordings.ts`

### 2.3 System Configuration
Manages global system settings.

**Endpoints Summary:**
- `GET /api/config` - Retrieve system configuration
- `PUT /api/config` - Update system configuration

**Detailed Specification**: TODO

**Implementation**: TODO

---

## 3. WebSocket API

### 3.1 Connection Endpoint
WebSocket endpoint for real-time AI metadata streaming from the Gateway to connected clients (Display service, remote browsers).

**Endpoint**: `ws://gateway-host:port/ws`

### 3.2 Message Format
Real-time AI detection metadata including bounding boxes, classes, confidence scores, and tracking information.

**Detailed Specification**: See `shared/protocols/websocket/schema.txt`

### 3.3 Client Integration
- Display service establishes WebSocket connection on startup to receive AI metadata for UI rendering
- Remote browsers connect to the same endpoint for live detection visualization in the web interface
- Detections are synchronized with the video stream via timestamps

---

## 4. IPC Protocol References

### 4.1 PipeWire Protocol
Defines the zero-copy frame sharing mechanism between Stream Engine and AI Inference using PipeWire.

**Covers:**
- Stream Engine hosts PipeWire daemon and uses GStreamer `pipewiresink`
- AI Inference connects as PipeWire client using `pipewire-rs` (native API)
- Zero-copy DMA-BUF frames negotiated and shared via PipeWire streams

**Detailed Specification**: See `pipewire-protocol.md`
**Data Structure**: See `services/common/schema/frame-metadata-schema.json`

### 4.2 UDS Messages
Defines the Unix Domain Socket mechanism for service coordination and metadata exchange.

**Covers:**
- Using `SOCK_SEQPACKET` for reliable, message-oriented communication
- 1:1 bidirectional connections:
  - Network Gateway ↔ Stream Engine (Video layout, stream configs)
  - Network Gateway ↔ AI Inference (AI configs, detection results)

**Detailed Specification**: See `UDS-protocol.md`

### 4.3 Wayland linux-dmabuf Protocol
Defines the zero-copy buffer sharing mechanism between Wayland clients and the Compositor.

**Covers:**
- Stream Engine (client) submits video surfaces via `linux-dmabuf-v1`
- Display (client) submits UI surfaces via `linux-dmabuf-v1`
- Compositor (server) receives and composites surfaces for DRM/KMS output

**Detailed Specification**: See `wayland-dmabuf-protocol.md`

