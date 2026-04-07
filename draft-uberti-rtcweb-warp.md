---
title: "WebRTC Abridged Roundtrip Protocol (WARP)"
abbrev: "WARP"
category: info

docname: draft-uberti-rtcweb-warp-latest
submissiontype: IETF
number:
date: 2026-01-07
consensus: true
v: 3
area: RAI
workgroup: RTCWEB Working Group
keyword:
 - WebRTC
 - ICE
 - DTLS
 - SCTP
 - session setup
 - latency
venue:
  group: rtcweb
  type: Working Group
  mail: rtcweb@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/rtcweb/

author:
 -
    fullname: Justin Uberti
    organization: OpenAI
    email: justin@example.com
 -
    fullname: Philipp Hancke
    organization: Google
    email: fippo@example.com

normative:
  RFC2119:
  RFC8174:
  RFC8445:
  RFC5763:
  RFC8832:
  RFC9147:

informative:
  RFC8829:
  RFC8831:
  RFC9000:
  RFC4960:
  RFC6347:
  RFC9260:
  RFC9429:
  I-D.draft-hancke-rtcweb-sped:
  I-D.draft-uberti-rtcweb-snap:
...

--- abstract

WebRTC session setup commonly requires multiple serialized handshakes (signaling, ICE,
DTLS, SCTP, and DCEP), resulting in high connection-establishment latency. This
document describes the WebRTC Abridged Roundtrip Protocol (WARP), a set of compatible
optimizations that reduce typical setup latency from roughly 6 round-trip times (RTTs)
to approximately 2 RTTs in common deployments, by collapsing redundant handshakes,
shifting declarative parameters into SDP, and overlapping DTLS with ICE via STUN
extensions. WARP is designed for incremental, backward-compatible deployment.

--- middle

# Introduction

WebRTC connection setup, as described in {{RFC8829}}, commonly incurs multiple RTTs
before media can be sent and before WebRTC data channels can be used. Historically,
in peer-to-peer calling scenarios, this latency was often masked by human interaction
(e.g., the callee answering), but modern WebRTC use increasingly resembles client-server
session establishment (conferencing, cloud gaming/streaming, and centrally-hosted
real-time services). In these cases, setup latency is directly user-visible.

Reducing RTT count and packet count also improves robustness: fewer serialized steps
reduces sensitivity to delay and loss, and lowers the surface area for retransmission
timer interactions across stacked protocols.

WARP addresses setup latency by:

- removing handshake steps that are redundant when protocols are encapsulated (e.g.,
  SCTP over DTLS);
- moving purely declarative negotiation earlier (into SDP);
- leveraging newer protocols (DTLS 1.3 {{RFC9147}}); and
- overlapping remaining exchanges where feasible.

WARP is a *set* of orthogonal mechanisms. Endpoints can negotiate and deploy each
mechanism independently.

# Conventions and Definitions

{{::boilerplate bcp14-tagged}}

This document uses the following terms:

RTT:
: Round Trip Time between the two endpoints over the selected candidate pair.

Offerer / Answerer:
: As used in SDP offer/answer exchange.

ICE Lite / Full ICE:
: As defined in {{RFC8445}}.

WARP:
: WebRTC Abridged Roundtrip Protocol; the umbrella for the optimizations described here.

SNAP:
: SCTP Negotiation Abridgement Protocol; collapses SCTP association setup in WebRTC.

SPED:
: STUN Payload Extension for DTLS; carries DTLS handshake records within STUN messages.

# Background and Problem Statement

## The "Protocol Sandwich"

A typical WebRTC stack for media + data includes:

- a signaling protocol (often HTTP-based), used for offer/answer exchange;
- ICE {{RFC8445}} to establish a working transport and provide consent;
- DTLS (legacy DTLS 1.2 {{RFC6347}} is still common), to secure the ICE transport;
- SCTP {{RFC4960}} to provide the transport for WebRTC data channels; and
- DCEP {{RFC8832}} to map WebRTC data channels onto SCTP streams (see also {{RFC8831}}).

These protocols were each designed with their own handshake assumptions, and when stacked,
their handshakes are frequently serialized.

## Typical Legacy RTT Accounting

In many implementations, media becomes usable after roughly 4 RTT and data channels after
roughly 6 RTT.

The following ladder diagram illustrates one common serialized setup (illustrative; not
all retransmits and variants are shown):

```
Offerer                                    Answerer
   |                                            |
   |------------- SDP Offer (actpass) --------->|
   |<------------ SDP Answer (active) ----------|  (1 RTT)
   |                                            |
   |<---------- ICE Connectivity Checks -------->|  (2 RTT)
   |                                            |
   |<------------ DTLS ClientHello -------------|
   |------------- DTLS ServerHello ------------>|  (3 RTT)
   |<------------ DTLS Finished ----------------|
   |------------- DTLS Finished --------------->|  (4 RTT)
   |                                            |
   |------------- SCTP INIT ------------------->|
   |<------------ SCTP INIT ACK ----------------|  (5 RTT)
   |------------- SCTP COOKIE ECHO ------------>|
   |<------------ SCTP COOKIE ACK --------------|  (6 RTT)
   |                                            |
   |------------- DCEP OPEN + app data -------->|
   |<------------ DCEP ACK + app data ----------|
```

DCEP was designed to avoid an additional RTT by allowing application data to be sent
concurrently with DCEP OPEN ({{RFC8832}}). However, if the endpoint that first wants to
send application data is not the endpoint that opened the data channel, an extra half-RTT
may be observed before that endpoint can send.

# Discussion

## DTLS-SRTP is Not the Only Cause

It is tempting to attribute setup latency primarily to DTLS-SRTP vs SDES. However, DTLS
is used not only for SRTP keying, but also as the secure transport for WebRTC data
channels; even if SDES were used for media, a DTLS-secured data transport would still
incur significant handshake latency.

## Design Questions

WARP focuses on three questions:

1. Are some aspects of protocol negotiation unnecessary in an encapsulated context?
2. For aspects that remain necessary, can they be shifted into an earlier negotiation?
3. Can improvements in underlying protocols (e.g., DTLS 1.3) be leveraged?

# WARP Overview

WARP combines three optimizations:

- **SNAP** (2 RTT saved): remove SCTP cookie and INIT/INIT-ACK exchange by:
  - relying on DTLS to provide anti-hijacking properties; and
  - moving declarative SCTP parameters into SDP.
- **SPED** (1 RTT saved): piggyback DTLS handshake records onto ICE connectivity checks
  by carrying DTLS records in STUN messages.
- **DTLS 1.3** (1 RTT saved vs DTLS 1.2): use DTLS 1.3’s 1-RTT handshake {{RFC9147}}.

These mechanisms are orthogonal and can be negotiated independently.

## Target Timing

In a common deployment, WARP targets:

- 1 RTT for signaling (offer/answer), and
- 1 RTT for ICE checks (with DTLS 1.3 overlapped via SPED),

for a total of ~2 RTT before the endpoint that initiates the first ICE check is ready
to send encrypted media and data.

The following ladder diagram illustrates one optimized sequence (illustrative; role
choices are discussed in Section 7.2):

```
Offerer                                 Answerer
  |                                          |
  |--- SDP Offer (actpass, sped, snap) ----->|
  |<-- SDP Answer (passive, sped, snap) -----|  (1 RTT)
  |                                          |
  |--- ICE Check + DTLS ClientHello -------->|
  |<-- ICE Resp + DTLS SHello/Fin -----------|  (2 RTT)
  |<-- ICE Triggered Check ------------------|
  |    OFFERER MEDIA/DATA READY              |
  |                                          |
  |--- DTLS Finished + DCEP Open + app ----->|
  |--- ICE Response ------------------------>|
  |    ANSWERER MEDIA/DATA READY (~2.5 RTT)  |
  |                                          |
  |<-- DCEP ACK + app -----------------------|
```

# Protocol Elements

This document defines WARP at the architecture level. Detailed wire formats and SDP
attributes are expected to be defined in the component documents.

## SNAP

SCTP {{RFC4960}} includes cookie exchange and related mechanisms intended for direct use
over IP, including defenses against certain hijacking and DoS scenarios. In WebRTC,
SCTP is always carried over DTLS; DTLS already provides endpoint authentication and
anti-hijacking properties for the SCTP association.

SNAP therefore:

- removes SCTP cookie exchange when SCTP runs over DTLS for WebRTC; and
- moves required SCTP initialization parameters into SDP so that no SCTP association
  handshake is required on the media path.

SNAP is described in more detail in {{I-D.draft-uberti-rtcweb-snap}}.

## SPED

DTLS typically starts after ICE has selected or at least validated a candidate pair.
SPED allows a DTLS handshake record to be carried within STUN transactions used by ICE,
overlapping the first DTLS flight with ICE.

SPED is described in more detail in {{I-D.draft-hancke-rtcweb-sped}}.

## DTLS 1.3

DTLS 1.3 {{RFC9147}} provides a 1-RTT handshake and a clearer acknowledgement and
retransmission model. WARP assumes DTLS 1.3 support as the best baseline for reducing
RTT count; however, SPED and SNAP can still be useful in partial-deployment modes.

# Negotiation and Backward Compatibility

WARP mechanisms are intended to be incrementally deployable:

- An endpoint offering WARP SHOULD indicate support for each mechanism independently
  (e.g., via SDP attributes).
- Endpoints MUST be prepared to fall back per-mechanism if the peer does not support it.
- A WARP-capable endpoint SHOULD interoperate with non-WARP WebRTC endpoints without
  changing application signaling beyond SDP content.

# Best Practices and Deployment Guidance

## ICE

To minimize setup delay, endpoints MUST follow the guidance in {{RFC8445}} to begin
sending as soon as a valid candidate pair is available, rather than waiting for pair
selection to complete.

### ICE Lite on Servers

In many client-server deployments, ICE Lite on the server can reduce complexity and can
allow the server to send earlier in some WARP scenarios.

When using WARP on a server, ICE Lite is RECOMMENDED where operationally appropriate.

If a deployment requires explicit consent checks beyond what ICE Lite provides, an
implementation can employ a hybrid approach where receipt of a valid STUN request causes
a candidate pair to be treated as valid for sending (ICE Lite behavior), but the server
still sends a check to the remote endpoint; if that check fails, the candidate pair is
removed from the valid set.

Full ICE remains RECOMMENDED for clients, including for:

- peer-to-peer connectivity establishment,
- candidate pair optimization, and
- mitigation of amplification risks (see Section 8).

## DTLS Role and SDP a=setup

DTLS client/server roles in SDP are controlled by the `a=setup` attribute {{RFC5763}}.
Some WebRTC guidance (e.g., {{RFC9429}}) historically resulted in the answerer selecting
`a=setup:active` in the answer. In WARP deployments, the choice of roles interacts with
SPED and with whether ICE Lite is used.

WARP-capable endpoints SHOULD choose DTLS roles to maximize overlap of DTLS with the first
ICE check in the expected topology. In particular, in client-server deployments where
the client initiates the first ICE check, servers SHOULD consider using `a=setup:passive`
to allow the client to piggyback a ClientHello in that first check.

The following sub-sections provide illustrative timing variants.

### Client-server, server is DTLS client (answer: active)

In this scenario the answerer acts as DTLS client following `a=setup:active`:

```
Offerer                                 Answerer
  |                                          |
  |--- SDP Offer (actpass, sped, snap) ----->|
  |<-- SDP Answer (active, sped, snap) ------|  (1 RTT)
  |                                          |
  |--- ICE Check --------------------------->|
  |<-- ICE Resp + DTLS ClientHello ----------|  (2 RTT)
  |<-- ICE Check (Triggered)-----------------|
  |    OFFERER MEDIA/DATA READY              |
  |                                          |
  |--- DTLS SHello/Fin + DCEP + app -------->|
  |--- ICE Response ------------------------>|
  |    ANSWERER MEDIA/DATA READY (~2.5 RTT)  |
  |                                          |
  |<-- DTLS Finished + DCEP ACK + app -------|
```

Because the initial client->server STUN request carries no DTLS payload (the client is not
the DTLS client), SPED’s benefit is reduced in this particular direction, although total
setup time can remain similar in some deployments.

### Client-server, server is DTLS server (answer: passive)

In this scenario the answerer acts as DTLS server (`a=setup:passive`) and uses ICE Lite;
the server also initiates DCEP for latency:

```
Offerer (client)                         Answerer (server)
  |                                          |
  |--- SDP Offer (actpass, sped, snap) ----->|
  |<-- SDP Answer (passive, sped, snap, lite)|  (1 RTT)
  |                                          |
  |--- ICE Check + DTLS ClientHello -------->|
  |    ANSWERER MEDIA/DATA READY (~1.5 RTT)  |
  |                                          |
  |<-- ICE Resp + DTLS SHello/Fin + DCEP ----|  (2 RTT)
  |    OFFERER MEDIA/DATA READY              |
  |                                          |
  |--- DTLS Finished + DCEP ACK + app ------>|
  |<-- app ----------------------------------|
```

Because the DTLS ClientHello is piggybacked on the first ICE check, the server can often
complete DTLS processing and begin sending encrypted media/data as soon as that message is
received, subject to its consent and ICE policy.

### Peer-to-peer, remote peer is DTLS client (answer: active)

If the offerer is reachable on the first inbound STUN check, the answerer may be able to
send earlier:

```
Offerer                                 Answerer
  |                                          |
  |--- SDP Offer (actpass, sped, snap) ----->|
  |<-- SDP Answer (active, sped, snap) ------|  (1 RTT)
  |                                          |
  |<-- ICE Check + DTLS ClientHello ---------|
  |--- ICE Resp + DTLS SHello/Fin ---------->|
  |    ANSWERER MEDIA/DATA READY (~1.5 RTT)  |
  |                                          |
  |--- ICE Check (Triggered) --------------->|
  |<-- ICE Response -------------------------|  (2 RTT)
  |    OFFERER MEDIA/DATA READY              |
  |                                          |
  |--- DCEP OPEN + app --------------------->|
  |<-- DCEP ACK + app -----------------------|
```

### Peer-to-peer, remote peer is DTLS server (answer: passive)

If the answerer is DTLS server, its initial STUN check will not carry a DTLS message; if
that check succeeds, the answerer can still become ready at ~1.5 RTT:

```
Offerer                                 Answerer
  |                                          |
  |--- SDP Offer (actpass, sped, snap) ----->|
  |<-- SDP Answer (passive, sped, snap) -----|  (1 RTT)
  |                                          |
  |<-- ICE Check ----------------------------|
  |--- ICE Resp + DTLS ClientHello --------->|
  |--- ICE Check (Triggered) --------------->|
  |    ANSWERER MEDIA/DATA READY (~1.5 RTT)  |
  |                                          |
  |<-- ICE Resp + DTLS SHello/Fin + DCEP ----|  (2 RTT)
  |    OFFERER MEDIA/DATA READY              |
  |                                          |
  |--- DTLS Finished + DCEP ACK + app ------>|
  |<-- app ----------------------------------|
```

## DTLS 1.3: Post-Quantum Cryptography Considerations

Some DTLS 1.3 handshakes (e.g., with post-quantum key material) can exceed a single MTU,
splitting handshake messages across multiple DTLS records and packets. This can complicate
SPED in scenarios where only one DTLS record can be carried per STUN transaction. Servers
using ICE Lite may prefer sending raw DTLS records after validating consent, rather than
encapsulating multiple DTLS records across multiple STUN transactions.

## DTLS 1.2: Handshake Flight Acknowledgements

DTLS 1.2 relies on implicit acknowledgement of handshake flights by receipt of the next
flight. The final server flight is typically acknowledged by application data; without
prompt application data, the server may retransmit the final flight. Implementations that
fall back to DTLS 1.2 SHOULD send an empty DTLS application-data record promptly after
receiving the server’s final flight to avoid unnecessary retransmissions.

## Retransmission Timeout (RTO) Synchronization

ICE, DTLS, and SCTP each specify their own retransmission behavior with different defaults
and minimums (e.g., ICE’s timers in {{RFC8445}}, DTLS guidance in {{RFC9147}}, and SCTP’s
initial RTO recommendations in {{RFC9260}}). With WARP collapsing and overlapping handshakes,
these disparate timers can produce undesirable behavior when loss occurs early.

When using WARP, implementations SHOULD:

- propagate observed ICE RTT estimates into DTLS and SCTP timers where feasible;
- update these estimates if the selected candidate pair changes; and
- permit caching of RTT estimates per server to improve loss recovery for subsequent sessions.

# Conclusions

For many deployments, the full benefits of WARP can be realized while retaining default
WebRTC role behavior, achieving ~2 RTT readiness for the offerer and ~2.5 RTT readiness
for the answerer. In client-server scenarios with ICE Lite and where the server is willing
to act as DTLS server (`a=setup:passive`) and send first, server readiness can often be
reduced to ~1.5 RTT.

# Security Considerations

Security considerations for the underlying mechanisms are described in their component
documents ({{I-D.draft-hancke-rtcweb-sped}} and {{I-D.draft-uberti-rtcweb-snap}}).

DTLS 1.3 guidance on amplification and related threats is discussed in {{RFC9147}}.
In ICE Lite deployments, an attacker with valid ICE credentials could potentially induce
a server to send a larger DTLS handshake flight (e.g., with post-quantum key material) to
a spoofed address. Deployments concerned about this risk can:

- require Full ICE behavior before sending DTLS flights; and/or
- use DTLS cookies per {{RFC9147}} where compatible with WebRTC’s authentication model.

# IANA Considerations

This document has no IANA actions.

# Future Work

## 0-RTT Resumption

DTLS 1.3 supports 0-RTT resumption. If a prior session exists, it may be possible to reduce
media-path RTTs further by sending early data alongside the first ICE check, reaching a
single signaling RTT and (effectively) zero additional media RTTs in the best case.

Illustrative timing:

```
Client                                    Server
  |                                          |
  |--- SDP Offer (actpass, sped, snap) ----->|
  |<-- SDP Answer (passive, sped, snap, lite)|  (1 RTT)
  |   CLIENT MEDIA/DATA READY                |
  |--- ICE Check + DTLS CHello + DCEP + app >|
  |   SERVER MEDIA/DATA READY (~1.5 RTT)     |
  |<-- ICE Resp + DTLS SHello/Fin + DCEP ACK-|
  |--- EndOfEarlyData + DTLS Finished ------>|
```

This approach requires careful treatment of key changes between early-data and 1-RTT keys,
particularly for DTLS-SRTP key derivation and media decoding.

## Application Data over SPED

SPED currently focuses on carrying DTLS handshake records. Carrying selected application
control messages (e.g., DCEP OPEN) may further reduce perceived latency in some cases.
Extending SPED to carry full media is likely out of scope.

--- back

# Appendix A: Implementation Status
{:numbered="false"}

This appendix is non-normative and records example implementation status:

- Piggyback DTLS on ICE to save ~1 RTT (SPED): implementation in progress.
- Use DTLS 1.3 to save ~1 RTT (vs DTLS 1.2): experimentation ongoing; may require servers
  to select `a=setup:passive` for best results.
- SNAP: collapse SCTP handshaking into SDP to save ~2 RTT: design and specification work
  ongoing.

# Appendix B: Notes
{:numbered="false"}

- Some observed traces show DTLS 1.3 reducing answerer completion time by ~1 RTT, while
  first RTP emission may still depend on role choices (`a=setup`) and ICE behavior.
- Further tuning may require exposing DTLS role selection via application APIs rather than
  relying on SDP-only role negotiation.

# Acknowledgments
{:numbered="false"}

Thanks to the WebRTC and RTCWEB communities for prior discussions on setup latency and
handshake optimization.
