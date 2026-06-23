# WebRTC SFrame Architecture

These notes give a high-level overview of the native WebRTC SFrame
architecture. The central design choice is that SFrame is negotiated per media
section, then pushed down as a one-shot channel enablement that marks every
stream in that section as requiring SFrame. Implementation details live in the
linked documents.

## Documents

| Document | Focus |
|---|---|
| [api-definitions.md](api-definitions.md) | Public SFrame API: modes, cipher suites, key-management handles, and the sender/receiver creation entry points |
| [transceiver-enablement.md](transceiver-enablement.md) | How an application enables SFrame on a transceiver, the transceiver SFrame state, and the renegotiation it triggers |
| [negotiation.md](negotiation.md) | Offer/answer behavior, the SFrame SDP attribute, downgrade handling, and answer validation |

## How It Works

SFrame is requested and negotiated at the media-section level through the SDP
offer/answer exchange. A media section maps to one transceiver and one media
channel, so the negotiated result is a single yes/no decision for that channel.
The decision always reflects the locally agreed outcome (an offer expresses
local intent, an answer expresses the negotiated result), which keeps a peer
that declines SFrame on the plaintext path.

## Two Pieces of State

SFrame protection depends on two facts that arrive at different times:

- **SFrame is required** — decided by SDP negotiation and scoped to the whole
  media channel.
- **Media crypto is available** — supplied by the application as an encryptor
  (for sending) or a decryptor (for receiving) and scoped to a single stream.

The channel records the requirement first. Each stream then receives a small
per-stream configuration that carries the requirement and, once the application
provides it, the actual encryptor or decryptor.

## Fail-Closed Enforcement

Because the requirement can be known before the crypto object is attached, the
system fails closed. During that gap a stream drops media rather than sending or
delivering it without the expected SFrame protection.

## From Negotiation to Media

Once SFrame is negotiated for a section, the requirement is latched once at the
media channel and fanned out to all current and future streams of that channel:

- **Send side** — outgoing frames are dropped until an encryptor is attached,
  then encrypted before they leave.
- **Receive side** — incoming SFrame media is routed through the SFrame path and
  dropped until a decryptor is attached, then decrypted before decoding.

## Scope

The video media pipeline is fully wired: transceiver enablement, per-stream
configuration, RTP packetization and depacketization, receive buffering, and
encryption/decryption. Audio uses the same public API and negotiation model;
extending the requirement into the voice-engine stream pushdown is a planned
follow-up of the same architecture.

## Testing

The behavior is covered by unit tests across the main layers: transceiver
state, the enable API and channel pushdown, SDP negotiation and downgrade
handling, RTP SFrame packetization and depacketization, receive buffering, and
the SFrame crypto wrapper.
