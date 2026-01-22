# DMA-BUF Zero-Copy Protocol

## 1. Overview
This protocol defines the low-level mechanism for zero-copy video frame sharing between the **Stream Engine** (Producer) and **AI Inference** (Consumer). It leverages Unix Domain Sockets (UDS) with `SCM_RIGHTS` to pass file descriptors (FDs) pointing to GPU/VPU memory buffers, avoiding expensive CPU copies.

## 2. Transport Mechanism
- **Socket Type**: `SOCK_SEQPACKET` (Preserves message boundaries).
- **Payload**:
  1. **Primary Data**: JSON metadata (refer to `shared/schema/detection-schema.json`).
  2. **Ancillary Data**: The DMA-BUF file descriptor (sent via `sendmsg` with `SCM_RIGHTS`).

## 3. Protocol Lifecycle

### Phase 1: Allocation (Stream Engine)
1. Stream Engine initializes a hardware decoder (V4L2 Request API or VA-API).
2. Decoded frames are allocated in hardware-accessible DMA-BUF memory.
3. Stream Engine maintains the master reference to the buffer.

### Phase 2: Handover (Stream Engine → AI Inference)
- **Message Direction**: `send`
- **Action**: Stream Engine sends the JSON header containing stream metadata (width, height, format) along with the FD.
- **FD Handling**: The kernel duplicates the FD for the receiving process. The AI service now shares ownership of the underlying memory.

### Phase 3: Inference (AI Inference)
1. AI Service receives the message and extracts the FD.
2. **Memory Mapping**: Uses platform-specific APIs (e.g., `rknn_import`, `vaImportSurface`) to map the FD without CPU copy.
3. **Processing**: Runs the ONNX model against the mapped buffer.

### Phase 4: Release & Result (AI Inference → Stream Engine)
- **Message Direction**: `return`
- **Action**: AI Service sends the inference results (bounding boxes, class IDs) back to Stream Engine via the same UDS connection.
- **Resource Management**:
    - AI Service closes its copy of the FD immediately after processing.
    - Stream Engine receives the result, synchronizes it with the display timestamp, and releases its reference to the DMA-BUF.
    - **Buffer Recycling**: The memory's ownership returns to the GStreamer pipeline within the Stream Engine. GStreamer's buffer pool manager decides whether to reuse the buffer for a subsequent frame or release it back to the system.

## 4. Error Handling
- **FD Leak Prevention**: If the AI Inference service crashes, the kernel automatically closes its open FDs. Stream Engine monitors the socket status to reclaim resources.
- **Timeout**: If no "return" message is received within a specific threshold (e.g., 200ms), Stream Engine treats the frame as processed (no detections) and recycles the buffer to prevent memory exhaustion.
- **Mapping Failures**: If `mmap` fails (e.g., incompatible formats), AI Service must immediately send an error status return message and close the FD.
