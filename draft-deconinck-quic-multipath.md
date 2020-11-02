---
title: Multipath Extensions for QUIC (MP-QUIC)
abbrev: MP-QUIC
docname: draft-deconinck-quic-multipath-05
date: {DATE}
category: std
consensus: true
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
  email: olivier.bonaventure@uclouvain.be

normative:
  RFC2119:
  RFC8174:
  I-D.ietf-quic-transport:
  I-D.ietf-quic-tls:
  I-D.ietf-quic-recovery:
  I-D.ietf-quic-invariants:
informative:
  I-D.huitema-quic-mpath-req:
  I-D.huitema-quic-1wd:
  I-D.bonaventure-iccrg-schedulers:
  RFC0793:
  RFC6356:
  RFC6824:
  RFC7050:
  RFC7225:
  RFC8041:
  RFC3077:
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

This document specifies extensions to the QUIC protocol to enable the
simultaneous usage of multiple paths for a single connection.

These extensions are compliant with the single-path QUIC design and
preserve QUIC privacy features.


--- middle

Introduction
============

Today's endhosts are equipped with several
network interfaces. Users expect to be able to seamlessly switch
from one interface to another one or use them simultaneously to aggregate
bandwidth whenever needed. During the last years, several multipath extensions
to transport protocols have been proposed (e.g., {{RFC6824}}, {{MPRTP}}, or
{{SCTPCMT}}). Multipath TCP {{RFC6824}} is the most mature one. It is already
deployed on popular smartphones, but also for other use cases {{RFC8041}}
{{IETFJ}}.

With regular TCP and UDP, all the packets that belong to a given flow
share the same 4-tuple {source IP address, source port number, destination IP
address, destination port number} that acts as an identifier for this flow. This
prevents these flows from using multiple paths.

QUIC
{{I-D.ietf-quic-transport}} does not use the 4-tuple as an implicit
connection identifier. Instead, a QUIC flow is identified by a Connection ID.
This enables QUIC to cope with events affecting the 4-tuple, such as NAT
rebinding or IP address changes. The QUIC connection migration feature,
specified in Section 9 of {{I-D.ietf-quic-transport}}, enables migrating a flow
from one 4-tuple to another one, sustaining in particular a connection over
different network paths.

Still, there is a void to specify simultaneous usage of QUIC over available
network paths for a single connection. Use cases such as bandwidth aggregation
or seamless network handovers would be applicable to QUIC, as they are now with
Multipath TCP {{RFC8041}}{{IETFJ}}. Experience with Multipath TCP on smartphones
shows that the ability to simultaneously use WLAN and cellular during handovers
improves the user perceived quality of experience.
A performance evaluation of an early solution for such use cases and a
comparison between Multipath QUIC and Multipath TCP may be found in {{MPQUIC}}.

In this document, we leverage many of the lessons learned from the
design of Multipath TCP and the comments received on the first versions
of this document to propose extensions to the current QUIC design to
enable it to simultaneously use several network paths. This document focuses
mainly on network paths that are distinguishable by the endpoints.

This document is organized as follows. It first provides in {{overview}} an
overview of the operation of Multipath QUIC. It then states the required changes
in the current QUIC design {{I-D.ietf-quic-transport}} and specifies in {{spec}}
the usage of multiple paths. Finally, it discusses some security considerations.


Conventions and Definitions
===========================

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in BCP 14
{{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as
shown here.

We assume that the reader is familiar with the terminology used in
{{I-D.ietf-quic-transport}}. In addition, we define the following terms:

- Uniflow: A unidirectional flow of packets between a QUIC host and its peer.
  This flow is identified by an internal identifier, called Uniflow ID. Packets
  sent over a uniflow use a Destination Connection ID that may change during the
  lifetime of the connection. When being in use, an uniflow is temporarily bound
  to a 4-tuple (Source IP Address, Source Port Number, Destination IP Address,
  Destination Port Number).

- Initial Uniflows: The two uniflows used by peers for the establishment of a
  QUIC connection. One is the uniflow from the client to the server and the
  other is the uniflow from the server to the client. The cryptographic
  handshake is done on these uniflows. These are identified by Uniflow ID 0.


Notation Conventions
----------------------

Packet and frame diagrams use the format described in Section 12 of
{{I-D.ietf-quic-transport}}.


Overview {#overview}
========

The design of QUIC {{I-D.ietf-quic-transport}} provides reliable
transport with multiplexing, confidentiality, integrity, and authenticity of
data flows. A wide range of devices on today's Internet are multihomed.
Examples include smartphones equipped with both WLAN and cellular
interfaces, but also regular dual-stack hosts that use both IPv4 and
IPv6.

The current design of QUIC does not enable multihomed devices to efficiently
use different paths simultaneously. This document proposes multipath extensions
with the following design goals:

* Each host keeps control on the number of uniflows being used over the
  connection.

* The simultaneous usage of multiple uniflows should not introduce new privacy
  concerns.

* A host must ensure that all the paths it uses actually reach its peer to
  avoid packet flooding towards a victim (see Section 21.12.3 of
  {{I-D.ietf-quic-transport}})

* The multipath extensions should handle the asymmetrical nature of paths
  between two peers.

We first explain why a multipath extension would be beneficial to QUIC
and then describe it at a high level.


Moving from Bidirectional Paths to Uniflows
-------------------------------------------

To understand the overall architecture of the multipath extensions, let us
first refine the notion of "path". A path may be denoted by a 4-tuple (Source IP
Address, Source Port Number, Destination IP Address, Destination Port Number).
In QUIC, this is namely a UDP path from the local host to the remote one.
Considering a smartphone interacting with a single-homed server, the smartphone
might want to use one path over the WLAN network and another over the cellular
one. Those paths are not necessarily disjoint. For example, when interacting
with a dual-stack server, a smartphone may create two paths over WLAN: one over
IPv4 and the other one over IPv6.

A regular QUIC connection is composed of two independent active packet flows.
The first flow gathers the packets from the client to the server and the other
the packets from the server to the client. To illustrate this, let us consider
the example depicted in {{examplequic}}. In this example, the client has two IP
addresses: IPc1 and IPc2. The server has one single address: IPs1.

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
The server then replies to the client if it accepts the connection. If the
handshake succeeds, the connection is established. Still, this "path" actually
consists in two independent UDP flows. Each host has its own view of (i) the
4-tuple used to send packets and (ii) the 4-tuple on which it receives packets.
While the 4-tuple used by the client to send packets may be the same as the one
seen and used by the server, this is not always the case since middleboxes
(e.g., NATs) may alter the 4-tuple of packets.

To further emphasize on this flow asymmetry,
QUIC embeds a path validation mechanism {{I-D.ietf-quic-transport}} assessing
whether a host can reach its peer through a given 4-tuple. This process is
unidirectional, i.e., the sender checks that it can reach the receiver, but not
the reverse. A host receiving a PATH_CHALLENGE frame on a new 4-tuple may in
turn initiate a path validation, but this is up to the peer.

A QUIC connection is a collection of unidirectional flows (called, uniflows). A
plain QUIC connection is composed of a main uniflow from client to server and
another main uniflow from server to client. These uniflows have their own
Connection IDs. They are host-specific, i.e., the uniflow(s) from A to B are
different from the ones from B to A. This potentially enables the use of
unidirectional links such as non-broadcast satellite links {{RFC3077}}, which
cannot be used with TCP.


Beyond Connection Migration
---------------------------

Unlike TCP {{RFC0793}}, QUIC is not bound to a particular 4-tuple during the
lifetime of a connection. A QUIC connection is identified by a (set of)
Connection ID(s), placed in the public header of each QUIC packet. This
enables hosts to continue the connection even if the 4-tuple changes due to,
e.g., NAT rebinding. This ability to shift a connection from one 4-tuple to
another is called: Connection Migration. One of its use cases is fail-over when
the IP address in use fails but another one is available. A smartphone losing
the WLAN connectivity can then continue the connection over its cellular
interface, for instance.

A QUIC connection can thus start on a given set of uniflows, denoted as the
initial uniflows, and end on another ones. However, the current QUIC design
{{I-D.ietf-quic-transport}} assumes that only one pair of uniflows is in use for
a given connection. The current transport specification
{{I-D.ietf-quic-transport}} does not provide means to distinguish path migration
from simultaneous usage of available uniflows for a given connection.

This document fills that void. It first proposes mechanisms to
communicate endhost addresses to the peer. It then leverages the Address
Validation procedure with PATH_CHALLENGE and PATH_RESPONSE frames described
in Section 8 of {{I-D.ietf-quic-transport}} to verify whether the additional
addresses advertised by the host are reachable. In this case, those addresses
can be used to initiate new uniflows to spread packets over several network
paths following a packet scheduling policy that is out of scope of this
document.

The example of {{examplempquic}} illustrates a data exchange between a
dual-homed client sending a request spanning two packets and a single-homed
server. Uniflow IDs are independently chosen by each host. In the presented
example, the client sends packets over WLAN on Uniflow 0 and over LTE on Uniflow
1, while the packets sent by the server over WLAN are on Uniflow 2 and
those over LTE are on Uniflow 1.

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
  | Pkt(DCID=E,PN=3,frames=[    | Pkt(DCID=D,PN=2,frames=[    |
  |  STREAM("Response 3")])     |  STREAM("Response 4")])     |
  |---------------------------->|                         ----|
  |                             |                   ------|   |
  |            ...              |    ...  <---------|         |
~~~~~~~~~~
{: #examplempquic title="Data flow with Multipath QUIC"}


The remaining of this section presents a high-level overview of the
multipath operations in QUIC.


Establishment of a Multipath QUIC Connection
--------------------------------------------

A Multipath QUIC connection starts as a regular QUIC connection
{{I-D.ietf-quic-transport}} {{I-D.ietf-quic-tls}}. The multipath extensions
defined in this document are negotiated using the `max_sending_uniflow_id`
transport parameter. Any value for this transport parameter advertises the
support of the multipath extensions.

Notice that a host advertising a value of 0 for the `max_sending_uniflow_id`
transport parameter indicates that it does not want additional uniflows to send
packets, but it still supports the multipath extensions. Such situation might be
useful when the host does not require multiple uniflows for packet
sending but still wants to let the peer use multiple uniflows to reach it.


Architecture of Multipath QUIC
------------------------------

To illustrate the architecture of a Multipath QUIC connection, consider
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
|        | <===================================== |        |
+--------+                                        +--------+
~~~~~~~~~~
{: #uniflowsexample title="An Example of Uniflow Distribution over a Multipath QUIC Connection"}

Once established, a Multipath QUIC connection consists in one or more uniflows
from the client to the server and one or more uniflows from the server to the
client. The number of uniflows in one direction can be different from the one in
the other direction. The example in {{uniflowsexample}} shows two
uniflows from the client to the server and three uniflows from the server to the
client. From the end-hosts' viewpoint, they observe two kinds of uniflows:

* Sending uniflows: uniflows over which the host can send packets

* Receiving uniflows: uniflows over which the host can receive packets

Reconsidering the example in {{uniflowsexample}}, the client has two
sending uniflows and three receiving uniflows. The server has three sending
uniflows and two receiving uniflows. There is thus a one-to-one mapping between
the sending uniflows of a host and the receiving uniflows of its peer. A
uniflow is seen as a sending uniflow from the sender's perspective and as a
receiving uniflow from the receiver's viewpoint.

Each uniflow is associated with a specific four-tuple and identified by a
Uniflow ID, as shown in {{architectural}}.

~~~~~~~~~~
Client state
    +-----------------------------------------------------+
    |                      Connection                     |
    |   +-----------+ +-----------+ ... +-------------+   |
    |   |  Sending  | |  Sending  | ... |   Sending   |   |
    |   | Uniflow 0 | | Uniflow 1 |     | Uniflow N-1 |   |
    |   | UCID set A| | UCID set B| ... |  UCID set C |   |
    |   |  Tuple A  | |  Tuple B  | ... |   Tuple C   |   |
    |   |   PNS A   | |   PNS B   |     |    PNS C    |   |
    |   +-----------+ +-----------+ ... +-------------+   |
    |   +-----------+ +-----------+ ... +-------------+   |
    |   | Receiving | | Receiving | ... |  Receiving  |   |
    |   | Uniflow 0 | | Uniflow 1 |     | Uniflow M-1 |   |
    |   | UCID set X| | UCID set Y| ... |  UCID set Z |   |
    |   |  Tuple X  | |  Tuple Y  | ... |   Tuple Z   |   |
    |   |   PNS X   | |   PNS Y   |     |    PNS Z    |   |
    |   +-----------+ +-----------+ ... +-------------+   |
    +-----------------------------------------------------+

Server state
    +-----------------------------------------------------+
    |                      Connection                     |
    |   +-----------+ +-----------+ ... +-------------+   |
    |   |  Sending  | |  Sending  | ... |   Sending   |   |
    |   | Uniflow 0 | | Uniflow 1 |     | Uniflow M-1 |   |
    |   | UCID set X| | UCID set Y| ... |  UCID set Z |   |
    |   |  Tuple X* | |  Tuple Y* | ... |   Tuple Z*  |   |
    |   |   PNS X   | |   PNS Y   |     |    PNS Z    |   |
    |   +-----------+ +-----------+ ... +-------------+   |
    |   +-----------+ +-----------+ ... +-------------+   |
    |   | Receiving | | Receiving | ... |  Receiving  |   |
    |   | Uniflow 0 | | Uniflow 1 |     | Uniflow N-1 |   |
    |   | UCID set A| | UCID set B| ... |  UCID set C |   |
    |   |  Tuple A* | |  Tuple B* | ... |   Tuple C*  |   |
    |   |   PNS A   | |   PNS B   |     |    PNS C    |   |
    |   +-----------+ +-----------+ ... +-------------+   |
    +-----------------------------------------------------+
~~~~~~~~~~
{: #architectural title="Architectural view of Multipath QUIC for a host having N sending uniflows and M receiving uniflows"}

A Multipath QUIC connection starts using two Initial
Uniflows, identified by Uniflow ID 0 on each peer. The packets can then be
spread over several uniflows. Each uniflow has its (set of) Uniflow
Connection ID(s) (UCID) packets that are used to explicitly mark where they
belong to. Depending on the direction of the uniflow, the host keeps either the
Uniflow Source Connection ID (USCID, for the receiving uniflows) or the Uniflow
Destination Connection ID (USCID, for the sending uniflows). Notice that the
(set of) UDCID(s) of a sending uniflow of a host is the same as the (set of)
USCID(s) of the corresponding receive uniflow of the remote peer.

Preventing the
linkability of different uniflows is an important requirement for the multipath
extensions {{I-D.huitema-quic-mpath-req}}. We address it by using UCIDs as
implicit uniflow identifiers. This makes the linkability harder than having
explicit signaling as in earlier version of this draft. Furthermore, it does not
require any public header change and thus preserves the QUIC invariants
{{I-D.ietf-quic-invariants}}.

When a uniflow is in use, each endhost associates it with a network path. In
practice, this consists in a particular 4-tuple over which packets are sent
(or received) on a sending (or receiving) uniflow. Each endhost has a
specific vision of the 4-tuple, which might differ between endhosts. For
instance, a client located behind a NAT sends data from a private IP address and
the server will receive packets coming from the NAT's public IP address. Notice
that while uniflows may share a common network path, this is not mandatory.

Each uniflow is an independent flow of packets over a given network path.
Uniflows can experience very different network conditions (latency,
bandwidth, ...). To handle this, each uniflow has its own packet sequence number
space.

In addition to the UCIDs, 4-tuple and packet number space, some additional
information is maintained for each uniflow. The Uniflow ID identifies the
uniflow at the frame level and ensures uniqueness of the nonce (see
{{nonce-considerations}} for details) while limiting the number of concurrently
used uniflows.


Uniflow Establishment
---------------------

The `max_sending_uniflow_id` transport parameter exchanged during the
cryptographic handshake fixes an upper bound on the number of sending uniflows a
host wants to support. Then, hosts provide to their peer Uniflow Connection IDs
to use on uniflows. Both hosts dynamically control how many sending uniflows can
currently be in use by the peer, i.e., the number of different Uniflow IDs
proposed to the peer. While the sender determines the upper bound of sending
paths it can have, it is the receiver that initializes uniflows, as the sender
needs a UCID communicated by the receiver before using a uniflow.

Notice that
the peers might advertise different values for the `max_sending_uniflow_id`
transport parameters, setting different upper bounds to the
sending and receiving uniflows of each host.

Hosts initiate the creation of their receiving uniflows by sending
MP_NEW_CONNECTION_ID frames (see {{mpnewconnectionidsec}}) which are an extended
version of the NEW_CONNECTION_ID frame. This frame associates a UCID to a
uniflow. Upon reception of the MP_NEW_CONNECTION_ID frame, a host can start
using the proposed sending uniflow having the referenced Uniflow ID by marking
sent packets with the provided UCID. Therefore, once a host sends a
MP_NEW_CONNECTION_ID frame, it announces that it is ready to receive packets
from that Uniflow ID with the proposed UCID. As frames are encrypted, adding new
uniflows over a QUIC connection does not leak cleartext identifiers
{{I-D.huitema-quic-mpath-req}}.

A server might provide several Uniflow Connection IDs for the same Uniflow ID
with multiple MP_NEW_CONNECTION_ID frames. This can be useful to cope with
migration cases, as described in {{path-migration}}. Multipath QUIC is
thus asymmetrical.


Exchanging Data over Multiple Uniflows
--------------------------------------

A QUIC packet acts as a container for one or more frames. Multipath QUIC uses
the same STREAM frames as QUIC to carry data. A byte offset is associated with
the data payload. One of the key design of (Multipath) QUIC is that frames
are independent of the packets carrying them. This implies that a frame
transmitted over one uniflow could be retransmitted later on another uniflow
without any change. Furthermore, all current QUIC frames are idempotent and
could be optimistically duplicated over several uniflows.

The uniflow on which data is sent is a packet-level information. This means that
a frame can be sent regardless of the uniflow of the packet carrying it. Other
flow control considerations like the stream receive window advertised by the
MAX_STREAM_DATA frame remain unchanged when there are multiple sending uniflows.

As previously described, Multipath QUIC might face reordering at packet-level
when using uniflows having different latencies. The presence of different
Uniflow Connection IDs ensures that the packets sent over a given uniflow will
contain monotonically increasing packet numbers. To ensure more flexibility and
potentially to reduce the ACK block section of the (MP_)ACK frame when
aggregating bandwidth of uniflows exhibiting different network characteristics,
each uniflow keeps its own monotonically increasing Packet Number space. This
potentially allows sending up to 2 * 2^64 packets on a QUIC connection since
each uniflow has its own packet number space (see {{nonce-considerations}} for
the detail of this limit).

With the introduction of multiple uniflows, there is a need to acknowledge
packets sent on different uniflows separately. The packets sent on Initial
Uniflows (with Uniflow ID 0) are still acknowledged with regular ACK frames,
such that no modification is introduced in a core frame. For the other
uniflows, the multipath extensions introduce a MP_ACK frame which prefixes the
ACK frame with a Uniflow ID field indicating from which receiving uniflow the
host acknowledges packets. To better explain this, let us consider the situation
illustrated in {{ack-uniflows}}.

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
{: #ack-uniflows title="Acknowledging Packets Sent on Uniflows"}

Here five uniflows are in use, two in the client to server direction and three
in the reverse one. The client first sends a packet on its sending Uniflow 1
(linked to CID B). The server receives the packet on its receiving Uniflow 1.
Therefore, it generates a MP_ACK frame for Uniflow ID 1 and transmits it to the
client. The server can choose any of its sending uniflows to transmit this
frame. In the provided situation, the server sends its packets on Uniflow 2. The
client thus receives this packet on its receiving Uniflow 2.

Similarly, packets sent over a given uniflow might be acknowledged by (MP_)ACK
frames sent on another uniflow that does not share the same network path.
Looking at {{examplempquic}} again, "Response 2" packet on server's sending
uniflow 1 with DCID D using the LTE network is acknowledged by a MP_ACK frame
received on a uniflow using the WLAN network.


Exchanging Addresses
--------------------

When a multi-homed device connects to a dual-stacked server using its
IPv4 address, it is aware of its local addresses (e.g., the WLAN and the
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
starts using it as described in Section 8 of {{I-D.ietf-quic-transport}}. This
process ensures that the advertised address actually belongs to the peer and
that the peer can receive packets sent by the host on the provided address. It
also prevents hosts from launching amplification attacks to a victim address.

If the client is behind a NAT, it could announce a private address in an
ADD_ADDRESS frame. In such situations, the server would not be able to validate
the communicated address. The client might still use its NATed addresses to
start using its sending uniflows. To enable the server to make the link between
the private and the public addresses and hence conciliate the different 4-tuple
views, Multipath QUIC provides the UNIFLOWS frame that lists the current active
sending Uniflow IDs along with their associated local Address ID. Notice that a
host might also discover the public addresses of its peer by observing its
remote IP addresses associated to the connection.

A receiving uniflow is active as soon as the host has sent the
MP_NEW_CONNECTION_ID frames proposing the corresponding Uniflow Connection IDs
to its peer. A sending uniflow is active when it has received its Uniflow
Connection IDs and is bound to a validated 4-tuple. The UNIFLOWS frame indicates
the local Address IDs that the uniflow uses from the sender's perspective. With
this information, the remote host can validate the public address and associate
the advertised one with the perceived addresses.


Coping with Address Removals
----------------------------

During the lifetime of a QUIC connection, a host might lose some of its
addresses. A concrete example is a smartphone going out of reach of a
WLAN network or shutting off one of its network interfaces. Such address
removals are advertised using REMOVE_ADDRESS frames. The REMOVE_ADDRESS frame
contains the Address ID of the lost address previously communicated through
ADD_ADDRESS. Notice that because a given Address ID might encounter several
events that need to be ordered (e.g., ADD_ADDRESS, REMOVE_ADDRESS and
ADD_ADDRESS again), both ADD_ADDRESS and REMOVE_ADDRESS frames include an
Address ID related Sequence Number.


Uniflow Migration {#path-migration}
-----------------

At a given time, a Multipath QUIC endpoint gathers a set of active sending and
receiving uniflows, each associated to a 4-tuple. This association is mutable.
Hosts can change the 4-tuple used by their sending uniflows at any time,
enabling QUIC to migrate uniflows from one network path to another. Yet, to
address privacy issues due to the linkability of addresses, hosts should avoid
reusing the same Connection ID used by a sending uniflow when the 4-tuple
changes, as described in Section 9.5 of {{I-D.ietf-quic-transport}}.


Handling Multiple Network Paths
-------------------------------

The simultaneous usage of several sending uniflows introduces new algorithms
(packet scheduling, path management) whose specifications are out of scope of
this document. Nevertheless, these algorithms are actually present in any
multipath-enabled transport protocol like Multipath TCP, CMT-SMTP and
Multipath DCCP. A companion draft {{I-D.bonaventure-iccrg-schedulers}} provides
several general-purpose packet schedulers depending on the application goals. A
similar document can be created to discuss path/uniflow management
considerations.


Congestion Control
------------------

The QUIC congestion control scheme is defined in {{I-D.ietf-quic-recovery}}.
This congestion control scheme is not suitable when several sending uniflows are
active. Using the congestion control scheme defined in
{{I-D.ietf-quic-recovery}} with Multipath QUIC would result in unfairness. Each
sending uniflow of a Multipath QUIC connection MUST have its own congestion
control state. As for Multipath TCP, the windows of the different sending
uniflows MUST be coupled together {{RFC6356}}.



Mapping Uniflow IDs to Connection IDs
=====================================

As described in the overview section, hosts need to identify on which uniflows
packets are sent. The Uniflow ID must then be inferred from the public header.
This is done by using the Destination Connection ID field of Short Header
packets.

The Initial Uniflow Connection IDs are determined during the cryptographic
handshake and actually correspond to both Connection IDs in the current
single-path QUIC design {{I-D.ietf-quic-transport}}. Additional Uniflow
Connection IDs for the Initial Uniflows can be provided with the regular
NEW_CONNECTION_ID frames. The Uniflow Connection IDs of the other uniflows
are determined when the MP_NEW_CONNECTION_ID frames are exchanged.

Hosts MUST accept packets coming from their peer using the UCIDs they proposed
in the (MP_)NEW_CONNECTION_ID frames they sent and associate them with the
corresponding receiving Uniflow ID. Upon reception of a
(MP_)NEW_CONNECTION_ID frame, hosts MUST acknowledge it and MUST store the
advertised Uniflow Destination Connection ID and the Uniflow ID of the proposed
sending uniflow.

Hosts MUST ensure that all advertised Uniflow Connection IDs are available
for the whole connection lifetime, unless they have been retired by their peer
in the meantime by the reception of a (MP_)RETIRE_CONNECTION_ID.

A host MUST NOT send MP_NEW_CONNECTION_ID frames with a Uniflow ID greater than
the value of `max_sending_uniflow_id` advertised by its peer.


Using Multiple Uniflows {#spec}
=======================

This section describes in details the Multipath QUIC operations.


Multipath Negotiation
---------------------

The Multipath negotiation takes place during the cryptographic handshake with
the `max_sending_uniflow_id` transport parameter. A QUIC connection is initially
single-path in QUIC. During this handshake, hosts advertise their support for
multipath operations. When a host advertises a value for the
`max_sending_uniflow_id` transport parameter, it indicates that it supports the
multipath extensions, i.e., the extensions defined in this document (not to be
mixed with the availability of local multiple network paths). If any host does
not advertise the `max_sending_uniflow_id` transport parameter, multipath
extensions are disabled.

The usage of multiple uniflows relies on the ability to use several Connection
IDs over a same QUIC connection. Therefore, zero-length Connection IDs MUST NOT
be used if the peer advertises a value different from 0 for the
`max_sending_uniflow_id` transport parameter.


### Transport Parameter Definition {#tp-definition}

A host MAY use the following transport parameter:

max_sending_uniflow_id (0x40):

 :  Indicates the support of the multipath extension presented in this document,
    regardless of the carried value. Its integer value puts an upper bound on
    the number of sending uniflows the host advertising the value is ready to
    support. If absent, this means that the host does not agree to use the
    multipath extension over the connection.


Coping with Additional Remote Addresses
---------------------------------------

Hosts can learn remote addresses either by receiving ADD_ADDRESS frames or
observing the 4-tuple of incoming packets. Hosts MUST first validate the newly
learned remote IP addresses before starting sending packets to those addresses.
This requirement is explained in {{validation_address}}. Hosts MUST initiate
Address Validation Procedure as specified in {{I-D.ietf-quic-transport}}.

A host MAY cache a validated address for a limited amount of time.


Receiving Uniflow State
-----------------------

When proposing uniflows to their peer, hosts need to maintain some state for
their receiving uniflows. This state is created upon the sending of a first
MP_NEW_CONNECTION_ID frame proposing the corresponding Uniflow ID. As long as
there is still one active Uniflow Connection ID for this receiving uniflow
(i.e., one UCID which was not retired yet using a MP_RETIRE_CONNECTION_ID), the
host MUST accept packets over the receiving uniflow. Once created, hosts MUST
keep the following receiving uniflow information:

Uniflow ID:

 : An integer that uniquely identifies the receiving uniflow in the connection.
   This value is immutable.

Uniflow Connection IDs:

 : Possible values for the Connection ID field of packets belonging to this
   receiving uniflow. This value contains the sequence of active UCIDs that
   were advertised in previously sent MP_NEW_CONNECTION_ID frames. Notice that
   this sequence might be empty, e.g., when all advertised UCIDs have been
   retired by the peer.

Packet Number Space:

 : Packet number space dedicated to this receiving uniflow. Packet number
   considerations described in Section 12.3 of {{I-D.ietf-quic-transport}} apply
   within a given receiving uniflow.

Associated 4-tuple:

 : The tuple (sIP, dIP, sport, dport) currently observed to receive packets over
   this uniflow. This value is mutable, because a host might receive a packet
   with a different (possibly) validated remote address and/or port than the one
   previously recorded. If a host observes a change in the 4-tuple of the
   receiving uniflow, it follows the considerations of Section 9.5 of
   {{I-D.ietf-quic-transport}}.

Associated local Address ID:

 : The Address ID advertised in ADD_ADDRESS frames sent by the peer
   corresponding to the local address used to receive packets. This helps to
   generate UNIFLOWS frames advertising the mapping between uniflows and
   addresses. The addresses on which the connection was established have Address
   ID 0.


Hosts can also collect network measurements on a per-uniflow basis, like the
number of packets received.


Sending Uniflow State
---------------------

During a Multipath QUIC connection, hosts maintain some state for sending
uniflows. The state of the sending uniflow determines information that hosts
are required to store. The possible sending uniflow states are depicted in
{{snd_uniflow_state}}.

TODO: intermediate state for address validation? Yes

~~~~~~~~~~
      o
      |
      | receive a first MP_NEW_CONNECTION_ID
      |    with the associated Uniflow ID
      |
      v       path usage over a validated 4-tuple
+----------+ ------------------------------------> +----------+
|  UNUSED  |                                       |  ACTIVE  |
+----------+ <------------------------------------ +----------+
                 address change or retired UCID
~~~~~~~~~~
{: #snd_uniflow_state title="Finite-State Machine describing the possible states of a sending uniflow"}

Once a sending uniflow has been proposed by the peer in a received
MP_NEW_CONNECTION_ID frame, it is in the UNUSED state. In this situation, hosts
MUST keep the following sending uniflow information:

Uniflow ID:

 : An integer that uniquely identifies the sending uniflow in the connection.
   This value is immutable.

Uniflow Connection IDs:

 : Possible values for the Connection ID field of packets belonging to this
   sending uniflow. This value contains the sequence of active UCIDs that were
   advertised in previously received MP_NEW_CONNECTION_ID frames. Notice that
   this sequence might be empty, e.g., when all advertised UCIDs have been
   retired.

Sending Uniflow State:

 : The current state of the sending uniflow, being one of the values presented
   in {{snd_uniflow_state}}.

Packet Number Space:

 : Packet number space dedicated to this sending uniflow. Packet number
   considerations described in Section 12.3 of {{I-D.ietf-quic-transport}} apply
   within a given sending uniflow.


When the host wants to start using the sending uniflow over a validated address,
the sending uniflow goes to the ACTIVE state. This is the state where a sending
uniflow can be used to send packets. Having an uniflow in ACTIVE state only
guarantees that it can be used, but the host is not forced to. In addition to
the fields required in the UNUSED state, the following elements MUST be tracked:


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

Congestion controller:

 : A congestion window limiting the transmission rate of the sending uniflow.

Performance metrics:

 : Basic statistics such as one-way delay or the number of packets sent. This
   information can be useful when a host needs to perform packet scheduling
   decisions and flow control.


It might happen that a sending path is temporarily unavailable, because one of
the endpoint's addresses is no more available or because the host retired all
the UCIDs of the sending uniflow. In such cases, the path goes back to the
UNUSED state. When performing a transition back to the UNUSED state, hosts MUST
reset the additional state added by the ACTIVE state. In the UNUSED state, the
host MUST NOT send non-probing packets on it. At this state, the host might want
to restart using the uniflow over another validated 4-tuple, switching the
uniflow state back to the ACTIVE state. However, its congestion controller state
MUST be restarted and its performance metrics SHOULD be reset.


Losing Addresses
----------------

During the lifetime of a connection, a host might lose addresses, e.g., a
network interface that was shut down. All the ACTIVE sending uniflows that were
using that local address MUST stop sending packets from that address. To
advertise the loss of an address to the peer, the host MUST send a
REMOVE_ADDRESS frame indicating which local Address IDs has been lost. The host
MUST also send an UNIFLOWS frame indicating the status of the remaining ACTIVE
uniflows.

Upon reception of the REMOVE_ADDRESS, the receiving host MUST stop using the
ACTIVE sending uniflows affected by the address removal.

Hosts MAY reuse one of these sending uniflows by changing the assigned 4-tuple.
In this case, it MUST send an UNIFLOWS frame describing that change.


New Frames
==========

To support the multipath operations, new frames have been defined to coordinate
hosts. All frames defined in this document MUST be exchanged in 1-RTT packets.
A host receiving one of the following frames in other encryption context MUST
close the connection with a PROTOCOL_VIOLATION error.


MP_NEW_CONNECTION_ID Frame {#mpnewconnectionidsec}
--------------------------

The MP_NEW_CONNECTION_ID frame (type=0x40) is an extension of the
NEW_CONNECTION_ID frame defined by {{I-D.ietf-quic-transport}}. It provides the
peer with alternative Connection IDs and associates them to a particular uniflow
using the Uniflow ID.

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

Compared to the frame specified in {{I-D.ietf-quic-transport}}, an Uniflow ID
varint field of is prefixed to associate the Connection ID with a uniflow. This
frame can be sent by both hosts. Upon reception of the frame with a specified
Uniflow ID, the peer MUST update the related sending uniflow state and store the
communicated Connection ID.

To limit the delay of the multipath usage upon handshake completion, hosts
SHOULD send MP_NEW_CONNECTION_ID frames for receive uniflows they allow using as
soon the connection establishment completes.

The generation of Connection ID MUST follow the same considerations as presented
in Section 5.1 of {{I-D.ietf-quic-transport}}.


MP_RETIRE_CONNECTION_ID Frame
-----------------------------

The MP_RETIRE_CONNECTION_ID frame (type=0x41) is an extension of the
RETIRE_CONNECTION_ID frame defined by {{I-D.ietf-quic-transport}}. It indicates
that the end-host will no longer use a Connection ID related to a given uniflow
that was issued by its peer.

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

The frame is handled as described in {{I-D.ietf-quic-transport}} on an uniflow
basis.


MP_ACK Frame
------------

The MP_ACK frame (types 0x42 and 0x43) is an extension of the ACK frame defined
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

Compared to the ACK frame, the MP_ACK frame is prefixed by a varint
Uniflow ID field indicating to which uniflow the acknowledged packet sequence
numbers relate.


ADD_ADDRESS Frame
-----------------

The ADD_ADDRESS frame (type=0x44) is used by a host to advertise its currently
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
   address. When this field is present, it indicates that an uniflow can use the
   2-tuple (IP addr, port).


Upon reception of an ADD_ADDRESS frame, the receiver SHOULD store the
communicated address for future use.

The receiver MUST NOT send packets others than validation ones to the
communicated address without having validated it as specified in Section 8 of
{{I-D.ietf-quic-transport}}. ADD_ADDRESS frames SHOULD contain globally
reachable addresses. Link-local and possibly private addresses SHOULD NOT be
exchanged.


REMOVE_ADDRESS Frame
--------------------

The REMOVE_ADDRESS frame (type=0x45) is used by a host to signal that a
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

The UNIFLOWS frame (type=0x46) communicates the uniflows' state of the sending
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

The left most bits of nonce MUST be the Uniflow ID that identifies the
current uniflow up to max_sending_uniflow_id. The remaining bits of the
nonce is formed by an exclusive OR of the least significant bits of the
packet protection IV with the padded packet number (left-padded with 0s).
The nonce MUST be left-padded with a 0 if max_sending_uniflow_id <= 2, and
the max_sending_uniflow_id MUST NOT be higher than 2^61. If a uniflow has
sent 2^62-max_sending_uniflow_id packets, another uniflow MUST be used
to avoid re-using the same nonce.


Validation of Exchanged Addresses {#validation_address}
---------------------------------

To use addresses communicated by the peer through ADD_ADDRESS frames, hosts are
required to validate them before using uniflows to these addresses as described
in Section 8 of {{I-D.ietf-quic-transport}}. Section 21.12.3 of
{{I-D.ietf-quic-transport}} provides additional motivation for this process. In
addition, hosts MUST send ADD ADDRESS frames in 1-RTT frames to prevent off-path
attacks.


IANA Considerations
===================

QUIC Transport Parameter Registry
---------------------------------

This document defines a new transport parameter for the negotiation of
multiple paths. The following entry in {{iana-tp-table}} should be added to
the "QUIC Transport Parameters" registry under the "QUIC Protocol" heading.

| Value  | Parameter Name         | Specification     |
|:-------|:-----------------------|:------------------|
|  0x40  | max_sending_uniflow_id | {{tp-definition}} |
{: #iana-tp-table title="Addition to QUIC Transport Parameters Entries"}


Acknowledgments
================

We would like to thank Masahiro Kozuka and Kazuho Oku for their numerous
comments on the first version of this draft. We also thank Philipp Tiesel for
his early comments that led to the current design, and Ian Swett for later
feedback. We also want to thank Christian Huitema for his draft about
multipath requirements to identify critical elements about the multipath
feature. Mohamed Boucadair provided lot of useful inputs on the second version
of this document. Maxime Piraux and Florentin Rochet helped us to improve the
last versions of this draft. This project was partially supported by the MQUIC
project funded by the Walloon Government.


--- back

Comparison with Multipath TCP
=============================

Multipath TCP {{RFC6824}} is currently the most widely deployed multipath
transport protocol on the Internet. While its design impacted the initial
versions of the Multipath extensions for the QUIC protocol, there are now major
differences between both protocols that we now highlight.

Multipath TCP Bidirectional Paths vs. QUIC Uniflows
---------------------------------------------------

The notion of bidirectional paths, i.e., paths where packets flow in both
directions, became a de facto standard with TCP. The Multipath TCP extension
{{RFC6824}} combines several TCP connections to spread a single data stream
over them. Hence, all the paths of a Multipath TCP connection must be
bidirectional. However, networking experiences showed that packets following
a direction do not always share the exact same road as the packets in the
opposite direction. Furthermore, QUIC does not require a network path to be
bidirectional in order to be used.

Uniflow Establishment
---------------------

Unlike Multipath TCP {{RFC6824}}, both hosts dynamically control how many
sending uniflows can currently be in use by the peer.


Exchanging Data over Multiple Uniflows
--------------------------------------

The uniflow on which data is sent is a packet-level information. This means that
a frame can be sent regardless of the uniflow of the packet carrying it.
Furthermore, because the data offset is a frame-level information, there is no
need to define additional sequence numbers to cope with reordering across
uniflows, unlike Multipath TCP {{RFC6824}} that uses a Data Sequence Number at
the Multipath TCP level.

Decoupling the network paths of data with their acknowledgment can be useful
to limit the latency due to (MP_)ACK transmissions on high-latency network
paths and to enable the usage of unidirectional networks. Such scheduling
decision would not have been possible in Multipath TCP {{RFC6824}} which must
acknowledge data on the (bidirectional) path it was received on.


Congestion Control
------------------

Multipath TCP uses the LIA congestion control scheme specified in {{RFC6356}}.
This scheme can immediately be adapted to Multipath QUIC. Other coupled
congestion control schemes have been proposed for Multipath TCP such as
{{OLIA}}.


ACK Frame
---------

Since frames are independent of packets, and the uniflow notion relates to the
packets, the (MP_)ACK frames can be sent on any uniflow, unlike Multipath TCP
{{RFC6824}} which is constrained to send ACKs on the same path.


To move in companion drafts
===========================

Uniflow Establishment
---------------------

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
to a CPE), a Multipath QUIC capable endhost SHOULD advertise a
`max_sending_uniflow_id` value of at least 4 and SHOULD propose at least 4
receiving uniflows to its peer.


Exchanging Addresses
--------------------

Likewise, the client may be located behind a NAT64. As such it may announce an
IPv6 address in an ADD_ADDRESS frame, that will be received over IPv4 by an
IPv4-only server. The server should not discard that address, even if it is not
IPv6-capable.

An IPv6-only client may also receive from the server an ADD_ADDRESS frame which
may contain an IPv4 address. The client should rely on means, such as
{{RFC7050}} or {{RFC7225}}, to learn the IPv6 prefix to build an IPv4-converted
IPv6 address.

Hosts that are connected behind an address sharing mechanism may collect the
external IP address and port numbers assigned to the hosts and then use their
addresses in the ADD_ADDRESS. Means to gather such information include, but not
limited to, UPnP IGD, PCP, or STUN.


Uniflow Migration
-----------------

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
and an UNIFLOWS frame to inform that its Initial sending uniflow is now using
the cellular address. If the cellular address validation succeeds (which could
have been done as soon as the cellular address was advertised), the server can
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
multi-homed CPE), the server may inadvertently conclude that an uniflow is not
anymore valid leading thus to frequently sending PATH_UPDATE frames as a
function of the traffic distribution scheme enforced by the on-path device. To
prevent such behavior, the server SHOULD wait for at least X seconds to ensure
this is about a connection migration and not a side effect of an on-path
multi-interfaced device.


Scheduling Strategies
---------------------

The current QUIC design {{I-D.ietf-quic-transport}} offers a single scheduling
space, i.e., which frames will be packed inside a given packet. With the
simultaneous use of several sending uniflows, a second dimension is added,
i.e., the sending uniflow on which the packet will be sent. This dimension can
have a non negligible impact on the operations of Multipath QUIC, especially if
the available sending uniflows exhibit very different network characteristics.

The progression of the data flow depends on the reception of the MAX_DATA and
MAX_STREAM_DATA frames. Those frames SHOULD be duplicated on several or all
ACTIVE sending uniflows. This helps to limit the head-of-line blocking issue due
to the transmission of the frames over a slow or lossy network path.

The sending path on which (MP_)ACK frames are sent impacts the peer. The
(MP_)ACK frame is notably used to determine the latency of a combination of
uniflows. In particular, the peer can compute the round-trip-time of the
combination of its sending uniflow with its receive one. The peer would compute
the latency as the sum of the forward delay of the acknowledged uniflow and the
return delay of the uniflow used to send the (MP_)ACK frame. Choosing between
acknowledging packets symmetrically (on uniflow B to A if packet was sent on A
to B) or not is up to the implementation, if only possible. However, hosts
SHOULD keep a consistent acknowledgment strategy. Selecting a random uniflow to
acknowledge packets may affect the performance of the connection. Notice that
the inclusion of a timestamp field in the (MP_)ACK frame, as proposed by
{{I-D.huitema-quic-1wd}}, may help hosts estimate more precisely the one-way
delay of each uniflow, therefore leading to improved scheduling decisions.
Unlike MAX_DATA and MAX_STREAM_DATA, (MP_)ACK frames SHOULD NOT be
systematically duplicated on several sending uniflows as they can induce a large
network overhead.


Change Log
==========

Since draft-deconinck-quic-multipath-04
---------------------------------------

- Mostly editorial and reference fixes

Since draft-deconinck-quic-multipath-03
---------------------------------------

- Clarify the notion of asymmetric paths by introducing uniflows
- Remove the PATH_UPDATE frame
- Rename PATHS frame to UNIFLOWS frame and adapt its content
- Add a sequence number to frames involving Address ID events (#4)
- Disallow Zero-length connection ID (#2)
- Correctly handle nonce computation (thanks to Florentin Rochet)
- Focus on the core concepts of multipath and delegate algorithms to companion
  drafts
- Updated text to match draft-ietf-quic-transport-27

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
