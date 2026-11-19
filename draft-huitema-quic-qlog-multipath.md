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

# Event Schema Definition

This document describes how the extend extensions to the core QUIC protocol
defined in {{RFC10069}} can be expressed by extending the schema
defined in {{I-D.ietf-quic-qlog-quic-events}}.
Per the requirements in Section 8 of {{I-D.ietf-quic-qlog-main-schema}},
this document registers extensions to the quic namespace.
The event schema URI is: urn:ietf:params:qlog:events:quic-multipath.


## Draft Event Schema Identification

This section is to be removed before publishing as an RFC.

   Only implementations of the final, published RFC can use the events
   belonging to the event schema with the URI
   urn:ietf:params:qlog:events:quic-multipath.  Until such an RFC exists,
   implementations MUST NOT identify themselves using this URI.

   Implementations of draft versions of the event schema MUST append the
   string "-" and the corresponding draft number to the URI.  For
   example, draft 00 of this document is identified using the URI
   urn:ietf:params:qlog:events:quic-multipath-00.

# Multipath specific events and event extensions

TODO.

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
registry, which was created in Section 15 of [QLOG-MAIN]:

   Event schema URI:  urn:ietf:params:qlog:events:quic-multipath

Event-types: TODO. List the new events that are created here.

Description:  Event definitions related to the extensions to
the QUIC transport protocol defined in {{RFC10069}}.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
