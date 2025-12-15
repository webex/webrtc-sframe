# SFrame Negotiation Trigger Proposals

This document contains various proposals for how SFrame transformers can trigger SDP negotiation and modify SDP attributes.

## Overview

When implementing SFrame encryption through frame transformers, there's a need for transformers to:
1. Signal that specific SDP attributes should be added (e.g., `a=sframe`)
2. Trigger SDP renegotiation when transformers are attached
3. Control which packetizer/depacketizer is used based on the encryption mode

### Comparison of Approaches

| Aspect | Proposal 1: Features Parameter | Proposal 2: Observer Pattern |
|--------|-------------------------------|------------------------------|
| **Pattern** | Parameter-passing (pull) | Observer/callback (push) |
| **Control** | Application explicitly specifies features | Transformer requests features via callback |
| **Complexity** | Simple transformer implementation | Transformers need negotiation logic |
| **Flexibility** | Features can be changed independently | Transformers control their own requirements |
| **Testability** | Easy to test with explicit parameters | Requires observer mock/stub setup |
| **Timing** | Features specified at attachment time | Features can be requested dynamically |
| **Thread Safety** | Clear ownership model | Requires careful callback threading |
| **API Surface** | Extends SetFrameTransformer signature | Adds new observer interface |
| **Discovery** | Application must know required features | Transformers declare their needs |

## Proposal 1: Features Parameter Approach

### Motivation

This approach uses a **parameter-passing pattern** where the application explicitly specifies which SDP features should be associated with a transformer when attaching it to a sender/receiver. This keeps the transformer implementation simple and gives the application explicit control over feature negotiation.

### API Design

#### SdpFeature Enum

Define an enumeration of SDP features that can be requested:

```cpp
enum class SdpFeature {
  // Secure Frame (SFrame) encryption
  // Adds "a=sframe" to the SDP
  kSFrame,
};
```

#### FrameTransformerHost Interface

The base interface that RTP senders/receivers implement to accept transformers with features:

```cpp
class FrameTransformerHost {
 public:
  virtual ~FrameTransformerHost() {}
  
  virtual void SetFrameTransformer(
      scoped_refptr<FrameTransformerInterface> frame_transformer,
      const std::vector<SdpFeature>& features = {}) = 0;
      
  virtual void SetPacketTransformer(
      scoped_refptr<FrameTransformerInterface> packet_transformer,
      const std::vector<SdpFeature>& features = {}) = 0;
};
```

#### RtpSenderInterface / RtpReceiverInterface

Extended to accept features parameter:

```cpp
class RtpSenderInterface : public FrameTransformerHost {
 public:
  void SetFrameTransformer(
      scoped_refptr<FrameTransformerInterface> frame_transformer,
      const std::vector<SdpFeature>& features = {}) override;
      
  void SetPacketTransformer(
      scoped_refptr<FrameTransformerInterface> packet_transformer,
      const std::vector<SdpFeature>& features = {}) override;
};
```

### Implementation

#### RtpSenderBase

Stores features and triggers negotiation when they change:

```cpp
class RtpSenderBase : public RtpSenderInternal {
 public:
  void SetFrameTransformer(
      scoped_refptr<FrameTransformerInterface> frame_transformer,
      const std::vector<SdpFeature>& features) override {
    RTC_DCHECK_RUN_ON(signaling_thread_);
    
    // Check if features have changed
    bool features_changed = (frame_transformer_features_ != features);
    
    frame_transformer_ = std::move(frame_transformer);
    frame_transformer_features_ = features;
    
    // Set transformer on worker thread
    if (media_channel_ && ssrc_ && !stopped_) {
      worker_thread_->BlockingCall([&] {
        media_channel_->SetEncoderToPacketizerFrameTransformer(
            ssrc_, frame_transformer_);
      });
    }

    // Trigger negotiation if features changed
    if (features_changed && on_negotiation_needed_) {
      on_negotiation_needed_();
    }
  }

 private:
  std::vector<SdpFeature> frame_transformer_features_;
  std::vector<SdpFeature> packet_transformer_features_;
  OnNegotiationNeededCallback on_negotiation_needed_;
};
```

#### VideoRtpReceiver

Similar implementation for receivers:

```cpp
class VideoRtpReceiver : public RtpReceiverInternal {
 public:
  void SetFrameTransformer(
      scoped_refptr<FrameTransformerInterface> frame_transformer,
      const std::vector<SdpFeature>& features) override {
    RTC_DCHECK_RUN_ON(&signaling_thread_checker_);
    
    // Store features on signaling thread
    frame_transformer_features_ = features;
    
    // Set transformer on worker thread
    worker_thread_->BlockingCall([this, frame_transformer = std::move(frame_transformer)]() mutable {
      RTC_DCHECK_RUN_ON(worker_thread_);
      frame_transformer_ = std::move(frame_transformer);
      
      if (media_channel_) {
        media_channel_->SetDepacketizerToDecoderFrameTransformer(
            signaled_ssrc_.value_or(0), frame_transformer_);
      }
    });

    // On neegotiation needed trigger
  }

 private:
  std::vector<SdpFeature> frame_transformer_features_
      RTC_GUARDED_BY(signaling_thread_checker_);
};
```

#### Blink

##### RTCRtpSenderInternal

Currently:

```cpp
RTCRtpSenderInternal(
      webrtc::scoped_refptr<webrtc::PeerConnectionInterface>
          native_peer_connection,
      scoped_refptr<blink::WebRtcMediaStreamTrackAdapterMap> track_map,
      RtpSenderState state)
      : native_peer_connection_(std::move(native_peer_connection)),
        track_map_(std::move(track_map)),
        main_task_runner_(state.main_task_runner()),
        signaling_task_runner_(state.signaling_task_runner()),
        webrtc_sender_(state.webrtc_sender()),
        state_(std::move(state)) {
    DCHECK(track_map_);
    DCHECK(state_.is_initialized());

    if (webrtc_sender_->media_type() == webrtc::MediaType::AUDIO) {
      // Frame-level transformer
      encoded_audio_transformer_ =
          std::make_unique<RTCEncodedAudioStreamTransformer>(main_task_runner_);
      webrtc_sender_->SetFrameTransformer(
          encoded_audio_transformer_->Delegate());
      
      // Packet-level transformer
      encoded_audio_packet_transformer_ =
          std::make_unique<RTCEncodedAudioStreamTransformer>(main_task_runner_);
      webrtc_sender_->SetPacketTransformer(
          encoded_audio_packet_transformer_->Delegate(), {});
    } else {
      CHECK(webrtc_sender_->media_type() == webrtc::MediaType::VIDEO);
      // Frame-level transformer
      encoded_video_transformer_ =
          std::make_unique<RTCEncodedVideoStreamTransformer>(
              main_task_runner_, /*metronome=*/nullptr);
      webrtc_sender_->SetFrameTransformer(
          encoded_video_transformer_->Delegate());
      
      // Packet-level transformer
      encoded_video_packet_transformer_ =
          std::make_unique<RTCEncodedVideoStreamTransformer>(
              main_task_runner_, /*metronome=*/nullptr);
      webrtc_sender_->SetPacketTransformer(
          encoded_video_packet_transformer_->Delegate(), {});
    }
  }
```

With new recreate method:

```cpp
void RTCRtpSenderInternal::MaybeRecreateFrameTransformers() {
  if (encoded_video_transformer_) {
    // Recreate frame transformer
    auto features = encoded_video_transformer_->GetTransformationFeatures();

    webrtc_sender_->SetFrameTransformer(
      encoded_video_transformer_->Delegate(), features);
  }
    
  // Recreate packet transformer
  if (encoded_video_packet_transformer_) {
    auto features = encoded_video_packet_transformer_->GetTransformationFeatures();

    webrtc_sender_->SetPacketTransformer(
      encoded_video_packet_transformer_->Delegate(), features);
  }
}
```

##### RTCRtpSender

```cpp
void RTCRtpSender::setTransform(
    V8UnionRTCRtpScriptTransformOrSFrameTransform* transform,
    ExceptionState& exception_state) {
  /// Perform creation of transformers

  if (sender_) {
    sender_->MaybeRecreateFrameTransformers();
  }
}
```