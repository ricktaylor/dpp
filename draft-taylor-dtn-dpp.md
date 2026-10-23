---
title: "DTN Peering Protocol"
abbrev: "DPP"
category: exp

docname: draft-taylor-dtn-dpp-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Internet"
workgroup: "Delay/Disruption Tolerant Networking"
keyword:
 - DTN
 - peering
 - routing
 - delay-tolerant networking
venue:
  group: "Delay/Disruption Tolerant Networking"
  type: "Working Group"
  mail: "dtn@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/dtn/"
  github: "ricktaylor/dpp"
  latest: "https://ricktaylor.github.io/dpp/draft-taylor-dtn-dpp.html"

author:
 -
    fullname: "Rick Taylor"
    organization: Aalyria Technologies
    email: "rtaylor@aalyria.com"

normative:
  RFC4035:
  RFC6698:
  RFC7671:
  RFC8126:
  RFC8446:
  RFC8552:
  RFC9171:
  RFC9364:
  RFC9460:
  RFC9758:
  I-D.ietf-dtn-eid-pattern:
  I-D.taylor-dtn-eid-pattern-dtn:
    title: "Endpoint ID Patterns for the dtn URI Scheme"
    author:
      - fullname: Rick Taylor
    date: 2026
    seriesinfo:
      Internet-Draft: draft-taylor-dtn-eid-pattern-dtn-00
    ann: "Work in progress; not yet published."
  PROTOBUF:
    title: "Protocol Buffers Version 3 Language Specification"
    author:
      - org: Google
    target: https://protobuf.dev/reference/protobuf/proto3-spec/
  GRPC:
    title: "gRPC over HTTP2"
    author:
      - org: The gRPC Authors
    target: https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md

informative:
  RFC3345:
  RFC4271:
  RFC4451:
  RFC4724:
  RFC7606:
  RFC6480:
  RFC6811:
  RFC7911:
  RFC9319:
  RFC9461:
  RFC8205:
  RFC9174:
  I-D.ek-dtn-ipn-arpa:
  I-D.ietf-dtn-bibe:
  I-D.ietf-sidrops-aspa-verification:
  CCSDS-SABR:
    title: "Schedule-Aware Bundle Routing"
    author:
      - org: Consultative Committee for Space Data Systems
    date: 2019
    seriesinfo:
      CCSDS: "Recommended Standard 734.3-B-1"
    target: https://ccsds.org/Pubs/734x3b1.pdf

...

--- abstract

This document specifies the DTN Peering Protocol (DPP), an inter-domain
routing protocol for the Delay-Tolerant Networking (DTN) ecosystem. DPP
facilitates the exchange of reachability information between distinct
Administrative Domains (ADs), enabling inter-domain routing across networks
operated by different agencies and organizations. DPP separates the control
plane from the data plane: a DPP
speaker need not be a gateway that forwards bundles, allowing centralized
route controllers or orchestration systems to participate in peering on
behalf of the gateways they manage.

DPP advertises the reachability of bundle nodes in both DTN addressing
schemes, `ipn` (integer-based) and `dtn` (name-based), and selects routes
for both by the same process. DPP speakers
authenticate each other using mutual TLS, with each Administrative Domain
authorizing its speakers' keys through DNSSEC-signed TLSA records. DPP
supports both persistent routes and routes tied to the scheduled contact
windows of deep-space networks.


--- middle

# Introduction

The DTN Peering Protocol (DPP) exchanges reachability information between
distinct Administrative Domains (ADs) of a Delay-Tolerant Network. It is an
inter-domain routing protocol, analogous in role to BGP {{RFC4271}} in the
Internet, and complements, rather than replaces, the routing that each AD
uses internally.

A DPP speaker, acting for an AD, establishes an authenticated session with
a speaker of a neighboring AD, over which each advertises the bundle nodes
that are reachable through its AD, together with the gateway to which
bundles for those nodes should be sent and attributes such as when the
route is available. A speaker that receives routes selects among them, uses
the result to direct the forwarding of bundles by the gateways of its own
AD, and may advertise it onward to other ADs, offering transit. How a
speaker configures its AD's gateways is outside the scope of this
document.

## Motivation

Two DPP speakers establish a session to exchange reachability information
across administrative boundaries. Common scenarios include:

Inter-Agency Connectivity:
: Two space agencies operate independent DTN infrastructures. DPP allows
  their border gateways to advertise which spacecraft and ground stations
  are reachable through each network, enabling bundles to flow between
  agencies.

Commercial-Government Integration:
: A commercial satellite operator peers with a government science network. DPP
  enables the commercial network to advertise transit capacity to deep-space
  assets, while the science network advertises its spacecraft endpoints.

Redundant Path Discovery:
: A Mars relay satellite is reachable via two ground station complexes. If
  both complexes belong to the same AD, DPP allows both paths to be
  advertised with appropriate metrics, enabling the network to prefer one
  path while maintaining the other as backup. If they belong to different
  ADs, each AD advertises its own path, and peers choose between them by
  local policy, estimated delivery time, and path length.

Scheduled Contact Windows:
: Deep-space links are available only during planned contact windows. DPP's
  time-variant routing attributes allow DPP speakers to advertise future
  connectivity, enabling proactive bundle scheduling.

Independent Intra-Domain Routing:
: DPP plays the role of an exterior gateway protocol: it carries routing
  information between ADs and leaves each AD free to choose how it routes
  bundles internally. One AD might use Contact Graph Routing (CGR),
  standardized by CCSDS as Schedule-Aware Bundle Routing (SABR)
  {{CCSDS-SABR}}; another might use static routes, or routes computed by
  its mission planning or network orchestration systems. DPP speakers
  advertise reachability derived from whatever their AD uses internally,
  and peers need neither know nor share that mechanism. This also avoids
  any need for a single contact plan spanning all agencies, spacecraft, and
  ground stations: each organization maintains its own mission planning
  systems and contact schedules, and the cost of contact-based routing
  grows with network size.

## Design Principles

Transport:
: Messages are defined using Protocol Buffers {{PROTOBUF}} and carried
  over gRPC {{GRPC}}, and so over any transport that gRPC supports.

Both EID schemes:
: Reachability is carried for both the `ipn` and `dtn` schemes, and routes
  for both are selected by the same process.

DNS-rooted identity:
: ADs are identified by DNS domain names, and each AD uses its own DNS
  zone to authorize the speakers that act on its behalf.

Identity separate from reachability:
: An AD's identity is independent of the nodes it advertises routes for.

## Scope

This specification defines the protocol messages, state machine, and
operational requirements for DPP implementations. It does not define:

- Routing within an Administrative Domain (e.g., CGR or static routing),
  which is chosen independently by each AD
- How a DPP speaker configures the gateways of its AD with the routes it
  has selected
- Bundle forwarding algorithms
- Policy languages for route filtering


# Conventions and Terminology

{::boilerplate bcp14-tagged}

## Definitions {#definitions}

Administrative Domain (AD):
: An autonomous unit of routing control identified by a DNS domain name.
  Examples include agency networks (e.g., `dsn.example.org`), commercial
  providers, or research institutions.

DPP Speaker:
: An entity that participates in the DPP protocol, maintaining peering
  sessions with other DPP speakers and exchanging route information on behalf
  of an Administrative Domain. A DPP speaker may be a gateway, a centralized
  route controller, or any other system authorized to advertise and receive
  routes for its AD.

Gateway:
: A bundle node within an Administrative Domain that forwards bundles to
  or from other ADs. A gateway may or may not also be a DPP speaker. A
  route names the gateway to which bundles should be sent
  ({{gateway-eid-resolution}}).

Session:
: A connection between two DPP speakers over which routes are exchanged.
  The speaker that opens the connection is the Initiator; the other is the
  Responder. Once a session is established the roles are symmetric.

Route:
: A statement, received over a session, that bundles for the nodes matching
  an EID pattern can be delivered via the advertising AD, together with
  the attributes of that statement. A speaker also holds routes that it
  originates for nodes within its own AD.

EID Pattern:
: A pattern matching one or more Bundle Protocol {{RFC9171}} Endpoint
  Identifiers, as defined in {{I-D.ietf-dtn-eid-pattern}} and
  {{I-D.taylor-dtn-eid-pattern-dtn}}.

AD_PATH:
: The ordered list of Administrative Domains that a route has traversed,
  most recent first. The last entry is the originating AD. DPP is a
  path-vector protocol: AD_PATH is used to detect loops and to compare path
  lengths, as AS_PATH is in BGP.

Metric:
: A numeric value set by the originating AD to express its preference
  among routes into its own network. It is analogous to BGP's Multi-Exit
  Discriminator, with the differences described in {{metric}}.

Local preference:
: A preference that a speaker assigns to routes by local policy, for
  example by originating AD or by the peer from which they were learned.
  It is the first criterion of route selection after specificity
  ({{route-selection}}).

Transitive attribute:
: A route attribute that a speaker propagates when it re-advertises a route
  even if it does not understand the attribute. Non-transitive attributes
  are dropped by a speaker that does not understand them
  ({{forward-compatibility}}).

Routing Information Base (RIB):
: The set of routes a speaker holds, from all sessions and its own
  origination, from which it selects routes. This document does not
  prescribe how it is organized.

Forwarding Information Base (FIB):
: The table used by a gateway to forward bundles. A speaker populates the
  FIBs of its AD's gateways from the routes it selects; how it does so is
  outside the scope of this document.


# Architecture Overview

## Administrative Domains

An Administrative Domain is an autonomous unit of routing control.

Identity:
: An AD is identified solely by a DNS domain name (e.g., `dsn.example.org`).

Independence:
: An AD Identity is distinct from the nodes it advertises. An AD named
  `dsn.example.org` may advertise `ipn:100.*.*` and
  `dtn://rover1.example.org/**`.

Advertising versus originating:
: An AD that advertises a route is not necessarily the AD that originates it.
  An AD originates routes for the resources it serves, and may also
  re-advertise routes learned from other ADs, providing transit. The
  originating AD is identified by the last element of the route's AD_PATH
  (see {{loop-detection}}).

## DPP Speakers and Gateways

DPP speakers and gateways are defined in {{definitions}}. A DPP speaker and
a gateway MAY be the same node, but they need not be. For example, a
centralized route controller may run DPP sessions on behalf of multiple
gateways within its AD, naming the appropriate gateway in each route it
advertises. This separation of the control plane (DPP) from the data plane
(bundle forwarding) allows flexible deployment topologies.

An Administrative Domain MAY deploy multiple DPP speakers, each maintaining
independent sessions with external ADs. Because loop detection operates on
AD identity rather than on individual speaker identity, routes that leave
an AD through one speaker and return through another are detected as loops
({{loop-detection}}). Keeping the routes selected by an AD's several
speakers consistent with one another, so that the AD's gateways forward
coherently, is an intra-domain matter outside the scope of this document.

## Deployment Model

DPP depends on DNS, which is not available on spacecraft and requires
low-latency connectivity. This is compatible with deep-space deployment
because of where the protocol's components sit:

DPP speakers are ground infrastructure.
: DPP operates at administrative boundaries. DPP speakers are ground
  stations, mission control centers, network operations facilities, or
  centralized route controllers, all of which have Internet connectivity
  and can perform DNS lookups. Gateways, by contrast, are the forwarding
  nodes that handle bundle traffic and may be located anywhere, including
  on spacecraft or relay satellites, since they need neither DNS access nor
  direct participation in DPP.

DNS is used at session establishment only.
: DPP performs DNS lookups only when establishing a session, to find a
  peer's DPP service and to retrieve the TLSA records that authorize the
  peer's key. Once a session is established, no further DNS queries are
  required, and the session is unaffected by the subsequent unavailability
  of DNS.

Spacecraft do not run DPP.
: Spacecraft run the Bundle Protocol and may use CGR or static routing for
  their own forwarding decisions, under instructions from their ground-based
  mission operations, which participate in DPP on their behalf. Spacecraft
  and relay satellites MAY serve as gateways, forwarding bundles between
  ADs, without participating in DPP: a ground-based DPP speaker advertises
  routes on their behalf, naming the in-space node as the gateway.

As with BGP, the systems that run the routing protocol require connectivity
that the systems they route for need not have.


# Trust and Verification Model {#trust}

DPP uses DNS as the root of trust for Administrative Domain identities. An
AD authorizes DPP speakers to act on its behalf by publishing, in its own
DNS zone, TLSA records {{RFC6698}} that identify the speakers' keys. DPP
speakers authenticate each other using mutual TLS {{RFC8446}}, and each
verifies that the key its peer presented in the TLS handshake is one that
the peer's claimed AD has authorized, following the DNS-Based
Authentication of Named Entities (DANE) model {{RFC7671}}.

Because the key authorized in DNS is the key that authenticates the TLS
connection, the proof of authorization is bound to the session itself.

In the anticipated mode of operation, a DPP speaker is provisioned with a
list of DNS domain names, one for each Administrative Domain with which its
operator wishes to peer. The speaker uses each name to verify the identity
of the corresponding peer (as described below), and may also use it to
discover how to reach that peer's DPP service.

## DNS Records at _dpp {#dns-records}

An AD publishes the records used by DPP at the owner name
`_dpp.<AD-Domain>`. The `_dpp` label is specific to this protocol: the
records it holds authorize and locate DPP speakers, and do not describe
DTN nodes or convergence layers in general.

The zone containing these records MUST be signed with DNSSEC {{RFC9364}}.

### Authorization: TLSA Records {#tlsa-records}

To authorize a DPP speaker to act for an AD Identity (e.g.,
`dsn.example.org`), the administrator MUST publish a TLSA record {{RFC6698}}
for the speaker's key at `_dpp.<AD-Domain>`:

~~~
_dpp.<AD-Domain>.  IN  TLSA  <usage> <selector> <mtype> <data>
~~~

This owner name differs from the `_port._protocol.host` form described in
{{Section 3 of RFC6698}}: the records authorize keys to act for an AD, not
for a service at a particular host and port, so they are published under
the AD's own name. A speaker's TLSA record is therefore independent of the
speaker's hostname, and of whether the speaker accepts incoming sessions.

The consequences of this naming are as follows. The owner name is
registered for the TLSA RR type in the "Underscored and Globally Scoped
DNS Node Names" registry ({{iana-underscore}}), as {{RFC8552}} requires
for any underscored name used with a given RR type. The rules of
{{Section 3 of RFC6698}} and {{Section 7 of RFC7671}} for deriving the
TLSA base domain from a transport name, port, and CNAME chain do not
apply: the base domain is always `_dpp.<AD-Domain>` for the AD named in
the peer's `Hello`, and a speaker MUST NOT derive it from the hostname it
connected to or from any SVCB record. The same records serve to
authenticate both the Initiator and the Responder, whereas {{RFC6698}} and
{{RFC7671}} describe the authentication of a TLS server only. Because the
certificate usage is DANE-EE(3), the names in the certificate are not
checked ({{Section 5.1 of RFC7671}}), which is what allows the
authorization to be independent of the speaker's hostname.

The following rules apply:

- The certificate usage MUST be DANE-EE(3) ({{Section 5.1 of RFC7671}}).
  Records with any other certificate usage MUST be treated as unusable. As
  a consequence, a speaker's certificate need not be issued by any
  certification authority, and the names and validity period in the
  certificate are ignored; a self-signed certificate is sufficient.

- Implementations MUST support the SubjectPublicKeyInfo selector, SPKI(1),
  and the SHA2-256(1) matching type. ADs SHOULD publish "3 1 1" records, as
  recommended in {{Section 5.1 of RFC7671}}. Because such a record matches
  the speaker's public key rather than its certificate, it need not change
  when the certificate is renewed with the same key.

Example:

~~~
_dpp.dsn.example.org.  IN  TLSA  3 1 1 (
    8f3a5e...c2 )
~~~

### Discovery: SVCB Records {#svcb-records}

An AD whose DPP speakers accept incoming sessions MAY also publish SVCB
records {{RFC9460}} at `_dpp.<AD-Domain>`, identifying the hosts and
ports at which those speakers can be reached. An Initiator MAY use these
records to locate a peer's DPP service, or MAY instead be configured with
the peer's address; in the terms of {{Section 3 of RFC9460}}, DPP is
SVCB-optional. SVCB records play no part in authorization: a speaker
listed in an SVCB record is authorized only if it presents a key matching
a TLSA record.

#### SVCB Protocol Mapping for DPP {#svcb-mapping}

This subsection is the protocol mapping for DPP required by
{{Section 2.4.3 of RFC9460}}.

Query name:
: `_dpp.<AD-Domain>`, where `<AD-Domain>` is the AD Identity. Port Prefix
  Naming ({{Section 2.3 of RFC9460}}) is not used: the port is carried in
  the `port` SvcParam, and no port label is prepended.

Authentication name:
: None. The TLS connection to a DPP speaker is authenticated against the
  TLSA records at `_dpp.<AD-Domain>` ({{tlsa-records}}), not against the
  TargetName. The TargetName therefore need not appear in the speaker's
  certificate, and a speaker reached through an SVCB record is
  authenticated in exactly the same way as one reached through
  configuration.

Default ALPN set:
: `h2`, since DPP is carried over gRPC {{GRPC}}, which uses HTTP/2. The
  `alpn` SvcParam MAY be used to list additional protocols that the
  speaker supports for gRPC, and `no-default-alpn` MAY be used to
  exclude `h2`. There is no DPP-specific ALPN identifier.

Applicable SvcParamKeys:
: `mandatory`, `alpn`, `no-default-alpn`, `port`, `ipv4hint`, and
  `ipv6hint`, with the meanings given in {{Section 7 of RFC9460}}. Other
  keys MUST be ignored.

Default port:
: 50051, the port conventionally used by gRPC services. No port is
  assigned to DPP by IANA; see {{iana-future}}. A ServiceMode record that
  lacks the `port` SvcParam directs the Initiator to the default port
  ({{Section 7.2 of RFC9460}}). An Initiator that is configured with a
  peer's address, rather than discovering it, likewise uses the default
  port unless configured otherwise.

Automatically mandatory keys:
: None.

AliasMode:
: Permitted, but NOT RECOMMENDED, for the reasons given in
  {{Section 6 of RFC9461}} for a comparable protocol. Whether or not an
  alias is followed, the TLSA records used for authorization are those at
  `_dpp.<AD-Domain>` for the AD named in the peer's `Hello`, never those
  at an alias target.

The TargetName of each ServiceMode record SHOULD be the hostname of a DPP
speaker. A TargetName of "." would make the `_dpp` owner name the
effective TargetName ({{Section 2.5.2 of RFC9460}}), requiring address
records at a name beginning with an underscore label, which some
authoritative servers cannot publish ({{Section 10.3 of RFC9460}}).

Examples, the first using the default port and the second a port chosen
by the operator:

~~~
_dpp.dsn.example.org.  IN  SVCB  1 speaker.dsn.example.org.
_dpp.dsn.example.org.  IN  SVCB  2 backup.dsn.example.org. port=4600
~~~

### Multiple DPP Speakers per AD {#multiple-speakers}

An Administrative Domain that deploys multiple DPP speakers has two options
for key management:

Shared key:
: All speakers share a single key pair. The AD publishes one TLSA record and
  distributes the private key to each speaker. This is operationally simple
  but means that compromise of any single speaker exposes the key for the
  entire AD.

Per-speaker keys:
: Each speaker has its own key pair, and the AD publishes one TLSA record
  for each. A peer accepts any speaker whose key matches one of the records.

Per-speaker keys provide better operational security: a compromised speaker
can be revoked by removing its TLSA record, without affecting other
speakers in the AD. The choice between shared and per-speaker keys is a
local operational decision and is transparent to the peering partner.

The same mechanism allows an AD to delegate peering to a third party, such
as a network operations provider that runs DPP speakers on the AD's behalf.
The AD publishes a TLSA record for the delegate's key. The delegate never
holds a credential for the AD's zone, and the delegation is revoked by
removing the record.

Example with per-speaker keys, one of them delegated, and SVCB records for
the two speakers that accept incoming sessions:

~~~
; Speakers accepting incoming sessions, on the default port
_dpp.dsn.example.org.  IN  SVCB  1 gds.dsn.example.org.
_dpp.dsn.example.org.  IN  SVCB  2 cbr.dsn.example.org.

; Keys authorized to act for dsn.example.org
_dpp.dsn.example.org.  IN  TLSA  3 1 1 (
    8f3a5e...c2 )   ; Goldstone
_dpp.dsn.example.org.  IN  TLSA  3 1 1 (
    51d07b...9e )   ; Canberra
_dpp.dsn.example.org.  IN  TLSA  3 1 1 (
    a7e214...41 )   ; Delegated operator (initiates only)
~~~

## Session Establishment and Authentication {#handshake}

A DPP session is established as follows:

1. TLS: The Initiator opens a TLS connection to the Responder. Both
   speakers MUST use TLS 1.3 {{RFC8446}} or later, and both MUST present a
   certificate: the Responder MUST request a certificate from the
   Initiator, and MUST abort the connection if none is provided. The
   Responder does not learn which AD the Initiator claims to represent
   until it receives the Initiator's `Hello`, so it accepts the
   Initiator's certificate provisionally and completes its verification in
   step 5.

2. Hello: Once the gRPC stream is established, each speaker MUST send a
   `Hello` message, containing its `local_ad_id`, as its first message on
   the stream. A speaker need not wait for its peer's `Hello` before
   sending its own.

3. Peer check: On receiving the peer's `Hello`, the Initiator MUST reject
   the session if the `local_ad_id` it contains is not the AD to which the
   Initiator intended to connect. The Responder MUST reject the session if
   the `local_ad_id` is not an AD with which it is configured to peer.
   Performing this check before the DNS lookup prevents an unauthenticated
   peer from causing lookups of arbitrary names. Either speaker MUST also
   reject the session if the `version` in the peer's `Hello` is not one it
   supports, or if the peer's `hold_time_seconds` is unacceptable
   ({{hold-time}}).

4. Lookup: The speaker MUST query DNS for TLSA records at
   `_dpp.<local_ad_id>`, where `local_ad_id` is the AD named in the
   peer's `Hello`. The TLSA RRset MUST be validated as secure using
   DNSSEC. The speaker MUST either perform DNSSEC validation itself, or
   obtain the validation result from a validating resolver that it
   trusts and that it reaches over a channel protected from
   modification, as described in {{Section 4.9.3 of RFC4035}}; an AD
   bit received over an unprotected channel MUST NOT be relied on. If
   the validation state is insecure, indeterminate, or bogus, or the
   RRset contains no usable records ({{Section 4.1 of RFC6698}}), the
   speaker MUST reject the session.

5. Verification: The certificate that the peer presented in the TLS
   handshake MUST match at least one usable TLSA record, using the
   DANE-EE matching rules of {{Section 5.1 of RFC7671}}. If none matches,
   the speaker MUST reject the session.

A speaker that rejects a session SHOULD send a `Notification` with an
appropriate error code before closing the stream. From each speaker's point
of view, the session is ESTABLISHED once it has both sent its own `Hello`
and verified its peer's. A speaker MUST NOT send `RouteUpdate` messages
before then.

Because the Initiator knows which AD it is connecting to, it MAY perform
steps 4 and 5 for the Responder during the TLS handshake itself, and so
avoid sending its `Hello` to an unauthorized peer.

DNS is consulted only during session establishment, and an established
session is not affected by the subsequent unavailability of DNS. A speaker
MAY periodically repeat steps 4 and 5 for an established session, and
terminate the session if the peer's key is no longer authorized. A lookup
failure during such a check MUST NOT by itself cause the session to be
terminated.


# Session State Machine

A conformant DPP speaker MUST implement the following session state
machine. The messages it refers to are defined in {{protocol-definition}}.
The state machine, like the decision process of {{route-selection}}, is
conceptual: an implementation need not be structured as described,
provided that its externally visible behavior is the same.

## States {#states}

Both speakers run the same state machine. The steps referred to are those
of {{handshake}}.

| State | Description |
|-------|-------------|
| CONNECTING | TLS connection established with certificates exchanged, and gRPC stream open (step 1); the peer is not yet verified. Speaker MUST send its `Hello` and move to HELLO_SENT. |
| HELLO_SENT | `Hello` sent; waiting for the peer's `Hello`. Upon receipt, speaker moves to VERIFYING. |
| VERIFYING | Peer's `Hello` received. Speaker performs the peer check, DNS lookup, and verification (steps 3-5). On success, it moves to ESTABLISHED. |
| ESTABLISHED | Peer verified. Speaker MAY send and receive `RouteUpdate` and `KeepAlive` messages, beginning with the initial route exchange ({{initial-exchange}}). |
| CLOSING | The session is ending, because of an error or an orderly close. Speaker MUST send a `Notification` (if it has not already received one that ends the session, and if it can) and close the stream ({{termination}}). |

A `Notification` MAY be sent and received in any state.

A speaker MUST move to CLOSING, sending the `Notification` code shown, if:

- any step of {{handshake}} fails (`UNSUPPORTED_VERSION`, `UNEXPECTED_AD`,
  `AUTHORIZATION_FAILED`, or `UNACCEPTABLE_HOLD_TIME`, as appropriate);
- it receives any message other than `Hello` or `Notification` before
  reaching ESTABLISHED, or a second `Hello` on the same stream
  (`UNEXPECTED_MESSAGE`);
- the session does not reach ESTABLISHED within a locally configured time
  (`UNSPECIFIED`);
- the hold timer expires ({{hold-time}}; `HOLD_TIMER_EXPIRED`);
- it receives more routes from the peer than it will accept
  ({{denial-of-service}}; `MAX_ROUTES_EXCEEDED`); or
- it decides to close the session for any other reason (`CEASE`, or
  `CONNECTION_COLLISION` in the case of {{sessions}}).

A speaker MUST also move to CLOSING, without sending a `Notification`, if
it receives a `Notification` whose code ends the session
({{control-messages}}), or if the gRPC stream is closed by the peer or
fails.

In the ESTABLISHED state, a speaker that receives a `PeerMessage` whose
payload it does not recognize, or a `Hello`, SHOULD send a `Notification`
with level `WARNING` and code `UNEXPECTED_MESSAGE`, and MUST otherwise
ignore the message. A speaker MUST NOT send a message defined by an
extension that is not in effect on the session ({{identity-message}}).

## Hold Time and Keepalives {#hold-time}

Each speaker proposes a hold time in the `hold_time_seconds` field of its
`Hello`, which MUST be at least 3 seconds. A speaker that receives a
`Hello` with a `hold_time_seconds` of less than 3 MUST reject the session
with code `UNACCEPTABLE_HOLD_TIME`. The hold time for the session is the
smaller of the two values.

In the ESTABLISHED state, a DPP speaker MUST send `KeepAlive` messages at
intervals not exceeding one third of the session hold time. If a speaker
receives no message from its peer for a period equal to the session hold
time, it MUST move to CLOSING with code `HOLD_TIMER_EXPIRED`.

## Initial Route Exchange {#initial-exchange}

On entering ESTABLISHED, a speaker MUST send to its peer every route that
it would advertise to that peer at that time ({{readvertisement}}),
followed by a `RouteUpdate` with `initial_update_complete` set. That
`RouteUpdate` MAY also carry announcements or withdrawals. A speaker MUST
send it even if it has no routes to advertise. The flag MUST NOT be set in
any later `RouteUpdate` on the session, and a receiving speaker MUST
ignore it if it is.

Until a speaker has received its peer's `RouteUpdate` with
`initial_update_complete` set, or a locally configured time has elapsed
since the session became ESTABLISHED, it SHOULD defer updating its own
advertisements to other peers in response to routes received over the new
session. Without this, each advertisement received during the initial
exchange could change the speaker's selected routes, and so cause a
separate update to every other peer. Routes received during the initial
exchange MAY be used for forwarding as soon as they are received. This is
analogous to the End-of-RIB marker of {{RFC4724}}.

## Multiple Sessions and Collisions {#sessions}

A speaker MAY hold sessions with several speakers of the same AD, and an
AD's speakers MAY each hold sessions with the same peer; routes received
over each session are independent ({{advertisement-id}}). Two sessions
between the same pair of speakers, which can arise when each is configured
to initiate a session to the other, are not harmful, but carry the same
routes twice.

A speaker identifies a peer speaker by the public key that the peer
presented in the TLS handshake ({{handshake}}) together with the peer's
`speaker_id`; speakers of one AD that share a key ({{multiple-speakers}})
SHOULD therefore set distinct `speaker_id` values. When a speaker has two
sessions with the same peer speaker, both of which are ESTABLISHED or
being established, it SHOULD close one of them with code
`CONNECTION_COLLISION`. So that both speakers close the same session, the
session to close is the one initiated by the speaker whose public key has
the numerically greater SHA-256 digest, with the digests compared as
unsigned big-endian integers. A speaker MUST NOT close a session for this
reason until the newer session is ESTABLISHED, so that a session is not
lost to a connection attempt that fails.

## Session Termination {#termination}

A speaker that is ending a session, whether because of an error or by
choice, MUST send a `Notification` whose code describes the reason before
closing the gRPC stream, if it is able to; `CEASE` is used for an orderly
close with no other cause. It MUST then close the stream. It SHOULD log
the reason.

A session has ended, from a speaker's point of view, when it has sent or
received a `Notification` whose code ends the session, or when the gRPC
stream has been closed by the peer or has failed. When a session ends, the
speaker MUST remove all routes received over it ({{advertisement-id}}),
repeat route selection, and update its advertisements to other peers
({{readvertisement}}).

An Initiator whose session has ended MAY establish a new one. It SHOULD
wait before doing so, for an interval that increases with each successive
failure to establish or keep a session with the same peer, so that a
persistently failing peer is not retried continuously; it MAY reconnect
promptly after an orderly `CEASE`.


# Protocol Definition {#protocol-definition}

This section defines the messages exchanged over a session, as Protocol
Buffers {{PROTOBUF}} carried over a single bidirectional gRPC {{GRPC}}
stream for the lifetime of the session. The sections that follow specify
how the routing messages are processed.

## Service Definition

~~~ protobuf
syntax = "proto3";
package dtn.peering.v1;

import "google/protobuf/timestamp.proto";

service DtnPeering {
  // Single stream for Hello, Routes, and Keepalives
  rpc Peer(stream PeerMessage) returns (stream PeerMessage);
}
~~~

## Message Envelope

~~~ protobuf
message PeerMessage {
  uint64 sequence_number = 1;
  reserved 11, 12;
  oneof payload {
    Hello hello = 10;
    KeepAlive keep_alive = 20;
    RouteUpdate update = 21;
    Notification notification = 30;
  }
}
~~~

## Identity Message {#identity-message}

~~~ protobuf
message Hello {
  // DNS Domain (e.g., "dsn.example.org")
  string local_ad_id = 1;
  // Optional, free-form (e.g., "goldstone-1")
  string speaker_id = 2;
  uint32 hold_time_seconds = 3;
  // Protocol version; 1 for this document
  uint32 version = 4;
  // Optional protocol extensions supported by the sender
  repeated string capabilities = 5;
}
~~~

The `version` field identifies the version of this protocol that the
sender implements. This document defines version 1. A speaker MUST set
`version` to the highest version it supports, and MUST reject a session
if the `version` in its peer's `Hello` is not one that it supports
({{handshake}}). A `Hello` in which `version` is absent has version 0,
which is not valid.

The `capabilities` field lists optional extensions to this protocol that
the sender supports, by the names registered for them
({{iana-capabilities}}); this document defines none. An extension is in
effect on a session only if both speakers listed it. A speaker MUST ignore
capability names that it does not recognize, and MUST NOT send any message
or attribute defined by an extension that is not in effect.

The optional `speaker_id` field allows a speaker to identify itself
among the speakers of its AD, for example in logs and diagnostics, and to
distinguish itself from other speakers of its AD that present the same
key ({{sessions}}). Its content is chosen by the AD and has no defined
format; in particular, it need not be an EID or a DNS name, since a DPP
speaker need not be a BP node. A receiving speaker MUST NOT use
`speaker_id` for authentication, authorization, loop detection, or route
selection.

Authentication is performed by TLS (see {{handshake}}), so no
authentication messages are defined.

## Routing Messages {#routing-messages}

The semantics of these messages are specified in {{routes}}.

~~~ protobuf
message RouteUpdate {
  repeated RouteAdvertisement announcements = 1;
  repeated RouteWithdrawal withdrawals = 2;
  // Set once per session, on the last message of the
  // initial route exchange (see Initial Route Exchange)
  bool initial_update_complete = 3;
}

// One or more routes sharing the same attributes
message RouteAdvertisement {
  // Destination nodes; each pattern creates a route
  repeated EidPattern patterns = 1;

  // Required attributes
  // AD Domains traversed, most recent first
  repeated string ad_path = 2;
  // Originating AD's preference (lower = preferred)
  uint32 metric = 3;

  // Optional attributes
  repeated RouteAttribute attributes = 10;

  // Advertisement ID: identifies this advertisement within the
  // session; re-sending the same id replaces the advertisement
  uint64 id = 11;
}

// Removes all routes created by the advertisements with the
// listed advertisement IDs
message RouteWithdrawal {
  reserved 1, 2;
  repeated uint64 advertisement_ids = 3;
}
~~~

## Route Attributes {#route-attributes}

The semantics of each attribute are specified in {{routes}}.

~~~ protobuf
// Optional route attribute; see Optional Attributes
message RouteAttribute {
  oneof attribute {
    // Bundle forwarding endpoint, for data plane separation
    string gateway_eid = 1;

    // Time window: when the advertising AD starts and stops
    // accepting bundles
    google.protobuf.Timestamp valid_from = 2;
    google.protobuf.Timestamp valid_until = 3;

    // What the advertising AD accepts
    uint64 bandwidth_bps = 4;     // Rate (bits per second)
    uint32 max_bundle_size = 5;   // Maximum bundle size (bytes)

    // Estimated earliest delivery time at the destination
    google.protobuf.Timestamp arrival_time = 6;

    // Forward compatibility: an attribute type this
    // implementation does not define (see Forward Compatibility)
    UnknownAttribute unknown = 255;
  }
}

// For forward compatibility with future attribute types
message UnknownAttribute {
  uint32 type_id = 1;   // Identifies the attribute type
  bytes value = 2;      // Opaque value
  bool transitive = 3;  // If true, propagate even if not understood
}
~~~

## Data Types {#data-types}

~~~ protobuf
message EidPattern {
  oneof scheme {
    IpnPattern ipn = 1;
    DtnPattern dtn = 2;
  }
}

// Matches every Service Number of the nodes it identifies.
// An unset component is a wildcard; ipn:** has both unset.
message IpnPattern {
  IpnComponent allocator = 1;
  IpnComponent node = 2;
}

message IpnComponent {
  oneof value {
    uint32 exact = 1;
    IpnRange range = 2;
  }
}

// Aligned power-of-two range of size > 1 (inclusive bounds)
message IpnRange {
  uint32 min = 1;
  uint32 max = 2;
}

// Matches every demux of the nodes it identifies. The authority
// is in the text form of the dtn EID Pattern specification.
message DtnPattern {
  string authority = 1;  // e.g. "*.example.org"
}
~~~

## Control Messages {#control-messages}

~~~ protobuf
message KeepAlive {}

message Notification {
  enum Level { INFO = 0; WARNING = 1; ERROR = 2; }
  enum Code {
    UNSPECIFIED = 0;
    // Session establishment
    UNSUPPORTED_VERSION = 1;
    UNEXPECTED_AD = 2;
    AUTHORIZATION_FAILED = 3;
    UNACCEPTABLE_HOLD_TIME = 4;
    // Session operation
    HOLD_TIMER_EXPIRED = 5;
    UNEXPECTED_MESSAGE = 6;
    MALFORMED_UPDATE = 7;
    MAX_ROUTES_EXCEEDED = 8;
    // Orderly close
    CEASE = 9;
    CONNECTION_COLLISION = 10;
  }
  Level level = 1;
  // Human-readable detail; not for machine interpretation
  string message = 2;
  Code code = 3;
}
~~~

A `Notification` reports a condition to the peer. It MAY be sent in any
state, including before a `Hello`. The `level` indicates severity for
logging and diagnostics. The `code` identifies the condition, and
determines whether the sender is ending the session:

| Code | Sent when | Ends the session |
|------|-----------|------------------|
| UNSPECIFIED | No other code applies | No |
| UNSUPPORTED_VERSION | The peer's `version` is not supported ({{handshake}}) | Yes |
| UNEXPECTED_AD | The peer's `local_ad_id` fails the peer check ({{handshake}}) | Yes |
| AUTHORIZATION_FAILED | TLSA lookup or verification fails ({{handshake}}) | Yes |
| UNACCEPTABLE_HOLD_TIME | The peer's hold time is unacceptable ({{hold-time}}) | Yes |
| HOLD_TIMER_EXPIRED | No message received within the hold time ({{hold-time}}) | Yes |
| UNEXPECTED_MESSAGE | A message not permitted in the current state, or not recognized, was received ({{states}}) | Only before ESTABLISHED |
| MALFORMED_UPDATE | A malformed advertisement was received ({{malformed}}) | No |
| MAX_ROUTES_EXCEEDED | The peer sent more routes than the receiver will accept ({{denial-of-service}}) | Yes |
| CEASE | The sender is closing the session in an orderly way ({{termination}}) | Yes |
| CONNECTION_COLLISION | The sender is closing a redundant session ({{sessions}}) | Yes |

A speaker that receives a `Notification` whose code ends the session MUST
treat the session as ended ({{termination}}). A speaker that receives any
other `Notification` MUST NOT end the session on that account. The
`message` field carries human-readable detail, such as which
advertisement was malformed; a speaker MUST NOT depend on its content.
Codes are registered in the "DPP Notification Codes" registry
({{iana-codes}}).


# Route Attributes and Advertisements {#routes}

Once a session is established ({{handshake}}), DPP speakers share
reachability information by sending `RouteUpdate` messages
({{routing-messages}}), each carrying route advertisements and
withdrawals. A route advertisement tells the peer that bundles destined
for nodes matching a set of EID patterns can be delivered via the
advertising AD, together with the attributes of that route. The form of
those patterns, and how a speaker chooses between routes that match the
same EID, are described in {{addressing}}; how a speaker derives the
routes it advertises from those it has selected is described in
{{readvertisement}}.

Routes are identified by the session they were received over and an
advertisement ID chosen by the sender ({{advertisement-id}}), not by their
patterns. This differs from BGP, in which a new advertisement for a prefix
implicitly replaces the previous one from the same peer: in DPP, a new
advertisement with a different ID adds routes, re-sending an ID replaces
them, and a withdrawal names IDs. The model resembles the path identifiers
of the BGP ADD-PATH extension {{RFC7911}}.

DPP uses an extensible attribute model inspired by BGP path attributes
{{RFC4271}}. This allows new route properties to be defined in future protocol
versions without breaking existing implementations.

## Advertisement Structure

A `RouteAdvertisement` ({{routing-messages}}) carries:

Advertisement ID:
: An `id` that identifies the advertisement within the session, so that it
  can later be replaced or withdrawn ({{advertisement-id}}).

Patterns:
: One or more EID patterns identifying the destination nodes, constrained
  as described in {{pattern-constraints}}. Each pattern creates a separate
  route; all share the advertisement's attributes.

Attributes:
: The properties of the route. Two are always present, `ad_path` and
  `metric`; the rest are optional and are carried in an extensible list.

## Required Attributes

| Field | Description |
|-------|-------------|
| `ad_path` | List of AD domains traversed (for loop detection and path length; see {{loop-detection}}) |
| `metric` | Originating AD's preference among routes into its network (lower = preferred; see {{metric}}) |

## Loop Detection {#loop-detection}

DPP is a path-vector protocol: routing loops are prevented using the
AD_PATH attribute (similar to BGP's AS_PATH {{RFC4271}}). Loop detection
operates on Administrative Domain identity, not on individual DPP speaker
identity. This means an AD MAY deploy multiple speakers without risk of
routing loops, as all speakers within an AD share the same `local_ad_id`.

Mechanism:
: Every route advertisement MUST include the ordered list of ADs it has
  traversed.

Propagation:
: When a DPP speaker advertises a route to a peer, it MUST prepend its own
  `local_ad_id` to the AD_PATH. This applies both to routes it originates
  (which therefore carry an AD_PATH containing only the originating AD) and
  to routes it re-advertises.

Verification:
: When a DPP speaker receives a route advertisement, the first (left-most)
  entry in AD_PATH MUST match the `local_ad_id` that the peer authenticated
  during session establishment ({{handshake}}). If it does not, or if
  AD_PATH is empty, the advertisement is malformed ({{malformed}}). This
  prevents a peer from omitting its own AD from the path, or from
  presenting a route as though it had been received directly from another
  AD.

Detection:
: If a DPP speaker receives an advertisement whose AD_PATH contains its
  own `local_ad_id`, it MUST NOT create routes from it; the advertisement
  is treated as a withdrawal of its ID ({{malformed}}).

AD identities in AD_PATH are DNS domain names, and MUST be compared
case-insensitively.

## Optional Attributes

Optional attributes, each carried as a `RouteAttribute`
({{route-attributes}}):

| Attribute | Description |
|-----------|-------------|
| `gateway_eid` | Bundle forwarding endpoint (see {{gateway-eid-resolution}}) |
| `valid_from` | When the advertising AD starts accepting bundles (if omitted: immediately; see {{time-variant}}) |
| `valid_until` | When the advertising AD stops accepting bundles (if omitted: until withdrawn; see {{time-variant}}) |
| `arrival_time` | Estimated earliest delivery time at the destination (optional; see {{time-variant}}) |
| `bandwidth_bps` | Rate at which the advertising AD accepts bundles, in bits per second |
| `max_bundle_size` | Largest bundle the advertising AD accepts, in bytes |

How each attribute is updated when a route is re-advertised is specified
in {{readvertisement}}.

## Forward Compatibility {#forward-compatibility}

An attribute type that an implementation does not define is received as
an `UnknownAttribute` ({{route-attributes}}), which carries the type, an
opaque value, and a `transitive` flag. Implementations MUST handle it as
follows:

1. If `transitive` is true: preserve the attribute and propagate it
   unchanged when re-advertising the route.
2. If `transitive` is false: discard the attribute, and do not propagate
   it.

This allows intermediate DPP speakers to forward routes with attributes
they do not understand, enabling incremental deployment of new attribute
types.

## Time-Variant Routing {#time-variant}

The `valid_from` and `valid_until` attributes allow an AD to advertise
reachability that is available only at certain times, as is common in
deep-space networks. Together they define the time window during which
the advertising AD will accept bundles for the advertised destinations. A
receiving AD can use these time windows as input to its own routing,
whether that is contact-based (e.g., {{CCSDS-SABR}}) or not. Routes without
these attributes are valid immediately and remain valid until explicitly
withdrawn.

Because DTN is a store-and-forward network, the window in which an AD
accepts bundles need not coincide with the contacts over which it will
forward them. An AD with sufficient storage can accept bundles before its
next contact towards their destination opens, and hold them until it does;
it can also offer a single window that spans several such contacts.

A route is available from the time given by its `valid_from` attribute.
A route with no `valid_from` attribute, or whose `valid_from` time has
passed, is available immediately. A route expires at the time given by its
`valid_until` attribute: once that time has passed, a DPP speaker MUST
treat the route as withdrawn. An advertisement that carries both
attributes with a `valid_until` time not later than its `valid_from` time
is malformed ({{malformed}}).

The optional `arrival_time` attribute gives the advertising AD's estimate
of the earliest time at which a bundle that it accepts at the start of the
route's window can be delivered to the advertised destinations. Unlike the
time window, which describes only the advertising AD, the arrival time
describes the whole path to the destination. It is set by the originating
AD, if that AD can estimate when it will deliver bundles, and is updated by
each AD that re-advertises the route ({{readvertisement}}). A route without
an `arrival_time` attribute provides no estimate.

An advertisement that carries both an `arrival_time` and a `valid_from`,
with the `arrival_time` earlier than the `valid_from`, is malformed
({{malformed}}): a bundle cannot be delivered before it is accepted.

Because the availability of routes changes over time, a DPP speaker MUST
repeat route selection ({{route-selection}}) for the current time whenever
any of the following occurs, and MUST update its own advertisements if the
routes that it would advertise change ({{readvertisement}}):

- it receives, replaces, withdraws, or loses a route;
- the `valid_from` time of a route it holds is reached, making the route
  available;
- the `arrival_time` of a route it holds is reached, after which the route
  is estimated to deliver bundles as soon as they are accepted
  ({{route-selection}}); or
- the `valid_until` time of a route it holds is reached, expiring the
  route.

A peer MAY send multiple advertisements for the same pattern, each with its
own advertisement ID ({{advertisement-id}}), to describe a series of distinct
periods in which it accepts bundles, each with its own attributes.

## Advertisement IDs and Withdrawal {#advertisement-id}

Each `RouteAdvertisement` carries an advertisement ID, in its `id` field,
chosen by the sending speaker, that identifies the advertisement within
the session over which it is sent. Each EID pattern in the advertisement
creates a separate route, which has the advertisement's attributes; the
routes created by an advertisement are identified by the session, the
advertisement ID, and their patterns. The ID identifies the advertisement
as a whole, not any one of the routes it creates.

An advertisement ID is meaningful only within the session in which it is
used. A speaker MAY choose advertisement IDs in any way, provided that it does not
use the same value for two advertisements that it intends to be held at
the same time on the same session. When a session ends, all routes
received over it are removed, and the advertisement IDs used in it have no
significance in any later session.

When a DPP speaker receives a route advertisement, it MUST replace all of
the routes previously received over the same session with the same
advertisement ID, if any, with the routes created by the new advertisement. The
replacement is atomic: the new advertisement's patterns, attributes, and
time window take effect together, and any pattern that was in the previous
advertisement but is not in the new one is removed. This allows a peer to
change any aspect of an advertisement, including its time window, by
re-sending it with the same advertisement ID, without first withdrawing it.

Advertisements with different advertisement IDs are independent, even if
they contain the same pattern and their time windows overlap. A speaker
can therefore hold several routes for the same pattern from a single
peer, for example when a centralized route controller advertises several
entry points into its AD, each with a different `gateway_eid` and metric
({{gateway-eid-resolution}}). Route selection chooses between them in the
same way as between routes from different peers ({{route-selection}}).

A `RouteWithdrawal` lists, in its `advertisement_ids` field, the IDs of
advertisements to be removed. For each ID that it lists, all routes
received over the same session from the advertisement with that ID are
removed. If no advertisement with that ID is held, the ID is silently
ignored. A peer that wishes to remove only some of the patterns in an
advertisement re-sends the advertisement with the same ID, omitting those
patterns.

A route also ceases to exist when it expires ({{time-variant}}), and does
not need to be withdrawn. When every route created by an advertisement has
expired, its advertisement ID is no longer in use.

## Malformed Advertisements and Processing Order {#malformed}

A `RouteAdvertisement` is malformed if any of the following holds:

- it contains no patterns;
- a pattern does not satisfy the constraints of {{pattern-constraints}},
  including the range constraints of {{ipn-constraints}}, or a `dtn`
  pattern has an empty authority;
- its AD_PATH is empty, or the first entry of its AD_PATH does not match
  the AD that the peer authenticated during session establishment
  ({{loop-detection}});
- it carries more than one attribute of the same type, or more than one
  `UnknownAttribute` with the same `type_id`;
- it carries both `valid_from` and `valid_until`, and `valid_until` is not
  later than `valid_from` ({{time-variant}});
- it carries both `valid_from` and `arrival_time`, and `arrival_time` is
  earlier than `valid_from` ({{time-variant}});
- its `gateway_eid` is not the administrative endpoint EID of a node, or
  is a LocalNode EID ({{gateway-eid-resolution}}); or
- its `bandwidth_bps` or `max_bundle_size` is zero.

A pattern whose scheme the receiving speaker does not recognize (that is,
an `EidPattern` in which no known scheme is set) is not malformed: the
speaker MUST ignore that pattern and process the remainder of the
advertisement. This allows patterns for additional schemes to be defined
in future. If ignoring such patterns leaves an advertisement with no
patterns, the advertisement is treated as a withdrawal of its ID, without
notification.

A receiving speaker MUST treat a malformed advertisement as a withdrawal
of its advertisement ID: any routes previously received over the session
with that ID are removed, and no routes are created. The speaker SHOULD
send a `Notification` with level `WARNING` and code `MALFORMED_UPDATE`,
identifying the advertisement in the `message` field, and MUST NOT end the
session on account of a malformed advertisement. This follows the approach
of {{RFC7606}} for BGP: ending the session would remove every route
received over it, which is a far greater disruption than the loss of the
one advertisement, and would make a peer's single error into an outage.

An advertisement whose AD_PATH contains the receiving speaker's own AD
({{loop-detection}}) is likewise treated as a withdrawal of its ID, but
without a `Notification`, since receiving one's own routes back is a
normal occurrence in a meshed topology.

Within a single `RouteUpdate`, a receiving speaker MUST process the
withdrawals before the announcements, and the announcements in the order
in which they appear. Consequently, if an advertisement ID appears both in
the withdrawals and in an announcement, the announcement takes effect; and
if two announcements carry the same ID, the second replaces the first.
Senders SHOULD NOT construct such messages.

A `PeerMessage` that cannot be decoded, or that contains a `RouteUpdate`
before the session is ESTABLISHED, is handled as described in {{states}}.

## Gateway EID Resolution {#gateway-eid-resolution}

The `gateway_eid` attribute identifies the gateway node to which bundles
matching the advertised patterns should be forwarded. This enables
separation of the control plane (DPP session) from the data plane (bundle
forwarding).

The gateway EID always identifies a gateway in the AD that sent the
advertisement, which is the next hop for matching bundles, not a gateway
in the originating AD. A speaker that re-advertises a route therefore
replaces the gateway EID with one of its own ({{readvertisement}}).

A `gateway_eid` MUST be the EID of the gateway node's administrative
endpoint ({{Section 4.2.5.2 of RFC9171}}): for the `ipn` scheme,
`ipn:A.N.0`, the EID with Service Number zero for the node's FQNN
({{Section 5.7 of RFC9758}}); for the `dtn` scheme, `dtn://<node-name>/`,
the EID with a zero-length demux ({{Section 4.2.5.1.1 of RFC9171}}). Any
singleton EID of a node may serve as a node ID for it, but the
administrative endpoint EID is the one such EID that every speaker will
form in the same way, so requiring it ensures that two routes via the same
gateway name it identically. An advertisement whose `gateway_eid` is not
of this form, or is a LocalNode EID ({{Section 5.4 of RFC9758}}), is
malformed ({{malformed}}). A `gateway_eid` SHOULD use the same scheme as the route's patterns, since
the receiving AD must find a convergence layer path to the gateway using
the EID given.

Resolution rules:

1. If a `gateway_eid` attribute is present, forward matching bundles to
   the node it identifies.

2. If `gateway_eid` is absent, derive the gateway EID from the peer's AD
   identity (`local_ad_id` from Hello) by forming the administrative
   endpoint EID of a `dtn` node whose node-name is the AD's domain name:

   ~~~
   <dns_name>  ->  dtn://<dns_name>/
   ~~~

   For example: `dsn.example.org` -> `dtn://dsn.example.org/`

   This default assumes that the AD operates a gateway node whose `dtn`
   node-name is the AD's own domain name. An AD for which that is not the
   case MUST include an explicit `gateway_eid`.

No mapping from a domain name to an `ipn` FQNN is defined: the `ipn.arpa`
zone proposed in {{I-D.ek-dtn-ipn-arpa}} maps an FQNN to the records needed
to reach the node, but does not map a domain name to an FQNN. An AD whose
gateways have only `ipn` EIDs therefore needs to include an explicit
`gateway_eid` attribute.

Determining how to reach a gateway at the convergence layer, once its EID
is known, is a function of the receiving AD's BP agents rather than of
DPP, and can use any available means, including configuration. For a
gateway with an `ipn` EID, {{I-D.ek-dtn-ipn-arpa}} specifies a DNS-based
procedure: for example, TCPCLv4 {{RFC9174}} endpoints for the node
(978879,2) are found from SRV records at
`_dtn-bundle._tcp.2.978879.ipn.arpa.`. No corresponding procedure is
currently specified for gateways with `dtn` EIDs.

This allows the following deployments:

DNS-named gateway:
: An AD with identity `dsn.example.org` that operates a gateway node named
  `dtn://dsn.example.org/` need not include a `gateway_eid`.

Separated control plane:
: A DPP speaker that is not itself a gateway (e.g., a centralized route
  controller) can advertise routes with explicit `gateway_eid` pointing to
  the forwarding nodes within the AD, enabling load distribution or failover.

Load distribution:
: Multiple routes to the same destination can be advertised with different
  `gateway_eid` values and metrics.

Examples:

~~~
# AD identity is DNS name - gateway_eid derived automatically
AD Identity: dsn.example.org
Default Gateway EID: dtn://dsn.example.org/

RouteAdvertisement {
  patterns: [ipn:100.*.*]
  ad_path: ["dsn.example.org"]
  metric: 10
  // No gateway_eid needed - defaults to dtn://dsn.example.org/
}
~~~

~~~
# Explicit gateway_eid for control/data plane separation
AD Identity: dsn.example.org

RouteAdvertisement {
  patterns: [ipn:100.*.*]
  ad_path: ["dsn.example.org"]
  metric: 10
  attributes: [
    gateway_eid: "dtn://ingress1.dsn.example.org/"
  ]
}
~~~


# Addressing and Metrics {#addressing}

This section describes the EID patterns carried in route advertisements
({{routes}}), and how a DPP speaker selects between routes whose patterns
match the same EID.

## Routes Identify Nodes {#routes-identify-nodes}

DPP advertises the reachability of bundle nodes, not of the services
registered at them. A bundle is forwarded to the node that is the sole
member of its destination endpoint; the service or demux component of the
destination EID selects an application at that node, after forwarding is
complete, and plays no part in it. Routes therefore match nodes:

- For the `ipn` scheme, a node is identified by its Fully Qualified Node
  Number (FQNN), the Allocator Identifier and Node Number together. All
  `ipn` EIDs that share an FQNN refer to the same node
  ({{Section 5.1 of RFC9758}}), so an `ipn` route pattern specifies
  Allocator Identifiers and Node Numbers only, and matches every Service
  Number.
- For the `dtn` scheme, a node is identified by the node-name in the
  authority of its EIDs ({{Section 4.2.5.1.1 of RFC9171}}), so a `dtn`
  route pattern specifies an authority only, and matches every demux.

In the text form of {{I-D.ietf-dtn-eid-pattern}}, an `ipn` route pattern
always ends in `.*` (any Service Number), and a `dtn` route pattern always
ends in `/**` (any demux). The protocol encoding ({{data-types}}) omits
these components, since they carry no information.

Whether a particular service is offered at a node, and whether a gateway
will accept bundles for it, are matters of application registration and
of local policy at the receiving AD, not of reachability, and are outside
the scope of DPP.

A node may have EIDs in more than one scheme, for example an `ipn` EID and
a `dtn` EID. The Bundle Protocol does not define any relationship between
such EIDs, and nor does DPP: routes for the two schemes are independent,
and a bundle is forwarded using routes for the scheme of its destination
EID. Only routes for singleton endpoints are advertised; non-singleton
endpoints ({{Section 4.2.5.1.1 of RFC9171}}) are outside the scope of this
document.

## EID Pattern Constraints (Monotonic Specificity) {#pattern-constraints}

Route patterns are EID Patterns: `ipn` patterns as defined in
{{I-D.ietf-dtn-eid-pattern}}, and `dtn` patterns as defined in
{{I-D.taylor-dtn-eid-pattern-dtn}}, which supplies the scheme-specific
`dtn` pattern that {{I-D.ietf-dtn-eid-pattern}} leaves to be defined by
other documents. To keep specificity comparison and longest-match lookup
well-defined, DPP restricts both to a Strict Monotonic Subset.

A pattern exhibits monotonic specificity if all wildcard components are
confined to the logical "leaves" of the naming hierarchy. A specific child
component cannot exist under a wildcard parent. Equivalently, when a pattern
is read in hierarchy order, the wildcard (if present) MUST be the terminal
element: no specific component, label, or character may follow it. This is
what guarantees that any two patterns matching the same EID are nested,
keeping both specificity comparison and longest-prefix lookup well-defined.

### `ipn` Scheme Constraints (Left-to-Right Hierarchy) {#ipn-constraints}

The `ipn` route pattern hierarchy is Allocator Identifier -> Node Number
{{RFC9758}}; the Service Number is always a wildcard
({{routes-identify-nodes}}). If the Allocator Identifier is not specific
(i.e., is a wildcard or range), the Node Number MUST be a wildcard.

A range MUST consist of a single interval whose size is a power of two,
greater than one, and whose lower bound is a multiple of that size (i.e.,
it is equivalent to a bit prefix, in the same way as an Allocator
Identifier range in {{RFC9758}}). A range of size one MUST be expressed as
a specific value. Arbitrary ranges are not permitted, because two
overlapping ranges (e.g., `[10-13]` and `[12-15]`) can match the same EID
without either containing the other.

- Valid: `ipn:100.1.*` (Specific Node), `ipn:100.[12-15].*` (Aligned Node
  Range), `ipn:100.*.*` (Node Wildcard), `ipn:**` (Default Route,
  equivalent to `ipn:*.*.*`).
- Invalid: `ipn:100.1.5` (Specific Service; not a route pattern),
  `ipn:*.1.*` (Specific Node under Wildcard Allocator),
  `ipn:100.[10-13].*` (Unaligned Range), `ipn:100.[0-3,8-11].*` (Multiple
  Intervals).

### `dtn` Scheme Constraints {#dtn-constraints}

A `dtn` route pattern is a `dtn` EID Pattern
({{I-D.taylor-dtn-eid-pattern-dtn}}) whose demux is `**`
({{routes-identify-nodes}}) and whose authority is either exact or
contains a single wildcard that, in the hierarchy order defined by that
document, is the terminal element. Which authority patterns satisfy this
condition is determined by {{I-D.taylor-dtn-eid-pattern-dtn}}; this
document imposes only the monotonic requirement above. An advertisement
containing a `dtn` pattern that does not satisfy it is malformed
({{malformed}}).

For example, `dtn://rover1.example.org/**` (a single node) and
`dtn://*.example.org/**` (every node whose authority ends in
`.example.org`) are route patterns, and `dtn://rover1.example.org/svc` is
not, since it names a demux.

## Specificity Scoring {#specificity}

The specificity score prioritizes Exact Matches over Wildcards, and Longer
Prefixes over Shorter Prefixes.

> The specificity score is computed locally by the receiver from the EidPattern.
> It is NOT transmitted in route advertisements, as it can be deterministically
> derived from the pattern itself.

The score is calculated as a `uint32`:

~~~
Score = (IsExact x 256) + LiteralLength
~~~

IsExact (0 or 1):
: 1 if the pattern identifies exactly one node, i.e., its Allocator
  Identifier and Node Number (`ipn`) or its authority (`dtn`) contain no
  wildcard or range; 0 otherwise.

Multiplier (256):
: A weight ensuring a pattern for a single node outranks any wildcard
  pattern matching that node. For `ipn` patterns the maximum LiteralLength
  is 64 (2 x 32 bits), which is well below 256. For `dtn` patterns
  LiteralLength can exceed 256, but a single-node pattern always has at
  least as many literal characters as any wildcard pattern that matches
  the same node, so the ordering is preserved.

LiteralLength:
: The count of non-wildcard characters (or equivalent bits for integers).

### `dtn` Scheme Scoring

For `dtn` patterns, LiteralLength is the number of characters in the
authority, in the text form of {{I-D.taylor-dtn-eid-pattern-dtn}},
excluding wildcard characters. The demux is always `**` and is not
counted.

| Pattern | Characters | Score |
|---------|------------|-------|
| `dtn://rover1.example.org/**` | 18 (single node) | 274 |
| `dtn://*.example.org/**` | 12 (wildcard) | 12 |

### `ipn` Scheme Scoring {#ipn-scoring}

For `ipn` patterns, LiteralLength represents the "Effective Bit Depth" of
the pattern. The FQNN space is treated as a 64-bit integer composed of two
32-bit components, the Allocator Identifier and the Node Number, which
{{RFC9758}} limits to 32 bits each. The Service Number does not
contribute.

~~~
LiteralLength = Bits_Allocator + Bits_Node
~~~

Each component contributes:

- Specific value (e.g., `100`): 32 bits.
- Wildcard (`*`): 0 bits.
- Range `[min-max]`: 32 - log2(Count) bits, where Count is the number of
  values in the range (a power of two).

| Pattern | Classification | IsExact | LiteralLength | Score |
|-------------|----------------|---------|---------------|-------|
| `ipn:100.1.*` | Single Node | 1 | 64 | 320 |
| `ipn:100.[12-15].*` | Node Range (Size 4) | 0 | 62 | 62 |
| `ipn:100.*.*` | Node Wildcard | 0 | 32 | 32 |
| `ipn:**` | Default Route | 0 | 0 | 0 |

## The Metric Field {#metric}

The `metric` field in route advertisements is analogous to BGP's Multi-Exit
Discriminator (MED) {{RFC4271}}. It allows an Administrative Domain to express
a preference for how traffic should enter its network when multiple paths
exist.

For example:

~~~
Organization B has two DPP speakers peering with Organization A:
  - Speaker in Europe  (gateway with high latency to Mars relay)
  - Speaker in Australia (gateway with low latency to Mars relay)

Organization B advertises routes to ipn:200.*.* via both speakers:
  - Europe speaker:    metric = 100
  - Australia speaker: metric = 10

Organization A, having learned the same route from both peers,
prefers the lower metric and routes traffic via Australia.

If Organization A instead learns the Australia route only
indirectly, via a third AD, it still prefers Australia over the
direct Europe route: both routes originate in Organization B, and
once Organization B has been chosen as the destination AD, its
metric is applied regardless of AD_PATH length.
~~~

| Property | Description |
|----------|-------------|
| Direction | Controls incoming traffic (traffic TO your AD) |
| Scope | Only compared between routes from the SAME origin AD |
| Preference | Lower metric = more preferred |
| Use Cases | Capacity balancing, latency optimization, cost management |

The metric is set by the originating AD and propagated unchanged through
intermediate ADs: a DPP speaker that re-advertises a route MUST NOT modify
its metric. It lets an AD steer incoming traffic without coordination with
other ADs.

The metric expresses the originating AD's preference between ways into its
own network. It does not express a preference between different
originating ADs, and is not used to choose between them (see
{{route-selection}}).

This is a notable departure from the BGP MED, which is non-transitive: a
MED received from a neighboring AS is not propagated to other neighboring
ASes, and MEDs are compared only between routes received from the same
neighboring AS ({{Section 5.1.4 of RFC4271}} and
{{Section 9.1.2.2 of RFC4271}}; see also {{RFC4451}}). The
DPP metric is transitive, and is compared between routes with the same
originating AD, regardless of the neighbor from which they were learned.

## Route Selection Algorithm {#route-selection}

When a DPP speaker has multiple routes whose patterns match the same EID, it
MUST select the best path in two stages: it first chooses the originating
AD (the last entry in AD_PATH) to which bundles for that EID will be sent,
and then chooses among the routes to that AD. The process is specified as
a sequence of steps for clarity; an implementation MAY compute the result
by any means that produces the same outcome. The reasons for the order of
the steps are given in {{selection-rationale}}.

Routes that a speaker originates for nodes within its own AD take part in
selection alongside routes received from peers. For this purpose an
originated route has an AD_PATH containing only the speaker's own AD, the
metric and other attributes that the speaker would advertise for it, and
is available from when it is configured.

A pattern may legitimately be originated by more than one AD. For example,
a spacecraft may be reachable through the ground stations of several
agencies, each of which originates routes to it. This is expected in DTN,
and the selection process is designed for it.

Route selection is performed for a time t: the time at which a bundle is,
or would be, accepted. A speaker forwarding a bundle that it holds selects
a route for the current time; a speaker deciding what to re-advertise
selects routes for each time within the windows that it advertises
({{readvertisement}}). Only routes that have not expired by time t are
considered, including routes that are not yet available at time t: in a
store-and-forward network, a bundle accepted at time t can be held until
such a route becomes available. A route is available at time t if it has
no `valid_from` attribute, or its `valid_from` time is not later than t.
The outcome of selection for the current time changes only at the events
listed in {{time-variant}}, and a speaker needs to repeat selection only
when one of them occurs.

Stage 1, choosing the originating AD:

1. Specificity: Keep only the routes with the highest Specificity Score
   ({{specificity}}). For example, prefer `dtn://rover1.example.org/**`
   over `dtn://*.example.org/**`.

2. Local preference: If local policy assigns preferences to originating
   ADs, or to the peers from which routes were learned (for example, to
   reflect cost, contractual arrangements, or trust), keep only the routes
   with the highest preference. A speaker with no such policy treats all
   routes as equally preferred.

3. Earliest delivery: For each route, take its estimated delivery time to
   be its `arrival_time` if it has one, and otherwise the time at which it
   becomes available, given by its `valid_from` attribute
   ({{time-variant}}); in either case, if that time is earlier than t,
   take it to be t. For each originating AD among the remaining routes,
   find the earliest estimated delivery time among its routes. Keep only
   the ADs for which this time is earliest.

4. Shortest path to each origin: For each remaining AD, find the length of
   the shortest AD_PATH among its routes whose estimated delivery time is
   the time found in step 3. Keep only the ADs for which this length is
   smallest.

5. Tie-break between origins: If more than one AD remains, choose the AD
   with the longest-held route among the routes considered in step 4; if
   that does not decide, choose the AD whose name sorts first in a
   case-insensitive comparison. A route's age is the time since the
   advertisement that created it was first received on its session;
   replacing an advertisement with the same ID does not reset the age of
   routes whose pattern it retains.

Let T be the earliest time, not earlier than t, at which any of the chosen
AD's routes considered in step 4 is available. Stage 2, choosing a route to
that AD, from among the routes of the AD chosen in stage 1 that remain
after steps 1 and 2 and are available at time T:

{: start="6"}
6. Lowest metric: Keep only the routes with the lowest metric
   ({{metric}}), regardless of the length of their AD_PATHs.

7. Earliest arrival: Keep only the routes with the earliest
   `arrival_time`. Routes without an `arrival_time` attribute are
   considered only if no remaining route has one.

8. Latest expiry: Keep only the routes that remain valid the longest, as
   given by their `valid_until` attributes ({{time-variant}}). A route
   with no `valid_until` attribute is treated as remaining valid
   indefinitely.

9. Shortest AD_PATH: Keep only the routes with the fewest entries in
   AD_PATH.

10. Oldest route: If more than one route remains, choose the one that has
    been held the longest, for stability. If more than one route remains,
    choose the one with the lower advertisement ID, and then the one
    received over the session whose peer's public key has the numerically
    smaller SHA-256 digest.

Treating a route without an `arrival_time` as delivering bundles as soon as
it accepts them is optimistic, and so favors routes that omit the
attribute. For this reason, a speaker that re-advertises a route that
carries an `arrival_time` is required to include one ({{readvertisement}}),
and an originating AD that can estimate when it will deliver bundles
SHOULD include an `arrival_time` in the routes it originates.

Selecting a route by this process is meaningful only if each AD along the
path forwards bundles according to the route that it advertised. A DPP
speaker therefore MUST advertise to its peers, for each pattern and each
time, only a route derived from the route that it has selected for that
time ({{readvertisement}}), and MUST NOT advertise a route that it has not
selected.

A DPP speaker MAY implement additional local policy that further modifies
this process, but such policy is outside the scope of this specification.


# Re-advertising Routes {#readvertisement}

When a DPP speaker re-advertises a route that it learned from a peer, it
offers its own AD as a transit path to the route's destinations. Because
DTN is a store-and-forward network, most route attributes describe what the
advertising AD offers (when it accepts bundles, and which bundles it
accepts), rather than properties of the whole path; only AD_PATH, the
metric, and the arrival time describe the path beyond the advertising AD.

Route selection depends on time ({{route-selection}}): the route that a
speaker selects for a pattern can differ for bundles that it accepts at
different times, as routes become available and expire. For each pattern,
a speaker that re-advertises routes therefore advertises a series of
routes with non-overlapping time windows, such that, for any time t within
one of those windows, the advertised route is derived from the route that
the speaker selects for time t. If the selected route changes at some time
within a window that the speaker would otherwise advertise, the speaker
splits that window at that time, and advertises each part with attributes
derived from the route selected for it. A speaker MUST NOT advertise a
route for a pattern for any time at which it has no route selected for
that pattern.

Each such window is advertised as a separate advertisement, with its own
advertisement ID ({{advertisement-id}}), so that the speaker can update or
withdraw it independently when the routes it has selected change. The
speaker MAY combine several patterns into one advertisement if they share
the same time window and attributes.

When deriving a re-advertised route from a selected route, the speaker
MUST update its attributes as follows:

AD_PATH:
: The speaker MUST prepend its own `local_ad_id` ({{loop-detection}}).

Metric:
: The speaker MUST NOT modify the metric ({{metric}}).

Gateway EID:
: The `gateway_eid` attribute identifies a gateway in the AD that sent the
  advertisement, which is the next hop for bundles matching the route
  ({{gateway-eid-resolution}}). When the speaker re-advertises the route,
  its own AD becomes that next hop. The speaker MUST NOT propagate the
  `gateway_eid` it received. It MUST either include a `gateway_eid` giving
  the administrative endpoint EID of a gateway in its own AD, or omit the
  attribute, so that the receiving speaker derives the default gateway EID
  from the re-advertising speaker's AD identity.

Time window:
: The `valid_from` and `valid_until` attributes describe when the
  re-advertising speaker's own AD will accept bundles for the route's
  destinations ({{time-variant}}). The speaker MUST set them to reflect
  that, and not simply copy the values it received. Because its AD can
  store bundles until they can be forwarded, the window it advertises MAY
  begin earlier than the window it received, provided that its AD can hold
  bundles accepted before the received window opens until it does. The
  window it advertises MUST NOT end later than the window it received,
  since bundles accepted after the received route expires cannot be
  forwarded over it. A single advertised window that spans several received
  routes for the same pattern arises not by extending any one of them, but
  from the derivation described above: for each time within the advertised
  window, the advertised route is derived from the route selected for that
  time, which may be a route that is not yet available.

Arrival time:
: If the received advertisement includes an `arrival_time` attribute, the
  speaker MUST include an `arrival_time` giving its own estimate of the
  earliest time at which a bundle accepted at the start of its advertised
  window can be delivered to the destination. That estimate MUST NOT be
  earlier than the received arrival time, since a bundle cannot reach the
  destination via the next AD sooner than that AD can deliver it. It will
  typically be later, reflecting the time needed to forward the bundle to
  the next AD, including any time for which the bundle must be stored. If
  the received advertisement does not include an `arrival_time`, the
  speaker MUST NOT add one.

Bundle size:
: The `max_bundle_size` attribute gives the largest bundle that the
  re-advertising speaker's AD will accept. The speaker MUST set it to
  reflect that, and not simply copy the value it received. It MAY be
  larger than the received value if the AD can deliver larger bundles to
  the next AD, for example by fragmenting them ({{Section 5.8 of RFC9171}})
  or by encapsulating them using Bundle-in-Bundle Encapsulation
  {{I-D.ietf-dtn-bibe}} and fragmenting the encapsulating bundle. If the
  AD can do neither, the advertised value MUST NOT be larger than the
  received value.

Bandwidth:
: The `bandwidth_bps` attribute gives the rate at which the re-advertising
  speaker's AD accepts bundles for the route's destinations. The speaker
  MUST set it to reflect that, and not simply copy the value it received.
  Because the AD can store bundles, this rate is not limited by the rate of
  later parts of the path, although sustained traffic in excess of those
  rates will eventually exhaust the AD's storage.

Unknown attributes:
: The speaker MUST treat attributes that it does not understand as
  described in {{forward-compatibility}}. The definition of any new route
  attribute MUST specify how the attribute is treated when a route is
  re-advertised.


# Security Considerations

## AD Authentication

DPP authenticates Administrative Domains using mutual TLS and TLSA records
published in each AD's DNS zone ({{trust}}). The
security of this mechanism depends on:

DNSSEC:
: The TLSA records are only as trustworthy as the DNSSEC chain that
  protects them. For this reason, DPP requires the TLSA RRset to be
  validated as secure ({{handshake}}), and an AD that cannot sign its zone
  cannot be authenticated. An attacker who can forge a secure DNSSEC
  response for an AD's zone, for example by compromising the zone's
  signing keys or those of a parent zone, can impersonate that AD.

Private Key Protection:
: The private key of each speaker authorized by a TLSA record MUST be
  protected. Compromise of a key allows impersonation of the Administrative
  Domain until its TLSA record is removed and the record's DNS TTL and
  signature validity have expired. Speakers that periodically re-verify
  their peers' authorization (see {{handshake}}) limit the lifetime of an
  established session using a revoked key.

Channel binding:
: Because the key authorized in DNS is the key that authenticates the TLS
  connection, an attacker cannot relay one speaker's authentication to a
  different session. An application-layer proof of key possession, such as
  a signed nonce, would not have this property unless it were explicitly
  bound to the TLS session.

DNS lookups triggered by unauthenticated peers:
: The Responder looks up TLSA records for an AD named by a peer whose
  authorization has not yet been verified. Restricting lookups to ADs with
  which the Responder is configured to peer ({{handshake}}) prevents a
  peer from causing lookups of arbitrary names.

Misdirected SVCB records:
: An SVCB record at `_dpp.<AD-Domain>` can direct an Initiator to any
  host and port. Because the resulting TLS connection must be
  authenticated against the AD's TLSA records before any DPP message is
  acted on, a forged or misconfigured SVCB record can cause connection
  attempts to a host of the attacker's choosing, but cannot produce an
  authenticated session with it. Initiators SHOULD apply the same
  restrictions on destination ports to SVCB-derived addresses as to
  configured ones ({{Section 12 of RFC9460}}).

## Transport Security

All DPP sessions use TLS 1.3 {{RFC8446}} or later, with both speakers
authenticated, as specified in {{handshake}}. This protects the
confidentiality and integrity of route advertisements in transit, and
ensures that every message on a session comes from the authenticated peer.

## Route Injection Attacks {#route-injection}

A malicious or compromised DPP speaker could advertise routes for destinations
it does not legitimately serve. Implementations SHOULD support policy-based
route filtering, so that operators can restrict the patterns accepted from
each peer, and SHOULD provide the means to monitor route changes. Operators
are advised to confirm, through channels outside DPP, that a peer is
entitled to advertise routes to destinations that matter to them.

Specificity is the first step of route selection, ahead of local
preference ({{route-selection}}). A malicious AD can therefore attract
traffic for part of another AD's destinations by advertising a more
specific pattern than the legitimate originating AD does: for example,
`ipn:100.5.*` within another AD's `ipn:100.*.*`, or
`dtn://rover1.example.org/**` within another AD's
`dtn://*.example.org/**`. Such a route is preferred for the EIDs that it
matches, whatever its local preference, delivery time, or path length, so
none of the later selection steps limits its effect. This is the DPP
equivalent of a BGP sub-prefix hijack. Until origin authorization is
available (see below), the principal defense is route filtering: operators
SHOULD configure, for each peer, the patterns that they are prepared to
accept from that peer, and SHOULD reject more specific patterns than they
expect.

The AD_PATH verification in {{loop-detection}} ensures that the first entry
in AD_PATH is the authenticated peer, but a receiving speaker cannot verify
the remaining entries: a malicious peer can still add or remove ADs further
along the path, including the originating AD.

Similarly, the metric is not protected. A malicious transit AD could lower
the metric of routes it re-advertises, in violation of {{metric}}, and so
attract traffic for the originating AD, even over a longer path: once that
AD has been chosen as the destination, the metric takes precedence over
AD_PATH length ({{route-selection}}). Because metrics are not used to
choose between originating ADs, such manipulation cannot divert traffic
from one originating AD to another, but it can divert traffic between
routes into the same AD. Operators that are concerned by this can apply
local preference to the peers from which they learn routes, or ignore the
metric on routes from untrusted peers.

The time window, arrival time, bundle size, and bandwidth attributes are
not protected either. Because estimated delivery time is used to choose
between originating ADs ({{route-selection}}), a malicious transit AD that
advertises an `arrival_time` earlier than the one it received, in
violation of {{readvertisement}}, or that omits the `arrival_time` it
received, can attract traffic away from other originating ADs, not just
between routes into the same AD. Local preference, which is applied before
estimated delivery time, can be used to limit the influence of untrusted
peers.

The time window, bundle size, and bandwidth attributes describe what the
advertising AD itself offers, and a transit AD can legitimately advertise
values that differ from those it received, in either direction
({{readvertisement}}). A receiving speaker therefore cannot detect a false
value by comparing it with the values advertised by other ADs on the path.
An AD that overstates what it can store or forward can attract bundles
that it then fails to deliver; the consequences are similar to those of
any other AD that accepts bundles and discards them.

Mechanisms that could address these threats, analogous to those used to
secure BGP, are discussed in {{future-security}}.

## Denial of Service {#denial-of-service}

DPP speakers are potentially vulnerable to denial of service attacks:

Connection exhaustion:
: Implementations SHOULD limit the number of concurrent sessions and
  implement rate limiting for new connections.

Route table exhaustion:
: Implementations SHOULD limit the number of routes accepted from a single
  peer, and close the session with code `MAX_ROUTES_EXCEEDED` when the
  limit is exceeded ({{states}}).

Keepalive flooding:
: Implementations SHOULD rate-limit incoming messages.

## Privacy Considerations

Route advertisements reveal network topology and reachability information.
Operators should consider whether this information is sensitive before
establishing peering relationships.


# IANA Considerations

## Underscore-Prefixed DNS Name {#iana-underscore}

This document requests registration of `_dpp` in the "Underscored and
Globally Scoped DNS Node Names" registry per {{RFC8552}}, for both RR types
used at that name:

| RR Type | _NODE NAME | Reference |
|---------|------------|-----------|
| SVCB | _dpp | This document |
| TLSA | _dpp | This document |

The SVCB entry satisfies the requirement of {{Section 14.3 of RFC9460}}
that a protocol mapping be accompanied by such a registration; the mapping
itself is in {{svcb-mapping}}.

## DPP Notification Codes Registry {#iana-codes}

This document requests the creation of a "DPP Notification Codes"
registry. The registration policy is Specification Required
({{Section 4.6 of RFC8126}}). Each entry consists of a numeric value, a
name, whether the code ends the session, and a reference. The initial
contents are the values defined in {{control-messages}}, with this
document as the reference. Values 0 through 10 are assigned by this
document; values 11 through 65535 are available for assignment.

## DPP Capabilities Registry {#iana-capabilities}

This document requests the creation of a "DPP Capabilities" registry of
names that may appear in the `capabilities` field of the `Hello` message
({{identity-message}}). The registration policy is Specification Required
({{Section 4.6 of RFC8126}}). Each entry consists of a name, which MUST
consist of lowercase ASCII letters, digits, and hyphens, and a reference.
This document registers no names.

## Future Considerations {#iana-future}

This document does not request a port assignment. DPP uses port 50051 by
default ({{svcb-mapping}}), which is the port conventionally used by gRPC
services and is not assigned to any protocol in the "Service Name and
Transport Protocol Port Number Registry". Should a port be assigned for
DPP by a later document, it would become the default.

Future versions may additionally request establishment of a registry for
DPP route attribute types.


--- back

# Relationship to DNS-Based Convergence Layer Discovery {#cl-discovery}

This approach aligns with {{I-D.ek-dtn-ipn-arpa}}, which uses DNS for
convergence layer endpoint discovery (see also {{gateway-eid-resolution}}).
The two use separate parts of the DNS namespace because they name
different things. Names beneath `ipn.arpa` identify BP nodes, the
destinations and next hops of bundles, and are administered by the
Allocators that assign their Node Numbers. DPP speakers are not
necessarily BP nodes, and act for ADs rather than for Allocators: an AD
may hold several Allocator Identifiers, or none. DPP speakers are
therefore identified, authorized, and located under the AD's own domain
name, at `_dpp.<AD-Domain>`, and not beneath `ipn.arpa`. Because the
`_dpp` label is specific to DPP, it does not conflict with
the names used for convergence layer discovery (e.g., `_dtn-bundle._tcp`),
or with those that future specifications may define for other DTN
services.

DNS discovery under `ipn.arpa` and DPP are complementary sources of
forwarding information. An Allocator can publish, for each of its nodes,
the convergence layer endpoints of an ingress service that accepts bundles
for that node, as in the examples of {{I-D.ek-dtn-ipn-arpa}}, giving other
networks a simple, static means of reaching the node without peering. DPP
provides what such discovery does not: reachability that varies with time,
transit through intermediate ADs, and selection between several ADs
through which the same destination can be reached. How a BP agent combines
forwarding information from the two sources is a matter of local
forwarding policy, as {{I-D.ek-dtn-ipn-arpa}} also notes.


# Design Rationale for Route Selection {#selection-rationale}

This appendix explains the choices made in {{route-selection}}. It is not
normative.

Estimated delivery time is compared before path length because, in a DTN,
when a bundle can be delivered typically matters far more than the number
of ADs that it traverses. Arrival time is used where it is known, because
in a store-and-forward network the time at which an AD will accept a
bundle can be much earlier than the time at which the bundle will be
delivered. Where no arrival time is known, the time at which a route
becomes available is used instead; for routes that are already available,
step 3 then has no effect, and the choice between ADs is made on path
length.

Stage 2 considers only those routes to the chosen AD that are available at
the time at which the route for which it was chosen becomes available. The
originating AD's metric therefore chooses between entry points into that
AD that are available at that time, even if the entry point it prefers has
a later arrival time; as with path length, this is the originating AD's
decision. The metric cannot delay traffic to wait for a preferred entry
point that becomes available later. Among routes of equal metric, the one
that delivers bundles soonest is preferred, and then the one that remains
valid the longest, reducing both the risk that a route expires before
bundles are forwarded and the frequency with which the selection changes.

Metrics are not consulted in stage 1, and are compared only between routes
with the same originating AD. Choosing the originating AD before
considering metrics means that the outcome does not depend on the order in
which routes are compared. It also means that a new route originated by an
AD can only make that AD more likely to be chosen (if its estimated
delivery time is earlier, or the same over a shorter AD_PATH), or change
the route by which that AD is reached; it can never cause some other AD to
be chosen instead.

As a result, the selected route is not necessarily the shortest or the
fastest route available. The originating AD's metric can direct traffic
over a longer or slower path into that AD, and the AD so chosen can be
reached over a longer path than some route to a different AD. This is
intended: it allows the originating AD to control how traffic enters its
network, while the choice between ADs is made on local policy, estimated
delivery time, and path length alone.

This process differs from the BGP decision process
({{Section 9.1.2.2 of RFC4271}}), which compares AS_PATH length before the
MED, and compares MEDs only between routes received from the same
neighboring AS. Because that comparison is not a total order, the BGP
decision can depend on the order in which routes are compared, which can
cause persistent route oscillation {{RFC3345}}. Choosing the originating AD
first, without reference to metrics, avoids this.


# Future Route Security Mechanisms {#future-security}

This appendix outlines mechanisms, analogous to those used to secure BGP,
that could address the route injection threats described in
{{route-injection}}. None is specified here; they are recorded to inform
future work. BGP relies on the Resource Public Key Infrastructure (RPKI)
{{RFC6480}}, a certificate hierarchy that follows the allocation of IP
addresses and AS numbers; DPP uses neither, and DNS, signed with DNSSEC,
already provides a suitable hierarchy, as it does for the authorization of
DPP speakers ({{tlsa-records}}).

Origin validation:
: Comparable to BGP Route Origin Validation {{RFC6811}}, the holder of a
  namespace could publish in DNS which ADs are entitled to originate
  patterns within it, and a receiving speaker could check the originating
  AD in AD_PATH against that list, which would also address the
  sub-pattern hijack described above.

  For `ipn` patterns, the allocation model of {{RFC9758}} provides the
  necessary hierarchy. IANA assigns Allocator Identifiers, and each
  Allocator assigns Node Numbers within its own space, much as IP address
  space is delegated from IANA to Regional Internet Registries and then to
  network operators. The `ipn.arpa` zone proposed in {{I-D.ek-dtn-ipn-arpa}}
  mirrors this hierarchy in DNS: a zone for each Allocator Identifier, at
  `A.ipn.arpa.`, would be delegated to the nameservers of the Allocator
  that holds it, and can be linked into the DNSSEC chain of trust of
  `ipn.arpa`. An Allocator could publish in its zone the ADs entitled to
  originate routes for its Node Numbers, much as a Route Origin
  Authorization in the RPKI does for a range of IP addresses; such
  authorizations would need to be signed, so the Allocator's zone would
  need to be signed. Because `ipn.arpa` represents each Node Number as a
  single decimal label, ranges of Node Numbers do not correspond to DNS
  subtrees, so authorizations for ranges would be published as data at
  the Allocator's name rather than beneath it. The range constraints of
  {{pattern-constraints}}, under which every range corresponds to a bit
  prefix, would allow these authorizations to be expressed compactly and
  aligned with the patterns that DPP carries. The record formats would
  need to be defined.

  {{I-D.ek-dtn-ipn-arpa}} excludes the Default Allocator (Allocator
  Identifier zero) and the Experimental Use Allocator Identifiers from
  publication in the global DNS. Origin validation based on `ipn.arpa`
  would therefore not be available for patterns within those Allocators'
  spaces, which include the Node Numbers that the Default Allocator has
  assigned to existing deployments. Other means, such as route filtering,
  would continue to be needed for them.

  For `dtn` patterns, the holder of the DNS name in the pattern's
  authority could publish this information under that name.

  Unlike route origin validation in BGP, more than one AD would commonly
  be authorized for a pattern, since a destination may legitimately be
  reachable through several ADs. As with the guidance on the use of
  maxLength in the RPKI {{RFC9319}}, authorizations would need to cover
  only the patterns that are actually advertised: an authorization that
  also covered unadvertised more specific patterns would allow a malicious
  AD to advertise them with a forged originating AD in its AD_PATH, a
  sub-pattern hijack that neither origin validation nor competing
  legitimate routes would prevent.

Neighbor authorization:
: Comparable to Autonomous System Provider Authorization
  {{I-D.ietf-sidrops-aspa-verification}}, each AD could publish, alongside
  its TLSA records, the ADs that it authorizes to re-advertise its routes.
  A receiving speaker could then check that each pair of adjacent ADs in
  AD_PATH is so authorized. This would detect most forged paths, including
  a malicious AD that appends a legitimate originating AD to its own
  path, without requiring routes to be signed.

Path signatures:
: Comparable to BGPsec {{RFC8205}}, each speaker could sign each route
  that it advertises, using the key authorized by its AD's TLSA records,
  so that a receiving speaker could verify every entry in AD_PATH. Unlike
  BGPsec, such signatures could also cover the metric, which is not
  modified in transit, and the arrival time advertised at each hop, so
  that a receiving speaker could check that no transit AD advanced it.

Any of these mechanisms would require DPP speakers to look up DNS records
while processing routes, rather than only during session establishment
({{handshake}}). Their design would therefore need to consider the volume
of such lookups, the caching of their results, and how a failure to
obtain or validate DNS records is treated. In particular, an attacker
able to block DNSSEC validation of the records for a single pattern must
not thereby be able to cause a route that would otherwise be rejected to
be accepted.


# Acknowledgments
{:numbered="false"}

This document was developed as part of the Hardy BPv7 DTN Router project.

The author thanks Erik Kline and Adam Wiethuechter for their reviews and
comments.
