# SFrame Public API Definitions

This is the public API surface an application uses to enable SFrame and manage
its keys. The types live under `api/sframe/`; the creation entry points are
methods on the existing `RtpSenderInterface` and `RtpReceiverInterface`.

How these pieces fit together is described in
[transceiver-enablement.md](transceiver-enablement.md).

Each type below maps to a definition in the W3C
[WebRTC Encoded Transform specification](https://w3c.github.io/webrtc-encoded-transform/#sframe-transforms).
The native API uses the same modes, cipher suites, and key-management
operations; it exposes them through sender/receiver creation methods rather than
through constructible transform objects.

## `SframeMode`

Selects how the sender applies SFrame protection. Corresponds to the spec enum
[`SFrameType`](https://w3c.github.io/webrtc-encoded-transform/#enumdef-sframetype).

| Value | Meaning | Specification |
|---|---|---|
| `kPerFrame` | Protect each encoded frame as a unit (decrypt after frame reassembly) | [`"per-frame"`](https://w3c.github.io/webrtc-encoded-transform/#dom-sframetype-per-frame) |
| `kPerPacket` | Protect each packetized payload (decrypt each RTP payload before depacketization) | [`"per-packet"`](https://w3c.github.io/webrtc-encoded-transform/#dom-sframetype-per-packet) |

## `SframeCipherSuite`

Selects the AEAD cipher and authentication-tag length used for SFrame.
Corresponds to the spec enum
[`SFrameCipherSuite`](https://w3c.github.io/webrtc-encoded-transform/#enumdef-sframeciphersuite),
whose values are defined in [RFC 9605].

| Value | Cipher | Authentication tag | Specification |
|---|---|---|---|
| `kAes128CtrHmacSha256_80` | AES-128-CTR with HMAC-SHA-256 | 80-bit tag | [`AES_128_CTR_HMAC_SHA256_80`](https://w3c.github.io/webrtc-encoded-transform/#dom-sframeciphersuite-aes_128_ctr_hmac_sha256_80) |
| `kAes128CtrHmacSha256_64` | AES-128-CTR with HMAC-SHA-256 | 64-bit tag | [`AES_128_CTR_HMAC_SHA256_64`](https://w3c.github.io/webrtc-encoded-transform/#dom-sframeciphersuite-aes_128_ctr_hmac_sha256_64) |
| `kAes128CtrHmacSha256_32` | AES-128-CTR with HMAC-SHA-256 | 32-bit tag | [`AES_128_CTR_HMAC_SHA256_32`](https://w3c.github.io/webrtc-encoded-transform/#dom-sframeciphersuite-aes_128_ctr_hmac_sha256_32) |
| `kAes128GcmSha256_128` | AES-128-GCM | 128-bit tag | [`AES_128_GCM_SHA256_128`](https://w3c.github.io/webrtc-encoded-transform/#dom-sframeciphersuite-aes_128_gcm_sha256_128) |
| `kAes256GcmSha512_128` | AES-256-GCM | 128-bit tag | [`AES_256_GCM_SHA512_128`](https://w3c.github.io/webrtc-encoded-transform/#dom-sframeciphersuite-aes_256_gcm_sha512_128) |

## `SframeEncryptorInit`

Carries the sender's choices into the encryptor creation call. The sender picks
both the mode and the cipher suite. Corresponds to the spec dictionary
[`RTCRtpSFrameEncryptorOptions`](https://w3c.github.io/webrtc-encoded-transform/#dictdef-rtcrtpsframeencryptoroptions)
(which extends [`SFrameTransformOptions`](https://w3c.github.io/webrtc-encoded-transform/#dictdef-sframetransformoptions)).

| Field | Type | Meaning | Specification |
|---|---|---|---|
| `mode` | `SframeMode` | Per-frame or per-packet protection | [`RTCRtpSFrameEncryptorOptions.type`](https://w3c.github.io/webrtc-encoded-transform/#dom-rtcrtpsframeencryptoroptions-type) |
| `cipher_suite` | `SframeCipherSuite` | AEAD cipher and tag length | [`SFrameTransformOptions.cipherSuite`](https://w3c.github.io/webrtc-encoded-transform/#dom-sframetransformoptions-ciphersuite) |

## `SframeEncryptorInterface`

Reference-counted key-management handle returned to the application for a
sender. It is the only API used to install or replace the sending key after
SFrame is enabled; the media crypto object itself stays inside the pipeline.
Corresponds to the spec mixin
[`SFrameEncryptorManager`](https://w3c.github.io/webrtc-encoded-transform/#sframeencryptormanager).

| Method | Parameters | Returns | Description | Specification |
|---|---|---|---|---|
| `SetEncryptionKey` | `key_id` (unsigned 64-bit), `key_material` (byte span) | `RTCError` | Installs or replaces the sender's key for the given key id | [`setEncryptionKey`](https://w3c.github.io/webrtc-encoded-transform/#dom-sframeencryptormanager-setencryptionkey) |

## `SframeDecryptorInterface`

Reference-counted key-management handle returned to the application for a
receiver. It is the only API used to register or revoke receive keys after
SFrame is enabled. Corresponds to the spec mixin
[`SFrameDecryptorManager`](https://w3c.github.io/webrtc-encoded-transform/#sframedecryptormanager).

| Method | Parameters | Returns | Description | Specification |
|---|---|---|---|---|
| `AddDecryptionKey` | `key_id` (unsigned 64-bit), `key_material` (byte span) | `RTCError` | Registers a receive key for the given key id | [`addDecryptionKey`](https://w3c.github.io/webrtc-encoded-transform/#dom-sframedecryptormanager-adddecryptionkey) |
| `RemoveDecryptionKey` | `key_id` (unsigned 64-bit) | `RTCError` | Revokes a previously registered receive key | [`removeDecryptionKey`](https://w3c.github.io/webrtc-encoded-transform/#dom-sframedecryptormanager-removedecryptionkey) |

`key_id` identifies the key within the SFrame header; `key_material` is the raw
key bytes for the selected cipher suite.

## Creation Entry Points

These methods enable SFrame on the owning transceiver and, on success, return a
key-management handle. They are the application's single entry point for turning
SFrame on. They are the native counterpart of constructing a spec SFrame
transform and assigning it to a sender or receiver.

| Method | Parameter | Returns | On failure | Specification |
|---|---|---|---|---|
| `RtpSenderInterface::CreateSframeEncryptorOrError` | `SframeEncryptorInit` | A send-side `SframeEncryptorInterface` handle | `RTCError` (for example, `InvalidModification` if SFrame was already negotiated off) | [`RTCRtpSFrameEncryptor`](https://w3c.github.io/webrtc-encoded-transform/#rtcrtpsframeencryptor) |
| `RtpReceiverInterface::CreateSframeDecryptorOrError` | `SframeCipherSuite` | A receive-side `SframeDecryptorInterface` handle | `RTCError` with no handle created | [`RTCRtpSFrameDecryptor`](https://w3c.github.io/webrtc-encoded-transform/#rtcrtpsframedecryptor) |

[RFC 9605]: https://www.rfc-editor.org/rfc/rfc9605.html
