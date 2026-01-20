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

**Detailed Specification**: See `shared/protocols/rest-api/config.yaml`

**Implementation**: `services/network-gateway/src/api/config.ts`

---

## 3. WebSocket API

### 3.1 Connection Endpoint
WebSocket endpoint for real-time AI metadata streaming from the Gateway to connected clients (Display service, remote browsers).

**Endpoint**: `ws://gateway-host:port/ws`

### 3.2 Message Format
Real-time AI detection metadata including bounding boxes, classes, confidence scores, and tracking information.

**Detailed Specification**: See `shared/protocols/websocket/schema.txt`

### 3.3 Client Integration
- Display service establishes WebSocket connection on startup to receive AI metadata for UI overlay
- Remote browsers can connect to the same endpoint for live detection visualization

---

## 4. IPC Protocol References

### 4.1 DMA-BUF Protocol
Defines the zero-copy frame sharing mechanism between Stream Engine and AI Inference. Extended to support detection results in return messages.

**Covers:**
- DMA-BUF file descriptor sharing via Unix Domain Socket SOCK_SEQPACKET
- Map FD to memory for next step

**Detailed Specification**: See `dma-buf-protocol.md`
**Data Structure**: See `shared/schema/detection-schema.json`'s sending schema

### 4.2 UDS messages
Defines the Unix Domain Socket mechanism for services to communicate

**Covers:**
- Using SOCK_SEQPACKET to avoid framing problem
- 1:1 connection dual directional model (Gateway to AI Inference, Gateway to Stream Engine)

**Detailed Specification**: See `UDS-protocol.md`

