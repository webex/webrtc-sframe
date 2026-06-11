# Video Receive Pipeline — SFrame Integration Plan

## Table of Contents

- [Overview](#overview)
- [Components at a Glance](#components-at-a-glance)
- [SFrame Intercept in `ReceivePacket`](#sframe-intercept-in-receivepacket)
- [T=0 Decryption Signal](#t0-decryption-signal)
- [Component Details](#component-details)
  - [`SFrameDescriptor`](#sframedescriptor)
  - [`SframeRtpPacketReceived`](#sframertppacketreceived)
  - [`RtpDepacketizerSframe`](#rtpdepacketizersframe)
  - [`SFramePacketBuffer`](#sframepacketbuffer)
  - [`SframeDecrypter`](#sframedecrypter)

## Overview

This document describes the changes needed to add the SFrame receive
pipeline to `RtpVideoStreamReceiver2`, handling both T=0 (per-frame) and
T=1 (per-packet) modes.

The core idea is a small intercept in front of the existing receive path:
an SFrame-aware depacketizer parses the 1-byte payload descriptor, a
dedicated packet buffer reassembles `S→E` runs, and decryption is then
dispatched per-packet (T=1) or per-frame (T=0).  Everything downstream of
that intercept is unchanged.

---

## Components at a Glance

| Component | Purpose |
|---|---|
| `SFrameDescriptor` | S/E/T bit struct (header-only) |
| `SframeRtpPacketReceived` | `RtpPacketReceived` + parsed descriptor |
| `SFramePacketBuffer` | Circular buffer, validates S→E runs |
| `RtpDepacketizerSframe` | Depacketizer: parses + strips 1-byte descriptor from RTP payload |
| SFrame decrypter | Actual SFrame decryption (per-packet for T=1, per-frame for T=0) |

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
     `OnAssembledFrame` after assembly (see [T=0 Decryption
     Signal](#t0-decryption-signal) below for how the signal reaches
     that point).

`SframeRtpPacketReceived` is scoped to the intercept block: it carries
the parsed S/E/T descriptor into `SFramePacketBuffer` and is unwrapped
back to `RtpPacketReceived` before flowing downstream.

---

## T=0 Decryption Signal

The T=1 path needs no signal — it decrypts before the codec depacketizer
and from there is identical to the non-SFrame flow.  The T=0 path is the
problem: by the time the bitstream reaches `OnAssembledFrame` it looks
like a normal reassembled frame, and we need a way to tell that method
“this one is ciphertext, decrypt it before releasing it downstream.”

We thread a `bool sframe_per_frame_decrypt = false` parameter through
`OnReceivedPayloadData` → `OnInsertedPacket` → `OnAssembledFrame`.
The T=0 path passes `true`; everywhere else the default `false` keeps
existing call sites unchanged.  Decryption happens at `OnAssembledFrame`
when the flag is set.

### Signature changes

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

### SFrame intercept (T=0 path)

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

### Decryption point (in `OnAssembledFrame`)

The `else if` chain is the existing decrypt/transform decision — we only
add the new branch at the top.

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

## Component Details

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

### `RtpDepacketizerSframe`

Codec-agnostic depacketizer for the SFrame payload descriptor:
1. Reads byte 0 of the RTP payload → `SFrameDescriptor{S, E, T}`
2. Strips byte 0 from the payload (the rest is encrypted data)
3. Returns `SframeRtpPacketReceived(packet, descriptor)`

It does not implement `VideoRtpDepacketizer` — the descriptor format is
independent of the wrapped media, so the same class is reused for any
encrypted RTP payload.

**Location:** `modules/rtp_rtcp/source/rtp_depacketizer_sframe.{h,cc}`

### `SFramePacketBuffer`

A circular buffer that sits **before** `PacketBuffer` and **before** SFrame
decryption.  It collects incoming SFrame RTP packets and validates complete
S→E runs per draft-ietf-avtcore-rtp-sframe section 5.2.

**Location:** `modules/video_coding/sframe_packet_buffer.{h,cc}`

#### Internal design

**Storage.** Circular buffer indexed by `seq_num % buffer_size`, same
pattern as the existing `PacketBuffer`.  Starts at 96 slots and doubles
up to 2048.

**`InsertPacket` flow:**

1. `UpdateWindowStart(seq_num)` — reject packets older than the window.
2. `ResolveSlot(seq_num)` — find a buffer slot.  Returns one of:
   - `kOk` — empty slot found (possibly after expansion).
   - `kDuplicate` — packet already buffered; return an empty result.
   - `kFull` — buffer at max capacity; `Clear()` and set `buffer_cleared`.
3. Store packet in the slot.
4. `FindFrame(seq_num)` — check if insertion completes a frame:
   `FindFrameStart` walks backward looking for `S=1`, `FindFrameEnd`
   walks forward looking for `E=1`, and `AssembleFrame` validates the
   run (see below) and moves packets out of the buffer into the result.
   On validation failure the frame is dropped and the slots are cleared.

#### Validation rules (in `AssembleFrame`)

All packets in an `S→E` run must share the same:

- **T-bit** — mixed per-packet/per-frame is invalid.
- **Payload type** — all packets belong to the same codec.
- **RTP timestamp** — all packets belong to the same frame.

If any mismatch is found, the entire run is dropped and the slots are
cleared.

#### Window management

`UpdateWindowStart` tracks the lowest seq num.  Reordered packets that
still fit in the buffer extend the window backward.  `ClearTo(seq_num)`
discards all packets up to and including `seq_num` and locks the window
start at that point — late packets before the cleared point are
rejected.

### `SframeDecrypter`

Handles SFrame decryption for both modes:
- **T=1 (per-packet):** decrypts individual packet payloads before they
  enter the codec depacketizer.
- **T=0 (per-frame):** decrypts the reassembled bitstream at
  `OnAssembledFrame` after `PacketBuffer` assembly.
