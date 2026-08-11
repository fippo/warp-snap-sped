---
title: "WebRTC Abridged Roundtrip Protocol (WARP)"
abbrev: "WARP"
category: std

docname: draft-uberti-tsvwg-warp-00
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: Web and Internet Transport
workgroup: Transport and Services Working Group
keyword:
  - webrtc
  - ice
  - dtls
  - sctp
  - latency
venue:
  group: tsvwg
  type: Working Group
  mail: tsvwg@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/tsvwg/
  github: fippo/warp-snap-sped
  latest: https://fippo.github.io/warp-snap-sped/draft-uberti-tsvwg-warp-latest.html

author:
  -
    fullname: Justin Uberti
    organization: OpenAI
    email: justin@uberti.name
  -
    fullname: Philipp Hancke
    organization: Meta Platforms Inc.
    email: philipp.hancke@googlemail.com

normative:

informative:

--- abstract

This document outlines a set of improvements to the WebRTC session setup
protocols aimed at significantly reducing connection setup latency. This
improved setup mechanism is known as the WebRTC Abridged Roundtrip Protocol
(WARP). By addressing inefficiencies within the current multi-protocol handshake
process, WARP can reduce setup latency from 6 RTTs to 2 RTTs when its
optimizations are used together, with a direct impact on perceived performance
and reliability.

--- middle

# Introduction

The current WebRTC connection setup, as outlined in {{?RFC8829}}, incurs 4 RTTs
before media can be sent and 6 RTTs before the data channel opens. In 2011, this
wasn’t much worse than the 4 RTTs needed to set up a WebSocket over TCP/TLS, but
nowadays, compared to QUIC’s ({{?RFC9000}}) optimized 0-RTT setup (1 RTT for a
WebSocket), it seems incredibly slow.

In addition, the typical topology for a WebRTC session has changed from being a
peer-to-peer to client-server interaction, on account of the rise of the use of
WebRTC in conferencing, game streaming, and other services that have a centrally
hosted component. While setup latency for peer-to-peer calls was largely masked
by the time needed for the human callee to actually answer the call, this
latency is immediately observable in most client-server contexts. As WebRTC
gains traction in additional low-latency domains (e.g., AI services and
robotics), this trend will continue to increase.

Furthermore, reducing the number of RTTs and packets on the wire has a direct
impact on the robustness of the protocol. With fewer steps, there are fewer
things that can be affected by network delays or losses, which will translate
into improved reliability for all WebRTC topologies.

Accordingly, it is clear that a problem exists, and it is one worth solving.

## Analysis

First, let's look at the current WebRTC setup protocol.

Similar to the situation noted above regarding WebSockets over TCP/TLS, much of
the current slowness comes from multiple protocols being stacked atop one
another, so that there are multiple handshakes and these handshakes are all
serialized.

For WebRTC, there are five separate protocols involved:

- the signaling protocol (eg HTTP), used for offer-answer exchange
- ICE ({{!RFC8445}}), used to establish a transport from the signaling exchange
- DTLS 1.2 ({{!RFC6347}}), used to secure the ICE transport
- SCTP ({{!RFC4960}}), used to establish a reliability layer on top of DTLS
- DCEP ({{!RFC8832}}), used to map WebRTC data channels to SCTP streams

To make matters worse, both DTLS 1.2 and SCTP have a 4-way handshake, which
means that they each incur 2 round trips, which, when combined with the
individual RTTs from signaling and ICE, bring the total number of RTTs to 6.

On the bright side, DCEP was intentionally designed to not incur an additional
roundtrip, as data can be sent at the same time as the DCEP open message.
However, if the client is the one initiating the data channel, there will be an
extra 0.5 RTT before the server can send data (when DCEP negotiation is used).

This protocol sandwich is discussed in {{Section 5 of ?RFC8831}}, but a full
accounting of all the round trips is not provided. Accordingly, we illustrate
this Matryoshka of handshakes below.

~~~
Offerer                                    Answerer
   |                                            |
   |------------- SDP Offer (actpass) --------->|
   |<-1---------- SDP Answer (active) ----------|
   |                                            |
   |<-2---------- ICE/Connectivity Checks ----->|
   |                                            |
   |<------------ DTLS ClientHello -------------|
   |--3---------- DTLS ServerHello ------------>|
   |<------------ DTLS Finished ----------------|
   |     ANSWERER MEDIA READY (3.5RTT)          |
   |--4---------- DTLS Finished --------------->|
   |      OFFERER MEDIA READY (4 RTT)           |
   |                                            |
   |------------- SCTP INIT ------------------->|
   |<-5---------- SCTP INIT ACK ----------------|
   |------------- SCTP COOKIE ECHO ------------>|
   |<-6---------- SCTP COOKIE ACK --------------|
   |      OFFERER DATA READY (6 RTT)            |
   |                                            |
   |------------- DCEP (Open Channel) --------->|
   |------------- "hello" --------------------->|
   |     ANSWERER DATA READY (6.5 RTT)          |
   |                                            |
   |<------------ DCEP ACK ---------------------|
   |<------------ "world" ----------------------|
~~~

As noted above, DCEP does not require an additional RTT, as the first data
channel payload can be sent concurrently with the DCEP OPEN, per
{{Section 4 of RFC8832}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Discussion

## Role of DTLS-SRTP

It might be tempting to blame this chatty setup sequence on the requirement to
use DTLS-SRTP ({{!RFC5764}}) rather than SDES ({{?RFC4568}}). During the long
discussions that went into mandating use of DTLS-SRTP for securing the WebRTC
media transport, the two extra round trips needed to establish DTLS were
explicitly discussed but ultimately considered to be an acceptable trade off, as
noted in the IETF 87 RTCWEB minutes.

However, DTLS is used not just for media keying, but also as a secure transport
for WebRTC data channels. Therefore, even if SDES had been retained for media
keying, we would still have the same slow setup for data channels.

## Areas for improvement

When considering what we can be done to improve this set of stacked handshakes,
we considered the following:

- Are some aspects of protocol negotiation unnecessary when used in an
  encapsulated context?
- For the aspects that are necessary, can some of them be shifted forward into
  an earlier negotiation?
- Have there been any improvements to underlying protocols that we can benefit
  from?

# Proposed Solution

Looking at the various handshakes through this lens, we can find multiple
potential areas for optimization, which are enumerated below.

## SNAP

SCTP was designed to be a Layer 4 protocol, similar to TCP, and therefore
includes mechanisms to prevent connection hijacking and DDoS attacks. However,
when run atop DTLS ({{?RFC8261}}), as is always the case in WebRTC, these
concerns do not exist, and therefore these mechanisms can be bypassed entirely.
As a direct result, we can dispense with the SCTP auth cookie exchange that
typically adds its own roundtrip.

Moreover, once the auth cookie exchange is removed, all that is left to
negotiate are the various initialization parameters (e.g., the starting sequence
number). However, these parameters are declarative and simply need to be
transferred to the remote party. As a result they can easily be pulled forward
and exchanged via SDP during session negotiation, eliminating the SCTP INIT
message exchange and its associated roundtrip.

This improvement to the SCTP setup process, which saves two roundtrips, is known
as SNAP and is documented in {{!I-D.hancke-tsvwg-snap}}.

## SPED

Turning our attention to DTLS, we can see that in the current ladder diagram
DTLS cannot start until ICE completes, as ICE needs to first find a viable
candidate pair on which to transmit. However, we can extend the STUN messages
used in ICE with a new DATA attribute to allow DTLS data to be piggybacked over
STUN, which enables the first DTLS roundtrip to occur concurrently with the
initial STUN exchange. This mechanism, known as SPED, saves a DTLS roundtrip and
is documented in {{!I-D.hancke-webrtc-sped}}.

## DTLS 1.3

As noted above, DTLS 1.2 requires a 2-RTT setup before data can be sent.
However, DTLS 1.3 ({{!RFC9147}}) supports a 1-RTT setup, as well as a 0-RTT
resumption mechanism. We focus here on the 1-RTT mechanism as it is simpler and
more general, and, thanks to SPED, we can eliminate its cost by piggybacking it
on the underlying ICE exchange.

The net result is that we can save yet another roundtrip by moving to DTLS 1.3's
1-RTT setup.

## Total Savings

With these optimizations, we can completely eliminate almost all of WebRTC's
media roundtrips:

- SNAP: 2 RTT
- SPED: 1 RTT
- DTLS 1.3: 1 RTT

In total, we can reduce the cost to establish WebRTC connections with both media
and data to 1 signaling roundtrip and 1 media roundtrip (2 total roundtrips), as
shown in the sequence diagram below. (Note the use of a=setup:passive in the
answer, discussed further in the section on DTLS roles later in this document.)

~~~
Offerer                                 Answerer
  |                                          |
  |--- SDP Offer (actpass, sped, snap) ----->|
  |<-1 SDP Answer (passive, sped, snap) -----|
  |                                          |
  |--- ICE Check + DTLS ClientHello -------->|
  |<-2 ICE Response + DTLS ServerHello/Fin --|
  |<-- ICE Triggered Check ------------------|
  |    OFFERER MEDIA/DATA READY (2RTT)       |
  |                                          |
  |--- DTLS Finished + DCEP Open + "hello" ->|
  |--- ICE Response ------------------------>|
  |    ANSWERER MEDIA/DATA READY (2.5RTT)    |
  |                                          |
  |<----------- DCEP ACK --------------------|
  |<------------ "world" --------------------|
~~~

However, these improvements are all orthogonal, meaning they can be mixed and
matched if needed if conditions require partial adoption. These mechanisms are
all also backwards compatible, allowing clients to offer a fully optimized
approach and negotiate down individual optimizations if necessary.

## Deployment Experience

In one production deployment, WARP was observed to reduce setup latency by
approximately 4 RTTs at p50 and approximately 12 RTTs at p95. The median
improvement matches the 4-RTT savings expected from the combined protocol
optimizations. The additional improvement at p95 was attributed to the
reduction in handshake packets and, consequently, the reduced chance of packet
loss.

# Best Practices

As these optimizations touch on various parts of the WebRTC stack, several
interactions have complexities that warrant closer analysis.

## ICE

To minimize setup delay, clients MUST follow the {{Section 12.1 of RFC8445}}
guidance to begin sending as soon as they have a valid candidate pair, rather
than waiting for pair selection to complete.

{{RFC8445}} also suggests that both sides use Full ICE implementations for
improved security. Naturally, Full ICE provides significant benefits when used
in a peer-to-peer context, especially in terms of improving hole punching
success as well as consent checking, but this is of nominal value in a
client-server context where the server is directly reachable and consent can be
mostly inferred by the receipt of a valid STUN check. (One edge case, which
requires an on-path attacker, is the replay of a valid STUN check from a forged
source address, i.e., that of a victim.)

In fact, use of Full ICE can make life more complicated for a server, in that
when it receives a DTLS ClientHello tunneled in a STUN Binding Request via SPED,
it cannot respond with an untunnelled ServerHello until it completes its own ICE
check, whereas a Lite ICE server would be able to send immediately, as outlined
in {{Section 12.1.1 of RFC8445}}. In particular, when using post-quantum crypto
suites or other suites with large keys that are split across multiple DTLS
records, it is typically more convenient to send these records as raw DTLS
rather than encapsulating them in STUN using SPED. This is because only a single
DTLS record can be piggybacked on a STUN Response (which would mean the other
records need to be sent via different STUN transactions).

In addition, in the figure above, the answerer has completed DTLS after only 1.5
RTTs, but it cannot send data because it has not yet performed its own ICE
checks and identified a Valid candidate pair. If it were using ICE Lite, this
step would be unnecessary and it would be ready to send at this earlier
juncture, which would save an additional roundtrip.

Therefore, when using WARP on a server, ICE Lite is RECOMMENDED. If true STUN
consent checks are required, a modified version of Full ICE can be employed,
where upon receipt of a STUN request, the associated candidate pair is
immediately added to the Valid List (as is the case in ICE Lite), but the ICE
Agent will still send a check to the remote endpoint to verify consent, with the
candidate pair being removed from the Valid List if the check fails.

Note that Full ICE is still recommended for all client-to-client scenarios. In
addition to its value in establishing peer-to-peer connections and identifying
the optimal candidate pair, the STUN checks sent by a client as part of Full ICE
provide protection against DTLS amplification attacks, and are often quite
useful in demultiplexing traffic at the server.

## DTLS Role and a=setup

The role of DTLS client/server in a WebRTC session is controlled by the a=setup
SDP attribute, which is defined in {{!RFC5763}}, and the default guidance in
WebRTC (from {{!RFC9429}}) is to use a=setup:active in the answer. Depending on
the scenario, additional considerations may apply, as shown in a few scenarios
below.

### Client-server, server is DTLS client (a=setup:active)

In this scenario the answerer is acting as a DTLS client following the default
a=setup:active guidance.

~~~
Offerer                                 Answerer
  |                                          |
  |---- SDP Offer (actpass, sped, snap) ---->|
  |<-1- SDP Answer (active, sped, snap) -----|
  |                                          |
  |--- ICE Check --------------------------->|
  |<-2 ICE Response + DTLS ClientHello ------|
  |<-- ICE Check (Triggered)-----------------|
  |    OFFERER MEDIA/DATA READY (2RTT)       |
  |                                          |
  |--- DTLS SHello/Fin + DCEP + "hello" ---->|
  |--- ICE Response ------------------------>|
  |    ANSWERER MEDIA/DATA READY (2.5RTT)    |
  |                                          |
  |<-- DTLS Finished + DCEP ACK -------------|
  |<------------ "world" --------------------|
~~~

Note that the initial client-server STUN request carries no DTLS payload as the
client (as offerer) is acting as the DTLS server, which reduces the benefit of
SPED slightly. While the overall time to establish media and data is the same as
shown in the a=setup:passive example above (namely, 2RTT before the offerer can
send, and 2.5 RTTs before the answerer can send), we can do slightly better when
we use a=setup:passive with ICE Lite. ICE Lite is still desirable in this
a=setup:active scenario, as noted above, but does not change the overall number
of RTTs.

### Client-server, server is DTLS server (a=setup:passive)

In this scenario the answerer is acting as a DTLS server by specifying
a=setup:passive in the answer, and has also elected to use ICE Lite. The server
has also been made to send the DCEP open message in this scenario for maximum
latency benefit.

~~~
Offerer                                 Answerer
  |                                          |
  |--- SDP Offer (actpass, sped, snap) ----->|
  |<-1 SDP Answer (passive, sped, snap, lite)|
  |                                          |
  |--- ICE Check + DTLS ClientHello -------->|
  |    ANSWERER MEDIA/DATA READY (1.5RTT)    |
  |                                          |
  |<-2 ICE Response + DTLS SHello/Fin + DCEP |
  |    OFFERER MEDIA/DATA READY (2RTT)       |
  |                                          |
  |--- DTLS Finished + DCEP ACK + "hello" -->|
  |<------------ "world" --------------------|
~~~

Because the DTLS ClientHello is piggybacked on the initial client-server STUN
request, the server can complete DTLS processing and start sending encrypted
media and data as soon as that message is received, at only 1.5 RTTs.

### Peer to Peer, remote peer is DTLS client (a=setup:active)

The sequence diagrams above presume that the answerer will not be able to send
STUN checks directly to the offerer, due to the offerer (as a client) being
behind NAT. However, in the event the offerer is in fact reachable on the first
inbound STUN check, this can also lead to the answerer being able to send in
only 1.5 RTTs. (If the offerer is not reachable, this turns into the
client-server, server is DTLS client scenario above.)

~~~
Offerer                                 Answerer
  |                                          |
  |---- SDP Offer (actpass, sped, snap) ---->|
  |<-1- SDP Answer (active, sped, snap) -----|
  |                                          |
  |<-- ICE Check + DTLS ClientHello ---------|
  |--- ICE Res + DTLS SHello/Fin ----------->|
  |    ANSWERER MEDIA/DATA READY (1.5RTT)    |
  |                                          |
  |--- ICE Check (Triggered) --------------->|
  |<-2 ICE Response -------------------------|
  |    OFFERER MEDIA/DATA READY (2RTT)       |
  |                                          |
  |------------ DCEP OPEN + "hello" ---------|
  |<----------- DCEP ACK --------------------|
  |<------------ "world" --------------------|
~~~

In fact, thanks to SPED, the offerer completes DTLS in a single RTT, and has to
wait for its own triggered check to complete before it can actually start to
send. This suggests that SPED might be useful to tunnel application data as well
as handshake messages, which would result in the offerer being ready to send
after just a single RTT. However, sending an actual media stream is likely
beyond the scope of SPED, so we leave this possibility for future exploration.

### Peer to Peer, remote peer is DTLS server (a=setup:passive)

If the remote peer is a DTLS server, the initial STUN check that it sends to the
offerer will not have a piggybacked DTLS message. Regardless, if this STUN check
succeeds, the answerer will also be ready to send in only 1.5 RTT. (If not, this
scenario turns into the client-server, server is DTLS server scenario above.)

~~~
Offerer                                 Answerer
  |                                          |
  |---- SDP Offer (actpass, sped, snap) ---->|
  |<-1- SDP Answer (passive, sped, snap) ----|
  |                                          |
  |<-- ICE Check ----------------------------|
  |--- ICE Response + DTLS ClientHello ----->|
  |--- ICE Check (Triggered) --------------->|
  |    ANSWERER MEDIA/DATA READY (1.5RTT)    |
  |                                          |
  |<-2 ICE Response + DTLS SHello/Fin + DCEP |
  |    OFFERER MEDIA/DATA READY (2RTT)       |
  |                                          |
  |--- DTLS Finished + DCEP ACK + "hello" -->|
  |<------------ "world" --------------------|
~~~

### Conclusions

For the most part, the full benefits of WARP can be realized even when using the
existing default behavior for a=setup, with the offerer needing 2 RTT and the
answerer 2.5 RTT to complete the handshaking. However, in a client-server
scenario where ICE Lite is in use and the server wants to send data first, the
server handshaking latency can be reduced to 1.5 RTT by using a=setup:passive.

## RTO sync

During the analysis of the current WebRTC handshake, it was noted that each
handshake layer has its own retransmission timeout (RTO). ICE specifies a
minimum of 500ms for its timeout in {{Section 14.3 of RFC8445}}; DTLS specifies
a suggested value of 400ms in {{Section 5.8.2 of RFC9147}} (although it notes
that a value from ICE MAY be used), and SCTP specifies a recommended initial
value of 1 second and a minimum of 1 second in {{Section 16 of !RFC9260}}.

These separate RTO values are problematic not just because they lead to
considerably different behavior from the various stack layers in the event of a
packet loss, but also because a loss may occur in a layer that has not yet
completed a roundtrip, leading to a fallback to a (conservative) default value
even if there is actual knowledge of the RTT in another layer.

To address this, when using WARP, implementations SHOULD propagate the observed
RTT values from the ICE layer to the DTLS and SCTP layers, with no fixed minimum
value, as it can be difficult to predict future needs (TODO: should DTLS' 1.5x
offset be recommended?). If the ICE selected candidate pair changes, the RTT
SHOULD be updated accordingly throughout the layers.

One complexity with this approach is that the ICE RTT is not known until the
first request-response cycle is complete, which means that if one of the initial
messages (e.g., a STUN request with a piggybacked DTLS ClientHello) is lost, a
default RTO value will be used and therefore the DTLS implementation will be
slow to request a retransmit. To address this, implementations SHOULD allow
caching of ICE RTT values, which can be reused when connecting to the same
server in future sessions.

## Early Application Data

DTLS 1.3 permits a server to send application data after sending its own
`Finished`, before receiving the client's `Finished` ({{Section 2 of
!RFC8446}}). The client MUST NOT deliver that data until it has received and
validated the server's certificate, certificate verification, and `Finished`,
including the signaled certificate fingerprint. If application data arrives
first due to packet loss or reordering, the client SHOULD buffer it until
validation completes.

## DTLS HelloVerifyRequest

When DTLS is run in a WebRTC context, authenticated ICE connectivity checks
already provide the protection for which a DTLS cookie exchange is normally
used. Consequently, the DTLS 1.2 `HelloVerifyRequest`, or the cookie-bearing
DTLS 1.3 `HelloRetryRequest`, is not needed ({{Section 5.1 of RFC9147}}). This
is existing WebRTC behavior, not an optimization introduced by WARP.

# Security Considerations

See the respective SPED ({{I-D.hancke-webrtc-sped}}) and SNAP
({{I-D.hancke-tsvwg-snap}}) docs for the security considerations associated with
those protocols.

## DTLS 1.3

{{RFC9147}} details how DTLS can be used for amplification attacks and provides
some guidance to defend against this threat, including the use of a cookie to
ensure the client's IP address is legitimate. This mechanism is usually disabled
when DTLS is used in WebRTC, as STUN credentials usually provide protection
against spoofed traffic, and the self-signed ECC certs used in modern WebRTC are
also typically quite small.

# IANA Considerations

This document requires no IANA action.

--- back
