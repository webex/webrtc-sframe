# Blink SFrameTransform

This document describes how SFrameTransform will be integrated into Blink layer.

## IDL

The IDL definition specifies how `SFrameTransform` is exposed to JavaScript and defines the interface between the JavaScript layer and the C++ Blink implementation. This includes the SFrame operating modes and configuration options.

`third_party/blink/renderer/modules/peerconnection/sframe_transform.idl`
```idl
enum SFrameMode {
  "per-frame",
  "per-packet",
};

dictionary SFrameTransformOptions {
    SFrameMode mode;
};

[Exposed=(Window,Worker)]
interface SFrameTransform {
    [CallWith=ScriptState, RaisesException] constructor(optional SFrameTransformOptions options = {});
    
    readonly attribute ReadableStream readable;
    readonly attribute WritableStream writable;
};
```

## SFrameTransform

`SFrameTransform` is the main C++ implementation class that bridges the JavaScript API with the underlying WebRTC transformation infrastructure. It manages the lifecycle of SFrame encryption/decryption operations and handles both stream-based (pipeThrough) and direct assignment usage patterns.

`third_party/blink/renderer/modules/peerconnection/s_frame_transform.h`
```cpp
class MODULES_EXPORT SFrameTransform final : public ScriptWrappable {
  DEFINE_WRAPPERTYPEINFO();

 public:
  static SFrameTransform* Create(ScriptState* script_state,
                                 const SFrameTransformOptions* options,
                                 ExceptionState& exception_state);

  explicit SFrameTransform(ScriptState* script_state,
                           const SFrameTransformOptions* options,
                           ExceptionState& exception_state);
  ~SFrameTransform() override = default;

  // Expose readable/writable for pipeThrough usage
  ReadableStream* readable() const { return readable_; }
  WritableStream* writable() const { return writable_; }

  // Called when this transform is assigned to an RTCRtpSender or
  // RTCRtpReceiver via setTransform().
  void CreateAudioUnderlyingSourceAndSink(
      CrossThreadOnceClosure disconnect_callback_source,
      scoped_refptr<blink::RTCEncodedAudioStreamTransformer::Broker>
          encoded_audio_frame_transformer,
      scoped_refptr<blink::RTCEncodedAudioStreamTransformer::Broker>
          encoded_audio_packet_transformer);

  void CreateVideoUnderlyingSourceAndSink(
      CrossThreadOnceClosure disconnect_callback_source,
      scoped_refptr<blink::RTCEncodedVideoStreamTransformer::Broker>
          encoded_video_frame_transformer,
      scoped_refptr<blink::RTCEncodedVideoStreamTransformer::Broker>
          encoded_video_packet_transformer);

  void Attach();
  void Detach();
  bool HasBeenUsed() const;

  void Trace(Visitor* visitor) const override;

 private:
  // Streams exposed for pipeThrough usage
  Member<ReadableStream> readable_;
  Member<WritableStream> writable_;

  Member<SFrameTransformer> transformer_;
  
  // Configuration options
  Member<SFrameTransformOptions> options_;
  
  bool is_attached_ = false;
  bool is_unused_ = true;
};
```

It's most important functions are `CreateAudioUnderlyingSourceAndSink` and `CreateVideoUnderlyingSourceAndSink` as it will be used to create a underlying SFrame transformer which will perform SFrame encryption/decryption on one of the provided `Frame`/`Packet` transformers passthough.

```cpp
void SFrameTransform::CreateVideoUnderlyingSourceAndSink(
    CrossThreadOnceClosure disconnect_callback_source,
    scoped_refptr<blink::RTCEncodedVideoStreamTransformer::Broker>
        encoded_video_frame_transformer,
    scoped_refptr<blink::RTCEncodedVideoStreamTransformer::Broker>
        encoded_video_packet_transformer) {
  if (!video_transformer_) {
    video_transformer_ = MakeGarbageCollected<SFrameTransformer>();
  }
  
  // Select transformer based on mode
  scoped_refptr<blink::RTCEncodedVideoStreamTransformer::Broker> selected_transformer;
  if (options_ && options_->hasMode() && options_->mode() == V8SFrameMode::Enum::kPerPacket) {
    selected_transformer = std::move(encoded_video_packet_transformer);
  } else {
    selected_transformer = std::move(encoded_video_frame_transformer);
  }
  
  video_transformer_->SetUpVideo(std::move(disconnect_callback_source),
                                  std::move(selected_transformer));
}
```

## SFrameTransformer

`SFrameTransformer` acts as the bridge between Blink's encoded stream transformers and the actual SFrame encryption/decryption logic. It receives encoded frames from the WebRTC pipeline, applies SFrame transformations, and sends the processed frames back to the pipeline. This component manages:

- **Frame callback registration** with WebRTC transformer brokers
- **Cross-thread communication** for safe frame processing
- **SFrame encryption/decryption operations** on audio and video frames
- **Error handling and cleanup** when transforms are detached

```cpp
class MODULES_EXPORT SFrameTransformer final
    : public GarbageCollected<SFrameTransformer> {
 public:
  SFrameTransformer();
  ~SFrameTransformer() = default;

  // Set up for audio transformation
  void SetUpAudio(
      CrossThreadOnceClosure disconnect_callback_source,
      scoped_refptr<blink::RTCEncodedAudioStreamTransformer::Broker>
          encoded_audio_transformer);

  // Set up for video transformation
  void SetUpVideo(
      CrossThreadOnceClosure disconnect_callback_source,
      scoped_refptr<blink::RTCEncodedVideoStreamTransformer::Broker>
          encoded_video_transformer);

  // Clear callbacks
  void Clear();

  void Trace(Visitor* visitor) const {}

 private:
  // Audio frame transformation (dummy encryption/decryption)
  void TransformAudioFrame(
      std::unique_ptr<webrtc::TransformableAudioFrameInterface> frame);

  // Video frame transformation (dummy encryption/decryption)
  void TransformVideoFrame(
      std::unique_ptr<webrtc::TransformableVideoFrameInterface> frame);

  // Post the transformed frame back to the sink
  void SendAudioFrameToSink(
      std::unique_ptr<webrtc::TransformableAudioFrameInterface> frame);
  
  void SendVideoFrameToSink(
      std::unique_ptr<webrtc::TransformableVideoFrameInterface> frame);

  scoped_refptr<base::SingleThreadTaskRunner> task_runner_;
  scoped_refptr<blink::RTCEncodedAudioStreamTransformer::Broker>
      audio_broker_;
  scoped_refptr<blink::RTCEncodedVideoStreamTransformer::Broker>
      video_broker_;
};
```

Draft of the methods (To be implemented).

```cpp
SFrameTransformer::SFrameTransformer()
    : task_runner_(base::SingleThreadTaskRunner::GetCurrentDefault()) {}

void SFrameTransformer::SetUpAudio(
    CrossThreadOnceClosure disconnect_callback_source,
    scoped_refptr<blink::RTCEncodedAudioStreamTransformer::Broker>
        encoded_audio_transformer) {
  audio_broker_ = std::move(encoded_audio_transformer);

  // Set up the transformer callback to receive frames from WebRTC
  audio_broker_->SetTransformerCallback(
      CrossThreadBindRepeating(&SFrameTransformer::TransformAudioFrame,
                               WrapCrossThreadPersistent(this)));
}

void SFrameTransformer::SetUpVideo(
    CrossThreadOnceClosure disconnect_callback_source,
    scoped_refptr<blink::RTCEncodedVideoStreamTransformer::Broker>
        encoded_video_transformer) {
  video_broker_ = std::move(encoded_video_transformer);

  // Set up the transformer callback to receive frames from WebRTC
  video_broker_->SetTransformerCallback(
      CrossThreadBindRepeating(&SFrameTransformer::TransformVideoFrame,
                               WrapCrossThreadPersistent(this)));
}

void SFrameTransformer::Clear() {
  if (audio_broker_) {
    audio_broker_->ResetTransformerCallback();
    audio_broker_ = nullptr;
  }
  if (video_broker_) {
    video_broker_->ResetTransformerCallback();
    video_broker_ = nullptr;
  }
}

void SFrameTransformer::TransformAudioFrame(
    std::unique_ptr<webrtc::TransformableAudioFrameInterface> frame) {
  if (!frame || !audio_broker_) {
    return;
  }

  // Push frame to libwebrtc Transformer
}

void SFrameTransformer::TransformVideoFrame(
    std::unique_ptr<webrtc::TransformableVideoFrameInterface> frame) {
  if (!frame || !video_broker_) {
    return;
  }

  // Push frame to libwebrtc Transformer
}

void SFrameTransformer::SendAudioFrameToSink(
    std::unique_ptr<webrtc::TransformableAudioFrameInterface> frame) {
  if (audio_broker_) {
    audio_broker_->SendFrameToSink(std::move(frame));
  }
}

void SFrameTransformer::SendVideoFrameToSink(
    std::unique_ptr<webrtc::TransformableVideoFrameInterface> frame) {
  if (video_broker_) {
    video_broker_->SendFrameToSink(std::move(frame));
  }
}
```
