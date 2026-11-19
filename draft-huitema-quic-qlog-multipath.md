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

TODO.

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
