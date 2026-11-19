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

## Add a Path identifier to existing QUIC events

The `path_id` field identifies the QUIC path associated with a qlog event.
It is an event-level custom field, at the same level as `time`, `name`, and
`data`; it is not a member of the event's `data` map. The qlog main schema's
abstract `Event` permits custom event-level fields. This extension assigns
the name and semantics of one such field:

~~~ cddl
PathID = uint32
~~~

The value of `path_id` is the Path ID defined in {{RFC10069}}. It is scoped to
the QUIC connection, is shared by both directions, starts at 0 for the
initial path, increases monotonically, and is not reused. The range is
0..2^32-1, inclusive. The field is unsigned: negative values are invalid.

`path_id` is distinct from qlog's `tuple` / `tuple_id`. A Path ID identifies a
logical QUIC path; multiple Path IDs can use the same 4-tuple, and one Path ID
can use a new 4-tuple after migration. Implementations MUST NOT infer a Path
ID from a tuple identifier.

When a QUIC event describes an operation or state for one known path,
implementations MUST include `path_id`, including when its value is 0. They
MUST omit the field when the event is connection-wide or when the associated
Path ID is not known. Omission does not mean path 0. An implementation MAY
place `path_id` in `common_fields` only when the same value applies to every
event in that trace, following the common-field rules in {{I-D.ietf-quic-qlog-main-schema}}.

The event-level field identifies the path on which the event's packet or
operation applies. A Path ID carried inside a frame remains a frame field and
can identify a different path. For example, a packet sent on path 0 can carry
a PATH_ACK frame acknowledging packets from path 1; the event has
`path_id: 0`, while the PATH_ACK frame has `path_id: 1`.

The field applies to the following existing events in the `quic` namespace
when the stated path context is known:

| Event | Meaning of `path_id` |
| --- | --- |
| `packet_sent`, `packet_received` | Path carrying the packet. |
| `packet_lost` | Path carrying the lost packet. |
| `packet_dropped`, `packet_buffered` | Path associated with the packet, if it can be identified at that point. |
| `packets_acked` | Path of the acknowledged packets, not the path carrying the ACK or PATH_ACK frame. Implementations MUST split events when acknowledged packets belong to different paths. |
| `recovery_metrics_updated`, `recovery_parameters_set` | Path whose recovery or congestion-control values are reported. |
| `congestion_state_updated`, `ecn_state_updated`, `timer_updated` | Path whose state or timer changed. Omit it for a connection-wide update. |
| `mtu_updated`, `spin_bit_updated` | Path whose MTU or spin-bit state changed. |
| `connection_id_updated` | Path associated with the connection ID being updated. |
| `tuple_assigned` | Path only when the tuple assignment is for one specific Path ID; the tuple itself remains identified by `tuple_id`. |
| `migration_state_updated` | Path being migrated. Its Path ID remains the same as the tuple changes during migration. |
| `frames_parsed`, `frames_created` | Path associated with the packet or packets. Implementations MUST split an event if its packet numbers span different paths. Omit `path_id` for frames created before a path is selected. |

Path-management events defined by this document MUST include `path_id` when
they refer to one path. For a frame whose RFC 10069 definition contains a
Path ID, the frame's `data` representation MUST also retain that frame field;
the event-level `path_id` does not replace it. For other QUIC events,
implementations MUST include `path_id` when they describe a single known
path, and MUST omit it when they are connection-wide or the path is unknown.

An example packet event is:

~~~ json
{
  "time": 123.4,
  "name": "quic:packet_sent",
  "path_id": 1,
  "data": {
    "header": {
      "packet_type": "1RTT",
      "packet_number": 0
    },
    "frames": [
      { "frame_type": "stream", "stream_id": 0, "offset": 0 }
    ]
  }
}
~~~

The field is outside `data`, so existing QUIC event data definitions do not
need to be copied or redefined for each event. The table above specifies the
event-by-event semantics and applicability of the common field. New event
types and frame encodings for RFC 10069 remain defined in the sections below.

# Multipath specific frames

TODO.

# Security Considerations {#security}

TODO Security


# IANA Considerations {#iana}


This document registers a new entry in the "qlog event schema URIs"
registry, which was created in Section 15 of [QLOG-MAIN]:

   Event schema URI:  urn:ietf:params:qlog:events:quic-multipath

Event-types: `packet_sent`, `packet_received`, `packet_dropped`,
`packet_buffered`, `packets_acked`, `packet_lost`, `recovery_metrics_updated`,
`recovery_parameters_set`, `congestion_state_updated`, `ecn_state_updated`,
`timer_updated`, `mtu_updated`, `spin_bit_updated`, `connection_id_updated`,
`tuple_assigned`, `migration_state_updated`, `frames_parsed`, and
`frames_created`.

Description:  Event definitions related to the extensions to
the QUIC transport protocol defined in {{RFC10069}}.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
