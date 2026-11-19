---
title: "QLOG extensions for Multiple Paths for a QUIC Connection"
abbrev: "qlog-multipath"
category: info

docname: draft-huitema-quic-qlog-multipath-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "QUIC"
keyword:
 - QUIC
 - multipath
 - qlog
venue:
  group: "QUIC"
  type: "Working Group"
  mail: "quic@ietf.org"
  github: "huitema/qlog-multipath-extension"

author:
 -
    fullname: Christian Huitema
    organization: Private Octopus Inc.
    email: huitema@huitema.net
 -
    fullname: Altanai Bisht
    organization: Cisco Systems
    email: albisht@cisco.com


normative:

   RFC9000:
   RFC10069:
   I-D.ietf-quic-qlog-quic-events:
   I-D.ietf-quic-qlog-main-schema:


informative:

...

--- abstract

This draft define the QLOG elements that have to be added to
the definition of QLOG events for QUIC connections.


--- middle

# Introduction {#intro}

The QLOG drafts define the syntax of QLOG {{I-D.ietf-quic-qlog-main-schema}}
and the events and frames for logging QUIC events {{I-D.ietf-quic-qlog-quic-events}}
found in QUIC connections per {{RFC9000}}. The extensions to {{RFC9000}} for
managing multiple paths for a QUIC connections are defined in {{RFC10069}},
which introduces new concepts such as path, new events such as creating or
abandoning paths, and new frames for managing traffic when using multiple
paths. This draft defines the corresponding QLOG extensions.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Event Schema Definition {#event-schema}

This document extends the `quic` namespace of
{{I-D.ietf-quic-qlog-quic-events}} to cover the multipath extension
defined in {{RFC10069}}. The event schema URN for this extension is
`urn:ietf:params:qlog:events:quic#multipath`. The fragment `multipath`
identifies the extension within the `quic` namespace, as specified in
{{Section 8.1 of I-D.ietf-quic-qlog-main-schema}}.


## Draft Event Schema Identification

This section is to be removed before publishing as an RFC.

Only implementations of the published RFC can use the unversioned URN
defined in {{event-schema}}. Implementations of a draft version MUST
append a hyphen and the corresponding draft number to that URN. For
example, draft 00 uses `urn:ietf:params:qlog:events:quic#multipath-00`.

# Multipath specific events and event extensions {#mp-events}

The three events in this section have Base importance. They record
changes to the paths of a connection as seen by the endpoint. Packet
events record the frames that carry path management signals. A single
change can involve several frames, and a local decision can precede the
first frame that signals it.

Each of these events MUST include the following field in its top-level
`Event` map alongside `time`, `name`, and `data`
({{Section 7 of I-D.ietf-quic-qlog-main-schema}}).

~~~ cddl
MultipathEventFields = (path_id: uint32)
~~~

`path_id` identifies the affected path within the connection. It is an
unsigned 32-bit integer in the range 0 to 4294967295 inclusive
({{Section 2.1 of RFC10069}}).
For example, if a packet on path 0 changes the status of path 1, the
status event has `path_id` 1. The optional qlog `tuple` field identifies
the network tuple associated with an event. Two Path IDs can share a
tuple, and a path can change tuples during migration. Tuple details can
be recorded with `quic:tuple_assigned` ({{I-D.ietf-quic-qlog-quic-events}}).

The event data types extend the qlog main schema as follows. The
optional `error_code` and `sequence_number` fields use the qlog
`uint64` type. Their values are limited to 0 through 2^62 - 1 by
{{Sections 4.2 and 4.3 of RFC10069}}.

~~~ cddl
$ProtocolEventData /= QUICPathCreated / QUICPathAbandoned /
                      QUICPathStatusUpdated
~~~

## path_created

`quic:path_created` is emitted once when the endpoint starts using a Path ID
for a new path. For an additional path, the client logs the event when it
initiates the path and the server logs it when it accepts the attempt and
sets up the path locally ({{Section 3.1 of RFC10069}}). Validation can
still be pending. For the initial path, the endpoint logs the event with
`path_id` 0 when the handshake completes with multipath negotiated.
Issuing connection IDs for future paths alone does not trigger this event.
Migration to a new tuple retains the existing Path ID and can be logged
with `quic:migration_state_updated`.

~~~ cddl
QUICPathCreated = {
    * $$quic-pathcreated-extension
}
~~~

## path_abandoned

`quic:path_abandoned` is emitted once when the endpoint abandons the
indicated Path ID under {{Section 3.4 of RFC10069}}. The `initiator` is
`local` when the endpoint decides to close the path and `remote` when it
acts on the peer's PATH_ABANDON. These values use the `Initiator` type
from {{I-D.ietf-quic-qlog-quic-events}}. If known, `error_code` records
the numeric PATH_ABANDON error code for that decision.

An unused Path ID can be abandoned without a preceding `path_created`.
A repeated or reciprocal PATH_ABANDON and the eventual release of retained
path state do not produce additional abandonment events.

~~~ cddl
QUICPathAbandoned = {
    initiator: Initiator
    ? error_code: uint64 .le 4611686018427387903
    * $$quic-pathabandoned-extension
}
~~~

## path_status_updated

`quic:path_status_updated` is emitted when the endpoint changes the path
usage preference it advertises (`initiator` is `local`), or records a
change in the peer's advertised preference (`initiator` is `remote`).
Each endpoint expresses its own preference for how its peer should use
the path ({{Section 3.3 of RFC10069}}). A sender can choose how to
schedule packets independently of the advertised preference.

`new` gives the updated preference. If included, `old` gives the previous
preference for the same Path ID and initiator. Omit `old` when no previous
preference is known. The first advertisement can precede path creation.
`sequence_number`, if present, is the Path Status Sequence Number of the
advertisement. An ignored out-of-order or duplicate advertisement, or a
newer advertisement that repeats the current preference, produces no
status change event ({{Section 4.3 of RFC10069}}).

~~~ cddl
QUICPathStatusUpdated = {
    initiator: Initiator
    ? old: QUICPathStatus
    new: QUICPathStatus
    ? sequence_number: uint64 .le 4611686018427387903
    * $$quic-pathstatusupdated-extension
}

QUICPathStatus = "available" / "backup"
~~~

## Example

In this excerpt from a client's event list, the client opens path 1.
It then receives the peer's first preference for path 1 in a packet on
path 0. Later the client abandons path 1 at the application's request.

~~~ json
[
  { "time": 12.0, "name": "quic:path_created",
    "path_id": 1, "data": {} },
  { "time": 18.0, "name": "quic:path_status_updated",
    "path_id": 1, "data": { "initiator": "remote", "new": "backup",
                            "sequence_number": 0 } },
  { "time": 97.0, "name": "quic:path_abandoned",
    "path_id": 1, "data": { "initiator": "local", "error_code": 62 } }
]
~~~

# Multipath specific frames

The following CDDL extends the `$QuicFrame` type socket defined in
{{Section 8.13 of I-D.ietf-quic-qlog-quic-events}}. These frame objects
can appear wherever a QUIC event records frames, including the `frames`
array of `quic:packet_sent` and `quic:packet_received`. The optional
`raw` field uses `RawInfo` from {{I-D.ietf-quic-qlog-main-schema}}.

Within a frame, `path_id` is the Path Identifier carried on the wire.
For example, a PATH_ACK sent on path 0 can acknowledge packets on path 1.
In that case the PATH_ACK object contains `"path_id": 1`. Path IDs and
path ID limits are unsigned 32-bit values. QUIC variable-length integer
fields are logged as decoded values within the limits of {{RFC10069}}.

~~~ cddl
$QuicFrame /= PathAckFrame / PathAbandonFrame / PathStatusFrame /
              PathNewConnectionIDFrame / PathRetireConnectionIDFrame /
              MaxPathIDFrame / PathsBlockedFrame / PathCIDsBlockedFrame
~~~

## PATH_ACK

Both wire types in {{Section 4.1 of RFC10069}} use the frame type
`path_ack`. The `path_id` selects the packet number space being
acknowledged. As in the base `AckFrame`, `ack_delay` is expressed in
milliseconds after applying the ACK delay exponent. The `acked_ranges`
field contains decoded, inclusive packet number ranges. For example,
`[[40, 42], [48]]` acknowledges packets 40 through 42 and packet 48.

For wire type 0x3e, omit the ECN counts. For 0x3f, include all three
cumulative counts for the acknowledged path, including counts of zero.

~~~ cddl
PathAckFrame = { PathAckFields } /
               { PathAckFields, ect0: uint64, ect1: uint64, ce: uint64 }

PathAckFields = (
    frame_type: "path_ack"
    path_id: uint32
    ? ack_delay: float32
    ? acked_ranges: [+ AckRange]
    ? raw: RawInfo
)
~~~

## PATH_ABANDON

For PATH_ABANDON (0x3e75), `path_id` identifies the path being closed.
The `error` field uses qlog transport error names, including `no_error`
for code 0. The four additional names below correspond to the codes in
{{Section 4.2.1 of RFC10069}}. For a code with no known name, log
`"error": "unknown"` and include its numeric value in `error_code`.

~~~ cddl
PathAbandonFrame = {
    frame_type: "path_abandon"
    path_id: uint32
    error: $TransportError / CryptoError
    ? error_code: uint64
    ? raw: RawInfo
}

$TransportError /= "application_abandon_path" /      ; 0x3e
                   "path_resource_limit_reached" /   ; 0x3e75
                   "path_unstable_or_poor" /         ; 0x3e76
                   "no_cid_available_for_path"       ; 0x3e77
~~~

## PATH_STATUS_AVAILABLE and PATH_STATUS_BACKUP

The `frame_type` identifies the preference advertised for `path_id`.
Use `path_status_available` for 0x3e77 and `path_status_backup` for 0x3e76.
Both use the same per-path status sequence number space
({{Section 4.3 of RFC10069}}). The frame object records the received
sequence number even when the receiver ignores an outdated advertisement.

~~~ cddl
PathStatusFrame = {
    frame_type: "path_status_available" / "path_status_backup"
    path_id: uint32
    sequence_number: uint64
    ? raw: RawInfo
}
~~~

## PATH_NEW_CONNECTION_ID and PATH_RETIRE_CONNECTION_ID

These frames use the fields of the base `NewConnectionIDFrame` and
`RetireConnectionIDFrame`, with an additional `path_id`. Both the
connection ID sequence number and the retirement threshold apply to the
indicated path ({{Sections 4.4 and 4.5 of RFC10069}}). For
PATH_NEW_CONNECTION_ID (0x3e78), `connection_id_length`, when present,
is the length in bytes of the connection ID on the wire. For
PATH_RETIRE_CONNECTION_ID (0x3e79), the pair (`path_id`,
`sequence_number`) identifies the connection ID to retire.

~~~ cddl
PathNewConnectionIDFrame = {
    frame_type: "path_new_connection_id"
    path_id: uint32
    sequence_number: uint64
    retire_prior_to: uint64
    ? connection_id_length: uint8
    connection_id: ConnectionID
    ? stateless_reset_token: StatelessResetToken
    ? raw: RawInfo
}

PathRetireConnectionIDFrame = {
    frame_type: "path_retire_connection_id"
    path_id: uint32
    sequence_number: uint64
    ? raw: RawInfo
}
~~~

## MAX_PATH_ID, PATHS_BLOCKED, and PATH_CIDS_BLOCKED

MAX_PATH_ID (0x3e7a) advertises the largest acceptable Path ID in
`maximum`. PATHS_BLOCKED (0x3e7b) reports the Path ID limit in effect
at the sender in `limit`. Both fields represent the Maximum Path
Identifier on the wire. PATH_CIDS_BLOCKED (0x3e7c) identifies the path
that lacks an unused connection ID and reports the next connection ID
sequence number expected from the peer
({{Sections 4.6 and 4.7 of RFC10069}}).

~~~ cddl
MaxPathIDFrame = {
    frame_type: "max_path_id"
    maximum: uint32
    ? raw: RawInfo
}

PathsBlockedFrame = {
    frame_type: "paths_blocked"
    limit: uint32
    ? raw: RawInfo
}

PathCIDsBlockedFrame = {
    frame_type: "path_cids_blocked"
    path_id: uint32
    next_sequence_number: uint64
    ? raw: RawInfo
}
~~~

## Example

In this packet sent on path 0, the PATH_ACK acknowledges packets on
path 1 and the PATH_CIDS_BLOCKED reports that the sender has run out
of connection IDs for path 2.

~~~ json
{
  "time": 16.5,
  "name": "quic:packet_sent",
  "data": {
    "header": { "packet_type": "1RTT", "packet_number": 90 },
    "frames": [
      { "frame_type": "path_ack", "path_id": 1,
        "acked_ranges": [[40, 42], [48]],
        "ect0": 4, "ect1": 0, "ce": 0 },
      { "frame_type": "path_cids_blocked", "path_id": 2,
        "next_sequence_number": 3 }
    ]
  }
}
~~~

# Security Considerations {#security}

TODO Security


# IANA Considerations {#iana}


This document registers a new entry in the "qlog event schema URIs"
registry defined in {{Section 15 of I-D.ietf-quic-qlog-main-schema}}.

Event schema URI:  `urn:ietf:params:qlog:events:quic#multipath`

Namespace:  `quic`

Event Types:  `path_created`, `path_abandoned`, `path_status_updated`

Description:  Event definitions related to the extensions to
the QUIC transport protocol defined in {{RFC10069}}.

Reference:  {{mp-events}} of this document.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
