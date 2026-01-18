# UDS Detection Publisher Protocol

## Overview
This protocol defines how AI Inference publishes detection results to Network Gateway and Stream Engine using Unix Domain Socket.

## Communication Model
- **Direction**: AI Inference → Network Gateway
- **Direction**: Reuse from DMA-BUF return line, AI Inference → Stream Engine (persistence)
- **Transport**: Unix Domain Socket (SOCK_SEQPACKET)
- **Connection Type**: 1:1 (one UDS socket per service)

## Message Format
Reuses DMA-BUF return message schema. See `shared/schema/detection-schema.json`.

## Socket Path Conventions
- **Gateway**: `/tmp/ai-gateway.sock`
- **Stream Engine**: `/tmp/ai-stream.sock`

## Error Handling
If connection is lost, AI Inference should attempt to reconnect with exponential backoff.
