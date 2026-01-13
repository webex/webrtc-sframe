# SFrameReceiverTransform

## Definition

`SFrameReceiverTransform` will be a class dedicated for performing an SFrame on the receiver.
It's role will be to create appropriate transformer and configure receiver for provided transformation type.

It's role will be to:
* Create appropriate `transformer` and inject it to the transformer slot
* Expose a API necessary to perform adding/removing keys operations

It won't tell transceiver to enable SFrame, user have to do it manually with `SetUseSFrame` call.

`modules/sframe/sframe_receiver_transform.h`
```cpp
class SFrameReceiverTransform : public SFrameDecrypterManager {
  SFrameReceiverTransform(const SFrameTransformOptions& options,
                          RtpReceiverInterface* host,
                          Thread* worker_thread);
  ~SFrameReceiverTransform() override;

  bool AddDecryptionKey(const std::string& key, CryptoKeyID key_id) override;
  bool RemoveDecryptionKey(CryptoKeyID key_id) override;

private:
  const SFrameTransformOptions options_;
  RtpReceiverInterface* host_;
  scoped_refptr<SFrameDecrypterInterface> transformer_;
};
```

## Simplified Implementation

`modules/sframe/sframe_receiver_transform.cc`
```cpp
SFrameReceiverTransform::SFrameReceiverTransform(const SFrameTransformOptions& options,
                                                 RtpReceiverInterface* host,
                                                 Thread* worker_thread) :
    options_{options},
    host_{host} {
  // Create appropriate transformer based on SFrame mode
  transformer_ = CreateSFrameDecrypterProxy(
      worker_thread,
      webrtc::make_ref_counted<SFrameReceiverTransformer>(options_));
  
  if (host_) {
    host_->SetFrameTransformer(transformer_);
  }
}

SFrameReceiverTransform::~SFrameReceiverTransform() {
  if (host_) {
    host_->SetFrameTransformer(nullptr);
  }
}

SFrameReceiverTransform::AddDecryptionKey(const std::string& key,
                                          CryptoKeyID key_id) {
    if (transformer_) {
        transformer_->AddDecryptionKey(key, key_id);
        return true;
    }
    return false;
}

SFrameReceiverTransform::RemoveDecryptionKey(CryptoKeyID key_id) {
    if (transformer_) {
        transformer_->RemoveDecryptionKey(key_id);
        return true;
    }
    return false;
}
```

## Factory

`SFrameReceiverTransform` constructor will pick up following arguments:
* `SFrameTransformOptions` - as described above, options defining how decryption should work.
* `RtpReceiverInterface` - receiver to which sframe should be applied.
* `Thread` - worker thread to which performing tasks should be delegated.

It will be created by the exposed factory method:
`api/sframe/sframe_transform_factory.h`
```cpp
namespace webrtc {
  std::unique_ptr<SFrameDecrypterManager> CreateSFrameReceiverTransform(
    const SFrameTransformOptions& options,
    RtpReceiverInterface* host,
    Thread* worker_thread);
}
```

`api/sframe/sframe_transform_factory.cc`
```cpp
namespace webrtc {
  std::unique_ptr<SFrameDecrypterManager> CreateSFrameReceiverTransform(
      const SFrameTransformOptions& options,
      RtpReceiverInterface* host,
      Thread* worker_thread) {
    return std::make_unique<SFrameReceiverTransform>(options, host, worker_thread);
  }
}
```

## SFrameReceiverTransformer

`SFrameReceiverTransformer` will be an implementation of the transformer responsible for performing SFrame decryption. It will derive from `SFrameDecrypterInterface` defined [here](README.md#sframedecryptorinterface)

```cpp
class SFrameReceiverTransformer : public SFrameDecrypterInterface {
public:
  virtual void Transform(
      std::unique_ptr<TransformableFrameInterface> transformable_frame) {
        // Perform decryption
      }

  virtual void AddDecryptionKey(const std::string& key, CryptoKeyID key_id) {
    // Set key
  }

  virtual void RemoveDecryptionKey(CryptoKeyID key_id) {
    // Remove key
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

To make sure that calls are done on appropriate threads, proxy for the `SFrameReceiverTransformer` should be defined.

Most of the functions will be called directly on the `Worker` thread, so no proxy is required. The most important part is to properly delegate setting/removing keys (`AddDecryptionKey`, `RemoveDecryptionKey`) to the `Worker` thread. Destruction of the `Transformer` should also be done on the `Worker` thread to ensure smooth integration with existing underlying implementation.

```cpp
namespace webrtc {

BEGIN_PRIMARY_PROXY_MAP(SFrameDecrypter)

PROXY_PRIMARY_THREAD_DESTRUCTOR()

// SFrameDecrypterInterface methods
PROXY_METHOD2(void,
              AddDecryptionKey,
              const std::string&,
              CryptoKeyID)

PROXY_METHOD1(void,
              RemoveDecryptionKey,
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

END_PROXY_MAP(SFrameDecrypter)

}  // namespace webrtc
```