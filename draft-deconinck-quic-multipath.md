---
title: Multipath Extensions for QUIC (MP-QUIC)
abbrev: MP-QUIC
docname: draft-deconinck-quic-multipath-03
date: {DATE}
category: std
updates:

ipr: trust200902
area: Transport
workgroup: QUIC Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
  ins: Q. De Coninck
  name: Quentin De Coninck
  organization: UCLouvain
  email: quentin.deconinck@uclouvain.be
 -
  ins: O. Bonaventure
  name: Olivier Bonaventure
  organization: UCLouvain
  email: Olivier.Bonaventure@uclouvain.be

normative:
  RFC2119:
  I-D.ietf-quic-transport:
  I-D.ietf-quic-tls:
  I-D.ietf-quic-recovery:
  I-D.ietf-quic-invariants:
informative:
  I-D.huitema-quic-mpath-req:
  I-D.huitema-quic-1wd:
  RFC0793:
  RFC6356:
  RFC6824:
  RFC7050:
  RFC7225:
  RFC8041:
  MPRTP:
     title: "MPRTP: Multipath considerations for real-time media"
     date: "2013"
     seriesinfo: "Proceedings of the 4th ACM Multimedia Systems Conference"
     author:
      -
        ins: V. Singh
        name:
        org: Aalto University
      -
        ins: S. Ahsan
        name: Saba Ahsan
        org: Aalto University
      -
        ins: J. Ott
        name: Jorg Ott
        org: Aalto University
  SCTPCMT:
     title: "Concurrent multipath transfer using SCTP multihoming over independent end-to-end paths"
     date: "2006"
     seriesinfo: "IEEE/ACM Transactions on networking, Vol. 14, no 5"
     author:
     -
       ins: J. Iyengar
     -
       ins: P. Amer
     -
       ins: R. Stewart
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
  MPQUIC:
    title: "Multipath QUIC: Design and Evaluation"
    date: "Dec. 2017"
    seriesinfo: "13th International Conference on emerging Networking EXperiments and Technologies (CoNEXT 2017). http://multipath-quic.org"
    author:
    -
       ins: Q. De Coninck
    -
       ins: O. Bonaventure
  IETFJ:
    title: "Multipath TCP Deployments"
    date: "Nov. 2016"
    seriesinfo: "IETF Journal"
    author:
    -
        ins: O. Bonaventure
    -
        ins: S. Seo

--- abstract

This document specifies extensions to the QUIC protocol to enable,
other than connection migration, simultaneous usage of multiple paths
for a single connection. Those extensions remain compliant with the
current single-path QUIC design. They allow devices to benefit from
multiple network paths while preserving the privacy features of QUIC.


--- middle

Introduction
============

Endhosts have evolved. Today's endhosts are equipped with several
network interfaces and users expect to be able to seamlessly switch
from one to another or use them simultaneously to aggregate or grab
bandwidth whenever needed. During the last years, several multipath
extensions to transport protocols have been proposed {{RFC6824}},{{MPRTP}},
{{SCTPCMT}}. Multipath TCP {{RFC6824}} is the most mature one. It is
already deployed on popular smartphones, but also for other use cases
{{RFC8041}}.

With regular TCP and UDP, all the packets that belong to a given flow
share the same 5-tuple that acts as an identifier for this flow. Such
characterization prevents these flows from using multiple paths. QUIC
{{I-D.ietf-quic-transport}} does not use the 5-tuple as an implicit
connection identifier. A QUIC flow is identified by a Connection ID. This
nables QUIC flows to cope with events affecting the 5-tuple, such as NAT
rebinding or IP address changes. The QUIC connection migration feature,
described in more details in {{I-D.ietf-quic-transport}}, is key to
migrate a flow from one 5-tuple to another one. This migration feature offers
the opportunity for QUIC to sustain a connection over multiple paths, but still
there is a void to specify simultaneous usage of available network paths for a
single connection. Use cases such as bandwidth aggregation or seamless network
handovers would be applicable to QUIC, as they are now with Multipath
TCP. An early performance evaluation of such use cases and a comparison
between Multipath QUIC and Multipath TCP may be found in {{MPQUIC}}.

In this document, we leverage many of the lessons learned from the
design of Multipath TCP and the comments received on the first versions
of this draft to propose extensions to the current QUIC design to
enable it to simultaneously use several network paths. This document focuses
mainly on paths that are locally visible to an endpoint. This document is
organized as follows. It first provides in {{overview}} an overview of the
operation of Multipath QUIC. It then states
changes required in the current QUIC design {{I-D.ietf-quic-transport}} and
specifies in {{spec}} the usage of multiple paths. Finally, it discusses some
security considerations.


Conventions and Definitions
===========================

The words "MUST", "MUST NOT", "SHOULD", and "MAY" are used in this
document.  It's not shouting; when they are capitalized, they have
the special meaning defined in {{RFC2119}}.

We assume that the reader is familiar with the terminology used in
{{I-D.ietf-quic-transport}}. In addition, we define the following terms:

- Uniflow: A unidirectional flow of packets between a QUIC host and its peer.
  It is identified by an internal Uniflow ID. Packets sent over a uniflow use
  a given (set of) Connection ID(s). When being in use, a uniflow is
  (temporarily) bound to a 4-tuple (Source IP Address, Source Port Number,
  Destination IP Address, Destination Port Number).

- Initial Uniflows: The two uniflows used by peers for the establishment of the
  QUIC connection. The cryptographic handshake is done on these uniflows. These
  are identified by Uniflow ID 0.


Notational Conventions
----------------------

Packet and frame diagrams use the format described in Section 2.1 of
{{I-D.ietf-quic-transport}}.


Overview {#overview}
========

The current design of QUIC {{I-D.ietf-quic-transport}} provides reliable
transport with multiplexing, confidentiality, integrity and authenticity of data
flows. A wide range of devices on today's Internet are multihomed.
Examples include smartphones equipped with both WLAN and cellular
interfaces, but also regular dual-stack hosts that use both IPv4 and
IPv6. Experience with Multipath TCP has shown that the ability to
combine different paths during the lifetime of a connection provides
various benefits including bandwidth aggregation or seamless handovers
{{RFC8041}},{{IETFJ}}.

The current design of QUIC does not enable multihomed devices to efficiently
use different paths simultaneously. This draft proposes multipath extensions
with the following design goals:

* Each host keeps control on the number of paths being used over the connection

* The simultaneous usage of multiple paths should not introduce new privacy
  concerns

* Hosts must ensure that all the paths it uses actually reaches its peer to
  avoid packet flooding towards a victim

* The multipath extensions should handle the asymetrical nature of the networks

We first explain why a multipath extension would be beneficial to QUIC
and then describe it at a high level.


Moving from Bidirectional Paths to Uniflows
-------------------------------------------

To understand the overall architecture of the multipath extensions, let us
first refine the notion of "path". Since the early days of computer networks,
a path was denoted by a 4-tuple (Source IP Address, Source Port Number,
Destination IP Address, Destination Port Number). In QUIC, this is namely a UDP
road from the local host to the remote one. Considering a smartphone interacting
with a single-homed server, the smartphone might want to use one path over the
WLAN network and another over the cellular one. Those paths are not necessarily
disjoint. For example, when interacting with a dual-stack server, a smartphone
may create two paths over the Wi-Fi network: one over IPv4 and the other one
over IPv6.

The notion of bidirectional paths, i.e., paths where packets flow in both
directions, became a de-facto standard with TCP. The Multipath TCP extension
{{RFC6824}} combines several TCP connections to spread a single data stream
over them. Hence, all the paths of a Multipath TCP connection must be
bidirectional. However, networking experiences showed that packets following
a direction do not always share the exact same road as the packets in the
opposite direction. Furthermore, QUIC does not require a network path to be
bidirectional in order to be used. For this, let us consider the {{examplequic}}
below.

~~~~~~~~~~
                  Probed flow IPc2 to IPs1
 +---------------------------------------------------------+
 |                                                         |
 |   From IPc1 to IPs1                                     v
 |   +--------+          Client to Server Flow         +--------+
 |   |        | =====================================> |        |
 +-- | Client |                                        | Server |
     |        | <===================================== |        |
     +--------+          Server to Client Flow         +--------+
                                              From IPc1* to IPs1*
~~~~~~~~~~
{: #examplequic title="Identifying Unidirectional Flows in QUIC"}

The client initiates the QUIC connection by sending packets towards the server.
The server can then reply to the client with other packets and if the handshake
succeeds, the connection is established. This process serves at establishing the
connection path between end-hosts. Still, this "path" actually consists in two
independent UDP flows. Each host has its own view of i) the 4-tuple used to send
packets and ii) the 4-tuple on which it receives packets. While the 4-tuple used
by the client to send packets may be the same as the one seen and used by the
server, it is not necessarily the case with the presence of in-network
middleboxes that may alter the 4-tuple of packets (e.g., NATs). To further
exphasize this flow asymmetry, QUIC embeds a path validation mechanism
{{I-D.ietf-quic-transport}} that checks if a host can reach its peer through a
given 4-tuple. This process is unidirectional, i.e., the sender checks that it
can reach its receiver, but not the reverse. A host receiving a PATH_CHALLENGE
frame on a new 4-tuple may in turn initiates a path validation, but this is up
to the peer.

A QUIC connection is a collection of unidirectional flows, or uniflows. A plain
QUIC one has one main uniflow from client to server and one main uniflow from
server to client, each having their own (set of) Connection ID(s). These
uniflows are host-specific, i.e., the uniflow(s) from A to B are different from
the ones from B to A. This potentially enables the use of unidirectional
networks such as satellites, unapplicable when using TCP.


Beyond Connection Migration
---------------------------

Unlike TCP {{RFC0793}}, QUIC is not bound to a particular 4-tuple during the
lifetime of a connection. A QUIC connection is identified by a (set of)
Connection ID(s), placed in the public header of each QUIC packet. This
enables hosts to continue the connection even if the 4-tuple changes due to,
e.g., NAT rebinding. This ability to shift a connection from one 4-tuple to
another is called Connection Migration. One of its use cases is fail-over when
the IP address in use fails but another one is available. A device losing the
WLAN connectivity can then continue the connection over its cellular interface,
for instance.

A QUIC peer can thus start on a given set of uniflows, denoted as the initial
uniflows, and end on another ones. However, the current QUIC design
{{I-D.ietf-quic-transport}} assumes that only one pair of uniflows is in use for
a given connection. The specification does not support means to distinguish path
migration from simultaneous usage of available uniflows for a given connection.

This document fills that void. Concretely, instead of switching the 4-tuple for
the whole connection, this draft first proposes mechanisms to communicate
endhost addresses to the peer. It then leverages the address validation process
with the PATH_CHALLENGE and PATH_RESPONSE frames proposed in
{{I-D.ietf-quic-transport}} to verify if additional addresses advertised by the
communicating host are available and actually belong to it. In this case, those
addresses can be used to initiate new uniflows to spread packets over several
networks following a traffic distribution policy that is out of scope of this
document.

TODO: Add a companion document discussing the packet scheduling and path
management considerations.

When multiple paths are available, different delays may be experienced as a
function of the initial network path selected for the establishment of the QUIC
connection. The first version of the specification does not discuss
considerations related to the selection of the initial path to place the
connection.

The example of {{examplempquic}} shows a possible data exchange between a
dual-homed client performing a request fitting in two packets and a single-homed
server. Notice that the Uniflow ID used by a host over a specific network path
is not necessarily the same as the one used by the remote peer. In the presented
example, the phone sends packets over WLAN on uniflow 0 and over LTE on uniflow
1, while the packets sent by the server over WLAN are on uniflow 2 and over LTE
are on uniflow 1.


~~~~~~~~~~
Server                        Phone                        Server
via WLAN                                                  via LTE
-------                      -------                        -----
  | Pkt(DCID=A,PN=5,frames=[    |                             |
  |  STREAM("Request (1/2)")])  | Pkt(DCID=B,PN=1,frames=[    |
  |<----------------------------|  STREAM("Request (2/2)")])  |
  | Pkt(DCID=E,PN=1,frames=[    |--------                     |
  |  ACK(LargestAcked=5)])      |       |----------           |
  |---------------------------->|                 |--------   |
  | Pkt(DCID=E,PN=2,frames=[    |                         |-->|
  |  STREAM("Response 1")])     | Pkt(DCID=D,PN=1,frames=[    |
  |---------------------------->|  MPACK(UID=1,LargestAck=1), |
  |                             |  STREAM("Response 2")])  ---|
  | Pkt(DCID=A,PN=6,frames=[    |                 ---------|  |
  |  MPACK(UID=2,LargestAck=2), |       ----------|           |
  |  MPACK(UID=1,LargestAck=1)])|<------|                     |
  |<----------------------------|                             |
  | Pkt(DCID=A,PN=7,frames=[    | Pkt(DCID=D,PN=2,frames=[    |
  |  STREAM("Response 3")])     |  STREAM("Response 4")])     |
  |---------------------------->|                         ----|
  |                             |                   ------|   |
  |            ...              |    ...  <---------|         |
~~~~~~~~~~
{: #examplempquic title="Dataflow with Multipath QUIC"}


The remaining of this section focuses on providing a high-level overview of the
multipath operations in QUIC.


Establishment of a Multipath QUIC Connection
--------------------------------------------

A Multipath QUIC connection starts like a regular QUIC connection. A
cryptographic handshake takes place with CRYPTO frames and follows the classical
process {{I-D.ietf-quic-transport}} {{I-D.ietf-quic-tls}}. It is during that
handshake that the multipath capability is negotiated between hosts. This is
performed using the `max_sending_uniflow_id` transport parameter, where both
hosts advertise their support for the multipath extension. When a host includes
this transport parameter, it advertises its support of the multipath extensions
over the connection. If one of the hosts does not advertise the
`max_sending_uniflow_id` transport parameter, the QUIC connection will not use
the multipath extensions presented in this document.

The handshake is performed on two given uniflows (one for each direction). These
uniflows are called the Initial Uniflows and are identified by Uniflow ID 0.

Notice that a host advertising a value of 0 for the `max_sending_uniflow_id`
transport parameter indicates that it does not want additional uniflows to send
packets, but it still supports the multipath extensions. Such situation might be
useful when the host does not require multiple networks paths for packet sending
but still wants to let the peer use multiple uniflows to reach it.

The usage of multiple uniflows relies on the ability to use several Connection
IDs over a same QUIC connection. Therefore, zero-length Connection IDs MUST NOT
be used if the multipath extensions are enabled and the peer advertises a value
different from 0 for the `max_sending_uniflow_id` transport parameter.


Architecture of Multipath QUIC
------------------------------

To illustrate the architecture of a Multipath QUIC connection, consider the
{{uniflowsexample}}.

~~~~~~~~~~
+--------+          CID A - Uniflow ID 1          +--------+
|        | =====================================> |        |
|        |          CID B - Uniflow ID 0          |        |
|        | =====================================> |        |
|        |                                        |        |
| Client |          CID C - Uniflow ID 0          | Server |
|        | <===================================== |        |
|        |          CID D - Uniflow ID 1          |        |
|        | <===================================== |        |
|        |          CID E - Uniflow ID 2          |        |
+--------+ <===================================== +--------+
~~~~~~~~~~
{: #uniflowsexample title="An example of uniflow distribution over a Multipath QUIC connection"}

Once established, a Multipath QUIC connection consists in one or more uniflow(s)
from the client to the server and one or more uniflow(s) from the server to the
client. The number of uniflows in one direction can be different from the one in
the other direction. The example in {{uniflowsexample}} shows two
uniflows from the client to the server and three uniflows from the server to the
client. From the end-hosts' viewpoint, they observe two kinds of uniflows:

* Sending uniflows: uniflows on which the host can send packets over it

* Receiving uniflows: uniflows on which the host can receive packets over it

Reconsidering the example in {{uniflowsexample}}, the client has two
sending uniflows and three receiving uniflows. The server has three sending
uniflows and two receiving uniflows. There is thus a one-to-one mapping between
the sending uniflows of a host and the receiving uniflows of its peer. A uniflow
is seen as a sending uniflow from the sender's perspective and as a receiving
uniflow from the receiver's viewpoint.

Uniflows may share a common network path, but this is not mandatory.

Each uniflow is associated with a specific four-tuple and identified by a
Uniflow ID, as shown in {{architectural}}.

~~~~~~~~~~
    +-----------------------------------------------------+
    |                      Connection                     |
    |   +-----------+ +-----------+ ... +-------------+   |
    |   |  Sending  | |  Sending  | ... |   Sending   |   |
    |   | Uniflow 0 | | Uniflow 1 |     | Uniflow N-1 |   |
    |   |   Tuple   | |   Tuple'  | ... |    Tuple"   |   |
    |   | UDCID set | | UDCID set'| ... |  UDCID set" |   |
    |   |    PNS    | |    PNS'   |     |     PNS"    |   |
    |   +-----------+ +-----------+ ... +-------------+   |
    |   +-----------+ +-----------+ ... +-------------+   |
    |   | Receiving | | Receiving | ... |  Receiving  |   |
    |   | Uniflow 0 | | Uniflow 1 |     | Uniflow M-1 |   |
    |   |   Tuple   | |   Tuple'  | ... |    Tuple"   |   |
    |   | USCID set | | USCID set'|     |  USCID set" |   |
    |   |    PNS    | |    PNS'   |     |     PNS"    |   |
    |   +-----------+ +-----------+ ... +-------------+   |
    +-----------------------------------------------------+
~~~~~~~~~~
{: #architectural title="Architectural view of Multipath QUIC for a host having N sending uniflows and M receiving uniflows"}

As described before, a Multipath QUIC connection starts using the Initial
Uniflows, identified by Uniflow ID 0. It can then spread packets over several
uniflows. Each uniflow has its (set of) Uniflow Connection ID(s) (UCID) packets
use to explicitly mark they belong to. Depending on the direction of the
uniflow, the host keeps either the Uniflow Source Connection ID (USCID, for the
receiving uniflows) or the Uniflow Destination Connection ID (USCID, for the
sending uniflows). Notice that the (set of) UDCID(s) of a sending uniflow of a
host is the same as the (set of) USCID(s) of the corresponding receive uniflow
of the remote. Preventing the linkability of different uniflows is an important
requirement for the multipath extensions {{I-D.huitema-quic-mpath-req}}. Using
UCIDs as implicit uniflow identifiers makes this linkability harder than having
explicit signaling as in the early version of this draft and does not require
public header change to keep invariants {{I-D.ietf-quic-invariants}}.

In addition to the UCIDs, some additional information is kept for each uniflow.
The Uniflow ID identifies the uniflow at the frame level and ensures uniqueness
of the nonce (see {{nonce-considerations}} for details) while limiting the
number of concurrently used uniflows. A congestion window is maintained for each
sending uniflow. Hosts can also collect network measurements on a per-uniflow
basis, such as one-way delay measurements and lost packets.


Path Establishment
------------------

The `max_sending_uniflow_id` transport parameter exchanged during the
cryptographic handshake fixes a upper bound on the number of sending uniflows a
host want to support. Then, hosts provide to their peer Uniflow Connection IDs
to use on uniflows. Unlike Multipath TCP {{RFC6824}}, both hosts dynamically
control how many sending uniflows can currently be in use by the peer, i.e., the
number of different Uniflow IDs proposed to the peer. While the advertisement of
the upper bound of sending paths is a sender-oriented process, the actual
initation of a uniflow is a receiver-oriented process, as the sender needs a
UCID communicated by the receiver before using a uniflow. Notice that the peers
might advertise different values for the `max_sending_uniflow_id` transport
parameters to their peer, setting different upper bounds to the sending and
receiving uniflows of each host.

Hosts initiate the creation of their receiving uniflows by sending
MP_NEW_CONNECTION_ID frames (see {{mpnewconnectionidsec}}) which can be seen as
an extended version of the NEW_CONNECTION_ID frame. This frame provides to the
peer a UDCID for a given sending uniflow. Upon reception of the
MP_NEW_CONNECTION_ID frame, a host can start using the proposed sending uniflow
having the referenced Uniflow ID by marking sent packets with the provided
UDCID. Therefore, once a host sent such MP_NEW_CONNECTION_ID frame, it MUST be
ready to receive packets from that Uniflow ID with the proposed UCID. As frames
are encrypted, adding new uniflows over a QUIC connection does not leak
cleartext identifiers {{I-D.huitema-quic-mpath-req}}.

A server might provide several Uniflow Connection IDs for the same Uniflow IDs
with multiple MP_NEW_CONNECTION_ID frames. This can be useful to cope with
migration cases, as described in {{path-migration}}. Multipath QUIC is
asymmetrical. Any host can start using new sending uniflows once their
corresponding UDCIDs have been provided by the remote peer.

Hosts are not able to use new uniflows as long as their peer does not send
MP_NEW_CONNECTION_ID frames. To limit the delay between the connection
establishment and the usage of additional uniflows, hosts should send those
frames as soon as possible, i.e., just after the 0-RTT handshake packet.

Sending useful data on a fresh new sending uniflow might lead to poor
performance as the network path used by the QUIC uniflow might not be usable. A
typical case is when a server wants to initiate a new sending uniflow to a
client behind a NAT. The client would possibly never receive this packet,
leading to connectivity issues on that uniflow. In addition, an opportunistic
usage of network paths might also lead to possible attacks, such as a client
advertising the IP address of a victim hoping that the server will flood it. To
avoid these issues, a remote address MUST have been validated as described in
{{I-D.ietf-quic-transport}} before associating it on a sending uniflows.

Because attaching to new networks may be volatile and an endpoint does not have
full visibility on multiple paths that may be available (e.g., hosts connected
to a CPE), an Multipath QUIC capable endhost SHOULD advertise a
`max_sending_uniflow_id` value of at least 4 and SHOULD propose at least 4
receiving uniflows to its peer.


Exchanging Data over Multiple Paths
-----------------------------------

A QUIC packet acts as a container for one of more frames. Multipath QUIC uses
the same STREAM frames as QUIC to carry data. A byte offset is associated to the
data payload. One of the key design decision of (Multipath) QUIC is that frames
are independent of the packets carrying them. This implies that a frame
transmitted over one uniflow could be retransmitted later on another uniflow
without any change.

The uniflow on which data is sent is a packet-level information. This means a
frame can be sent regardless of the uniflow of the packet carrying it.
Furthermore, because the data offset is a frame-level information, there is no
need to define additional sequence numbers to cope with reordering across
uniflows, unlike Multipath TCP {{RFC6824}} that uses a Data Sequence Number at
the Multipath TCP level. Other flow control considerations like the stream
receive window advertised by the MAX_STREAM_DATA frame remain unchanged when
there are multiple sending uniflows.

However, Multipath QUIC might face reordering at packet-level when using paths
having different latencies. The presence of different Uniflow Connection IDs
ensures that the packets sent over a given uniflow will contain monotonically
increasing packet numbers. To ensure more flexibility and potentially to reduce
the ACK block section of the (MP_)ACK frame when aggregating bandwidth of
uniflows exhibiting different network characteristics, each uniflow keeps its
own monotonically increasing Packet Number space. This potentially allows
sending up to 2 * 2^62 * 2^62 packets on a QUIC connection since each uniflow
has its own packet number space.

With the introduction of multiple uniflows, there is a need to acknowledge them
separately. The Initial Uniflows (with Uniflow ID 0) are still acknowledged with
regular ACK frames, such that no modification is introduced in a core frame. For
the other uniflows, the multipath extensions introduce a MP_ACK frame which
contains the same fields as the ACK frame, plus a Uniflow ID field indicating
which receving uniflow the host acknowledges. To better explain this, let us
consider the situation illustrated in {{ack-uniflows}}.

~~~~~~~~~~
Sending uniflow 0 - CID A      |    Receiving uniflow 0 - CID A
Sending uniflow 1 - CID B      |    Receiving uniflow 1 - CID B
Receiving uniflow 0 - CID C    |      Sending uniflow 0 - CID C
Receiving uniflow 1 - CID D    |      Sending uniflow 1 - CID D
Receiving uniflow 2 - CID E    |      Sending uniflow 2 - CID E

Client                                                   Server
------                                                   ------
   |                                                        |
   |       Pkt(DCID=B,PN=42,frames=[STREAM("Request")])     |
   |---------------------------                             |
   |                          |---------------------------->|
   |                                                        |
   |  Pkt(DCID=E,PN=58,frames=[                             |
   |   STREAM("Response"), MP_ACK(UID=1,LargestAcked=42)])  |
   |                          ------------------------------|
   |<-------------------------|                             |
   |                                                        |
~~~~~~~~~~
{: #ack-uniflows title="Acknowledging packets sent on uniflows"}

Here five uniflows are in use, two in the client to server direction and three
in the reverse one. In this case, the client first sends a packet on its sending
uniflow 1 (linked to the CID B). The server hence receives the packet on its
receiving uniflow 1. Therefore, it generates a MP_ACK frame for the Uniflow ID 1
and transmits it to the client. The server has the freedom to choose any of its
sending unflows to transmit this frame. In the provided situation, the server
sends its packets on its sending uniflow 2 (and the client will thus receives
this packet on its receiving uniflow 2). Similarly, by looking at
{{examplempquic}} again, packets that were sent over a given uniflow (e.g., the
"Response 2" packet on server's sending uniflow 1 with DCID D) can be
acknowledged on another uniflow (here, client's sending uniflow 0 with DCID A)
that does not correspond to the same underlying network to limit the latency due
to (MP_)ACK transmissions on high-latency network paths and to enable the usage
of unidirectional networks. Such scheduling decision would not have been
possible in Multipath TCP {{RFC6824}} which must acknowledge data on the
(bidirectional) path it was received on.


Exchanging Addresses
--------------------

When a multi-homed mobile device connects to a dual-stacked server using its
IPv4 address, it is aware of its local addresses (e.g., the Wi-Fi and the
cellular ones) and the IPv4 remote address used to establish the QUIC
connection. If the client wants to create new uniflows and use them over the
IPv6 network, it needs to learn the other addresses of the remote peer.

This is possible with the ADD_ADDRESS frames that are sent by a Multipath QUIC
host to advertise its current addresses. Each advertised address is identified
by an Address ID. The addresses attached to a host can vary during the lifetime
of a Multipath QUIC connection. A new ADD_ADDRESS frame is transmitted when a
host has a new address. This ADD_ADDRESS frame is protected as other QUIC
control frames, which implies that it cannot be spoofed by attackers. The
communicated address MUST first be validated by the receiving host before it
starts using it. This process ensures that the advertised address actually
belongs to the peer and that the peer can receive packets sent by the host on
the provided address. It also prevents hosts from launching amplification
attacks to a victim address.

If the client is behind a NAT, it could announce a private address in an
ADD_ADDRESS frame. In such situations, the server would not be able to validate
the communicated address. The client might still use its NATed addresses to
start using its sending uniflows. To enable the server to make the link between
the private and the public addresses and hence conciliate the different 4-tuple
views, Multipath QUIC provides the UNIFLOWS frame that lists the current active
sending Uniflow IDs along with their associated local Address ID. Notice that a
host might also discover the public addresses of its peer by observing its
remote IP addresses associated to the connection.

Likewise, the client may be located behind a NAT64. As such it may announce an
IPv6 address in an ADD_ADDRESS frame, that will be received over IPv4 by an
IPv4-only server. The server should not discard that address, even if it is not
IPv6-capable.

TODO maybe move this paragraph in a companion draft

An IPv6-only client may also receive from the server an ADD_ADDRESS frame which
may contain an IPv4 address. The client should rely on means, such as
{{RFC7050}} or {{RFC7225}}, to learn the IPv6 prefix to build an IPv4-converted
IPv6 address.

A receiving uniflow is active as soon as the host has sent the
MP_NEW_CONNECTION_ID frames proposing the corresponding Uniflow Connection IDs
to its peer. A sending uniflow is active when it has received its Uniflow
Connection IDs and is bound to a validated 4-tuple. The UNIFLOWS frame indicates
the local Address IDs that the uniflow uses from the sender's perspective. With
this information, the remote can then validate the public address and associate
the advertised with the perceived addresses.

TODO maybe move this paragraph in a companion draft

Hosts that are connected behind an address sharing mechanism may collect the
external IP address and port numbers assigned to the hosts and then use their
addresses in the ADD_ADDRESS. Means to gather such information include, but not
limited to, UPnP IGD, PCP, or STUN.


Coping with Address Removals
----------------------------

During the lifetime of a QUIC connection, a host might lose some of its
addresses. A concrete example is a smartphone going out of reachability of a
Wi-Fi network or shutting off one of its network interfaces. Such address
removals are advertised using REMOVE_ADDRESS frames. The REMOVE_ADDRESS frame
contains the Address ID of the lost address previously communicated through
ADD_ADDRESS. Notice that because a given Address ID might encounter several
events that need to be ordered (e.g., ADD_ADDRESS, REMOVE_ADDRESS and
ADD_ADDRESS again), both ADD_ADDRESS and REMOVE_ADDRESS frames include an
Address ID related Sequence Number.


Path Migration {#path-migration}
--------------

TODO: should this whole section be part of a companion draft?

At a given time, a Multipath QUIC endpoint gathers a set of active sending and
receiving uniflows, each associated to a 4-tuple. To address privacy issues due
to the linkability of addresses with Connection IDs, hosts should avoid changing
the 4-tuple used by a sending uniflow. There still remain situations where this
change is unavoidable. These can be categorized into two groups: host-aware
changes (e.g., network handover from Wi-Fi to cellular) and host-unaware changes
(e.g., NAT rebinding).

For the host-aware case, let us consider the case of a Multipath QUIC connection
where the client is a smartphone with both Wi-Fi and cellular. It advertised
both addresses and the server currently enables only one client's sending
uniflow, the initial one. The Initial Uniflow uses the Wi-Fi address. Then, for
some reason, the Wi-Fi address becomes unusable. To preserve connectivity, the
client might then decide to use the cellular address for its Initial sending
uniflow. It thus sends a REMOVE_ADDRESS announcing the loss of the Wi-Fi address
and a UNIFLOWS frame to inform that its Initial sending uniflow is now using the
cellular address. If the cellular address validation succeeds (which could have
been done as soon as the cellular address was advertised), the server can
continue exchanging data through the cellular address.

TODO: I don't think we have to change the Uniflow ID, we can just rely on
(MP_)NEW_CONNECTION_ID and (MP_)RETIRE_CONNECTION_ID -- hence, not sure the
PATH_UPDATE frame is required, but will depend on the use cases

TODO: update following paragraph to remove PATH_UPDATE

However, both server and client might want to change the uniflow used on the
cellular address for privacy concerns. If the server provides an additional
uniflow (e.g., with Uniflow ID 1) through MP_NEW_CONNECTION_ID frame at the
beginning of the connection, the client can perform the network path change
directly and avoid using the Initial Uniflow Connection ID on the cellular
network. This can be done using the PATH_UPDATE frame. It can indicate that the
host stopped to use the Initial sending uniflow to use the one with Uniflow ID
1 instead. This frame is placed in the first packet sent to the new sending
uniflow with its corresponding UCID. The client can then send the REMOVE_ADDRESS
and UNIFLOWS frames on this new uniflow. Compared to the previous case, it is
harder to link the uniflows with the IP addresses to observe that they belong to
the same Multipath QUIC connection.

For the host-unaware case, the situation is similar. In case of NAT rebinding,
the server will observe a change in the 2-tuple (source IP, source port) of the
receiving uniflow of the packet. The server first validates that the 2-tuple
actually belongs to the client {{I-D.ietf-quic-transport}}. If it is the case,
the server can send a PATH_UPDATE frame on a previously communicated but unused
Uniflow ID. The client might have sent some packets with a given UCID on a
different 4-tuple, but the server did not use the given UCID on that 4-tuple.
Because some on-path devices may rewrite the source IP address to forward
packets via the available network attachments (e.g., a host located behind a
multi-homed CPE), the server may inadvertently conclude that a uniflow is not
anymore valid leading thus to frequently sending PATH_UPDATE frames as a
function of the traffic distribution scheme enforced by the on-path device. To
prevent such behavior, the server SHOULD wait for at least X seconds to ensure
this is about a connection migration and not a side effect of an on-path
multi-interfaced device.


Sharing Path Policies
---------------------

TODO: should this whole section be part of a companion draft?

Some access networks are subject to a volume quota. To prevent a peer from
aggressively using a given network path while available resources can be freely
grabbed using existing uniflows, it is desirable to support a signal to indicate
to a remote peer how it must place data into available uniflows. An approach may
consist in indicating in an ADD_ADDRESS the type of the interface (e.g.,
cellular, WLAN, fixed) through the interface-type field. The remote peer may
rely on interface-type to select the uniflow to be used for sending data. For
example, fixed interfaces will be preferred over WLAN and cellular interfaces,
and WLAN interface will be preferred over cellular interface.

This information might also be used to avoid draining battery for some devices.


Congestion Control
------------------

TODO: should this whole section be part of a companion draft?

The QUIC congestion control scheme is defined in {{I-D.ietf-quic-recovery}}.
This congestion control scheme is not suitable when several sending uniflows are
active. Using the congestion control scheme defined in
{{I-D.ietf-quic-recovery}} with Multipath QUIC would result in unfairness. Each
sending uniflow of a Multipath QUIC connection MUST have its own congestion
window. The windows of the different sending uniflows MUST be coupled together.
Multipath TCP uses the LIA congestion control scheme specified in {{RFC6356}}.
This scheme can immediately be adapted to Multipath QUIC. Other coupled
congestion control schemes have been proposed for Multipath TCP such as
{{OLIA}}.


Mapping Uniflow IDs to Connection IDs
=====================================

As described in the overview section, hosts need to identify on which uniflows
packets are sent. The Uniflow ID must then be inferred from the public header.
This is done by using the Destination Connection ID field of Short Header
packets.

The Initial Uniflow Connection IDs are determined during the cryptographic
handshake and actually correspond to both Connection IDs in the current
single-path QUIC design {{I-D.ietf-quic-transport}}. The Uniflow Connection IDs
of the other uniflows are determined when the MP_NEW_CONNECTION_ID frames are
exchanged. Additional Uniflow Connection IDs can be provided with the regular
NEW_CONNECTION_ID frames.

Hosts MUST ensure that all advertised Uniflow Connection IDs are available
for the whole connection lifetime, unless they have been retired by their peer
in the meanwhile by the reception of a (MP_)RETIRE_CONNECTION_ID. Once a host
sends a (MP_)NEW_CONNECTION_ID frame containing a UCID, it can start receiving
packets with the advertised Connection ID as belonging to the corresponding
receiving uniflow. Hosts MUST wait until the reception of a MP_NEW_CONNECTION_ID
frame before sending packets on the corresponding (non-initial) sending
uniflows.

Hosts MUST ensure that they can receive packets coming from their peer using the
UCIDs they proposed in the (MP_)NEW_CONNECTION_ID frames they sent and associate
them with the corresponding receiving Uniflow ID. Upon reception of the
(MP_)NEW_CONNECTION_ID frame, hosts MUST acknowledge it and MUST store the
advertised Uniflow Destination Connection ID and the Uniflow ID of the proposed
sending uniflow.

Hosts MUST NOT send MP_NEW_CONNECTION_ID frames with a Uniflow ID greated than
the value of `max_sending_uniflow_id` advertised by their peer.


Using Multiple Paths {#spec}
====================

This section describes in details the multipath QUIC operations.


Multipath Negotiation
---------------------

The Multipath negotiation takes place during the cryptographic handshake with
the `max_sending_uniflow_id` transport parameter. A QUIC connection is initially
single-path in QUIC, and all packets prior to handshake completion MUST be
exchanged over the Initial Uniflows. During this process, hosts advertise their
support for multipath operations. When a host advertises a value for the
`max_sending_uniflow_id` transport parameter, it indicates that it supports the
multipath extensions, i.e., the extensions defined in this document (not to be
mixed with the availability of local multiple network paths). If any host does
not advertise the `max_sending_uniflow_id` transport parameter, multipath
extensions are disabled and this leads to a single-path QUIC connection over the
Initial Uniflows.

Notice that a host advertising a value of 0 for the `max_sending_uniflow_id`
transport parameter indicates that it does not want additional uniflows to send
packets, but it still supports the multipath extensions. Such situation might be
useful when the host does not require multiple networks paths for packet sending
but still wants to let the peer use multiple uniflows to reach it.


### Transport Parameter Definition {#tp-definition}

An endhost MAY use the following transport parameter:

max_sending_paths (0x0020):

 :  Indicate the support of the multipath extension presented in this document,
    regardless of the carried value. Its integer value puts a upper bound on the
    number of sending uniflows the host advertising the value is ready to
    support. If absent, this means that the host does not agree to use the
    multipath extension over the connection.


Coping with Additional Remote Addresses
---------------------------------------

The usage of multiple networks paths is often done using multiple IP addresses.
For instance, a smartphone willing to use both Wi-Fi and LTE will use the
corresponding addresses assigned by these networks. It can then safely send
packets to a previously-used IP address of a server. The server can receive
packets sourced with different IP addresses, but it MUST first validate the new
remote IP addresses before starting sending packets to those addresses.

Similarly, additional addresses could be communicated using ADD_ADDRESS frames.
Such addresses MUST be validated before starting to send packets to them. This
requirement is explained in {{validation_address}}.

The validation of an address is performed with PATH_CHALLENGE and PATH_RESPONSE
frames as described in {{I-D.ietf-quic-transport}}. A validated address MAY be
cached for a given host for a limited amount of time.


Receiving Uniflow State
-----------------------

When proposing uniflows to their peer, hosts need to maintain some state for
their receiving uniflows. This state is created upon the sending of a first
MP_NEW_CONNECTION_ID frame proposing the corresponding Uniflow ID. Then, hosts
MUST ensure that they can associate a packet containing one of the advertised
UCIDs with its corresponding receiving uniflow at any time. As long as there
is still one active Uniflow Connection ID for this receiving uniflow (i.e., one
UCID which was not retired yet using a MP_RETIRE_CONNECTION_ID), the host must
expect receiving packets over the uniflow. Once created, hosts MUST keep the
following receiving uniflow information:

Uniflow ID:

 : Encoded as a integer. It uniquely identifies the receiving uniflow in the
   connection. This value is immutable.

Uniflow Source Connection IDs:

 : Possible values for the Connection ID field of packets belonging to this
   receiving uniflow. This value contains the set of active UCIDs that where
   advertised in previously sent MP_NEW_CONNECTION_ID frames. Notice that this
   set might be empty, e.g., when all advertised UCIDs have been retired by the
   peer.

Packet Number Space:

 : Receiving uniflow specific monotonically increasing packet number space,
   dedicated for the packet receptions. Packet number considerations described
   in {{I-D.ietf-quic-transport}} apply within a given receiving uniflow.

Associated 4-tuple:

 : The tuple (sIP, dIP, sport, dport) currently observed to receive packets over
   this uniflow. This value is mutable, because a host might receive a packet
   with a different (possibly) validated remote address and/or port than the one
   previously recorded. If a host observes a change in the 4-tuple of the
   receiving uniflow, it SHOULD send a MP_NEW_CONNECTION_ID with an increased
   Retire Prior To field to make the peer change the Uniflow Connection ID.

Associated local Address ID:

 : The Address ID advertised (or to be advertised) in ADD_ADDRESS frames sent by
   the peer corresponding to the local IP used to receive packets. This helps
   generating UNIFLOWS frames advertising the mapping between uniflows and
   addresses. The addresses on which the connection was established have Address
   ID 0.

Performance metrics:

 : Basic statistics such as the number of packets received.



Sending Uniflow State
---------------------

During the Multipath QUIC connection, hosts maintain some state for sending
uniflows. Information about the sending uniflow that hosts are required to store
depends on its state. The possible sending uniflow states are depicted in
{{snd_uniflow_state}}.

TODO: intermediate state for address validation?

~~~~~~~~~~
      o
      |
      | receive a first NEW_CONNECTION_ID
      |    with the associated Path ID
      |
      v       path usage over a validated 4-tuple
+----------+ ------------------------------------> +----------+
|  UNUSED  |                                       |  ACTIVE  |
+----------+ <------------------------------------ +----------+
                 address change or retired UCID
~~~~~~~~~~
{: #snd_uniflow_state title="Finite-State Machine of sending uniflows"}

Once a sending uniflow has been proposed by the peer in a received
MP_NEW_CONNECTION_ID frame, it is in the UNUSED state. In this situation, hosts
MUST keep the following sending uniflow information:

Uniflow ID:

 : Encoded as a integer. It uniquely identifies the sending uniflow in the
   connection. This value is immutable.

Uniflow Source Connection IDs:

 : Possible values for the Connection ID field of packets belonging to this
   sending uniflow. This value contains the set of active UCIDs that where
   advertised in previously received MP_NEW_CONNECTION_ID frames. Notice that
   this set might be empty, e.g., when all advertised UCIDs have been retired.

Sending Uniflow State:

 : The current state of the sending uniflow, being one of the values presented
   in {{snd_uniflow_state}}.


When the host wants to start using the sending uniflow over a validated address,
the sending uniflow goes to the ACTIVE state. This is the state where a sending
uniflow can be used to send packets. Having a uniflow in ACTIVE state only
guarantees that it can be used, but the host is not forced to. In addition to
the fields required in the UNUSED state, the following elements MUST be tracked:

Packet Number Space:

 : Sending uniflow specific monotonically increasing packet number space,
   dedicated for the packet sendings. Packet number considerations described
   in {{I-D.ietf-quic-transport}} apply within a given sending uniflow.

Associated 4-tuple:

 : The tuple (sIP, dIP, sport, dport) currently used to packets over this
   uniflow. This value is mutable, as the host might decide to change its local
   (or remote) address and/or port. A host that changes the 4-tuple of a
   sending uniflow SHOULD migrate it.

Associated (local Address ID, remote Address ID) tuple:

 : Those identifiers come from the ADD_ADDRESS sent (local) and received
   (remote). This enables a host to temporarily stop using a sending uniflow
   when, e.g., the remote Address ID is declared as lost in a REMOVE_ADDRESS.
   The addresses on which the connection was established have Address ID 0. The
   reception of UNIFLOWS frames helps hosts associate the remote Address ID used
   by the sending uniflow.

Performance metrics:

 : Basic statistics such as one-way delay or the number of packets sent. This
   information can be useful when a host needs to perform packet scheduling
   decisions and flow control management.


It might happen that a sending path is temporarily unavailable, because one of
the endpoint's addresses is no more available or because the host retired all
the UCIDs of the sending uniflow. In such cases, the path goes back to the
UNUSED state. In that state, the host MUST NOT send non-probing packets on it.
At this state, the host might want to restart using the uniflow over a validated
4-tuple, switching the uniflow state back to the ACTIVE state. However, its
congestion window MUST be restarted and its performance metrics SHOULD be reset.


Dynamic Management of Paths
---------------------------

TODO: merge with previous section?

Both hosts determine how many sending uniflows their peer can use over a
Multipath QUIC connection. It is their responsibility to propose Uniflow IDs
with corresponding UCIDs that could be used by the peer. A host can propose
several different Uniflow Connection IDs with MP_NEW_CONNECTION_ID frames for a
given Uniflow ID. This can be useful in migration scenarios.


Losing Addresses
----------------

During the lifetime of a connection, a host might lose addresses, e.g., a
network interface that was shut down. All the ACTIVE sending uniflows that were
using that local address MUST stop sending packets from that address. To
advertise the address losses to the peer, the host MUST send a REMOVE_ADDRESS
frame indicating which local Address IDs has been lost. The host MUST also
send a UNIFLOWS frame indicating the status of the remaining ACTIVE paths.

Upon reception of the REMOVE_ADDRESS, the receiving host MUST stop using the
ACTIVE sending uniflows affected by the address removal.

Hosts MAY reuse one of these sending uniflows by changing the assigned 4-tuple.
In this case, it MUST send a UNIFLOWS frame describing that change.


Scheduling Strategies
---------------------

TODO: move to a companion draft?

The current QUIC design {{I-D.ietf-quic-transport}} offers a two-dimensional
scheduling space, i.e., which frames will be packed inside a given packet. With
the simultaneous use of several sending uniflows, a third dimension is added,
i.e., the sending uniflow on which the packet will be sent. This dimension can
have a non negligible impact on the operations of Multipath QUIC, especially if
the available sending uniflows exhibit very different network characteristics.

The progression of the data flow depends on the reception of the MAX_DATA and
MAX_STREAM_DATA frames. Those frames SHOULD be duplicated on several or all
ACTIVE sending uniflows. This would limit the head-of-line blocking issue due to
the transmission of the frames over a slow or lossy network path.

The sending path on which (MP_)ACK frames are sent impacts the peer. The
(MP_)ACK frame is notably used to determine the latency of a combination of
uniflows. In particular, the peer can compute the round-trip-time of the
combination of its sending uniflow with its receive one. The peer would compute
the latency as the sum of the forward delay of the acknowledged uniflow and the
return delay of the uniflow used to send the (MP_)ACK frame. Choosing between
acknowledging packets symmetrically (on uniflow B to A if packet was sent on A
to B) or not is up to the implementation, if only possible. However, hosts
SHOULD keep a consistent acknowledgement strategy. Selecting a random uniflow to
acknowledge packets may affect the performance of the connection. Notice that
the inclusion of a timestamp field in the (MP_)ACK frame, as proposed by
{{I-D.huitema-quic-1wd}}, may help hosts estimate more precisely the one-way
delay of each uniflow, therefore leading to improved scheduling decisions.
Unlike MAX_DATA and MAX_STREAM_DATA, (MP_)ACK frames SHOULD NOT be
systematically duplicated on several sending uniflows as they can induce a large
network overhead.


New Frames
==========

To support the multipath operations, new frames have been defined to coordinate
hosts. This draft uses a type field containing the bitmask 0x20 to indicate that
the frame is related to multipath operations. Notice that all the defined frames
are only valid in 1-RTT packets, as the multipath extensions can be enabled upon
the completion of the connection handshake.


MP_NEW_CONNECTION_ID Frame {#mpnewconnectionidsec}
--------------------------

The MP_NEW_CONNECTION_ID frame (type=0x20) is an extension of the
NEW_CONNECTION_ID frame defined by {{I-D.ietf-quic-transport}}. It keeps its
ability to provide the peer with alternative Connection IDs that can be used to
break linkability when migrating connections. It also allows the host to
indicate which uniflow each communicated Connection ID is linked to.

The format of the MP_NEW_CONNECTION_ID frame is as follows.

~~~~~~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Uniflow ID (i)                      ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Sequence Number (i)                    ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Retire Prior To (i)                    ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Length (8)  |                                               |
+-+-+-+-+-+-+-+-+       Connection ID (8..160)                  +
|                                                             ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                                                               |
+                   Stateless Reset Token (128)                 +
|                                                               |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #mpnewconnectionid title="MP_NEW_CONNECTION_ID frame"}

Compared to the frame specified in {{I-D.ietf-quic-transport}}, a Uniflow ID
field of variable size is prefixed to associate the Uniflow ID with the
Connection ID. This frame can be sent by both hosts. Upon reception of the frame
with a specified Uniflow ID, the peer can update the related sending uniflow
state and store the communicated Connection ID as the Destination Connection ID
of the sending uniflow.

A host MUST NOT start using a uniflow as long as the peer have not proposed a
Destination Connection ID with a MP_NEW_CONNECTION_ID frame. To limit the delay
of the multipath usage upon handshake completion, hosts SHOULD send
MP_NEW_CONNECTION_ID frames for receive uniflows they allow using as soon the
connection establishment completes.

To cope with privacy issues, it should be hard to make the link between two
different connections or two different uniflows of a same connection by just
looking at the Connection ID contained in packets. Therefore, Uniflow Connection
IDs chosen by hosts MUST be random.


MP_RETIRE_CONNECTION_ID Frame
-----------------------------

The MP_RETIRE_CONNECTION_ID frame (type=0x21) is an extension of the
RETIRE_CONNECTION_ID frame defined by {{I-D.ietf-quic-transport}}. It keeps its
ability to indicate that the end-host will no longer use a Connection ID that
was issued by its peer. The multipath extensions link Connection IDs to
uniflows. Therefore, this frame must contain the Uniflow ID on which it applies.

The format of the MP_RETIRE_CONNECTION_ID frame is shown below.

~~~~~~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Uniflow ID (i)                      ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Sequence Number (i)                    ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #mpretireconnectionid title="MP_RETIRE_CONNECTION_ID frame"}

The frame is handled as described in {{I-D.ietf-quic-transport}} on a uniflow
basis.


MP_ACK Frame
------------

The MP_ACK frame (types 0x22 and 0x23) is an extension of the ACK frame defined
by {{I-D.ietf-quic-transport}}. It allows hosts to acknowledge packets that were
sent on non-initial uniflows.

The format of the MP_ACK frame is shown below.

~~~~~~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Uniflow ID (i)                      ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Largest Acknowledged (i)                 ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         ACK Delay (i)                       ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      ACK Range Count (i)                    ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      First ACK Range (i)                    ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         ACK Ranges (*)                      ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         [ECN Counts]                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #ack_frame title="ACK frame adapted to Multipath"}

Compared to the ACK frame, the MP_ACK frame is prefixed by a variable size
Uniflow ID field indicating to which uniflow the acknowledged packet sequence
numbers relate to.

Since frames are independent of packets, and the uniflow notion relates to the
packets, the (MP_)ACK frames can be sent on any uniflow, unlike Multipath TCP
{{RFC6824}} which is constrained to send ACKs on the same path.


ADD_ADDRESS Frame
-----------------

The ADD_ADDRESS frame (type=0x24) is used by a host to advertise its currently
reachable addresses.

The format of the ADD_ADDRESS frame is shown below.

~~~~~~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0|0|0|P|IPVers.|Address ID (8) |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Sequence Number (i)                   ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Interface T.(8)|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       IP Address (32/128)                   ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          [Port (16)]          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #add_address_frame title="ADD_ADDRESS Frame"}

The ADD_ADDRESS frame contains the following fields.

Reserved bits:

 : The three most-significant bits of the first byte are set to 0, and are
   reserved for future use.

P bit:

 : The fourth most-significant bit of the first byte indicates, if set, the
   presence of the Port field.

IPVers.:

 : The remaining four least-significant bits of the first byte contain the
   version of the IP address contained in the IP Address field.

Address ID:

 : An unique identifier for the advertised address for tracking and removal
   purposes. This is needed when, e.g., a NAT changes the IP address such that
   both hosts see different IP addresses for a same network path.

Sequence Number:

 : An Address ID related sequence number of the event. The sequence number space
   is shared with REMOVE_ADDRESS frames mentioning the same Address ID.

Interface Type:

 : Used to provide an indication about the interface type to which this address
   is bound. The following values are defined:

   * 0: fixed. Used as default value.
   * 1: WLAN
   * 2: cellular

IP Address:

 : The advertised IP address, in network order.

Port:

 : This optional field indicates the port number related to the advertised IP
   address. When this field is present, it indicates that a uniflow can use the
   2-tuple (IP addr, port).


Upon reception of an ADD_ADDRESS frame, the receiver SHOULD store the
communicated address for future use. The receiver MUST NOT send packets others
than validation ones to the communicated address without having validated it.
ADD_ADDRESS frames SHOULD contain globally reachable addresses. Link-local and
possibly private addresses SHOULD NOT be exchanged.


REMOVE_ADDRESS Frame
--------------------

The REMOVE_ADDRESS frame (type=0x25) is used by a host to signal that a
previously announced address was lost.

The format of the REMOVE_ADDRESS frame is shown below.

~~~~~~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+
|Address ID (8) |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Sequence Number (i)                   ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #remove_address_frame title="REMOVE_ADDRESS Frame"}

The REMOVE_ADDRESS frame contains the following fields.

Address ID:

 : The identifier of the address to remove.

Sequence Number:

 : An Address ID related sequence number of the event. The sequence number space
   is shared with ADD_ADDRESS frames mentioning the same Address ID. This help
   the receiver figure out that a REMOVE_ADDRESS might have been sent before an
   ADD_ADDRESS frame implying the same Address ID, even if for some reason the
   REMOVE_ADDRESS reaches the receiver after the newer ADD_ADDRESS one.


A host SHOULD stop using sending uniflows using the removed address and set them
in the UNUSED state. If the REMOVE_ADDRESS contains an Address ID that was not
previously announced, the receiver MUST silently ignore the frame.


UNIFLOWS Frame
--------------

The UNIFLOWS frame (type=0x26) communicates the uniflows' state of the sending
host to the peer. It allows the sender to communicate its active uniflows to the
peer in order to detect potential connectivity issue over uniflows.

The format of the UNIFLOWS frame is shown below.

~~~~~~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Sequence (i)                       ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      ReceivingUniflows (i)                  ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   ActiveSendingUniflows (i)                 ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               Receiving Uniflow Info Section (*)            ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                Sending Uniflow Info Section (*)             ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #uniflows_frame title="UNIFLOWS Frame"}

The UNIFLOWS frame contains the following fields.

Sequence:

 : A variable-length integer. This value starts at 0 and increases by 1 for each
   UNIFLOWS frame sent by the host. It allows identifying the most recent
   UNIFLOWS frame.

ReceivingUniflows:

 : The current number of receiving uniflows considered as being usable from the
   sender's point of view.

ActiveSendingUniflows:

 : The current number of sending uniflows in the ACTIVE state from the sender's
   point of view.

Receiving Uniflow Info Section:

 : Contains information about the receiving uniflows (there are
   ReceivingUniflows entries).

Sending Uniflow Info Section:

 : Contains information about the sending uniflows in ACTIVE state (there are
   ActiveSendingUniflows entries).


Both Receiving Uniflow Info and Sending Uniflow Info Sections share the same
format, which is shown below.

~~~~~~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Uniflow ID 0 (i)                      ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|LocAddrID 0 (8)|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                               ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Uniflow ID N (i)                      ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|LocAddrID N (8)|
+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #uniflow_info_section title="Uniflow Info Section"}

The fields in the Uniflow Info Section are the following.

Uniflow ID:

 : The Uniflow ID of the uniflow the sending host provides information about.

LocAddrID:

 : The local Address ID of the address currently used by the uniflow.


The Uniflow Info section only contains the local Address ID so far, but this
section can be extended later with other potentially useful information.


Extension of the Meaning of Existing QUIC Frames
================================================

The multipath extensions do not modify the wire format of existing QUIC frames.
However, they extend the meaning of a few of them while keeping this addition
transparent and consistent with the single-path QUIC design. The concerned
frames (and their extended meaning) are the following.

NEW_CONNECTION_ID:

 : Equivalent to a MP_NEW_CONNECTION_ID frame with Uniflow ID set to 0.

RETIRE_CONNECTION_ID:

 : Equivalent to a MP_RETIRE_CONNECTION_ID frame with Uniflow ID set to 0.

ACK:

 : Equivalent to a MP_ACK frame with Uniflow ID set to 0.



Security Considerations
=======================

Nonce Computation {#nonce-considerations}
-----------------

With Multipath QUIC, each uniflow has its own packet number space. With the
current nonce computation {{I-D.ietf-quic-tls}}, using twice the same packet
number over two different uniflows on the same direction leads to the same
cryptographic nonce. Using twice the same nonce MUST NOT happen, hence
MP-QUIC has a different nonce computation than {{I-D.ietf-quic-tls}}

the left most bits of nonce MUST be the Uniflow ID that identifies the
current uniflow up to max_sending_uniflow_id. The remaining bits of the
nonce is formed by an exclusive OR of the least signicant bits of the packet
protection IV with the padded packet number (left-padded with 0s). The
nonce MUST be left-padded with a 0 if max_sending_uniflow_id <= 2, and
the max_sending_uniflow_id MUST NOT be higher than 2^61. If a uniflow has
sent 2^62-max_sending_uniflow_id packets, another uniflow MUST be used
to avoid re-using the same nonce.

Validation of Exchanged Addresses {#validation_address}
---------------------------------

To use addresses communicated by the peer through ADD_ADDRESS frames, hosts are
required to validate them before using uniflows to these addresses. The main
reason for this validation is that the remote host might have sent, purposely or
not, a packet with a source IP that does not correspond to the IP of the remote
interface. This could lead to amplification attacks where the client start using
a new path with a source IP corresponding to the victim's one. Without
validation, the server might then flood the victim. Similarly for ADD_ADDRESS
frames, a malicious server might advertise the IP address of a victim, hoping
that the client will use it without validating it before.


IANA Considerations
===================

QUIC Transport Parameter Registry
---------------------------------

This document defines a new transport parameter for the negotiation of
multiple paths. The following entry in {{iana-tp-table}} should be added to
the "QUIC Transport Parameters" registry under the "QUIC Protocol" heading.

| Value  | Parameter Name         | Specification     |
|:-------|:-----------------------|:------------------|
| 0x0020 | max_sending_uniflow_id | {{tp-definition}} |
{: #iana-tp-table title="Addition to QUIC Transport Parameters Entries"}


Acknowledgements
================

We would like to thanks Masahiro Kozuka and Kazuho Oku for their numerous
comments on the first version of this draft. We also thanks Philipp Tiesel for
his early comments that led to the current design, and Ian Swett for later
feedbacks. We also want to thank Christian Huitema for his draft about
multipath requirements to identify critical elements about the multipath
feature. Mohamed Boucadair provided lot of useful inputs on the second version
of this document. Maxime Piraux helped us to improve the third version of this
draft.


--- back

Change Log
==========

Since draft-deconinck-quic-multipath-03
---------------------------------------

- Clarify the notion of asymmetric paths by introducing uniflows
- Remove the PATH_UPDATE frame
- Rename PATHS frame to UNIFLOWS frame and adapts its content
- Add a sequence number to frames involving Address ID events (#4)
- Disallow Zero-length connection ID (#2)

Since draft-deconinck-quic-multipath-02
---------------------------------------

- Consider asymmetric paths

Since draft-deconinck-quic-multipath-01
---------------------------------------

- Include path policies considerations
- Add practical considerations thanks to Mohamed Boucadair inputs
- Adapt the RETIRE_CONNECTION_ID frame
- Updated text to match draft-ietf-quic-transport-18

Since draft-deconinck-quic-multipath-00
---------------------------------------

- Comply with asymmetric Connection IDs
- Add max_paths transport parameter to negotiate initial number of active paths
- Path ID as a regular varint
- Remove max_path_id transport parameter
- Updated text to match draft-ietf-quic-transport-14

Since draft-deconinck-multipath-quic-00
---------------------------------------

- Added PATH_UPDATE frame
- Added MAX_PATHS frame
- No more packet header change
- Implicit Path ID notification using Connection ID and NEW_CONNECTION_ID
  frames
- Variable-length encoding for Path ID
- Updated text to match draft-ietf-quic-transport-10
- Fixed various typos
