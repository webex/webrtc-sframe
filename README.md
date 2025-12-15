# WebRTC SFrame Integration

This document describes how to integrate SFrame (Secure Frame) encryption into WebRTC applications. SFrame provides end-to-end media security that works even when media flows through untrusted servers or intermediaries.

## Overview

SFrame encryption can be applied at two different levels:
- **Per-frame**: Encrypts complete video/audio frames before packetization
- **Per-packet**: Encrypts individual RTP packets after packetization

## Architecture

The SFrame implementation leverages and extends WebRTC's existing transformer infrastructure:

1. **`FrameTransformerHost`** - Base interface that provides `SetFrameTransformer()` and `SetPacketTransformer()` methods.
2. **`FrameTransformerInterface`** - For frame-level and packet-level encryption (defined in `api/frame_transformer_interface.h`)
3. **`TransformableFrameInterface`** - Base interface to be used for Frame and Packet level encryption (defined in `api/frame_transformer_interface.h`)

### Extend `FrameTransformerHost`

`FrameTransformerHost` should have an ability to handle both - frame and packet level transformers. Extend it to allow setting also packet level transformers.

```cpp
class FrameTransformerHost {
 public:
  virtual ~FrameTransformerHost() {}
  virtual void SetFrameTransformer(
      scoped_refptr<FrameTransformerInterface> frame_transformer) = 0;
  virtual void SetPacketTransformer(
      scoped_refptr<FrameTransformerInterface> packet_transformer) = 0;
};
```

### RTP Senders/Receivers

RTP interfaces inherit from `FrameTransformerHost` to support SFrame injection:

`api/rtp_sender_interface.h`
```cpp
class RtpSenderInterface : public FrameTransformerHost {
 public:
  void SetFrameTransformer(
      scoped_refptr<FrameTransformerInterface> frame_transformer) {};

  void SetPacketTransformer(
      scoped_refptr<FrameTransformerInterface> packet_transformer) {};
};
```

`api/rtp_receiver_interface.h`
```cpp
class RtpReceiverInterface : public FrameTransformerHost {
 public:
  void SetFrameTransformer(
      scoped_refptr<FrameTransformerInterface> frame_transformer) {};

  void SetPacketTransformer(
      scoped_refptr<FrameTransformerInterface> packet_transformer) {};
};
```
