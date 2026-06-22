# WebRTC SFrame Architecture

These design notes describe the native WebRTC SFrame architecture. The central
design choice is that SFrame is negotiated at the RTP transceiver / m-section
level, then pushed down as a one-shot channel enablement that causes each
media stream to install an SFrame controller.

## Documents

| Document | Focus |
|---|---|
| [transceiver-stream-enablement.md](transceiver-stream-enablement.md) | How the channel-level SFrame requirement is propagated into per-stream enforcement |
| [negotiation.md](negotiation.md) | Offer/answer behavior, `a=sframe`, downgrade handling, and answer validation |
| [video-receive-pipeline.md](video-receive-pipeline.md) | Video receive packet processing, SFrame descriptor parsing, buffering, and T=0/T=1 decryption |

## Architecture Summary

SFrame starts from public sender/receiver APIs:

```cpp
RTCErrorOr<scoped_refptr<SframeEncryptorInterface>>
RtpSenderInterface::CreateSframeEncryptorOrError(
    const SframeEncryptorInit& options);

RTCErrorOr<scoped_refptr<SframeDecryptorInterface>>
RtpReceiverInterface::CreateSframeDecryptorOrError(
    SframeCipherSuite cipher_suite);
```

The sender chooses both `SframeMode` and `SframeCipherSuite`. The receiver
chooses only the cipher suite; the receive path derives per-frame vs
per-packet processing from the SFrame RTP descriptor T bit:

| T bit | Internal level | Meaning |
|---|---|---|
| `0` | `SframeEncryptionLevel::kFrame` | Per-frame/raw SFrame; decrypt after frame reassembly |
| `1` | `SframeEncryptionLevel::kPacket` | Per-packet/packetized SFrame; decrypt each RTP payload before codec depacketization |

The public key-management handles are proxies. The media pipeline receives
the internal media interfaces:

| Public handle | Internal media interface | Implementation |
|---|---|---|
| `SframeEncryptorInterface` | `SframeMediaEncryptorInterface` | `modules/sframe` media crypto adapter |
| `SframeDecryptorInterface` | `SframeMediaDecryptorInterface` | `modules/sframe` media crypto adapter |

## Enablement Model

The transceiver owns the negotiated/desired SFrame state:

```cpp
std::optional<bool> RtpTransceiver::sframe_enabled_;
```

| State | Meaning |
|---|---|
| `std::nullopt` | No local or negotiated decision yet |
| `true` | SFrame has been requested or negotiated for this m-section |
| `false` | Negotiation completed without SFrame; future enablement is rejected |

`RtpSenderBase` and `RtpReceiverBase` do not mutate this state directly.
They receive an `absl::AnyInvocable<RTCError()> enable_sframe_at_owner_`
callback bound to `RtpTransceiver::TryToEnableSframe()`. Calling either
public SFrame creation API invokes that callback first. If it succeeds, the
sender/receiver creates the internal encryptor/decryptor, pushes it to the
media channel on the worker thread, and returns a proxy handle to the app.

When SDP is applied, `SdpOfferAnswerHandler` calls
`RtpTransceiver::ApplySframeEnabled(media_desc->sframe_enabled())`. A `true`
value also calls `channel_->EnableSframe()` on the worker thread. That is the
point where negotiated SFrame is pushed from signaling state into the media
engine.

## Stream-Level Signal

Inside the video media pipeline, the stream-level signal is the presence of a
controller:

| Stream config slot | Non-null means | Missing inner crypto means |
|---|---|---|
| `VideoSendStream::Config::sframe_controller` | SFrame is required for this send stream | Drop outgoing frames |
| `VideoReceiveStreamInterface::Config::sframe_controller` | SFrame is required for this receive stream | Drop incoming ciphertext |

`WebRtcVideoSendStream::EnableSframe()` installs a
`SframeSendController(nullptr)` and recreates the stream. Later,
`SetSframeEncryptor()` replaces it with `SframeSendController(encryptor)` and
recreates the stream again so `RTPSenderVideo` sees an immutable controller.

The receive side mirrors that model with `SframeReceiveController(nullptr)`
and `SetSframeDecryptor()`. The `RtpVideoStreamReceiver2` constructor receives
the controller; if it is non-null, every RTP packet enters the SFrame
intercept path.

## Media Scope

The video media pipeline covers transceiver enablement, stream controller
installation, RTP packetization, RTP depacketization, buffering, and
encryption/decryption.

Audio uses the same public API, SDP negotiation model, and media-channel
surface. The voice-engine stream pushdown is reserved as a follow-up extension
of the same architecture.

## Verification Map

Relevant tests cover these layers:

| Layer | Tests |
|---|---|
| Transceiver state | `pc/rtp_transceiver_unittest.cc` |
| API callback and channel pushdown | `pc/rtp_sender_receiver_unittest.cc` |
| SDP negotiation and downgrade handling | `pc/sdp_offer_answer_unittest.cc`, `pc/media_session_unittest.cc` |
| SDP serialization/parsing | `api/webrtc_sdp.cc` coverage in SDP tests |
| RTP SFrame descriptor packetizer/depacketizer | `modules/rtp_rtcp/source/rtp_format_sframe_unittest.cc` |
| SFrame receive buffering | `modules/video_coding/sframe_packet_buffer_unittest.cc` |
| SFrame crypto wrapper | `modules/sframe` crypto adapter unit tests |
