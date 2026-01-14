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

##### TransformationFeatures Interface

`TransformationFeatures` interface class will define features which `FrameTransformer` can implement.

`api/frame_transformer_interface.h`
```cpp
class TransformationFeatures {
 public:
  virtual ~TransformationFeatures() {}

  virtual bool UseSFrame() const { return false; }
};
```

`FrameTransformerInterface` will derive from this interface, and give underlying implementation ability to specify what features it implements.
It will be used by the underlying WebRTC implementation to find out how it should behave.
By default all features should be disabled.

##### FrameTransformerInterface

`FrameTransformerInterface` as described above, it will derive from `TransformationFeatures` to give implementators of `FrameTransformerInterface` specify what kind fo features it supports.

`api/frame_transformer_interface.h`
```cpp
class FrameTransformerInterface : public RefCountInterface, 
                                  public TransformationFeatures {
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

##### PacketTransformerInterface

`PacketTransformerInterface` will be a transformer interface dedicated for packet transformation.
It's very similar to the frame transformer, however it requires one additional method - `GetReservedNumberOfBytes`.
It's necessary as user of the transformer might want to add some data to the packet, which could lead to exceeding the MTU size. This function will be dedicated for the user to inform packetizer how much data user would like inject after packetization.

`PacketTransformerInterface` will reuse `TransformableFrameInterface` and other types defined for `Frame` transformer, as it provides sufficient data for packets to work.

`PacketTransformerInterface` will also derive from `TransformationFeatures` as it also has to be able to inform WebRTC implementation what kind of transformation is being done.

```cpp
class PacketTransformerInterface : public RefCountInterface, 
                                   public TransformationFeatures {
 public:
  // Transforms `frame` using the implementing class' processing logic.
  virtual void Transform(
      std::unique_ptr<TransformableFrameInterface> transformable_frame) = 0;

  virtual size_t GetReservedNumberOfBytes() const = 0;

  virtual void RegisterTransformedPacketCallback(
      scoped_refptr<TransformedFrameCallback>) {}
  virtual void RegisterTransformedPacketSinkCallback(
      scoped_refptr<TransformedFrameCallback>,
      uint32_t /* ssrc */) {}
  virtual void UnregisterTransformedPacketCallback() {}
  virtual void UnregisterTransformedPacketSinkCallback(uint32_t /* ssrc */) {}

 protected:
  ~PacketTransformerInterface() override = default;
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
    
+  virtual void SetPacketTransformer(
+      scoped_refptr<PacketTransformerInterface> packet_transformer) = 0;
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

+  void SetPacketTransformer(
+      scoped_refptr<PacketTransformerInterface> packet_transformer) override {}
};
```

`api/rtp_receiver_interface.h`
```cpp
class RtpReceiverInterface : public RefCountInterface,
                             public FrameTransformerHost {
 public:
  void SetFrameTransformer(
      scoped_refptr<FrameTransformerInterface> frame_transformer) override {}

+  void SetPacketTransformer(
+      scoped_refptr<PacketTransformerInterface> packet_transformer) override {}
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

#### SFrameTransformOptions Configuration

Defines how SFrame transforms are applied to media data.

`api/sframe/sframe_transform_options.h`
```cpp
enum class SFrameMode {
  kFrame,   // Frame-level transformation (default)
  kPacket   // Packet-level transformation
};
```

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
  SFrameMode sframe_mode = kFrame;

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

### Triggering negotiation

There are two possibilities how from the API perspective we could approach triggering SFrame.

One assumes changes to the interface of the `RtpTransceiverInterface` (option 1).
In this scenario, `SFrame` negotiation is triggered manually by the user.

Second approach doesnt require any changes to the `RtpTransceiverInterface`. `SFrame` is being enabled/disabled automatically by assigning a `FrameTransformer` or `PacketTransformer` to the Sender/Receiver with `UseSFrame` transformation feature enabled. It would automatically inform transceiver about this.
Transceiver would propagate new sframe configuration to remaining senders/receivers and trigger `ONN`.

#### Option 1

First option would be to trigger neegotiation by directly setting `UseSFrame` flag on the transceiver.

##### RtpTransceiverInterface

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

Setting `SFrame` mode with the call to the `SetUseSFrame` will propagate sframe request to underlying `Senders`/`Receivers` and then trigger `ONN`.

##### Initialization flow

User of the `C++` api to enable SFrame would need to take following steps assuming that we already have created `Peer Connection` and a `Transceiver`.
1. Call a `SetUseSFrame` on the `Transceiver.
1. Call a `CreateSFrameSenderTransform` on the `Sender` associated with the `Transceiver`.
1. Call a `CreateSFrameSenderTransform` on the `Sender` associated with the `Transceiver`

```cpp
PeerConnection pc{};

auto transceiver = pc.addTransceiver('video');

transceiver.SetUseSFrame(true);

SFrameTransformOptions options{};

auto sframe_transform = CreateSFrameSenderTransform(options,
                                                    transceiver.sender(),
                                                    worker_thread);
// CreateSFrameSenderTransform would perform following actions:
// auto transformer = new Frame/Packet Transformer (depending on provided options)
// transformer->SetFrameTransformer/SetPacketTransformer (depending on provided options)
```

#### Option 2

Second approach assumes that no `RtpTransceiver` interface changes is needed.
`SFrame` initialization would be done with the call to the `SetFrameTransformer` and `SetPacketTransformer`, which would check on the `Sender` and `Receiver` level verify if it has `UseSFrame` transformation feature set. If it does, then we would inform transceiver with internal API's, about SFrame request, and then transceiver would propagate down information about SFrame usage.

##### Initialization flow

User of the `C++` api to enable SFrame would need to take following steps assuming that we already have created `Peer Connection` and a `Transceiver`.
1. Call a `CreateSFrameSenderTransform` on the `Sender` associated with the `Transceiver`
1. Call a `CreateSFrameReceiverTransform` on the `Receiver` associated with the `Transceiver`

```cpp
PeerConnection pc{};

auto transceiver = pc.addTransceiver('video');

SFrameTransformOptions options{};

auto sframe_transform = CreateSFrameSenderTransform(options,
                                                    transceiver.sender(),
                                                    worker_thread);
// CreateSFrameSenderTransform would perform following actions:
// auto transformer = new Frame/Packet Transformer (depending on provided options)
// transformer->SetFrameTransformer/SetPacketTransformer (depending on provided options)
```