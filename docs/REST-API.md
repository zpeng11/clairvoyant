# Clairvoyant REST API Specification

## 1. Overview

Base URL: `/api`

### 1.1 Authentication
All requests must include a Bearer token in the `Authorization` header:
`Authorization: Bearer <token>`

### 1.2 Response Format

**Success (200 OK)**
```json
{
  "data": { ... }
}
```

**Error (4xx/5xx)**
```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human readable message"
  }
}
```

---

## 2. Streams Management

### 2.1 Stream Object
Aligned with `video-config-schema.json`.

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique Stream ID (generated) |
| `name` | string | Human readable name |
| `stream_url` | string | RTSP Source URL |
| `x` | number | Normalized X position (0-1) |
| `y` | number | Normalized Y position (0-1) |
| `width` | number | Normalized width (0-1) |
| `height` | number | Normalized height (0-1) |
| `rotation` | number | Rotation degrees (0, 90, 180, 270) |
| `visibility` | boolean | Visible on display |
| `color` | boolean | Color enabled |
| `crop` | object | `{x, y, width, height}` (Optional) |
| `region_of_interest` | object | `{x, y, width, height}` (Optional) |

### 2.2 Endpoints

#### List Streams
`GET /streams`

**Response**
```json
{
  "data": [
    {
      "id": "cam-01",
      "name": "Front Door",
      "stream_url": "rtsp://192.168.1.10:554/live",
      "x": 0.0, "y": 0.0, "width": 0.5, "height": 0.5,
      "rotation": 0,
      "visibility": true,
      "color": true
    }
  ]
}
```

#### Register Stream
`POST /streams`

**Request Body**
```json
{
  "name": "Backyard",
  "stream_url": "rtsp://192.168.1.11:554/live",
  "x": 0.5, "y": 0.0, "width": 0.5, "height": 0.5
}
```

**Response**
Returns the created Stream Object.

#### Get Stream Details
`GET /streams/:id`

#### Update Stream
`PUT /streams/:id`

**Request Body**
Partial object allowed.
```json
{
  "visibility": false,
  "x": 0.1
}
```

#### Delete Stream
`DELETE /streams/:id`

---

## 3. Recordings Management

### 3.1 Recording Object

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique Recording ID |
| `stream_id` | string | ID of the source stream |
| `start_time` | string | ISO 8601 Timestamp |
| `end_time` | string | ISO 8601 Timestamp |
| `size_bytes` | number | File size in bytes |
| `filename` | string | Physical filename |

### 3.2 Endpoints

#### List Recordings
`GET /recordings`

**Query Parameters**
- `stream_id` (optional): Filter by stream
- `from` (optional): Start timestamp (ISO 8601)
- `to` (optional): End timestamp (ISO 8601)

#### Get Recording Metadata
`GET /recordings/:id`

#### Delete Recording
`DELETE /recordings/:id`

---

## 4. System Configuration

### 4.1 Configuration Object
TODO: Define global system settings (NTP, Logging, Storage thresholds, etc.)

### 4.2 Endpoints

#### Get System Config
`GET /config`

**Response**
TODO

#### Update System Config
`PUT /config`

**Request Body**
TODO
