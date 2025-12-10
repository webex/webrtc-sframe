# Blink RTCRtpScriptTransform

This document describes how RTCRtpScriptTransform is integrated into the Blink layer and how it was extended to support both frame-level and packet-level transformations.

## Overview

`RTCRtpScriptTransform` is the existing JavaScript-accessible transform that allows custom processing of encoded frames through Web Workers. With the SFrame integration, it has been extended to support both frame-level and packet-level transformers, enabling it to work alongside `SFrameTransform` as part of the unified `RTCRtpTransform` interface.

## IDL

The RTCRtpScriptTransform interface remains unchanged in IDL, but it's now part of the RTCRtpTransform union type.

`third_party/blink/renderer/modules/peerconnection/rtc_rtp_transform.idl`
```idl
typedef (SFrameTransform or RTCRtpScriptTransform) RTCRtpTransform;
```

The original RTCRtpScriptTransform interface continues to exist with its original functionality:
```idl
[Exposed=(Window,Worker)]
interface RTCRtpScriptTransform {
    constructor(Worker worker, optional any options, optional sequence<object> transfer);
    
    readonly attribute ReadableStream readable;
    readonly attribute WritableStream writable;
};
```

## RTCRtpScriptTransform

`RTCRtpScriptTransform` is the C++ layer associated with the JavaScript transform that enables custom frame processing via Web Workers.

`third_party/blink/renderer/modules/peerconnection/rtc_rtp_script_transform.h`
```cpp
class MODULES_EXPORT RTCRtpScriptTransform : public ScriptWrappable {
  DEFINE_WRAPPERTYPEINFO();

 public:
  static RTCRtpScriptTransform* Create(
      ScriptState* script_state,
      Worker* worker,
      const ScriptValue& options,
      const HeapVector<ScriptValue>& transfer,
      ExceptionState& exception_state);

  explicit RTCRtpScriptTransform(/* constructor parameters */);
  ~RTCRtpScriptTransform() override = default;

  // Expose readable/writable for pipeThrough usage
  ReadableStream* readable() const { return readable_; }
  WritableStream* writable() const { return writable_; }

  // Called when this transform is assigned to an RTCRtpSender or
  // RTCRtpReceiver via setTransform().
  void CreateAudioUnderlyingSourceAndSink(
      CrossThreadOnceClosure disconnect_callback_source,
      scoped_refptr<blink::RTCEncodedAudioStreamTransformer::Broker>
          encoded_audio_transformer);

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

  // The Web Worker that processes frames
  Member<Worker> worker_;

  // RTP transformer handles frame processing in worker context
  std::optional<CrossThreadWeakHandle<RTCRtpScriptTransformer>> rtp_transformer_;
  scoped_refptr<base::SingleThreadTaskRunner> rtp_transformer_task_runner_;

  // Extended to support both frame and packet transformers
  scoped_refptr<blink::RTCEncodedAudioStreamTransformer::Broker>
      encoded_audio_transformer_;
  scoped_refptr<blink::RTCEncodedVideoStreamTransformer::Broker>
      encoded_video_frame_transformer_;
  scoped_refptr<blink::RTCEncodedVideoStreamTransformer::Broker>
      encoded_video_packet_transformer_;
  
  CrossThreadOnceClosure disconnect_callback_source_;
  
  bool is_attached_ = false;
  bool is_unused_ = true;
};
```

The key changes from the original implementation:

1. **Dual Transformer Support**: `CreateVideoUnderlyingSourceAndSink` now accepts both frame-level and packet-level transformer brokers
2. **Extended State Management**: Tracks both frame and packet transformers separately
3. **Unified Interface**: Works as part of the RTCRtpTransform union alongside SFrameTransform

## Key Methods Implementation

### CreateVideoUnderlyingSourceAndSink

The method was extended to handle both frame-level and packet-level transformers:

```cpp
void RTCRtpScriptTransform::CreateVideoUnderlyingSourceAndSink(
    CrossThreadOnceClosure disconnect_callback_source,
    scoped_refptr<blink::RTCEncodedVideoStreamTransformer::Broker>
        encoded_video_frame_transformer,
    scoped_refptr<blink::RTCEncodedVideoStreamTransformer::Broker>
        encoded_video_packet_transformer) {
  DCHECK_CALLED_ON_VALID_SEQUENCE(sequence_checker_);
  
  if (rtp_transformer_) {
    // If transformer is available, set up immediately
    SetUpVideoRtpTransformer(std::move(disconnect_callback_source),
                             std::move(encoded_video_frame_transformer),
                             std::move(encoded_video_packet_transformer));
  } else {
    // Store for later setup when transformer becomes available
    encoded_video_frame_transformer_ = std::move(encoded_video_frame_transformer);
    encoded_video_packet_transformer_ = std::move(encoded_video_packet_transformer);
    disconnect_callback_source_ = std::move(disconnect_callback_source);
  }
}
```

### SetUpVideoRtpTransformer

Updated to handle dual transformer setup:

```cpp
void RTCRtpScriptTransform::SetUpVideoRtpTransformer(
    CrossThreadOnceClosure disconnect_callback_source,
    scoped_refptr<blink::RTCEncodedVideoStreamTransformer::Broker>
        encoded_video_frame_transformer,
    scoped_refptr<blink::RTCEncodedVideoStreamTransformer::Broker>
        encoded_video_packet_transformer) {
  DCHECK_CALLED_ON_VALID_SEQUENCE(sequence_checker_);
  CHECK(rtp_transformer_);
  
  PostCrossThreadTask(
      *rtp_transformer_task_runner_, FROM_HERE,
      CrossThreadBindOnce(
          &RTCRtpScriptTransformer::SetUpVideo,
          MakeUnwrappingCrossThreadWeakHandle(*rtp_transformer_),
          std::move(disconnect_callback_source),
          std::move(encoded_video_frame_transformer),
          std::move(encoded_video_packet_transformer)));
}
```

## RTCRtpScriptTransformer

`RTCRtpScriptTransformer` runs in the Web Worker context and handles the actual frame processing. It was also extended to support dual transformers.

```cpp
class MODULES_EXPORT RTCRtpScriptTransformer : public ScriptWrappable {
 public:
  RTCRtpScriptTransformer(ScriptState* script_state, 
                          CrossThreadWeakHandle<RTCRtpScriptTransform> transform);
  ~RTCRtpScriptTransformer() override = default;

  // Exposes readable/writable streams to the Worker
  ReadableStream* readable(ScriptState* script_state);
  WritableStream* writable(ScriptState* script_state);

  void SetUpAudio(CrossThreadOnceClosure disconnect_callback_source,
                  scoped_refptr<blink::RTCEncodedAudioStreamTransformer::Broker>
                      encoded_audio_transformer);

  void SetUpVideo(CrossThreadOnceClosure disconnect_callback_source,
                  scoped_refptr<blink::RTCEncodedVideoStreamTransformer::Broker>
                      encoded_video_frame_transformer,
                  scoped_refptr<blink::RTCEncodedVideoStreamTransformer::Broker>
                      encoded_video_packet_transformer);

  void Clear();

  // Extended functionality for transformation features
  scoped_refptr<blink::RTCEncodedVideoStreamTransformer::Broker> GetVideoBroker();
  void SetVideoTransformationFeatures(
      const Vector<webrtc::TransformationFeature>& features);

  void Trace(Visitor* visitor) const override;

 private:
  const Member<ScriptState> script_state_;
  Member<RTCEncodedUnderlyingSourceWrapper> rtc_encoded_underlying_source_;
  Member<RTCEncodedUnderlyingSinkWrapper> rtc_encoded_underlying_sink_;
  
  // Cached streams for worker access
  Member<ReadableStream> readable_;
  Member<WritableStream> writable_;
};
```

### SetUpVideo Implementation

```cpp
void RTCRtpScriptTransformer::SetUpVideo(
    CrossThreadOnceClosure disconnect_callback_source,
    scoped_refptr<blink::RTCEncodedVideoStreamTransformer::Broker>
        encoded_video_frame_transformer,
    scoped_refptr<blink::RTCEncodedVideoStreamTransformer::Broker>
        encoded_video_packet_transformer) {
  DCHECK_CALLED_ON_VALID_SEQUENCE(sequence_checker_);
  base::UnguessableToken owner_id = base::UnguessableToken::Create();

  // For now, default to frame-level transformer
  // Custom logic could be added here to choose based on worker configuration
  scoped_refptr<blink::RTCEncodedVideoStreamTransformer::Broker> selected_transformer =
      std::move(encoded_video_frame_transformer);

  rtc_encoded_underlying_source_->CreateVideoUnderlyingSource(
      std::move(disconnect_callback_source), owner_id);
  selected_transformer->SetTransformerCallback(
      rtc_encoded_underlying_source_->GetVideoTransformer());
  selected_transformer->SetSourceTaskRunner(rtp_transformer_task_runner_);
  rtc_encoded_underlying_sink_->CreateVideoUnderlyingSink(
      std::move(selected_transformer), owner_id);
}
```

## Integration with RTCRtpSender/Receiver

The RTCRtpScriptTransform integrates with the updated RTCRtpSender/Receiver interfaces that now handle the RTCRtpTransform union type:

```cpp
// In RTCRtpSender::setTransform
if (transform->IsRTCRtpScriptTransform()) {
    RTCRtpScriptTransform* script_transform = 
        transform->GetAsRTCRtpScriptTransform();
    
    script_transform->Attach();
    
    if (kind_ == "video") {
        script_transform->CreateVideoUnderlyingSourceAndSink(
            CrossThreadBindOnce(&RTCRtpSender::UnregisterEncodedVideoStreamCallback,
                                WrapCrossThreadWeakPersistent(this)),
            encoded_video_transformer_,
            encoded_video_packet_transformer_);
    }
    // Similar for audio...
}
```