# WebRTC SFrame Integration Architecture

This document describes the architectural design for integrating SFrame (Secure Frame) encryption into WebRTC applications. SFrame provides end-to-end media security that works even when media flows through untrusted servers or intermediaries.

## Overview

SFrame encryption can be applied at two different levels:
- **Per-frame**: Encrypts complete video/audio frames before packetization
- **Per-packet**: Encrypts individual RTP packets after packetization

Both approaches offer strong security guarantees, with per-packet providing finer granularity and per-frame offering better performance characteristics.

## Architecture Overview

```mermaid
graph TD
    subgraph "User Layer"
        A[WebRTC Application]
        K[Key Management]
    end
    
    subgraph "SFrame API"
        F[Transform Factory]
        O[Transform Options]
    end
    
    subgraph "WebRTC Interfaces"
        T[RTP Transceiver]
        S[RTP Sender]
        R[RTP Receiver]
    end
    
    subgraph "Transform Implementations"
        FT[Frame Transformer]
        PT[Packet Transformer]
        SE[SFrame Encrypter]
        SD[SFrame Decrypter]
    end
    
    subgraph "Media Processing"
        ENC[Encoder]
        PAC[Packetizer]
        NET[Network]
        DEPAC[Depacketizer] 
        DEC[Decoder]
    end
    
    %% User interactions
    A --> F
    A --> K
    A --> T
    
    %% Factory creates transformers
    F --> SE
    F --> SD
    F --> O
    
    %% Transceiver manages sender/receiver
    T --> S
    T --> R
    
    %% Transformers attach to RTP interfaces
    S --> FT
    S --> PT
    R --> FT
    R --> PT
    
    %% SFrame implements transformers
    SE -.-> FT
    SD -.-> FT
    
    %% Media flow (frame-level)
    ENC --> FT
    FT --> PAC
    PAC --> NET
    NET --> DEPAC
    DEPAC --> DEC
    
    %% Key management
    K --> SE
    K --> SD
    
    style A fill:#e1f5fe
    style F fill:#f3e5f5
    style T fill:#e8f5e8
    style FT fill:#fff3e0
    style SE fill:#ffebee
    style SD fill:#ffebee
```

## Core Interface Architecture

### Transformation Features Hierarchy

```mermaid
classDiagram
    class TransformationFeatures {
        <<interface>>
        +UseSFrame() bool
        #~TransformationFeatures()
    }
    
    class FrameTransformerInterface {
        <<interface>>
        +Transform(frame)
        +RegisterTransformedFrameCallback(callback)
        +RegisterTransformedFrameSinkCallback(callback, ssrc)
        +UnregisterTransformedFrameCallback()
        +UnregisterTransformedFrameSinkCallback(ssrc)
        #~FrameTransformerInterface()
    }
    
    class PacketTransformerInterface {
        <<interface>>
        +Transform(frame)
        +GetReservedNumberOfBytes() size_t
        +RegisterTransformedPacketCallback(callback)
        +RegisterTransformedPacketSinkCallback(callback, ssrc)
        +UnregisterTransformedPacketCallback()
        +UnregisterTransformedPacketSinkCallback(ssrc)
        #~PacketTransformerInterface()
    }
    
    TransformationFeatures <|-- FrameTransformerInterface
    TransformationFeatures <|-- PacketTransformerInterface
```

> **Note**: The `GetReservedNumberOfBytes()` method in `PacketTransformerInterface` ensures that the packetizer reserves enough space for transformer data to avoid MTU overflow. This is critical for packet-level transformations where additional encryption overhead needs to be accommodated within network packet size constraints.

### RTP Interface Integration

```mermaid
classDiagram
    class FrameTransformerHost {
        <<interface>>
        +SetFrameTransformer(transformer)
        +SetPacketTransformer(transformer)
        +~FrameTransformerHost()
    }
    
    class RtpSenderInterface {
        <<interface>>
        +SetFrameTransformer(transformer)
        +SetPacketTransformer(transformer)
    }
    
    class RtpReceiverInterface {
        <<interface>>
        +SetFrameTransformer(transformer)
        +SetPacketTransformer(transformer)
    }
    
    FrameTransformerHost <|-- RtpSenderInterface
    FrameTransformerHost <|-- RtpReceiverInterface
```

### SFrame Management Interfaces

```mermaid
classDiagram
    class SFrameEncrypterInterface {
        <<interface>>
        +SetEncryptionKey(key, key_id) bool
    }
    
    class SFrameDecrypterInterface {
        <<interface>>
        +AddDecryptionKey(key, key_id) bool
        +RemoveDecryptionKey(key_id) bool
    }
```

> **Note**: These interfaces provide key management capabilities for SFrame transformers. The encrypter interface manages a single encryption key, while the decrypter interface can manage multiple decryption keys simultaneously to handle key rotation scenarios.

For each of `SFrameEncrypterInterface` and `SFrameDecrypterInterface` proxy will be defined.
To ensure that calls of `SetEncryptionKey` and `AddDecryptionKey`/`RemoveDecryptionKey` will be delegated to valid thread.


## Configuration and Options

```mermaid
classDiagram
    class SFrameMode {
        <<enumeration>>
        kFrame
        kPacket
    }
    
    class SFrameCipherSuite {
        <<enumeration>>
        kAES_128_CTR_HMAC_SHA256_80
        kAES_128_CTR_HMAC_SHA256_64
        kAES_128_CTR_HMAC_SHA256_32
        kAES_128_GCM_SHA256_128
        kAES_256_GCM_SHA512_128
    }
    
    class SFrameTransformOptions {
        +cipher_suite: SFrameCipherSuite
        +sframe_mode: SFrameMode
    }
    
    SFrameTransformOptions --> SFrameMode
    SFrameTransformOptions --> SFrameCipherSuite
```

## Core objects

### SFrameSenderFrameTransformer

The `SFrameSenderFrameTransformer` provides frame-level SFrame encryption that operates on frames.

```mermaid
classDiagram
    class TransformationFeatures {
        <<interface>>
        +UseSFrame() bool
        #~TransformationFeatures()
    }
    
    class FrameTransformerInterface {
        <<interface>>
        +Transform(frame)
        +RegisterTransformedFrameCallback(callback)
        +RegisterTransformedFrameSinkCallback(callback, ssrc)
        +UnregisterTransformedFrameCallback()
        +UnregisterTransformedFrameSinkCallback(ssrc)
        #~FrameTransformerInterface()
    }
    
    class SFrameEncrypterInterface {
        <<interface>>
        +SetEncryptionKey(key, key_id) bool
    }
    
    class SFrameSenderFrameTransformer {
        +SFrameSenderFrameTransformer(options)
        +Transform(frame) override
        +SetEncryptionKey(key, key_id) bool override
        +UseSFrame() bool override
        +RegisterTransformedFrameCallback(callback) override
        +UnregisterTransformedFrameCallback() override
    }
    
    TransformationFeatures <|-- FrameTransformerInterface
    FrameTransformerInterface <|-- SFrameSenderFrameTransformer
    SFrameEncrypterInterface <|-- SFrameSenderFrameTransformer
```

### SFrameSenderPacketTransformer

The `SFrameSenderPacketTransformer` provides packet-level SFrame encryption that operates on individual RTP packets.

```mermaid
classDiagram
    class TransformationFeatures {
        <<interface>>
        +UseSFrame() bool
        #~TransformationFeatures()
    }
    
    class PacketTransformerInterface {
        <<interface>>
        +Transform(packet)
        +GetReservedNumberOfBytes() size_t
        +RegisterTransformedPacketCallback(callback)
        +RegisterTransformedPacketSinkCallback(callback, ssrc)
        +UnregisterTransformedPacketCallback()
        +UnregisterTransformedPacketSinkCallback(ssrc)
        #~PacketTransformerInterface()
    }
    
    class SFrameEncrypterInterface {
        <<interface>>
        +SetEncryptionKey(key, key_id) bool
    }
    
    class SFrameSenderPacketTransformer {
        +SFrameSenderPacketTransformer(options)
        +Transform(packet) override
        +GetReservedNumberOfBytes() size_t override
        +SetEncryptionKey(key, key_id) bool override
        +UseSFrame() bool override
        +RegisterTransformedPacketCallback(callback) override
        +UnregisterTransformedPacketCallback() override
    }
    
    TransformationFeatures <|-- PacketTransformerInterface
    PacketTransformerInterface <|-- SFrameSenderPacketTransformer
    SFrameEncrypterInterface <|-- SFrameSenderPacketTransformer
```

> **MTU Consideration**: The transformer reserves additional bytes for encryption overhead to prevent packet fragmentation. This is critical for maintaining network efficiency and avoiding packet loss due to size constraints.

### SFrameSenderTransform

```mermaid
classDiagram
    class SFrameSenderFrameTransformer {
        +SFrameSenderFrameTransformer(options)
        +SetEncryptionKey(key, key_id) bool
        +Transform(frame)
    }

    class SFrameSenderPacketTransformer {
      +SFrameSenderPacketTransformer(options)
      +SetEncryptionKey(key, key_id) bool
      +Transform(packet)
    }
    
    class SFrameSenderTransform {
        -sender: RtpSenderInterface*
        -worker_thread: Thread*
        -transformer: SFrameEncrypterInterface* (proxy)
        +SFrameSenderTransform(options, sender, thread)
        +SetEncryptionKey(key, key_id) bool
    }
    
    SFrameSenderTransform --> SFrameEncrypterProxy : transformer_
```

The `SFrameSenderTransform` class serves as the main orchestrator for sender-side SFrame encryption. 
The key architectural pattern is that the proxy wraps the transformer for thread safety:

**Architecture Responsibilities:**

- **SFrameSenderFrameTransformer**: The actual encryption implementation that transforms frames
- **SFrameSenderPacketTransformer**: The actual encryption implementation that transforms packets
- **SFrameEncrypterProxy**: Thread-safe wrapper that marshals calls to the worker thread
- **SFrameSenderTransform**: High-level orchestrator that manages the proxy-wrapped transformer

**Key Design Elements:**

- **Proxy Wrapping**: The proxy wraps the `SFrameSenderFrameTransformer`/`SFrameSenderPacketTransformer` instance for thread safety
- **Thread Marshaling**: All `SetEncryptionKey()` calls are automatically routed to the worker thread

**Initialization Flow:**
1. `SFrameSenderTransform` constructor receives options, sender, and worker thread
2. Creates `SFrameSenderFrameTransformer`or `SFrameSenderPacketTransformer` instance with encryption based on the provided sframe mode.
3. Sets created transformer to corresponding transformation slot (`SetFrameTransformer`/`SetPacketTransformer`).
4. Wraps transformer in `SFrameEncrypterProxy` for thread safety.
5. Stores the proxy as `transformer_`.

> **Thread Safety**: The proxy pattern ensures all key management operations are thread-safe by automatically marshaling calls from any thread to the designated worker thread, while maintaining the same interface as the underlying transformer.
Transformer itself will always work on the worker thread, which will ensure that tranformer will get modified only from that one thread.

### SFrameReceiverFrameTransformer

The `SFrameReceiverFrameTransformer` provides frame-level SFrame decryption that operates on encrypted frames.

```mermaid
classDiagram
    class TransformationFeatures {
        <<interface>>
        +UseSFrame() bool
        #~TransformationFeatures()
    }
    
    class FrameTransformerInterface {
        <<interface>>
        +Transform(frame)
        +RegisterTransformedFrameCallback(callback)
        +RegisterTransformedFrameSinkCallback(callback, ssrc)
        +UnregisterTransformedFrameCallback()
        +UnregisterTransformedFrameSinkCallback(ssrc)
        #~FrameTransformerInterface()
    }
    
    class SFrameDecrypterInterface {
        <<interface>>
        +AddDecryptionKey(key, key_id) bool
        +RemoveDecryptionKey(key_id) bool
    }
    
    class SFrameReceiverFrameTransformer {
        +SFrameReceiverFrameTransformer(options)
        +Transform(frame) override
        +AddDecryptionKey(key, key_id) bool override
        +RemoveDecryptionKey(key_id) bool override
        +UseSFrame() bool override
        +RegisterTransformedFrameCallback(callback) override
        +UnregisterTransformedFrameCallback() override
    }
    
    TransformationFeatures <|-- FrameTransformerInterface
    FrameTransformerInterface <|-- SFrameReceiverFrameTransformer
    SFrameDecrypterInterface <|-- SFrameReceiverFrameTransformer
```

### SFrameReceiverPacketTransformer

The `SFrameReceiverPacketTransformer` provides packet-level SFrame decryption that operates on individual encrypted RTP packets.

```mermaid
classDiagram
    class TransformationFeatures {
        <<interface>>
        +UseSFrame() bool
        #~TransformationFeatures()
    }
    
    class PacketTransformerInterface {
        <<interface>>
        +Transform(packet)
        +GetReservedNumberOfBytes() size_t
        +RegisterTransformedPacketCallback(callback)
        +RegisterTransformedPacketSinkCallback(callback, ssrc)
        +UnregisterTransformedPacketCallback()
        +UnregisterTransformedPacketSinkCallback(ssrc)
        #~PacketTransformerInterface()
    }
    
    class SFrameDecrypterInterface {
        <<interface>>
        +AddDecryptionKey(key, key_id) bool
        +RemoveDecryptionKey(key_id) bool
    }
    
    class SFrameReceiverPacketTransformer {
        +SFrameReceiverPacketTransformer(options)
        +Transform(packet) override
        +GetReservedNumberOfBytes() size_t override
        +AddDecryptionKey(key, key_id) bool override
        +RemoveDecryptionKey(key_id) bool override
        +UseSFrame() bool override
        +RegisterTransformedPacketCallback(callback) override
        +UnregisterTransformedPacketCallback() override
    }
    
    TransformationFeatures <|-- PacketTransformerInterface
    PacketTransformerInterface <|-- SFrameReceiverPacketTransformer
    SFrameDecrypterInterface <|-- SFrameReceiverPacketTransformer
```

> **Key Management**: The receiver transformers can manage multiple decryption keys simultaneously to handle key rotation scenarios, where new keys are added before old keys are removed to ensure seamless decryption during key transitions.

### SFrameReceiverTransform

```mermaid
classDiagram
    class SFrameReceiverFrameTransformer {
        +SFrameReceiverFrameTransformer(options)
        +AddDecryptionKey(key, key_id) bool
        +RemoveDecryptionKey(key_id) bool
        +Transform(frame)
    }

    class SFrameReceiverPacketTransformer {
      +SFrameReceiverPacketTransformer(options)
      +AddDecryptionKey(key, key_id) bool
      +RemoveDecryptionKey(key_id) bool
      +Transform(packet)
    }
    
    class SFrameReceiverTransform {
        -receiver: RtpReceiverInterface*
        -worker_thread: Thread*
        -transformer: SFrameDecrypterInterface* (proxy)
        +SFrameReceiverTransform(options, receiver, thread)
        +AddDecryptionKey(key, key_id) bool
        +RemoveDecryptionKey(key_id) bool
    }
    
    SFrameReceiverTransform --> SFrameDecrypterProxy : transformer_
```

The `SFrameReceiverTransform` class serves as the main orchestrator for receiver-side SFrame decryption. The key architectural pattern is that the proxy wraps the transformer for thread safety:

**Architecture Responsibilities:**

- **SFrameReceiverFrameTransformer**: The actual decryption implementation that transforms encrypted frames
- **SFrameReceiverPacketTransformer**: The actual decryption implementation that transforms encrypted packets
- **SFrameDecrypterProxy**: Thread-safe wrapper that marshals calls to the worker thread
- **SFrameReceiverTransform**: High-level orchestrator that manages the proxy-wrapped transformer

**Key Design Elements:**

- **Proxy Wrapping**: The proxy wraps the `SFrameReceiverFrameTransformer`/`SFrameReceiverPacketTransformer` instance for thread safety
- **Thread Marshaling**: All `AddDecryptionKey()`/`RemoveDecryptionKey()` calls are automatically routed to the worker thread
- **Multi-Key Support**: Can manage multiple decryption keys simultaneously for seamless key rotation

**Initialization Flow:**
1. `SFrameReceiverTransform` constructor receives options, receiver, and worker thread
2. Creates `SFrameReceiverFrameTransformer` or `SFrameReceiverPacketTransformer` instance based on the provided sframe mode
3. Sets created transformer to corresponding transformation slot (`SetFrameTransformer`/`SetPacketTransformer`)
4. Wraps transformer in `SFrameDecrypterProxy` for thread safety
5. Stores the proxy as `transformer_`

> **Thread Safety**: The proxy pattern ensures all key management operations are thread-safe by automatically marshaling calls from any thread to the designated worker thread. The transformer itself always works on the worker thread, ensuring single-threaded access to decryption state.

## Media Pipeline Integration

### Key Management Architecture

### Sender Key Management Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant SenderTransform as SFrameSenderTransform
    participant Proxy as SFrameEncrypterProxy
    participant Transformer as SFrameSenderTransformer

    App->>SenderTransform: SetEncryptionKey(key, key_id)
    SenderTransform->>Proxy: SetEncryptionKey(key, key_id)
    
    rect rgb(255, 240, 230)
        Note over Proxy: Calling Thread
        Proxy->>Proxy: PostTask to Worker Thread
    end
    
    rect rgb(230, 255, 230)
        Note over Transformer: Worker Thread
        Proxy->>+Transformer: SetEncryptionKey(key, key_id)
        Transformer-->>-Proxy: return success
    end
    
    rect rgb(255, 240, 230)
        Note over Proxy: Calling Thread
        Proxy-->>SenderTransform: return success
    end
    
    SenderTransform-->>App: return success
    
    Note over App,Transformer: Key is now active for encryption
```

### Receiver Key Management Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant ReceiverTransform as SFrameReceiverTransform
    participant Proxy as SFrameDecrypterProxy
    participant Transformer as SFrameReceiverTransformer

    App->>ReceiverTransform: AddDecryptionKey(key, key_id)
    ReceiverTransform->>Proxy: AddDecryptionKey(key, key_id)
    
    rect rgb(255, 240, 230)
        Note over Proxy: Calling Thread
        Proxy->>Proxy: PostTask to Worker Thread
    end
    
    rect rgb(230, 255, 230)
        Note over Transformer: Worker Thread
        Proxy->>+Transformer: AddDecryptionKey(key, key_id)
        Transformer-->>-Proxy: return success
    end
    
    rect rgb(255, 240, 230)
        Note over Proxy: Calling Thread
        Proxy-->>ReceiverTransform: return success
    end
    
    ReceiverTransform-->>App: return success
    
    Note over App,Transformer: New key available for decryption
    
    Note over App: Later, during key rotation...
    
    App->>ReceiverTransform: RemoveDecryptionKey(old_key_id)
    ReceiverTransform->>Proxy: RemoveDecryptionKey(old_key_id)
    
    rect rgb(255, 240, 230)
        Note over Proxy: Calling Thread
        Proxy->>Proxy: PostTask to Worker Thread
    end
    
    rect rgb(230, 255, 230)
        Note over Transformer: Worker Thread
        Proxy->>+Transformer: RemoveDecryptionKey(old_key_id)
        Transformer-->>-Proxy: return success
    end
    
    rect rgb(255, 240, 230)
        Note over Proxy: Calling Thread
        Proxy-->>ReceiverTransform: return success
    end
    
    ReceiverTransform-->>App: return success
    
    Note over App,Transformer: Old key removed, only new key remains
```

### RtpSender FrameTransformer Installation Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant Sender as RtpSenderBase
    participant Passthrough as Internal Layers (Passthrough)
    participant VideoSender as RTPSenderVideo

    App->>Sender: SetFrameTransformer(transformer)
    
    Note over Sender,VideoSender: Propagation through internal layers
    
    Sender->>Passthrough: SetFrameTransformer(transformer)
    Passthrough->>VideoSender: SetFrameTransformer(transformer)
    
    Note over VideoSender: Install transformer to FrameTransformerDelegate

    VideoSender-->>Passthrough: return success
    Passthrough-->>Sender: return success
    Sender-->>App: return success
    
    Note over VideoSender : Frame transformer is now active in media pipeline
```

### RtpSender PacketTransformer Installation Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant Sender as RtpSenderBase
    participant Passthrough as Internal Layers (Passthrough)
    participant VideoSender as RTPSenderVideo

    App->>Sender: SetPacketTransformer(transformer)
    
    Note over Sender,VideoSender: Propagation through internal layers
    
    Sender->>Passthrough: SetPacketTransformer(transformer)
    Passthrough->>VideoSender: SetPacketTransformer(transformer)
    
    Note over VideoSender: Install transformer in PacketTransformerDelegate
    
    VideoSender-->>Passthrough: return success
    Passthrough-->>Sender: return success
    Sender-->>App: return success
    
    Note over VideoSender: Packet transformer now active in media pipeline
```

