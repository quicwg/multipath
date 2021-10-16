---
title: Multipath Extension for QUIC
abbrev: Multipath QUIC
docname: draft-lmbdhk-quic-multipath-latest
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
  ins: Y. Liu
  name: Yanmei Liu
  org: Alibaba Inc.
  email: miaoji.lym@alibaba-inc.com
-
   ins: Y. Ma
   name: Yunfei Ma
   org: Alibaba Inc.
   email: yunfei.ma@alibaba-inc.com
-
   ins: Q. De Coninck
   name: Quentin De Coninck
   org: UCLouvain
   email: quentin.deconinck@uclouvain.be
-
   ins: O. Bonaventure
   name: Olivier Bonaventure
   org: UCLouvain
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
  RFC2119:
  QUIC-TRANSPORT: rfc9000
  QUIC-TLS: rfc9001


informative:
  RFC6356:
  I-D.bonaventure-iccrg-schedulers:
  QUIC-RECOVERY: rfc9002
  QUIC-Invariants: rfc8999
  OLIA:
    title: "MPTCP is not pareto-optimal: performance issues and a possible solution"
    date: "2012"
    seriesinfo: "Proceedings of the 8th international conference on Emerging networking experiments and technologies, ACM"
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

This document specifies multipath extension for the QUIC protocol to enable the simultaneous usage of multiple paths for a single connection.

--- middle

# Introduction

This document specifies an extension to the QUIC v1 {{QUIC-TRANSPORT}} to enable the simultaneous usage of multiple paths for a single connection.

This proposal is based on several basic design points:

* Re-use as much as possible mechanisms of QUIC-v1. In particular this proposal uses path validation as specified for QUIC v1 and aims to re-use as much as possible of QUIC's connection migration.
* Use the same packet header formats as QUIC v1 to avoid the risk of packets being dropped by middleboxes (which may only support QUIC v1)
* Congestion Control, RTT measurements and PMTU discovery should be per-path (following {{QUIC-TRANSPORT}})
* A path is determined by the 4-tuple of source and destination IP address as well as source and destination port. Therefore there can be at most one active paths/connection ID per 4-tuple.

The path management specified in section 9 of {{QUIC-TRANSPORT}} fulfills multiple goals: it directs a peer to switch sending through a new preferred path, and it allows the peer to release resources associated with the old path. Multipath requires several changes to that mechanism:

  *  Allow simultaneous transmission of non probing frames on multiple
     paths.

  *  Continue using an existing path even if non-probing frames have
     been received on another path.

  *  Manage the removal of paths that have been abandoned.

As such this extension specifies a departure from the specification of path management in section 9 of {{QUIC-TRANSPORT}} and therefore requires negotiation between the two endpoints using a new transport parameter, as specified in {{nego}}.

This proposal supports the negotiation of either the use of one packet number space for all paths or the use of separate packet number spaces per path. While separate packet number spaces allow for more efficient ACK encoding especially when different paths have highly difference latencies, it requires the use of a connection ID. Therefore both approaches support different use cases as a single number space can be beneficial in highly constrained networks that do not benefit from exposing the connection ID in the header. More evaluation and implementation experience is needed to either only select one of the approaches or decide for one mandatory to implement approach to ensure interoperability.

This proposal does not cover address discovery and management. Addresses and the actual decision process to setup or tear down paths are assumed to be handled by the application that is using the QUIC multipath extension. Further, this proposal only specifies a simple basic packet scheduling algorithm in order to provide some basic implementation guidance. However, more advanced algorithms as well as potential extensions to enhance signaling of the current paths are expected as future work.

## Conventions and Definitions {#definition}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT",
"RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted
as described in BCP 14 {{!RFC2119}} {{!RFC8174}} when, and only when, they appear in all
capitals, as shown here.

We assume that the reader is familiar with the terminology used in {{QUIC-TRANSPORT}}.
In addition, we define the following terms:

- Path Identifier (Path ID): An identifier that is used to identify a path in a QUIC connection
  at an endpoint. Path Identifier is used in multi-path control frames (etc. PATH_ABANDON frame)
  to identify a path. By default, it is defined as the sequence number of the destination Connection ID
  used for sending packets on that particular path, but alternative definitions can be used
  if the length of that connection ID is zero.

- Packet Number Space Identifier (PN Space ID): An identifier that is used to distinguish packet
    number spaces for different paths. It is used in 1-RTT packets and ACK_MP frames. Each node
    maintains a list of "Received Packets" for each of the CID that it provided to the peer,
    which is used for acknowledging packets received with that CID.

The difference between Path Identifier and Packet Number Space Identifier, is that Path Identifier
is used in multi-path control frames to identify a path, and Packet Number Space Identifier is used
in 1-RTT packets and ACK_MP frames to distinguish packet number spaces for different paths.
Both identifiers have the sam value, which is the sequence number of the connection ID, if a
non-zero connection ID is used. If the connection ID is zero length, the Packet Number Space
Identifier is 0, while the Path Identifier is selected on path establishment.

# Handshake Negotiation and Transport Parameter {#nego}

This extension defines a new transport parameter, used to negotiate the use of the multipath extension
during the connection handshake, as specified in {{QUIC-TRANSPORT}}. The new transport parameter is
defined as follow:

- name: enable_multipath (TBD - experiments use 0xbaba)
- value: 0 (default) for disabled, 1 for multipath support with multiple packet number spaces, 2 for multipath support with one packet number space (this is compatible with {{?I-D.liu-multipath-quic}})

If the peer does not carry the enable_multipath transport parameter, which means the peer does not support multipath, endpoint MUST fallback to {{QUIC-TRANSPORT}} with single path and MUST NOT use any frame or mechanism defined in this document.

Note that the transport parameter "active_connection_id_limit" {{QUIC-TRANSPORT}} limits the number of usable
Connection IDs, and also limits the number of concurrent paths. For the QUIC multipath extension this limit even applies when no connection ID is exposed in the QUIC header.

# Path Setup and Removal

After completing the handshake and endpoints have agreed to enable multipath feature,
endpoints can start using multiple paths.

This proposal adds one multi-path control frame for path management:

- PATH_ABANDON frame for the receiver side to abandon the path {{path-abandon-frame}}

All the new frames are sent in 1-RTT packets {{QUIC-TRANSPORT}}.


## Path Initiation

When the multipath option is negotiated, clients that want to use an additional
path MUST first initiate the Address Validation procedure with PATH_CHALLENGE and PATH_RESPONSE
frames described in Section 8 of {{QUIC-TRANSPORT}}. After
receiving packets from the client on the new paths, the servers MAY
in turn attempt to validate these paths using the same mechanisms.

If validation succeed, the client can send non-probing, 1-RTT packets on the new paths.  In
contrast with the specification in section 9 of [QUIC-TRANSPORT], the
server MUST NOT assume that receiving non-probing packets on a new
path indicates an attempt to migrate to that path.  Instead, servers
SHOULD consider paths over which non-probing packets have been received as
available for transmission.

## Path State Management

The current draft defines PATH_ABANDON frame to inform the peer to abandon a
path, and release the corresponding resources. More complex path management can
be made possible with additional extensions (e.g., PATH_STATUS frame in
{{?I-D.liu-multipath-quic}} ). An endpoint uses the sequence number of the CID
used by the peer for PATH_ABANDON frames (describing the sender's path
identifier).

PATH_ABANDON frame can be sent on any path, not only the path identified by the
Path Identifier field.

## Path Close

At any time in the connection, each endpoint manages a set of paths that are available for transmission. At
any time, the client can decide to abandon one of these paths,
following for example changes in local connectivity or changes in
local preferences.  After a client abandons a path, the server will
not receive any more non-probing packets on that path.

An endpoint that wants to close a path SHOULD NOT rely on implicit signals like idle time or packet losses,
but instead SHOULD use explicit request to terminate path by sending the PATH_ABANDON frame.

### Use PATH_ABANDON Frame to Close a Path

Both client and server can close a path, by sending PATH_ABANDON frame which
abandons the path with a corresponding Path Identifier. Once a path is marked
as "abandoned", it means that the resources related to the path can be released.

### Effect of RETIRE_CONNECTION_ID Frame

Receiving a RETIRE_CONNECTION_ID frame causes the endpoint to discard the resources associated with
that connection ID. If the connection ID was used by the peer to identify a path from the peer to
this endpoint, the resources include the list of received packets used to send acknowledgements.
The peer MAY decide to keep sending data using the same IP addresses and UDP ports previously
associated with the connection ID, but MUST use a different connection ID when doing so.

### Idle Timeout

{{QUIC-TRANSPORT}} allows for closing of connections if they stay idle for too long.
The connection idle timeout in multipath QUIC is defined as "no packet received on any path for the
duration of the idle timeout". It means that if all paths remain idle for the idle timeout, the connection
is implicitly closed.

When only one path is available, servers MUST follow the
specifications in {{QUIC-TRANSPORT}}.  When more than one path is
available, servers shall monitor the arrival of non-probing packets
on the available paths.  Servers SHOULD stop sending traffic on paths
through where no non-probing packet was received in the last 3 path
RTTs, but MAY ignore that rule if it would disqualify all available
paths.  Server MAY release the resource associated with paths for
which no non-probing packet was received for a sufficiently long
path-idle delay, but SHOULD only release resource for the last
available path if no traffic is received for the duration of the idle
timeout, as specified in section 10.1 of {{QUIC-TRANSPORT}}.  Server
implementations manage the value of the path-idle delay as a trade-
off between keeping resource such as Connection Identifiers in use
for an excessive time, and having to promptly reestablish a path
after a spurious estimate of path abandonment by the client.

# Congestion Control

Senders MUST manage per-path congestion status, and MUST NOT send more data on a given path than congestion control on that path allows.  This is already a requirement of {{QUIC-TRANSPORT}}.

Multipath TCP uses the LIA congestion control scheme specified in {{RFC6356}}.  This scheme can immediately be adapted to Multipath QUIC. Other coupled congestion control schemes have been proposed for Multipath TCP such as {{OLIA}}.

# Packet Scheduling

The simultaneous usage of several sending uniflows introduces new
   algorithms (packet scheduling, path management) whose specifications
   are out of scope of this document.  Nevertheless, these algorithms
   are actually present in any multipath-enabled transport protocol like
   Multipath TCP, CMT-SCTP and Multipath DCCP.  A companion draft
   {{I-D.bonaventure-iccrg-schedulers}} provides several general-purpose
   packet schedulers depending on the application goals.  A similar
   document can be created to discuss path management
   considerations.

# Packet Number Space and Use of Connection ID

Acknowledgements of Initial and Handshake packets MUST be carried using ACK frames, as specified in {{QUIC-TRANSPORT}}.
The ACK frames, as defined in {{QUIC-TRANSPORT}}, do not carry path identifiers. If for some reason ACK frames are
received in 1-RTT packets while the state of multipath negotiation is ambiguous, they MUST be interpreted as acknowledging
packets sent on path ID 0.

If endpoints negotiate multipath support with value 1 to use multiple
packet number spaces, they SHOULD use ACK_MP frames
instead of ACK frames to signal acknowledgement of 1-RTT packets on path number 0, and also 0-RTT packets as specified in {{handling-of-0-rtt-packets}}.

1-RTT packets sent to different paths SHOULD carry different connection identifiers, if a non-zero
connection ID is used.

## Handling of 0-RTT Packets

Because multi-path is enabled after the handshake negotiation completes,
there will be a separate context for each Connection ID after multi-path is negotiated.
0-RTT packets are sent before these per path contexts are established. To avoid confusion, this draft provides a way for
implementations to deal with 0-RTT packets that is both easy to implement and compatible with {{QUIC-TRANSPORT}}:

- All 0-RTT packets are initially tracked in the "global" application context.
- On the client side, 0-RTT packets are initially sent in the "global" application context. The handshake concludes before
any 1-RTT packet can be sent or received. When the handshake completes, if multipath is negotiated, the tracking of 0-RTT
packets moves from the "global" application context to the "path ID 0" application context. That means the sequence number of
the first 1-RTT packets sent by the client will follow the sequence number of the last 0-RTT packet.
- On the server side, the negotiation completes after the client first flight is received and the the server first flight is sent.
0-RTT packets are received after that. If multipath is negotiated, they are considered received on "path ID 0".

In conclusion, 0-RTT packets are tracked and processed with path identifier 0.


## Using One packet Number Space

If the multipath option is negotiated to use one packet number space of all path, the packet sequence numbers are allocated from
the common number space, so that for example path number number N
could be sent on one path and packet number N+1 on another.

ACK frames report the numbers of packets that have been received so
far, regardless of the path on which they have been received.

If one common packet number space is used for all paths, the senders
maintain an association between previously sent packet numbers and
the path over which these packets were sent. This is necessary to implement per path congestion control.

When a packet is newly acknowledged, the delay between the transmission of that packet and
its first acknowledgement is used to update the RTT statistics for the sending path, and to update the state of the congestion control for that path.

Packet loss detection MUST be adapted to allow for different RTTs on
different paths.  For example, timer computations should take into
account the RTT of the path on which a packet was sent.  Detections
based on packet numbers shall compare a given packet number to the
highest packet number received for that path.

### Acknowledgement and Ranges

If senders decide to send packets on paths with different
transmission delays, some packets will very probably be received out
of order.  This will cause the ACK frames to carry multiple ranges of
received packets.  The large number of range increases the size of
ACK frames, causing transmission and processing overhead.

Early tests showed that the size and overhead of the ACK frames could
be controlled by the combination of one or several of the following:

*  Limiting the number of transmissions of a specific ACK range, on
   the assumption that a sufficient number of transmissions almost
   certainly ensures reception by the peer.

*  Not transmitting again ACK ranges that were present in an ACK
   frame acknowledged by the peer.

*  Delay acknowledgement to allow for arrival of "hole filling"
   packets.

*  Limit the total number of ranges sent in an ACK frame.

*  Send multiple messages for a given path in a single socket
   operation, so that a series of packets sent from a single path
   uses a series of consecutive sequence numbers without creating
   holes.

### Computing Path RTT

Acknowledgement delays are the sum of two one-way delays, the delay
on the packet sending path and the delay on the return path chosen
for the acknowledgements.  When different paths have different
characteristics, this can cause acknowledgement delays to vary
widely.  Consider for example multipath transmission using both a
terrestrial path, with a latency of 50ms in each direction, and a
geostationary satellite path, with a latency of 300ms in both
directions.  The acknowledgement delay will depend on the combination
of paths used for the packet transmission and the ACK transmission,
as shown in Table {{fig-example-ack-delay}}.

~~~
         +======================+=============+===========+
         | ACK Path \ Data path | Terrestrial | Satellite |
         +======================+=============+===========+
         | Terrestrial          | 100ms       | 350ms     |
         +----------------------+-------------+-----------+
         | Satellite            | 350ms       | 600ms     |
         +----------------------+-------------+-----------+
~~~
{: #fig-example-ack-delay title="Example of ACK delays using multiple paths"}

Using the default algorithm specified in {{QUIC-RECOVERY}}. would result
in suboptimal performance, computing average RTT and standard
deviation from a series of different delay measurements of different
combined paths.  At the same time, early tests show that it is
desirable to send ACKs through the shortest path, because a shorter
ACK delay results in a tighter control loop and better performances.
The tests also showed that it is desirable to send copies of the ACKs
on multiple paths, for robustness if a path experiences sudden losses.

An early implementation mitigated the delay variation issue by using
time stamps, as specified in [QUIC-Timestamp].  When the timestamps
are present, the implementation can estimate the transmission delay
on each one-way path, and can then use these one way delays for more
efficient implementations of recovery and congestion control
algorithms.

If timestamps are not available, implementations could estimate one
way delays using statistical techniques.  For example, in the example
shown in Table 1, implementations can use use "same path"
measurements to estimate the one way delay of the terrestrial path to
about 50ms in each direction, and that of the satellite path to about
300ms.  Further measurements can then be used to maintain estimates
of one way delay variations, using logical similar to Kalman filters.
But statistical processing is error-prone, and using time stamps
provides more robust measurements.

## Using Multiple Packet Number Spaces

If the multipath option is enabled with a value of 1, each path has its own packet number space for transmitting 1-RTT packets and a new ACK frame format is used as specified in {{mp-ack-frame}} which additionally contains a Packet Number Space Identifier (PN Space ID). The PN Space ID used to distinguish packet number spaces for different paths and is simply derived from the sequence number of Destination Connection ID. Therefore, the packet number space for 1-RTT packets is specific to the Destination Connection ID in these packets.

Note: Even if the use of multiple packet number spaces is negotiated but a peer uses an zero length connection ID, then all packets sent to that peer MUST be numbered in a single number space (as specified in the previous section), because the packet level decryption implementation will only see one Connection ID sequence number (the default number 0).

Endpoints can find which path a received packet belongs to according to the Destination Connection ID of
the 1-RTT packet. Endpoints identify the context of a path by its Connection ID or the Sequence number
of Connection ID.

Following {{QUIC-TRANSPORT}}, each endpoint uses NEW_CONNECTION_ID frames to issue usable connections IDs
to reach it. Before an endpoint adds a new path, it MUST check whether there is at least one unused
available Connection ID for each side.

If the transport parameter "active_connection_id_limit" is negotiated as N, the server provided N Connection IDs, and the client is already actively using N paths, the limit is reached. If the client wants to start a new path, it has to retire
one of the established paths.

ACK_MP frame {{mp-ack-frame}} can be returned via either a different path, or the same path identified
by the Path Identifier, based on different strategies of sending ACK_MP frames.

Using multiple packet number spaces requires changes in the way AEAD is applied for packet
protection, as explained in {{multipath-aead}}, and tighter constraints for key
updates, as explained in {{multipath-key-update}}.

### Packet Protection for QUIC Multipath {#multipath-aead}

Packet protection for QUIC v1 is specified is section 5 of {{QUIC-TLS}}. The general principles
of packet protection are not changed for QUIC Multipath. No changes are needed for setting
packet protection keys, initial secrets, header protection, use of 0-RTT keys, receiving
out-of-order protected packets, receiving protected packets,
or retry packet integrity. However, the use of multiple
number spaces for 1-RTT packets requires changes in AEAD usage.

Section 5.3 of {{QUIC-TLS}} specifies AEAD usage, and in particular the use of a
nonce, N, formed by combining the packet protection IV with the packet number. QUIC
multipath uses multiple packet number spaces, and thus the packet number alone would
not guarantee the uniqueness of the nonce. In order to guarantee this uniqueness, we
construct the nonce N by combining the packet protection IV with the packet number
and with the identifier of the path, which for 1-RTT packets is the Sequence Number of
the Destination Connection ID present in the packet header, as defined in Section 5.1.1
of {{QUIC-TRANSPORT}}, or zero if the Connection ID is zero-length. Section 19 of
{{QUIC-TRANSPORT}} encodes this Connection ID Sequence Number as a variable-length integer,
allowing values up to 2^62-1; for QUIC multipath, we require that a range of no more than 2^32-1
values be used without updating the packet protection key.

For QUIC multipath, the construction of the nonce starts with the construction of a 96 bit
path-and-packet-number, composed of the 32 bit Connection ID Sequence Number in byte order,
two zero bits, and the 62 bits of the reconstructed QUIC packet number in network byte order.
If the IV is larger than 96 bits, path-and-packet-number is left-padded with zeros to the
size of the IV. The exclusive OR of the padded packet number and the IV forms the AEAD nonce.

For example, assuming the IV value is `6b26114b9cba2b63a9e8dd4f`, the connection ID sequence
number is `3`, and the packet number is `aead`, the nonce will be set to
`6b2611489cba2b63a9e873e2`.

### Key Update for QUIC Multipath {#multipath-key-update}

The Key Phase bit update process for QUIC v1 is specified in Section 6 of
{{QUIC-TLS}}. The general principles of key update are not changed for
Multipath QUIC. Following QUIC V1, the Key Phase bit is used to indicate which
packet protection keys are used to protect the packet. The Key Phase bit is
toggled to signal each subsequent key update. Because of network delays,
packets protected with the older key might arrive later than the packets
protected with the new key. Therefore, the endpoint needs to retain old packet
keys to allow these delayed packets to be processed and it must distinguish
between the new key and the old key. In QUIC V1, this is done using packet
numbers so that the rule is made simple: Use the older key if packet number is
lower than any packet number frame the current key phase.

When using multiple packet number space on different paths,
some care is needed in the initiating Key Update process.
Because different paths use different packet number spaces but share a single
key. When a key update is initiated on one path, packets sent to the other path
needs to know when transition is complete. Otherwise, it is possible that the
other paths send packets with the old keys, but skip sending any packets in the
current key phase and directly jump to sending packet in the next key phase.
When that happens, as the endpoint can only retain two sets of packet
protection keys with the 1-bit Key Phase bit, the other paths cannot
distinguish which key should be used to decode received packets, which results
in a key rotation synchronization problem.

To address such a synchronization issue, if key update is
initilized on one path, the sender SHOULD send at least one packet with the new
key on all active paths. Regarding the responding to Key Update process, the
endpoint MUST NOT initiate a subsequent key update until a packet with the
current key has been acknowledged on each path.

Following the Section 5.4. of {{QUIC-TLS}}, the Key Phase bit is protected, so
sending multiple packets with Key Phase bit flipping at the same time should
not cause linkability issue.

# Examples

## Path Establishment

{{fig-example-new-path}} illustrates an example of new path establishment using multiple packet number spaces.

~~~
   Client                                                  Server

   (Exchanges start on default path)
   1-RTT[]: NEW_CONNECTION_ID[C1, Seq=1] -->
                         <-- 1-RTT[]: NEW_CONNECTION_ID[S1, Seq=1]
                         <-- 1-RTT[]: NEW_CONNECTION_ID[S2, Seq=2]
   ...
   (starts new path)
   1-RTT[0]: DCID=S2, PATH_CHALLENGE[X] -->
                     Checks AEAD using nonce(CID sequence 2, PN 0)
       <-- 1-RTT[0]: DCID=C1, PATH_RESPONSE[X], PATH_CHALLENGE[Y],
                                                ACK_MP[Seq=2,PN=0]
   Checks AEAD using nonce(CID sequence 1, PN 0)
   1-RTT[1]: DCID=S2, PATH_RESPONSE[Y],
             ACK_MP[Seq=1, PN=0], ... -->

~~~
{: #fig-example-new-path title="Example of new path establishment"}

As shown in {{fig-example-new-path}}, client provides one unused available Connection ID (C1 with sequence number 1),
and server provides two available Connection IDs (S1 with sequence number 1, and S2 with sequence number 2).
When client wants to start a new path, it checks whether there is unused available Connection IDs for each side,
and choose an available Connection ID S2 as the Destination Connection ID in the new path.

Endpoints need to exchange unused available Connection IDs with the NEW_CONNECTION_ID frame before an endpoint
starts a new path. For example, if the goal is to maintain 2 paths, each endpoint should provide at least 3 CID to its peer:
2 in use, and one spare. If the client has used all the allocated CID, it is supposed to retire those that are
not used anymore, and the server is supposed to provide replacements, as specified in {{QUIC-TRANSPORT}}.


## Path Closure

{{fig-example-path-close}} illustrates an example of path closing. In this
case, we are going to close the first path. For the first path, the server's
1-RTT packets use DCID C1, which has a sequence number of 1; the client's 1-RTT
packets use DCID S2, which has a sequence number of 2.  For the second path,
the server's 1-RTT packets use DCID C2, which has a sequence number of 2; the
client's 1-RTT packets use CID S3, which has a sequence number of 3. Note that
two paths use different packet number space.

~~~
  Client                                                          Server

  (client tells server to abandon a path)
  1-RTT[X]: DCID=S2 PATH_ABANDON[path_id=1]->
                                 (server tells client to abandon a path)
        <-1-RTT[Y]: DCID=C1 PATH_ABANDON[path_id=2], ACK_MP[Seq=2, PN=X]
  (client abandons the path that it is using)
  1-RTT[U]: DCID=S3 RETIRE_CONNECTION_ID[2], ACK_MP[Seq=1, PN=Y] ->
                             (server abandons the path that it is using)
       <- 1-RTT[V]: DCID=C2 RETIRE_CONNECTION_ID[1], ACK_MP[Seq=3, PN=U]

~~~
{: #fig-example-path-close title="Example of closing a path (path id type=0x00)"}

In scenarios such as client detects the network environment change (client's 4G/Wi-Fi is turned off,
Wi-Fi signal is fading to a threshold), or endpoints detect that the quality of RTT or loss rate is
becoming worse, client or server can terminate a path immediately.

# Implementation Considerations

TDB

# New Frames {#frames}

All the new frames MUST only be sent in 1-RTT packet, and MUST NOT use other encryption levels.

If an endpoint receives multipath-specific frames from packets of other encryption levels, it MUST return
MP_PROTOCOL_VIOLATION as a connection error and close the connection.

## PATH_ABANDON Frame {#path-abandon-frame}

PATH_ABANDON Frame are used by endpoints to inform the peer of the current status of one path, and the peer
should send packets according to the preference expressed in these frames. Endpoint use the sequence number of the
CID used by the peer for PATH_ABANDON frames (describing the sender's path identifier).
PATH_ABANDON frames are formatted as shown in {{fig-path-abandon-format}}.

~~~
  PATH_ABANDON Frame {
    Type (i) = TBD-03 (experiments use 0xbaba05),
    Path Identifier (..),
    Error Code (i),
    Reason Phrase Length (i),
    Reason Phrase (..),
  }
~~~
{: #fig-path-abandon-format title="PATH_ABANDON Frame Format"}

PATH_ABANDON frames contain the following fields:

Path Identifier: An identifier of the path, which is formatted as shown in {{fig-path-identifier-format}}.

- Identifier Type: Identifier Type field is set to indicate the type of path identifier.
  - Type 0: Refer to the connection identifier used by the sender of the control frame when
    sending data over the specified path. This method SHOULD be used if this connection
    identifier is non-zero length. This method MUST NOT be used if this connection identifier
    is zero-length.
  - Type 1: Refer to the connection identifier used by the receiver of the control frame when
    sending data over the specified path. This method MUST NOT be used if this connection
    identifier is zero-length.
  - Type 2: Refer to the path over which the control frame is sent or received.
- Path Identifier Content: A variable-length integer specifying the path identifier.

~~~
  Path Identifier {
    Identifier Type (i) = 0x00..0x02,
    Path Identifier Content (i),
  }
~~~
{: #fig-path-identifier-format title="Path Identifier Format"}

Note: If the receiver of the PATH_ABANDON frame is using non-zero length Connection ID on that path,
endpoint SHOULD use type 0x00 for path identifier in the control frame. If the receiver of the PATH_ABANDON frame
is using 0-length Connection ID, but the peer is using non-zero length Connection ID on that path, endpoints
SHOULD use type 0x01 for path identifier. If both endpoints are using 0-length Connection IDs on that path,
endpoints SHOULD only use type 0x02 for path identifier.

Error Code:
: A variable-length integer that indicates the reason for closing this connection.

Reason Phrase Length:
: A variable-length integer specifying the length of the reason phrase in bytes.
  Because an PATH_ABANDON frame cannot be split between packets, any limits
  on packet size will also limit the space available for a reason phrase.

Reason Phrase:
: Additional diagnostic information for the closure. This can be zero length if
  the sender chooses not to give details beyond the Error Code value.
  This SHOULD be a UTF-8 encoded string {{!RFC3629}}, though the frame
  does not carry information, such as language tags, that would aid comprehension
  by any entity other than the one that created the text.

PATH_ABANDON frames SHOULD be acknowledged. If a packet containing a PATH_ABANDON
frame is considered lost, the peer should repeat it.


## ACK_MP Frame {#mp-ack-frame}

The ACK_MP frame (types TBD-00 and TBD-01; experiments use 0xbaba00..0xbaba01
or 0x42..x43) is an extension of the ACK frame defined by {{QUIC-TRANSPORT}}.
It is used to acknowledge packets that were sent on different paths. If the
frame type is TBD-01, ACK_MP frames also contain the sum of QUIC packets with
associated ECN marks received on the connection up to this point.

ACK_MP frame is formatted as shown in {{fig-mp-ack-format}}.

~~~
  ACK_MP Frame {
    Type (i) = TBD-00..TBD-01 (experiments use 0xbaba00..0xbaba01 or 0x42..x43),
    Packet Number Space Identifier (i),
    Largest Acknowledged (i),
    ACK Delay (i),
    ACK Range Count (i),
    First ACK Range (i),
    ACK Range (..) ...,
    [ECN Counts (..)],
  }
~~~
{: #fig-mp-ack-format title="ACK_MP Frame Format"}

Compared to the ACK frame specified in {{QUIC-TRANSPORT}}, the following field is added.

Packet Number Space Identifier: An identifier of the path packet number space, which is the sequence number of
Destination Connection ID of the 1-RTT packets which are acknowledged by the ACK_MP frame. If the endpoint receives
1-RTT packets with 0-length Connection ID, it SHOULD use Packet Number Space Identifier 0 in ACK_MP frames.
If an endpoint receives a ACK_MP frame with a non-existing packet number space ID, it MUST treat this
as a connection error of type MP_CONNECTION_ERROR and close the connection.


# IANA Considerations

This document defines a new transport parameter for the negotiation of enable multiple paths for QUIC,
and two new frame types. The draft defines provisional values for experiments, but we expect IANA to
allocate short values if the draft is approved.

The following entry in {{transport-parameters}} should be added to the "QUIC Transport Parameters" registry
under the "QUIC Protocol" heading.

Value                        | Parameter Name.   | Specification
-----------------------------|-------------------|-----------------
TBD (experiments use 0xbaba) | enable_multipath  | {{nego}}
{: #transport-parameters title="Addition to QUIC Transport Parameters Entries"}


The following frame types defined in {{frame-types}} should be added to the "QUIC Frame Types" registry
under the "QUIC Protocol" heading.


Value                                              | Frame Name          | Specification
---------------------------------------------------|---------------------|-----------------
TBD-00 - TBD-01 (experiments use 0xbaba00-0xbaba01)| ACK_MP              | {{mp-ack-frame}}
TBD-02 (experiments use 0xbaba05)                  | PATH_ABANDON         | {{path-abandon-frame}}
{: #frame-types title="Addition to QUIC Frame Types Entries"}



# Security Considerations

TBD

# Contributors

This document is a collaboration of authors that combines work from three proposals.
Further contributors that were also involved one of the original proposals are:

* Qing An
* Zhenyu Li

# Acknowledgments

TBD
