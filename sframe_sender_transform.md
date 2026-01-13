# SFrameSenderTransform

## Definition

`SFrameSenderTransform` will be a class dedicated for performing an SFrame on the sender.
It's role will be to create appropriate transformer and configure sender for provided transformation type.

It's role will be to:
* Create appropriate `transformer` and inject it to the transformer slot
* Expose a API necessary to perform adding keys operations

It won't tell transceiver to enable SFrame, user have to do it manually with `SetUseSFrame` call.

`modules/sframe/sframe_sender_transform.h`
```cpp
class SFrameSenderTransform : public SFrameEncrypterManager {
  SFrameSenderTransform(const SFrameTransformOptions& options,
                        RtpSenderInterface* host,
                        Thread* worker_thread);
  ~SFrameSenderTransform() override;

  bool SetEncryptionKey(const std::string& key, CryptoKeyID key_id) override;

private:
  const SFrameTransformOptions options_;
  RtpSenderInterface* host_;
  scoped_refptr<SFrameEncrypterInterface> transformer_;
};
```

## Simplified Implementation

`modules/sframe/sframe_sender_transform.cc`
```cpp
SFrameSenderTransform::SFrameSenderTransform(const SFrameTransformOptions& options,
                                             RtpSenderInterface* host,
                                             Thread* worker_thread) :
    options_{options},
    host_{host} {
  // Create appropriate transformer based on SFrame mode
  transformer_ = CreateSFrameEncrypterProxy(
      worker_thread,
      webrtc::make_ref_counted<SFrameSenderTransformer>(options_));
  
  if (host_) {
    host_->SetFrameTransformer(transformer_);
  }
}

SFrameSenderTransform::~SFrameSenderTransform() {
  if (host_) {
    host_->SetFrameTransformer(nullptr);
  }
}

SFrameSenderTransform::SetEncryptionKey(const std::string& key,
                                        CryptoKeyID key_id) {
    if (transformer_) {
        transformer_->SetEncryptionKey(key, key_id);
    }
}
```

## Factory

`SFrameSenderTransform` constructor will pick up following arguments:
* `SFrameTransformOptions` - as described above, options defining how encryption should work.
* `RtpSenderInterface` - sender to which sframe should be applied.
* `Thread` - worker thread to which performing tasks should be delegated.

It will be created by the exposed factory method:
`api/sframe/sframe_transform_factory.h`
```cpp
namespace webrtc {
  std::unique_ptr<SFrameEncrypterManager> CreateSFrameSenderTransform(
    const SFrameTransformOptions& options,
    RtpSenderInterface* host,
    Thread* worker_thread);
}
```

`api/sframe/sframe_transform_factory.cc`
```cpp
namespace webrtc {
  std::unique_ptr<SFrameEncrypterManager> CreateSFrameSenderTransform(
      const SFrameTransformOptions& options,
      RtpSenderInterface* host,
      Thread* worker_thread) {
    return std::make_unique<SFrameSenderTransform>(options, host, worker_thread);
  }
}
```

## SFrameSenderTransformer

`SFrameSenderTransformer` will be an implementation of the transformer responsible for performing SFrame encryption. It will derive from `SFrameEncrypterInterface` defined [here](README.md#sframeencrypterinterface).

```cpp
class SFrameSenderTransformer : public SFrameEncrypterInterface {
public:
  virtual void Transform(
      std::unique_ptr<TransformableFrameInterface> transformable_frame) {
        // Perform encryption
      }

  virtual void SetEncryptionKey(const std::string& key, CryptoKeyID key_id) {
    // Set key
  }

  virtual void RegisterTransformedFrameCallback(
      scoped_refptr<TransformedFrameCallback>) {}
  virtual void RegisterTransformedFrameSinkCallback(
      scoped_refptr<TransformedFrameCallback>,
      uint32_t /* ssrc */) {}
  virtual void UnregisterTransformedFrameCallback() {}
  virtual void UnregisterTransformedFrameSinkCallback(uint32_t /* ssrc */) {}

private:
    scoped_refptr<TransformedFrameCallback> callback_;
};
```

## Proxy

To make sure that calls are done on appropriate threads, proxy for the `SFrameSenderTransformer` should be defined.

Most of the functions will be called directly on the `Worker` thread, so no proxy is required. The most important part is to properly delegate setting a key (`SetEncryptionKey`) to the `Worker` thread. Destruction of the `Transformer` should also be done on the `Worker` thread to ensure smooth integration with existing underlying implementation.

```cpp
namespace webrtc {

BEGIN_PRIMARY_PROXY_MAP(SFrameEncrypter)

PROXY_PRIMARY_THREAD_DESTRUCTOR()

// SFrameEncrypterInterface methods
PROXY_METHOD2(void,
              SetEncryptionKey,
              const std::string&,
              CryptoKeyID)

// FrameTransformerInterface methods
BYPASS_PROXY_METHOD1(void,
              Transform,
              std::unique_ptr<TransformableFrameInterface>)
    
BYPASS_PROXY_CONSTMETHOD0(size_t, GetReservedNumberOfBytes)

BYPASS_PROXY_METHOD1(void,
              RegisterTransformedFrameCallback,
              scoped_refptr<TransformedFrameCallback>)

BYPASS_PROXY_METHOD2(void,
              RegisterTransformedFrameSinkCallback,
              scoped_refptr<TransformedFrameCallback>,
              uint32_t)

BYPASS_PROXY_METHOD0(void, UnregisterTransformedFrameCallback)

BYPASS_PROXY_METHOD1(void,
              UnregisterTransformedFrameSinkCallback,
              uint32_t)

END_PROXY_MAP(SFrameEncrypter)

}  // namespace webrtc
```

`Proxy` creation with the factory method:

`modules/sframe/sframe_transformer_proxy.h`
```cpp
namespace webrtc {

scoped_refptr<SFrameEncrypterInterface> CreateSFrameEncrypterProxy(
    Thread* worker_thread,
    scoped_refptr<SFrameEncrypterInterface> sframe_transformer) {
  return SFrameEncrypterProxy::Create(
      worker_thread, 
      sframe_transformer);
}

}  // namespace webrtc
```