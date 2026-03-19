# WebRTC SFrame Integration Architecture

This document describes the architectural design for integrating SFrame (Secure Frame) encryption into WebRTC's native C++ API. SFrame provides end-to-end media security as defined by [RFC 9605](https://www.rfc-editor.org/rfc/rfc9605.html) and [draft-ietf-avtcore-rtp-sframe](https://github.com/ietf-wg-avtcore/draft-ietf-avtcore-rtp-sframe).

## Overview

SFrame encryption can be applied at two levels:
- **Per-frame** (`SframeMode::kPerFrame`): Encrypts complete video/audio frames before packetization
- **Per-packet** (`SframeMode::kPerPacket`): Encrypts individual RTP packets after packetization

Both modes are enabled through the same public API on `RtpSenderInterface` and `RtpReceiverInterface`. The application creates an SFrame encrypter or decrypter by calling a method on the sender or receiver, receives a key management handle, and the internal pipeline handles encrypter/decrypter installation and SDP negotiation automatically.

## Architecture Overview

```mermaid
sequenceDiagram
    participant App as Application
    participant T as RtpTransceiver
    participant S as RtpSender
    participant R as RtpReceiver
    participant SendPipeline as Send Pipeline
    participant RecvPipeline as Receive Pipeline
    participant EncHandle as Encrypter Handle
    participant DecHandle as Decrypter Handle

    Note over App,DecHandle: SFrame Enablement Flow

    App->>S: CreateSframeEncrypterOrError(options)
    S->>T: TryToEnableSframe()
    T-->>S: OK
    Note over S: Create SframeEncrypterImpl
    S->>SendPipeline: Install encrypter (worker thread)
    S-->>App: Return key management handle
    App->>EncHandle: Store handle

    App->>R: CreateSframeDecrypterOrError(options)
    R->>T: TryToEnableSframe()
    T-->>R: OK (already enabled)
    Note over R: Create SframeDecrypterImpl
    R->>RecvPipeline: Install decrypter (worker thread)
    R-->>App: Return key management handle
    App->>DecHandle: Store handle

    Note over T: Triggers onnegotiationneeded
    T->>App: onnegotiationneeded event

    Note over App: Offer/answer exchange includes a=sframe

    Note over App,DecHandle: Key Management (via handles)

    App->>EncHandle: SetEncryptionKey(key_id, material)
    EncHandle->>SendPipeline: Key applied (worker thread)

    App->>DecHandle: AddDecryptionKey(key_id, material)
    DecHandle->>RecvPipeline: Key applied (worker thread)

    Note over SendPipeline,RecvPipeline: ✓ Encryption/decryption active in media pipeline
```

## Public API

### SFrame Types (`api/sframe/sframe_types.h`)

```cpp
enum class SframeMode {
  kPerFrame,
  kPerPacket,
};

enum class SframeCipherSuite {
  kAesCtr128HmacSha256_80,
  kAesCtr128HmacSha256_64,
  kAesCtr128HmacSha256_32,
  kAesGcm128,
  kAesGcm256,
};
```

### SFrame Encrypter Configuration and Interface (`api/sframe/sframe_encrypter_interface.h`)

```mermaid
classDiagram
    class SframeEncrypterInit {
        +mode: SframeMode
        +cipher_suite: SframeCipherSuite
    }

    class SframeEncrypterInterface {
        <<interface>>
        +SetEncryptionKey(key_id: uint64_t, key_material: ArrayView~const uint8_t~) RTCError
    }

    SframeEncrypterInit --> SframeMode
    SframeEncrypterInit --> SframeCipherSuite
    RefCountInterface <|-- SframeEncrypterInterface
```

The encrypter init carries both `mode` (per-frame or per-packet) and `cipher_suite`. The mode determines how the send pipeline processes frames — either encrypting complete frames before packetization or encrypting individual packets after packetization. The `cipher_suite` selects the cryptographic algorithm used for encryption.

### SFrame Decrypter Configuration and Interface (`api/sframe/sframe_decrypter_interface.h`)

```mermaid
classDiagram
    class SframeDecrypterInit {
        +cipher_suite: SframeCipherSuite
    }

    class SframeDecrypterInterface {
        <<interface>>
        +AddDecryptionKey(key_id: uint64_t, key_material: ArrayView~const uint8_t~) RTCError
        +RemoveDecryptionKey(key_id: uint64_t) RTCError
    }

    SframeDecrypterInit --> SframeCipherSuite
    RefCountInterface <|-- SframeDecrypterInterface
```

The decrypter init carries only `cipher_suite` — the mode (per-frame or per-packet) is automatically determined from the received encrypted data. The decrypter supports multiple simultaneous keys to handle key rotation scenarios, where new keys are added before old keys are removed.

> **Note:** The SFrame header contains a `T` bit that indicates whether the payload is an encrypted full frame (`T=1`) or an encrypted packet (`T=0`). The decrypter uses this bit to automatically select the correct decryption path without requiring the application to specify the mode.

### API on RtpSenderInterface (`api/rtp_sender_interface.h`)

```cpp
virtual RTCErrorOr<scoped_refptr<SframeEncrypterInterface>>
CreateSframeEncrypterOrError(const SframeEncrypterInit& options) {
  RTC_DCHECK_NOTREACHED();
  return RTCError();
}
```

Creates an internal SFrame encrypter, installs it in the send pipeline, notifies the transceiver of SFrame enablement, and returns a key management handle. The returned `SframeEncrypterInterface` is used solely for key management (`SetEncryptionKey`). Can only be called once per sender — subsequent calls return an error.

**Errors:**
- `INVALID_MODIFICATION`: SFrame negotiation has already been disabled on this transceiver (SFrame can only be enabled during the initial offer/answer exchange)

### API on RtpReceiverInterface (`api/rtp_receiver_interface.h`)

```cpp
virtual RTCErrorOr<scoped_refptr<SframeDecrypterInterface>>
CreateSframeDecrypterOrError(const SframeDecrypterInit& options) {
  RTC_DCHECK_NOTREACHED();
  return RTCError();
}
```

Creates an internal SFrame decrypter, installs it in the receive pipeline, notifies the transceiver of SFrame enablement, and returns a key management handle. The returned `SframeDecrypterInterface` is used for key management (`AddDecryptionKey`, `RemoveDecryptionKey`). Can only be called once per receiver — subsequent calls return an error.

**Errors:**
- `INVALID_MODIFICATION`: SFrame negotiation has already been disabled on this transceiver (SFrame can only be enabled during the initial offer/answer exchange)

## Internal Architecture

### SframeEnablementDelegate

When `CreateSframeEncrypterOrError` or `CreateSframeDecrypterOrError` is called, the sender/receiver notifies its transceiver via the `SframeEnablementDelegate` callback. This follows the `SetStreamsObserver` pattern used throughout libwebrtc — a raw pointer to an observer interface, injected via constructor and stored as a `const` member.

```mermaid
classDiagram
    class SframeEnablementDelegate {
        <<interface>>
        +TryToEnableSframe() RTCError
    }

    class RtpTransceiver {
        -sframe enabled state: optional bool
        +TryToEnableSframe() RTCError
    }

    class RtpSenderBase {
        -state observer: SframeEnablementDelegate* const
    }

    class RtpReceiverBase {
        -state observer: SframeEnablementDelegate* const
    }

    SframeEnablementDelegate <|.. RtpTransceiver
    RtpSenderBase --> SframeEnablementDelegate : notifies
    RtpReceiverBase --> SframeEnablementDelegate : notifies
```

**Transceiver state machine:**

| SFrame enabled state | Meaning | `TryToEnableSframe()` result |
|---|---|---|
| Not yet decided | SFrame has not been enabled or disabled | Sets to enabled, returns OK |
| Enabled | SFrame is enabled on this transceiver | Returns OK (idempotent) |
| Disabled | SFrame was explicitly turned off | Returns `INVALID_MODIFICATION` error |

The transceiver's SFrame enabled state is used during SDP generation to determine whether `a=sframe` should be included in the corresponding media section. Once enabled, it cannot be reverted. The disabled state is set when an offer/answer exchange completes without SFrame — at that point, SFrame can no longer be enabled on this transceiver.

**Constructor injection:** The observer pointer is passed via constructor to `RtpSenderBase` and `RtpReceiverBase`. The transceiver passes `this` when creating sender/receiver instances through the `CreateSender()` and `CreateReceiver()` helper functions.

```mermaid
sequenceDiagram
    participant App as Application
    participant Sender as RtpSenderBase
    participant Observer as SframeEnablementDelegate (Transceiver)

    App->>Sender: CreateSframeEncrypterOrError(options)
    Sender->>Observer: TryToEnableSframe()
    Observer->>Observer: Check SFrame enabled state

    alt SFrame not yet decided
        Observer->>Observer: Mark SFrame as enabled
        Observer->>App: Trigger onnegotiationneeded
        Observer-->>Sender: RTCError::OK()
    else SFrame already enabled
        Observer-->>Sender: RTCError::OK() (idempotent)
    end

    Note over Sender: Create internal encrypter
    Note over Sender: Install encrypter in send pipeline
    Sender-->>App: RTCErrorOr<SframeEncrypterInterface> handle
```

### SframeEncrypterImpl (Internal)

The `SframeEncrypterImpl` is the internal implementation of `SframeEncrypterInterface`, created by `RtpSenderBase::CreateSframeEncrypterOrError`. It provides SFrame encryption and exposes the key management interface (returned to the app via proxy). Internally, it is installed in the send pipeline to encrypt frames or packets depending on the configured mode.

```mermaid
classDiagram
    class SframeEncrypterInterface {
        <<interface>>
        +SetEncryptionKey(key_id, key_material) RTCError
    }
    
    class SframeEncrypterImpl {
        -cipher_suite: SframeCipherSuite
        -sframe_mode: SframeMode
        +SframeEncrypterImpl(init)
        +SetEncryptionKey(key_id, key_material) RTCError
        +EncryptFrame(frame) EncryptedFrame
        +EncryptPacket(packet) EncryptedPacket
    }
    
    SframeEncrypterInterface <|-- SframeEncrypterImpl
```

### SframeDecrypterImpl (Internal)

The `SframeDecrypterImpl` is the internal implementation of `SframeDecrypterInterface`, created by `RtpReceiverBase::CreateSframeDecrypterOrError`. It provides SFrame decryption and exposes the key management interface (returned to the app via proxy). Internally, it is installed in the receive pipeline to decrypt frames or packets.

```mermaid
classDiagram
    class SframeDecrypterInterface {
        <<interface>>
        +AddDecryptionKey(key_id, key_material) RTCError
        +RemoveDecryptionKey(key_id) RTCError
    }
    
    class SframeDecrypterImpl {
        -cipher_suite: SframeCipherSuite
        +SframeDecrypterImpl(init)
        +AddDecryptionKey(key_id, key_material) RTCError
        +RemoveDecryptionKey(key_id) RTCError
        +DecryptFrame(frame) DecryptedFrame
        +DecryptPacket(packet) DecryptedPacket
    }
    
    SframeDecrypterInterface <|-- SframeDecrypterImpl
```

### CreateSframeEncrypterOrError Internal Flow

When the application calls `CreateSframeEncrypterOrError`, the sender:
1. Notifies the transceiver via the observer
2. Creates the internal `SframeEncrypterImpl`
3. Installs the encrypter in the send pipeline
4. Wraps the encrypter in a thread-safe proxy
5. Returns the proxy as the key management handle

```mermaid
sequenceDiagram
    participant App as Application
    participant Sender as RtpSenderBase
    participant Observer as SframeEnablementDelegate
    participant Encrypter as SframeEncrypterImpl
    participant Proxy as SframeEncrypterProxy
    participant Pipeline as Send Pipeline

    App->>Sender: CreateSframeEncrypterOrError(options)

    Sender->>Observer: TryToEnableSframe()
    Note over Observer: Mark SFrame as enabled
    Observer->>App: Trigger onnegotiationneeded
    Observer-->>Sender: RTCError::OK()

    Sender->>Encrypter: Create SframeEncrypterImpl(options)
    Encrypter-->>Sender: encrypter instance

    Sender->>Pipeline: Install SFrame encrypter
    Note over Pipeline: Encrypter installed for encryption
    Pipeline-->>Sender: installed

    Sender->>Proxy: Wrap encrypter as SframeEncrypterProxy
    Note over Proxy: Thread-safe wrapper for key management
    Proxy-->>Sender: proxy instance

    Sender-->>App: RTCErrorOr<scoped_refptr<SframeEncrypterInterface>>(proxy)
```

### CreateSframeDecrypterOrError Internal Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant Receiver as RtpReceiverBase
    participant Observer as SframeEnablementDelegate
    participant Decrypter as SframeDecrypterImpl
    participant Proxy as SframeDecrypterProxy
    participant Pipeline as Receive Pipeline

    App->>Receiver: CreateSframeDecrypterOrError(options)

    Receiver->>Observer: TryToEnableSframe()
    Note over Observer: Mark SFrame as enabled
    Observer->>App: Trigger onnegotiationneeded
    Observer-->>Receiver: RTCError::OK()

    Receiver->>Decrypter: Create SframeDecrypterImpl(options)
    Decrypter-->>Receiver: decrypter instance

    Receiver->>Pipeline: Install SFrame decrypter
    Note over Pipeline: Decrypter installed for decryption
    Pipeline-->>Receiver: installed

    Receiver->>Proxy: Wrap decrypter as SframeDecrypterProxy
    Note over Proxy: Thread-safe wrapper for key management
    Proxy-->>Receiver: proxy instance

    Receiver-->>App: RTCErrorOr<scoped_refptr<SframeDecrypterInterface>>(proxy)
```

## Encrypter/Decrypter Propagation

This section describes how the internally created `SframeEncrypterImpl` and `SframeDecrypterImpl` are propagated from the RtpSender/RtpReceiver down to the actual media pipeline actors that perform encryption and decryption.

### Design Principle

The application never sees or touches the internal encrypter/decrypter implementation. Instead:

1. **RtpSender/RtpReceiver creates** the internal `SframeEncrypterImpl`/`SframeDecrypterImpl`
2. **RtpSender/RtpReceiver installs** it into the media send/receive pipeline on the worker thread
3. **RtpSender/RtpReceiver returns** a key management handle (the `SframeEncrypterInterface`/`SframeDecrypterInterface` ref-counted proxy) to the application

The application only interacts with the handle for key management. The actual encryption/decryption happens internally in the media pipeline, invisible to the caller.

### Encrypter Propagation (RtpSender → Send Pipeline)

When `CreateSframeEncrypterOrError` is called on the RtpSender, the encrypter is created and pushed down to the send channel where it intercepts frames or packets depending on the configured mode.

```mermaid
flowchart TD
    subgraph API Layer
        App[Application]
        SenderAPI[RtpSender]
    end

    subgraph Internal - Signaling Thread
        SenderAPI -->|1. CreateSframeEncrypterOrError| Create[Create SframeEncrypterImpl]
        Create -->|2. Notify| Delegate[SframeEnablementDelegate\nRtpTransceiver]
    end

    subgraph Internal - Worker Thread
        Create -->|3. BlockingCall| Install[Install encrypter into\nMediaSendChannel]
        Install --> SendChannel[MediaSendChannelInterface]

        subgraph Video Path
            SendChannel --> VSS[VideoSendStream]
            VSS --> VEncoder[Video Encoder]
            VEncoder -->|Encoded frame| EncryptPoint_V[SframeEncrypterImpl\nEncrypt frame/packet]
            EncryptPoint_V --> Packetizer[Packetizer]
            Packetizer --> Network_V[Network]
        end

        subgraph Audio Path
            SendChannel --> ASS[AudioSendStream]
            ASS --> AEncoder[Audio Encoder]
            AEncoder -->|Encoded frame| EncryptPoint_A[SframeEncrypterImpl\nEncrypt frame/packet]
            EncryptPoint_A --> RtpPacketizer[RTP Packetizer]
            RtpPacketizer --> Network_A[Network]
        end
    end

    subgraph Returned to App
        Create -->|4. Return handle| Handle[SframeEncrypterInterface\nkey management proxy]
        Handle -->|SetEncryptionKey| EncryptPoint_V
        Handle -->|SetEncryptionKey| EncryptPoint_A
    end

    App -->|Calls| SenderAPI
    App -->|Uses handle for keys| Handle

    style EncryptPoint_V fill:#f96,stroke:#333
    style EncryptPoint_A fill:#f96,stroke:#333
    style Handle fill:#6f9,stroke:#333
```

**Step-by-step:**

| Step | Thread | Action |
|------|--------|--------|
| 1 | Signaling | App calls `sender->CreateSframeEncrypterOrError(init)` |
| 2 | Signaling | RtpSender notifies transceiver via `SframeEnablementDelegate::TryToEnableSframe()` |
| 3 | Worker (BlockingCall) | RtpSender creates `SframeEncrypterImpl` and installs it into the `MediaSendChannelInterface` |
| 4 | Signaling | RtpSender wraps the impl in a thread-safe proxy and returns the handle to the app |

After installation, the `SframeEncrypterImpl` lives on the worker thread and is invoked by the send pipeline at the appropriate point (before packetization for per-frame mode, after packetization for per-packet mode). The application only interacts with it through the returned proxy handle for key management.

### Decrypter Propagation (RtpReceiver → Receive Pipeline)

When `CreateSframeDecrypterOrError` is called on the RtpReceiver, the decrypter is created and pushed down to the receive channel where it intercepts packets or frames depending on the mode signaled in the SFrame header.

```mermaid
flowchart TD
    subgraph API Layer
        App[Application]
        ReceiverAPI[RtpReceiver]
    end

    subgraph Internal - Signaling Thread
        ReceiverAPI -->|1. CreateSframeDecrypterOrError| Create[Create SframeDecrypterImpl]
        Create -->|2. Notify| Delegate[SframeEnablementDelegate\nRtpTransceiver]
    end

    subgraph Internal - Worker Thread
        Create -->|3. BlockingCall| Install[Install decrypter into\nMediaReceiveChannel]
        Install --> RecvChannel[MediaReceiveChannelInterface]

        subgraph Video Path
            Network_V[Network] --> VRS[VideoReceiveStream]
            VRS -->|RTP packets| DecryptPoint_V[SframeDecrypterImpl\nDecrypt packet/frame]
            DecryptPoint_V --> Depacketizer[Depacketizer]
            Depacketizer --> VDecoder[Video Decoder]
        end

        subgraph Audio Path
            Network_A[Network] --> ARS[AudioReceiveStream]
            ARS -->|RTP packets| DecryptPoint_A[SframeDecrypterImpl\nDecrypt packet/frame]
            DecryptPoint_A --> AudioDepacketizer[Audio Depacketizer]
            AudioDepacketizer --> ADecoder[Audio Decoder]
        end
    end

    subgraph Returned to App
        Create -->|4. Return handle| Handle[SframeDecrypterInterface\nkey management proxy]
        Handle -->|AddDecryptionKey\nRemoveDecryptionKey| DecryptPoint_V
        Handle -->|AddDecryptionKey\nRemoveDecryptionKey| DecryptPoint_A
    end

    App -->|Calls| ReceiverAPI
    App -->|Uses handle for keys| Handle

    style DecryptPoint_V fill:#69f,stroke:#333
    style DecryptPoint_A fill:#69f,stroke:#333
    style Handle fill:#6f9,stroke:#333
```

**Step-by-step:**

| Step | Thread | Action |
|------|--------|--------|
| 1 | Signaling | App calls `receiver->CreateSframeDecrypterOrError(init)` |
| 2 | Signaling | RtpReceiver notifies transceiver via `SframeEnablementDelegate::TryToEnableSframe()` |
| 3 | Worker (BlockingCall) | RtpReceiver creates `SframeDecrypterImpl` and installs it into the `MediaReceiveChannelInterface` |
| 4 | Signaling | RtpReceiver wraps the impl in a thread-safe proxy and returns the handle to the app |

After installation, the `SframeDecrypterImpl` lives on the worker thread and is invoked by the receive pipeline at the appropriate point (before depacketization for per-packet mode, after depacketization for per-frame mode). The application only interacts with it through the returned proxy handle for key management.

### Ownership Model

```mermaid
flowchart LR
    subgraph Ownership
        direction TB
        Impl[SframeEncrypterImpl / SframeDecrypterImpl\nscoped_refptr - shared ownership]
        Pipeline[MediaSendChannel / MediaReceiveChannel\nholds scoped_refptr]
        Proxy[Proxy returned to App\nholds scoped_refptr]
    end

    Impl --- Pipeline
    Impl --- Proxy

    style Impl fill:#ffd,stroke:#333
```

The internal impl is ref-counted (`RefCountInterface`). Two parties hold references:
- **The media channel** (worker thread) — uses the impl for actual encryption/decryption
- **The proxy handle** (returned to the app) — uses the impl for key management calls, marshaled to the worker thread

This ensures the impl stays alive as long as either the pipeline or the application needs it. When both release their references, the impl is destroyed.

## Key Management

### Thread Safety

Key management calls on the returned `SframeEncrypterInterface` and `SframeDecrypterInterface` handles are thread-safe. The proxy pattern ensures all calls are marshaled to the worker thread:

- **SframeEncrypterProxy**: Wraps `SframeEncrypterImpl`, marshals `SetEncryptionKey` to worker thread
- **SframeDecrypterProxy**: Wraps `SframeDecrypterImpl`, marshals `AddDecryptionKey`/`RemoveDecryptionKey` to worker thread

The encrypter/decrypter itself always runs on the worker thread, ensuring single-threaded access to encryption/decryption state.

### Sender Key Management Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant Proxy as SframeEncrypterProxy (returned handle)
    participant Encrypter as SframeEncrypterImpl

    App->>Proxy: SetEncryptionKey(key_id, key_material)
    
    Note over Proxy: Calling Thread - Post task to worker thread
    Proxy->>Proxy: PostTask to Worker Thread
    
    Note over Encrypter: Worker Thread - Execute key update
    Proxy->>+Encrypter: SetEncryptionKey(key_id, key_material)
    Encrypter-->>-Proxy: RTCError::OK()
    
    Note over Proxy: Calling Thread - Return result
    Proxy-->>App: RTCError::OK()
    
    Note over App,Encrypter: ✓ Key is now set for encryption
```

### Receiver Key Management Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant Proxy as SframeDecrypterProxy (returned handle)
    participant Decrypter as SframeDecrypterImpl

    App->>Proxy: AddDecryptionKey(key_id, key_material)
    
    Note over Proxy: Calling Thread - Post task to worker thread
    Proxy->>Proxy: PostTask to Worker Thread
    
    Note over Decrypter: Worker Thread - Execute add key
    Proxy->>+Decrypter: AddDecryptionKey(key_id, key_material)
    Decrypter-->>-Proxy: RTCError::OK()
    
    Note over Proxy: Calling Thread - Return result
    Proxy-->>App: RTCError::OK()
    
    Note over App,Decrypter: ✓ New key available for decryption
    
    Note over App: Later, during key rotation...
    
    App->>Proxy: RemoveDecryptionKey(old_key_id)
    
    Note over Proxy: Calling Thread - Post task to worker thread
    Proxy->>Proxy: PostTask to Worker Thread
    
    Note over Decrypter: Worker Thread - Execute remove key
    Proxy->>+Decrypter: RemoveDecryptionKey(old_key_id)
    Decrypter-->>-Proxy: RTCError::OK()
    
    Proxy-->>App: RTCError::OK()
    
    Note over App,Decrypter: ✓ Old key removed, only new key remains
```

## Media Processing Flows

### RtpSender Frame Processing Flow - Video

This flow describes how encoded video frames are processed through the RTP sender pipeline with SFrame encryption support. The pipeline handles both frame-level and packet-level encryption modes, with the `SframeEncrypterImpl` invoked directly at the appropriate stage.

**Flow Overview:**

1. **Frame Reception**: The video encoder produces an encoded frame and forwards it to the RTP sender for transmission.

2. **SFrame Configuration Validation**: The sender validates the SFrame configuration:
   - If SFrame is enabled, it checks whether the `SframeEncrypterImpl` has been installed and has a key set
   - If SFrame is enabled but the encrypter is not ready, the frame is dropped
   - If SFrame is not enabled, processing continues normally without encryption

3. **Frame Encryption** (per-frame mode only): When SFrame per-frame mode is enabled, the `SframeEncrypterImpl` encrypts the complete encoded frame before packetization:
   - The frame payload is encrypted and the SFrame header is prepended
   - If SFrame is not enabled or per-packet mode is enabled, this step is skipped

4. **Packetization**: The sender determines the appropriate packetizer based on the enabled SFrame mode:
   - If SFrame per-frame mode is enabled, an SFrame-aware packetizer is used that creates RTP packets with SFrame-specific headers (the frame payload is already encrypted at this point)
   - If SFrame per-packet mode is enabled or SFrame is not enabled, a codec-specific packetizer is used for standard RTP packet creation

5. **Packet Encryption** (per-packet mode only): When SFrame per-packet mode is enabled, the `SframeEncrypterImpl` encrypts each RTP packet individually:
   - Each packet payload is encrypted and the SFrame header is prepended
   - This occurs after packetization, encrypting individual packets
   - If SFrame is not enabled or per-frame mode is enabled, this step is skipped

6. **Repacketization** (per-packet mode only): SFrame-specific RTP headers are inserted into the encrypted packets.

```mermaid
sequenceDiagram
    participant Encoder as Video Encoder
    participant Sender as RtpSender
    participant Encrypter as SframeEncrypterImpl
    participant Packetizer as Packetizer
    participant Network as Network

    Encoder->>Sender: 1. EncodedFrame received
    
    Note over Sender: STEP 2: Validate SFrame Configuration

    alt SFrame Enabled
        Sender->>Sender: Check encrypter availability and key status
        alt Encrypter not ready
            Sender-->>Encoder: ❌ return (drop frame)
        end
    else SFrame Not Enabled
        Note over Sender: Continue normal processing
    end
    
    Note over Sender,Encrypter: STEP 3: Frame Encryption (per-frame mode)
    
    alt SFrame per-frame mode enabled
        Sender->>Encrypter: EncryptFrame(frame)
        Encrypter->>Encrypter: Encrypt payload, prepend SFrame header
        Encrypter-->>Sender: Encrypted frame
    else
        Note over Sender: Skip frame encryption
    end
    
    Note over Sender,Packetizer: STEP 4: Packetization
    
    Sender->>Sender: Check SFrame mode
    
    alt SFrame per-frame mode enabled
        Note over Sender,Packetizer: Use SFrame-aware packetizer
        
        Sender->>Packetizer: Packetize with SFrame Packetizer
        Packetizer-->>Sender: SFrame RTP packets
    else SFrame per-packet mode or no SFrame
        Note over Sender,Packetizer: Use codec specific packetizer
        
        Sender->>Packetizer: Codec Specific packetization
        Packetizer-->>Sender: Codec specific RTP packets
    end
    
    Note over Sender,Encrypter: STEP 5: Packet Encryption (per-packet mode)
    
    alt SFrame per-packet mode enabled
        loop Each RTP Packet
            Sender->>Encrypter: EncryptPacket(packet)
            Encrypter->>Encrypter: Encrypt payload, prepend SFrame header
            Encrypter-->>Sender: Encrypted packet
        end

        Note over Sender: STEP 6: Repacketization
        loop Each Encrypted Packet
            Sender->>Sender: Insert SFrame-specific RTP headers
        end
        
    else
        Note over Sender: Skip packet encryption
    end
    
    Note over Sender,Network: STEP 7: Network Transmission
    Sender->>Network: Send packets
    
    Note over Encoder,Network: ✓ Processing complete - Frame encrypted and sent
```

### RtpReceiver Frame Processing Flow - Video

This flow describes how received RTP packets are processed through the video receiver pipeline with SFrame decryption support. The pipeline handles both packet-level and frame-level decryption modes, with the `SframeDecrypterImpl` invoked directly at the appropriate stage.

**Flow Overview:**

1. **Packet Reception**: RTP packets arrive from the network transport and are received by the VideoRtpReceiver.

2. **SFrame Configuration Validation**: The receiver validates the SFrame configuration:
   - If SFrame is enabled, it checks whether the `SframeDecrypterImpl` has been installed and has keys available
   - If SFrame is enabled but the decrypter is not ready, the packets are dropped
   - If SFrame is not enabled, processing continues normally without decryption

3. **SFrame Header Removal** (per-packet mode only): When SFrame per-packet mode is enabled, SFrame-specific RTP headers are stripped from each packet before decryption.

4. **Packet Decryption** (per-packet mode only): When SFrame per-packet mode is enabled, the `SframeDecrypterImpl` decrypts each RTP packet individually:
   - The SFrame header is parsed and the packet payload is decrypted
   - This occurs before depacketization, working on individual encrypted packets
   - If SFrame is not enabled or per-frame mode is enabled, this step is skipped

5. **Depacketization**: The receiver determines the appropriate depacketizer based on the enabled SFrame mode:
   - If SFrame per-frame mode is enabled, an SFrame depacketizer is used to reconstruct the encrypted frame from SFrame-formatted packets
   - If SFrame per-packet mode is enabled or SFrame is not enabled, a codec-specific depacketizer reconstructs the frame using standard media depacketization
   - The depacketizer assembles RTP packets into complete video frames

6. **Frame Decryption** (per-frame mode only): When SFrame per-frame mode is enabled, the `SframeDecrypterImpl` decrypts the complete assembled frame:
   - The SFrame header is parsed and the frame payload is decrypted
   - This occurs after depacketization, working on the reassembled encrypted frame
   - If SFrame is not enabled or per-packet mode is enabled, this step is skipped

7. **Decoding**: The processed (and potentially decrypted) frame is forwarded to the video decoder for final decoding into displayable video.

The flow supports three operational modes:
- **Per-frame mode**: SFrame depacketizer reconstructs encrypted frame, `SframeDecrypterImpl` decrypts the complete frame
- **Per-packet mode**: `SframeDecrypterImpl` decrypts individual packets, standard depacketizer reconstructs the frame
- **No SFrame**: Standard RTP packet reception, depacketization, and decoding without decryption

This receiver flow is the inverse of the sender flow, ensuring that encrypted frames can be properly reconstructed and decrypted regardless of whether frame-level or packet-level encryption was used.

```mermaid
sequenceDiagram
    participant Network as Network Transport
    participant Receiver as VideoRtpReceiver
    participant Decrypter as SframeDecrypterImpl
    participant Depacketizer as Depacketizer
    participant Decoder as Video Decoder

    Network->>Receiver: 1. Receive RTP Packets
    
    Note over Receiver: STEP 2: Validate SFrame Configuration

    alt SFrame Enabled
        Receiver->>Receiver: Check decrypter availability and key status
        alt Decrypter not ready
            Receiver-->>Network: ❌ return (drop packets)
        end
    else SFrame Not Enabled
        Note over Receiver: Continue normal processing
    end
    
    Note over Receiver,Decrypter: STEP 3: SFrame Header Removal (per-packet mode)
    
    alt SFrame per-packet mode enabled
        loop Each RTP Packet
            Receiver->>Receiver: Remove SFrame-specific RTP headers
        end
    end
    
    Note over Receiver,Decrypter: STEP 4: Packet Decryption (per-packet mode)
    
    alt SFrame per-packet mode enabled
        loop Each RTP Packet
            Receiver->>Decrypter: DecryptPacket(packet)
            Decrypter->>Decrypter: Parse SFrame header, decrypt payload
            Decrypter-->>Receiver: Decrypted packet
        end
    else
        Note over Receiver: Skip packet decryption
    end
    
    Note over Receiver,Depacketizer: STEP 5: Depacketization
    
    alt SFrame per-frame mode enabled
        Note over Receiver,Depacketizer: Use SFrame depacketizer
        
        Receiver->>Depacketizer: Depacketize with SFrame Depacketizer
        Depacketizer-->>Receiver: Assembled SFrame encrypted frame
    else SFrame per-packet mode or no SFrame
        Note over Receiver,Depacketizer: Use codec specific depacketizer
        
        Receiver->>Depacketizer: Codec Specific depacketization
        Depacketizer-->>Receiver: Assembled codec specific frame
    end
    
    Note over Receiver,Decrypter: STEP 6: Frame Decryption (per-frame mode)
    
    alt SFrame per-frame mode enabled
        Receiver->>Decrypter: DecryptFrame(frame)
        Decrypter->>Decrypter: Parse SFrame header, decrypt payload
        Decrypter-->>Receiver: Decrypted frame
    else
        Note over Receiver: Skip frame decryption
    end
    
    Note over Receiver,Decoder: STEP 7: Decoding
    Receiver->>Decoder: Send frame to decoder
    
    Note over Network,Decoder: ✓ Processing complete - Frame decrypted and decoded
```

