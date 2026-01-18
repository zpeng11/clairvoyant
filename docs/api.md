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
- Unified message schema for both send and return directions
- Return message includes detection results (model_name, timestamp, detections array)

**Detailed Specification**: See `dma-buf-protocol.md`
**Data Structure**: See `shared/schema/detection-schema.json`

### 4.2 UDS Detection Publisher
Defines the Unix Domain Socket mechanism for AI Inference to publish detection results to Gateway and Stream Engine.

**Covers:**
- UDS transport (SOCK_SEQPACKET)
- 1:1 connection model (AI Inference to Gateway, AI Inference to Stream Engine)
- Reuses DMA-BUF return message schema

**Detailed Specification**: See `UDS-protocol.md`

---

## 5. Implementation Notes

### 5.1 Gateway Service Implementation
Network Gateway REST API implementation is located at:
- `services/network-gateway/src/api/streams.ts` - Streams endpoints
- `services/network-gateway/src/api/recordings.ts` - Recordings endpoints
- `services/network-gateway/src/api/config.ts` - Configuration endpoints
- `services/network-gateway/src/websocket/server.ts` - WebSocket server

### 5.2 Client Libraries
- REST API clients can use standard HTTP libraries (e.g., axios, fetch)
- WebSocket clients should support JSON message parsing
- Bearer token authentication must be included in all requests

### 5.3 MediaMTX Integration
Gateway uses MediaMTX as the RTSP proxy. MediaMTX API is accessed via the client wrapper at:
- `services/network-gateway/src/mediamtx/client.ts`
