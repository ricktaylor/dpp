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
  RFC6698:
  RFC7671:
  RFC8446:
  RFC8552:
  RFC9171:
  RFC9364:
  RFC9460:
  RFC9758:
  I-D.ietf-dtn-eid-pattern:
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
  RFC6480:
  RFC6811:
  RFC7911:
  RFC9319:
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
Administrative Domains (ADs), enabling inter-domain routing across the Solar
System DTN. DPP separates the control plane from the data plane: a DPP
speaker need not be a gateway that forwards bundles, allowing centralized
route controllers or orchestration systems to participate in peering on
behalf of the gateways they manage.

DPP harmonizes the two DTN addressing schemes -- `ipn` (integer-based) and
`dtn` (name-based) -- into a unified routing framework. DPP speakers
authenticate each other using mutual TLS, with each Administrative Domain
authorizing its speakers' keys through DNSSEC-signed TLSA records. DPP
supports both persistent routes and routes tied to the scheduled contact
windows of deep-space networks.


--- middle

# Introduction

The DTN Peering Protocol (DPP) is designed for the exchange of reachability
information between distinct Administrative Domains in Delay-Tolerant Networks.
It functions as the inter-domain routing protocol for the Solar System DTN,
analogous to the role of BGP {{RFC4271}} in the Internet, and complements,
rather than replaces, the routing that each Administrative Domain uses
internally.

## Motivation

Two DPP speakers establish a session to exchange reachability information
across administrative boundaries. Common scenarios include:

Inter-Agency Connectivity:
: NASA's Deep Space Network and ESA's ESTRACK operate independent DTN
  infrastructures. DPP allows their border gateways to advertise which
  spacecraft and ground stations are reachable through each network, enabling
  bundles to flow between agencies.

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
: DPP plays the role of an exterior gateway protocol: like BGP between
  Autonomous Systems, it carries routing information between ADs, and
  leaves each AD free to choose how it routes bundles internally, just as
  each Autonomous System chooses its own interior gateway protocol. One AD
  might use Contact Graph Routing (CGR), standardized by CCSDS as
  Schedule-Aware Bundle Routing (SABR) {{CCSDS-SABR}}; another might use
  static routes, or routes computed by its mission planning or network
  orchestration systems. DPP speakers advertise summarized reachability,
  with optional time windows, derived from whatever their AD uses
  internally, and peers need neither know nor share that mechanism. This
  separation also avoids any need for a single contact plan or contact
  graph spanning all agencies, spacecraft, and ground stations, which would
  be neither practical nor desirable: the cost of contact-based routing
  grows with network size, and each organization maintains its own mission
  planning systems and contact schedules.

Without DPP, each network would require manual configuration of routes to
external destinations -- an approach that does not scale as the Solar System
DTN grows.

## Design Principles

Transport Agnostic:
: Defined using Protocol Buffers {{PROTOBUF}} and carried over gRPC
  {{GRPC}}, allowing it to run over any reliable transport that gRPC
  supports (e.g., TCP or QUIC).

Unified Routing:
: Harmonizes `ipn` and `dtn` schemes into a single routing table (FIB).

DNS-Rooted Identity:
: Uses standard DNS domains as Administrative Domain identifiers, and the
  AD's own DNS zone to authorize the speakers that act on its behalf.

Decoupled Trust:
: Separates "Who I am" (AD Identity) from "What I route" (Allocators/Names).

## Scope

This specification defines the protocol messages, state machine, and
operational requirements for DPP implementations. It does not define:

- Routing within an Administrative Domain (e.g., CGR or static routing),
  which is chosen independently by each AD
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
: A border node within an Administrative Domain that forwards bundles to
  other ADs. A gateway may or may not also be a DPP speaker. When a DPP
  speaker is not itself a gateway, it uses the `gateway_eid` attribute
  (see {{gateway-eid-resolution}}) to direct traffic to the appropriate
  gateway.

EID Pattern:
: A pattern matching one or more Bundle Protocol {{RFC9171}} Endpoint
  Identifiers, as defined in {{I-D.ietf-dtn-eid-pattern}}.

Forwarding Information Base (FIB):
: The consolidated routing table used by a gateway to forward bundles.

Metric:
: A numeric value expressing route preference, analogous to BGP's Multi-Exit
  Discriminator (MED), with the differences described in {{metric}}.

AD_PATH:
: An ordered list of Administrative Domains that a route advertisement has
  traversed, used for loop detection and path length comparison.

## On the Use of DNS

Readers familiar with deep-space networking may question the reliance on DNS,
given that DNS is not deployed on spacecraft and requires low-latency
connectivity. This concern, while understandable, misses the architectural
context:

DPP speakers are ground infrastructure.
: DPP operates at administrative boundaries. DPP speakers are ground stations,
  mission control centers, network operations facilities, or centralized route
  controllers -- all of which have reliable Internet connectivity and can
  perform DNS lookups in milliseconds. Gateways, by contrast, are the
  forwarding nodes that handle bundle traffic and may be located anywhere --
  including on spacecraft or relay satellites -- since they do not need DNS
  access or direct participation in DPP.

DNS is used for authentication, not routing.
: DPP itself performs DNS lookups only during session establishment, to
  find a peer's DPP service and to retrieve the TLSA records that
  authorize the peer's key. Once a session is established, no further DNS
  queries are required. Route advertisements and withdrawals flow over the
  established session without DNS involvement.

Spacecraft do not run DPP.
: Spacecraft run the Bundle Protocol and may use CGR or static routing for
  internal forwarding decisions. They receive forwarding instructions from their
  ground-based mission operations, which participate in DPP on their behalf.
  The spacecraft need not know about Administrative Domains or DNS. However,
  spacecraft and relay satellites MAY serve as gateways -- forwarding bundles
  between Administrative Domains -- without participating in DPP directly.
  A ground-based DPP speaker advertises routes on their behalf using the
  `gateway_eid` attribute to identify the in-space forwarding node.

This architecture mirrors the Internet: BGP routers require IP connectivity to
establish sessions, but the end hosts they route traffic for need not
understand BGP. Similarly, DPP speakers require DNS access, but the spacecraft
and sensors they route bundles for need not understand DPP.


# Architecture Overview

## Administrative Domains

An Administrative Domain is an autonomous unit of routing control.

Identity:
: An AD is identified solely by a DNS domain name (e.g., `dsn.example.org`).

Independence:
: An AD Identity is distinct from the resources it advertises. An AD named
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
gateways within its AD, using the `gateway_eid` attribute to direct traffic
to the appropriate forwarding node. This separation of the control plane
(DPP) from the data plane (bundle forwarding) allows flexible deployment
topologies.

An Administrative Domain MAY deploy multiple DPP speakers, each maintaining
independent peering sessions with external ADs. Because loop detection
operates on the AD identity (the `local_ad_id` in the AD_PATH) rather than
on individual speaker identity, multiple speakers within the same AD do not
introduce routing loops (see {{loop-detection}}).


# Route Attributes and Advertisements {#routes}

Once a session is established, DPP speakers share reachability information
by sending route advertisements and withdrawals. A route advertisement
tells the peer that bundles destined for EIDs matching a set of EID
patterns can be delivered via the advertising AD, together with the
attributes of that route. The form of those patterns, and how a speaker
chooses between routes that match the same EID, are described in
{{addressing}}.

DPP uses an extensible attribute model inspired by BGP path attributes
{{RFC4271}}. This allows new route properties to be defined in future protocol
versions without breaking existing implementations.

## Advertisement Structure

A `RouteAdvertisement` follows the BGP UPDATE model:

Advertisement ID:
: An `id` that identifies the advertisement within the session, so that it
  can later be replaced or withdrawn ({{advertisement-id}})

Patterns:
: One or more EidPattern destinations (equivalent to NLRI), constrained as
  described in {{pattern-constraints}}

Path attributes:
: Properties shared by all destinations (ad_path, metric)

Extensible attributes:
: Optional properties (time window, arrival time, bundle size and rate,
  next-hop)

## Path Attributes

| Field | Description |
|-------|-------------|
| `ad_path` | List of AD domains traversed (for loop detection and path length; see {{loop-detection}}) |
| `metric` | Administrator preference (lower = preferred, analogous to BGP MED; see {{metric}}) |

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
  AD_PATH is empty, the receiving speaker MUST discard the advertisement and
  SHOULD send a `Notification` with level `WARNING`. This prevents a peer
  from omitting its own AD from the path, or from presenting a route as
  though it had been received directly from another AD.

Detection:
: If a DPP speaker receives a route where AD_PATH contains its own
  `local_ad_id`, the route MUST be silently discarded.

AD identities in AD_PATH are DNS domain names, and MUST be compared
case-insensitively.

## Extensible Attributes

Optional attributes shared by all destinations:

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

Implementations MUST handle unknown attributes as follows:

1. If `transitive = true`: Preserve and propagate the attribute unchanged.
2. If `transitive = false`: Silently discard the attribute.

This allows intermediate DPP speakers to forward routes with attributes they
do not understand, enabling incremental deployment of new attribute types.

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
treat the route as withdrawn. If an advertisement carries both attributes
and its `valid_until` time is not later than its `valid_from` time, the
receiving speaker MUST discard the advertisement.

The optional `arrival_time` attribute gives the advertising AD's estimate
of the earliest time at which a bundle that it accepts at the start of the
route's window can be delivered to the advertised destinations. Unlike the
time window, which describes only the advertising AD, the arrival time
describes the whole path to the destination. It is set by the originating
AD, if that AD can estimate when it will deliver bundles, and is updated by
each AD that re-advertises the route ({{readvertisement}}). A route without
an `arrival_time` attribute provides no estimate.

Because the availability of routes changes over time, a DPP speaker MUST
repeat route selection ({{route-selection}}) whenever it receives,
replaces, or loses a route, and whenever the `valid_from` or `valid_until`
time of a route it holds is reached, and MUST update its own
advertisements if the routes that it would advertise change
({{readvertisement}}).

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

This model is similar to the path identifiers that the BGP ADD-PATH
extension {{RFC7911}} introduced, which allow a BGP speaker to advertise
more than one path for the same prefix.

## Gateway EID Resolution {#gateway-eid-resolution}

The `gateway_eid` attribute specifies the endpoint where bundles matching the
advertised patterns should be forwarded. This enables separation of the
control plane (DPP session) from the data plane (bundle forwarding).

The gateway EID always identifies a gateway in the AD that sent the
advertisement, which is the next hop for matching bundles, not a gateway
in the originating AD. A speaker that re-advertises a route therefore
replaces the gateway EID with one of its own ({{readvertisement}}).

Resolution rules:

1. If a `gateway_eid` attribute is present, use it as the forwarding
   destination.

2. If `gateway_eid` is absent, derive the gateway EID from the peer's AD
   identity (`local_ad_id` from Hello) by converting the DNS domain name to
   the node ID of a `dtn` scheme EID {{RFC9171}}:

   ~~~
   <dns_name>  ->  dtn://<dns_name>/
   ~~~

   For example: `dsn.example.org` -> `dtn://dsn.example.org/`

No mapping from a domain name to an `ipn` node ID is defined: the
`ipn.arpa` zone proposed in {{I-D.ek-dtn-ipn-arpa}} maps an `ipn` node to
the records needed to reach it, but does not map a domain name to an `ipn`
node. An AD whose gateways are reachable only via `ipn` EIDs therefore needs
to include an explicit `gateway_eid` attribute.

Determining how to reach a gateway at the convergence layer, once its EID
is known, is a function of the receiving AD's BP agents rather than of
DPP, and can use any available means, including configuration. For a
gateway with an `ipn` EID, {{I-D.ek-dtn-ipn-arpa}} specifies a DNS-based
procedure: for example, TCPCLv4 {{RFC9174}} endpoints for the node
(978879,2) are found from SRV records at
`_dtn-bundle._tcp.2.978879.ipn.arpa.`. No corresponding procedure is
currently specified for gateways with `dtn` EIDs.

This design allows flexibility in deployment while providing sensible
defaults:

DNS-based AD:
: An AD with identity `dsn.example.org` automatically maps to
  `dtn://dsn.example.org/` as the default gateway. No explicit `gateway_eid`
  is required.

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

## Re-advertising Routes {#readvertisement}

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
  `gateway_eid` it received. It MUST either include a `gateway_eid` that
  identifies a gateway in its own AD, or omit the attribute, so that the
  receiving speaker derives the default gateway EID from the
  re-advertising speaker's AD identity.

Time window:
: The `valid_from` and `valid_until` attributes describe when the
  re-advertising speaker's own AD will accept bundles for the route's
  destinations ({{time-variant}}). The speaker MUST set them to reflect
  that, and not simply copy the values it received. Because its AD can
  store bundles until they can be forwarded, the window it advertises MAY
  begin earlier, end later, or both, than the window it received, provided
  that its AD can hold bundles accepted during the window until they can be
  forwarded within the received window. A speaker MUST NOT advertise a
  window during which its AD would accept bundles that it cannot expect to
  forward before the received route expires.

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


# Addressing and Metrics {#addressing}

This section describes the EID patterns carried in route advertisements
({{routes}}), and how a DPP speaker selects between routes whose patterns
match the same EID.

DPP introduces a Harmonized Specificity Score to allow integer-based `ipn`
patterns and string-based `dtn` patterns to coexist in a single routing
decision process.

## EID Pattern Constraints (Monotonic Specificity) {#pattern-constraints}

To ensure efficient FIB lookups and O(1) specificity scoring, DPP restricts the
generic EID patterns defined in {{I-D.ietf-dtn-eid-pattern}} to a Strict
Monotonic Subset.

{{I-D.ietf-dtn-eid-pattern}} defines a structured pattern syntax for the
`ipn` scheme, but for the `dtn` scheme it currently defines only the
any-SSP form (`dtn:**`), because the structure of a `dtn` authority is not
otherwise well-defined. The `dtn` pattern forms used in this document
(e.g., `dtn://*.example.org/**`) therefore go beyond
{{I-D.ietf-dtn-eid-pattern}}, and their precise syntax and matching rules
remain to be specified.

A pattern exhibits monotonic specificity if all wildcard components are
confined to the logical "leaves" of the naming hierarchy. A specific child
component cannot exist under a wildcard parent. Equivalently, when a pattern
is read in hierarchy order, the wildcard (if present) MUST be the terminal
element: no specific component, label, or character may follow it. This is
what guarantees that any two patterns matching the same EID are nested,
keeping both specificity comparison and longest-prefix lookup well-defined.

### `ipn` Scheme Constraints (Left-to-Right Hierarchy)

The `ipn` scheme hierarchy is defined as Allocator -> Node -> Service Number
{{RFC9758}}. If a component is not specific (i.e., is a wildcard or range),
every component to its right in the hierarchy MUST be a wildcard.

A range MUST consist of a single interval whose size is a power of two and
whose lower bound is a multiple of that size (i.e., it is equivalent to a
bit prefix, in the same way as an Allocator Identifier range in
{{RFC9758}}). Arbitrary ranges are not permitted, because two overlapping
ranges (e.g., `[10-13]` and `[12-15]`) can match the same EID without
either containing the other.

- Valid: `ipn:100.1.5` (Fully specific), `ipn:100.1.*` (Service Wildcard),
  `ipn:100.*.*` (Node Wildcard), `ipn:100.[12-15].*` (Aligned Node Range),
  `ipn:**` (Default Route, equivalent to `ipn:*.*.*`).
- Invalid: `ipn:*.1.*` (Specific Node under Wildcard Allocator),
  `ipn:100.*.5` (Specific Service under Wildcard Node),
  `ipn:100.[10-13].*` (Unaligned Range), `ipn:100.[0-3,8-11].*` (Multiple
  Intervals).

### `dtn` Scheme Constraints (Right-to-Left Hierarchy) {#dtn-constraints}

The `dtn` scheme hierarchy follows standard DNS rules: TLD <- Domain <- Host.

- Wildcards are only permitted in the left-most label (the Host or lowest
  subdomain), and only as the entire label or as its trailing suffix: the
  wildcard MUST be the last character of that label. A wildcard followed by
  literal characters does not exhibit monotonic specificity -- two such
  patterns can match the same EID without either containing the other.
- A pattern whose authority contains a wildcard MUST use `**` as its service
  path: a specific path under a wildcard host is a specific child under a
  wildcard parent.
- Valid: `dtn://rover1.example.org/svc` (Exact),
  `dtn://rover1.example.org/**` (Any Service on a Specific Host),
  `dtn://*.example.org/**` (Wildcard Host),
  `dtn://rover*.example.org/**` (Partial Host).
- Invalid: `dtn://rover1.*.example.org/**` (Specific Child under Wildcard
  Parent), `dtn://r*1.example.org/**` and `dtn://*rover.example.org/**`
  (Non-terminal Wildcard), `dtn://rover*.example.org/svc` (Specific Path
  under a Wildcard Host).

### Single Wildcard Constraint

To support the simplified specificity metric, a `dtn` pattern MUST contain
at most one wildcard character (`*`), not counting a `**` service path.
Complex globs (e.g., `dtn://r*v*r.example.org/**`) MUST NOT be used.

The `ipn` constraints above already limit an `ipn` pattern to at most one
range, followed only by wildcard components.

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
: 1 if the pattern matches exactly one EID (i.e., contains no wildcards or
  ranges); 0 otherwise.

Multiplier (256):
: A weight ensuring an Exact Match outranks any Wildcard Match of the same
  EID. For `ipn` patterns the maximum LiteralLength is 96 (3 x 32 bits),
  which is well below 256. For `dtn` patterns LiteralLength can exceed 256,
  but an exact pattern always has at least as many literal characters as
  any wildcard pattern that matches the same EID, so the ordering is
  preserved.

LiteralLength:
: The count of non-wildcard characters (or equivalent bits for integers).

### `dtn` Scheme Scoring

For `dtn` patterns, LiteralLength is the number of characters following
`dtn://` (authority and service path), excluding `*` characters.

| Pattern | Characters | Score |
|---------|------------|-------|
| `dtn://rover1.example.org/svc` | 22 (exact) | 278 |
| `dtn://rover1.example.org/**` | 19 (wildcard) | 19 |
| `dtn://rover*.example.org/**` | 18 (wildcard) | 18 |
| `dtn://*.example.org/**` | 13 (wildcard) | 13 |

### `ipn` Scheme Scoring {#ipn-scoring}

For `ipn` patterns, LiteralLength represents the "Effective Bit Depth" of
the pattern. The `ipn` address space is treated as a 96-bit virtual integer,
composed of three 32-bit components: Allocator, Node, and Service Number.
{{RFC9758}} limits Allocator Identifiers and Node Numbers to 32 bits. It
places no such limit on Service Numbers, but the IANA registry it
establishes reserves Service Numbers of 2^32 and above for future expansion
({{Section 9.3 of RFC9758}}). Should such Service Numbers come into use, the
scoring model and the `IpnPattern` encoding ({{data-types}}) would need to
be extended.

~~~
LiteralLength = Bits_Allocator + Bits_Node + Bits_Service
~~~

Each component contributes:

- Specific value (e.g., `100`): 32 bits.
- Wildcard (`*`): 0 bits.
- Range `[min-max]`: 32 - log2(Count) bits, where Count is the number of
  values in the range (a power of two).

| Pattern | Classification | IsExact | LiteralLength | Score |
|-------------|----------------|---------|---------------|-------|
| `ipn:100.1.5` | Specific EID | 1 | 96 | 352 |
| `ipn:100.1.*` | Service Wildcard | 0 | 64 | 64 |
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
its metric. It provides a policy knob for traffic engineering without
requiring coordination between ADs.

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
and then chooses among the routes to that AD.

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
Because the outcome changes only when a route is received, replaced,
withdrawn, becomes available, or expires, a speaker needs to repeat
selection only when one of those events occurs ({{time-variant}}).

Stage 1, choosing the originating AD:

1. Specificity: Keep only the routes with the highest Specificity Score
   ({{specificity}}). For example, prefer `dtn://rover1.example.org/**`
   over `dtn://rover*.example.org/**`.

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
   case-insensitive comparison.

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
    been held the longest, for stability.

Estimated delivery time is compared before path length because, in a DTN,
when a bundle can be delivered typically matters far more than the number
of ADs that it traverses. Arrival time is used where it is known, because
in a store-and-forward network the time at which an AD will accept a
bundle can be much earlier than the time at which the bundle will be
delivered. Where no arrival time is known, the time at which a route
becomes available is used instead; for routes that are already available,
step 3 then has no effect, and the choice between ADs is made on path
length.

Treating a route without an `arrival_time` as delivering bundles as soon as
it accepts them is optimistic, and so favors routes that omit the
attribute. For this reason, a speaker that re-advertises a route that
carries an `arrival_time` is required to include one ({{readvertisement}}),
and an originating AD that can estimate when it will deliver bundles
SHOULD include an `arrival_time` in the routes it originates.

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

Selecting a route by this process is meaningful only if each AD along the
path forwards bundles according to the route that it advertised. A DPP
speaker therefore SHOULD advertise to its peers, for each pattern and each
time, only a route derived from the route that it has selected for that
time ({{readvertisement}}).

This process differs from the BGP decision process
({{Section 9.1.2.2 of RFC4271}}), which compares AS_PATH length before the
MED, and compares MEDs only between routes received from the same
neighboring AS. Because that comparison is not a total order, the BGP
decision can depend on the order in which routes are compared, which can
cause persistent route oscillation {{RFC3345}}. Choosing the originating AD
first, without reference to metrics, avoids this.

A DPP speaker MAY implement additional local policy that further modifies
this process, but such policy is outside the scope of this specification.


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
the peer's address. SVCB records play no part in authorization: a speaker
listed in an SVCB record is authorized only if it presents a key matching
a TLSA record.

The following SvcParams are relevant:

| Parameter | Key ID | Description |
|-----------|--------|-------------|
| `alpn` | 1 | Application protocols (`h2` for gRPC over HTTP/2) |
| `port` | 3 | DPP service port |
| `ipv4hint` | 4 | IPv4 address hints |
| `ipv6hint` | 6 | IPv6 address hints |

Because DPP is carried over gRPC {{GRPC}}, the ALPN identifier negotiated
in TLS is that of the underlying HTTP version (e.g., `h2`), not a
DPP-specific identifier.

The TargetName of each SVCB record SHOULD be the hostname of a DPP speaker.
A TargetName of "." would make the `_dpp` owner name the effective
TargetName ({{Section 2.5.2 of RFC9460}}), requiring address records at a
name beginning with an underscore label, which some authoritative servers
cannot publish ({{Section 10.3 of RFC9460}}).

Example:

~~~
_dpp.dsn.example.org.  IN  SVCB  1 speaker.dsn.example.org. (
    alpn=h2 port=4556 )
~~~

### Multiple DPP Speakers per AD

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
; Speakers accepting incoming sessions
_dpp.dsn.example.org.  IN  SVCB  1 gds.dsn.example.org. (
    alpn=h2 port=4556 )
_dpp.dsn.example.org.  IN  SVCB  2 cbr.dsn.example.org. (
    alpn=h2 port=4556 )

; Keys authorized to act for dsn.example.org
_dpp.dsn.example.org.  IN  TLSA  3 1 1 (
    8f3a5e...c2 )   ; Goldstone
_dpp.dsn.example.org.  IN  TLSA  3 1 1 (
    51d07b...9e )   ; Canberra
_dpp.dsn.example.org.  IN  TLSA  3 1 1 (
    a7e214...41 )   ; Delegated operator (initiates only)
~~~

### Alignment with CL Discovery

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
   peer from causing lookups of arbitrary names.

4. Lookup: The speaker MUST query DNS for TLSA records at
   `_dpp.<local_ad_id>`, where `local_ad_id` is the AD named in the
   peer's `Hello`. The TLSA RRset MUST be validated as secure using
   DNSSEC. If its validation state is insecure, indeterminate, or bogus,
   or it contains no usable records ({{Section 4.1 of RFC6698}}), the
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


# Protocol Definition

The protocol uses a single bi-directional gRPC stream for the entire
lifecycle of the session.

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
  oneof payload {
    Hello hello = 10;
    // Field numbers 11 and 12 were used by HelloChallenge and
    // HelloResponse in earlier drafts
    KeepAlive keep_alive = 20;
    RouteUpdate update = 21;
    Notification notification = 30;
  }
}
~~~

## Identity Message

~~~ protobuf
message Hello {
  // DNS Domain (e.g., "dsn.example.org")
  string local_ad_id = 1;
  // Optional, free-form (e.g., "goldstone-1")
  string speaker_id = 2;
  uint32 hold_time_seconds = 3;
}
~~~

The optional `speaker_id` field allows a speaker to identify itself
among the speakers of its AD, for example in logs and diagnostics. Its
content is chosen by the AD and has no defined format; in particular, it
need not be an EID or a DNS name, since a DPP speaker need not be a BP
node. A receiving speaker MUST NOT use `speaker_id` for authentication,
authorization, loop detection, or route selection.

Authentication is performed by TLS (see {{handshake}}), so no
authentication messages are defined.

## Routing Messages

~~~ protobuf
message RouteUpdate {
  repeated RouteAdvertisement announcements = 1;
  repeated RouteWithdrawal withdrawals = 2;
}

// Route advertisement with extensible attributes
// Next-hop is determined per Gateway EID Resolution
// (explicit attribute or derived from AD identity)
// All fields except `patterns` are path attributes
// shared by all destinations
message RouteAdvertisement {
  // Destinations (NLRI equivalent)
  repeated EidPattern patterns = 1;

  // Path attributes (shared by all destinations)
  // List of AD Domains, for loop detection and path length
  repeated string ad_path = 2;
  // Administrator preference (lower = preferred)
  uint32 metric = 3;
  // Note: specificity_score is NOT transmitted;
  // receiver computes it from pattern

  // Extensible attributes
  repeated RouteAttribute attributes = 10;

  // Advertisement ID: identifies this advertisement within the
  // session; re-sending the same id replaces the advertisement
  uint64 id = 11;
}

// Removes all routes created by the advertisements with the
// listed advertisement IDs
message RouteWithdrawal {
  // Field numbers 1 and 2 were used for patterns and valid_from
  // in earlier drafts
  repeated uint64 advertisement_ids = 3;
}
~~~

## Route Attributes

~~~ protobuf
// Extensible route attribute (BGP-style path attributes)
// New attribute types can be added in future protocol versions
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

    // Forward compatibility: unknown attributes are preserved
    // and propagated
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

// An unset component is a wildcard; ipn:** has all three unset
message IpnPattern {
  IpnComponent allocator = 1;
  IpnComponent node = 2;
  IpnComponent service = 3;
}

message IpnComponent {
  oneof value {
    uint32 exact = 1;
    IpnRange range = 2;
  }
}

// Aligned power-of-two range (inclusive bounds)
message IpnRange {
  uint32 min = 1;
  uint32 max = 2;
}

message DtnPattern {
  string authority = 1;     // e.g. "rover*.example.org"
  string service_path = 2;  // e.g. "svc", or "**" for any service
}
~~~

## Control Messages

~~~ protobuf
message KeepAlive {}

message Notification {
  enum Level { INFO=0; WARNING=1; ERROR=2; }
  Level level = 1;
  string message = 2;
  int32 code = 3;
}
~~~


# Operational State Machine

A conformant DPP speaker MUST implement the following connection state
machine.

## States

Both speakers run the same state machine. The steps referred to are those
of {{handshake}}.

| State | Description |
|-------|-------------|
| CONNECTING | Mutually authenticated TLS connection and gRPC stream established (step 1). Speaker MUST send its `Hello` and move to HELLO_SENT. |
| HELLO_SENT | `Hello` sent; waiting for the peer's `Hello`. Upon receipt, speaker moves to VERIFYING. |
| VERIFYING | Peer's `Hello` received. Speaker performs the peer check, DNS lookup, and verification (steps 3-5). On success, it moves to ESTABLISHED. |
| ESTABLISHED | Peer verified. Speaker MAY send and receive `RouteUpdate` and `KeepAlive` messages. |
| FAILED | Error occurred. Speaker MUST send `Notification` (if possible) and close the stream. |

A speaker MUST move to FAILED if:

- it receives any message other than `Hello` or `Notification` before
  reaching ESTABLISHED;
- it receives a second `Hello` on the same stream;
- any step of {{handshake}} fails; or
- the session does not reach ESTABLISHED within a locally configured time.

## Hold Time and Keepalives

Each speaker proposes a hold time in the `hold_time_seconds` field of its
`Hello`, which MUST be at least 3 seconds. The hold time for the session is
the smaller of the two values.

In the ESTABLISHED state, a DPP speaker MUST send `KeepAlive` messages at
intervals not exceeding one third of the session hold time. If a speaker
receives no message from its peer for a period equal to the session hold
time, it MUST move to FAILED.

## Error Handling

Any error condition (DNS lookup failure, authentication failure, hold timer
expiry, transport error) MUST cause the DPP speaker to:

1. Send a `Notification` message with an appropriate error code (if possible).
2. Close the stream.
3. Log the failure reason (SHOULD).


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

## Transport Security

All DPP sessions use TLS 1.3 {{RFC8446}} or later, with both speakers
authenticated, as specified in {{handshake}}. This protects the
confidentiality and integrity of route advertisements in transit, and
ensures that every message on a session comes from the authenticated peer.

## Route Injection Attacks

A malicious or compromised DPP speaker could advertise routes for destinations
it does not legitimately serve. Mitigations include:

Out-of-band verification:
: Operators SHOULD verify route advertisements through external channels
  before accepting routes to critical destinations.

Route filtering:
: Implementations SHOULD support policy-based route filtering to reject
  unexpected advertisements.

Anomaly detection:
: Operators SHOULD monitor for unexpected route changes.

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

Mechanisms analogous to those used to secure BGP could address some of
these threats. Their specification is left to future work, but the
following approaches appear suitable. BGP relies on the Resource Public
Key Infrastructure (RPKI) {{RFC6480}}, a certificate hierarchy that
follows the allocation of IP addresses and AS numbers; DPP uses neither,
and DNS, signed with DNSSEC, already provides a suitable hierarchy, as it
does for the authorization of DPP speakers ({{tlsa-records}}).

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

## Denial of Service

DPP speakers are potentially vulnerable to denial of service attacks:

Connection exhaustion:
: Implementations SHOULD limit the number of concurrent sessions and
  implement rate limiting for new connections.

Route table exhaustion:
: Implementations SHOULD limit the number of routes accepted from a single
  peer.

Keepalive flooding:
: Implementations SHOULD rate-limit incoming messages.

## Privacy Considerations

Route advertisements reveal network topology and reachability information.
Operators should consider whether this information is sensitive before
establishing peering relationships.


# IANA Considerations

## Underscore-Prefixed DNS Name

This document requests registration of `_dpp` in the "Underscored and
Globally Scoped DNS Node Names" registry per {{RFC8552}}, for both RR types
used at that name:

| RR Type | _NODE NAME | Reference |
|---------|------------|-----------|
| SVCB | _dpp | This document |
| TLSA | _dpp | This document |

## Future Considerations

Future versions may additionally request:

- Registration of a well-known port for DPP (if not using existing gRPC
  conventions)
- Establishment of a registry for DPP route attribute types


--- back

# Acknowledgments
{:numbered="false"}

This document was developed as part of the Hardy BPv7 DTN Router project.

The author thanks Erik Kline and Adam Wiethuechter for their reviews and
comments.
