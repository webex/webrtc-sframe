# SFrame Requirement Propagation

This document describes how the SFrame requirement moves from application intent
to media-stream enforcement, specifies the propagation behavior for every
offer/answer scenario, and compares **three designs** for where the
section-level requirement is held.

SFrame is negotiated for an SDP media section, and a media section maps to one
transceiver and its media channels. Once SFrame is required for that section,
the requirement must reach every stream the section owns — both the streams that
already exist and any added later. All three designs achieve this; they differ
only in *where the requirement is stored* and *how it is read back*.

This file is the overview: the problem, the per-stream contract shared by every
design, a short summary of each design, and the comparison. Each design's full
write-up and its four offer/answer flow diagrams live in its own file, linked
from [The Three Designs](#the-three-designs) below.

## Problem

SFrame is requested and agreed at the **signaling** layer: an application opts
in on a transceiver, and the requirement is settled through SDP offer/answer for
a media section. Enforcement, however, happens far below that, in the
**per-stream RTP media pipeline** that actually encrypts, decrypts, or drops
media.

Bridging those two ends is the challenge this document addresses: how to
propagate the negotiated SFrame configuration from the transceiver down to every
media stream the section owns. The complication is that SFrame depends on two
pieces of state that arrive at different times, from different sources:

| State | Source | Scope |
|---|---|---|
| SFrame is required | SDP negotiation for the media section | Whole media section |
| Media crypto is available | Application-created encryptor or decryptor | One sender or receiver stream |

They can arrive in either order, before or after the channel and its streams
exist. The section records the requirement first; each stream then receives a
per-stream configuration that carries the requirement and, once the application
provides it, the actual encryptor or decryptor.

This makes the system **fail closed**: during the gap between "SFrame is
required" and "the media crypto object is attached", the stream drops media
rather than sending or delivering it without the expected SFrame protection. The
goal is a propagation path that converges on the correct per-stream state
regardless of ordering, and fails closed in the meantime.

## Ownership Model

| Layer | Responsibility |
|---|---|
| Application | Opts in to SFrame and receives key-management handles. |
| Transceiver | Records local SFrame intent and drives SFrame negotiation in SDP. |
| Section requirement | Holds the negotiated "SFrame required" decision for the media section. **Where this lives is the axis the three designs below differ on.** |
| Send and receive media channels | Own the per-direction streams and apply the requirement to each active and future stream's configuration. |
| Stream configuration | Tracks the SFrame requirement and optional media crypto for one stream. |
| RTP pipeline | Encrypts, decrypts, or drops based on the stream configuration. |

The important boundary is between the transceiver and the media side. The
transceiver participates in signaling and applies the negotiated result
downward. Crucially, the value applied comes from the **local** description
(offer = local intent, answer = the negotiated result), so the requirement
reflects the *agreed* media-section mode — not the remote peer's wish. This is
what keeps an answerer who declines SFrame on the plaintext path.

## The Per-Stream Contract

Every design feeds the same target: a small, explicit configuration object on
each stream. For each direction the configuration holds just two things: whether
SFrame is **required**, and an optional reference to the media crypto object — an
encryptor on the send side, a decryptor on the receive side. The reference keeps
the crypto object alive for as long as the sender or receiver holds its copy.

The configuration has exactly three meaningful forms:

| Configuration form | Meaning | Send behavior | Receive behavior |
|---|---|---|---|
| Not required | SFrame is not required for the stream. | Normal RTP send path. | Normal RTP receive path. |
| Required, no crypto | SFrame is mandatory but no crypto is attached yet. | Drop outgoing frames. | Drop incoming SFrame ciphertext. |
| Required, crypto attached | SFrame is mandatory and the RTP path can process media. | Encrypt per the sender's SFrame mode. | Decrypt per the packet descriptor. |

This keeps SFrame policy out of the RTP stack: the RTP path only checks whether
SFrame is required and uses the crypto object when present. Because the
requirement is explicit (rather than inferred from "an object exists"), the two
active states are unambiguous, which is what enables the fail-closed behavior in
the middle row.

This contract is the same in all three designs below. They differ only in **how
the "required" flag reaches it** — where the section requirement is stored, how
streams that already exist are marked, and how streams added later pick it up.

---

## The Three Designs

Three designs satisfy the per-stream contract above. They differ only in where
the section requirement is stored and how it reaches the streams. Each has its
own detail page with a full write-up and the four offer/answer flows.

### Option 1 — Channel latch with fan-out

The base channel holds the requirement as a single latch and fans it out into
stream configuration; no per-direction media channel keeps its own copy. One
source of truth, nothing re-cached across layers, but the base channel gains
SFrame-specific runtime policy.
→ [propagation-option-channel-latch.md](propagation-option-channel-latch.md)

### Option 2 — Per-channel requirement cache

Each media channel caches the requirement as its own member, set from the
content-application path, and fans it out; new streams read the cached member.
Smallest change, but the value is duplicated on the send and receive channels.
→ [propagation-option-per-channel-cache.md](propagation-option-per-channel-cache.md)

### Option 3 — Dedicated SFrame-state object

A worker-confined state object owned by the transceiver is the single source of
truth; both media channels observe it and read through it, keeping no copy. No
per-layer duplication and the base channel stays out of SFrame policy, but it
adds a new type with its own observer and lifetime rules.
→ [propagation-option-sframe-state.md](propagation-option-sframe-state.md)

---

## Comparison

| | Option 1 — Channel latch | Option 2 — Per-channel cache | Option 3 — State object |
|---|---|---|---|
| Where the requirement lives | Base-channel latch | A member on each media channel (two copies) | One shared state object |
| Per-layer duplication | None beyond the latch | Yes (send + receive copies) | None (read-through) |
| Base channel knows about SFrame | Yes | No (only forwards a content field) | No |
| Both directions share one value | Yes (the channel) | No (two copies) | Yes (the object) |

All three mark streams in the same two places — the streams present when the
requirement is recorded, and each stream added afterward — and all three are
fail-closed in the gap before crypto is attached. In the current SFrame flows
the **send** side is always empty when the requirement is first recorded (the
send stream is added later in the same local-description apply), so the
mark-existing-streams step only ever does real work on the **receive** side, and
only in the passive-answerer flow (Flow 2 in each option). In every other flow
the new-stream path does the work.

Under Unified Plan a section has at most one sender and one receiver, so each
direction has at most one stream — the "streams" above are really 0-or-1 per
direction (simulcast is still a single stream with multiple SSRCs). The fan-out
therefore exists to absorb **ordering** (stream-before-requirement vs.
requirement-before-stream) and to keep both directions on one path, not to handle
many streams.

## Propagation Lifecycle

| Step | What happens |
|---|---|
| 1. Application opts in | The application asks a sender or receiver for an SFrame key-management handle. |
| 2. Transceiver records intent | The transceiver marks SFrame as desired for the media section and requests renegotiation. |
| 3. SDP confirms SFrame | Offer/answer completes with SFrame agreed for the media section. |
| 4. Section requirement is recorded | Applying the negotiated result records SFrame as required for the section (in whichever store the design uses). |
| 5. Streams become required | The fan-out marks each owned send and receive stream required. |
| 6. Media crypto is attached | When the encryptor or decryptor is available for a stream, it is stored in that stream's configuration and the stream is recreated. |
| 7. RTP path enforces policy | The RTP path encrypts and decrypts when crypto is present, and drops media while only the requirement is present. |

The same section requirement applies to streams created after negotiation: a
newly created stream is marked required immediately if its section already
requires SFrame (fan-out point two).

## Flows

The four offer/answer scenarios — the offerer, the passive answerer, and the two
add-track answerer cases (accepting and declining SFrame) — are documented **per
design**, because the propagation steps differ between them. Each option file
contains the same four flows rendered for that design:

- [Option 1 — channel latch: flows](propagation-option-channel-latch.md#flows)
- [Option 2 — per-channel cache: flows](propagation-option-per-channel-cache.md#flows)
- [Option 3 — SFrame-state object: flows](propagation-option-sframe-state.md#flows)

They focus on the video path; audio takes the same route.
