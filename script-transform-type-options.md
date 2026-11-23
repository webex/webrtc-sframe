# Telling the RTP stack that a frame transformer exchanges Sframe frames

Status: proposal, two options under discussion. Bug: webrtc:479862368.

## Problem

The W3C Encoded Transform spec lets a script transform declare that it
exchanges Sframe encrypted frames with the user agent:

```webidl
enum RTCRtpScriptTransformType { "sframe" };
dictionary WorkerAndParameters { required Worker worker; RTCRtpScriptTransformType type; };
constructor(WorkerOrWorkerAndParameters workerOrWorkerAndParameters, ...);
```

The spec is expected to replace `type: "sframe"` with a boolean
`sframe: true`.

The application (in JS, a worker) then does the Sframe encryption itself. The
user agent:

- sends the transform output with the Sframe RTP payload format instead of
  codec packetization, and doesn't encrypt it again;
- on receive, gives the transform the Sframe ciphertext instead of decrypting
  it natively;
- decides Sframe on the transceiver when the transform is assigned
  (`transform` setter, §2.2):
  - a transform with Sframe enables Sframe if the transceiver's state is
    undefined, and throws `InvalidModificationError` if Sframe is disabled;
  - a transform without Sframe disables Sframe if the state is undefined, and
    throws `InvalidModificationError` if Sframe is enabled.

What the spec fixes:

- the Sframe flag belongs to the `RTCRtpScriptTransform` object and is set
  at construction (§5.2). No API changes it afterwards;
- the flag is checked when the transform is assigned to `sender.transform` or
  `receiver.transform` (§2.2);
- transforms "can be updated dynamically" (§2.2), so a sender can later get
  another transform, with or without Sframe.

Native WebRTC has a single entry point for transformers:

```cpp
class FrameTransformerHost {
 public:
  virtual void SetFrameTransformer(
      scoped_refptr<FrameTransformerInterface> frame_transformer) = 0;
};
```

Nothing in it says what kind of frames the transformer produces. This document
compares two ways of carrying that information from the application to the
places that need it.

## Three states, not a boolean

A script transform is always either with or without Sframe. A native
transformer can also be neither: every transformer that exists today, and any
transformer that doesn't stand for a script transform, has never said anything
about Sframe. Such a transformer must keep today's behavior and must not
change the transceiver's Sframe state.

A boolean defaulting to `false` can't tell "without Sframe, disable it" from
"not declared, leave it alone". Making `false` mean "disable" would lock every
existing transformer's transceiver to plaintext. Both options therefore carry
three states:

```cpp
// How a frame transformer relates to Sframe on its transceiver.
enum class FrameTransformerSframeUsage {
  // Not declared. The transformer doesn't affect Sframe negotiation. Default
  // for existing transformers.
  kUnspecified,
  // Exchanges Sframe encrypted frames (a script transform with Sframe).
  // Enables Sframe on the transceiver, fails if Sframe is disabled.
  kSframe,
  // Exchanges plain encoded frames (a script transform without Sframe).
  // Disables Sframe on the transceiver, fails if Sframe is enabled.
  kNoSframe,
};
```

`std::optional<bool>` would carry the same states. At call sites, though,
`if (usage)` and `if (*usage)` are easy to confuse.

The transceiver already has three states (`std::optional<bool>
sframe_enabled_`, see [transceiver-enablement.md](transceiver-enablement.md)),
but today it only reaches `false` when a negotiation completes without Sframe.
`kNoSframe` needs a local transition from undefined to `false`, the
counterpart of `TryToEnableSframe()`. From then on, offers and answers for
the section leave Sframe out.

## Option A: configuration passed with the transformer

The application says what kind of frames the transformer exchanges when it
sets it. The transformer interface doesn't change.

```cpp
struct FrameTransformerOptions {
  FrameTransformerSframeUsage sframe_usage =
      FrameTransformerSframeUsage::kUnspecified;
};

class FrameTransformerHost {
 public:
  virtual void SetFrameTransformer(
      scoped_refptr<FrameTransformerInterface> frame_transformer) = 0;

  // Returns an error, and keeps the previous transformer, if
  // `frame_transformer` can't be used with `options`.
  virtual RTCError SetFrameTransformerOrError(
      scoped_refptr<FrameTransformerInterface> frame_transformer,
      const FrameTransformerOptions& options);
};
```

Internally the pointer and the options travel together, for example as:

```cpp
struct FrameTransformerConfig {
  scoped_refptr<FrameTransformerInterface> transformer;
  FrameTransformerOptions options;
};
```

This struct replaces the bare `scoped_refptr<FrameTransformerInterface>`
on the way from the host down to `RTPSenderVideo` and
`RtpVideoStreamReceiver2`, and to the audio channels:

- `SetEncoderToPacketizerFrameTransformer` and
  `SetDepacketizerToDecoderFrameTransformer` on the media channels;
- the send and receive stream interfaces;
- `VideoSendStream::Config::frame_transformer`.

## Option B: the transformer declares itself

The transformer says how it relates to Sframe. Like in the spec, this is a
property of the object.

```cpp
class FrameTransformerInterface : public RefCountInterface {
 public:
  // How this transformer relates to Sframe. With kSframe it encrypts and
  // decrypts Sframe itself: on send it outputs Sframe ciphertext, which is
  // sent with the Sframe RTP payload format, and on receive it gets Sframe
  // ciphertext. Must not change during the lifetime of the transformer.
  virtual FrameTransformerSframeUsage GetSframeUsage() const {
    return FrameTransformerSframeUsage::kUnspecified;
  }
  ...
};
```

Each layer that needs to know asks the transformer it already has.

The uploaded chain 512240–512300, not yet sent for review, implements this
option for video send. It uses a two-value enum (`FrameTransformerType` with
`kDefault` and `kSframe`, returned by `GetType()`). Its `kDefault` acts as
`kUnspecified`, and it has no `kNoSframe`.
