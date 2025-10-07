---
title: "STUN Protocol for Embeddiing DTLS"
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
  latest: https://fippo.github.io/warp-snap-sped/draft-todo-webrtc-sped-latest.html

author:
 -
    fullname: Philipp Hancke
    organization: Meta Platforms Inc.
    email: philipp.hancke@googlemail.com

normative:

informative:

--- abstract

TODO

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
   |<-3---------- DTLS ServerHello —------------|
   |------------- DTLS Finished --------------->|
   |<-4---------- DTLS Finished ----------------|
   |------------- Application data ------------>|
~~~

In addition to that deployment experience has shown that there are reliability issues caused by the exponential backoff done by DTLS implementations in particular on connections with a high round trip time whereas the underlying ICE layer is doing periodic checks without exponential backoff.

The protocol defined in this specification aims to reduce this by embedding the DTLS handshake into STUN which eliminates the delay caused by the serialization of the protocols and enhances reliability with more frequent resends and acknowledgements. This is backward compatible with the demultiplexing of multiple ICE sessions on the same port which is frequently done by ICE servers and includes the DTLS handshake in the calculation of the STUN message-integrity field.

The protocol is designed to work with DTLS 1.2, DTLS 1.3 and can operate with DTLS 1.3 with PQC groups which increase the DTLS handshake flight size. As described in WARP-DOCUMENT it can reduce the number of RTTs to 2 which makes it on-par with the older SDES approach for some scenarios.

# Terminology

**DTLS Flight**: one of the DTLS handshake messages defined in {{Section 5.7 of ?RFC9147}}.
A flight may be split into multiple DTLS packets when the size of the flight exceeds the MTU.

**ICE candidate pair**: TODO

# SDP Offer/Answer procedures

NOTE: **no consensus yet**

## General

This section defines how the ‘sped’ ICE options can be negotiated using the SDP offer/answer mechanism.
Negotiating this in SDP provides that the other side is prepared to receive the new attributes even in
packets that are sent without prior STUN consent {{?RFC7675}}.

## Generating the Initial SDP Offer
If the offering endpoint supports the extension defined in this specification, it includes the "sped" ICE options in the SDP.
The offerer MUST be prepared to receive the STUN attributes described below even before receiving the SDP answer.

## Generating the SDP Answer
An answering endpoint not supporting the ice-option described in this document must not include it in its response.
Otherwise the endpoint includes the "sped" ice-option in its answer and can start using the STUN attributes using the semantics defined below.

## Offerer Processing of the SDP Answer
If the answer does not include the ‘sped’ ice-option, the offerer MUST ignore the STUN attributes defined in this specification and MUST NOT send them.
Otherwise the offerer includes the STUN attributes using the semantics defined below.

## Modifying the Session
Subsequent offers and answers MUST include the ice-option in the negotiated SDP with the same value as in the initial negotiation.
Remote offers MAY renegotiate ice-options only when negotiating a new DTLS association as described in {{Section 5.5 of !RFC8842}}.

# Embedding DTLS in STUN

For the protocol described in this specification the DTLS handshake is started before ICE finds a valid pair and SHOULD disable the DTLS
resend timeout as resends will be handled by the STUN/application layer.

For the purpose of resends, the ICE agent stores the outgoing DTLS packets until their receipt has been acknowledged as described below.

## For pairs that are in WAITING state
When an ICE candidate pair has not received a response DTLS can not be sent without being embedded as the candidate pair does not have consent from the other side. In that state, when sending a STUN binding request the STUN agent MUST embed any pending DTLS flights into the STUN packet using the META-DTLS-IN-STUN attribute. If the flight is split into multiple fragments, a separate binding request is sent for each fragment.

When receiving a binding request with an META-DTLS-IN-STUN attribute the DTLS response to any flight contained therein can be directly embedded in the binding response.

## For pairs in SUCCEEDED state
When the ICE candidate pair has received a binding response and is in succeeded state, any new DTLS flights SHOULD immediately be sent without being embedded in STUN or waiting for the next ICE check.

When the STUN agent is sending a scheduled check as described in {{Section 5.8 of ?RFC5245}} it should add any pending acknowledgements
and then embed any pending DTLS packet into the STUN message.

## ICE-lite considerations
TODO: go through Jonas' rules

## Resending flights
The STUN agent stores outgoing DTLS packets and their CRC-32 hash that are sent either embedded in STUN messages using the META-DTLS-IN-STUN
attribute or directly and resends them TODO(at what pace) until it receives an acknowledgement of the receipt of those packets as described below.

Resends should happen with the regularly scheduled ICE checks but MAY be triggered by the receipt of an acknowledgment and determining that a
packet did not arrive as expected (either because it was lost or it arrived out of order).

## Receipt acknowledgements
The STUN agent acknowledges the receipt of DTLS packets by including the CRC-32 of the packet in the META-DTLS-IN-STUN-ACKNOWLEDGEMENT attribute.
The STUN agent stops including the META-DTLS-IN-STUN-ACKNOWLEDGEMENT in messages it sends when receiving a STUN message that includes a
META-DTLS-IN-STUN-ACKNOWLEDGEMENT attribute but no META-DTLS-IN-STUN attribute which signals that the remote STUN agent has completed the DTLS handshake.

**TODO: needs consensus** During the DTLS handshake the receiving STUN agent MAY acknowledge packets both received embedded in STUN as well as received without embedding.

When a sender receives a STUN binding response for a packet that includes an embedded DTLS packet it adds the CRC-32 of that packet to the list
of acknowledgements from the META-DTLS-IN-STUN-ACKNOWLEDGEMENT attribute unless the CRC-32 is already included.

When a receiver receives a STUN binding response for a packet that includes a META-DTLS-IN-STUN-ACKNOWLEDGEMENT attribute it can remove the
CRC-32s from the list of received packets.

## Handling flights consisting of multiple packets
TODO: sending, receiving, acknowledgement. Send in batch?

## MTU considerations
Embedding DTLS in STUN requires considerations for reducing the MTU used by DTLS for the fragmentation of the handshake.
In addition to the STUN header  the following attributes must be taken into account:

|Attribute|Size|Defined in|
|STUN header|20|RFC 5389|
|ICE-CONTROLLED /ICE-CONTROLLING|12|{{Section 19.1 of RFC5245}}|
|PRIORITY|8|{{Section 19.1 of RFC5245}}|
|USE-CANDIDATE|4|{{Section 19.1 of RFC5245}}, not on first packet but subsequent packets|
|MESSAGE-INTEGRITY|24|{{Section 15.4 of ?RFC5389}}|
|FINGERPRINT|8|{{Section 15.5 of RFC5389}}|
|DTLS-in-STUN|4|This specification. Overhead for the attribute header.|
|DTLS-in-STUN-ACKNOWLEDGEMENT|20|This specification.|
|USERNAME|16+|{{Section 7.1.2.3 of RFC5245}}. Variable, typically 4 byte header plus 9 bytes for two four-byte username fragments and the colon plus 3 bytes padding. The actual size is known before the DTLS exchange starts, either from the SDP exchange or a peer-reflexive candidate.|
|TURN XOR-PEER-ADDRESS|24|Assuming 16 bytes IPv6.|
|Total|124+|

Applications that use custom STUN attributes MUST reduce the DTLS MTU further.

TODO: do we need wording for {{Section 7.1 of RFC5389}} (576 bytes for IPv4, 1200 for IPv6) in particular for PQC?
We can argue that WebRTC works despite sending packets larger than 576 by default)

# Example negotiation
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

# New STUN attributes

This STUN extension defines the following new attributes:

* **0xC070**: META-DTLS-IN-STUN (TODO: new name, same code)
* **0xC071**: META-DTLS-IN-STUN-ACKNOWLEDGEMENT (TODO: ^)

Some of these attributes have lengths that are not multiples of 4. By the rules of STUN, any attribute whose length is not a multiple
of 4 bytes MUST be immediately followed by 1 to 3 padding bytes to ensure the next attribute (if any) would start on a 4-byte boundary
(see {{RFC5389}}).

## META-DTLS-IN-STUN
The META-DTLS-IN-STUN attribute may be present in binding requests, responses and indications.  The value portion of this attribute is variable length and
consists of the DTLS handshake flights as described in {{Section 5.1 of RFC9147}} or {{Section 4.2 of ?RFC6347}}.

If the length of this attribute is not a multiple of 4, then padding must be added after this attribute. If the value portion of this attribute is empty
or the first byte is not DTLS (i.e. between 20 and 63 (inclusive) as described in {{Section 3 of !RFC9443}}) the attribute SHOULD be silently discarded

Once the DTLS handshake is completed this attribute SHOULD be silently discarded.

## META-DTLS-IN-STUN-ACKNOWLEDGEMENT
The META-DTLS-IN-STUN-ACKNOWLEDGEMENT attribute may be present in binding requests, responses and  indications. The value portion of this attribute is
variable length and consists of up to four CRC-32 values that are computed for the META-DTLS-IN-STUN values whose receipt the sender acknowledges
(similar to the STUN FINGERPRINT attribute defined in {{Section 15.5 of RFC5389}}.

If the length of the value is not a multiple of four or exceeds 16 bytes this attribute SHOULD be silently discarded.
The order does not need to match the order in which the packets were received.

The attribute MAY be empty but included in a binding request or binding response to signal support for the protocol defined in this specification.
This typically happens when the SDP answerer has a "passive" DTLS role and sends binding requests which may arrive at the SDP offerer before the answer.
It is recommended that this attribute is included before the META-DTLS-IN-STUN attribute.

# Security Considerations
**TODO** This has security implications. In particular caching DTLS flights and resending them. This is compatible with the replay detection

**TODO**: Upside: DTLS is authenticated as it is now covered by the STUN M-I.

**TODO**: Larger packets without consent, DDoS consideration needed?

# IANA Considerations
**TODO**: Attributes already registered, maybe rename them

IANA has registered the following ICE option in the "ICE Options" subregistry of the
"Interactive Connectivity Establishment (ICE) registry", following the procedures defined in {{?RFC6336}}.

**ICE Option**: sped

**Contact**: IESG <iesg@ietf.org>

**Change controller**: IESG

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





