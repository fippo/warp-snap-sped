---
title: "SCTP Negotiation Acceleration Protocol"
abbrev: "SNAP"
category: info

docname: draft-todo-tsvwg-snap
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: AREA
workgroup: Network Working Group
keyword:
 - sctp
 - webrtc
 - sdp
venue:
  group: WG
  type: Working Group
  mail: tsvwg@ietf.org
  arch: https://example.com/WG
  github: fippo/warp-snap-sped
  latest: https://fippo.github.io/warp-snap-sped/draft-todo-webrtc-snap-latest.html

author:
 -
    fullname: Philipp Hancke
    organization: Meta Platforms Inc
    email: philipp.hancke@googlemail.com

normative:

informative:

--- abstract

{{?RFC8831}} defines WebRTC Data Channels that allow the transport of arbitrary non-media data over a WebRTC PeerConnection.
This uses SCTP {{!RFC9260}} and a DTLS encapsulation of SCTP packets {{RFC8261}}.

Using the SDP negotiation procedure described in this document allows skipping the exchange of the SCTP INIT, INIT ACK,
COOKIE ECHO and COOKIE ACK chunks and reduces the time to open a datachannel by up to two round trip times.

--- middle

# Introduction

SCTP {{RFC9260}} establishes its associations using a four-way handshake, which primarily serves to protect against
half-open (SYN-flood) attacks. For WebRTC, SCTP runs encapsulated within DTLS {{?RFC8261}} which establishes a secure,
encrypted channel between the peers, which prevents half-open attacks.

The full connection setup between two entities using WebRTC, consisting of the exchange of SDP over the signaling channel,
ICE and DTLS establishment and the SCTP handshake, is shown below (with annotations for the number of full round trips):

~~~
Client                                       Server
   |                                            |
   |------------- SDP Offer ------------------->|
   |<-1---------- SDP Answer -------------------|
   |                                            |
   |<-2---------- ICE/Connectivity Checks ----->|
   |                                            |
   |<------------ DTLS ClientHello -------------|
   |--3---------- DTLS ServerHello ------------>|
   |<------------ DTLS Finished ----------------|
   |--4---------- DTLS Finished --------------->|
   |                                            |
   |------------- SCTP INIT ------------------->|
   |<-5---------- SCTP INIT ACK ----------------|
   |------------- SCTP COOKIE ECHO ------------>|
   |<-6---------- SCTP COOKIE ACK --------------|
   |                                            |
   |------------- DCEP (Open Channel) --------->|
   |------------- "hello world"---------------->|
   |<------------ DCEP ACK ---------------------|
   |                                            |
~~~

Note that SCTP typically does a "simultaneous open", i.e. both sides take the active role,
although only one direction is shown here for simplicity.

Note: {{RFC9260}} allows the packets with the COOKIE ECHO and COOKIE ACK to carry data.
This means the second RTT is not strictly necessary but was observed in practice.

When the SCTP INIT chunk is signalled in the SDP, using the "sctp-init" attribute described
in this document, this flow can be reduced to the following exchange:

~~~
Client                                       Server
   |                                            |
   |------------- SDP Offer (sctp-init)-------->|
   |<-1---------- SDP Answer (sctp-init)--------|
   |                                            |
   |<-2---------- ICE/Connectivity Checks ----->|
   |                                            |
   |<------------ DTLS ClientHello -------------|
   |--3---------- DTLS ServerHello ------------>|
   |<------------ DTLS Finished ----------------|
   |--4---------- DTLS Finished --------------->|
   |                                            |
   |------------- DCEP (Open Channel) --------->|
   |------------- "hello world"---------------->|
   |<------------ DCEP ACK ---------------------|
   |                                            |
~~~

Further reductions of the startup sequence are described in WARP-TODO

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Terminology

**SCTP INIT**: the SCTP INIT chunk as described in {{Section 3.3.2 of RFC9260}}.

**SCTP INIT ACK**: the SCTP INIT ACK chunk as described in {{Section 3.3.3 of RFC9260}}.

**SCTP COOKIE ECHO**: the SCTP COOKIE ECHO chunk as described in {{Section 3.3.11 of RFC9260}}.

**SCTP COOKIE ACK**: the SCTP COOKIE ACK chunk as described in {{Section 3.3.12 of RFC9260}}.

**RTT**: Round-Trip Time

# SDP sctp-init attribute

## General

This section defines a new SDP media-level attribute, "sctp-init". The attribute can be associated with
an SDP media description ("m=" line) with a "UDP/DTLS/SCTP" protocol identifier (defined in RFC 8841)
and the &lt;fmt&gt; parameter value of 'webrtc-datachannel' (defined in RFC 8832).

An example follows, see section X.X for the full SDP exchange:

~~~
a=sctp-init:AQAAHols3R0AUAAA/////+B5ZR3AAAAEgAgABoLA
~~~

This is equivalent to the following SCTP INIT CHUNK:

~~~
0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Type = 1    |Chunk Flags = 0|      Chunk Length = 30        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Initiate Tag = 0x896cdd1d                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      Advertised Receiver Window Credit (a_rwnd) = 0x00500000  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| # of Outbound Streams = 65535 | # of Inbound Streams = 65535  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Initial TSN = 0xe079651d                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
/              Optional/Variable-Length Parameters              /
\     Forward TSN       |  Supported extensions=RECONFIG(0x82), \
/                       |          FORWARD_TSN(0xc0)            /
\ 0xc0, 0x00, 0x00, 0x04| 0x80, 0x08, 0x00, 0x06, 0x82, 0xc0    \
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

## Syntax

**Attribute name**: sctp-init

**Type of attribute**: media

**Mux category**: CAUTION

**Subject to charset**: No

**Purpose**: allows the SDP to carry the information contained in a SCTP INIT chunk

**Appropriate values**: a base64 encoded blob.

**Syntax**: sctp-init-value = base64 ; base64 defined in RFC 4566

# SDP Offer/Answer Procedures

## General

This section defines how the "sctp-init" attribute can be negotiated using the SDP offer/answer mechanism.

## Generating the Initial SDP Offer

If the offering endpoint has created a WebRTC datachannel, it MUST let the SCTP implementation generate
a serialized INIT CHUNK that it would normally send over the network as a series of bytes.
This message is then base64-encoded and added to the data "m=" section as an "a=sctp-init" line.

## Answerer Processing of the SDP Offer

If the answering endpoint negotiates a data "m=" section, it will parse the "a=sctp-init" line from the data "m=" section, if present.

If the data is not properly base64 encoded the answerer MUST silently ignore the "a=sctp-init" line and not generate one in response.

The answering endpoint then informs its SCTP implementation of the series of bytes. The SCTP implementation is responsible for
validating the format of the SCTP init chunk.

If the series of bytes is not a valid SCTP init chunk, the answering endpoint MUST silently ignore the "a=sctp-init" line and not generate one in response.

## Generating the SDP Answer

An endpoint not supporting the extension described in this document MUST NOT include a "a=sctp-init" attribute in its response.

If the answering endpoint negotiated a data "m=" section and detected a valid "a=sctp-init" line in the offer, as described above,
it MUST let its SCTP implementation generate a serialized INIT CHUNK that it would normally send over the network as a series of bytes.

This message is base64-encoded and added to the data "m=" section as a "a=sctp-init" line.

## Offerer Processing of the SDP Answer

If the answer negotiated a data "m=" section, the offering endpoint parses the "a=sctp-init" line, if present, from the data "m=" section.

If the data is not properly base64 encoded the offer MUST silently ignore the "a=sctp-init" line.

The offering endpoint then informs its SCTP implementation of the series of bytes.
The SCTP implementation is responsible for validating the format of the SCTP init chunk.
If the series of bytes is not a valid SCTP init chunk, the offering endpoint MUST silently ignore the "a=sctp-init" line.

## Modifying the Session

Subsequent offers and answers MUST include the "a=sctp-init" line in the negotiated SDP with the same value as in the
initial negotiation.

Remote offers MAY negotiate a new "a=sctp-init" line in conjunction with either

* a new SCTP association as described in {{Section 9.3 of ?RFC8841}}
* or a new DTLS association as described in {{Section 5.5 of ?RFC8842}}.

# SCTP considerations

The creation of the "sctp-init" attribute SHOULD NOT change the state of the SCTP association to ASSOCIATE as described
in {Section 4 of RFC9260} and SHOULD NOT start the T1-init timer.

Processing of the "sctp-init" attribute from the remote side SHOULD NOT change the state of the SCTP association and should not start any timer.

When the "sctp-init" attribute has been negotiated, both endpoints SHOULD consider the SCTP association to be in the ESTABLISHED state,
as described in {Section 4 of RFC9260}, once the DTLS handshake finishes. The steps A-E described in {Section 5.1 of RFC9260} can be skipped.

# Example negotiation {#example}

The offering client generates a SCTP INIT chunk with its supported parameters:

~~~
0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Type = 1    |Chunk Flags = 0|      Chunk Length = 30        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Initiate Tag = 0x896cdd1d                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      Advertised Receiver Window Credit (a_rwnd) = 0x00500000  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| # of Outbound Streams = 65535 | # of Inbound Streams = 65535  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Initial TSN = 0xe079651d                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
/              Optional/Variable-Length Parameters              /
\     Forward TSN       |  Supported extensions=RECONFIG(0x82), \
/                       |          FORWARD_TSN(0xc0)            /
\ 0xc0, 0x00, 0x00, 0x04| 0x80, 0x08, 0x00, 0x06, 0x82, 0xc0    \
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

This is encoded in the offer SDP as follows:

~~~
v=0
o=- 8389853828849686268 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0
m=application 9 UDP/DTLS/SCTP webrtc-datachannel
c=IN IP4 0.0.0.0
a=ice-ufrag:UgEn
a=ice-pwd:f/+ugRILrIUlAkSmkStnZb/h
a=ice-options:trickle
a=fingerprint:sha-256
              6A:15:F0:08:9C:55:51:CD:55:27:BD:0D:FB:14:DD:41:
              F6:8C:82:9F:CA:AD:DA:E7:04:61:6F:A9:FF:99:2D:7A
a=setup:actpass
a=mid:0
a=sctp-port:5000
a=max-message-size:262144
a=sctp-init:AQAAHols3R0AUAAA/////+B5ZR3AAAAEgAgABoLA
~~~

In response, the answering client generates its SCTP INIT chunk with its supported parameters:

~~~
0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Type = 1    |Chunk Flags = 0|      Chunk Length = 30        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Initiate Tag = 0x5fb37474                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      Advertised Receiver Window Credit (a_rwnd) = 0x00500000  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| # of Outbound Streams = 65535 | # of Inbound Streams = 65535  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Initial TSN = 0xa1aadc74                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
/              Optional/Variable-Length Parameters              /
\     Forward TSN       |  Supported extensions=RECONFIG(0x82), \
/                       |          FORWARD_TSN(0xc0)            /
\ 0xc0, 0x00, 0x00, 0x04| 0x80, 0x08, 0x00, 0x06, 0x82, 0xc0    \
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

This yields an answer SDP shown below:

~~~
v=0
o=- 6156433258406980035 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0
m=application 9 UDP/DTLS/SCTP webrtc-datachannel
c=IN IP4 0.0.0.0
a=ice-ufrag:thzM
a=ice-pwd:YT278C3KYFmsedR5+OzE9OQY
a=ice-options:trickle
a=fingerprint:sha-256
              AC:48:A1:C4:78:6D:6D:46:63:BB:7D:70:E4:B8:5D:C4:
              E8:6E:66:40:8C:81:31:D3:9C:34:32:9A:C3:6B:BB:AE
a=setup:active
a=mid:0
a=sctp-port:5000
a=max-message-size:262144
a=sctp-init:AQAAHl+zdHQAUAAA/////6Gq3HTAAAAEgAgABoLA
~~~

# Security Considerations

SNAP removes SCTP's anti-amplification mechanism in order to accelerate connection startup.
However, when SCTP runs atop DTLS as specified in {RFC8261}, any attempt to send junk traffic over SCTP will fail
as it will not be properly encrypted. Therefore SNAP does not add any new amplification risk.

Exposing the content of the SCTP INIT, in particular the “Initiate Tag”, in the SDP does not introduce new security
concerns since running SCTP atop DTLS protects against the off-path attacks described in {Section 5.3.1 of RFC9260}.

Exposing the content of the SCTP init tag to the application layer, e.g. Javascript applications or the signaling
channel in the case of WebRTC does allow these to add or remove variable length parameters. Since these parameters
are optional and used for feature negotiation, removing one is equivalent to the remote side not supporting the
parameter and does not introduce a new security risk. Adding a parameter that is not supported may cause the remote
side to send those but this is equivalent to receiving a packet for a feature that was not negotiated and hence does
not introduce a new security concern.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

Lennart Grahl
