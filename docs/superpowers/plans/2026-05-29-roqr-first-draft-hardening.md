# RoQR First Draft Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn `draft-gregoire-rtmp-over-quic.md` from a buildable first scaffold into a technically reviewable `-00` individual Internet-Draft.

**Architecture:** Keep this as one draft-source edit pass with small, reviewable commits. Use `docs/draft-ietf-avtcore-rtp-over-quic-14.txt` as the local RoQ style/reference source first, and use `/media/mondain/terrorbyte/workspace/github-red5pro/roq` only to confirm implemented RTMP/RoQR wire behavior.

**Tech Stack:** kramdown-rfc, xml2rfc, idnits through the existing `Makefile` and i-d-template workflow.

---

## File Structure

- Modify: `draft-gregoire-rtmp-over-quic.md`
  - Adds all technical sections and refines existing sections.
- Reference only: `docs/draft-ietf-avtcore-rtp-over-quic-14.txt`
  - Use for RTP over QUIC structure, terminology style, transport guidance, congestion-control coverage, security coverage, IANA shape, and implementation-status style.
- Reference only: `/media/mondain/terrorbyte/workspace/github-red5pro/roq/src/rtmp.rs`
  - Use for the implemented RoQR frame fields and decoder behavior.
- Reference only: `/media/mondain/terrorbyte/workspace/github-red5pro/roq/tests/rtmp_roq.rs`
  - Use for confirmed round-trip and incomplete-frame behavior.

## Validation Commands

Run these after each task that changes the draft source:

```sh
make latest
make check-submission
rg -n "draft-gregoire-avtcore-rtmp-over-quic|Audio/Video Transport Core Maintenance|^workgroup:" draft-gregoire-rtmp-over-quic.md README.md CONTRIBUTING.md .github
```

Expected:

```text
make latest exits 0.
make check-submission exits 0.
The rg command exits 1 with no matches.
Known idnits residuals may remain LINE_PI, INVALID_REFERENCES_NAME, and NON_ASCII_UTF8 unless a task explicitly removes them.
```

### Task 1: Add Protocol Operation and Flow Lifecycle

**Files:**
- Modify: `draft-gregoire-rtmp-over-quic.md`
- Reference: `docs/draft-ietf-avtcore-rtp-over-quic-14.txt`
- Reference: `/media/mondain/terrorbyte/workspace/github-red5pro/roq/src/rtmp.rs`

- [ ] **Step 1: Read local reference sections**

Run:

```sh
rg -n "Flow ID|flow identifier|unknown flow|buffer|Multiplexing" docs/draft-ietf-avtcore-rtp-over-quic-14.txt
sed -n '148,230p' draft-gregoire-rtmp-over-quic.md
sed -n '1,220p' /media/mondain/terrorbyte/workspace/github-red5pro/roq/src/rtmp.rs
```

Expected: the local RoQ draft shows flow identifier handling and the implementation shows `DEFAULT_RTMP_FLOW_ID` and the frame fields.

- [ ] **Step 2: Add `Protocol Operation` after `Connection Establishment and ALPN`**

Insert this section before `# Encapsulation`:

```markdown
# Protocol Operation {#protocol-operation}

RoQR endpoints establish an application session over QUIC using the `roqr` ALPN
token.  After the QUIC handshake completes, RTMP application messages are
encoded as RoQR frames and carried over QUIC streams, QUIC DATAGRAM frames, or
both.

Flow ID `0` is the default RTMP flow.  Endpoints MAY use only Flow ID `0` when
they do not need to separate independent RTMP publications or subscriptions.
Applications that use additional Flow IDs MUST define the application event
that creates each Flow ID before sending media on that flow.  Examples include
successful publication setup, successful subscription setup, or an application
control exchange that binds a Flow ID to an RTMP stream name or message stream
ID.

A Flow ID MUST NOT be reused for unrelated RTMP media while frames for the
previous use of that Flow ID can still be delivered by QUIC.  An endpoint that
retires a Flow ID SHOULD wait until all reliable stream data for that flow has
been processed or reset before reusing the identifier.

If an endpoint receives a RoQR frame for a Flow ID that is not yet known to the
application, the endpoint MAY buffer a bounded number of frames for that Flow
ID, drop DATAGRAM-carried frames for that Flow ID, or close the connection with
`UNKNOWN_FLOW_ID`.  Implementations that buffer unknown-flow frames MUST bound
the number of buffered frames and buffered octets.
```

- [ ] **Step 3: Update `Scope` bullet list**

Add this bullet under "This specification defines":

```markdown
* Protocol operation for creating, using, retiring, and rejecting Flow IDs.
```

- [ ] **Step 4: Remove duplicate flow-lifecycle text from `Multiplexing`**

Keep `Multiplexing` focused on frame-level demultiplexing.  Replace the current unknown-flow paragraph with:

```markdown
An endpoint MUST associate each received RoQR frame with the Flow ID encoded in
that frame.  Flow lifecycle behavior is defined in {{protocol-operation}}.
```

Confirm the new heading includes the `{#protocol-operation}` anchor.

- [ ] **Step 5: Validate and commit**

Run:

```sh
make latest
make check-submission
git diff --check
```

Expected: all commands exit 0. Commit:

```sh
git add draft-gregoire-rtmp-over-quic.md
git commit -m "Add RoQR protocol operation"
```

### Task 2: Define RTMP Session Mapping

**Files:**
- Modify: `draft-gregoire-rtmp-over-quic.md`
- Reference: `/media/mondain/terrorbyte/workspace/github-red5pro/roq/src/rtmp.rs`
- Reference: `/media/mondain/terrorbyte/workspace/github-red5pro/roq/tests/rtmp_roq.rs`

- [ ] **Step 1: Confirm implemented message metadata**

Run:

```sh
rg -n "RtmpMessageType|RtmpMessageHeader|message_stream_id|chunk_stream_id|timestamp_ms" /media/mondain/terrorbyte/workspace/github-red5pro/roq/src/rtmp.rs /media/mondain/terrorbyte/workspace/github-red5pro/roq/tests/rtmp_roq.rs
```

Expected: the implementation preserves message type, timestamp, message stream ID, chunk stream ID, and payload.

- [ ] **Step 2: Add `RTMP Session Mapping` after `RTMP Message Type Handling`**

Insert:

```markdown
# RTMP Session Mapping {#rtmp-session-mapping}

RoQR carries RTMP messages after RTMP chunk reassembly.  RTMP command, control,
metadata, audio, video, data, shared-object, and aggregate message payloads are
encoded in the Payload field of a RoQR frame.  The RTMP message type, RTMP
timestamp, RTMP message stream ID, and RTMP chunk stream ID associated with the
message are carried in the RoQR frame header.

RTMP command and control messages that establish application state, including
connection establishment, stream creation, publication, subscription, and user
control messages, SHOULD be sent over QUIC streams unless the application has a
separate reliable control channel.  A receiver MUST process these messages
using the RTMP application semantics associated with the Message Type and
Message Stream ID fields.

Audio, video, and aggregate messages MAY be sent over QUIC DATAGRAM frames when
the application can tolerate loss of those messages.  A sender SHOULD send
messages needed for decoder initialization, metadata interpretation, or stream
resynchronization over QUIC streams, or repeat them often enough that a receiver
can recover after DATAGRAM loss.

RoQR does not define RTMP handshake bytes, RTMP chunk-size negotiation, or AMF
command syntax.  Applications that interoperate with existing RTMP application
logic are responsible for translating between local RTMP session state and the
RoQR frame fields defined by this document.
```

- [ ] **Step 3: Add cross-reference in `Choosing Streams, DATAGRAM Frames, or Both`**

Add this sentence at the end of the first paragraph:

```markdown
RTMP session-state guidance is described in {{rtmp-session-mapping}}.
```

- [ ] **Step 4: Validate and commit**

Run:

```sh
make latest
make check-submission
git diff --check
```

Expected: all commands exit 0. Commit:

```sh
git add draft-gregoire-rtmp-over-quic.md
git commit -m "Describe RTMP session mapping"
```

### Task 3: Specify Ordering, Stream Use, and Loss Behavior

**Files:**
- Modify: `draft-gregoire-rtmp-over-quic.md`
- Reference: `docs/draft-ietf-avtcore-rtp-over-quic-14.txt`

- [ ] **Step 1: Read stream and DATAGRAM guidance in the local RoQ draft**

Run:

```sh
sed -n '1181,1425p' docs/draft-ietf-avtcore-rtp-over-quic-14.txt
sed -n '1704,1888p' docs/draft-ietf-avtcore-rtp-over-quic-14.txt
```

Expected: the reference covers stream encapsulation, DATAGRAM behavior, and guidance for mixed operation.

- [ ] **Step 2: Add `Ordering and Loss Recovery` after `QUIC DATAGRAM Frames`**

Insert:

```markdown
# Ordering and Loss Recovery {#ordering-loss-recovery}

QUIC streams provide ordered reliable delivery within each stream.  RoQR does
not impose ordering across different QUIC streams.  Applications that require a
single total order for a set of RTMP messages SHOULD send those messages on the
same QUIC stream or define an application-level ordering rule.

Within a Flow ID, a sender SHOULD preserve RTMP application order for command,
control, metadata, and reliably delivered media messages.  A sender that sends
messages for the same Flow ID over multiple QUIC streams MUST ensure that the
receiver can reconstruct the application order or can safely process those
messages without a total order.

QUIC DATAGRAM frames can be lost or delayed relative to stream data.  A
receiver that detects a gap in DATAGRAM-carried media for a Flow ID MUST treat
the media timeline for that flow as discontinuous until it receives data that
the application can decode without relying on the missing message.  For video,
this normally requires a random access point and any metadata or decoder
configuration needed to decode from that point.

If a sender uses both streams and DATAGRAM frames for one Flow ID, it SHOULD
avoid dependencies from DATAGRAM-carried messages to later stream-carried
messages that would increase latency.  It also SHOULD avoid dependencies from
stream-carried messages to DATAGRAM-carried messages that can be lost unless
the application has a recovery mechanism.
```

- [ ] **Step 3: Refine `QUIC Streams` recommendation**

Append to the `QUIC Streams` section:

```markdown
Using one QUIC stream per Flow ID is a simple deployment model when the
application needs reliable in-order processing within that flow.  Applications
that open multiple streams for one Flow ID need an application-level reason to
do so and need receiver behavior for cross-stream ordering.
```

- [ ] **Step 4: Validate and commit**

Run:

```sh
make latest
make check-submission
git diff --check
```

Expected: all commands exit 0. Commit:

```sh
git add draft-gregoire-rtmp-over-quic.md
git commit -m "Specify RoQR ordering and loss behavior"
```

### Task 4: Expand Congestion Control and Rate Adaptation

**Files:**
- Modify: `draft-gregoire-rtmp-over-quic.md`
- Reference: `docs/draft-ietf-avtcore-rtp-over-quic-14.txt`

- [ ] **Step 1: Read congestion-control reference text locally**

Run:

```sh
sed -n '1500,1702p' docs/draft-ietf-avtcore-rtp-over-quic-14.txt
sed -n '2160,2185p' docs/draft-ietf-avtcore-rtp-over-quic-14.txt
```

Expected: the reference covers QUIC transport congestion control, application rate adaptation, and nested congestion controllers.

- [ ] **Step 2: Add normative reference for RFC9002**

Add under `normative`:

```yaml
  RFC9002: RFC9002
```

- [ ] **Step 3: Add `Congestion Control and Rate Adaptation` before `Choosing Streams, DATAGRAM Frames, or Both`**

Insert:

```markdown
# Congestion Control and Rate Adaptation {#congestion-control}

QUIC is a congestion-controlled transport protocol.  RoQR senders rely on the
QUIC congestion controller for network safety, including for QUIC DATAGRAM
frames.  QUIC DATAGRAM frames are not retransmitted by QUIC, but they are still
subject to QUIC congestion control and pacing.

An RTMP application that produces media faster than QUIC can send it MUST adapt
its sending behavior.  Application responses include reducing encoder bitrate,
dropping latency-sensitive media messages before they are submitted to QUIC,
switching selected media from streams to DATAGRAM frames, switching selected
media from DATAGRAM frames to streams for recovery, or closing a flow that
cannot be delivered usefully.

Applications SHOULD avoid running an independent media congestion controller
that fights the QUIC congestion controller.  If an application uses media-layer
rate adaptation, it SHOULD use information exposed by the QUIC implementation,
such as congestion window, pacing rate, bytes in flight, stream back pressure,
DATAGRAM send failures, and delivery or loss observations.

When QUIC congestion control delays DATAGRAM transmission, a sender MAY drop an
old DATAGRAM-carried RTMP media message before transmission if sending it later
would harm latency more than losing it.  A sender MUST NOT use this behavior
for RTMP command or control messages required for session correctness.
```

- [ ] **Step 4: Add cross-reference to `Choosing Streams, DATAGRAM Frames, or Both`**

Append:

```markdown
Congestion-control considerations for these choices are described in
{{congestion-control}}.
```

- [ ] **Step 5: Validate and commit**

Run:

```sh
make latest
make check-submission
git diff --check
```

Expected: all commands exit 0. Commit:

```sh
git add draft-gregoire-rtmp-over-quic.md
git commit -m "Add RoQR congestion control guidance"
```

### Task 5: Clarify Timestamp and Chunk Stream Semantics

**Files:**
- Modify: `draft-gregoire-rtmp-over-quic.md`
- Reference: `/media/mondain/terrorbyte/workspace/github-red5pro/roq/src/rtmp.rs`

- [ ] **Step 1: Read RTMP field handling in implementation**

Run:

```sh
sed -n '1,190p' /media/mondain/terrorbyte/workspace/github-red5pro/roq/src/rtmp.rs
```

Expected: timestamp, message stream ID, and chunk stream ID are encoded as QUIC variable-length integers.

- [ ] **Step 2: Add `RTMP Field Semantics` after `RoQR Frame Format`**

Insert:

```markdown
## RTMP Field Semantics {#rtmp-field-semantics}

The Timestamp field carries the RTMP message timestamp in milliseconds after
RTMP chunk processing.  If an implementation receives an RTMP extended
timestamp while translating from chunked RTMP, it encodes the resulting
timestamp value in the RoQR Timestamp field.  RoQR does not carry the RTMP
extended timestamp marker separately.

The Message Stream ID field carries the RTMP message stream identifier
associated with the RTMP message.  The Chunk Stream ID field carries the RTMP
chunk stream identifier associated with the source RTMP message.  RoQR carries
the Chunk Stream ID for applications that preserve RTMP chunk-stream affinity
or use it as part of local RTMP session state.  Applications that do not use
chunk-stream affinity can send a stable application-selected Chunk Stream ID
for messages on a given Flow ID.

The Message Type field is one octet and therefore preserves the RTMP message
type identifier without reinterpretation.  Unknown message type identifiers are
handled as described in {{rtmp-message-type-handling}}.
```

- [ ] **Step 3: Anchor the message type heading**

Change:

```markdown
# RTMP Message Type Handling
```

to:

```markdown
# RTMP Message Type Handling {#rtmp-message-type-handling}
```

- [ ] **Step 4: Validate and commit**

Run:

```sh
make latest
make check-submission
git diff --check
```

Expected: all commands exit 0. Commit:

```sh
git add draft-gregoire-rtmp-over-quic.md
git commit -m "Clarify RoQR RTMP field semantics"
```

### Task 6: Cover 0-RTT, Replay, and Connection Migration

**Files:**
- Modify: `draft-gregoire-rtmp-over-quic.md`
- Reference: `docs/draft-ietf-avtcore-rtp-over-quic-14.txt`

- [ ] **Step 1: Read 0-RTT and migration reference text locally**

Run:

```sh
rg -n "0-RTT|Early Data|Connection Migration|migration|replay" docs/draft-ietf-avtcore-rtp-over-quic-14.txt
```

Expected: the reference identifies 0-RTT, replay, and migration discussion points.

- [ ] **Step 2: Add `0-RTT and Connection Migration` before `Error Handling`**

Insert:

```markdown
# 0-RTT and Connection Migration {#zero-rtt-migration}

RoQR endpoints MAY use QUIC 0-RTT only when the application has determined that
the early data is safe to replay.  RTMP command messages that create sessions,
publish streams, subscribe to streams, authorize users, or mutate server state
SHOULD NOT be sent in 0-RTT unless the application has replay protection for
those commands.

DATAGRAM-carried media sent in 0-RTT can be replayed by an attacker within the
limits of QUIC early data.  Applications that permit media in 0-RTT MUST ensure
that replayed media cannot grant authorization, corrupt persistent state, or
cause unbounded resource consumption.

QUIC connection migration can change the path characteristics available to a
RoQR connection.  After migration, a sender SHOULD treat the available
DATAGRAM size, pacing behavior, and congestion state as path-dependent and
SHOULD be prepared to reduce media rate or switch delivery mode until the new
path is validated by the QUIC implementation.
```

- [ ] **Step 3: Add Security cross-reference**

Append to `Security Considerations`:

```markdown
0-RTT replay risks are described in {{zero-rtt-migration}}.
```

- [ ] **Step 4: Validate and commit**

Run:

```sh
make latest
make check-submission
git diff --check
```

Expected: all commands exit 0. Commit:

```sh
git add draft-gregoire-rtmp-over-quic.md
git commit -m "Describe RoQR 0-RTT and migration behavior"
```

### Task 7: Strengthen RTMP-Specific Security Considerations

**Files:**
- Modify: `draft-gregoire-rtmp-over-quic.md`
- Reference: `docs/draft-ietf-avtcore-rtp-over-quic-14.txt`

- [ ] **Step 1: Read current security sections**

Run:

```sh
sed -n '292,307p' draft-gregoire-rtmp-over-quic.md
sed -n '2656,2672p' docs/draft-ietf-avtcore-rtp-over-quic-14.txt
```

Expected: current RoQR security text is short and needs RTMP-specific concerns.

- [ ] **Step 2: Replace `Security Considerations` body with stronger text**

Use this body under the existing heading:

```markdown
RoQR inherits the security properties of QUIC {{RFC9000}} and QUIC-TLS
{{RFC9001}}.  QUIC encrypts application payloads and authenticates the
transport connection.

RTMP application payloads can contain commands, credentials, authorization
tokens, stream names, metadata, media, and application-specific data.  RoQR
does not add end-to-end object security above QUIC.  Applications that require
end-to-end media, command, or metadata protection across intermediaries need a
separate application-layer protection mechanism.

RTMP command messages can create, publish, subscribe, delete, or otherwise
mutate application state.  Endpoints MUST apply the same authentication,
authorization, and input validation to RTMP commands received over RoQR that
they apply to commands received over other RTMP transports.

Intermediaries that terminate QUIC can observe and modify RTMP command,
metadata, and media payloads unless the application applies end-to-end
protection above QUIC.  Applications that rely on intermediaries MUST define
which intermediaries are trusted for command processing, metadata inspection,
media forwarding, and logging.

DATAGRAM-carried media can be lost without retransmission.  Applications MUST
ensure that loss of DATAGRAM-carried RTMP messages does not cause unsafe parser
state, unbounded buffering, or resource exhaustion.  Receivers MUST bound any
buffering for unknown Flow IDs, incomplete stream frames, and media waiting for
decoder resynchronization.

0-RTT replay risks are described in {{zero-rtt-migration}}.
```

- [ ] **Step 3: Validate and commit**

Run:

```sh
make latest
make check-submission
git diff --check
```

Expected: all commands exit 0. Commit:

```sh
git add draft-gregoire-rtmp-over-quic.md
git commit -m "Strengthen RoQR security considerations"
```

### Task 8: Clean Up idnits Residuals Where Practical

**Files:**
- Modify: `draft-gregoire-rtmp-over-quic.md`

- [ ] **Step 1: Reproduce current idnits warnings**

Run:

```sh
make check-submission
```

Expected before this task: command exits 0 and may report `LINE_PI`, `INVALID_REFERENCES_NAME`, and `NON_ASCII_UTF8`.

- [ ] **Step 2: Investigate references name warning**

Run:

```sh
rg -n "<references|<name>References|<name>Normative|<name>Informative" versioned/draft-gregoire-rtmp-over-quic-00.xml
```

Expected: the generated XML shows a combined `References` wrapper around normative and informative references. If this comes from the template and matches `msfts`, document it in the commit message and do not force a fragile source workaround.

- [ ] **Step 3: Investigate UTF-8 comment**

Run:

```sh
LC_ALL=C rg -n "[^\\x00-\\x7F]" draft-gregoire-rtmp-over-quic.md README.md CONTRIBUTING.md LICENSE.md .github .editorconfig .gitignore Makefile
```

Expected: no source matches. If generated XML still reports `NON_ASCII_UTF8`, treat it as generated reference or boilerplate output unless a source character is found.

- [ ] **Step 4: Decide on LINE_PI**

Run:

```sh
rg -n "<\\?line" versioned/draft-gregoire-rtmp-over-quic-00.xml
```

Expected: line processing instructions are generated by the markdown-to-XML flow. Leave them if removing them would require changing the build pipeline.

- [ ] **Step 5: Add a short note to `README.md` only if a residual warning is confirmed as template-generated**

If all three warnings are template-generated or source-clean, add this under `Submission Checklist`:

```markdown
The first draft currently uses the standard markdown-to-XML workflow from
`i-d-template`.  Residual idnits warnings from generated `<?line ...?>`
processing instructions or generated reference wrappers should be reviewed
before submission but do not indicate source text errors.
```

Do not add this note if an actual source error is found; fix the source error instead.

- [ ] **Step 6: Validate and commit**

Run:

```sh
make latest
make check-submission
git diff --check
```

Expected: all commands exit 0. Commit if the README or draft changed:

```sh
git add README.md draft-gregoire-rtmp-over-quic.md
git commit -m "Document RoQR submission-check residuals"
```

### Task 9: Final Draft Review and Publish Refresh

**Files:**
- Modify if needed: `draft-gregoire-rtmp-over-quic.md`
- Generated by build: `draft-gregoire-rtmp-over-quic.html`
- Generated by build: `draft-gregoire-rtmp-over-quic.txt`
- Branch update: `gh-pages`

- [ ] **Step 1: Review final section order**

Run:

```sh
rg -n "^# |^## " draft-gregoire-rtmp-over-quic.md
```

Expected order:

```text
Introduction
Scope
Conventions and Definitions
Connection Establishment and ALPN
Protocol Operation
Encapsulation
Multiplexing
RoQR Frame Format
RTMP Field Semantics
QUIC Streams
QUIC DATAGRAM Frames
Ordering and Loss Recovery
RTMP Message Type Handling
RTMP Session Mapping
Congestion Control and Rate Adaptation
Choosing Streams, DATAGRAM Frames, or Both
0-RTT and Connection Migration
Error Handling
Security Considerations
IANA Considerations
Registration of the RoQR Identification String
RoQR Error Codes Registry
Implementation Status
Acknowledgments
```

- [ ] **Step 2: Confirm old-name scan is clean**

Run:

```sh
rg -n "draft-gregoire-avtcore-rtmp-over-quic|Audio/Video Transport Core Maintenance|^workgroup:" draft-gregoire-rtmp-over-quic.md README.md CONTRIBUTING.md .github
```

Expected: no matches and exit code 1.

- [ ] **Step 3: Run final validation**

Run:

```sh
make latest
make check-submission
git diff --check
```

Expected: all commands exit 0.

- [ ] **Step 4: Commit final review edits if any**

If Step 1 or Step 3 led to source edits, commit:

```sh
git add draft-gregoire-rtmp-over-quic.md README.md
git commit -m "Polish RoQR first draft"
```

- [ ] **Step 5: Refresh GitHub Pages**

Run:

```sh
make gh-pages
```

Expected: either the template pushes `gh-pages`, or it creates a commit in `/tmp/ghpages*` and fails only at the token-style push.

If token-style push fails locally, import the generated Pages commit and push with the normal remote:

```sh
git fetch /tmp/ghpagesNNNNNN gh-pages:gh-pages
git push origin main gh-pages
git fetch origin gh-pages
git branch -f gh-pages origin/gh-pages
```

Replace `/tmp/ghpagesNNNNNN` with the path shown by the failed `make gh-pages` output.

- [ ] **Step 6: Verify remote state**

Run:

```sh
git ls-remote --heads origin main gh-pages
git ls-tree -r --name-only origin/gh-pages
git status --short --branch
```

Expected:

```text
origin has updated main and gh-pages heads.
origin/gh-pages contains draft-gregoire-rtmp-over-quic.html and draft-gregoire-rtmp-over-quic.txt.
git status shows main tracking origin/main with no local modifications.
```
