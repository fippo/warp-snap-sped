---
title: "STUN Protocol for Embedding DTLS"
abbrev: "SPED"
category: info

docname: draft-hancke-webrtc-sped-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: AREA
workgroup: Network Working Group
keyword:
 - webrtc
 - stun
 - dtls
venue:
  group: WG
  type: Working Group
  mail: tsvwg@ietf.org
  arch: https://example.com/WG
  github: fippo/warp-snap-sped
  latest: https://fippo.github.io/warp-snap-sped/draft-hancke-webrtc-sped-latest.html

author:
 -
    fullname: Philipp Hancke
    organization: Meta Platforms Inc.
    email: philipp.hancke@googlemail.com

normative:

informative:

--- abstract

WebRTC uses the Interactive Connectivity Establishment (ICE) and Datagram Transport Layer Security (DTLS)
to establish secure connections. This document defines a protocol to embed the DTLS handshake into
the STUN packets used by ICE which allows parallelizing these sequential handshakes and reduces the number of round trips
it takes to establish a secure connection.

--- middle

# Introduction

The current WebRTC connection setup, as outlined in {{?RFC8829}}, incurs 4 RTTs before media can be sent and 6 RTTs before the data channel opens. The serialization of ICE and DTLS is a large contributor to that as illustrated below (for DTLS 1.2):

~~~
Client                                       Server
   |                                            |
   |------------- SDP Offer (actpass)---------->|
   |<-1---------- SDP Answer (passive)----------|
   |                                            |
   |<-2---------- ICE/Connectivity Checks ----->|
   |                                            |
   |------------- DTLS ClientHello ------------>|
   |<-3---------- DTLS ServerHello -------------|
   |------------- DTLS Finished --------------->|
   |<-4---------- DTLS Finished ----------------|
   |                                            |
   |------------- Application data ------------>|
~~~

In addition, deployment experience has shown reliability issues, caused by packet loss and the exponential backoff timer of DTLS implementation, particularly on connections with a high round-trip time. The underlying ICE layer, in contrast, uses periodic checks without exponential backoff.

The protocol defined in this specification aims to reduce this by embedding the DTLS handshake into STUN which eliminates the delay caused by the serialization of the protocols and enhances reliability with more frequent resends and acknowledgements.
The protocol is backward compatible with existing mechanisms for demultiplexing multiple ICE sessions on a single port, a common practice for ICE servers.
The protocol is designed for use with DTLS 1.2 {{?RFC6347}} and DTLS 1.3 {{?RFC9147}}. It can also accommodate post-quantum cryptography (PQC) which can significantly increase the size of DTLS handshake flights and number of packets.

The mechanism described can reduce the number of round-trips for session establishment, in some scenarios to as little as a single round-trip, which is comparable to the latency of the SDES key exchange mechanism {{?RFC4568}}.

# Terminology

**DTLS Flight**: A set of DTLS handshake messages sent by one peer as defined in {{Section 4 of ?RFC9147}}. A flight can consist of one or more messages, which may be split into multiple DTLS records.

**ICE Candidate**: A transport address, consisting of an IP address and port, that is a potential point of contact for communication with a peer.

**ICE candidate pair**: A pair consisting of a local and a remote ice candidate. Initially, it is unknown whether the pair is usable for sending data or not, but this is explored by the ICE agents.

**ICE Agent**: A peer acting according to the ICE protocol. This involves enumerating local ICE candidates and sending them to the peer, and systematically trying ICE Candidate pairs for usability.

**ICE Lite (Agent)**: A peer using a subset of the ICE procedure that does not send STUN binding requests but only replies to them. ICE Lite is typically used on servers.

# Embedding DTLS in STUN

## Example negotiation
The normal negotiation process serializes DTLS after STUN as shown below for a DTLS 1.3 handshake:

~~~
Client                                       Server
   |                                            |
   |------------- SDP Offer (actpass)---------->|
   |<-1---------- SDP Answer (passive)----------|
   |                                            |
   |<-2---------- ICE/Connectivity Checks ----->|
   |                                            |
   |------------- DTLS ClientHello ------------>|
   |<-3---------- DTLS ServerHello/Fin----------|
   |------------- DTLS Finished --------------->|
   |------------- Application data ------------>|
~~~

Embedding the DTLS handshake into the STUN Binding Requests reduces the setup time by one RTT:

~~~
Client                                       Server
   |                                            |
   |------------- SDP Offer (actpass) --------->|
   |<-1---------- SDP Answer (passive) -------->|
   |                                            |
   |----- ICE Check + DTLS ClientHello -------->|
   |<-2-- ICE Response + DTLS ServerHello/Fin --|
   |----- DTLS Finished (lost) --------x        |
   |------------- Application data ------------>|
   |----- ICE Check + DTLS Finished (resend)--->|
~~~

## Embedding data, acknowledgements and resends

The protocol defined in this specification embeds DTLS data as a STUN attribute, similar to the DATA attribute defined in {{?RFC5766}}.
The recipient acknowledges data with this attribute using another STUN attribute containing a list of CRC-32 hashes of the data.

In addition to allowing saving one round-trip time for establishing the connection this has also shown to be useful for dealing
with packet loss due to the higher frequency of ICE connectvity checks compared to DTLS resends (which commonly use an exponential backoff)
as well as utilizing that ICE is potentially probing multiple candidate pairs. This is particularly important for fragmented handshake packets,
such as those used in DTLS-PQC, where only one of many packets in a DTLS flight might be lost and need retransmission.

While some of the semantics defined in this specification are specific to establishing the DTLS connection,
the concept of using STUN attributes, e.g. on the periodic STUN consent defined in {{?RFC7675}},
as a transport for embedding data

* whose receipt should be acknowledged
* should be sent on multiple paths
* is not time-critical

is applicable on a wider scope and can be specified by defining STUN attribute pairs for data and acknowledgements.

## ICE procedures
To manage delivery of DTLS handshake packets, the ICE agent maintains a list of outbound DTLS packets that have not yet been acknowledged by the peer. Each packet is identified by a CRC-32 hash. Packets are resent until they are acknowledged or removed from the list of outbound packets.

Packets can be sent embedded in STUN messages using the META-DTLS-IN-STUN attribute, or without embedding on a validated ICE candidate pair.

The agent also maintains a list of the CRC-32 hashes of received DTLS handshake packets to send as acknowledgements to the peer.

### Inband discovery and negotiation
The protocol is using in-band discovery to determine support in addition to the use of ice-options for negotiation using SDP offer/answer described below.
This is necessary as the STUN binding requests may reach the offerer before the SDP answer.

Until the DTLS handshake has finished, the ICE agent includes the META-DTLS-IN-STUN-ACKNOWLEDGEMENT and META-DTLS-IN-STUN
attributes in binding requests and responses as described below.

As a "closing handshake" signaling that this protocol (and the embedded DTLS handshake) has finished, the ICE agent

* stops including the META-DTLS-IN-STUN attribute in STUN messages it sends when receiving a STUN messages that includes an empty META-DTLS-IN-STUN attribute and a META-DTLS-IN-STUN-ACKNOWLEDGMENT attribute.
* stops including the META-DTLS-IN-STUN-ACKNOWLEDGEMENT attribute in STUN messages it sends when receiving a STUN message that includes a
META-DTLS-IN-STUN-ACKNOWLEDGEMENT attribute but no META-DTLS-IN-STUN attribute.

An ICE agent receiving a STUN binding request or response that contains neither a META-DTLS-IN-STUN-ACKNOWLEDGEMENT nor a
META-DTLS-IN-STUN attribute determines that the peer does not support the protocol defined in this specification
and stops including of these attributes.

### Receipt acknowledgements
An ICE agent maintains two lists related to DTLS packet delivery:

* A "pending" list of CRC-32 hashes of all outbound DTLS packets that have not yet been acknowledged.
* A "received" list of CRC-32 hashes of all unique inbound DTLS packets that it has processed.

When an agent sends a STUN message, it includes the contents of its "received" list in a META-DTLS-IN-STUN-ACKNOWLEDGEMENT attribute.
This list MUST be included, even if empty, until the ICE agent receives a STUN binding request or response that
contains a META-DTLS-IN-STUN-ACKNOWLEDGEMENT attribute but does not contain a META-DTLS-IN-STUN attribute. This
acts as a "closing handshake" for the embedding.

When an agent receives a STUN binding request or response containing a META-DTLS-IN-STUN attribute, it calculates the CRC-32 hash of the embedded DTLS packet.
If this hash is not already in its "received" list, it adds it.

An outbound DTLS packet is considered acknowledged and removed from the "pending" list if either of these conditions is met:

* A STUN message is received from the peer containing a META-DTLS-IN-STUN-ACKNOWLEDGEMENT attribute that includes the packet's CRC-32 hash.
* For a DTLS packet sent embedded in a STUN binding request, a STUN binding success response is received for that request. This serves as an implicit acknowledgement.

The "pending" list is cleared when the DTLS layer signals that

* a new flight is going to be sent.
* the current flight of packets has timed out and a resend is going to happen.

### Completion of the DTLS handshake
The DTLS layer MUST notify the ICE agent when the DTLS handshake is complete, its role and what DTLS version was negotiated.

The ICE agent clears the "pending" list of outgoing packets if either
* the DTLS layer is acting as a DTLS client and the DTLS version is 1.2, or
* the DTLS layer is acting as a DTLS server and the DTLS version is 1.3.

### For pairs that are in WAITING state
When an ICE candidate pair has not received a response, DTLS can not be sent without being embedded as the candidate pair does not have consent
from the other side. In that state, when sending a STUN binding request the ICE agent embeds

* any pending acknowledgments using the META-DTLS-IN-STUN-ACKNOWLEDGMENT attribute and
* the next pending DTLS packet (determined in a round-robin fashion) using the META-DTLS-IN-STUN attribute. If the "pending" list is empty, the attribute MUST be included with a zero-length value.

When receiving a binding request with an META-DTLS-IN-STUN attribute, the ICE agents embeds

* any pending acknowledgments using the META-DTLS-IN-STUN-ACKNOWLEDGMENT attribute and
* the next pending DTLS packet (determined in a round-robin fashion) using the META-DTLS-IN-STUN attribute. If the "pending" list is empty, the attribute MUST be included with a zero-length value.

### For pairs in SUCCEEDED state
When the ICE candidate pair has received a binding response and is in succeeded state, any new DTLS flights SHOULD immediately be sent without being embedded in
STUN or waiting for the next ICE check. This avoids race conditions where the last part of the DTLS handshake is buffered waiting for the next STUN packet
while DTLS data is already sent ahead of the final handshake packets.

When the STUN agent is sending a scheduled check as described in {{Section 5.8 of ?RFC5245}} it SHOULD embed

* any pending acknowledgments using the META-DTLS-IN-STUN-ACKNOWLEDGMENT attribute and
* the next pending DTLS packet (determined in a round-robin fashion) using the META-DTLS-IN-STUN attribute.

### Resending packets
Resends should happen with the regularly scheduled ICE checks but MAY be triggered by the receipt of an acknowledgment and determining that a
packet did not arrive as expected (either because it was lost or it arrived out of order).

### Sending on multiple candidate pairs
ICE performs checks on multiple candidate pairs in parallel. This implies that embedding a DTLS packet
can happen in rapid succession without the packet embedded first having a chance to reach the peer and
the other side will potentially receive many duplicated DTLS packets.

### Lite agents
Lite ICE agents which are commonly used by servers by definition only respond to binding requests and do not send
binding requests themselves. Due to the lock-step behavior of DTLS this is not a problem, also Lite ICE Agents
can send DTLS without embedding once they received a valid binding request from a peer.

## DTLS procedures
For the protocol described in this specification the DTLS handshake is started before ICE finds a valid pair.

Instead of receiving the DTLS packets after demultiplexing (described in {{Section 7 of ?RFC7983}}),
the DTLS layer receives packets from the ICE layer directly.

### MTU considerations
Embedding DTLS in STUN requires considerations for reducing the MTU used by the DTLS layer for the fragmentation of the handshake.
The goal is to fit the DTLS handshake packets into STUN packets with a predefined maximum size.

The following attributes must be taken into account:

|Attribute|Size|Defined in|
|STUN header|20|{{?RFC8489}}|
|ICE-CONTROLLED /ICE-CONTROLLING|12|{{Section 7.3.1 of ?RFC8445}}|
|PRIORITY|8|{{Section 7.3.1 of ?RFC8445}}|
|USE-CANDIDATE|4|{{Section 7.3.1 of ?RFC8445}}, not on first packet but subsequent packets|
|MESSAGE-INTEGRITY|24|{{Section 15.4 of ?RFC8489}}|
|FINGERPRINT|8|{{Section 15.5 of ?RFC8489}}|
|DTLS-in-STUN|4|This specification. Overhead for the attribute header.|
|DTLS-in-STUN-ACKNOWLEDGEMENT|20|This specification.|
|USERNAME|16+|{{Section 7.3.1 of ?RFC8445}}. Variable, typically 4 byte header plus 9 bytes for two four-byte username fragments and the colon plus 3 bytes padding. The actual size is known before the DTLS exchange starts, either from the SDP exchange or a peer-reflexive candidate.|
|TURN XOR-PEER-ADDRESS|24|Assuming 16 bytes IPv6, see {{?RFC8656}}|
|Total|124+|

Applications that use additional STUN attributes MUST reduce the DTLS MTU further.
The reduced MTU should only be used until the DTLS handshake is complete.

### Handling flights consisting of multiple packets
A single DTLS flight may be too large to fit into a single UDP packet, especially when using Post-Quantum Cryptography (PQC).

Addressing this without re-introducing additional delays is an open question.

# STUN Extensions

## New attributes

This specification extends {{?RFC8489}} and defines two new attributes, META-DTLS-IN-STUN and
META-DTLS-IN-STUN-ACKNOWLEDGEMENT. These attributes are comprehension-optional.

## META-DTLS-IN-STUN
The META-DTLS-IN-STUN attribute may be present in binding requests, responses and indications.  The value portion of this attribute is variable length and
consists of the DTLS handshake flights as described in {{Section 5.1 of RFC9147}} or {{Section 4.2 of ?RFC6347}}.

If the length of this attribute is not a multiple of 4, then padding must be added after this attribute. If the value portion of this attribute is empty
or the first byte is not DTLS (i.e. between 20 and 63 (inclusive) as described in {{Section 3 of !RFC9443}}) the attribute SHOULD be silently discarded

Once the DTLS handshake is completed this attribute SHOULD be silently discarded.

## META-DTLS-IN-STUN-ACKNOWLEDGEMENT
TODO(NEEDS CONSENSUS): does this attribute have a maximum length?

The META-DTLS-IN-STUN-ACKNOWLEDGEMENT attribute may be present in binding requests, responses and  indications. The value portion of this attribute is
variable length and consists of up to four CRC-32 values that are computed for the META-DTLS-IN-STUN values whose receipt the sender acknowledges
(similar to the STUN FINGERPRINT attribute defined in {{Section 15.5 of ?RFC8489}}.

If the length of the value is not a multiple of four or exceeds 16 bytes this attribute SHOULD be silently discarded.
The order does not need to match the order in which the packets were received.

The attribute MAY be empty but included in a binding request or binding response to signal support for the protocol defined in this specification.
This typically happens when the SDP answerer has a "passive" DTLS role and sends binding requests which may arrive at the SDP offerer before the answer.
It is recommended that this attribute is included before the META-DTLS-IN-STUN attribute.

# SDP Offer/Answer Procedures
TODO(NEEDS CONSENSUS): is this needed?

The protocol is designed to work with in-band discovery as described above. If negotiating the protocol via the SDP
offer/answer mechanism using the 'ice-options' attribute is desired, the procedures for that are outlined below.

## Generating the Initial SDP Offer
If the offering endpoint supports the extension defined in this specification, it includes the "sped" ICE option in the SDP.
The offerer MUST be prepared to receive the STUN attributes described below even before receiving the SDP answer.

## Generating the SDP Answer
An answering endpoint not supporting the ice-option described in this document must not include it in its response.
Otherwise the endpoint includes the "sped" ice-option in its answer and can start using the STUN attributes using the semantics defined below.

## Offerer Processing of the SDP Answer
If the answer does not include the "sped" ice-option, the offerer SHOULD ignore the STUN attributes defined in this specification
and MUST NOT send them. Otherwise the offerer includes the STUN attributes using the semantics defined below.

## Modifying the Session
Subsequent offers and answers MUST include the ice-option in the negotiated SDP with the same value as in the initial negotiation.
Remote offers MAY renegotiate ice-options only when negotiating a new DTLS association as described in {{Section 5.5 of !RFC8842}}.

# Security Considerations

## DTLS security considerations
This specification uses application layer caching of DTLS packets which means packets may be sent multiple times using the same
sequence number. For the receiver these will be considered a replay if received multiple times and rejected as described in {{Section 4.5.1 of ?RFC9147}}.

The embedded DTLS handshake is authenticated by the ICE username and message-integrity.

## Pacing and Congestion
The protocol defined in this specification increases the size of the STUN packets that are sent by the ICE agent to a peer without
knowing if that peer consents to receiving the packets. The STUN requests used for embedding DTLS are already paced as described
in {{Section B.1 of ?RFC8845}} which should prevent issues.

# IANA Considerations

This document defines two new STUN attributes, META-DTLS-IN-STUN and META-DTLS-IN-STUN-ACKNOWLEDGEMENT. These attributes will need to be registered with IANA in the "STUN Attributes" registry, following the procedures defined in {{?RFC8489}}. Provisional names have been used in this draft and the registry.

If an ice-option is considered necessary, the IANA shall register the following ICE option in the "ICE Options" subregistry of the
"Interactive Connectivity Establishment (ICE) registry", following the procedures defined in {{?RFC6336}}.

**ICE Option**: sped

**Contact**: TODO

**Change controller**: TODO

**Description**: An ICE option of 'sped' indicates support for embedding DTLS in STUN as described in this specification.

# Prior Work

## ICE-DTLS

The ICE-DTLS draft from 2012 proposed a similar mechanism to SPED, in which a single RTT could be removed from session setup by
replacing STUN Request and Response messages with DTLS ClientHello and ServerHello messages (rather than piggybacking the DTLS
messages as SPED does).

The ICE-DTLS mechanism ends up being considerably more complex than SPED, on account of the fact that the entirety of ICE
functionality needs to be ported over to DTLS (eg consent checks) or retained as a complementary approach (eg peer address discovery).
Furthermore, since it changes the details of connectivity negotiation, it is not backward compatible and therefore must be negotiated
via SDP ice-options.

SPED, on the other hand, is inherently backward compatible with existing WebRTC implementations. It is also more compatible with
demultiplexing multiple ICE sessions using different STUN username fragments on the same UDP port.

--- back

# Acknowledgments
{:numbered="false"}





