# Blink RTCRtpSender

## Overview

This document describes how to integrate SFrame (Secure Frame) encryption into the Blink RTCRtpSender.

## RTCRtpSenderImpl::RTCRtpSenderInternal

`RTCRtpSenderImpl::RTCRtpSenderInternal` registers both - `Frame` and `Packet` level transformers.

```cpp
class RTCRtpSenderImpl::RTCRtpSenderInternal
    : public ThreadSafeRefCounted<
          RTCRtpSenderImpl::RTCRtpSenderInternal,
          RTCRtpSenderImpl::RTCRtpSenderInternalTraits> {
 public:
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
      encoded_audio_frame_transformer_ =
          std::make_unique<RTCEncodedAudioStreamTransformer>(main_task_runner_);
      webrtc_sender_->SetFrameTransformer(
          encoded_audio_frame_transformer_->Delegate());

      encoded_audio_packet_transformer_ =
          std::make_unique<RTCEncodedAudioStreamTransformer>(main_task_runner_);
      webrtc_sender_->SetPacketTransformer(
        encoded_audio_packet_transformer_->Delegate());
    } else {
      CHECK(webrtc_sender_->media_type() == webrtc::MediaType::VIDEO);
      encoded_video_frame_transformer_ =
          std::make_unique<RTCEncodedVideoStreamTransformer>(
              main_task_runner_, /*metronome=*/nullptr);
      webrtc_sender_->SetFrameTransformer(
          encoded_video_frame_transformer_->Delegate());

      encoded_video_packet_transformer_ =
          std::make_unique<RTCEncodedVideoStreamTransformer>(
              main_task_runner_, /*metronome=*/nullptr);
      webrtc_sender_->SetFrameTransformer(
          encoded_video_packet_transformer_->Delegate());
    }
  }

  RTCEncodedAudioStreamTransformer* GetEncodedAudioFrameStreamTransformer() const {
    return encoded_audio_frame_transformer_.get();
  }

  RTCEncodedVideoStreamTransformer* GetEncodedVideoFrameStreamTransformer() const {
    return encoded_video_frame_transformer_.get();
  }

  RTCEncodedAudioStreamTransformer* GetEncodedAudioPacketStreamTransformer() const {
    return encoded_audio_packet_transformer_.get();
  }

  RTCEncodedVideoStreamTransformer* GetEncodedVideoPacketStreamTransformer() const {
    return encoded_video_packet_transformer_.get();
  }

 private:
  // Frame Transformers
  std::unique_ptr<RTCEncodedAudioStreamTransformer> encoded_audio_frame_transformer_;
  std::unique_ptr<RTCEncodedVideoStreamTransformer> encoded_video_frame_transformer_;
  // Packet Transformers
  std::unique_ptr<RTCEncodedAudioStreamTransformer> encoded_audio_packet_transformer_;
  std::unique_ptr<RTCEncodedVideoStreamTransformer> encoded_video_packet_transformer_;
}
```

## RTCRtpSender

This class provides direct binding between Javascript and C++ layer for senders.
It will store now brokers associated with both `Frame` and `Packet` level transformations.

```cpp
class MODULES_EXPORT RTCRtpSenderImpl : public blink::RTCRtpSenderPlatform {
 public:
  V8UnionRTCRtpScriptTransformOrSFrameTransform* transform() { return transform_.Get(); }
  void setTransform(V8UnionRTCRtpScriptTransformOrSFrameTransform*, ExceptionState& exception_state);
 private:
  Member<V8UnionRTCRtpScriptTransformOrSFrameTransform> transform_;

 const scoped_refptr<blink::RTCEncodedAudioStreamTransformer::Broker>
      encoded_audio_frame_transformer_;
 const scoped_refptr<blink::RTCEncodedVideoStreamTransformer::Broker>
      encoded_video_frame_transformer_;

 // Packet-level transformer brokers
 const scoped_refptr<blink::RTCEncodedAudioStreamTransformer::Broker>
      encoded_audio_packet_transformer_;
 const scoped_refptr<blink::RTCEncodedVideoStreamTransformer::Broker>
      encoded_video_packet_transformer_;
}
```

`RTCRtpSender::RTCRtpSender` will initialized in the constructor `Broker`s provided from the `RTCRtpSenderImpl::RTCRtpSenderInternal`.

```cpp
RTCRtpSender::RTCRtpSender(RTCPeerConnection* pc,
                           std::unique_ptr<RTCRtpSenderPlatform> sender,
                           String kind,
                           MediaStreamTrack* track,
                           MediaStreamVector streams,
                           bool require_encoded_insertable_streams,
                           scoped_refptr<base::SequencedTaskRunner>
                               encoded_transform_shortcircuit_runner)
    : ExecutionContextLifecycleObserver(pc->GetExecutionContext()),
      pc_(pc),
      sender_(std::move(sender)),
      kind_(std::move(kind)),
      track_(track),
      streams_(std::move(streams)),
      encoded_audio_frame_transformer_(
          kind_ == "audio"
              ? sender_->GetEncodedAudioStreamTransformer()->GetBroker()
              : nullptr),
      encoded_video_frame_transformer_(
          kind_ == "video"
              ? sender_->GetEncodedVideoStreamTransformer()->GetBroker()
              : nullptr),
      encoded_audio_packet_transformer_(
          kind_ == "audio"
              ? sender_->GetEncodedAudioPacketTransformer()->GetBroker()
              : nullptr),
      encoded_video_packet_transformer_(
          kind_ == "video"
              ? sender_->GetEncodedVideoPacketTransformer()->GetBroker()
               : nullptr) {
  // Current implementation
}
```

After the updates to the `RTCRtpSender::setTransform`, it will be able to distinguish which type of `transform` have been provided.

```cpp
void RTCRtpSender::setTransform(
    V8UnionRTCRtpScriptTransformOrSFrameTransform* transform,
    ExceptionState& exception_state) {
  DCHECK_CALLED_ON_VALID_THREAD(thread_checker_);
  if (transform_ == transform) {
    return;
  }
  
  if (!transform) {
    transform_->Detach();
    transform_ = nullptr;
    return;
  }

  transform_ = transform;

  if (transform->IsRTCRtpScriptTransform()) {
    setUpRTCRtpScriptTransform(transform_->GetAsRTCRtpScriptTransform(), exception_state);
  }

  if (transform->IsSFrameTransform()) {
    setUpSFrameTransform(transform_->GetAsSFrameTransform(), exception_state);
  }
}
```

To the `SFrameTransform` and `RTCRtpScriptTransform` both transformers will be provided, so that it could setup a pipeline depending on the request.

For now `setUpSFrameTransform` and `setUpRTCRtpScriptTransform` probably will look like the same. Leaving it as a separate methods for this draft purpose.

```cpp
void RTCRtpSender::setUpSFrameTransform(
    SFrameTransform* transform,
    ExceptionState& exception_state) {
  if (kind_ == "audio") {
    transform->CreateAudioUnderlyingSourceAndSink(
      CrossThreadBindOnce(&RTCRtpSender::UnregisterEncodedAudioStreamCallback,
                          WrapCrossThreadWeakPersistent(this)),
          encoded_audio_transformer_,
          encoded_audio_packet_transformer_);
      return;
  } else {
    transform->CreateVideoUnderlyingSourceAndSink(
      CrossThreadBindOnce(&RTCRtpSender::UnregisterEncodedVideoStreamCallback,
                          WrapCrossThreadWeakPersistent(this)),
      encoded_video_frame_transformer_,
      encoded_video_packet_transformer_);
  }
}

void RTCRtpSender::setUpRTCRtpScriptTransform(
    RTCRtpScriptTransform* transform,
    ExceptionState& exception_state) {
  if (kind_ == "audio") {
    transform->CreateAudioUnderlyingSourceAndSink(
      CrossThreadBindOnce(&RTCRtpSender::UnregisterEncodedAudioStreamCallback,
                          WrapCrossThreadWeakPersistent(this)),
          encoded_audio_transformer_,
          encoded_audio_packet_transformer_);
      return;
  } else {
    transform->CreateVideoUnderlyingSourceAndSink(
      CrossThreadBindOnce(&RTCRtpSender::UnregisterEncodedVideoStreamCallback,
                          WrapCrossThreadWeakPersistent(this)),
      encoded_video_frame_transformer_,
      encoded_video_packet_transformer_);
  }
}
```
