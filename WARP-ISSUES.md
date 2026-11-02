# WARP publication issues

This document tracks issues in the WARP overview, not issues in SPED or SNAP.
Protocol details, component-specific security considerations, signaling, wire
formats, and IANA registrations belong in their respective specifications.

## P0: Blocks the initial draft

None currently identified. The original overview's sections, prose, and
message-flow diagrams are preserved. The two WARP-level clarifications needed
for the initial draft are included in Best Practices:

- The client buffers early server application data until the server
  certificate, fingerprint, and `Finished` have been validated.
- Authenticated WebRTC ICE connectivity checks make a separate DTLS
  `HelloVerifyRequest` or cookie-bearing `HelloRetryRequest` unnecessary.

## P1: Address during working group review

1. **Baseline and RTT accounting.** The original says a conventional SCTP
   handshake costs two round trips. RFC 9260 allows DATA to be bundled with
   COOKIE ECHO and COOKIE ACK, so qualify the six-to-two-RTT comparison as a
   representative implementation baseline rather than an unavoidable protocol
   minimum. Distinguish signaling RTT from media-path RTT.

2. **Full ICE valid-list proposal.** The original suggests immediately adding a
   candidate pair to the Valid List before the Full ICE connectivity check
   succeeds. Identify that behavior as a proposed optimization, establish
   whether RFC 8445 permits it, or remove it. Keep the existing recommendation
   of ICE Lite for reachable servers and Full ICE for client-to-client sessions.

3. **Retransmission guidance.** The original recommends sharing RTT estimates
   with DTLS and SCTP "with no fixed minimum value" and contains an unresolved
   question about the DTLS 1.5x multiplier. Reconcile that text with RFC 9147's
   1.5x guidance and RFC 9260's recommended SCTP minimum without redefining
   either protocol.

4. **Role and readiness diagrams.** Confirm the assumptions behind the `active`
   and `passive` DTLS-role examples and the 1.5-RTT ready markers, including
   peer authentication, loss or reordering, and handshake flights that span
   multiple packets. Keep SPED packet-handling rules in the SPED draft.

5. **Partial deployment and restarts.** Clarify at the overview level how the
   independently useful mechanisms compose when an optimization is unavailable
   or when ICE, DTLS, or SCTP restarts. Do not define new SPED or SNAP state
   machines.

## P2: Editorial and deployment follow-up

1. Publish supporting measurement methodology if the single-deployment p50 and
   p95 observations are retained or expanded.

2. Decide whether to update historical WebRTC and SCTP citations from RFC 8829
   and RFC 4960 to RFC 9429 and RFC 9260 while preserving the original
   explanation.

3. Review terminology, case, diagram alignment, and references to the IETF 87
   RTCWEB minutes.
