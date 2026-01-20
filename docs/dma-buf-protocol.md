# DMA-BUF Zero-Copy Protocol

## Overview
This protocol defines how Stream Engine shares decoded video frames with AI Inference using DMA-BUF zero-copy mechanism.

## Communication Model
- **Direction**: Stream Engine → AI Inference, using UDS to send FD.
- **Consumer**: Consume the file descriptor in UDS SCM_RIGHTS to get DMA-BUF as inference input.
- **Message Format**: JSON + file descriptor

## Error Handling
- Invalid message format, FD mmap failure: Return UDS message with `status: "error"`
- Timeout: Stream Engine may forcibly reclaim DMA-BUF after timeout threshold
