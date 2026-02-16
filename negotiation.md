# SFrame SDP Negotiation

This document describes the SFrame negotiation flow within the WebRTC offer/answer exchange. SFrame support is signaled per media section using the `a=sframe` SDP attribute (per `draft-ietf-avtcore-rtp-sframe`).

## Negotiation Layers

| Layer | Description |
|-------|-------------|
| **Application API** | `RtpTransceiverInterface::SetUseSFrame()` / `UseSFrame()` — the application's intent |
| **Offer/Answer** | SFrame preference is carried per media section during offer/answer creation |
| **SDP Wire Format** | `a=sframe` attribute line present in the m= section when SFrame is enabled |

## SFrame Negotiation Rules

1. **Opt-in only**: SFrame is disabled by default. The application must call `SetUseSFrame()` to enable it.
2. **No downgrade**: Once SFrame has been set, it cannot be re-enabled after being explicitly disabled. Attempting to do so returns `InvalidModification`.
3. **Both sides must agree**: During answer creation, if the offer and answerer disagree on SFrame, `CreateAnswer()` fails with an error. No answer SDP is produced.
4. **Downgrade protection (safety net)**: If a local offer included `a=sframe` but the remote answer does not (e.g. due to SDP munging), the transceiver is stopped. This is a second line of defense — normally `CreateAnswer()` prevents this situation from arising.
5. **Triggers negotiation**: Changing the SFrame state triggers the `onnegotiationneeded` event, initiating a new offer/answer exchange.

---

## SDP Representation

When SFrame is enabled for a media section, the `a=sframe` attribute is added to the corresponding m= block:

```
m=audio 9 UDP/TLS/RTP/SAVPF 111
a=rtpmap:111 opus/48000/2
a=sframe
...

m=video 9 UDP/TLS/RTP/SAVPF 96
a=rtpmap:96 VP8/90000
a=sframe
...
```

The attribute follows `draft-ietf-avtcore-rtp-sframe`:
- **Serialization**: If SFrame is enabled for a media section, `a=sframe` is written to the SDP
- **Parsing**: When `a=sframe` is encountered in a received SDP, SFrame is marked as enabled for that media section

---

## Offer/Answer Negotiation Flows

### Flow 1: Successful SFrame Enablement (Both Sides)

The happy path where both peers enable SFrame and negotiate successfully.

```mermaid
sequenceDiagram
    participant AppA as Application A
    participant PCA as PeerConnection A
    participant SDP as SDP Exchange
    participant PCB as PeerConnection B
    participant AppB as Application B

    Note over AppA,AppB: Both applications enable SFrame before negotiation

    AppA->>PCA: transceiver.SetUseSFrame()
    PCA-->>AppA: onnegotiationneeded

    AppB->>PCB: transceiver.SetUseSFrame()

    Note over AppA,AppB: Offer/Answer Exchange

    AppA->>PCA: CreateOffer()
    PCA-->>AppA: SDP Offer with "a=sframe"

    AppA->>PCA: SetLocalDescription(offer)
    AppA->>SDP: Send offer to Peer B

    SDP->>AppB: Receive offer
    AppB->>PCB: SetRemoteDescription(offer)

    AppB->>PCB: CreateAnswer()
    Note over PCB: Offer wants SFrame ✓, local wants SFrame ✓ → agree
    PCB-->>AppB: SDP Answer with "a=sframe"

    AppB->>PCB: SetLocalDescription(answer)
    AppB->>SDP: Send answer to Peer A

    SDP->>AppA: Receive answer
    AppA->>PCA: SetRemoteDescription(answer)
    Note over PCA: Local offer has a=sframe ✓, answer has a=sframe ✓

    Note over AppA,AppB: ✅ SFrame negotiated successfully on both sides
```

---

### Flow 2: Offerer Enables SFrame, Answerer Does Not — CreateAnswer Fails

The offerer enables SFrame but the answerer does not. `CreateAnswer()` fails on the answerer — no answer is ever produced.

```mermaid
sequenceDiagram
    participant AppA as Application A
    participant PCA as PeerConnection A
    participant SDP as SDP Exchange
    participant PCB as PeerConnection B
    participant AppB as Application B

    AppA->>PCA: transceiver.SetUseSFrame()
    PCA-->>AppA: onnegotiationneeded

    Note over AppB: Application B does NOT enable SFrame

    AppA->>PCA: CreateOffer()
    PCA-->>AppA: SDP Offer with "a=sframe"

    AppA->>PCA: SetLocalDescription(offer)
    AppA->>SDP: Send offer to Peer B

    SDP->>AppB: Receive offer
    AppB->>PCB: SetRemoteDescription(offer)

    AppB->>PCB: CreateAnswer()
    Note over PCB: Offer wants SFrame ✓, local does NOT ✗ → mismatch
    PCB-->>AppB: ❌ CreateAnswer() fails (INTERNAL_ERROR)

    Note over AppB: 🛑 No answer SDP is produced
    Note over AppA,AppB: ❌ Negotiation cannot proceed — peers disagree on SFrame
```

---

### Flow 3: Remote Offer With SFrame, Local Agrees

The remote peer offers SFrame and the local peer also has SFrame enabled.

```mermaid
sequenceDiagram
    participant AppA as Application A (Offerer)
    participant SDP as SDP Exchange
    participant PCB as PeerConnection B (Answerer)
    participant AppB as Application B

    Note over AppB: Application B has already enabled SFrame
    AppB->>PCB: transceiver.SetUseSFrame()

    SDP->>AppB: Receive offer with "a=sframe"
    AppB->>PCB: SetRemoteDescription(offer)

    AppB->>PCB: CreateAnswer()
    Note over PCB: Offer wants SFrame ✓, local wants SFrame ✓ → agree
    PCB-->>AppB: SDP Answer with "a=sframe"

    AppB->>PCB: SetLocalDescription(answer)
    AppB->>SDP: Send answer to Peer A

    Note over AppB: ✅ SFrame agreed — both sides will encrypt
```

---

### Flow 4: Remote Offer With SFrame, Local Does Not Support — CreateAnswer Fails

The remote peer offers SFrame but the local peer has not enabled it. `CreateAnswer()` fails — no answer is produced.

```mermaid
sequenceDiagram
    participant AppA as Application A (Offerer)
    participant SDP as SDP Exchange
    participant PCB as PeerConnection B (Answerer)
    participant AppB as Application B

    Note over AppB: Application B has NOT enabled SFrame

    SDP->>AppB: Receive offer with "a=sframe"
    AppB->>PCB: SetRemoteDescription(offer)

    AppB->>PCB: CreateAnswer()
    Note over PCB: Offer wants SFrame ✓, local does NOT ✗ → mismatch
    PCB-->>AppB: ❌ CreateAnswer() fails (INTERNAL_ERROR)

    Note over AppB: 🛑 No answer SDP is produced
    Note over AppB: ❌ Negotiation cannot proceed — peers disagree on SFrame
```

---

### Flow 5: Answerer Adds `a=sframe` That Was Not In the Offer

This is an abnormal case — a spec-compliant answerer should not introduce `a=sframe` if the offer did not include it. A buggy or malicious remote peer could produce such an answer. The offerer rejects it with an invalid SDP error.

```mermaid
sequenceDiagram
    participant AppA as Application A (Offerer)
    participant PCA as PeerConnection A
    participant SDP as SDP Exchange
    participant PCB as PeerConnection B (Answerer)

    Note over AppA: Offerer did NOT enable SFrame
    Note over PCB: Answerer (buggy/malicious) inserts a=sframe in answer

    AppA->>PCA: CreateOffer()
    PCA-->>AppA: SDP Offer WITHOUT "a=sframe"
    AppA->>PCA: SetLocalDescription(offer)
    AppA->>SDP: Send offer

    SDP->>PCB: Receive offer (no a=sframe)
    PCB-->>SDP: Answer WITH "a=sframe" (abnormal)
    SDP->>AppA: Receive answer

    AppA->>PCA: SetRemoteDescription(answer)
    PCA-->>AppA: ❌ InvalidSdp error

    Note over PCA: 🛑 Answer rejected — a=sframe in answer<br/>but not in the corresponding offer
```

**What happens:**

1. During `SetRemoteDescription(answer)`, the offerer compares each media section in the answer against the local offer.
2. If an answer media section contains `a=sframe` but the corresponding offer media section does not, the answer is rejected with an **invalid SDP** error.
3. The PeerConnection state is not modified — the previous session description remains in effect.

**Rationale:** Silently ignoring the spurious `a=sframe` would lead to a state mismatch: the answerer believes SFrame is active and encrypts its outgoing media, while the offerer has no SFrame decryptor. This causes media failure on the offerer's receive path (undecryptable media). Failing early with a clear error is safer and easier to diagnose.

> **Note:** A well-behaved answerer should never add `a=sframe` if the offer did not include it. This case represents a protocol violation by the answerer.

---

### Summary of Negotiation Outcomes

| Offer `a=sframe` | Answer `a=sframe` | Result |
|:-:|:-:|:--|
| ✅ | ✅ | SFrame active on both sides (Flow 1) |
| ✅ | ❌ | `CreateAnswer()` fails — no answer produced (Flow 2) |
| ❌ | ❌ | No SFrame, normal media flow |
| ❌ | ✅ | **Rejected** — invalid SDP error (Flow 5) |

> **Safety net:** If an answer without `a=sframe` is received despite the offer including it (e.g. due to SDP munging), the offerer stops the transceiver as a downgrade protection measure.

---

## Additional Scenarios

### Negotiation-Needed Triggered by SFrame State Change

When the transceiver's SFrame state differs from what was last negotiated, a new offer/answer round is triggered.

```mermaid
sequenceDiagram
    participant App as Application
    participant PC as PeerConnection
    participant T as RtpTransceiver

    Note over App,T: Initial state: SFrame not enabled, media flowing

    App->>T: SetUseSFrame()
    T-->>App: OK

    PC-->>App: onnegotiationneeded
    Note over PC: Transceiver's SFrame state differs from<br/>what was last negotiated → needs renegotiation

    App->>PC: CreateOffer()
    PC-->>App: New offer with "a=sframe"

    Note over App,T: Application proceeds with offer/answer exchange...
```

---

### SFrame Cannot Be Re-Enabled After Being Disabled

Once SFrame has been explicitly disabled (e.g. through a negotiation outcome), attempting to re-enable it is rejected.

```mermaid
sequenceDiagram
    participant App as Application
    participant T as RtpTransceiver

    App->>T: SetUseSFrame()
    T-->>App: OK (SFrame enabled)

    Note over App,T: After negotiation, SFrame was<br/>disabled on this transceiver

    App->>T: SetUseSFrame()
    T-->>App: ❌ InvalidModification
    Note over App: "Cannot set useSFrame to true<br/>after it has been set to false"
```

---

### New Transceiver Created From Remote Offer With SFrame

When a remote offer creates a new transceiver, the SFrame state from the offer is applied to it.

```mermaid
sequenceDiagram
    participant SDP as SDP Exchange
    participant PC as PeerConnection B
    participant T as New RtpTransceiver

    SDP->>PC: Remote offer with new m= section containing "a=sframe"
    PC->>PC: SetRemoteDescription(offer)
    Note over PC: No existing transceiver for this m= section

    PC->>T: Create new RtpTransceiver with SFrame enabled
    Note over T: Transceiver inherits SFrame state from remote offer

    Note over PC: Application can now create answer with matching SFrame
```
