---
title: "RTMP over QUIC"
abbrev: "RTMP over QUIC"
category: exp

docname: draft-gregoire-rtmp-over-quic-latest
submissiontype: IETF
number:
date:
consensus: false
v: 3
# area: "Applications and Real-Time"
keyword:
 - RTMP
 - QUIC
 - media transport
venue:
  github: "mondain/roqr"
  latest: "https://mondain.github.io/roqr/draft-gregoire-rtmp-over-quic.html"

author:
 -
    fullname: Paul Gregoire
    organization: Red5
    email: paul@red5.net

normative:
  RFC9000: RFC9000
  RFC9001: RFC9001
  RFC9221: RFC9221
  RFC7301: RFC7301
  RTMP:
    title: "Adobe's Real Time Messaging Protocol"
    target: "https://rtmp.veriskope.com/docs/spec/"
    author:
      -
        org: "Adobe Systems Incorporated"
    date: 2012

informative:
  RoQ: I-D.draft-ietf-avtcore-rtp-over-quic

--- abstract

This document specifies a mapping for encapsulating Real-Time Messaging
Protocol (RTMP) messages within QUIC.  RTMP over QUIC carries RTMP message
metadata and message payloads over QUIC streams, QUIC DATAGRAM frames, or a
combination of both.  The mapping is intended for implementations that need
RTMP semantics with QUIC transport properties, including encryption,
congestion control, connection migration, and unreliable delivery for
latency-sensitive media messages.

--- middle

# Introduction

The Real-Time Messaging Protocol (RTMP) {{RTMP}} is widely deployed for live
media contribution, publication, and distribution workflows.  RTMP is commonly
run over TCP.  TCP provides ordered reliable delivery, but it also couples all
application data to a single ordered byte stream.  That coupling can be a poor
fit for latency-sensitive media messages where newer audio or video data can be
more valuable than retransmitting old data.

QUIC {{RFC9000}} provides authenticated and encrypted transport {{RFC9001}},
congestion control, stream multiplexing, and optional unreliable DATAGRAM
frames {{RFC9221}}.  This document defines RTMP over QUIC (RoQR), a minimal
mapping that carries RTMP message metadata and payloads over QUIC while
preserving RTMP message identity for existing RTMP and FLV processing paths.

RoQR follows the general transport approach used by RTP over QUIC {{RoQ}}:
application media units are associated with a flow identifier and can be sent
over QUIC streams, QUIC DATAGRAM frames, or both.  Unlike RTP over QUIC, the
payload carried by RoQR is an RTMP message payload with RTMP message metadata
rather than an RTP or RTCP packet.

# Scope

This specification defines:

* QUIC connection establishment and ALPN use for RoQR.
* A flow identifier used to multiplex RTMP flows on one QUIC connection.
* A common RoQR frame format for QUIC streams and QUIC DATAGRAM frames.
* Receiver behavior for decoding RTMP message metadata and payloads.
* Guidance for choosing streams, DATAGRAM frames, or mixed operation.
* Error codes for closing RoQR connections.

This specification does not define:

* New RTMP command, control, audio, video, aggregate, or AMF message syntax.
* A replacement for RTMP application-level session establishment.
* A media codec, container, playlist, or adaptive bitrate format.
* A reliable DATAGRAM extension.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the conventions detailed in Section 1.3 of {{RFC9000}} when
describing the binary encoding.  Integer fields described as QUIC variable-
length integers use the encoding defined in Section 16 of {{RFC9000}}.

The following terms are used throughout this document:

RoQR:
: RTMP over QUIC, the mapping defined by this document.

RTMP message:
: An RTMP message consisting of RTMP message metadata and message payload.

RTMP message payload:
: The RTMP message body bytes carried after RTMP chunk reassembly.  RoQR does
  not carry RTMP chunk basic headers, message headers, or extended timestamps
  as serialized RTMP chunk bytes.

Flow:
: A sequence of RTMP messages with application-defined meaning.  A single QUIC
  connection can carry multiple flows.

Flow ID:
: A QUIC variable-length integer that identifies the flow associated with a
  RoQR frame.

RoQR frame:
: The RoQR application frame defined in {{roqr-frame-format}}.

# Connection Establishment and ALPN

QUIC uses Application-Layer Protocol Negotiation (ALPN) {{RFC7301}} during
connection setup.  RoQR endpoints use the ALPN token `roqr` to identify this
mapping.

An endpoint that negotiates `roqr` MUST use the framing and error handling
defined in this document for RTMP media carried on that QUIC connection.
Endpoints MUST NOT send RoQR frames before the QUIC handshake has completed
unless an application explicitly enables QUIC early data and accepts the replay
risks described by QUIC and TLS.

RoQR endpoints that intend to send RTMP messages in QUIC DATAGRAM frames MUST
negotiate the QUIC DATAGRAM extension {{RFC9221}}.  If the DATAGRAM extension
is not negotiated, endpoints MUST use QUIC streams for RoQR frames.

# Encapsulation

RoQR uses the same RoQR frame format for both QUIC streams and QUIC DATAGRAM
frames.  The frame identifies a flow, carries RTMP message metadata, and then
carries one complete RTMP message payload.

## Multiplexing

Every RoQR frame starts with a Flow ID.  The Flow ID is a QUIC variable-length
integer.  Flow ID `0` is the default RTMP flow.  Other Flow ID values are
application-defined and can be used to separate publications, subscriptions,
control-related messages, media-related messages, or other RTMP application
units.

An endpoint MUST associate each received RoQR frame with the Flow ID encoded in
that frame.  If an endpoint receives a frame for an unknown Flow ID, it MAY
buffer the frame until the application creates that flow, drop the frame, or
close the connection with the `UNKNOWN_FLOW_ID` error code.

## RoQR Frame Format {#roqr-frame-format}

The RoQR frame format is:

~~~ ascii-art
+========+==============+==============+===================+
| Field  | Type         | Length       | Description       |
+========+==============+==============+===================+
| Flow   | QUIC varint  | 1, 2, 4, or  | RoQR Flow ID      |
| ID     |              | 8 octets     |                   |
+--------+--------------+--------------+-------------------+
| Time-  | QUIC varint  | 1, 2, 4, or  | RTMP timestamp in |
| stamp  |              | 8 octets     | milliseconds      |
+--------+--------------+--------------+-------------------+
| Message| uint8        | 1 octet      | RTMP message type |
| Type   |              |              | identifier        |
+--------+--------------+--------------+-------------------+
| Message| QUIC varint  | 1, 2, 4, or  | RTMP message      |
| Stream |              | 8 octets     | stream ID         |
| ID     |              |              |                   |
+--------+--------------+--------------+-------------------+
| Chunk  | QUIC varint  | 1, 2, 4, or  | RTMP chunk stream |
| Stream |              | 8 octets     | ID associated     |
| ID     |              |              | with the message  |
+--------+--------------+--------------+-------------------+
| Payload| QUIC varint  | 1, 2, 4, or  | RTMP message      |
| Length |              | 8 octets     | payload length    |
+--------+--------------+--------------+-------------------+
| Payload| bytes        | Payload      | RTMP message      |
|        |              | Length       | payload           |
+--------+--------------+--------------+-------------------+
~~~

The Timestamp field contains the RTMP timestamp in milliseconds.  The Message
Type field contains the one-octet RTMP message type identifier.  The Message
Stream ID field contains the RTMP message stream identifier.  The Chunk Stream
ID field contains the RTMP chunk stream identifier associated with the original
RTMP message.  The Payload Length field contains the number of octets in the
Payload field.

The Payload field MUST contain exactly one complete RTMP message payload.  A
sender MUST NOT split one RTMP message payload across multiple RoQR frames.  A
sender MUST NOT concatenate multiple RTMP message payloads into one RoQR frame.
The Payload Length field MUST be greater than zero.

## QUIC Streams

When RoQR frames are sent over QUIC streams, the sender writes one or more RoQR
frames to a unidirectional or bidirectional QUIC stream.  Each frame is
self-delimiting because it contains a Payload Length field.  A receiver MUST
process complete frames in the order received on a stream.  If a partial frame
is received, the receiver waits for more stream data until the frame is
complete, the stream is reset, or the connection is closed.

Stream delivery is appropriate for RTMP messages that require reliable delivery
or are needed to preserve application semantics, such as command messages,
metadata, or media that the application has chosen to deliver reliably.

## QUIC DATAGRAM Frames

When RoQR frames are sent in QUIC DATAGRAM frames, each QUIC DATAGRAM frame
MUST contain exactly one complete RoQR frame and no trailing bytes.  Because
QUIC DATAGRAM frames are not retransmitted by QUIC, senders SHOULD use DATAGRAM
frames only for RTMP messages where dropping an old message is preferable to
adding latency through retransmission.

A sender MUST ensure that the encoded RoQR frame fits within the maximum
DATAGRAM size available for the QUIC path and peer.  If an RTMP message payload
does not fit in a QUIC DATAGRAM frame, the sender MUST send it over a QUIC
stream or drop it according to application policy.

# RTMP Message Type Handling

RoQR preserves the RTMP message type identifier.  Receivers MUST pass unknown
message type identifiers to the application together with the associated Flow
ID, timestamp, message stream ID, chunk stream ID, and payload.  Receivers MUST
NOT fail a frame solely because the Message Type field contains an unrecognized
value.

The following message type identifiers are common in RTMP media workflows:

| ID | RTMP message type |
|----|-------------------|
| 1  | Set Chunk Size |
| 2  | Abort Message |
| 3  | Acknowledgement |
| 4  | User Control |
| 5  | Window Acknowledgement Size |
| 6  | Set Peer Bandwidth |
| 8  | Audio |
| 9  | Video |
| 15 | AMF3 Data |
| 16 | AMF3 Shared Object |
| 17 | AMF3 Command |
| 18 | AMF0 Data |
| 19 | AMF0 Shared Object |
| 20 | AMF0 Command |
| 22 | Aggregate |

# Choosing Streams, DATAGRAM Frames, or Both

RTMP applications can use streams, DATAGRAM frames, or both on the same QUIC
connection.  The choice is application-specific.

Applications SHOULD use QUIC streams for RTMP messages that are required for
session correctness or decoder initialization.  Examples include command
messages, metadata messages, user control messages, and media messages whose
loss would prevent useful decoding of later media.

Applications MAY use QUIC DATAGRAM frames for latency-sensitive audio, video,
or aggregate messages when loss of an older message is acceptable.  A receiver
that detects missing DATAGRAM-carried media MUST treat the RTMP media timeline
as discontinuous until the application can resume useful decoding.

Applications that use both delivery modes SHOULD define which RTMP message
types and flows use each mode.  A receiver MUST be prepared to receive frames
for different Flow IDs over different delivery modes on the same QUIC
connection.

# Error Handling

RoQR endpoints can close the QUIC connection with application error codes from
{{error-codes}}.  A receiver that detects malformed RoQR framing MAY close the
connection with `FRAME_ENCODING_ERROR`.  A receiver that detects a frame for an
unknown flow MAY close the connection with `UNKNOWN_FLOW_ID`.

If a DATAGRAM-carried RoQR frame is malformed, a receiver MAY drop that
DATAGRAM without closing the connection.  This behavior is useful when
applications prefer to isolate unreliable media loss from the rest of the QUIC
connection.

# Security Considerations

RoQR inherits the security properties of QUIC {{RFC9000}} and QUIC-TLS
{{RFC9001}}.  QUIC encrypts application payloads and authenticates the
transport connection.

RTMP application payloads can contain commands, metadata, media, and
application-specific data.  RoQR does not add end-to-end object security above
QUIC.  Applications that require end-to-end media or metadata protection across
intermediaries need a separate application-layer protection mechanism.

DATAGRAM-carried media can be lost without retransmission.  Applications MUST
ensure that loss of DATAGRAM-carried RTMP messages does not cause unsafe parser
state, unbounded buffering, or resource exhaustion.

# IANA Considerations

This document requests registration of the `roqr` ALPN token and creates an
RTMP over QUIC error code registry.

## Registration of the RoQR Identification String

IANA is requested to add the following entry to the "TLS Application-Layer
Protocol Negotiation (ALPN) Protocol IDs" registry:

Protocol:
: RTMP over QUIC

Identification Sequence:
: `0x72 0x6f 0x71 0x72` ("roqr")

Specification:
: This document

## RoQR Error Codes Registry {#error-codes}

IANA is requested to create a new "RTMP over QUIC Error Codes" registry.  The
registry manages a 62-bit space.  New registrations use the Specification
Required policy defined by {{!RFC8126}}.

Initial entries are:

| Value | Name | Description |
|-------|------|-------------|
| 0x00 | NO_ERROR | No error |
| 0x01 | GENERAL_ERROR | General RoQR error |
| 0x02 | INTERNAL_ERROR | Internal endpoint error |
| 0x03 | FRAME_ENCODING_ERROR | Malformed RoQR frame |
| 0x04 | STREAM_CREATION_ERROR | Could not create a required QUIC stream |
| 0x05 | FRAME_CANCELLED | RoQR frame delivery was cancelled |
| 0x06 | UNKNOWN_FLOW_ID | Flow ID is unknown |
| 0x07 | EXPECTATION_UNMET | Peer behavior violated endpoint policy |

# Implementation Status

This section is to be removed before publishing as an RFC.

A prototype implementation exists in the Red5 Pro RoQ implementation.  It
supports RTMP over QUIC frame encoding and decoding for QUIC DATAGRAM frames
and QUIC streams, preserves unknown RTMP message type identifiers, and exposes
Rust and Java/JNI APIs for RTMP media publication and reception.

--- back

# Acknowledgments

This document borrows the overall QUIC transport structure from RTP over QUIC
and adapts it for RTMP message metadata and payloads.
