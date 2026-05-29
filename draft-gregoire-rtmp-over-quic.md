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
  RFC9002: RFC9002
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
* Protocol operation for creating, using, retiring, and rejecting Flow IDs.
* RTMP session handling over QUIC streams and QUIC DATAGRAM frames.
* A common RoQR frame format for QUIC streams and QUIC DATAGRAM frames.
* Receiver behavior for decoding RTMP message metadata and payloads.
* Guidance for choosing streams, DATAGRAM frames, or mixed operation.
* Congestion-control and rate-adaptation guidance for RoQR senders.
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

# Protocol Operation {#protocol-operation}

RoQR uses Adobe RTMP semantics for what an RTMP application flow represents and
uses QUIC transport behavior for how that flow is carried.  There is no RTMP
RFC that defines a newer application lifecycle for RTMP streams; RoQR therefore
preserves RTMP message-stream semantics from {{RTMP}} and adds a QUIC-facing
Flow ID only for multiplexing RoQR frames on one QUIC connection.

Flow ID `0` is the default RTMP session flow.  Endpoints MAY use only Flow ID
`0` when they do not need to separate independent RTMP publications,
subscriptions, control traffic, or media paths.  RTMP command and control
messages that establish connection state, create streams, publish media, play
media, or tear down streams can be carried on Flow ID `0`.

Additional Flow IDs are created by RTMP application state.  A Flow ID can be
bound to an RTMP stream name, an RTMP message stream ID, a successful publish
or play operation, or other local session state agreed by the application.
RoQR does not define new RTMP command syntax for announcing Flow IDs.  If an
application needs to signal the association explicitly, it does so using its
existing RTMP command or control model, or by an out-of-band mechanism.

Any command or control exchange that binds a Flow ID to RTMP application state
SHOULD be delivered reliably before DATAGRAM-carried media is sent on that Flow
ID.  A sender SHOULD NOT send DATAGRAM-carried media for a Flow ID until it has
reason to believe that the receiver can associate that Flow ID with the
corresponding RTMP application state.

If an endpoint receives a RoQR frame for a Flow ID that it cannot associate
with RTMP application state, the endpoint MAY buffer a bounded amount of data
for that Flow ID, drop DATAGRAM-carried frames for that Flow ID, or close the
connection with `UNKNOWN_FLOW_ID`.  Implementations that buffer unknown-flow
data MUST bound both the number of buffered frames and the number of buffered
octets.  If the unknown data is carried on a QUIC stream and the buffering
limit is exceeded, the receiver SHOULD stop receiving the affected stream with
the `UNKNOWN_FLOW_ID` error code.  If the unknown data is carried in QUIC
DATAGRAM frames and the buffering limit is exceeded, the receiver SHOULD drop
excess DATAGRAM frames.

The lifetime of a Flow ID follows the RTMP application state to which it is
bound.  A Flow ID is active while the corresponding RTMP connection, stream,
publication, subscription, or media path is active.  A Flow ID is retired when
the corresponding RTMP application state ends, such as through unpublish,
closeStream, deleteStream, connection close, or application policy.

A Flow ID MUST NOT be reused for unrelated RTMP media while old frames for the
previous use of that Flow ID can still arrive.  For DATAGRAM-carried flows, an
endpoint SHOULD either use monotonically increasing Flow IDs within a QUIC
connection or wait for an application-defined drain interval before reuse.  For
stream-carried flows, an endpoint SHOULD wait until reliable stream data for
the previous use has been consumed or reset before reuse.

The RTMP message stream ID remains RTMP application state.  RoQR Flow IDs do
not replace RTMP message stream IDs and do not change RTMP command, control,
audio, video, data, shared-object, or aggregate message semantics.

# RTMP Session Handling {#rtmp-session-handling}

RoQR carries RTMP application messages over a QUIC connection.  The QUIC
handshake and `roqr` ALPN negotiation replace the need for an RTMP transport
handshake on the RoQR connection.  Endpoints MUST NOT send the RTMP C0, C1,
C2, S0, S1, or S2 handshake octets as RoQR frames unless an application is
explicitly tunneling a legacy RTMP byte stream as payload data, which is
outside the scope of this mapping.

RTMP command messages, including commands such as `connect`, `createStream`,
`publish`, `play`, `closeStream`, and `deleteStream`, retain their RTMP command
semantics when carried by RoQR.  RoQR does not define new AMF command syntax
and does not change RTMP transaction identifiers, command object contents,
stream names, or application-specific command arguments.

RTMP command and control messages that establish or mutate session state SHOULD
be sent over QUIC streams.  This includes connection establishment commands,
stream creation commands, publication and subscription commands, stream
teardown commands, user control messages, acknowledgements, peer bandwidth
messages, and application-specific commands that affect authorization or stream
state.  A receiver MUST process these messages according to the RTMP
application semantics associated with their Message Type and Message Stream ID.

RTMP audio, video, data, shared-object, and aggregate messages are associated
with the RTMP Message Stream ID carried in the RoQR frame.  A sender MAY carry
these messages on Flow ID `0`, or MAY use additional Flow IDs to separate
publications, subscriptions, tracks, or application-defined media paths.  If a
sender uses additional Flow IDs, the receiver needs to learn the association
between the Flow ID and the RTMP application state before it can process media
for that flow, as described in {{protocol-operation}}.

RTMP Set Chunk Size messages do not change the RoQR frame format and do not
cause RoQR frames to be split at RTMP chunk boundaries.  When an endpoint is
translating between RoQR and a legacy chunked RTMP peer, Set Chunk Size affects
only the chunked RTMP side of that translation.  Similarly, RTMP
acknowledgement and peer-bandwidth messages remain RTMP application messages,
but QUIC flow control, loss recovery, and congestion control govern transport
delivery on the RoQR connection.

Session state that is required before media can be decoded or interpreted
SHOULD be delivered reliably.  Examples include metadata, codec configuration,
authorization results, and command responses that establish a publish or play
operation.  Latency-sensitive RTMP media messages MAY be sent in QUIC DATAGRAM
frames only when the application can tolerate loss of those messages and can
resynchronize the receiver after loss.

# Encapsulation

RoQR uses the same RoQR frame format for both QUIC streams and QUIC DATAGRAM
frames.  The frame identifies a flow, carries RTMP message metadata, and then
carries one complete RTMP message payload.

## Multiplexing

Every RoQR frame starts with a Flow ID.  The Flow ID is a QUIC variable-length
integer.  Flow ID lifecycle is defined in {{protocol-operation}}.

An endpoint MUST associate each received RoQR frame with the Flow ID encoded in
that frame.

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

The Message Type, Message Stream ID, Chunk Stream ID, Timestamp, and Payload
Length fields represent RTMP message metadata after chunk reassembly.  The
Message Stream ID remains RTMP application state.  The Chunk Stream ID is
preserved as metadata for implementations that use chunk-stream affinity, but
RoQR does not require a receiver to recreate the sender's original RTMP chunk
boundaries.

The Payload field MUST contain exactly one complete RTMP message payload.  A
sender MUST NOT split one RTMP message payload across multiple RoQR frames.  A
sender MUST NOT concatenate multiple RTMP message payloads into one RoQR frame.
The Payload Length field MUST be greater than zero.

## RTMP Field Semantics {#rtmp-field-semantics}

This section defines timestamp, extended timestamp, message stream, chunk
stream, and chunking semantics for RoQR.  RoQR carries RTMP messages after RTMP
chunk reassembly.  The RoQR Timestamp field carries the RTMP message timestamp
in milliseconds as a QUIC variable-length integer.  It does not carry the
three-octet timestamp field from an RTMP chunk header, and it does not carry
the RTMP Extended Timestamp field as a separate field.

When translating chunked RTMP into RoQR, an endpoint MUST resolve RTMP
timestamp and timestamp-delta encoding, including RTMP Extended Timestamp
fields, before encoding the RoQR frame.  If an RTMP chunk uses the extended
timestamp mechanism because the chunk timestamp or timestamp delta is greater
than or equal to `0xFFFFFF`, the RoQR Timestamp field carries the resulting
complete RTMP message timestamp value.  The `0xFFFFFF` chunk-header sentinel
MUST NOT be preserved as a sentinel in RoQR.

When translating RoQR back into chunked RTMP, an endpoint reconstructs RTMP
chunk headers according to {{RTMP}}.  If the RoQR Timestamp value or the
timestamp delta selected for a generated RTMP chunk cannot fit in the
three-octet RTMP chunk timestamp or timestamp-delta field, the generated RTMP
chunk uses the RTMP Extended Timestamp field as defined by {{RTMP}}.  This
translation is local to the RTMP chunking layer and does not change the RoQR
wire image.

RoQR frames are message-oriented.  A RoQR frame payload contains one complete
RTMP message payload after RTMP chunk reassembly.  RTMP chunk basic headers,
RTMP chunk message headers, extended timestamp fields, and chunk-size
boundaries are not serialized into the RoQR Payload field.  A sender MUST NOT
split one RTMP message payload across multiple RoQR frames for the purpose of
preserving RTMP chunk boundaries.

The RoQR Chunk Stream ID field carries the RTMP chunk stream identifier
associated with the source RTMP message.  It is retained for implementations
that preserve RTMP chunk-stream affinity or use chunk stream identifiers as
part of local RTMP session state.  RoQR does not require the receiver to
reconstruct the same RTMP chunk boundaries that were present at the sender.
When an endpoint generates chunked RTMP from RoQR, it MAY choose a chunk size
and chunk boundaries appropriate for its local RTMP peer, subject to RTMP chunk
stream rules and the Message Stream ID, Message Type, Timestamp, and Payload
Length values carried in the RoQR frame.

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
Using one QUIC stream per Flow ID is a simple deployment model when the
application needs reliable in-order processing within that flow.  Applications
that open multiple streams for one Flow ID need an application-level reason to
do so and need receiver behavior for cross-stream ordering.

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

# Ordering and Loss Handling {#ordering-loss-handling}

QUIC streams provide reliable ordered delivery within each stream.  RoQR does
not impose ordering across different QUIC streams and does not impose ordering
between QUIC streams and QUIC DATAGRAM frames.  Applications that require a
single total order for a set of RTMP messages SHOULD send those messages on the
same QUIC stream or define an application-level ordering rule.

Within a Flow ID, a sender SHOULD preserve RTMP application order for command,
control, metadata, and reliably delivered media messages.  A sender that sends
messages for the same Flow ID over multiple QUIC streams MUST ensure that the
receiver can reconstruct the application order or can safely process those
messages without a total order.

QUIC DATAGRAM frames preserve application frame boundaries but are not
retransmitted by QUIC.  A receiver MUST be prepared for DATAGRAM-carried RoQR
frames to be lost or delivered later than stream-carried RoQR frames for the
same Flow ID.  A receiver MUST NOT wait indefinitely for a missing
DATAGRAM-carried RoQR frame.

A receiver that detects a gap in DATAGRAM-carried RTMP media for a Flow ID MUST
treat the media timeline for that flow as discontinuous until it receives data
that the application can decode without relying on the missing message.  For
video, this normally requires a random access point and any metadata or decoder
configuration needed to decode from that point.  For audio, this can require
discarding or concealing missing audio data until the next usable audio frame.

RTMP command, control, authorization, metadata, and decoder-configuration
messages that are required for correct session operation SHOULD be sent over
QUIC streams or repeated often enough that a receiver can recover after
DATAGRAM loss.  A sender SHOULD NOT send a DATAGRAM-carried media message that
depends on a previous DATAGRAM-carried message unless the application can
tolerate loss of either message.

If a sender uses both QUIC streams and QUIC DATAGRAM frames for one Flow ID, it
SHOULD avoid dependencies from DATAGRAM-carried messages to later
stream-carried messages that would increase latency.  It also SHOULD avoid
dependencies from stream-carried messages to DATAGRAM-carried messages that can
be lost unless the application has a recovery mechanism.

If an RTMP media message sent on a QUIC stream becomes too old to be useful
before it is completely delivered, an endpoint MAY reset or stop receiving the
affected stream using the `FRAME_CANCELLED` error code.  This signal applies to
the QUIC stream carrying the stale RoQR frame; it is not a request to tear down
the RTMP application stream or retire the Flow ID unless the application also
performs that teardown.

# RTMP Message Type Handling {#rtmp-message-type-handling}

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
connection.  The choice is application-specific.  Ordering and loss behavior
for these delivery modes is described in {{ordering-loss-handling}}.
Congestion-control and rate-adaptation considerations are described in
{{congestion-control}}.

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

# Congestion Control and Rate Adaptation {#congestion-control}

RoQR relies on QUIC congestion control for network safety.  QUIC endpoints
perform congestion control and loss recovery as specified by QUIC Recovery
{{RFC9002}}.  RoQR senders MUST treat both QUIC streams and QUIC DATAGRAM
frames as congestion-controlled transport.  QUIC DATAGRAM frames are not
retransmitted by QUIC, but they remain subject to QUIC congestion control,
pacing, path validation, and amplification limits.

An RTMP application can produce media faster than the QUIC connection can send
without building unacceptable latency.  When this happens, a RoQR sender SHOULD
adapt the application media rate or delivery behavior.  Possible responses
include reducing encoder bitrate, dropping stale DATAGRAM-carried media before
submitting it to QUIC, switching selected media to a different delivery mode,
or closing a flow that can no longer meet application latency requirements.

Applications SHOULD avoid running a competing congestion controller above
QUIC.  If an application has media-rate adaptation logic, it SHOULD use
transport signals exposed by the QUIC implementation, such as congestion
window, pacing rate, bytes in flight, stream backpressure, DATAGRAM send
failures, observed round-trip time, delivery observations, and loss
observations.  Applications that cannot access these signals can still bound
latency by limiting per-flow queues and dropping stale media before transport
submission.

QUIC stream flow control and congestion control can cause reliable RoQR frames
to queue behind earlier stream data.  Applications that carry media on streams
SHOULD define a maximum useful media age.  If a stream-carried media frame
becomes too old to be useful before delivery, an endpoint MAY reset or stop
receiving the affected stream with the `FRAME_CANCELLED` error code, as
described in {{ordering-loss-handling}}.

When real-time media, RTMP commands, metadata, and non-real-time application
data share one QUIC connection, an endpoint SHOULD define priority and
bandwidth-allocation policy across Flow IDs and RTMP message classes.  Command
and control messages that are required for session correctness need reliable
delivery, but they SHOULD NOT be allowed to build unbounded queues that prevent
the sender from adapting real-time media.

# Error Handling

RoQR endpoints can close the QUIC connection with application error codes from
{{error-codes}}.  A receiver that detects malformed RoQR framing MAY close the
connection with `FRAME_ENCODING_ERROR`.  A receiver that detects a frame for an
unknown flow MAY close the connection with `UNKNOWN_FLOW_ID`.

If a DATAGRAM-carried RoQR frame is malformed, a receiver MAY drop that
DATAGRAM without closing the connection.  This behavior is useful when
applications prefer to isolate unreliable media loss from the rest of the QUIC
connection.

# 0-RTT and Connection Migration {#zero-rtt-migration}

RoQR endpoints MAY use QUIC 0-RTT only when the application has determined that
the early data is safe to replay.  RTMP command messages that create sessions,
authorize users, publish streams, play streams, delete streams, or otherwise
mutate application state SHOULD NOT be sent in 0-RTT unless the application has
replay protection for those commands.

DATAGRAM-carried media sent in 0-RTT can be replayed by an attacker within the
limits of QUIC early data.  Applications that permit media in 0-RTT MUST ensure
that replayed media cannot grant authorization, corrupt persistent state, or
cause unbounded resource consumption.  A receiver MUST be prepared for 0-RTT
rejection and for early media to be unavailable if the QUIC handshake completes
without accepting early data.

QUIC connection migration can change the network path used by a RoQR
connection.  After migration, a sender SHOULD treat DATAGRAM size, pacing
behavior, congestion state, loss behavior, and observed round-trip time as
path-dependent until the QUIC implementation validates the new path and exposes
updated transport state.  A sender SHOULD be prepared to reduce media rate,
drop stale DATAGRAM-carried media before transmission, or switch selected media
to QUIC streams while the new path is being validated.

Application authorization is bound to the RTMP application session carried by
the QUIC connection.  Applications that make authorization or policy decisions
based on client IP address, port, network attachment, or other path properties
MUST reevaluate those policies after connection migration or disable migration
for sessions that cannot safely tolerate a path change.

# Security Considerations

RoQR inherits the security properties of QUIC {{RFC9000}} and QUIC-TLS
{{RFC9001}}.  QUIC encrypts application payloads, authenticates the transport
connection, provides integrity protection, provides congestion control, and
supports connection migration.  The security considerations of QUIC, QUIC-TLS,
and QUIC DATAGRAM apply to RoQR.

QUIC protects traffic between QUIC endpoints.  RoQR does not add end-to-end
object security above QUIC.  If a relay, gateway, load balancer, or other
intermediary terminates QUIC, that intermediary can observe and modify RTMP
commands, metadata, and media unless the application applies a separate
end-to-end protection mechanism above RoQR.  Applications that rely on
intermediaries MUST define which intermediaries are trusted for command
processing, metadata inspection, media forwarding, and logging.

RTMP application payloads can contain commands, credentials, authorization
tokens, stream names, metadata, media, and application-specific data.  These
values can be sensitive even when the media payload itself is not.  Endpoints
and intermediaries SHOULD avoid logging credentials, stream keys, authorization
tokens, and application command objects unless explicitly required by local
policy.

RTMP command messages can create sessions, publish streams, play streams, tear
down streams, and mutate application state.  Endpoints MUST apply the same
authentication, authorization, and input validation to RTMP commands received
over RoQR that they apply to commands received over other RTMP transports.  A
receiver MUST NOT treat a command as authorized solely because it arrived on an
already-authenticated QUIC connection.

DATAGRAM-carried media can be lost or delayed without retransmission.
Applications MUST ensure that loss of DATAGRAM-carried RTMP messages does not
cause unsafe parser state, unbounded buffering, decoder resource exhaustion, or
application state corruption.  Receivers MUST bound buffering for unknown Flow
IDs, incomplete stream frames, and media waiting for decoder
resynchronization.

Attackers can attempt to exhaust resources by sending many unknown Flow IDs,
opening many QUIC streams, sending large payloads, sending incomplete stream
frames, or sending expensive RTMP commands.  Implementations MUST enforce local
limits on active Flow IDs, buffered unknown-flow data, stream count, payload
size, command processing, and incomplete-frame buffering.  When limits are
exceeded, an endpoint MAY drop DATAGRAM frames, stop affected streams, or close
the connection with an appropriate RoQR error code.

If an application falls back from RoQR to RTMP over TCP, RTMPS, or another
transport, it MUST NOT silently reduce the security policy expected by the
application.  In particular, fallback MUST NOT disable authentication,
authorization, encryption, token validation, or metadata privacy requirements
that were required for the RoQR session.

Gateways that translate between RoQR and legacy RTMP expose a trust boundary.
If the legacy RTMP side is not protected by TLS or another security mechanism,
commands, metadata, media, stream names, and credentials can be visible or
modifiable on that side of the gateway.  Deployments using translation
gateways SHOULD document whether the gateway is trusted, whether the legacy
side is protected, and which security properties do not cross the gateway.

Metadata visible to RoQR endpoints can reveal session structure.  Message
types, message sizes, timing, Flow IDs, stream names, and RTMP metadata can
reveal information about application behavior even when media payloads are
encrypted on the wire by QUIC.

0-RTT replay and connection migration considerations are described in
{{zero-rtt-migration}}.

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
