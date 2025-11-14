# WebRTC SFrame Integration

This document describes how to integrate SFrame (Secure Frame) encryption into WebRTC applications. SFrame provides end-to-end media security that works even when media flows through untrusted servers or intermediaries.

## Overview

SFrame encryption can be applied at two different levels:
- **Per-frame**: Encrypts complete video/audio frames before packetization
- **Per-packet**: Encrypts individual RTP packets after packetization

Both approaches offer strong security guarantees, with per-packet providing finer granularity and per-frame offering better performance characteristics.

## JavaScript API

### SFrame Modes

```javascript
// Available modes for SFrame encryption
const SFrameMode = {
    "per-frame",   // Encrypt at frame level
    "per-packet"   // Encrypt at packet level
};
```

### Basic Usage

Assumption; JS API usage after recent W3C meeting.

```javascript
// Create peer connection
const peerConnection = new RTCPeerConnection();

// Add video track
const videoTransceiver = peerConnection.addTransceiver("video", {
  direction: "sendonly"
});

// Configure SFrame options
const sframeOptions = {
  mode: "per-packet"  // or "per-frame"
};

// Apply SFrame transform
videoTransceiver.sender.transform = new SFrameTransform(sframeOptions);
```

### Key Points

- SFrame can be configured independently for each media track
- Both sender and receiver need compatible SFrame configurations
- The API maintains backward compatibility with non-SFrame endpoints

## C++ API

The SFrame API design provides transformer injection at the sender and receiver level, following the JavaScript API pattern:

```javascript
videoTransceiver.sender.transform = new SFrameTransform(sframeOptions);
```

Following this design proposal principles, the `SetSFrameTransformer` method will be exposed through the `RTPSenderInterface` and `RTPReceiverInterface` to provide this granular control and maintain consistency with the JavaScript API surface.

### Core Interfaces

#### SFrameMode Enumeration

Defines the granularity at which SFrame encryption is applied to media data.

`api/sframe/sframe_options.h`
```cpp
enum SFrameMode {
  kPerFrame,   // Frame-level encryption
  kPerPacket   // Packet-level encryption
};
```

#### SFrameOptions Configuration

Configuration structure that defines SFrame encryption behavior.

`api/sframe/sframe_options.h`
```cpp
struct SFrameOptions {
    SFrameMode mode;
    // Additional options can be added here in the future
};
```

#### SFrameTransformerHost Interface

The `SFrameTransformerHost` interface provides a unified way to inject SFrame transformers into WebRTC components. This interface will be implemented by RTP senders and receivers to enable SFrame encryption and decryption capabilities.

`api/sframe/sframe_transformer_interface.h`
```cpp
class SFrameTransformerHost {
 public:
  virtual ~SFrameTransformerHost() = default;

  // Configures SFrame transformer with specified options
  virtual void SetSFrameTransformer(
      scoped_refptr<SFrameTransformerInterface> sframe_transformer,
      SFrameOptions options) = 0;

  // Retrieves currently configured transformer (may return nullptr)
  virtual scoped_refptr<SFrameTransformerInterface> GetSFrameTransformer() = 0;
};
```

#### SFrame Transformer Interface

The main interface for implementing SFrame encryption/decryption:

`api/sframe/sframe_transformer_interface.h`
```cpp
class SFrameTransformerInterface : public RefCountInterface {
 public:
  virtual ~SFrameTransformerInterface() = default;

  // Configure encryption keys
  virtual void SetEncryptionKey(/*Keys*/) = 0;

  // Transform media data (encrypt or decrypt)
  virtual void Transform(CopyOnWriteBuffer buffer) = 0;
};
```

#### Integration with RTP Senders/Receivers

RTP interfaces inherit from `SFrameTransformerHost` to support SFrame injection:

`api/rtp_sender_interface.h`
```cpp
class RtpSenderInterface : public SFrameTransformerHost {
 public:
  // Default implementation, actuall implementation will be done by classes inheriting RtpSenderInterface
  void SetSFrameTransformer(
      scoped_refptr<SFrameTransformerInterface> sframe_transformer,
      SFrameOptions options) {};
};
```

`api/rtp_receiver_interface.h`
```cpp
class RtpReceiverInterface : public SFrameTransformerHost {
 public:
 // Default implementation, actuall implementation will be done by classes inheriting RtpReceiverInterface
  void SetSFrameTransformer(
      scoped_refptr<SFrameTransformerInterface> sframe_transformer,
      SFrameOptions options) {};
};
```

### Cisco SFrame Library

WebRTC will provide a built-in SFrame implementation using the [Cisco SFrame](https://github.com/cisco/sframe) library.

#### Encrypt or decrypt

Decision whether SFrameTransformer should perform encryption or decryption will be defined by the `SFrameRole` parameter.

`api/sframe/sframe_transformer.h`
```cpp
enum SFrameRole {
  Encrypt,
  Decrypt
};
```

#### Supported Cipher Suites

`api/sframe/sframe_transformer.h`
```cpp
enum SFrameCipherSuite {
  AES_128_CTR_HMAC_SHA256_8,   // AES-128-CTR with 8-byte auth tag
  AES_128_CTR_HMAC_SHA256_64,  // AES-128-CTR with 64-byte auth tag
  AES_128_CTR_HMAC_SHA256_32,  // AES-128-CTR with 32-byte auth tag
  AES_128_GCM_SHA256_128,      // AES-128-GCM with 128-bit auth tag
  AES_256_GCM_SHA512_128       // AES-256-GCM with 128-bit auth tag
};
```

#### RTCSFrameTransformer

`RTCSFrameTransformer` will leverage [Cisco SFrame](https://github.com/cisco/sframe) library to implement SFrame encryption/decryption.

`modules/sframe/rtc_sframe_transformer.h`
```cpp
class RTCSFrameTransformer : public SFrameTransformerInterface {
 public:
  explicit RTCSFrameTransformer(SFrameRole role, SFrameCipherSuite cipher_suite);

  void SetEncryptionKey(std::string key) override {
    // Update keys in SFrame Context
  }

  // Transform media data (encrypt or decrypt)
  void Transform(CopyOnWriteBuffer buffer) override {
    // role == SFrameRole::Encrypt ? Encrypt(buffer) : Decrypt(buffer);
  }

private:
  void Encrypt(CopyOnWriteBuffer buffer);

  void Decrypt(CopyOnWriteBuffer buffer);
};
```

#### RTCSFrameTransformer Factory

`RTCSFrameTransformer` creation will be possible throught the factory method exposed from the libwebrtc API.

`api/sframe/sframe_transformer_factory.h`
```cpp
scoped_refptr<SFrameTransformerInterface> CreateSFrameTransformer(SFrameRole role, SFrameCipherSuite cipher_suite);
```

```cpp
scoped_refptr<SFrameTransformerInterface> CreateSFrameTransformer(SFrameRole role, SFrameCipherSuite cipher_suite)
{
  return make_ref_counted<RTCSFrameTransformer>(role, cipher_suite);
}
```

### Complete Usage Example

```cpp
// Create peer connection
RTCPeerConnection pc;

// Add video transceiver
auto transceiver = pc.AddTransceiver("video");

// Create SFrame transformer for encryption
auto sframe_transformer = CreateSFrameTransformer(SFrameRole::kEncrypt, 
    SFrameCipherSuite::AES_128_GCM_SHA256_128);

// Configure for per-packet encryption
SFrameOptions options;
options.mode = SFrameMode::kPerPacket;

// Apply to sender
transceiver->sender()->SetSFrameTransformer(sframe_transformer, options);

sframe_transformer->SetEncryptionKey("XYZ");
```

## Additional Resources

### Related Specifications
- [SFrame Protocol](https://datatracker.ietf.org/doc/draft-omara-sframe/) - The underlying SFrame specification
- [WebRTC API](https://w3c.github.io/webrtc-pc/) - W3C WebRTC specification
- [Cisco SFrame Library](https://github.com/cisco/sframe) - Reference implementation
- [RTP SFrame](https://datatracker.ietf.org/doc/draft-ietf-avtcore-rtp-sframe)
- [Encoded Transform](https://www.w3.org/TR/webrtc-encoded-transform)

**Open Questions:**
- Should we use `CopyOnWriteBuffer` or rather `ArraView<uint8_t>` for transformation
- Do we need on libwebrtc level to have single "Transformer" interface. 
  Maybe it's fine to split it into encrypter and decrypter, and let browser level code decide which to create.
  Instead of having an generic `RTCSFrameTransformer` with `SFrameRole` in the parameter, we could create a separate objects for handling encryption and decryption.

```cpp
  class RTCSFrameEncryptor : public SFrameTransformerInterface {
  public:
    explicit RTCSFrameEncryptor(SFrameCipherSuite cipher_suite);
    // Implementation handles the crypto details
  };

  class RTCSFrameDecryptor : public SFrameTransformerInterface {
  public:
    explicit RTCSFrameDecryptor(SFrameCipherSuite cipher_suite);
    // Implementation handles the crypto details
  };
```