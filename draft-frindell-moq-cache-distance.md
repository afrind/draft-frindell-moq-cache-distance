---
title: "Cache Distance Property for MOQT"
abbrev: "moq-cache-distance"
category: std

docname: draft-frindell-moq-cache-distance-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Media Over QUIC"
keyword:
 - media over quic
 - cache
 - relay
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "afrind/draft-frindell-moq-cache-distance"
  latest: "https://afrind.github.io/draft-frindell-moq-cache-distance/draft-frindell-moq-cache-distance.html"

smart_quotes: no

author:
 -
    ins: A. Frindell
    fullname: Alan Frindell
    organization: Meta
    email: afrind@meta.com

normative:
  MOQT: I-D.ietf-moq-transport

informative:


--- abstract

This document defines CACHE_DISTANCE, a MOQT Object Property that a Relay
adds to the Objects it sends on a FETCH stream to report how far away the
cache that supplied each Object was.  The value composes across a chain of
Relays, and is carried only where it changes, so a Fetch served by a mix of
caches costs one Property per transition rather than one per Object.


--- middle

# Introduction {#intro}

A subscriber issuing a Fetch to a Media over QUIC Transport (MOQT) {{MOQT}}
Relay cannot tell how the request was served.  The Objects may have come from
the Relay's own cache, from a cache further upstream, or from the Original
Publisher, and the difference matters for operating a Relay deployment:
it is the basis for cache hit rate, for judging whether a Relay is well
positioned for a Track, and for measuring load reaching the origin.

This document defines a single Object Property that reports that information
per Object.  Because a Fetch is commonly satisfied from more than one source
-- the leading Objects from a nearby cache and the remainder from upstream --
the value is carried only on the Objects where it changes and applies to the
Objects that follow, so the cost is proportional to the number of transitions
rather than to the number of Objects.  The value also composes across a chain
of Relays with a local rule at each hop.

This document defines only the Property.  It does not define cache admission,
eviction, or placement policy, nor any required receiver behavior.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the terms Track, Object, Group, Relay, and Original
Publisher as defined in {{MOQT}}.


# Cache Distance {#cache-distance}

CACHE_DISTANCE is an Object Property added by a Relay to the Objects it
sends on a FETCH stream, giving the number of hops between the receiver
and the cache that supplied the Object.  The value is a variable-length
integer.  A value of 1 means the Relay serving the Fetch supplied the
Object from its own cache; a value of n greater than 1 means a cache
n-1 hops upstream of that Relay supplied it.  A value of 0 means the
Object is not attributed to any cache: it came from the Original
Publisher, or the path is not fully instrumented.  Because the property
describes how a request was served rather than the content itself, it
applies only to Objects sent on a FETCH stream ({{MOQT}}); a receiver
that sees it on a Subgroup stream or a datagram MUST ignore it.

| Value | Meaning |
|------:|:--------|
| 0 | Not attributed to a cache: from the Original Publisher, or the path is not fully instrumented |
| 1 | Supplied by the Relay serving this Fetch |
| n | Supplied by a cache n-1 hops upstream of that Relay |

## Stream State {#cache-distance-state}

The property is stateful for the duration of a FETCH stream.  Its value
applies to the Object that carries it and to every subsequent Object on
the same stream until another CACHE_DISTANCE appears.  The state
initializes to 0, so a Relay that does not implement this property sends
nothing and the entire Fetch correctly reads as unattributed.  A Relay
that does implement it MUST send the property on an Object whose value
differs from the current state, and MUST NOT send a value equal to the
current state.  Objects with a status other than Normal cannot carry
properties ({{MOQT}}) and do not disturb the state, which carries across
them.  The state does not survive the end of the stream: a Fetch resumed
as a new request begins again at 0.

## Forwarding {#cache-distance-forwarding}

A Relay computes the value it sends for each Object as follows:

* If the Relay supplied the Object from its own cache, it sends 1.

* If the Relay obtained the Object from upstream to satisfy this Fetch
  and the upstream value for that Object is nonzero, it sends that
  value plus one.  If the result would not fit in a variable-length
  integer, it sends 0.

* If the Relay obtained the Object from upstream and the upstream value
  is 0, or upstream sent no CACHE_DISTANCE, it sends 0.

A Relay MUST NOT forward received CACHE_DISTANCE properties unchanged.
It recovers the incoming value for each Object from the upstream
stream's state, applies the rules above, and re-encodes its own
transitions.  The encoding remains sparse under composition: a
transition appears only where the upstream value changes or where the
Relay's own hit pattern changes, so a Relay that serves a contiguous
run from its own cache collapses that run to a single property however
it was marked upstream.

For example, consider a Fetch for Objects 1 through 7 over the path
OP -> R2 -> R1 -> subscriber, where R1 holds Objects 1 through 3 in its
cache and R2 holds Objects 4 and 5.  R2 sends CACHE_DISTANCE=1 on Object
4 and 0 on Object 6, two properties for the four Objects it supplies.
R1 sends 1 on Object 1, 2 on Object 4, and 0 on Object 6.  The
subscriber resolves the sequence 1, 1, 1, 2, 2, 0, 0 from three
properties on the wire.

## Processing Rules {#cache-distance-processing}

CACHE_DISTANCE is hop-by-hop.  It is generated by the Relay serving the
Fetch, describes only how that Relay served the request, and is not a
property of the content.  An Original Publisher MUST NOT set it, and it
MUST NOT be placed in Immutable Properties ({{MOQT}}).  A cache MUST NOT
store it, and the Relay behavior above is permitted notwithstanding the
general rule in {{MOQT}} that a Relay supporting a Property neither
modifies it nor omits it from the cache.

The value is advisory.  A receiver MUST NOT rely on it for correctness,
and a Relay MAY report a value smaller than the true distance,
including reporting only 0 and 1, where the depth of its topology is
sensitive.


# IANA Considerations {#iana}

This document registers the following entry in the "MOQ Properties"
registry established by {{MOQT}}.

| Type | Name | Scope | Specification |
|-----:|:-----|:------|:--------------|
| 0x1F0A | CACHE_DISTANCE | Object | This document, {{cache-distance}} |

The value is a variable-length integer giving the distance in hops to
the cache that supplied the Object.  The code point is provisional for
interoperability testing; the final value is to be assigned by IANA.  A
two-byte code point is used because the property is sent per Object.


# Security Considerations {#security}

CACHE_DISTANCE reveals how a Relay served a request and, on a fully
instrumented path, the depth of the serving hierarchy.  Where that is
sensitive, a Relay can flatten the values it reports as described in
{{cache-distance-processing}}.  An endpoint MUST NOT treat the value as
authenticated: it is set by the adjacent Relay, is not covered by
end-to-end Object authentication, and a misbehaving Relay can report any
value.  Because the property is never cached, a false value cannot
persist beyond the Fetch that carried it.


--- back

# Acknowledgments
{:numbered="false"}

Portions of this document were drafted with the assistance of Claude
(Claude Code, Anthropic).
