# Video Receive Pipeline — SFrame Integration Plan

## Overview

This document describes all the changes needed to add the SFrame receive
pipeline to `RtpVideoStreamReceiver2`, handling both T=0 (per-frame) and
T=1 (per-packet) modes.

---

## New Components to Create

| Component | Location | Purpose |
|---|---|---|
| `SFrameDescriptor` | `modules/rtp_rtcp/source/sframe_descriptor.h` | S/E/T bit struct (header-only) |
| `SframeRtpPacketReceived` | `modules/rtp_rtcp/source/sframe_rtp_packet_received.h` | `RtpPacketReceived` + parsed descriptor |
| `SFramePacketBuffer` | `modules/video_coding/sframe_packet_buffer.{h,cc}` | Circular buffer, validates S→E runs, returns `InsertResult{packetized, packets, buffer_cleared}` |
| `SframeDepacketize()` | - | Parse + strip 1-byte descriptor from RTP payload |
| `SframeDecrypt()` | modules/sframe | Actual SFrame decryption (per-packet for T=1, per-frame for T=0) |

---

## Current Receiver Flow (no SFrame)

```
OnRtpPacket
  → ReceivePacket
      → parse_and_insert(packet)
          → VideoRtpDepacketizer::Parse     (codec depacketizer)
          → OnReceivedPayloadData           (header extensions, PacketBuffer insert)
              → PacketBuffer::InsertPacket
              → OnInsertedPacket            (assemble frame, create RtpFrameObject)
                  → OnAssembledFrame        (reference finder → decrypt → decode)
```

---

## Target Receiver Flow (with SFrame)

When `sframe_enabled` is set, `ReceivePacket` intercepts the packet **before** `parse_and_insert`:

```
ReceivePacket
  │
  ├─ [SFrame NOT enabled] → parse_and_insert(packet)  (unchanged path)
  │
  └─ [SFrame enabled]
       │
       │  1. SFrame depacketize: parse + strip the 1-byte descriptor
       │     auto sframe_pkt = SframeDepacketize(packet);
       │       → SframeRtpPacketReceived(packet, {S, E, T})
       │
       │  2. Insert into SFramePacketBuffer
       │     auto result = sframe_packet_buffer_->InsertPacket(sframe_pkt);
       │     if (result.buffer_cleared) RequestKeyFrame();
       │     if (result.packets.empty()) return;   // incomplete frame
       │
       │  3. Branch on T-bit (result.packetized)
       │
       ├─ T=1 (per-packet): decrypt BEFORE codec depacketizer
       │    for (auto& pkt : result.packets) {
       │      SframeDecrypt(pkt);                   // in-place → raw codec data
       │      parse_and_insert(pkt);                // normal codec depacketizer
       │    }                                       //   → OnReceivedPayloadData
       │                                            //   → PacketBuffer
       │                                            //   → OnInsertedPacket
       │
       └─ T=0 (per-frame): decrypt AFTER PacketBuffer assembly
            for (auto& pkt : result.packets) {
              parse_and_insert_raw(pkt);            // raw/opaque depacketizer
            }                                       //   → OnReceivedPayloadData
                                                    //   → PacketBuffer
                                                    //   → OnInsertedPacket
                                                    //       └→ decrypt assembled bitstream
                                                    //   → OnAssembledFrame
```

### T=0 Decryption Point

For T=0, packets enter `parse_and_insert` with a **raw depacketizer**
(`VideoRtpDepacketizerRaw`) since payloads are opaque ciphertext.  `PacketBuffer`
reassembles them and `AssembleFrame()` concatenates the fragments into
`bitstream`.  **Decryption happens in `OnInsertedPacket` after assembly**,
before `OnAssembledFrame` is called.

The `RTPVideoHeader` is already populated correctly at this point because
`ParseGenericDependenciesExtension()` reads the dependency descriptor from
the **RTP header** (outside the encrypted payload).  Codec type, frame type,
spatial/temporal indices are all available without decryption.

---

## Required Changes to `RtpVideoStreamReceiver2`

### 1. Signature Changes

Add `bool sframe_per_frame_decrypt = false` to thread the T-bit through the pipeline:

**`rtp_video_stream_receiver2.h`:**
```cpp
bool OnReceivedPayloadData(CopyOnWriteBuffer codec_payload,
                           const RtpPacketReceived& rtp_packet,
                           const RTPVideoHeader& video,
                           int times_nacked,
                           bool sframe_per_frame_decrypt = false);

void OnInsertedPacket(video_coding::PacketBuffer::InsertResult result,
                      bool sframe_per_frame_decrypt = false)
    RTC_RUN_ON(packet_sequence_checker_);
```

The default `= false` means all existing call sites (tests, padding via
`NotifyReceiverOfEmptyPacket`, non-SFrame paths) are unchanged.  Only the
SFrame T=0 path passes `true`.

### 2. SFrame Intercept in `ReceivePacket`

In `ReceivePacket`, **before** `parse_and_insert(packet)` is called, insert the
SFrame branch:

```cpp
if (sframe_packet_buffer_) {
  // 1. Depacketize: parse + strip the 1-byte SFrame descriptor.
  auto sframe_pkt = SframeDepacketize(packet);

  // 2. Insert into SFramePacketBuffer.
  auto result = sframe_packet_buffer_->InsertPacket(sframe_pkt);
  if (result.buffer_cleared) {
    RequestKeyFrame();
  }

  if (result.packets.empty()) {
    return;  // incomplete frame, wait for more packets
  }

  // 3. Branch on T-bit.
  if (result.packetized) {
    // T=1: decrypt each packet, then normal codec depacketize.
    for (auto& pkt : result.packets) {
      SframeDecrypt(pkt);       // in-place
      parse_and_insert(pkt);    // codec depacketizer → PacketBuffer
    }
  } else {
    // T=0: payloads are opaque ciphertext → raw depacketizer.
    // Pass sframe_per_frame_decrypt=true so OnInsertedPacket
    // knows to decrypt after assembly.
    for (auto& pkt : result.packets) {
      parse_and_insert_raw(pkt, /*sframe_per_frame_decrypt=*/true);
    }
  }
  return;
}
// ... existing non-SFrame path ...
```

### 3. T=0 Decryption in `OnInsertedPacket`

After `AssembleFrame()` produces `bitstream`, check the flag:

```cpp
if (sframe_per_frame_decrypt) {
  // bitstream is the full SFrame ciphertext (concatenated fragments).
  // Decrypt → raw encoded bitstream (e.g. H.264 NALUs).
  auto plaintext = SframeDecrypt(bitstream);
  bitstream = std::move(plaintext);
}
OnAssembledFrame(std::move(frame));
```

### 4. `SframeDepacketize`

A free function or method that:
1. Reads byte 0 of the RTP payload → `SFrameDescriptor{S, E, T}`
2. Strips byte 0 from the payload (the rest is encrypted data)
3. Returns `SframeRtpPacketReceived(packet, descriptor)`

```cpp
SframeRtpPacketReceived SframeDepacketize(const RtpPacketReceived& packet) {
  auto payload = packet.payload();
  RTC_CHECK(!payload.empty());
  uint8_t desc_byte = payload[0];
  SFrameDescriptor descriptor{
      .start = (desc_byte & 0x80) != 0,
      .end = (desc_byte & 0x40) != 0,
      .packetized = (desc_byte & 0x20) != 0,
  };
  // Create a copy with the descriptor byte stripped.
  RtpPacketReceived stripped = packet;
  // ... strip first byte from payload ...
  return SframeRtpPacketReceived(stripped, descriptor);
}
```

### 5. `SFramePacketBuffer`

A circular buffer that sits **before** `PacketBuffer` and **before** SFrame
decryption.  It collects incoming SFrame RTP packets and validates complete
S→E runs per draft-ietf-avtcore-rtp-sframe section 5.2.

**Location:** `modules/video_coding/sframe_packet_buffer.{h,cc}`

#### Public API

```cpp
class SFramePacketBuffer {
 public:
  struct InsertResult {
    bool packetized = false;                    // T-bit (per-packet vs per-frame)
    std::vector<RtpPacketReceived> packets;     // validated S→E run, in order
    bool buffer_cleared = false;                // overflow → caller requests keyframe
  };

  explicit SFramePacketBuffer(size_t start_size = 96, size_t max_size = 2048);

  InsertResult InsertPacket(const SframeRtpPacketReceived& packet);
  void ClearTo(uint16_t seq_num);
  void Clear();
};
```

#### Internal Design

- **Circular buffer** indexed by `seq_num % buffer_size`, same pattern as
  the existing `PacketBuffer`.  Starts at 96 slots, doubles up to 2048.
- **`InsertPacket` flow:**
  1. `UpdateWindowStart(seq_num)` — reject packets older than the window.
  2. `ResolveSlot(seq_num)` — find a buffer slot; returns `SlotOutcome`:
     - `kOk` — empty slot found (possibly after expansion).
     - `kDuplicate` — packet already buffered, return empty result.
     - `kFull` — buffer at max capacity, `Clear()` + set `buffer_cleared`.
  3. Store packet in the slot.
  4. `FindFrame(seq_num)` — check if insertion completes a frame:
     - `FindFrameStart` — walk backward from `seq_num` looking for S=1.
     - `FindFrameEnd` — walk forward from `seq_num` looking for E=1.
     - `AssembleFrame` — validate T-bit, payload type, and timestamp are
       consistent across the entire run.  Move packets out of the buffer
       into the result.  On validation failure, drop the frame and clear
       those slots.

#### Validation Rules (in `AssembleFrame`)

All packets in a S→E run must share the same:
- **T-bit** — mixed per-packet/per-frame is invalid.
- **Payload type** — all packets belong to the same codec.
- **RTP timestamp** — all packets belong to the same frame.

If any mismatch is found, the entire run is dropped and the slots are cleared.

#### Window Management

- `UpdateWindowStart` tracks the lowest seq num.  Reordered packets that
  still fit in the buffer extend the window backward.  After `ClearTo()`,
  the window start is locked — late packets before the cleared point are
  rejected.
- `ClearTo(seq_num)` discards all packets up to and including `seq_num`,
  locking the window so it cannot move backward past that point.

#### BUILD.gn

```gn
# modules/video_coding/BUILD.gn
rtc_library("sframe_packet_buffer") {
  sources = [
    "sframe_packet_buffer.cc",
    "sframe_packet_buffer.h",
  ]
  deps = [
    "../../rtc_base:logging",
    "../../rtc_base:rtc_numerics",
    "../rtp_rtcp:rtp_rtcp_format",
    "../rtp_rtcp:sframe_descriptor",
  ]
}
```

---

## Key Design Decisions

1. **Both T=0 and T=1 go through `parse_and_insert` → `PacketBuffer`.**
   No separate "bypass" path.  The difference is only *when* decryption
   happens (before vs. after assembly).

2. **Explicit parameter (`sframe_per_frame_decrypt`) instead of member flag.**
   The T-bit is threaded as a function argument through `OnReceivedPayloadData`
   → `OnInsertedPacket`.  This gives clear data flow with no hidden state.
   Default `= false` keeps all existing call sites untouched.

3. **T=0 uses `VideoRtpDepacketizerRaw`** because the codec payload is
   encrypted ciphertext — the real codec depacketizer can't parse it.
   The dependency descriptor (in the RTP header, outside the encrypted
   payload) provides all the framing metadata needed.

4. **`SFramePacketBuffer` sits before `PacketBuffer`.**  It validates
   S→E runs and extracts the T-bit.  Only complete runs are forwarded
   to the codec depacketizer / `PacketBuffer`.

5. **`sframe_packet_buffer_` constructed conditionally** only when
   `config_.crypto_options.sframe.require_frame_encryption` is set,
   same guard as the existing `buffered_frame_decryptor_`.
