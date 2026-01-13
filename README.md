# WebRTC SFrame Integration

This document describes how to integrate SFrame (Secure Frame) encryption into WebRTC applications. SFrame provides end-to-end media security that works even when media flows through untrusted servers or intermediaries.

## Overview

SFrame encryption can be applied at two different levels:
- **Per-frame**: Encrypts complete video/audio frames before packetization
- **Per-packet**: Encrypts individual RTP packets after packetization

Both approaches offer strong security guarantees, with per-packet providing finer granularity and per-frame offering better performance characteristics.

## C++ API

The SFrame API design provides transformer injection at the sender and receiver level, following the JavaScript API pattern:

### Core Interfaces

#### RtpTransceiverInterface

Extend `RtpTransceiverInterface` to give possibility to the caller to enable `SFrame` support for this `Transceiver`.

Setting `SFrame` should trigger neegotiation needed event.

`api/rtp_transceiver_interface.h`
```cpp
class RTC_EXPORT RtpTransceiverInterface : public RefCountInterface {
 public:

  /* Existing fields */

  virtual void SetUseSFrame(bool use_sframe) = 0;

  virtual bool UseSFrame() const = 0;
}
```

#### SFrameTransformOptions Configuration

Defines which cipher suite SFrame transform will use.

`api/sframe/sframe_transform_options.h`
```cpp
enum class SFrameCipherSuite {
  kAES_128_CTR_HMAC_SHA256_80,
  kAES_128_CTR_HMAC_SHA256_64,
  kAES_128_CTR_HMAC_SHA256_32,
  kAES_128_GCM_SHA256_128,
  kAES_256_GCM_SHA512_128
};
```

Configuration structure that defines SFrame transform behavior.

`api/sframe/sframe_transform_options.h`
```cpp
struct SFrameTransformOptions {
  SFrameCipherSuite cipher_suite = SFrameCipherSuite::kAes128GcmSha256;
};
```

#### FrameTransformer Interfaces

##### FrameTransformerInterface Interface

`FrameTransformerInterface` will remain the same as no changes are needed.

`api/frame_transformer_interface.h`
```cpp
class FrameTransformerInterface : public RefCountInterface {
 public:
  // Transforms `frame` using the implementing class' processing logic.
  virtual void Transform(
      std::unique_ptr<TransformableFrameInterface> transformable_frame) = 0;

  virtual void RegisterTransformedFrameCallback(
      scoped_refptr<TransformedFrameCallback>) {}
  virtual void RegisterTransformedFrameSinkCallback(
      scoped_refptr<TransformedFrameCallback>,
      uint32_t /* ssrc */) {}
  virtual void UnregisterTransformedFrameCallback() {}
  virtual void UnregisterTransformedFrameSinkCallback(uint32_t /* ssrc */) {}

 protected:
  ~FrameTransformerInterface() override = default;
};
```

#### FrameTransformerHost Interface

`FrameTransformerHost` will remain the same as no changes are needed.

`api/frame_transformer_interface.h`
```cpp
class FrameTransformerHost {
 public:
  virtual ~FrameTransformerHost() = default;

  // Sets frame-level transformer
  virtual void SetFrameTransformer(
      scoped_refptr<FrameTransformerInterface> frame_transformer) = 0;
};
```

#### Integration with RTP Senders/Receivers

RTP interfaces inherit from `FrameTransformerHost` to support transformer injection:

`api/rtp_sender_interface.h`
```cpp
class RtpSenderInterface : public RefCountInterface,
                           public FrameTransformerHost {
 public:
  void SetFrameTransformer(
      scoped_refptr<FrameTransformerInterface> frame_transformer) override {}
};
```

`api/rtp_receiver_interface.h`
```cpp
class RtpReceiverInterface : public RefCountInterface,
                             public FrameTransformerHost {
 public:
  void SetFrameTransformer(
      scoped_refptr<FrameTransformerInterface> frame_transformer) override {}
};
```

#### SFrame Key Management Interfaces

Specialized interfaces for encryption and decryption key management:

`api/sframe/sframe_key_management.h`
```cpp
// Interface for encryption key management
class SFrameEncrypterManager {
 public:
  virtual ~SFrameEncrypterManager() = default;
  
  virtual bool SetEncryptionKey(const std::string& key,
                                CryptoKeyID key_id) = 0;
};

// Interface for decryption key management
class SFrameDecrypterManager {
 public:
  virtual ~SFrameDecrypterManager() = default;
  
  virtual bool AddDecryptionKey(const std::string& key,
                                CryptoKeyID key_id) = 0;
  
  virtual bool RemoveDecryptionKey(CryptoKeyID key_id) = 0;
};
```

#### SFrameEncrypterInterface

`SFrameEncrypterInterface` will extend  `FrameTransformerInterface` to give `Transformer` ability to handle key changing for encryption.

```cpp
class SFrameEncrypterInterface : public FrameTransformerInterface {
 public:
  virtual ~SFrameEncrypterInterface() = default;

  virtual void SetEncryptionKey(const std::string& key, CryptoKeyID key_id) = 0;
};
```

#### SFrameDecrypterInterface

`SFrameDecrypterInterface` will extend  `FrameTransformerInterface` to give `Transformer` ability to handle key changing for decryption.

```cpp
class SFrameDecrypterInterface : public FrameTransformerInterface {
 public:
  virtual ~SFrameDecrypterInterface() = default;

  virtual void AddDecryptionKey(const std::string& key, CryptoKeyID key_id) = 0;

  virtual void RemoveDecryptionKey(CryptoKeyID key_id) = 0;
};
```

#### SFrameSenderTransformFactory

`Factory` methods will pick up following arguments:
* `SFrameTransformOptions` - as described above, options defining how encryption should work.
* `RtpSenderInterface` - sender to which sframe should be applied.
* `Thread` - worker thread to which performing tasks should be delegated.

`api/sframe/sframe_transform_factory.h`
```cpp
namespace webrtc {

// Factory method to create SFrame sender transform (encryption).
std::unique_ptr<SFrameEncrypterManager> CreateSFrameSenderTransform(
    const SFrameTransformOptions& options,
    scoped_refptr<RtpSenderInterface> sender,
    Thread* worker_thread = nullptr);

// Factory method to create SFrame receiver transform (decryption).
std::unique_ptr<SFrameDecrypterManager> CreateSFrameReceiverTransform(
    const SFrameTransformOptions& options,
    scoped_refptr<RtpReceiverInterface> receiver,
    Thread* worker_thread = nullptr);

}  // namespace webrtc
```

`api/sframe/sframe_transform_factory.cc`
```cpp
namespace webrtc {

std::unique_ptr<SFrameEncrypterManager> CreateSFrameSenderTransform(
    const SFrameTransformOptions& options,
    FrameTransformerHost* host,
    Thread* worker_thread) {
  return std::make_unique<SFrameSenderTransform>(options, host, worker_thread);
}

std::unique_ptr<SFrameDecrypterManager> CreateSFrameReceiverTransform(
    const SFrameTransformOptions& options,
    FrameTransformerHost* host,
    Thread* worker_thread) {
  return std::make_unique<SFrameReceiverTransform>(options, host, worker_thread);
}

}  // namespace webrtc
```

#### SFrameSenderTransformer

### Initialization flow

User of the `C++` api to enable SFrame would need to take following steps assuming that we already have created `Peer Connection` and a `Transceiver`.
1. Call a `SetUseSFrame` on the `Transceiver.
1. Call a `CreateSFrameSenderTransform` on the `Sender` associated with the `Transceiver`
1. Call a `CreateSFrameReceiverTransform` on the `Receiver` associated with the `Transceiver`

```cpp
PeerConnection pc{};

auto transceiver = pc.addTransceiver('video');

transceiver.SetUseSFrame(true);

SFrameTransformOptions options{};

auto sframe_transform = CreateSFrameSenderTransform(options,
                                                      transceiver.sender(),
                                                      worker_thread);
```
