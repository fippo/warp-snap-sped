---
title: "STUN Protocol for Embedding DTLS (SPED)"
abbrev: "SPED"
category: info

docname: draft-hancke-webrtc-sped-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: ART
workgroup: Audio/Video Transport Core Maintenance
keyword:
 - webrtc
 - stun
 - dtls
venue:
  group: AVTCORE
  type: Working Group
  mail: avt@ietf.org
  arch: https://datatracker.ietf.org/wg/avtcore/
  github: fippo/warp-snap-sped
  latest: https://fippo.github.io/warp-snap-sped/draft-hancke-webrtc-sped-latest.html

author:
 -
  fullname: Philipp Hancke
  organization: Meta Platforms Inc.
  email: philipp.hancke@googlemail.com
 -
  fullname: Justin Uberti
  organization: OpenAI
  email: justin@uberti.name
 -
  fullname: Jonas Oreland
  organization: Google
  email: jonaso@google.com

normative:

informative:

--- abstract

WebRTC setup normally serializes ICE and DTLS, adding at least one extra round trip before secure
media can flow. This document defines the STUN Protocol for Embedding DTLS (SPED), which carries
DTLS handshake data and acknowledgements inside STUN Binding Requests and Responses. SPED allows
ICE and DTLS to proceed in parallel, improves setup behavior under loss, and remains backward
compatible with existing ICE processing.

--- middle

# Introduction

The current WebRTC connection setup, as outlined in {{?RFC8829}}, incurs a minimum of 4 RTTs with
DTLS 1.2, or 3 RTTs with DTLS 1.3, before media can be sent. The serialization of ICE and DTLS is
a large contributor to that as illustrated below for DTLS 1.2:

~~~
Client                                      Server
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

In addition, deployment experience has shown connection setup reliability issues in scenarios with
packet loss, caused by the exponential backoff timer typically used in DTLS implementations.

The protocol defined in this specification, SPED, aims to resolve these concerns by embedding the
DTLS handshake into STUN, eliminating the delay caused by the serialization of the protocols and
improving reliability by sending fewer packets as well as simplifying retransmissions. In fact,
when DTLS 1.3 is used, the protocol can reduce the setup latency to as little as a single
round-trip, comparable to the latency of the largely deprecated SDES key exchange mechanism
{{?RFC4568}}.

The protocol is backward compatible, supports both DTLS 1.2 {{?RFC6347}} and DTLS 1.3
{{?RFC9147}}, and can accommodate all DTLS cipher suites, including post-quantum cryptography
(PQC) suites that can increase the number of packets sent during DTLS handshaking.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Design

## Background

### ICE Overview

The ICE protocol {{?RFC8445}} is complex, but the core steps taken by each ICE agent (client) can be summarized
as follows:

1. Enumerate local ICE candidates and send them, out-of-band, to the peer.
2. Combine local ICE candidates with the received remote ICE candidates to form ICE candidate
   pairs.
3. Evaluate the usability of these ICE candidate pairs by sending STUN Binding Requests. This
   typically happens in parallel, i.e. an ICE agent may have several binding requests in flight
   when there are multiple candidate pairs.
4. When a STUN Binding Request is received, reply with a STUN Binding Response.
5. When a STUN Binding Response is received, in response to a request, mark the associated ICE
   candidate pair as valid.
6. If the ICE agent is in the controlling role, select the "best" ICE candidate pair for
   subsequent sending of data or media, and indicate the selected candidate pair to the remote ICE
   agent by sending a new STUN Binding Request with the USE-CANDIDATE flag set.

Some endpoints, typically servers, implement a simpler form of ICE known as ICE Lite {{?RFC8445}}. When this
form of ICE is used, the ICE Lite endpoint omits steps 3 and 5, and Binding Requests only flow in
one direction, from the full to the lite endpoint.

### DTLS Overview

In WebRTC, DTLS handshaking normally starts once ICE has identified a valid candidate pair, using
"client" and "server" roles determined through the `a=setup` attribute in WebRTC SDP signaling
{{?RFC5763}}. SPED changes when these DTLS packets can be sent, but not the DTLS handshake
contents themselves.

We define the term "DTLS packet" to mean the unit of DTLS data typically carried in a
single UDP packet.

DTLS handshake messages are also organized into "DTLS flights", as detailed in {{Section 5.7 of ?RFC9147}}. A
DTLS flight consists of a set of handshake messages that are sent together by a DTLS endpoint, and
those messages are carried in one or more DTLS packets.

Ideally, a flight, even if it contains multiple messages, can fit into a single DTLS packet.
However, if a message is large, for example a large certificate, it can be fragmented across
multiple DTLS packets.

The DTLS flights used during WebRTC session setup are described below. Note that because
ICE has already demonstrated remote consent, DTLS' HelloVerifyRequest is not needed to prevent DoS
attacks.

Once the DTLS handshake has completed, SRTP key extraction occurs and is used to key the sending
of media {{?RFC5764}}.
Media cannot be properly decrypted until all handshake messages have been received.

#### DTLS 1.2 Handshake

The DTLS 1.2 handshake, as specified in {{Section 4.2 of ?RFC6347}}, is organized into the
following DTLS flights:

1. The DTLS client sends the ClientHello message.
2. The DTLS server responds with the ServerHello, Certificate, ServerKeyExchange,
   CertificateRequest, and ServerHelloDone messages, packed as noted above.
3. The DTLS client sends the Certificate, ClientKeyExchange, CertificateVerify,
   ChangeCipherSpec, and Finished messages.
4. The DTLS server sends the ChangeCipherSpec and Finished messages.

#### DTLS 1.3 Handshake

The DTLS 1.3 handshake, as specified in {{Section 5 of ?RFC9147}}, is organized into the
following DTLS flights:

1. The DTLS client sends the ClientHello message.
2. The DTLS server sends the ServerHello, EncryptedExtensions, CertificateRequest, Certificate,
   CertificateVerify, and Finished messages.
3. The DTLS client sends the Certificate, CertificateVerify, and Finished messages.

Note that in DTLS 1.3, the DTLS server sends a DTLS acknowledgement packet upon receiving the
Finished message, but the client does not need to wait for this message to begin sending encrypted
data.

## Goals

The desired properties of this solution are:

* It makes WebRTC setup faster by 1 RTT, by allowing ICE and DTLS to proceed in parallel.
* It makes WebRTC session setup less susceptible to packet loss.
* It is strictly an optimization.
* It reduces the number of packets exchanged during session setup, but the size of the STUN
  Binding Request or Response increases.
* In the event of an incompatibility, each client proceeds with ICE and DTLS as usual.
* It works with all versions of DTLS >= 1.2, and all DTLS cipher suites.
* It is fully backward compatible with existing ICE processing, including interactions with ICE
  Lite endpoints, as well as endpoints that demultiplex multiple ICE sessions on the same port.

## SPED Protocol

### Summary

The overall mechanism can be summarized as follows:

1. DTLS is started at the same time as ICE.
2. If there is no valid ICE candidate pair, DTLS handshake packets are sent by encapsulating them
   in a new STUN attribute in the next STUN Binding Request or STUN Binding Response.
3. Once a valid ICE candidate pair exists, the client can continue to send DTLS packets either in
   embedded form, or as usual over the specified pair.

In addition, to improve the reliability of the DTLS handshake, an explicit acknowledgement
mechanism is built into SPED. Encapsulated DTLS handshake packets are acknowledged by sending their
CRC-32 in a new STUN attribute in the next STUN Binding Request or STUN Binding Response.

### New STUN Attributes

This STUN extension defines the following new IETF-assigned attributes in the comprehension-optional range:

* `TBD1`: `DTLS-IN-STUN-DATA`
* `TBD2`: `DTLS-IN-STUN-ACK`

These attributes have lengths that are not always multiples of 4. By the rules of STUN, any
attribute whose length is not a multiple of 4 bytes MUST be immediately followed by 1 to 3 padding
bytes to ensure the next attribute, if any, starts on a 4-byte boundary; see {{?RFC5389}}.

#### DTLS-IN-STUN-DATA

* This attribute contains one DTLS handshake packet, or is empty to indicate SPED support when no
  DTLS packet is being embedded.
* While SPED is active, this attribute MUST be present in every STUN Binding Request or Response
  sent by a SPED-capable agent.
* The value portion of this attribute is variable length and consists of one DTLS handshake packet
  from a DTLS flight, as described in {{Section 5.1 of ?RFC9147}} or {{Section 4.2 of ?RFC6347}}.
* As noted, if the attribute length is not a multiple of 4, padding must be added.
* If the value portion of this attribute is empty, it indicates SPED support and that no DTLS
  packet is being embedded in that STUN message. An empty value MUST NOT be injected into the DTLS
  layer.
* If the value portion of this attribute is non-empty but the first byte is not DTLS, i.e. between
  20 and 63 inclusive as described in {{Section 3 of ?RFC9443}}, the attribute SHOULD be silently
  discarded.

#### DTLS-IN-STUN-ACK

* This attribute contains acknowledgements of received `DTLS-IN-STUN-DATA` attributes.
* The attribute can be present in either a STUN Binding Request or Response.
* The attribute is variable length and contains a list of uint32 entries, where each entry is the
  computed CRC-32 of a received `DTLS-IN-STUN-DATA` attribute value, i.e. a DTLS handshake packet,
  ignoring padding.
* The attribute can be empty, i.e. the length of the list of uint32 values can be 0.

### MTU Considerations

When embedding DTLS in STUN, the DTLS MTU MUST take into account the STUN packet overhead, which
is noted in the table below:

| Attribute | Size | Defined in |
| --- | --- | --- |
| STUN header | 20 | {{?RFC5389}} |
| ICE-CONTROLLED / ICE-CONTROLLING | 12 | {{Section 19.1 of ?RFC5245}} |
| PRIORITY | 8 | {{Section 19.1 of ?RFC5245}} |
| USE-CANDIDATE | 4 | {{Section 19.1 of ?RFC5245}}; not on first packet but on subsequent packets |
| MESSAGE-INTEGRITY | 24 | {{Section 15.4 of ?RFC5389}} |
| MESSAGE-INTEGRITY-SHA256 | 36 | {{Section 14.6 of ?RFC8489}}; only applicable when `ice2` is used {{Section 10 of ?RFC8445}} |
| FINGERPRINT | 8 | {{Section 15.5 of ?RFC5389}} |
| DTLS-IN-STUN-DATA | 4 | This specification. Overhead for the attribute header |
| DTLS-IN-STUN-ACK | 4 | This specification. Overhead for the attribute header; TODO: define max size |
| USERNAME | 16+ | {{Section 7.1.2.3 of ?RFC5245}}. Variable, typically 4 byte header plus 9 bytes for two four-byte username fragments and the colon plus 3 bytes padding. The actual size is known before the DTLS exchange starts, either from the SDP exchange or a peer-reflexive candidate |
| TURN XOR-PEER-ADDRESS | 24 | {{?RFC8656}}. Assuming 16 byte IPv6; only applicable when TURN is used |

Accordingly, the typical 1200 byte DTLS MTU, based on the recommendation in {{?RFC8831}}, MUST be
reduced by the size of the expected overhead. Applications that use custom STUN attributes, i.e. not in the table above, MUST reduce the
DTLS MTU further.

### Backwards Compatibility

SPED is fully backwards compatible with existing ICE agents. If the peer ICE agent does not
support SPED, this can be detected via the lack of the mandatory `DTLS-IN-STUN-DATA` attribute in
its first authenticated ICE check or response, and upon recognizing this fact the local ICE agent
can easily fall back to standard unencapsulated DTLS.

Given this straightforward in-band negotiation, this specification does not currently define an
offer/answer negotiation mechanism or any ICE options.

# Mechanism

The specifics of the SPED algorithm are detailed below.

## Setup

When using SPED, an ICE agent keeps two lists:

1. A list, L1, of pending DTLS handshake packets.

   These packets are created by the DTLS layer. The list is cleared when the DTLS layer creates a
   new flight, or elements in the list are removed when ACKed by the peer.

2. A list, L2, of pending acknowledgements, as defined above.

## Sending a STUN Binding Request or Response

When sending a STUN Binding Request or Response, the ICE agent MUST follow the steps below:

1. Embed any pending ACKs from L2 in a DTLS-IN-STUN-ACK attribute.
2. If there is a pending DTLS handshake packet in L1 and sufficient space remains in the STUN
   message, embed one DTLS handshake packet from L1 into a `DTLS-IN-STUN-DATA` attribute.
3. Otherwise, include `DTLS-IN-STUN-DATA` with an empty value simply to indicate SPED support.


## Receiving a STUN Binding Request or Response

When receiving a STUN Binding Request or Response, the ICE agent MUST follow the steps below:

1. If this is the first authenticated STUN message received from the peer, and the
   `DTLS-IN-STUN-DATA` attribute is not present, conclude that the peer does not support SPED, and
   conclude SPED processing.
2. If the STUN message contains a `DTLS-IN-STUN-ACK` attribute, process the CRC-32 values in the
   attribute and remove each ACKed DTLS handshake packet from L1.
3. If the STUN message contains a non-empty `DTLS-IN-STUN-DATA` attribute, inject the DTLS
   handshake into the DTLS layer.

When receiving a STUN Binding Response, there is an implicit acknowledgement of any data sent in
the associated STUN Binding Request. Accordingly, the ICE agent MUST also follow the steps below:

1. Remove any DTLS packets sent in the Binding Request from L1.
2. Remove any ACKs sent in the Binding Request from L2.

However, if data is included in the STUN Binding Response, this MUST be ACKed using the explicit
ACK mechanism, and the ICE agent MUST add the CRC-32 of the DTLS packet to L2.

## Termination

Implementations SHOULD terminate use of SPED once a valid ICE candidate pair exists and direct
sending is possible, as this allows transmission of DTLS packets without waiting on an outgoing
STUN Binding Request. However, implementations MAY continue to send embedded DTLS if desired and
only terminate once DTLS handshaking is complete.

# Examples

## Vanilla DTLS 1.2

~~~
Client                                      Server
  |                                            |
  |--------- SDP Offer ----------------------->|
  |<-1------ SDP Answer (a=setup:passive) -----|
  |                                            |
  |--------- STUN BindingRequest ------------->|
  |<-2------ STUN BindingResponse -------------|
  |                                            |
  |--------- DTLS F1: ClientHello ------------>|
  |<-3------ DTLS F2: ServerHello, etc---------|
  |--------- DTLS F3: Finished, etc ---------->|
  |<-4------ DTLS F4: Finished, etc -----------|
  |--------- Application data ---------------->|
~~~

## DTLS 1.2 with SPED

With `a=setup:passive` in the SDP answer, the offerer is the DTLS client:

~~~
Client                                      Server
  |                                            |
  |--------- SDP Offer ----------------------->|
  |<-1------ SDP Answer (a=setup:passive)------|
  |                                            |
  |--------- BindingRequest/DTLS F1 ---------->|
  |<-2------ BindingResponse/DTLS F2 ----------|
  |                                            |
  |--------- DTLS F3: Finished --------------->|
  |<-3------ DTLS F4: Finished ----------------|
  |--------- Application data ---------------->|
~~~

With `a=setup:active` in the SDP answer, the answerer is the DTLS client:

~~~
Client                                      Server
  |                                            |
  |--------- SDP Offer ----------------------->|
  |<-1------ SDP Answer (a=setup:active)-------|
  |                                            |
  |--------- BindingRequest/{} --------------->|
  |<-2------ BindingResponse/DTLS F1 ----------|
  |                                            |
  |--------- DTLS F2: ServerHello ------------>|
  |<-3------ DTLS F3: Finished ----------------|
  |--------- DTLS F4: Finished --------------->|
  |--------- Application data ---------------->|
~~~

The flows are similar when the server uses ICE Lite.

## Vanilla DTLS 1.3

~~~
Client                                      Server
  |                                            |
  |--------- SDP Offer ----------------------->|
  |<-1------ SDP Answer (a=setup:passive) -----|
  |                                            |
  |--------- STUN BindingRequest ------------->|
  |<-2------ STUN BindingResponse -------------|
  |                                            |
  |--------- DTLS F1: ClientHello ------------>|
  |<-3------ DTLS F2: ServerHello, etc --------|
  |--------- DTLS F3: Finished --------------->|
  |--------- Application data ---------------->|
  |<-------- DTLS ACK -------------------------|
~~~

## DTLS 1.3 with SPED

With `a=setup:passive` in the SDP answer, the offerer is the DTLS client:

~~~
Client                                      Server
  |                                            |
  |--------- SDP Offer ----------------------->|
  |<-1------ SDP Answer (a=setup:passive)------|
  |                                            |
  |--------- BindingRequest/DTLS F1 ---------->|
  |<-2------ BindingResponse/DTLS F2 ----------|
  |                                            |
  |--------- DTLS F3: Finished --------------->|
  |--------- Application data ---------------->|
  |<-------- DTLS ACK -------------------------|
~~~

With `a=setup:active` in the SDP answer, the answerer is the DTLS client, and an additional RTT is incurred:

~~~
Client                                      Server
  |                                            |
  |--------- SDP Offer (a=setup:actpass)------>|
  |<-1------ SDP Answer (a=setup:active)-------|
  |                                            |
  |--------- BindingRequest/{} --------------->|
  |<-2------ BindingResponse/DTLS F1 ----------|
  |                                            |
  |--------- DTLS F2: ServerHello, etc ------->|
  |<-3------ DTLS F3: Finished ----------------|
  |--------- Application data ---------------->|
  |--------- DTLS ACK ------------------------>|
~~~

Again, the flows are similar when the server uses ICE Lite.

## DTLS 1.3 with Non-SPED Peer

~~~
Client                                      Server
  |                                            |
  |--------- BindingRequest/DTLS F1 ---------->|
  |<-------- BindingResponse/               ---|
~~~

The absence of a `DTLS-IN-STUN-DATA` attribute allows the client to conclude that the server does
not support SPED.

~~~
  |--------- DTLS F1: ClientHello ------------>|
  |<-------- DTLS F2: ServerHello ------------ |
  |--------- DTLS F3: Finished --------------->|
  |--------- Application data ---------------->|
  |<-------- DTLS ACK -------------------------|
~~~

## DTLS 1.3 with Flight 2 Loss

~~~
Client                                       Server
  |                                             |
  |--------- BindingRequest/DTLS F1 ----------->|
    <- LOST  BindingResponse/DTLS F2 -----------|
  |<-------- BindingRequest/DTLS F2 ------------|
  |--------- BindingResponse/DTLS F3 ---------->|
  |--------- Application data ----------------->|
  |<-------- DTLS ACK --------------------------|
~~~

## DTLS 1.3 with Flight 3 Loss

~~~
Client                                       Server
  |                                             |
  |<-------- BindingRequest/ACK={} -------------|
  |--------- BindingResponse/F1=ClientHello --->|
  |<-------- DTLS ServerHello ------------------|
  |--------- DTLS Finished ------------ LOST -> |
  |--------- BindingRequest/F3=DTLS Finished--->|
~~~

Note: The embedding continues until both client and server are known to be writable, but ICE does
not send any packets it would not otherwise send.

~~~
  |<-------- DTLS ACK --------------------------|
  |--------- Application data ----------------->|
  |<-------- BindingResponse/ACK={F3} ----------|
~~~

## DTLS 1.3 PQC with Certificate Fragment Loss

The DTLS ClientHello is split into 2 packets.
The DTLS ServerHello is split into 2 packets.

~~~
Client                                       Server
  |                                             |
  |--------- BindingRequest/F1=ClientHello/1 -->|
    <- LOST  BindingResponse/ACK={}          ---|
  |<-------- BindingRequest/ACK={F1/1} ---------|
~~~

Note: When the server sends `ACK{F1/1}`, it does not yet have a DTLS packet to send since both of
the packets from the ClientHello have arrived.

~~~
  |--------- BindingRequest/F1=ClientHello/2 -->|
  |<-------- BindingResponse/F2=ServerHello/1 --|
  |<-------- BindingRequest/F2=ServerHello/2 -->|
  |--------- BindingRequest/F3=DTLS Finished -->|
  |--------- DTLS Finished -------------------->|
  |--------- Application data ----------------->|
  |<-------- DTLS ACK --------------------------|
~~~

## DTLS 1.3 PQC with Multiple Candidate Pairs and Certificate Fragment Loss

The DTLS ClientHello is split into 2 packets.
The DTLS ServerHello is split into 2 packets.
There are two candidate pairs, CP1 and CP2.
The ICE agent retransmits BindingRequest once.

~~~
Client                                        Server
  |                                              |
CP1  |--------- BindingRequest/F1=ClientHello/1 --->|
CP1    <- LOST  BindingResponse/ACK={F1/1} ---------|
CP2  |--------- BindingRequest/F1=ClientHello/2  -->|
CP2    <- LOST  BindingResponse/F2=ServerHello/1 ---|
CP1  |--------- BindingRequest/F1=ClientHello/1 --->|
CP1  |<-------- BindingResponse/F2=ServerHello/1 ---|
CP2  |--------- BindingRequest/ACK={F2/1} --------->|
CP2  |<-------- BindingResponse/F2=ServerHello/2 ---|
  |--------- DTLS Finished ------------------------->|
  |--------- Application data ---------------------->|
  |<-------- DTLS ACK -------------------------------|
~~~

# Implementation Notes

The following configuration for the SPED stack is RECOMMENDED:

1. When SPED is active, disable internal DTLS timeouts, and resume them when receiving the first
   STUN Binding Response using a new BoringSSL feature that allows modifying timeouts for
   outstanding flights, <https://boringssl-review.git.corp.google.com/c/boringssl/+/86167>.
2. Limit the size of L2 to 4 elements.
3. When using a PQC cipher suite, force the BoringSSL downward MTU to 900 bytes, which smooths a
   DTLS PQC flight into 2 roughly equal sized DTLS packets, which can fit into a typical network
   MTU even with the STUN embedding overhead.

# Prior Work

## ICE-DTLS

The {{?I-D.thomson-rtcweb-ice-dtls}} draft from 2012 proposed a similar mechanism to SPED, in
which a single RTT could be removed from session setup by replacing STUN Request and Response
messages with DTLS ClientHello and ServerHello messages, rather than piggybacking the DTLS messages
as SPED does.

The ICE-DTLS mechanism ends up being considerably more complex than SPED, on account of the fact
that the entirety of ICE functionality needs to be ported over to DTLS, for example consent checks,
or retained as a complementary approach, for example peer address discovery. Furthermore, since it
changes the details of connectivity negotiation, it is not backward compatible and therefore must
be negotiated via SDP `ice-options`.

# Future Work

Embedding data into STUN requests is a technique that could also be used for early transmission or
improved reliability of other important data. For example, one could imagine transferring
key-frames, if small enough, or RTCP or SCTP control messages. However, this has not been
thoroughly sketched out in this proposal.

# Security Considerations

## DTLS Replay and Spoofing

This specification uses application-layer caching of DTLS packets which means packets can be sent
multiple times using the same sequence number. For the receiver these are considered replays if
received multiple times and rejected as described in {{Section 4.5.1 of ?RFC9147}}.

The embedded DTLS handshake is authenticated by existing ICE logic, i.e. the ICE USERNAME and
MESSAGE-INTEGRITY mechanisms. Any spoofed ICE packets are rejected accordingly.

## Pacing and Congestion

The protocol defined in this specification increases the size of the STUN packets that are sent by
the ICE agent to a peer without knowing if that peer can use the embedded data. Although the
initial data sent is just the DTLS ClientHello, this packet can be close to a MTU when a PQC
cipher suite is used. If this is unacceptable, an offer-answer mechanism for SPED can be used to
address this concern.

The STUN requests used for embedding DTLS are already paced as described in
{{Appendix B.1 of ?RFC8445}}, which limits the outgoing bandwidth from this mechanism.
However, that pacing assumes an ICE check of "less than 120 bytes", which will not be
the case when a DTLS ClientHello is embedded, especially a PQC one.

Solutions to this problem require transmitting the DTLS ClientHello less often, perhaps only
on certain candidate pairs, and is a subject for further study.

# IANA Considerations

This document defines two new STUN attributes, `DTLS-IN-STUN-DATA` and `DTLS-IN-STUN-ACK`. These
attributes need to be registered with IANA in the "STUN Attributes" registry, following the
procedures defined in {{?RFC8489}}. Provisional names have been used in this draft and the
registry.

--- back

# Appendix A: Benchmark Numbers

For the scenario without packet loss, benchmarking is straightforward, and the savings from SPED
amount to 1 RTT, as expected. However, in packet loss scenarios, the savings can be much larger,
especially in the worst, p95, cases.

In this benchmark, a 200 ms RTT is used. Packet loss is simulated using the virtual network
mechanism in Google's libwebrtc. Duration is measured as time from start until both peers have
completed the DTLS handshake.

| DTLS 1.3 with PQC | Loss % | p10 (ms) | p50 | average | p95 |
| --- | --- | --- | --- | --- | --- |
| Vanilla | 0 | 850 | 850 | 850 | 850 |
| DTLS-in-STUN | 0 | 650 | 650 | 650 | 650 |
|  |  |  |  |  |  |
| Vanilla | 5% | 850 | 850 | 947 | 1253 |
| DTLS-in-STUN | 5% | 650 | 650 | 656 | 700 |
|  |  |  |  |  |  |
| Vanilla | 10% | 850 | 900 | 1193 | 2170 |
| DTLS-in-STUN | 10% | 650 | 650 | 685 | 800 |

# Acknowledgments
{:numbered="false"}
