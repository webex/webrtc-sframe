# Video Receive Pipeline — SFrame Integration Plan

## Table of Contents

- [Overview](#overview)
- [New Components](#new-components)
- [Current Receiver Flow (no SFrame)](#current-receiver-flow-no-sframe)
- [SFrame Intercept in `ReceivePacket`](#sframe-intercept-in-receivepacket)
- [T=0 Decryption Signal Options](#t0-decryption-signal-options)
  - [Option 1: Parameter Threading](#option-1-parameter-threading-sframe_per_frame_decrypt-bool)
  - [Option 2: `RTPVideoHeader` Tag](#option-2-rtpvideoheader-tag-sframe_encrypted-bool)
- [Components](#components)
  - [`SFrameDescriptor`](#sframedescriptor)
  - [`SframeRtpPacketReceived`](#sframertppacketreceived)
  - [`VideoRtpDepacketizerSframe`](#videortpdepacketizersframe)
  - [`SFramePacketBuffer`](#sframepacketbuffer)
  - [`SframeDecrypter`](#sframedecrypter)

## Overview

This document describes the changes needed to add the SFrame receive
pipeline to `RtpVideoStreamReceiver2`, handling both T=0 (per-frame) and
T=1 (per-packet) modes.

Two implementation options are presented for review.

---

## New Components

| Component | Purpose |
|---|---|
| `SFrameDescriptor` | S/E/T bit struct (header-only) |
| `SframeRtpPacketReceived` | `RtpPacketReceived` + parsed descriptor |
| `SFramePacketBuffer` | Circular buffer, validates S→E runs |
| `VideoRtpDepacketizerSframe` | Depacketizer: parses + strips 1-byte descriptor from RTP payload |
| SFrame decrypter | Actual SFrame decryption (per-packet for T=1, per-frame for T=0) |

---

## Current Receiver Flow (no SFrame)

```
OnRtpPacket
  → ReceivePacket
      → parse_and_insert(packet)
          → VideoRtpDepacketizer::Parse     (codec depacketizer)
          → OnReceivedPayloadData
              → PacketBuffer::InsertPacket
              → OnInsertedPacket            (assemble frame, create RtpFrameObject)
                  → OnAssembledFrame        (reference finder → decrypt → decode)
```

---

## SFrame Intercept in `ReceivePacket`

When SFrame is enabled, `ReceivePacket` intercepts the packet **before**
the normal codec depacketizer path:

```
ReceivePacket
  │
  ├─ [SFrame NOT enabled] → parse_and_insert(packet)  (unchanged)
  │
  └─ [SFrame enabled]
       │
       │  1. sframe_depacketizer_->Parse(packet)
       │       → SframeRtpPacketReceived(packet, {S, E, T})
       │
       │  2. sframe_packet_buffer_->InsertPacket(sframe_pkt)
       │     if (result.buffer_cleared) RequestKeyFrame();
       │     if (result.packets.empty()) return;   // incomplete frame
       │
       │  3. Branch on T-bit
       │
       ├─ T=1 (per-packet): decrypt each packet → parse_and_insert()
       │    (normal codec depacketizer → PacketBuffer → OnInsertedPacket)
       │
       └─ T=0 (per-frame): raw depacketizer → OnReceivedPayloadData
            (PacketBuffer → OnInsertedPacket → OnAssembledFrame → decrypt)
```

### Step-by-step

1. **Parse the SFrame descriptor** — read the 1-byte SFrame payload
   descriptor (S/E/T bits) and strip it from the RTP payload.  The result
   is an `SframeRtpPacketReceived` that pairs the original packet with its
   parsed descriptor.

2. **Insert into `SFramePacketBuffer`** — the buffer collects packets and
   validates complete S→E runs (contiguous sequence of packets from
   start-of-frame to end-of-frame).  If the run is incomplete, buffer the
   packet and wait.  If the buffer overflows, clear it and request a
   keyframe.

3. **Branch on the T-bit** — once a complete S→E run is available:
   - **T=1 (per-packet):** each packet's payload is individually encrypted.
     Decrypt each packet first, then feed the cleartext through the normal
     codec depacketizer path.  From this point on, the pipeline is
     identical to the non-SFrame flow.
   - **T=0 (per-frame):** the entire frame is encrypted as one unit, split
     across packets.  The individual packet payloads are opaque ciphertext
     that the codec depacketizer cannot parse.  Instead, use a raw
     depacketizer to pass them through `OnReceivedPayloadData` →
     `PacketBuffer` → `OnInsertedPacket`, where they are reassembled into
     a single bitstream.  Frame-level decryption happens at
     `OnAssembledFrame` after assembly (the two options differ on how
     the signal reaches that point — see below).

`SframeRtpPacketReceived` is used at the SFramePacketBuffer boundary —
it carries the parsed S/E/T descriptor so the buffer can validate frame
boundaries.  It does **not** propagate past the SFrame intercept block;
downstream everything flows as `RtpPacketReceived`.

---

## T=0 Decryption Signal Options

The SFrame intercept and T=1 path are the same in both options.  The
difference is how the T=0 path signals `OnAssembledFrame` that the
reassembled bitstream needs per-frame decryption.  Option 1 threads an
explicit bool parameter through three methods.  Option 2 piggybacks on
`RTPVideoHeader` so no signatures change.

## Option 1: Parameter Threading (`sframe_per_frame_decrypt` bool)

### Approach

Thread a `bool sframe_per_frame_decrypt = false` parameter through
`OnReceivedPayloadData` → `OnInsertedPacket` → `OnAssembledFrame`.
The T=0 path passes `true`.  Decryption happens at `OnAssembledFrame`
when the bool is set.

### Signature Changes

```cpp
// rtp_video_stream_receiver2.h
bool OnReceivedPayloadData(CopyOnWriteBuffer codec_payload,
                           const RtpPacketReceived& rtp_packet,
                           const RTPVideoHeader& video,
                           int times_nacked,
                           bool sframe_per_frame_decrypt = false);

void OnInsertedPacket(video_coding::PacketBuffer::InsertResult result,
                      bool sframe_per_frame_decrypt = false)
    RTC_RUN_ON(packet_sequence_checker_);

void OnAssembledFrame(std::unique_ptr<RtpFrameObject> frame,
                      bool sframe_per_frame_decrypt = false)
    RTC_RUN_ON(packet_sequence_checker_);
```

Default `= false` keeps all existing call sites unchanged.

### Step-by-step (T=0)

1. **Raw depacketize** — for each packet in the completed S→E run, use
   `VideoRtpDepacketizerRaw` to pass the opaque ciphertext payload through
   without codec parsing.
2. **Call `OnReceivedPayloadData` with `sframe_per_frame_decrypt=true`** —
   the bool is explicitly passed as the last argument.
3. **`OnReceivedPayloadData` → `PacketBuffer::InsertPacket`** — packets
   enter `PacketBuffer` as usual.  The bool is forwarded to
   `OnInsertedPacket`.
4. **`OnInsertedPacket` → `OnAssembledFrame`** — once `PacketBuffer`
   assembles a complete frame, forward the bool to `OnAssembledFrame`.
5. **`OnAssembledFrame`** — check the bool: if `true`, decrypt the frame
   via `sframe_decrypter_->DecryptFrame()`.  After decryption the
   cleartext frame continues through the remaining pipeline
   (`frame_transformer_delegate_` → `OnCompleteFrames`).

### SFrame Intercept (T=0 path)

```cpp
// T=0: payloads are opaque ciphertext → raw depacketizer.
for (auto& pkt : result.packets) {
  auto parsed = raw_depacketizer.Parse(pkt.PayloadBuffer());
  int times_nacked = nack_module_
      ? nack_module_->OnReceivedPacket(pkt.SequenceNumber(), pkt.recovered())
      : -1;
  OnReceivedPayloadData(std::move(parsed->video_payload),
                        pkt, parsed->video_header, times_nacked,
                        /*sframe_per_frame_decrypt=*/true);
}
```

### Decryption Point (in `OnAssembledFrame`)

```cpp
if (sframe_per_frame_decrypt) {
  // Decrypt, then continue through the remaining pipeline.
  auto decrypted = sframe_decrypter_->DecryptFrame(std::move(frame));
  if (frame_transformer_delegate_) {
    frame_transformer_delegate_->TransformFrame(std::move(decrypted));
  } else {
    OnCompleteFrames(reference_finder_->ManageFrame(std::move(decrypted)));
  }
} else if (buffered_frame_decryptor_ != nullptr) {
  buffered_frame_decryptor_->ManageEncryptedFrame(std::move(frame));
} else if (frame_transformer_delegate_) {
  frame_transformer_delegate_->TransformFrame(std::move(frame));
} else {
  OnCompleteFrames(reference_finder_->ManageFrame(std::move(frame)));
}
```

---

## Option 2: `RTPVideoHeader` Tag (`sframe_encrypted` bool)

### Approach

Add a `bool sframe_encrypted = false` field to `RTPVideoHeader`.  The T=0
path sets it to `true` when using the raw depacketizer.  No signature
changes to any existing method.  The tag travels with the packet through
`OnReceivedPayloadData` → `PacketBuffer` → `OnInsertedPacket` →
`OnAssembledFrame`.  The decryption decision is made at `OnAssembledFrame`
by checking `frame->GetRtpVideoHeader().sframe_encrypted`.

### Step-by-step (T=0)

1. **Raw depacketize** — for each packet in the completed S→E run, use
   `VideoRtpDepacketizerRaw` to pass the opaque ciphertext payload through
   without codec parsing.
2. **Tag `video_header.sframe_encrypted = true`** — stamp the parsed
   `RTPVideoHeader` before it enters the pipeline.  No extra parameters.
3. **Call `OnReceivedPayloadData`** — unchanged signature.  The tag
   travels inside the `RTPVideoHeader` that is already passed.
4. **`OnReceivedPayloadData` → `PacketBuffer::InsertPacket`** — the
   `RTPVideoHeader` (with the tag) is stored alongside the packet in
   `PacketBuffer`.
5. **`OnInsertedPacket` → `OnAssembledFrame`** — once `PacketBuffer`
   assembles a complete frame, `RtpFrameObject` carries the tagged
   `RTPVideoHeader`.
6. **`OnAssembledFrame`** — check
   `frame->GetRtpVideoHeader().sframe_encrypted`: if `true`, decrypt
   the frame via `sframe_decrypter_->DecryptFrame()`.  After decryption
   the cleartext frame continues through the remaining pipeline
   (`frame_transformer_delegate_` → `OnCompleteFrames`).

### New Field

```cpp
// modules/rtp_rtcp/source/rtp_video_header.h
struct RTPVideoHeader {
  // ... existing fields ...

  // True when the payload is SFrame ciphertext (T=0 per-frame mode).
  // Tells OnAssembledFrame to route through SFrame decryption.
  bool sframe_encrypted = false;
};
```

### SFrame Intercept (T=0 path)

```cpp
// T=0: payloads are opaque ciphertext → raw depacketizer.
// Tag video_header so OnAssembledFrame knows to decrypt.
VideoRtpDepacketizerRaw raw_depacketizer;
for (auto& pkt : result.packets) {
  auto parsed = raw_depacketizer.Parse(pkt.PayloadBuffer());
  if (!parsed) continue;
  parsed->video_header.sframe_encrypted = true;
  int times_nacked = nack_module_
      ? nack_module_->OnReceivedPacket(pkt.SequenceNumber(), pkt.recovered())
      : -1;
  OnReceivedPayloadData(std::move(parsed->video_payload),
                        pkt, parsed->video_header, times_nacked);
}
```

### Decryption Point (in `OnAssembledFrame`)

```cpp
// At the existing decrypt/transform decision point:
if (sframe_decrypter_ &&
    frame->GetRtpVideoHeader().sframe_encrypted) {
  // Decrypt, then continue through the remaining pipeline.
  auto decrypted = sframe_decrypter_->DecryptFrame(std::move(frame));
  if (frame_transformer_delegate_) {
    frame_transformer_delegate_->TransformFrame(std::move(decrypted));
  } else {
    OnCompleteFrames(reference_finder_->ManageFrame(std::move(decrypted)));
  }
} else if (buffered_frame_decryptor_ != nullptr) {
  buffered_frame_decryptor_->ManageEncryptedFrame(std::move(frame));
} else if (frame_transformer_delegate_) {
  frame_transformer_delegate_->TransformFrame(std::move(frame));
} else {
  OnCompleteFrames(reference_finder_->ManageFrame(std::move(frame)));
}
```

---

## Components

### `SFrameDescriptor`

Header-only struct holding the three bits from the 1-byte SFrame payload
descriptor: S (start-of-frame), E (end-of-frame), T (per-packet vs
per-frame).

**Location:** `modules/rtp_rtcp/source/sframe_descriptor.h`

### `SframeRtpPacketReceived`

Pairs an `RtpPacketReceived` with its parsed `SFrameDescriptor`.  Used at
the `SFramePacketBuffer` boundary so the buffer can inspect S/E/T bits
without re-parsing.  Does not propagate past the SFrame intercept block.

**Location:** `modules/rtp_rtcp/source/sframe_rtp_packet_received.h`

### `VideoRtpDepacketizerSframe`

A `VideoRtpDepacketizer` implementation that:
1. Reads byte 0 of the RTP payload → `SFrameDescriptor{S, E, T}`
2. Strips byte 0 from the payload (the rest is encrypted data)
3. Returns `SframeRtpPacketReceived(packet, descriptor)`

**Location:** `modules/rtp_rtcp/source/video_rtp_depacketizer_sframe.{h,cc}`

### `SFramePacketBuffer`

A circular buffer that sits **before** `PacketBuffer` and **before** SFrame
decryption.  It collects incoming SFrame RTP packets and validates complete
S→E runs per draft-ietf-avtcore-rtp-sframe section 5.2.

**Location:** `modules/video_coding/sframe_packet_buffer.{h,cc}`

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

### `SframeDecrypter`

Handles SFrame decryption for both modes:
- **T=1 (per-packet):** decrypts individual packet payloads before they
  enter the codec depacketizer.
- **T=0 (per-frame):** decrypts the reassembled bitstream at
  `OnAssembledFrame` after `PacketBuffer` assembly.
