---
title: "ICE Renomination: Dynamically selecting ICE candidate pairs"
abbrev: ICE Renomination
docname: draft-thatcher-tsvwg-renomination-latest
submissiontype: IETF
number:
date: 2026-04-06
category: std
ipr: trust200902
v: 3
area: TSV
workgroup: Transport and Services Working Group
venue:
  group: tsvwg
  type: Working Group
  mail: tsvwg@ietf.org
  arch: https://datatracker.ietf.org/wg/tsvwg/

stand_alone: true
pi:
  toc: yes
  symrefs: yes
  sortrefs: yes

author:
  -
    initials: P.
    surname: Thatcher
    fullname: Peter Thatcher
    organization: Google
    email: pthatcher@google.com
  -
    initials: H.
    surname: Zhang
    fullname: Honghai Zhang
    email: honghaiz@google.com
  -
    initials: T.
    surname: Brandstetter
    fullname: Taylor Brandstetter
    organization: Google
    email: deadbeef@google.com
  -
    initials: J.
    surname: Uberti
    fullname: Justin Uberti
    organization: OpenAI
    email: justin@uberti.name

normative:
  RFC2119:
  RFC8174:
  RFC7675:
  RFC8489:
  RFC8445:

informative:
  RFC8838:
...

--- abstract

This document describes an extension to the Interactive Connectivity
Establishment (ICE) that enables the controlling ICE agent to dynamically
change its selected candidate pair over time as network conditions change, and
notify the controlled side accordingly.

--- middle

# Introduction

ICE {{RFC8445}} agents take either the controlling or controlled role.
During the ICE establishment process, the controlling and controlled agents
communicate, using either regular or aggressive nomination, to agree upon
which candidate pair they will use to exchange traffic. However, once ICE is
established, there is no defined procedure for changing the selected pair
without using an ICE restart.

While the controlling ICE agent could unilaterally select a given candidate pair
at any time, it has no straightforward way of authoritatively telling the
controlled side what pair it has selected. This greatly limits the controlling
side's options.

For example, if an ICE agent selects and nominates a candidate pair over a
cellular network, and then later connects to a Wi-Fi network and trickles ICE
candidates for the Wi-Fi network, it may wish to select and nominate a
Wi-Fi candidate pair. If soon thereafter the Wi-Fi network disconnects,
the ICE agent may wish to select and nominate the cellular candidate pair
again. These sorts of mid-session changes to the selected pair are not
possible under the current ICE procedures without an ICE restart.

This document continues the work from the earlier individual Internet-Draft
{{?I-D.thatcher-ice-renomination}}.

# Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Design

This document defines an extension to the ICE mechanism called renomination.
When active, renomination allows the controlling side to dynamically change
the selected pair at any time, and notify the controlled side which pair
it should use.

This allows switching between known pairs, or, as in the scenario outlined
above, if a new interface becomes available, the controlling side can switch
to a pair where the local candidate is on the new interface.

While this sort of dynamic change could be achieved with an ICE restart,
a restart is a somewhat heavy operation that involves signaling new ICE
credentials, gathering new candidates on both sides, and running the
connectivity check mechanism for a whole new set of candidate pairs.
In particular, the signaling requirement can be especially problematic if
the device is in the process of switching between networks.

In contrast, the renomination mechanism only requires the exchange of a single
ICE connectivity check.

As with regular nomination, this specification does not prescribe a specific
algorithm for the controlling agent to select a candidate pair.

## Behavior

When renomination is active, the controlled side follows a process similar to
regular nomination, except that more than one nomination request carrying
USE-CANDIDATE can be sent during the lifetime of the ICE session, and the last
valid nomination wins. To avoid ambiguity if requests are received out
of order, a new NOMINATION attribute ({{attribute}}) MUST be included in every
connectivity check that contains USE-CANDIDATE. The value of the NOMINATION
attribute is a monotonically increasing sequence number chosen by the
controlling agent. A nomination request whose NOMINATION value is less than or
equal to the highest previously accepted NOMINATION value for the same ICE
generation and component MUST be ignored.

In order to ensure that candidate pairs other than the selected pair remain usable,
both sides need to keep their candidates alive (e.g., by refreshing an associated
TURN allocation). In addition, the controlling side MUST periodically
send ICE consent checks on any candidate pair it wants to keep available for
future nomination, using the consent freshness mechanism defined in {{RFC7675}}.
The controlled side MUST issue a success response to these
checks for all candidate pairs it intends to keep available for future
nomination. However, either agent MAY limit the number and type of retained candidate
pairs according to local policy.

This allows the controlling side to maintain a set of valid candidate pairs,
any one of which it can nominate at any time.

## ICE Option {#option}

Naturally, this mechanism requires opt-in from both ICE agents. To accomplish this,
we define a new ICE option called "renomination2", where the "2" is added to
avoid ambiguities with earlier versions of this draft.

If one side signals "renomination2" and the other does not, renomination is
not in use for that ICE session. In that case, the controlling side MUST use
standard ICE nomination procedures and MUST NOT send more than one effective
nomination for a given component.

Note that the offering side (typically, the controlling side, but not always,
e.g., if the offerer is ICE Lite) MAY receive ICE checks prior to receiving
the SDP answer that definitively indicates support for renomination. In this case,
the ICE checks might or might not include the NOMINATION attribute defined below.
Accordingly, when offering "renomination2" and operating in the controlled role,
the offerer MUST be prepared to work with an answerer that is using either
renomination or regular nomination.

## "Nomination" attribute {#attribute}

To deal with out-of-order delivery of nominations, this document defines a new
STUN attribute {{RFC8489}}, NOMINATION, which carries a 32-bit unsigned integer
in network byte order. This attribute is comprehension-required, as it is only
used when both sides have explicitly negotiated support via the "renomination2"
ICE option.

When the "renomination2" ICE option has been negotiated, the controlling side
MUST include a NOMINATION attribute in every connectivity check that includes
USE-CANDIDATE. The first such request for an ICE generation (i.e., for a given
set of ICE credentials) and component MAY use any value.
Subsequent nomination requests for that same ICE generation and component MUST
use a strictly greater value than any prior nomination request sent for that
component. NOMINATION values for different components are not compared.

The controlled side MUST compare the received NOMINATION value against the
highest previously accepted NOMINATION value for the current ICE generation
and component. If the received value is greater, the nomination is accepted
and the nominated pair is updated for that component. If the received value
is less than or equal to the stored value, the request MUST be ignored for
the purpose of updating the nominated pair. If the NOMINATION attribute is
absent, the request MUST be similarly ignored, unless the scenario noted in
{{option}} applies, where the SDP answer has not yet arrived and it is
currently unclear to the offerer whether regular nomination or renomination
is in use. In this situation, a connectivity check with USE-CANDIDATE but
no NOMINATION attribute MUST be processed as a regular nomination.

If the ICE credentials are changed, i.e., by an ICE restart, this constitutes
a new ICE generation, and the first-request rules above apply for each component.

# Interaction with ICE Lite

Renomination is compatible with both full ICE and ICE Lite. When using
ICE Lite, the guidance in {{RFC8445}} for the controlling side to select the
candidate pair with the highest priority is superseded by the mechanism
described above.

# IANA Considerations

This document requests that IANA make the following registrations:

## ICE Option Registration

IANA is requested to add the following value to the
"Interactive Connectivity Establishment (ICE) Options" registry:

ICE Option name: `renomination2`
Description: The ICE option indicates that the ICE agent supports renomination.
Reference: RFC XXXX

## STUN Attribute Registration

IANA is requested to add the following value to the
"STUN Attributes" registry:

Attribute Name: `NOMINATION`
Value: `0x0030`
Reference: RFC XXXX

Note: Some implementations have already used `0x0030` for this attribute, and
IANA has confirmed that this codepoint is available for registration by this
specification.

# Security Considerations

This mechanism extends the period during which ICE nomination can occur.
Although it does not introduce any significant new protocol surface, it will
result in continued connectivity checks on non-selected candidate pairs.

The existing consent freshness mechanism defined in {{RFC7675}} is used to
ensure that the controlling side has permission to continue sending checks to
these non-selected candidate pairs.

However, keeping certain candidates alive may incur higher power and network usage
(e.g., cellular or TURN candidates). To control this, either side MAY
discard candidates that it considers non-essential.

This extension does not change the existing requirements for authenticating
STUN checks or protecting application or media traffic sent over candidate
pairs.

# Acknowledgements

This document overlaps with similar ideas in {{?I-D.uberti-mmusic-nombis}} and
{{?I-D.oreland-dispatch-ice-nicer}}.

--- back
