---
title: Multipath Extension for QUIC
abbrev: Multipath QUIC
docname: draft-ietf-quic-multipath-latest
date: {DATE}
category: std

ipr: trust200902
area: Transport
workgroup: QUIC Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
-
  fullname:
      :: "刘彦梅"
      ascii: "Yanmei Liu"
  role: editor
  org: Alibaba Inc.
  email: miaoji.lym@alibaba-inc.com
-
   fullname:
     :: "马云飞"
     ascii: "Yunfei Ma"
   org: Uber Technologies Inc.
   email: yunfei.ma@uber.com
-
   ins: Q. De Coninck
   name: Quentin De Coninck
   role: editor
   org: University of Mons (UMONS)
   email: quentin.deconinck@umons.ac.be
-
   ins: O. Bonaventure
   name: Olivier Bonaventure
   org: UCLouvain and Tessares
   email: olivier.bonaventure@uclouvain.be
-
   ins: C. Huitema
   name: Christian Huitema
   org: Private Octopus Inc.
   email: huitema@huitema.net
-
   ins: M. Kuehlewind
   name: Mirja Kuehlewind
   role: editor
   org: Ericsson
   email: mirja.kuehlewind@ericsson.com


normative:
  QUIC-TRANSPORT: rfc9000
  QUIC-TLS: rfc9001
  QUIC-RECOVERY: rfc9002

informative:
  RFC6356:
  OLIA:
    title: "MPTCP is not pareto-optimal: performance issues and
a possible solution"
    date: "2012"
    seriesinfo: "Proceedings of the 8th international conference on
Emerging networking experiments and technologies, ACM"
    author:
    -
      ins: R. Khalili
    -
      ins: N. Gast
    -
      ins: M. Popovic
    -
      ins: U. Upadhyay
    -
      ins: J.-Y. Le Boudec


--- abstract

This document specifies a multipath extension for the QUIC protocol to
enable the simultaneous usage of multiple paths for a single connection.

--- middle

# Introduction

This document specifies an extension to QUIC version 1 {{QUIC-TRANSPORT}}
to enable the simultaneous usage of multiple paths for a single
connection. This contrasts with
the base QUIC protocol {{QUIC-TRANSPORT}} that includes a connection migration mechanism that
selects only one path at a time to exchange such packets.

The path management specified in {{Section 9 of QUIC-TRANSPORT}}
fulfills multiple goals: it directs a peer to switch sending through
a new preferred path, and it allows the peer to release resources
associated with the old path. The multipath extension specified in this document requires
several changes to that mechanism:

  *  Simultaneous transmission on multiple paths.
  *  Introduction of a path identifier to manage connection IDs and
     packet number spaces per path.
  *  Removal of paths that have been abandoned.

As such, this extension specifies a departure from the specification of
path management in {{Section 9 of QUIC-TRANSPORT}} and therefore
requires negotiation between the two endpoints using a new transport
parameter, as specified in {{nego}}. Further, as different packet number
spaces are used for each path, this specification requires the use of
non-zero connection IDs in order to identify the path and respective
packet number space.

To add a new path to an existing QUIC connection with multipath support,
a client sends a packet with a connection ID belonging to a so-far unused Path ID on
the chosen path, as further described in {{path-initiation}}.
A new path can only be used once the associated 4-tuple has been validated
by ensuring that the peer is able to receive packets at that address
(see {{Section 8 of QUIC-TRANSPORT}}).
In this version of the document, a QUIC server does not initiate the creation
of a path, but it can validate a new path created by a client.

Each endpoint may use several IP addresses for the connection. In
particular, the multipath extension supports the following scenarios.

  * The client uses multiple IP addresses and the server listens on only
    one.
  * The client uses only one IP address and the server listens on
    several ones.
  * The client uses multiple IP addresses and the server listens on
    several ones.
  * The client uses only one IP address and the server
    listens on only one.

Note that in the last scenario, it still remains possible to have
multiple paths over the connection, given that a path is not only
defined by the IP addresses being used, but also the port numbers.
In particular, the client can use one or several ports per IP
address and the server can listen on one or several ports per IP
address.

In addition to these core features, an application using the multipath extension will typically
need additional algorithms to handle multiple, simultaneously open paths and how they are used to
send packets. As these differ depending on the application's requirements,
this proposal only specifies a simple basic packet
scheduling algorithm (see {{packet-scheduling}}),
in order to provide some basic implementation
guidance. However, more advanced algorithms as well as potential
extensions to enhance signaling of the current path status are expected
as future work.

Further, this proposal does also not cover address discovery and management. Addresses
and the actual decision process to setup or tear down paths are assumed
to be handled by the application that is using the QUIC multipath
extension. This is sufficient to address the first aforementioned
scenario. However, this document does not prevent future extensions from
defining mechanisms to address the remaining scenarios.

## Basic Design Points

This proposal is based on several basic design points:

  * Re-use as much as possible mechanisms of QUIC version 1. In
particular, this proposal uses path validation as specified for QUIC
version 1 and aims to re-use as much as possible of QUIC's connection
migration.
  * Use the same packet header formats as QUIC version 1 to minimize
    the difference between multipath and non-multipath traffic being
    exposed on wire.
  * Congestion Control must be per-path (following {{QUIC-TRANSPORT}})
which usually also requires per-path RTT measurements
  * PMTU discovery should be performed per-path
  * The use of this multipath extension requires the use of non-zero
length connection IDs in both directions.
  * Connection IDs are associated with a path ID. The path initiation
associates that path ID with a 4-tuple of source and destination IP
address as well as source and destination port.
  * Migration is detected without ambiguity
when a packet arrives with a connection ID
pointing to an existing path ID, but the connection ID and/or the
4-tuple are different from the value currently used for that path.
  * Paths can be closed at any time, as specified in {{path-close}}.
  * It is possible to create multiple paths sharing the same 4-tuple.
Each of these paths can be closed at any time, like any other path.

Further the design of this extension introduces an explicit path identifier
and use of multiple packet number spaces as further explained in the next sections.

## Introduction of an Explicit Path Identifier {#explicit-path-id}

This extension specifies a new path identifier (Path ID), which is an
integer between 0 and 2^32-1 (inclusive). Path identifies are generated
monotonically increasing and cannot be reused.
The connection ID of a packet binds the packet to a path identifier, and therefore
to a packet number space.

The same Path ID is used in both directions to
address a path in the new multipath control frames,
such as PATH_ABANDON {{path-abandon-frame}}, PATH_STANDBY {{path-backup-available-frame}}},
PATH_AVAILABLE {{path-backup-available-frame}} as well as PATH_ACK {{mp-ack-frame}}.
Further, connection IDs are issued per Path ID using the
PATH_NEW_CONNECTION_ID frame (see {{mp-new-conn-id-frame}}).
That means each connection ID is associated with exactly one path identifier
but multiple connection IDs are usually issued for each path identifier.

The Path ID of the initial path is 0. Connection IDs
which are issued by a NEW_CONNECTION_ID frame {{Section 19.15. of QUIC-TRANSPORT}}
respectively are associated with Path ID 0. Also, the Path ID for
the connection ID specified in the "preferred address" transport parameter is 0.
Use of the "preferred address" is considered as a migration event
that does not change the Path ID.

## Use of Multiple Packet Number Spaces

This extension uses multiple packet number spaces, one for each path.
As such, each path maintains distinct packet number states for sending and receiving packets, as in {{QUIC-TRANSPORT}}.
Using multiple packet number spaces enables direct use of the
loss recovery and congestion control mechanisms defined in
{{QUIC-RECOVERY}} on a per-path basis.

Each Path ID-specific packet number space starts at packet number 0. When following
the packet number encoding algorithm described in {{Section A.2 of QUIC-TRANSPORT}},
the largest packet number (largest_acked) that has been acknowledged by the
peer in the Path ID-specific packet number space is initially set to "None".

Using multiple packet number spaces requires changes in the way AEAD is
applied for packet protection, as explained in {{multipath-aead}}.
More concretely, the Path ID is used to construct the
packet protection nonce in addition to the packet number
in order to enable use of the same packet number on different paths.
Respectively, the Path ID is limited to 32 bits to ensure a unique nonce.
Additional consideration on key updates are explained in {{multipath-key-update}}.


## Conventions and Definitions {#definition}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in BCP 14
{{!RFC2119}} {{!RFC8174}} when, and only when, they appear in all
capitals, as shown here.

We assume that the reader is familiar with the terminology used in
{{QUIC-TRANSPORT}}. When this document uses the term "path", it refers
to the notion of "network path" used in {{QUIC-TRANSPORT}}.


# Handshake Negotiation and Transport Parameter {#nego}

This extension defines a new transport parameter, used to negotiate
the use of the multipath extension during the connection handshake,
as specified in {{QUIC-TRANSPORT}}. The new transport parameter is
defined as follows:

- initial_max_path_id (current version uses 0x0f739bbc1b666d0c): a
  variable-length integer specifying the maximum path identifier
  an endpoint is willing to maintain at connection initiation.
  This value MUST NOT exceed 2^32-1, the maximum allowed value for the Path ID due to
  restrictions on the nonce calculation (see {{multipath-aead}}).

For example, if initial_max_path_id is set to 1, only connection IDs
associated with Path IDs 0 and 1 should be issued by the peer.
If an endpoint receives an initial_max_path_id transport parameter with value 0,
the peer aims to  enable the multipath extension without allowing extra paths immediately.

If an initial_max_path_id transport parameter value that is higher than 2^32-1
is received, the receiver MUST close the connection with an error of type
TRANSPORT_PARAMETER_ERROR.

Setting initial_max_path_id parameter is equivalent to sending a
MAX_PATH_ID frame ({{max-paths-frame}}) with the same value.
As such to allow for the use of more paths later,
endpoints can send the MAX_PATH_ID frame to increase the maximum allowed path identifier.

If either of the endpoints does not advertise the initial_max_path_id transport
parameter, then the endpoints MUST NOT use any frame or
mechanism defined in this document.

When advertising the initial_max_path_id transport parameter, the endpoint
MUST use non-zero length Source and Destination Connection IDs.
If an initial_max_path_id transport
parameter is received and the carrying packet contains a zero-length
connection ID, the receiver MUST treat this as a connection error of type
PROTOCOL_VIOLATION and close the connection.

The initial_max_path_id parameter MUST NOT be remembered
for use in a subsequent connection ({{Section 7.4.1 of QUIC-TRANSPORT}}).
New paths can only be used after handshake completion.

This extension does not change the definition of any transport parameter
defined in {{Section 18.2. of QUIC-TRANSPORT}}.

The initial_max_path_id transport parameter limits the initial maximum number of open paths
that can be used during a connection.

The active_connection_id_limit transport parameter
{{QUIC-TRANSPORT}} limits the maximum number of active connection IDs per path when the
initial_max_path_id parameter is negotiated successfully.
As defined in {{Section 5.1.1 of QUIC-TRANSPORT}} connection IDs that are issued and not retired are considered active.
Endpoints might prefer to retain spare connection IDs so that they can
respond to unintentional migration events ({{Section 9.5 of QUIC-TRANSPORT}}).

Cipher suites with a nonce shorter than 12 bytes cannot be used together with
the multipath extension. If such a cipher suite is selected and the use of the
multipath extension is negotiated, endpoints MUST abort the handshake with a
an error of type TRANSPORT_PARAMETER_ERROR.

The PATH_ACK frame, as specified in {{mp-ack-frame}}, is used to
acknowledge 1-RTT packets.
Compared to the ACK frame as specified in {{Section 19.3 of QUIC-TRANSPORT}}, the PATH_ACK frame additionally
contains the receiver's Path ID to identify the path-specific packet number space.

As multipath support is unknown during the handshake, acknowledgments of
Initial and Handshake packets are sent using ACK frames.
If the multipath extension has been successfully
negotiated, ACK frames in 1-RTT packets acknowledge packets for the path with
Path ID 0.

After the handshake concluded if negotiation of multipath support succeeded,
endpoints SHOULD use PATH_ACK frames instead of ACK frames,
including for acknowledging so far unacknowledged 0-RTT packets using Path ID 0.
Endpoints MUST process ACK frames that acknowledge 0-RTT packets or 1-RTT packets
for Path ID 0, even after successful negotiation of this extension. For example,
ACK frames might be preferred by the sender as long as only the initial path
with Path ID 0 is in use.
Similarly after a successful handshake, endpoints SHOULD also use
the PATH_NEW_CONNECTION_ID frame to provide new connection IDs for Path ID 0 and,
respectively, the PATH_RETIRE_CONNECTION_ID frame to retire connection IDs for
Path ID 0.

# Path Management {#path-management}

After completing the handshake, endpoints have agreed to enable
multipath support. They can also start using multiple paths when both endpoints
have issued available connection IDs for at least one unused Path ID.
If an endpoint receives a disable_active_migration transport parameter
provided by the peer, it is forbidden to use a new local address
to establish new paths to the peer's handshake address. However,
establishment of additional paths from any local address to other peer addresses
(e.g carried by peer’s preferred_address) is valid immediately.

This document
does not specify how an endpoint that is reachable via several addresses
announces these addresses to the other endpoint. In particular, if the
server uses the preferred_address transport parameter, clients
cannot assume that the initial server address and the addresses
contained in this parameter can be simultaneously used for multipath
({{Section 9.6.2 of QUIC-TRANSPORT}}).
Furthermore, this document
does not discuss when a client decides to initiate a new path. We
delegate such discussion to separate documents.

To open a new path, an endpoint MUST use a connection ID associated with
a new, unused Path ID. To let the peer open a new path, an endpoint needs
to provide its peer with connection IDs for at least one unused path identifier.
Still, the receiver may observe a connection ID associated with a used Path ID
on different 4-tuples due to, e.g., NAT rebinding. In such a case, the receiver reacts
as specified in {{Section 9.3 of QUIC-TRANSPORT}} by initiating path validation
but MUST use a new connection ID for the same Path ID.

This proposal adds six multipath control frames for path management:

- PATH_ABANDON frame for the receiver side to abandon the path
(see {{path-abandon-frame}}),
- PATH_BACKUP and PATH_AVAILABLE frames to express a preference
in path usage (see {{path-backup-available-frame}}), and
- MAX_PATH_ID frame (see {{max-paths-frame}}) for increasing the limit of
path identifiers,
- PATHS_BLOCKED and PATH_CID_BLOCKED frames (see {{paths-and-cids-blocked-frame}})
to notify the peer of being blocked to open new paths as
the limit of active paths set by the peer has been reached
or there are no unused connection IDs available
for the corresponding Path ID.

All new frames are sent in 1-RTT packets {{QUIC-TRANSPORT}}.

## Path Initiation {#path-initiation}

Opening a new path requires the use of a new connection ID (see {{Section 9.5 of QUIC-TRANSPORT}})
mapped to an unused Path ID (see {{explicit-path-id}}).
Instead of NEW_CONNECTION_ID frame as specified in {{Section 19.15 of QUIC-TRANSPORT}},
each endpoint uses the PATH_NEW_CONNECTION_ID frame as specified in this extension
to issue Path ID-specific connections IDs.
The same Path ID is used in both directions. As such to open
a new path, both sides need at least
one connection ID (see {{Section 5.1.1 of QUIC-TRANSPORT}}), which is associated
with the same, unused Path ID. When the peer receives the PATH_CHALLENGE,
it MUST pick a Connection ID with the same Path ID for sending the PATH_RESPONSE.

When the multipath extension is negotiated, a client that wants to use an
additional path MUST validate the peer's address before sending any data packets
as described in ({{Section 8.2 of QUIC-TRANSPORT}}),
unless it has previously validated the 4-tuple used for that path.

After receiving packets from the
client on a new path, if the server decides to use the new path,
the server MUST validate the peer's address before sending any data packets
as described in ({{Section 8.2 of QUIC-TRANSPORT}}),
unless it has previously validated the 4-tuple used for that path.
Until the client's address is
validated, the anti-amplification limit from {{Section 8 of QUIC-TRANSPORT}}
applies.

The server may receive packets for a yet unused Path ID that do not
contain a path challenge. Such packets are valid if they can be properly decrypted
and if they contain a valid connection ID.

Each endpoint MUST also validate that a minimum MTU of 1200 bytes is supported
on the path. This can be done during initial path validation or separately later if
the amplification limit prevents it initially, as specified in Section 8.2.1 of RFC9000.

An endpoint that receives packets on a new path and does not want to establish
this path is expected to close the path by sending a PATH_ABANDON
on another path, as specified in section {{path-close}}.

An endpoint that has no active connection ID for this path or
lacks other resource to immediately configure a new path could
delay sending the PATH_RESPONSE until sufficient resource are available.
Long delays may cause the peer to repeat the PATH_CHALLENGE and eventually
send a PATH_ABANDON, in which case the procedures specified in
Section {{path-close}} apply.

PATH_ACK frames (defined in {{mp-ack-frame}}) can be returned on any path.
If the PATH_ACK is preferred to be sent on the same path as the acknowledged
packet (see {{compute-rtt}} for further guidance), it can be beneficial
to bundle a PATH_ACK frame with the PATH_RESPONSE frame during
path validation.

If validation succeeds, the client can continue to use the path.
If validation fails, the client MUST NOT use the path and can
remove any status associated to the path initiation attempt.
However, as the used Path ID is anyway consumed,
the endpoint MUST explicitly close the path as specified in
{{path-close}}.

{{Section 9.1 of QUIC-TRANSPORT}} introduces the concept of
"probing" and "non-probing" frames. A packet that contains at least
one "non-probing" frame is a "non-probing" packet. When the multipath extension
is negotiated, the reception of a "non-probing"
packet on a new path with a new so far unused Path ID
does not impact the path status of any existing
path. Therefore, any frame can be sent on a new path at any time
as long as the anti-amplification limits
({{Section 21.1.1.1 of QUIC-TRANSPORT}}) and the congestion control
limits for this path are respected.

Connection ID changes as specified in {{Section 5.1.2 of QUIC-TRANSPORT}} and
in {{Section 9 of QUIC-TRANSPORT}} for connection migration apply for
connection IDs associated to the same Path ID.
With the successful negotiation of the extension specified
in this draft, endpoints have to consider the reception of
a packet with a connection ID associated to an
so far unused Path ID as an attempt to establish a new path.

As specified in {{Section 9.3 of QUIC-TRANSPORT}}, the server is expected to send a new
address validation token to a client following the successful validation of a
new client address. The client will receive several tokens. When considering using a token
for subsequent connections, it might be difficult for the client
to pick the "right" token among multiple tokens obtained in a previous connection.
The client is likely to fall back to the strategy specified in {{Section 8.1.3 of QUIC-TRANSPORT}},
i.e., pick the last received token. To avoid issues when clients make the "wrong" choice,
a server should issue a token that is capable of validating
any of the previously validated addresses. Further guidance on token usage can be
found in {{Section 8.1.3 of QUIC-TRANSPORT}}.

## Path Status Management

An endpoint uses the PATH_BACKUP and PATH_AVAILABLE frames to inform the peer that it should
send packets with the preference expressed by these frames.
Note that an endpoint might not follow the peer’s advertisements,
but these frames are still a clear signal of the peer's preference of path usage.
Each peer indicates its preference of path usage independently of the other peer.
That means that peers may have different usage preferences for the same path.
Depending on the data sender's decisions, this may lead to usage of paths that have been
indicated as "standby" by the peer or non-usage of some locally available paths.

PATH_AVAILABLE indicates that a path is "available", i.e., it suggests to
the peer to use its own logic to split traffic among available paths.

PATH_BACKUP marks a path as "standby", i.e., it suggests that no traffic
should be sent on that path if another path is available.
If all established paths are marked as "standby", no guidance is provided about
which path should be used.

If an endpoint starts using a path that was marked as "standby" by its peer
because it has detected issues on the paths marked as "available", it is RECOMMENDED
to update its own path state signaling such that the peer avoids using the broken path.
An endpoint that detects a path breakage can also explicitly close the path
by sending a PATH_ABANDON frame (see {{path-close}}) in order to avoid
that its peer keeps using it and enable faster switch over to a standby path.
If the endpoints do not want to close the path immediately, as connectivity
could be re-established, PING frames can potentially be used to quickly detect
connectivity changes and switch back in a timely way.

If no frame indicating a path usage preference was received for a certain path,
the preference of the peer is unknown and the sender needs to decide based on it
own local logic if the path should be used.

Endpoints use the Path ID
in these frames to identify which path state is going to be
changed. Note that both frames can be sent via a different path
and therefore might arrive in different orders.
The PATH_AVAILABLE and PATH_BACKUP frames share a common sequence number space
to detect and ignore outdated information.

## Path Close {#path-close}

Each endpoint manages the set of paths that are available for
transmission. At any time in the connection, each endpoint can decide to
abandon one of these paths, for example following changes in local
connectivity or local preferences. After an endpoint abandons
a path, the peer can expect to not receive any more packets on
that path.

Note that other explicit closing mechanisms of {{QUIC-TRANSPORT}} still
apply on the whole connection. In particular, the reception of either a
CONNECTION_CLOSE ({{Section 10.2 of QUIC-TRANSPORT}}) or a Stateless
Reset ({{Section 10.3 of QUIC-TRANSPORT}}) closes the connection.

An endpoint that wants to close a path MUST explicitly
terminate the path by sending a PATH_ABANDON frame.
Note that while abandoning a path will cause
connection ID retirement, the inverse is not true: retiring the associated connection IDs
does not indicate path abandonment (see further {{consume-retire-cid}}).
This is true whether the decision to close the path results
from implicit signals such as idle time or packet losses
(see {{idle-time-close}}) or for any other reason, such as management
of local resources. It is also possible to abandon a path for which no
packet has been sent (see {{abandon-early}}).

When an endpoint receives a PATH_ABANDON frame, it MUST send a corresponding
PATH_ABANDON frame if it has not already done so. It MUST stop sending
any new packet on the abandoned path, and it MUST treat all
connection identifiers received from the peer for that path as immediately
retired. However, knowledge of the
connection identifiers received from the peer and of the state
of the number space associated to the path SHOULD be retained while
packets from the peer might still be in transit, i.e., for a delay of
3 PTO after the PATH_ABANDON frame has been received from the peer,
both to avoid generating spurious stateless packets as specified in
{{spurious-stateless-reset}} and to be able to acknowledge the
last packets received from the peer as specified in {{ack-after-abandon}}.

If a peer sends a PATH_ABANDON frame but never receives
a corresponding PATH_ABANDON frame, it might not be able to remove path state.
It is left to the implementation to handle this unexpected
behavior as it does not impact interoperability. If the endpoint is no longer
willing to process the issued connection IDs for the abandoned path,
it MAY close the connection, but SHOULD wait at least 3 PTOs after
sending the PATH_ABANDON frame.

After receiving or sending a PATH_ABANDON frame, the endpoints SHOULD
promptly send PATH_ACK frames to acknowledge all packets received on
the path and not yet acknowledged, as specified in {{ack-after-abandon}}.
When an endpoint finally deletes all resource associated with the path,
the packets sent over the path and not yet acknowledged MUST be considered lost.

After a path is abandoned, the Path ID MUST NOT be reused
for new paths, as the Path ID is part of the nonce calculation {{multipath-aead}}.

PATH_ABANDON frames can be sent on any path,
not only the path that is intended to be closed.
It is RECOMMENDED to send the PATH_ABANDON frames on another path,
especially if connectivity on the to-be-abandoned path
is expected to be broken.

If a PATH_ABANDON frame is received for the only open path of a QUIC
connection, the receiving peer SHOULD send a CONNECTION_CLOSE frame
and enter the closing state. If the client received a PATH_ABANDON
frame for the last open path, it MAY instead try to open a new path, if
available, and only initiate connection closure if path validation fails
or a CONNECTION_CLOSE frame is received from the server. Similarly
the server MAY wait for a short, limited time such as one PTO if a path
probing packet is received on a new path before sending the
CONNECTION_CLOSE frame.

### Avoiding Spurious Stateless Resets {#spurious-stateless-reset}

The peers that send a PATH_ABANDON frame MUST treat all connection
identifiers received from the peer for the Path ID as immediately
retired. The Stateless Reset Tokens associated with these connection
identifiers MUST NOT be used to identify Stateless Reset packets
per {{Section 10.3 of QUIC-TRANSPORT}}.

Due to packet losses and network delays, packets sent on the path may
well arrive after the PATH_ABANDON frames have been sent or received.
If these packets arrive after the connection identifiers sent to the peer
have been retired, they will not be recognized as bound for the local
connection and could trigger the peer to send a Stateless Reset
packet. The rule to "retain knowledge of connection ID for 3 PTO
after receiving a PATH_ABANDON"
is intended to reduce the risk of sending such spurious stateless
packets, but it cannot completely avoid that risk.

The immediate retirement of connection identifiers received for the
path guarantees that spurious stateless reset packets
sent by the peer will not cause the closure of the QUIC connection.

### Handling PATH_ACK for abandoned paths {#ack-after-abandon}

When an endpoint decides to send a PATH_ABANDON frame, there may
still be some unacknowledged packets. Some other packets may well
be in transit, and could be received shortly after sending the
PATH_ABANDON frame. As specified above, the endpoints SHOULD
send PATH_ACK frames promptly, to avoid unnecessary data
retransmission after the peer deletes path resources.

These PATH_ACK frames MUST be sent on a different path than the
path being abandoned.

PATH_ACK frames received after the endpoint has entirely deleted
a path MUST be silently discarded.

### Idle Timeout {#idle-time-close}

{{QUIC-TRANSPORT}} allows for closing of connections if they stay idle
for too long. The connection idle timeout when using the multipath extension is defined
as "no packet received on any path for the duration of the idle timeout".
When only one path is available, servers MUST follow the specifications
in {{QUIC-TRANSPORT}}.

This document does not specify per-path idle timeouts. An endpoint
can decide to close a path at any time, whether the path is in active
use or not, by sending a PATH_ABANDON frame. It is not required
to send a PATH_ABANDON frame at any specific point in time.
For example, an endpoint may wait until it will anyway send another frame.

If a path is not actively used for a while, it might not be usable anymore,
e.g. due to middlebox timeouts. To avoid such path breakage, endpoints
can send ack-eliciting packets such as packets containing PING frames
({{Section 19.2 of QUIC-TRANSPORT}}) on that path to keep it alive.
As discussed in
{{Section 10.1.2 of QUIC-TRANSPORT}}, the keep-alive interval depends
on the timeout in the middlebox.

If a path was not actively used for a while, an endpoint can
probe it before switching to active use if there are still other paths
that are currently usable.

### Early Abandon {#abandon-early}

There are scenarios in which an endpoint will receive a PATH_ABANDON frame
before receiving or sending any traffic on a path. For example, if the client
tries to initiate a path and the path cannot be established, it will send a
PATH_ABANDON frame (see {{path-initiation}}). An endpoint may also decide
to abandon a path for any reason, for example, removing a hole from
the sequence of path IDs in use. This is not an error. An endpoint that
receives such a PATH_ABANDON frame must treat it as specified in {{path-close}}.

## Allocating, Consuming, and Retiring Connection IDs {#consume-retire-cid}

With the multipath extension, each connection ID is associated with one path
that is identified by the Path ID that is specified in the Path Identifier field of
the PATH_NEW_CONNECTION_ID frame {{mp-new-conn-id-frame}}.
The Path ID 0 indicates the initial path of the connection.
Respectively, the connection IDs used during the handshake belong to the initial path
with Path ID 0.
The PATH_NEW_CONNECTION_ID frame is used to issue new connection IDs for all paths.
In order to let the peer open new paths, it is RECOMMENDED to proactively
issue a Connection ID for at least one unused Path ID, as long as it remains
compatible with the peer's Maximum Path ID limit.

When issuing path-specific connection IDs, an endpoint associates a sequence number
as specified in {{Section 5.1.1 of QUIC-TRANSPORT}}. Each Path ID has its own
connection ID sequence number space whose initial value is 0. On non-initial paths
(i.e., Path ID different from 0), the initial path connection ID is provided by a
PATH_NEW_CONNECTION_ID frame with a sequence number value of 0.

Each endpoint maintains the set of connection IDs received from its peer for each path,
any of which it can use when sending packets on that path; see also {{Section 5.1 of QUIC-TRANSPORT}}.
Usually, it is desired to provide at least one additional connection ID for
all used paths, to allow for migration.
As further specified in {{Section 5.1 of QUIC-TRANSPORT}} connection IDs
cannot be issued more than once on the same connection
and therefore are unique for the scope of the connection,
regardless of the associated Path ID.

{{Section 5.1.2 of QUIC-TRANSPORT}} indicates that an endpoint
can change the connection ID it uses to another available one
at any time during the connection. For the extension specified in
this draft, endpoints MUST only rotate to another connection IDs associated
to the same Path ID. Use of a connection ID associated with
another Path ID will be considered as an attempt to open a new path instead.

Over a given path, both endpoints use connection IDs associated to a given Path
ID. To initiate a path, each endpoint needs to advertise at least one connection ID
for a given Path ID to its peer. Endpoints SHOULD NOT introduce discontinuity
in the issuing of Path IDs through their connection ID advertisements as path initiation
requires available connection IDs for the same Path ID on both sides. For instance,
if the maximum Path ID limit is 2 and the endpoint wants to provide connection IDs
for only one Path ID inside range \[1, 2\], it should select Path ID 1 (and not Path
ID 2). Similarly, endpoints SHOULD consume Path IDs in a continuous way, i.e., when
creating paths. However, endpoints cannot expect to receive new connection IDs
or path initiation attempts with in order use of Path IDs
due to out-of-order delivery or path validation failure.

{{Section 5.1.2. of QUIC-TRANSPORT}} specifies the retirement of connection IDs.
In order to identify a connection ID correctly when the multipath extension is used,
endpoints have to use the PATH_RETIRE_CONNECTION_ID frame instead
of the RETIRE_CONNECTION_ID frame to indicate the respective Path ID together with the
connection ID sequence number, at least for all paths with a Path ID other than 0.
Endpoints can also use PATH_NEW_CONNECTION_ID and
PATH_RETIRE_CONNECTION_ID for the initial path with Path ID 0,
however, the use of NEW_CONNECTION_ID and RETIRE_CONNECTION_ID
is still valid as well and endpoints need to process these frames accordingly
as corresponding to Path ID 0.

Endpoints MUST NOT issue new connection IDs with Path IDs greater than
the Maximum Path Identifier field in MAX_PATH_ID frames (see {{max-paths-frame}})
or the value of initial_max_path_id transport parameter if no MAX_PATH_ID frame was received yet.
Receipt of a frame with a greater Path ID is a connection error as specified
in {{frames}}.
When an endpoint finds it has not enough available unused path identifiers,
it SHOULD either send a MAX_PATH_ID frame to increase the active path limit
(when limited by the sender) or a PATHS_BLOCKED frame
(see Section {{paths-and-cids-blocked-frame}}) to inform the peer that a new path
identifier was needed but the current limit set by the peer prevented the
creation of the new path.

If the client has consumed all the allocated connection IDs for a path, it is supposed to retire
those that are not used anymore, and the server is supposed to provide
replacements for that path, see {{Section 5.1.2. of QUIC-TRANSPORT}}.
Sending a PATH_RETIRE_CONNECTION_ID frame indicates that the connection ID
will not be used anymore. In response, if the path is still open, the peer
SHOULD provide new connection IDs using PATH_NEW_CONNECTION_ID frames.

Retirement of connection IDs will not retire the Path ID
that corresponds to the connection ID or any other path resources
as the packet number space is associated with a path.

The peer that sends the PATH_RETIRE_CONNECTION_ID frame can keep sending data
on the path that the retired connection ID was used on but has
to use a different connection ID for the same Path ID when doing so.


# Packet Protection {#multipath-aead}

Packet protection for QUIC version 1 is specified in {{Section 5 of QUIC-TLS}}.
The general principles of packet protection are not changed for
the multipath extension specified in this document.
No changes are needed for setting packet protection keys,
initial secrets, header protection, use of 0-RTT keys, receiving
out-of-order protected packets, receiving protected packets,
or retry packet integrity. However, the use of multiple number spaces
for 1-RTT packets requires changes in AEAD usage.

## Nonce Calculation

{{Section 5.3 of QUIC-TLS}} specifies AEAD usage, and in particular
the use of a nonce, N, formed by combining the packet protection IV
with the packet number. When multiple packet number spaces are used,
the packet number alone would not guarantee the uniqueness of the nonce.
Therefore, the nonce N is calculated by combining the packet protection
IV with the packet number and with the least significant 32 bits of the
Path ID. In order to guarantee the uniqueness of the nonce, the Path ID
is limited to a max value of 2^32-1.

To calculate the nonce, a 96-bit path-and-packet-number is composed of the least
significant 32 bits of the Path ID in network byte order,
two zero bits, and the 62 bits of the reconstructed QUIC packet number in
network byte order. If the IV is larger than 96 bits, the path-and-packet-number
is left-padded with zeros to the size of the IV. The exclusive OR of the padded
packet number and the IV forms the AEAD nonce.

For example, assuming the IV value is `6b26114b9cba2b63a9e8dd4f`,
the Path ID is `3`, and the packet number is `aead`,
the nonce will be set to `6b2611489cba2b63a9e873e2`.

## Key Update {#multipath-key-update}

The Key Phase bit update process is specified in
{{Section 6 of QUIC-TLS}}. The general principles of key update
are not changed in this specification. Following {{QUIC-TLS}}, the Key Phase bit is used to
indicate which packet protection keys are used to protect the packet.
The Key Phase bit is toggled to signal each subsequent key update.

Because of network delays, packets protected with the older key might
arrive later than the packets protected with the new key, however receivers
can solely rely on the Key Phase bit to determine the corresponding packet
protection key, assuming that there is sufficient interval between two
consecutive key updates ({{Section 6.5 of QUIC-TLS}}).

When this specification is used, endpoints SHOULD wait for at least three times
the largest PTO among all the paths before initiating a new key update
after receiving an acknowledgment that confirms the receipt of the previous key
update. This interval is different from that in {{QUIC-TLS}}
which used three times the PTO of the sole single path.

The choice of three times the largest PTO is a trade-off:
Packets that arrive after their decryption key has been discarded will be dropped.
Longer delays reduce the probability of losing packets but keeping old keys
longer can negatively impact the security of the protocol.
The use of three times the largest PTO aims to minimize packet lost for all paths
and therefore limits the impact on performance.

Following {{Section 5.4 of QUIC-TLS}}, the Key Phase bit is protected,
so sending multiple packets with Key Phase bit flipping at the same time
should not cause linkability issue.

# Examples

## Path Establishment

{{fig-example-new-path}} illustrates an example of new path establishment
using multiple packet number spaces.

In this example it is assumed that both endpoints have
indicated an initial_max_path_id value of at least 2, which means
both endpoints can use Path IDs 0, 1, and 2. Note that
Path ID 0 is already used for the initial path.

~~~
   Client                                                  Server

   (Exchanges start on default path)
   1-RTT[]: PATH_NEW_CONNECTION_ID[C1, Seq=0, PathID=1] -->
             <-- 1-RTT[]: PATH_NEW_CONNECTION_ID[S1, Seq=0, PathID=1]
             <-- 1-RTT[]: PATH_NEW_CONNECTION_ID[S2, Seq=0, PathID=2]
   ...
   (starts new path)
   1-RTT[0]: DCID=S1, PATH_CHALLENGE[X] -->
                           Checks AEAD using nonce(Path ID 1, PN 0)
        <-- 1-RTT[0]: DCID=C1, PATH_RESPONSE[X], PATH_CHALLENGE[Y],
                                             PATH_ACK[PathID=1, PN=0]
   Checks AEAD using nonce(Path ID 1, PN 0)
   1-RTT[1]: DCID=S1, PATH_RESPONSE[Y],
            PATH_ACK[PathID=1, PN=0], ... -->

~~~
{: #fig-example-new-path title="Example of new path establishment"}

In {{fig-example-new-path}}, the endpoints first exchange
new available connection IDs with the NEW_CONNECTION_ID frame.
In this example, the client provides one connection ID (C1 with
Path ID 1), and server provides two connection IDs
(S1 with Path ID 1, and S2 with Path ID 2).

Before the client opens a new path by sending a packet on that path
with a PATH_CHALLENGE frame, it has to check whether there is
an unused connection IDs for the same unused Path ID available for each side.
In this example the Path ID 1 is used which is the smallest unused Path ID available
as recommended in {{consume-retire-cid}}.
Respectively, the client chooses the connection ID S1
as the Destination Connection ID of the new path.


## Path Closure

{{fig-example-path-close1}} illustrates an example of path closure.

In this example, the client wants to close the path with Path ID 1.
It sends the PATH_ABANDON frame to terminate the path. After receiving
the PATH_ABANDON frame with Path ID 1, the server also send a
PATH_ABANDON frame with Path ID 1.

~~~
Client                                                      Server

(client tells server to abandon a path with Path ID 1)
1-RTT[X]: DCID=S1 PATH_ABANDON[Path ID=1]->
                           (server tells client to abandon a path)
                    <-1-RTT[Y]: DCID=C1 PATH_ABANDON[Path ID=1],
                                           PATH_ACK[PATH ID=1, PN=X]
1-RTT[U]: DCID=S2 PATH_ACK[Path ID=1, PN=Y] ->
~~~
{: #fig-example-path-close1 title="Example of closing a path."}

Note that the last acknowledgment needs to be send on a different path. This examples assumes another path which uses connection ID S2 exists.


# Implementation Considerations

## Number Spaces

As stated in {{introduction}}, when the multipath extension is negotiated, each
path uses a separate packet number space.
This is a major difference from
{{QUIC-TRANSPORT}}, which only defines three number spaces (Initial,
Handshake and Application packets).

The relation between packet number spaces and paths is fixed.
Connection IDs are separately allocated for each Path ID.
Rotating the connection ID on a path does not change the Path ID.
NAT rebinding, though it changes the 4-tuple of the path,
also does not change the path identifier.
The packet number space does not change when connection ID
rotation happens within a given Path ID.

Data associated with the transmission and reception such RTT measurements,
congestion control state, or loss recovery are maintained per packet number
space and as such per Path ID.


## Congestion Control {#congestion-control}

When the QUIC multipath extension is used, senders manage per-path
congestion status as required in {{Section 9.4 of QUIC-TRANSPORT}}.
However, in {{QUIC-TRANSPORT}} only one path is assumed and as such
the requirement is to reset the congestion control status on path migration.
With the multipath extension, multiple paths can be used simultaneously,
therefore separate congestion control state is maintained for each path.
This means a sender is not allowed to send more data on a given path
than congestion control for that path indicates.

When a Multipath QUIC connection uses two or more paths, there is no
guarantee that these paths are fully disjoint. When two (or more paths)
share the same bottleneck, using a standard congestion control scheme
could result in an unfair distribution of the bandwidth with
the multipath connection getting more bandwidth than competing single
paths connections. Multipath TCP uses the linked increased algorithm (LIA)
congestion control scheme
specified in {{RFC6356}} to solve this problem.  This scheme can
immediately be adapted to Multipath QUIC. Other coupled congestion
control schemes have been proposed for Multipath TCP such as {{OLIA}}.

## Computing Path RTT {#compute-rtt}

Acknowledgment delays are the sum of two one-way delays, the delay
on the packet sending path and the delay on the return path chosen
for the acknowledgments.  When different paths have different
characteristics, this can cause acknowledgment delays to vary
widely.  Consider for example a multipath transmission using both a
terrestrial path, with a latency of 50ms in each direction, and a
geostationary satellite path, with a latency of 300ms in both
directions.  The acknowledgment delay will depend on the combination
of paths used for the packet transmission and the ACK transmission,
as shown in {{fig-example-ack-delay}}.


ACK Path \ Data path         | Terrestrial   | Satellite
-----------------------------|-------------------|-----------------
Terrestrial | 100ms  | 350ms
Satellite   | 350ms  | 600ms
{: #fig-example-ack-delay title="Example of ACK delays using multiple paths"}

The PATH_ACK frames describe packets that were sent on the specified path,
but they may be received through any available path. There is an
understandable concern that if successive acknowledgments are received
on different paths, the measured RTT samples will fluctuate widely,
and that might result in poor performance. While this may be a concern,
the actual behavior is complex.

The computed values reflect both the state of the network path and the
scheduling decisions by the sender of the PATH_ACK frames. In the example
above, we may assume that the PATH_ACK will be sent over the terrestrial
link, because that provides the best response time. In that case, the
computed RTT value for the satellite path will be about 350ms. This
lower than the 600ms that would be measured if the PATH_ACK came over
the satellite channel, but it is still the right value for computing
for example the PTO timeout: if a PATH_ACK is not received after more
than 350ms, either the data packet or its PATH_ACK were probably lost.

The simplest implementation is to compute smoothed_rtt and rttvar per
{{Section 5.3 of QUIC-RECOVERY}} regardless of the path through which PATH_ACK frames are
received. This algorithm will provide good results,
except if the set of paths changes and the PATH_ACK sender
revisits its sending preferences. This is not very
different from what happens on a single path if the routing changes.
The RTT, RTT variance and PTO estimates will rapidly converge to
reflect the new conditions.
There is however an exception: some congestion
control functions rely on estimates of the minimum RTT. It might be prudent
for nodes to remember the path over which the PATH_ACK that produced
the minimum RTT was received, and to restart the minimum RTT computation
if that path is abandoned.

## Packet Scheduling {#packet-scheduling}

The transmission of QUIC packets on a regular QUIC connection is regulated
by the arrival of data from the application and the congestion control
scheme. QUIC packets that increase the number of bytes in flight can only be sent
when the congestion window allows it.
Multipath QUIC implementations also need to include a packet scheduler
that decides, among the paths whose congestion window is open, the path
over which the next QUIC packet will be sent. Most frames, including
control frames (PATH_CHALLENGE and PATH_RESPONSE being the notable
exceptions), can be sent and received on any open path. The scheduling
is a local decision, based on the preferences of the application and the
implementation.

Note that this implies that an endpoint may send and receive PATH_ACK
frames on a path different from the one that carried the acknowledged
packets. As noted in {{compute-rtt}} the values computed using
the standard algorithm reflect both the characteristics of the
path and the scheduling algorithm of PATH_ACK frames. The estimates will converge
faster if the scheduling strategy is stable, but besides that
implementations can choose between multiple strategies such as sending
PATH_ACK frames on the path they acknowledge packets, or sending
PATH_ACK frames on the shortest path, which results in shorter control loops
and thus better performance. However, since packets that only carry PATH_ACK frames
are not congestion controlled (see {{Section 7 of QUIC-RECOVERY}}),
senders should carefully consider the load induced
by these packets, especially if the capacity is unknown on that path.

## Retransmissions

Simultaneous use of multiple paths enables different
retransmission strategies to cope with losses such as:
a) retransmitting lost frames over the
same path, b) retransmitting lost frames on a different or
dedicated path, and c) duplicate lost frames on several paths (not
recommended for general purpose use due to the network
overhead). While this document does not preclude a specific
strategy, more detailed specification is out of scope.

As noted in {{Section 2.2 of QUIC-TRANSPORT}}, STREAM frame boundaries are not
expected to be preserved when data is retransmitted. Especially when STREAM
frames have to be retransmitted over a different path with a smaller MTU limit,
new smaller STREAM frames might need to be sent instead.

## Handling PTO

An implementation should follow the mechanism specified in {{QUIC-RECOVERY}}
for detecting packet loss on each individual path. A special case happens when
the PTO timer expires. According to {{QUIC-RECOVERY}}, no packet will be declared
lost until either the packet sender receives a new acknowledgement for this path,
or the path itself is finally declared broken. This cautious process minimizes
the risk of spurious retransmissions, but it may cause significant delivery delay
for the frames contained in these "lost packets".

Endpoints could take advantage of the multipath extension, and retransmit the content
of the delayed packets on other available paths if the congestion control window on these
paths allows.

## Handling different PMTU sizes

An implementation should take care to handle different PMTU sizes across
multiple paths. As specified in {{Section 14.3 of QUIC-TRANSPORT}} the
DPLPMTUD Maximum Packet Size (MPS) is maintained for each combination of local and remote IP addresses.
One simple option, if the PMTUs are relatively similar, is to apply the minimum PMTU of all paths to
each path. The benefit of such an approach is to simplify retransmission
processing as the content of lost packets initially sent on one path can be sent
on another path without further frame scheduling adaptations.

## Keep Alive

The QUIC specification defines an optional keep alive process, see {{Section 5.3 of QUIC-TRANSPORT}}.
Implementations of the multipath extension should map this keep alive process to a number of paths.
Some applications may keep only one path alive, while others could prefer to maintain
liveliness on two or more paths during the connection lifetime.
Different applications will likely require different strategies.
Once the implementation has decided which paths to keep alive, it can do so by sending Ping frames
on each of these paths before the idle timeout expires.

## Connection ID Changes

{{Section 5.1.2 of QUIC-TRANSPORT}} indicates that an endpoint
can change the connection ID it uses to another available one
at any time during the connection. As such a sole change of the Connection
ID without any change in the address does not indicate a path change and
the endpoint can keep the same congestion control and RTT measurement state.

## Migration and NAT Rebindings {#migration}

While endpoints assign a connection ID to a specific sending 4-tuple,
networks events such as NAT rebinding may make the packet's receiver
observe a different 4-tuple. Servers observing a 4-tuple change will
perform path validation (see {{Section 9 of QUIC-TRANSPORT}}).
If path validation process succeeds, the endpoints set
the path's congestion controller and round-trip time
estimator according to {{Section 9.4 of QUIC-TRANSPORT}}.

{{Section 9.3 of QUIC-TRANSPORT}} allows an endpoint to skip validation of
a peer address if that address has been seen recently. However, when the
multipath extension is used and an endpoint has multiple addresses that
could lead to switching between different paths, it should rather maintain
multiple open paths instead.

More generally, while migration cannot be avoided in case of network-based
NAT rebindings, opening a new path instead of active client migration
should be strongly preferred when the multipath extension is supported.
This enables a smoother handover and allows a simplified migration
handling at the server as NAT rebindings imply immediate loss of the old
address.

## Using multiple paths on the same 4-tuple

As noted in {{basic-design-points}}, it is possible to create paths that
refer to the same 4-tuple. For example, the endpoints may want
to create paths that use different Differentiated Service {{?RFC2475}} markings.
This could be done in conjunction with scheduling algorithms
that match streams to paths, so that for example data frames for
low priority streams are sent over low priority paths.
Since these paths use different path IDs, they can be managed
independently to suit the needs of the application.

There may be cases in which paths are created with different 4-tuples,
but end up using the same 4-tuples as a consequence of path
migrations. For example:

* Client starts path 1 from address 192.0.2.1 to server address 198.51.100.1
* Client starts path 2 from address 192.0.2.2 to server address 198.51.100.1
* both paths are used for a while.
* Server sends packet from address 198.51.100.1 to client address 192.0.2.1, with CID indicating path=2.
* Client receives packet, recognizes a path migration, update source address of path 2 to 192.0.2.1.

Such unintentional use of the same 4-tuple on different paths ought to
be rare. When they happen, the two paths would be redundant, and the
endpoint will want to close one of them.
Uncoordinated Abandon from both ends of the connection may result in deleting
two paths instead of just one. To avoid this pitfall, endpoints could
adopt a simple coordination rule, such as only letting the client
initiate closure of duplicate paths, or perhaps relying on
the application protocol to decide which paths should be closed.


# New Frames {#frames}

All frames defined in this document MUST only be sent in 1-RTT packets.

If an endpoint receives a multipath-specific frame in a different packet type,
it MUST close the connection with an error of type FRAME_ENCODING_ERROR.

Receipt of multipath-specific frames
that use a Path ID that is greater than the announced Maximum Paths value
in the MAX_PATH_ID frame or in the initial_max_path_id transport parameter,
if no MAX_PATH_ID frame was received yet,
MUST be treated as a connection error of type PROTOCOL_VIOLATION.

If an endpoint receives a multipath-specific frame
with a path identifier that it cannot process
anymore (e.g., because the path might have been abandoned), it
MUST silently ignore the frame.

## PATH_ACK Frame {#mp-ack-frame}

The PATH_ACK frame (types TBD-00 and TBD-01)
is an extension of the ACK frame specified in {{Section 19.3 of QUIC-TRANSPORT}}. It is
used to acknowledge packets that were sent on different paths, as
each path as its own packet number space. If the frame type is TBD-01, PATH_ACK frames
also contain the sum of QUIC packets with associated ECN marks received
on the acknowledged packet number space up to this point.

PATH_ACK frame is formatted as shown in {{fig-mp-ack-format}}.

~~~
  PATH_ACK Frame {
    Type (i) = TBD-00..TBD-01
         (experiments use  0x15228c00-0x15228c01),
    Path Identifier (i),
    Largest Acknowledged (i),
    ACK Delay (i),
    ACK Range Count (i),
    First ACK Range (i),
    ACK Range (..) ...,
    [ECN Counts (..)],
  }
~~~
{: #fig-mp-ack-format title="PATH_ACK Frame Format"}

Compared to the ACK frame specified in {{QUIC-TRANSPORT}}, the following
field is added.

Path Identifier:
: The Path ID associated with the packet number space of the 0-RTT and 1-RTT packets
  which are acknowledged by the PATH_ACK frame.

## PATH_ABANDON Frame {#path-abandon-frame}

The PATH_ABANDON frame informs the peer to abandon a path.

PATH_ABANDON frames are formatted as shown in {{fig-path-abandon-format}}.

~~~
  PATH_ABANDON Frame {
    Type (i) = TBD-02 (experiments use 0x15228c05),
    Path Identifier (i),
    Error Code (i),
  }
~~~
{: #fig-path-abandon-format title="PATH_ABANDON Frame Format"}

PATH_ABANDON frames contain the following fields:

Path Identifier:
: The Path ID to abandon.

Error Code:
: A variable-length integer that indicates the reason for abandoning
  this path. NO_ERROR(0x0) indicates that the path is being abandoned
  without any error being encountered. Other error codes can be found in {{error-codes}}.

PATH_ABANDON frames are ack-eliciting. If a packet containing
a PATH_ABANDON frame is considered lost, the peer SHOULD repeat it.

Use of the PATH_ABANDON frame is specified in section {{path-close}}.

## PATH_BACKUP and PATH_AVAILABLE frames {#path-backup-available-frame}

PATH_AVAILABLE frames are used by endpoints to inform the peer
that the indicated path is available for sending.
PATH_AVAILABLE frames are formatted as shown in {{fig-path-available-format}}.

~~~
  PATH_AVAILABLE Frame {
    Type (i) = TBD-04 (experiments use 0x15228c08),
    Path Identifier (i),
    Path Status Sequence Number (i),
  }
~~~
{: #fig-path-available-format title="PATH_AVAILABLE Frame Format"}

PATH_BACKUP frames are used by endpoints to inform the peer
about its preference to not use the indicated path for sending.
PATH_BACKUP frames are formatted as shown in {{fig-path-backup-format}}.

~~~
  PATH_BACKUP Frame {
    Type (i) = TBD-03 (experiments use 0x15228c07)
    Path Identifier (i),
    Path Status Sequence Number (i),
  }
~~~
{: #fig-path-backup-format title="PATH_BACKUP Frame Format"}

Both PATH_AVAILABLE and PATH_BACKUP Frames contain the following fields:

Path Identifier:
: The Path ID the status update corresponds to.
  All Path IDs that have been issued
  MAY be specified, even if they are not yet in use over a path.

Path Status Sequence Number:
: A variable-length integer specifying the per-path sequence number assigned for
  this frame.

The sequence number space is common to the two frame types,
and monotonically increasing values MUST be used when sending PATH_AVAILABLE or
PATH_BACKUP frames for a given Path ID.

Frames may be received out of order. A peer MUST ignore an incoming
PATH_AVAILABLE or
PATH_BACKUP frame if it previously received another PATH_BACKUP frame
or PATH_AVAILABLE frame for the same Path ID with a Path Status sequence number
equal to or higher than the Path Status sequence number of the incoming frame.

The requirement of monotonically increasing sequence numbers
is per path. Receivers could very well receive the
same sequence number for PATH_AVAILABLE or PATH_STANDBY Frames
on different paths. The receiver of
the PATH_AVAILABLE or PATH_BACKUP frame needs to use and compare the sequence numbers
separately for each Path ID.

PATH_BACKUP frames are ack-eliciting. If a packet containing a
PATH_BACKUP frame is considered lost, the peer SHOULD resend the frame
only if it contains the last status sent for that path -- as indicated
by the sequence number.

A PATH_BACKUP or a PATH_AVAILABLE frame MAY be bundled with a PATH_NEW_CONNECTION_ID frame or
a PATH_RESPONSE frame in order to indicate the preferred path usage
before or during path initiation.

## PATH_NEW_CONNECTION_ID frames {#mp-new-conn-id-frame}

The PATH_NEW_CONNECTION_ID frame (type=0x15228c09)
is an extension of the NEW_CONNECTION_ID frame specified in
{{Section 19.15 of QUIC-TRANSPORT}}.
It is used to provide its peer with alternative connection IDs for 1-RTT packets
for a specific path. The peer can then use a different connection ID on the same path
to break linkability when migrating on that path; see also {{Section 9.5 of QUIC-TRANSPORT}}.

PATH_NEW_CONNECTION_ID frames are formatted as shown in {{fig-mp-connection-id-frame-format}}.

~~~
PATH_NEW_CONNECTION_ID Frame {
  Type (i) = TBD-05 (experiments use 0x15228c09),
  Path Identifier (i),
  Sequence Number (i),
  Retire Prior To (i),
  Length (8),
  Connection ID (8..160),
  Stateless Reset Token (128),
}
~~~
{: #fig-mp-connection-id-frame-format title="PATH_NEW_CONNECTION_ID Frame Format"}

Compared to the NEW_CONNECTION_ID frame specified in
{{Section 19.15 of QUIC-TRANSPORT}}, the following
field is added:

Path Identifier:
: The Path ID associated with the connection ID. This
  means the provided connection ID can only be used on the corresponding path.

Note that, other than for the NEW_CONNECTION_ID frame of {{Section 19.15 of QUIC-TRANSPORT}},
the sequence number applies on a per-path context.
This means different connection IDs on different paths may have the same
sequence number value.

The Retire Prior To field indicates which connection IDs
should be retired among those that share the Path ID in the Path Identifier field.
Connection IDs associated with different path IDs are not affected.

Note that the NEW_CONNECTION_ID frame can only be used to issue or retire
connection IDs for the initial path with Path ID 0.

The last paragraph of {{Section 5.1.2 of QUIC-TRANSPORT}} specifies how to
verify the Retire Prior To field of an incoming NEW_CONNECTION_ID frame.
The same rule
applies for PATH_RETIRE_CONNECTION_ID frames, but it applies per path. After the
multipath extension is negotiated successfully, the rule
for RETIRE_CONNECTION_ID frame is only applied for Path ID 0.

## PATH_RETIRE_CONNECTION_ID frames {#mp-retire-conn-id-frame}

The PATH_RETIRE_CONNECTION_ID frame (type=0x15228c0a)
is an extension of the RETIRE_CONNECTION_ID frame specified in
{{Section 19.16 of QUIC-TRANSPORT}}. It is used
to indicate that it will no longer use a connection ID for a specific path
that was issued by its peer. To retire the connection ID used
during the handshake on the initial path, Path ID 0 is used.
Sending a PATH_RETIRE_CONNECTION_ID frame also serves as a request to the peer
to send additional connection IDs for this path (see also {{Section 5.1 of QUIC-TRANSPORT}}),
unless the path specified by the Path ID has been abandoned. New path-specific connection IDs can be
delivered to a peer using the PATH_NEW_CONNECTION_ID frame (see {{mp-new-conn-id-frame}}).

PATH_RETIRE_CONNECTION_ID frames are formatted as shown in {{fig-mp-retire-connection-id-frame-format}}.

~~~
PATH_RETIRE_CONNECTION_ID Frame {
  Type (i) = TBD-06 (experiments use 0x15228c0a),
  Path Identifier (i),
  Sequence Number (i),
}
~~~
{: #fig-mp-retire-connection-id-frame-format title="PATH_RETIRE_CONNECTION_ID Frame Format"}

Compared to the RETIRE_CONNECTION_ID frame specified in
{{Section 19.16 of QUIC-TRANSPORT}}, the following
field is added:

Path Identifier:
: The Path ID associated with the connection ID to retire.

Note that the RETIRE_CONNECTION_ID frame can only be used to retire
connection IDs for the initial path with Path ID 0.

As the PATH_NEW_CONNECTION_ID frames applies the sequence number per path,
the sequence number in the PATH_RETIRE_CONNECTION_ID frame is also per
path. The PATH_RETIRE_CONNECTION_ID frame retires the Connection ID with
the specified Path ID and sequence number.

The processing of an incoming RETIRE_CONNECTION_ID frame
is described in {{Section 19.17 of QUIC-TRANSPORT}}. The same processing
applies for PATH_RETIRE_CONNECTION_ID frames per path, while the
processing of a RETIRE_CONNECTION_ID frame is only applied for Path ID 0.

## MAX_PATH_ID frames {#max-paths-frame}

A MAX_PATH_ID frame (type=0x15228c0c) informs the peer of the maximum path identifier
it is permitted to use.

MAX_PATH_ID frames are formatted as shown in {{fig-max-paths-frame-format}}.

~~~
MAX_PATH_ID Frame {
  Type (i) = TBD-07 (experiments use 0x15228c0c),
  Maximum Path Identifier (i),
}
~~~
{: #fig-max-paths-frame-format title="MAX_PATH_ID Frame Format"}

MAX_PATH_ID frames contain the following field:

Maximum Path Identifier:
: The maximum path identifier that the sending endpoint is willing to accept.
  This value MUST NOT exceed 2^32-1, the maximum allowed value for the Path ID due to
  restrictions on the nonce calculation (see {{multipath-aead}}).
  The Maximum Path Identifier value MUST NOT be lower than the value
  advertised in the initial_max_path_id transport parameter.

Receipt of an invalid Maximum Path Identifier value MUST be treated as a
connection error of type PROTOCOL_VIOLATION.

Loss or reordering can cause an endpoint to receive a MAX_PATH_ID frame with
a smaller Maximum Path Identifier value than was previously received.
MAX_PATH_ID frames that do not increase the path limit MUST be ignored.

MAX_PATH_ID frames are ack-eliciting and SHOULD be retransmitted when lost
and no more recent MAX_PATH_ID frame has been sent in the mean time.

## PATHS_BLOCKED and PATH_CIDS_BLOCKED frames {#paths-and-cids-blocked-frame}

A sender can send a PATHS_BLOCKED frame (type=0x15228c0d) when
it wishes to open a path but is unable to do so due to the maximum path identifier
limit set by its peer.

A sender can send a PATH_CIDS_BLOCKED frame (type=0x15228c0e) when
it wishes to open a path with a valid Path ID or change the CID on an established path
but is unable to do so because there are no unused connection IDs available
for the corresponding Path ID.

Note that PATHS_BLOCKED and PATH_CIDS_BLOCKED frames are informational.
Sending a PATHS_BLOCKED or a PATH_CIDS_BLOCKED frame does not imply a particular action from the peer
like updating the new Max Path ID value, but informs the peer that the maximum path identifier limit
or the absence of unused connection IDs prevented the creation or the usage of paths.

PATHS_BLOCKED frames are formatted as shown in {{fig-paths-blocked-frame-format}}.

~~~
PATHS_BLOCKED Frame {
  Type (i) = TBD-08 (experiments use 0x15228c0d),
  Maximum Path Identifier (i),
}
~~~
{: #fig-paths-blocked-frame-format title="MAX_PATH_ID_BLOCKED Frame Format"}

PATHS_BLOCKED frames contain the following field:

Maximum Path Identifier:
: A variable-length integer indicating the maximum path identifier that was
  allowed at the time the frame was sent. If the received value is lower than
  the currently allowed maximum value, this frame can be ignored.


PATH_CIDS_BLOCKED frames are formatted as shown in {{fig-path-cid-blocked-frame-format}}.

~~~
PATH_CIDS_BLOCKED Frame {
  Type (i) = TBD-09 (experiments use 0x15228c0e),
  Path Identifier (i),
}
~~~
{: #fig-path-cid-blocked-frame-format title="PATH_CIDS_BLOCKED Frame Format"}

Path Identifier:
: Identifier of the path for which unused connection IDs are not available.

Receipt of a value of Maximum Path Identifier or Path Identifier that is higher than
the local maximum value MUST be treated as a connection error of type PROTOCOL_VIOLATION.

PATHS_BLOCKED and PATH_CIDS_BLOCKED frames are ack-eliciting and MAY be retransmitted
if the path is still blocked when the lost is detected.

# Error Codes {#error-codes}

QUIC transport error codes are 62-bit unsigned integers
(see {{Section 20.1 of QUIC-TRANSPORT}}. In addition to
NO_ERROR(0x0), the following QUIC error codes are defined
for use in the PATH_ABANDON frame:

APPLICATION_ABANDON (TBD-10): The endpoint is abandoning the path at the
request of the application. The application has determined that it no
longer needs this path. This error is used when the application layer
decides to stop using a specific path.

RESOURCE_LIMIT_REACHED (TBD-11): The endpoint is abandoning the path because
it cannot allocate sufficient resources to maintain it. This is due to
limitations in the transport layer's capacity. This error indicates that
resource constraints prevent the continuation of the path.

UNSTABLE_INTERFACE (TBD-12): The endpoint is abandoning the path because
the used interface is considered to be unstable. This condition can occur, e.g.,
due to a weak wireless signal or frequent handover events during high-speed mobility.

NO_CID_AVAILABLE (TBD-13): The endpoint is abandoning the path due to
the lack of a connection ID for this path.
This may occur when the peer initiates a new path
but has not provided a corresponding connection ID for the path ID
(or the packet containing the connection IDs has not arrived yet).


# IANA Considerations

This document defines a new transport parameter for the negotiation of
enable multiple paths for QUIC, and three new frame types. The draft defines
provisional values for experiments, but we expect IANA to allocate
short values if the draft is approved.

The following entry in {{transport-parameters}} should be added to
the "QUIC Transport Parameters" registry under the "QUIC Protocol" heading.

Value                                         | Parameter Name.   | Specification
----------------------------------------------|-------------------|-----------------
TBD (current version uses 0x0f739bbc1b666d0c) | initial_max_path_id | {{nego}}
{: #transport-parameters title="Addition to QUIC Transport Parameters Entries"}


The following frame types defined in {{frame-types}} should be added to
the "QUIC Frame Types" registry under the "QUIC Protocol" heading.


Value                                              | Frame Name          | Specification
---------------------------------------------------|---------------------|-----------------
TBD-00 - TBD-01 (experiments use 0x15228c00-0x15228c01) | PATH_ACK              | {{mp-ack-frame}}
TBD-02 (experiments use 0x15228c05)                  | PATH_ABANDON        | {{path-abandon-frame}}
TBD-03 (experiments use 0x15228c07)                  | PATH_BACKUP        | {{path-backup-available-frame}}
TBD-04 (experiments use 0x15228c08)                  | PATH_AVAILABLE      | {{path-backup-available-frame}}
TBD-05 (experiments use 0x15228c09)                  | PATH_NEW_CONNECTION_ID   | {{mp-new-conn-id-frame}}
TBD-06 (experiments use 0x15228c0a)                  | PATH_RETIRE_CONNECTION_ID| {{mp-retire-conn-id-frame}}
TBD-07 (experiments use 0x15228c0c)                  | MAX_PATH_ID            | {{max-paths-frame}}
TBD-08 (experiments use 0x15228c0d)                  | PATHS_BLOCKED     | {{paths-and-cids-blocked-frame}}
TBD-09 (experiments use 0x15228c0e)                  | PATH_CIDS_BLOCKED | {{paths-and-cids-blocked-frame}}
{: #frame-types title="Addition to QUIC Frame Types Entries"}

The following transport error code defined in {{tab-error-code}} are to
be added to the "QUIC Transport Error Codes" registry under
the "QUIC Protocol" heading.

Value                       | Code                  | Description                   | Specification
----------------------------|-----------------------|-------------------------------|-------------------
TBD-10 (experiments use 0x004150504142414e) | APPLICATION_ABANDON | Path abandoned at the application's request | {{error-codes}}
TBD-11 (experiments use 0x0052534c494d4954) | RESOURCE_LIMIT_REACHED | Path abandoned due to resource limitations in the transport | {{error-codes}}
TBD-12 (experiments use 0x00554e5f494e5446) | UNSTABLE_INTERFACE | Path abandoned due to unstable interfaces | {{error-codes}}
TBD-13 (experiments use 0x004e4f5f4349445f) | NO_CID_AVAILABLE | Path abandoned due to no available connection IDs for the path | {{error-codes}}
{: #tab-error-code title="Error Codes for Multipath QUIC"}


# Security Considerations

The multipath extension retains all the security features of {{QUIC-TRANSPORT}} and {{QUIC-TLS}}
but requires some additional consideration regarding the following amendments:

- the need of potential additional resources as connection IDs are now maintained per-path;
- the provisioning of multiple concurrent path contexts and the associated resources;
- the possibility to create and use multiple simultaneous paths and the corresponding increased amplification risk for request forgery attacks;
- the changes on encryption requirements due to the use of multiple packet number spaces.


## Memory Allocation for Per-Path Resources

The initial_max_path_id transport parameter and the Max Path ID field
in the MAX_PATH_ID frame limit the number of paths an endpoint is willing
to maintain and accordingly limit the associated path resources.

Furthermore, as connection IDs have to be issued by both endpoints for the
same path ID before an endpoint can open a path, each endpoint can further
control the per-path resource usage (beyond the connection IDs) by limiting
the number of Path ID that it issues connection IDs for.

Therefore, to avoid unnecessarily resource usage, that potentially could be exploited
in a resource exhaustion attack, endpoints should allocate those additional path resource,
such as e.g. for packet number handling, only after path validation has successfully completed.


## Request Forgery with Spoofed Address

The path validation mechanism as specified in {{Section 8.2. of QUIC-TRANSPORT}} for migration is used
unchanged for initiation of new paths in this extension. Therefore, the security considerations
on source address spoofing as outlined in {{Section 21.5.4 of QUIC-TRANSPORT}} equally apply.
Similarly, the anti-amplification limits as specified in {{Section 8 of QUIC-TRANSPORT}} need to be
followed to limit the amplification risk.

However, while {{QUIC-TRANSPORT}} only allows the use of one path simultaneously
and therefore only one path migration at the time should be validated,
this extension allows for multiple open paths, that could in theory be migrated
all at the same time, and it allows for multiple paths that could be initialized
simultaneously. Therefore, each path could be used to further amplify an attack.
Endpoints needs limit the number of maximum paths and might consider
additional measures to limit the number of concurrent path validation processes
e.g. by pacing them out or limiting the number of path initiation attempts
over a certain time period.


## Use of Transport Layer Security and the AEAD Encryption Nonce

The multipath extension as specified in this document is only enabled after a
successful handshake when both endpoints indicate support for this extension.
Respectively, all new frames defined in this extension are only used in 1-RTT packets.
As the handshake is not changed by this extension, the transport security mechanisms
as specified in {{QUIC-TLS}}, such as encryption key exchange and peer authentication,
remain unchanged as well and the respective security considerations in {{QUIC-TLS}} applied unaltered.
Note that with the use of this extension, multiple nonces can be in use simultaneously
for the same AEAD key.

Further note, that the limits as discussed on Appendix B of {{QUIC-TLS}}
apply to the total number of packets sent on all paths.

This specification changes the AEAD calculation by using the path identifier as part of
AEAD encryption nonce (see {{multipath-aead}}). To ensure a unique nonce, path identifiers
are limited to 32 bits and cannot be reused for another path in the same connection.


# Contributors

This document is a collaboration of authors that combines work from
three proposals. Further contributors that were also involved
one of the original proposals are:

* Qing An
* Zhenyu Li

# Acknowledgments

Thanks to Marten Seemann, Kazuho Oku, Martin Thomson, Magnus Westerlund, Mike Bishop, Lucas Pardue, Michael Eriksson, and Yu Zhu for their thorough reviews and valuable contributions.
