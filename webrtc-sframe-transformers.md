# SFrame Transformers Architecture

This document describes the architecture and functionality of the SFrame encryption transformers in the WebRTC implementation.

## Overview

The SFrame (Secure Frame) encryption system provides two transformation approaches:

1. **Frame-level transformation** (`SFrameFrameTransformer`) - Encrypts entire media frames before packetization
2. **Packet-level transformation** (`SFramePacketTransformer`) - Encrypts individual RTP packets after packetization

Both transformers implement the `FrameTransformerInterface` and follow the RFC draft-ietf-avtcore-rtp-sframe specification.

## SFrameFrameTransformer

### Purpose

`SFrameFrameTransformer` operates on complete media frames before they are fragmented into RTP packets.

### Architecture

```
TransformableFrameInterface (frame)
    |
    v
SFrameFrameTransformer::Transform()
    |
    v
EncryptFrame()
    |
    +-- Detect frame type via MIME type
    |   (mime_type.find("video") != std::string::npos)
    |
    +-- [Video Frame Path]
    |   Cast to TransformableVideoFrameInterface
    |   Modify metadata: SetCodec(VideoCodecType::kSFrame)
    |
    v
Return encrypted frame
    |
    v
SFrame RTP Packetizer (uses kSFrame codec for video)
    |
    v
Send encrypted packets
```

### Key Implementation Details

#### Video Frame Detection

The transformer detects video frames by examining the MIME type string:

```cpp
const std::string& mime_type = frame->GetMimeType();
bool is_video = mime_type.find("video") != std::string::npos;
```

#### Video Frame Codec Modification

For video frames, the transformer:
1. Casts to `TransformableVideoFrameInterface*`
2. Accesses the video metadata
3. Sets the codec type to `kSFrame`

```cpp
auto* video_frame = static_cast<TransformableVideoFrameInterface*>(frame.get());
VideoFrameMetadata& metadata = video_frame->GetMetadata();
metadata.SetCodec(VideoCodecType::kSFrame);
```

This codec modification ensures that when the frame returns to the WebRTC pipeline, the RTP packetizer will use the SFrame codec configuration.

### Code Example

```cpp
void SFrameFrameTransformer::Transform(
    std::unique_ptr<TransformableFrameInterface> frame) {
  MutexLock lock(&callback_mutex_);
  
  if (!callback_) {
    return;
  }

  auto encrypted_frame = EncryptFrame(std::move(frame));
  if (encrypted_frame) {
    callback_->OnTransformedFrame(std::move(encrypted_frame));
  }
}

std::unique_ptr<TransformableFrameInterface>
SFrameFrameTransformer::EncryptFrame(
    std::unique_ptr<TransformableFrameInterface> frame) {
  // TODO: Actual SFrame encryption
  
  const std::string& mime_type = frame->GetMimeType();
  
  if (mime_type.find("video") != std::string::npos) {
    // Video frame: modify codec type for proper packetization
    auto* video_frame = static_cast<TransformableVideoFrameInterface*>(frame.get());
    VideoFrameMetadata& metadata = video_frame->GetMetadata();
    metadata.SetCodec(VideoCodecType::kSFrame);
  } else {
    // Audio processing to be defined
  }
  
  return frame;
}
```

## SFramePacketTransformer

### Purpose

`SFramePacketTransformer` operates on individual RTP packets after packetization.

### Architecture

```
TransformableFrameInterface (RTP packet)
    |
    v
SFramePacketTransformer::Transform()
    |
    v
EncryptPacket()
    |
    +-- Get packet payload
    |
    +-- Encrypt payload (TODO)
    |   Per RFC 9605:
    |   - Encrypt with SFrame
    |   - Add SFrame header (1-17 bytes)
    |   - Add auth tag (up to 16 bytes)
    |
    +-- Prepend SFrame RTP header byte
    |   0xC0 = 11000000 (S=1, E=1)
    |   Indicates complete SFrame in single packet
    |
    v
Return encrypted packet
    |
    v
RTP Sender (sends packet on network)
```

### Key Implementation Details

#### SFrame RTP Header Byte

The transformer prepends a single byte to indicate SFrame packet boundaries:

```cpp
// S=1, E=1 (0xC0 = 11000000)
// Complete SFrame ciphertext in a single packet
constexpr uint8_t kSFrameHeaderSinglePacket = 0xC0;
```

Bit layout:
- **S bit (Start)**: Set to 1 for first packet of SFrame
- **E bit (End)**: Set to 1 for last packet of SFrame
- For single-packet frames: Both bits are 1 (0xC0)

### Code Example

```cpp
void SFramePacketTransformer::Transform(
    std::unique_ptr<TransformableFrameInterface> packet) {
  MutexLock lock(&callback_mutex_);
  
  if (!callback_) {
    return;
  }

  auto encrypted_packet = EncryptPacket(std::move(packet));
  if (encrypted_packet) {
    callback_->OnTransformedFrame(std::move(encrypted_packet));
  }
}

std::unique_ptr<TransformableFrameInterface>
SFramePacketTransformer::EncryptPacket(
    std::unique_ptr<TransformableFrameInterface> packet) {
  ArrayView<const uint8_t> payload = packet->GetData();
  
  std::vector<uint8_t> sframe_packet;
  sframe_packet.reserve(1 + payload.size());
  
  // Prepend SFrame RTP header byte: S=1, E=1
  sframe_packet.push_back(kSFrameHeaderSinglePacket);
  
  // Append encrypted payload (TODO: actual encryption)
  sframe_packet.insert(sframe_packet.end(), payload.begin(), payload.end());
  
  packet->SetData(ArrayView<const uint8_t>(sframe_packet.data(), 
                                            sframe_packet.size()));
  
  return packet;
}
```
