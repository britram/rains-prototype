---
title: RAINS (Another Internet Naming Service) Protocol Specification
abbrev: RAINS
docname: draft-trammell-rains-protocol-latest
date: 
category: exp

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: B. Trammell
    name: Brian Trammell
    organization: ETH Zurich
    street: Universitaetstrasse 6
    city: Zurich
    code: 8092
    country: Switzerland
    email: ietf@trammell.ch
 -
    ins: C. Fehlmann
    name: Christian Fehlmann
    org: ETH Zurich
    email: fehlmannch@gmail.com

normative:
    FIPS-186-3:
      author:
        -
          ins: NIST
      title: Digital Signature Standard FIPS 186-3
      date: June 2009

informative:
    XEP0115:
      title: XEP-0115 Entity Capababilities
      author:
        -
          ins: J. Hildebrand
        -
          ins: P. Saint-Andre
        -
          ins: R. Troncon
        - 
          ins: J. Konieczny
      date: 2008-02-26
    SCION:
      title: SCION Five Years Later - Revisiting Scalability, Control, and Isolation Next-Generation Networks (arXiv:1508.01651v1)
      author: 
        -
          ins: D. Barrera 
        - 
          ins: R. Reischuk
        - 
          ins: P. Szalachowski
        - 
          ins: A. Perrig
      date: 2015-08
    PARSER-BUGS:
      title: The Bugs We Have To Kill (USENIX login)
      author: 
        -
          ins: S. Bratus
        - 
          ins: M. Patterson
        - 
          ins: A. Shubina
      date: 2015-08
    BETTER-BLOOM-FILTER:
      title: Building a Better Bloom Filter
      author:
        -
          ins: Adam Kirsch
        -
          ins: Michael Mitzenmacher
      date: 2008-05-15
    I-D.ietf-dprive-dns-over-tls:
    I-D.ietf-dprive-dnsodtls:
    LUCID:
      target: https://www.ietf.org/proceedings/92/slides/slides-92-lucid-0.pdf
      title: LUCID problem (slides, IETF 92 LUCID BoF)
      author:
        -
          ins: A. Freytag
        -
          ins: A. Sullivan
    IAB-UNICODE7:
      target: https://www.iab.org/documents/correspondence-reports-documents/2015-2/iab-statement-on-identifiers-and-unicode-7-0-0/
      title: IAB Statement on Identifiers and Unicode 7.0.0      
      author:
        -
          ins: IAB

--- abstract

This document defines an alternate protocol for Internet name resolution,
designed as a prototype to facilitate conversation about the evolution or
replacement of the Domain Name System protocol. It attempts to answer the
question: "how would we design DNS knowing what we do now," on the
background of a set of properties of an idealized Internet naming service.

--- middle

# Introduction

This document defines an experimental protocol for providing Internet name
resolution services, as a replacement for DNS, called RAINS (RAINS, Another
Internet Naming Service). It is designed as a prototype to facilitate
conversation about the evolution or replacement of the Domain Name System
protocol, and was developed as a name resolution system for the SCION
("Scalability, Control, and Isolation on Next-Generation Networks") future
Internet architecture {{SCION}}. It attempts to answer the question: "how would
we design the DNS knowing what we do now," on the background of the properties
of an ideal naming service defined in {{pins}}.

Its architecture ({{architecture}}) and information model
({{information-model}}) are largely compatible with the existing Domain Name
System. However, it does take several radical departures from DNS as presently
defined and implemented:

- Delegation from a superordinate zone to a subordinate zone is done solely
  with cryptography: a superordinate defines the key(s) that are valid for
  signing assertions in the subordinate during a particular time interval.
  Assertions about names can therefore safely be served from any infrastructure.
- All time references in RAINS are absolute: instead of a time to live, each
  assertion's temporal validity is defined by the temporal validity of the
  signature(s) on it.
- All assertions have validity within a specific context. A context determines
  the rules for chaining signatures to verify validity of an assertion. The
  global context is a special case of context, which uses chains from the
  global naming root key. The use of context explicitly separates global usage
  of the DNS from local usage thereof, and allows other application-specific
  naming constraints to be bound to names; see {{context-in-assertions}}.
  Queries are valid in one or more contexts, with specific rules for
  determining which assertions answer which queries; see
  {{context-in-queries}}.
- There is an explicit separation between registrant-level names and
  sub-registrant-level names, and explicit information about registrars and
  registrants available in the naming system at runtime.
- Sets of valid characters and rules for valid names are defined on a per-zone
  basis, and can be verified at runtime.
- Reverse lookups are done using a completely separate tree, supporting
  delegations of any prefix length, in accordance with CIDR {{?RFC4632}} and
  the IPv6 addressing architecture {{?RFC4291}}.

Instead of using a custom binary framing as DNS, RAINS uses Concise Binary
Object Representation {{!RFC7049}}, partially in an effort to make
implementations easier to verify and less likely to contain potentially
dangerous parser bugs {{PARSER-BUGS}}. As with DNS, CBOR messages can be
carried atop any number of substrate protocols. RAINS is presently defined to
use TLS over persistent TCP connections (see {{protocol}}) as well as over UDP.

## About This Document

The source of this document is available in the repository
https://github.com/britram/rains-prototype, and a rendered working copy is
available at https://britram.github.io/rains-prototype. Open issues can be seen
and discussed at https://github.com/britram/rains-prototype/issues. 

# Terminology

The terms MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY, when they appear in
all-capitals, are to be interpreted as defined in {{!RFC2119}}.

In addition, the following terms are used in this document as defined:

- Subject: A name or address about which Assertions can be made.
- Object: A type/value pair of information about a name within an Assertion.
- Assertion: A mapping between a Subject and an Object, signed by the Authority
  for the namespace containing that Subject. See {{assertion}}.
- Authority: An entity that has the right to determine which Assertions exist
  within its Zone
- Delegation: An Assertion that an Authority has given the right to make
  assertions about the Assertions within the part of a namespace identified by a
  Subject to a subordinate Authority, by virtue of holding a secret key which
  can generate signatures verifiable using a public key associated with a
  delegation to the Zone.
- Zone: A portion of a namespace rooted at a given point in the namespace
  hierarchy. A Zone contains all the Assertions about Subjects tha exist within
  its part of the namespace. 
- Query: An expression of interest in certain types of objects pertaining to a
  Subject name in one or more contexts. See {{query}}.
- Context: Additional information about the scope in which an Assertion or Query
  is valid. See {{context-in-assertions}} and {{context-in-queries}}.
- Shard: A group of assertions common to a zone and valid at a given point in
  time, scoped to a lexicographic range of Subject names with in the Zone, for
  purposes of proving non-existence of an Assertion. Shards may be encoded to
  provide either absolute proof or probabalistic assurance of non-existence. See
  {{shards-and-p-shards}}.
- RAINS Message: Unit of exchange in the RAINS protocol, containing assertions,
  shards, zones, queries, and notifications. See {{cbor-message}}.
- Notification: A RAINS-internal message section carrying information about the
  operation of the protocol itself. See {{cbor-notification}}.
- Authority Service: A service provided by a RAINS Server for publishing
  assertions by an authority. See {{architecture}}.
- Query Service: A service provided by a RAINS Server for answering queries on
  behalf of a RAINS Client. See {{architecture}}.
- Intermediary Service: A service provided by a RAINS Server for answering
  queries and providing temporary storage for assertions on behalf of other
  RAINS Servers. See {{architecture}}.
- RAINS Server: A server that speaks the RAINS Protocol, and provides on or more
  services on behalf of other RAINS Servers and/or RAINS Clients. See
  {{architecture}}.
- RAINS Client: A client that uses the Query Service of one or more RAINS
  Servers to retrieve assertions on behalf of applications that wish to connect
  to named services in the Internet.

# An Ideal Internet Naming Service {#pins}

We begin by returning to first principles, to determine the dimensions of the
design space of desirable properties of an Internet-scale naming service. We
recognize that the choices made in the evolution of the DNS since its initial
design are only one path through the design space of Internet-scale naming
services. Many other naming services have been proposed, though none has been
remotely as successful for general-purpose use in the Internet. The following
subsections outline the space more generally. It is, of course, informed by
decades of experience with the DNS, but identifies a few key gaps which we then
aim to address directly with the design of RAINS.

{{pins-interfaces}} defines the set of operations a naming service should
provide for queriers and authorities, {{pins-properties}} defines a set of
desirable properties of the provision of this service, and {{pins-observations}}
examines implications of these properties.

## Interfaces {#pins-interfaces}

At its core, a naming service must provide a few basic functions for queriers,
associating a Subject of a query with information about that subject. The
information available from a naming service is that which is necessary for a
querier to establish a connection with some other entity in the Internet,
given a name identifying it.

- Name to Address: given a Subject name, the naming service returns a set of
  addresses associated with that name, if such an association exists, where the
  association is determined by the authority for that name. Names may be
  associated with addresses in one or more address families (e.g. IP version 4,
  IP version 6). A querier may specify which address families it is interested
  in receiving addresses for, and the naming system treats all address families
  equally. This mapping is implemented in the DNS protocol via the A and AAAA
  RRTYPES.

- Address to Name: given an Subject address, the naming service returns a set of
  names associated with that address, if such an association exists, where the
  association is determined by the authority for that address. This mapping is
  implemented in the DNS protocol via the PTR RRTYPE. IPv4 mappings exist within
  the in-addr.arpa. zone, and IPv6 mappings in the ip6.arpa. zone. These
  mappings imply a limited set of boundaries on which delegations may be made
  (octet boundaries for IPv4, nybble boundaries for IPv6).

- Name to Name: given a Subject name, the naming service returns a set of object
  names associated with that name, if such an association exists, where the
  association is determined by the authority for the subject name. This mapping
  is implemented in the DNS protocol via the CNAME RRTYPE. CNAME does not allow
  the association of multiple object names with a single subject, and CNAME may
  not combine with other RRTYPEs (e.g. NS, MX) arbitrarily.

- Name to Auxiliary Information: given a Subject name, the naming service
  returns other auxiliary information associated with that name that is useful
  for establishing communication over the Internet with the entities associated
  with that name. Most of the other RRTYPES in the DNS protocol implement these
  sort of mappings.

The query interface is not the only interface to the naming service: the
interface a naming service presents to an Authority allows updates to the set
of Assertions and Delegations in that Authority's namespace. Updates consist
of additions of, changes to, and deletions of Assertions and Delegations. In
the present DNS, this interface consists of the publication of a new zone file
with an incremented version number, but other authority interfaces are
possible.

## Properties {#pins-properties}

The following properties are desirable in a naming service providing the
functions in {{pins-interfaces}}.

### Meaningfulness 

A naming service must provide the ability to name objects that its human users
find more meaningful than the objects themselves.

### Distinguishability {#pins-distinguishability}

A naming service must make it possible to guarantee that two different names
are easily distinguishable from each other by its human users.

### Minimal Structure

A naming service should impose as little structure on the names it supports as
practical in order to be universally applicable. Naming services that impose a
given organizational structure on the names expressible using the service will
not translate well to societies where that organizational structure is not
prevalent.

### Federation of Authority

An Authority can delegate some part of its namespace to some other subordinate
Authority. This property allows the naming service to scale to the size of the
Internet, and leads to a tree-structured namespace, where each Delegation is
itself identified with a Subject at a given level in the namespace.

In the DNS protocol, this federation of authority is implemented through
delegation using the NS RRTYPE, redirecting queries to subordinate authorities
recursively to the final authority. When DNSSEC is used, the DS RRTYPE is used
to verify this delegation.

### Uniqueness of Authority

For a given Subject, there is a single Authority that has the right to
determine the Assertions and/or Delegations for that subject. The unitary
authority for the root of the namespace tree may be special, though; see
{{consensus-on-root-of-authority}}.

In the DNS protocol as deployed, unitary authority is approximated by the
entity identified by the SOA RRTYPE. The existence of registrars, which use
the Extensible Provisioning Protocol (EPP) {{?RFC5730}} to modify entries in
the zones under the authority of a top-level domain registry, complicates this
somewhat.

### Transparency of Authority

A querier can determine the identity of the Authority for a given Assertion.
An Authority cannot delegate its rights or responsibilities with respect to a
subject without that Delegation being exposed to the querier.

In DNS, the authoritative name server(s) to which a query is delegated via the
NS RRTYPE are known. However, we note that in the case of authorities which
delegate the ability to write to the zone to other entities (i.e., the
registry-registrar relationship), the current DNS provides no facility for a
querier to understand on whose behalf an authoritative assertion is being
made; this information is instead available via WHOIS. To our knowledge, no
present DNS name servers use WHOIS information retrieved out of band to make
policy decisions.

### Revocability of Authority

An ideal naming service allows the revocation and replacement of an authority
at any level in the namespace, and supports the revocation and replacement of
authorities with minimal operational disruption.

The current DNS allows the replacement of any level of delegation except the
root through changes to the appropriate NS and DS records. Authority
revocation in this case is as consistent as any other change to the DNS. 

### Consensus on Root of Authority

Authority at the top level of the namespace tree is delegated according to a
process such that there is universal agreement throughout the Internet as to
the subordinates of those Delegations.

### Authenticity of Delegation

Given a Delegation from a superordinate to a subordinate Authority, a querier
can verify that the superordinate Authority authorized the
Delegation.

Authenticity of delegation in DNS is provided by DNSSEC {{?RFC4033}}.

### Authenticity of Response

The authenticity of every answer is verifiable by the querier. The
querier can confirm that the Assertion returned in the answer is
correct according to the Authority for the Subject of the query.

Authenticity of response in DNS is provided by DNSSEC.

### Authenticity of Negative Response

Some queries will yield no answer, because no such Assertion exists. In
this case, the querier can confirm that the Authority for the
Subject of the query asserts this lack of Assertion.

Authenticity of negative response in DNS is provided by DNSSEC.

### Dynamic Consistency

Consistency in a naming service is important. The naming service should
provide the most globally consistent view possible of the set of Assertions
that exist at a given point in time, within the limits of latency and
bandwidth tradeoffs.

When an Authority makes changes to an Assertion, every query for a given
Subject returns either the new valid result or a previously valid result,
with known and/or predictable bounds on "how previously". Given that additions
of, changes to, and deletions of Asseretion may have different operational
causes, different bounds may apply to different operations.

The time-to-live (TTL) on a resource record in DNS provides a mechanism for
expiring old resource records. We note that this mechanism makes additions to
the system propagate faster than changes and deletions, which may not be a
desirable property. However, as no context information is explicitly available
in DNS, the DNS cannot be said to be dynamically consistent, as different
implicitly inconsistent views of an Assertion may be persistent.

### Explicit Inconsistency

Some techniques require giving different answers to different queries, even in
the absence of changes: the stable state of the namespace is not globally
consistent. This inconsistency should be explicit: a querier can know that an
answer might be dependent on its identity, network location, or other factors.

One example of such desirable inconsistency is the common practice of "split
horizon" DNS, where an organization makes internal names available on its own
network, but only the names of externally-visible subjects available to the
Internet at large. 

Another is the common practice of DNS-based content distribution, in which an
authoritative name server gives different answers for the same query depending
on the network location from which the query was received, or depending on the
subnet in which the end client originating a query is located (via the EDNS
Client Subnet extension {RFC7871}}). Such
inconsistency based on client identity or network address may increase query
linkability (see {{query-linkability}}).

These forms of inconsistency are implicit, not explicit, in the current DNS. We
note that while DNS can be deployed to allow essentially unlimited kinds of
inconsistency in its responses, there is no protocol support for a query to
express the kind of consistency it desires, or for a response to explicitly note
that it is inconsistent. {{?RFC7871}} does allow a querier to note that it would
specifically like the view of the state of the namespace offered to a certain
part of the network, and as such can be seen as inchoate support for this
property.

### Global Invariance

An Assertion which is not intended to be explicitly inconsistent by the
Authority issuing it must return the same result for every Query for it,
regardless of the identity or location of the querier.

This property is not provided by DNS, as it depends on the robust support on
the Explicit Inconsistency property above. Examples of global invariance
failures include geofencing and DNS-based censorship ordered by a local
jurisdiction.

### Availability

The naming service as a whole is resilient to failures of individual
nodes providing the naming service, as well as to failures of links among
them. Intentional prevention of successful, authenticated query by an
adversary should be as hard as practical.

The DNS protocol was designed to be highly available through the use of
secondary nameservers. Operational practices (e.g. anycast deployment) also
increase the availability of DNS as currently deployed.

### Lookup Latency

The time for the entire process of looking up a name and other necessary
associated data from the point of view of the querier, amortized over all
queries for all connections, should not significantly impact connection setup
or resumption latency.

### Bandwidth Efficiency

The bandwidth cost for looking up a name and other associated data necessary
for establishing communication with a given Subject, from the point of view of
the querier, amortized over all queries for all connections, should not
significantly impact total bandwidth demand for an application.

### Query Linkability

It should be costly for an adversary to monitor the infrastructure in order to
link specific queries to specific queriers.

DNS over TLS {{?RFC7858}} and DNS over DTLS {{?RFC8094}} provide this property
between a querier and a recursive resolver; mixing by the recursive helps with
mitigating upstream linkability.

### Explicit Tradeoff

A querier should be able to indicate the desire for a benefit with respect to
one performance property by accepting a tradeoff in another, including:

- Reduced latency for reduced dynamic consistency
- Increased dynamic consistency for increased latency
- Reduced request linkability for increased latency and/or reduced dynamic consistency
- Reduced aggregate bandwidth use for increased latency and/or reduced dynamic consistency

There is no support for explicit tradeoffs in performance properties available
to clients in the present DNS.

### Trust in Infrastructure

A querier should not need to trust any entity other than the authority as to the
correctness of association information provided by the naming service.
Specifically, the querier should not need to trust any intermediary of
infrastructure between itself and the authority, other than that under its own
control.

DNS provides this property with DNSSEC. However, the lack of mandatory DNSSEC,
and the lack of a viable transition strategy to mandatory DNSSEC (see
{{?I-D.trammell-optional-security-not}}), means that trust in infrastructure
will remain necessary for DNS even with large scale DNSSEC deployment.

## Observations {#pins-observations}

On a cursory examination, many of the properties of our ideal name service can
be met, or could be met, by the present DNS protocol or extensions thereto. We
note that there are further possibilities for the future evolution of naming
services meeting these properties. This section contains random observations
that might inform future work.

### Delegation and redirection are separate operations

Any system which can provide the authenticity properties enumerated above
is freed from one of the design characteristics of the present domain name
system: the requirement to bind a zone of authority to a specific set of
authoritative servers. Since the authenticity of delegation must be a
protected by a chain of signatures back to the root of authority, the location
within the infrastructure where an authoritative mapping "lives" is no longer
bound to a specific name server. While the present design of DNS does have its
own scalability advantages, this implication allows a much larger design space
to be explored for future name service work, as a Delegation need not always
be implemented via redirection to another name server.

### Unicode alone may not be sufficient for distinguishable names

Allowing names to be encoded in Unicode goes a long way toward meeting the
meaningfulness property (see {{meaningfulness}}) for the majority of speakers
of human languages. However, as noted by the Internet Architecture Board (see
{{IAB-UNICODE7}}) and discussed at the Locale-free Unicode Identifiers (LUCID)
BoF at IETF 92 in Dallas in March 2015 (see {{LUCID}}), it is not in the
general case sufficient for distinguishability (see {{pins-distinguishability}}).
An ideal naming service may therefore have to supplement Unicode by providing
runtime support for disambiguation of queries and assertions where the results
may be indistinguishable.

### Implicit inconsistency makes global invariance challenging to verify

DNS does not provide a generalized form of explicit inconsistency, so efforts to
verify global invariance, or rather, to discover Assertions for which global
invariance does not hold, are necessarily effort-intensive and dynamic. For
example, the Open Observatory of Network Interference performs DNS consistency
checking from multiple volunteer vantage points for a set of targeted (i.e.,
likely to be globally variant) domain names; see
https://ooni.torproject.org/nettest/dns-consistency/.

# RAINS Protocol Architecture

The RAINS architecture is simple, and vaguely resembles the architecture of
DNS. A RAINS Server is an entity that provides transient and/or permanent
storage for assertions about names, and a lookup function that finds assertions
for a given query about a name, either by searching local storage or by
delegating to another RAINS server. RAINS servers can take on any or all of
three roles:

- authority service, acting on behalf of an authority to ensure properly
  signed assertions are made available to the system (equivalent to an
  authoritative server in DNS);
- query service, acting on behalf of a client to answer queries with relevant
  assertions (equivalent to a recursive resolver in DNS), and to validate
  assertions on the client's behalf; and/or
- intermediary service, acting on behalf of neither but providing storage and
  lookup for assertions with certain properties for query and authority
  servers (partially replacing, but not really equivalent to, caching
  resolvers in DNS).

RAINS Servers use the RAINS Protocol defined in this document to exchange
queries and assertions. 

From the point of view of an authority (an entity owning some part of the
namespace by virtue of holding private keys associated with a zone delegation),
the RAINS protocol is used to publish signed assertions toward one or more
RAINS servers configured to provide authority service for their domains. Since
the signatures on these assertions expire periodically, the authority must
publish assertions continuously toward the authority services. In order to
provide a DNS-like operational experience, a RAINS server providing authority
service may be colocated with the infrastructure for publishing assertions;
however, the architecture of the protocol means these functions need not be
colocated.

Clients are configured, or use some out-of-band discovery mechanism, to contact
one or more query services using the RAINS protocol, and may trust those
services to verify assertion signatures on the client's behalf.

In this way, the same protocol is used between servers, from client to server,
and from publisher to server, with minor differences among the interactions
implemented as profiles. See {{protocol}} for details 

The protocol itself is designed in terms of its information and data model,
detailed in {{infomodel}}. Since all RAINS information is carried in messages
containing assertions, and an assertion is not valid unless it is signed, the
validity of an assertion is separated from whence the assertion was received.
This means the RAINS protocol itself is merely a means for moving RAINS
assertions around, and moving RAINS queries to places where they can be
answered. This document defines bindings for carrying RAINS messages over TLS
over TCP and over UDP, but bindings to other transports (e.g. QUIC
{{?QUIC=I-D.ietf-quic-transport}}) or session layers (e.g. HTTP {{?RFC7540}})
would be trivial to design, and the protocol provides a capability mechanism
for discovering alternate transports.

# Information and Data Model {{infomodel}}

The RAINS Protocol is based on an information model containing three primary
kinds of objects: Assertions, Queries, and Notifications. An Assertion contains
some information about a name or address, and a Query contains a request for
information about a name or address. Queries are answered with Assertions.
Notifications provide information about the operation of the protocol itself.
The protocol exchanges RAINS Messages, which act as envelopes containing
Assertions, Queries, and Notifications. RAINS Messages also provide for
capabilities-based versioning of the protocol, and for recognition of a chunk of
CBOR-encoded binary data at rest to be recognized as a RAINS message.

The RAINS data model is a relatively straightforward mapping of the information
model to the Concise Binary Object Representation (CBOR) {{!RFC7049}}, such
that Assertions are split into four subtypes of assertion depending on their
scope and purpose: singular Assertions and Zones for positive proof of the
existence of an association between a name and an object, Shards and P-Shards
for negative proof. Messages, singular Assertions, Shards, P-Shards, Zones,
Queries, and Notifications are each represented as a CBOR map of integer keys
to values, which allows each of these types to be extended in the future, as
well as the addition of non- standard, application-specific information to
RAINS messages and data items. A common registry of map keys is given in
{{tabmkey}}. RAINS implementations MUST ignore any data objects associated with
map keys they do not understand. Integer map keys in the range -22 to +23 are
reserved for the use of future versions or extensions to the RAINS protocol,
due to the efficiency of representation of these values in CBOR.

Message contents, signatures and object values are implemented as type-
prefixed CBOR arrays with fixed meanings of each array element; the structure
of these lower-level elements can therefore not be extended. Message section
types are given in {{tabsection}}, object types in {{tabobj}}, and signature
algorithms in {{tabsig}}.

{: #tabmkey title="CBOR Map Keys used in RAINS"}

| Code | Name           | Description                                            |
|-----:|----------------|------------------------------------------------        |
| 0    | signatures     | Signatures on a message or section                     |
| 1    | capabilities   | Capabilities of server sending message                 |
| 2    | token          | Token for referring to a data item                     |
| 3    | subject-name   | Subject name in an Assertion, Shard, P-Shard or Zone   |
| 4    | subject-zone   | Zone name in an Assertion, Shard, P-Shard or Zone      |
| 5    | subject-addr   | Subject address in address assertion                   |
| 6    | context        | Context of an Assertion, Shard, P-Shard, Zone or Query |
| 7    | objects        | Objects of an Assertion                                |
| 8    | query-name     | Fully qualified name for a Query                       |
| 9    | reserved       | Reserved for future use                                |
| 10   | query-types    | Acceptable object types for a Query                    |
| 11   | range          | Lexical range of Assertions in Shard or P-Shard        |
| 12   | query-expires  | Absolute timestamp for query expiration                |
| 13   | query-opts     | Set of query options requested                         |
| 14   | current-time   | Querier's latest assertion timestamp for a query       |
| 15   | reserved       | Reserved for future use                                |
| 16   | reserved       | Reserved for future use                                |
| 17   | query-keyphase | All requested key phases of a Query                    |
| 19   | shards         | Shard content of a zone                                |
| 20   | assertions     | Singular Assertion content of a Shard or Zone          |
| 21   | note-type      | Notification type                                      |
| 22   | note-data      | Additional notification data                           |
| 23   | content        | Content of a Message or a P-Shard                     |

The information model is designed to be representation-independent, and can be
rendered using alternate structured-data representations that support the
concepts of maps and arrays. For example, YAML or JSON could be used to
represent RAINS messages and data structures for debugging purposes. However,
signatures over messages and assertions need a single canonical representation
of the object to be signed as a bitstream. For RAINS, this is the CBOR
representation canonicalized as in {{c14n}}; therefore alternate
representations are always secondary to the CBOR data model. An alternate
representation designed for textual manipulation of RAINS data is described in
{{zonefiles}}.

The following subsections describe the information and data model of a RAINS
message from the top down.

## Messages

A Message is a self-contained unit of exchange in the RAINS protocol. Messages
have some content (the Assertions, Queries, and/or Notifications carried by the
Message) tagged with a token (see {{token}}). They may also carry information
about peer capabilities, and an optional signature.

More concretely, a Message is represented as a CBOR map with the CBOR tag value
15309736, which identifies the map as a RAINS message. This map MUST contain a
token key (2) and a content key (23), and MAY contain a capabilities key (1) a
signatures key (0).

The value of the content key is an array of zero or more Message Sections, as
defined in {{message-sections}}

The value of the token key is an opaque 16-byte octet array used to link
Messages, Queries, and Notifications; see {{tokens}} for details.

The value of the signatures key, when present, is an array of Signatures over
the entire Message, generated as in {{signatures}}, and to be verified against
an infrastructure key (see {{obj-infrakey}}) for the RAINS Server originating
the message.

A Message map MAY contain a capabilities (1) key, for exposing capabilties of
the message sender. The first Message sent from one peer to another MUST contain
the capabilities key. The capabilities mechanism is described in
{{capabilities}}.

A Message map MUST contain a token (2) key, whose value is a 16-byte array.
See {{tokens}} for details.

A Message map MUST contain a content (23) key, whose value is an array of
Message Sections; a Message Section is either an Assertion, Shard, Zone, Query,
or Notification.

### Message Section structure {#message-sections}

Each Message Section in the Message's content value is represented as a
two-element array. The first element in the array is the message section type,
encoded as an integer as in {{tabsection}}. The second element in the array is a
message section body, a CBOR map defined as in the subsections {{assertions}},
{{queries}}, and {{notifications}}

{: #tabsection title="Message Section Type Codes"}

| Code | Name         | Description                                       |
|-----:|--------------|---------------------------------------------------|
| 1    | assertion    | Singular Assertion (see {{singular-assertions}})  |
| -1   | revassertion | Address Assertion (see {{revassert}})             |
| 2    | shard        | Shard (see {{shards}})                            |
| 3    | zone         | Zone (see {{zones}})                              |
| 4    | query        | Query (see {{queries}})                           |
| -4   | revquery     | Address Query (see {{revquery}})                  |
| 5    | p-shard      | P-Shard (see {{p-shards}})                        |
| 23   | notification | Notification (see {{cbor-notification}})          |

## Assertions {#assertions}

Information about names in RAINS is carried by Assertions. An Assertion is a
statement about a mapping from a subject name to one or several object values of
the same type, signed by some authority for the namespace containing the
assertion, with a temporal validity determined by the lifetime of the
signature(s) on the Assertion.

The subject of an Assertion is identified by a name in three parts:

- the subject zone name, identifying the namespace within which the subject is
  contained;
- the subject name, identifying the name of the subject within that zone; and
- the subject context, as in {{assertion-context}}, identifying the context for
  purposes of explicit inconsistency.

The types of objects that can be associated with a subject are of several types,
described in {{obj-types}}.

There are four kinds of assertions, distinguished by their scope (how many
subjects are covered by a single Assertion) and their utility (whether the
assertion can be used for positive proof of a subject-object association, for
negative proof of the lack of such an association, or both):

- Singular Assertions contain a set of objects associated with a single given
  subject name in a given zone in a given context. The signature on a Singular
  Assertion can be used to prove the existance of an association between the
  subject name and the objects within the Assertion. Singular assertions are
  described in detail in {{singular-assertions}}.
- Zones contain Assertions for every object associated with every subject name
  within a given zone in a given context. The signature on a Zone can be used to
  prove both the existence of an association between a subject name and an
  object of a given type, as well as the absence of such an association. Zones
  are described in detail in {{zones}}. If signed, the assertions within a Zone
  can also be treated as Singular Assertions; in this case they inherit zone and
  context information from the containing zone.
- Shards contain Singular Assertions for every object associated with every
  subject name in a given lexicographic range of subject names within a given
  zone in a given context. The signature on a Shard can be used to prove the
  nonexistance of an object of a given type for a name within its range. Shards
  are described in detail in {{shards}}. If signed, the assertions within a
  Shard can also be treated as Singular Assertions; in this case they inherit
  zone and context information from the containing shard.
- P-Shards (or Probabalistic Shards) contain a data structure that can be used
  to demonstrate, within predictable bounds of false-negative probability, the
  non-existence of an object of a given type for a name within a lexicographic
  range of subjet names within a given zone in a given context. They allow an
  efficiency-accuracy tradeoff for negative proofs. P-Shards are described in
  detail in {{p-shards}}

\[EDITOR'S NOTE: fix this up, give it context: Assertions are sorted
lexicographically by their cbor encoded byte string in ascending order. That
means, they are sorted by the following elements in the mentioned order: fully
qualified name, context, type, object value(s), signature meta data.]
 
### Singular Assertions {#singular-assertions}

A Singular Assertion contains the set of objects associated with a single given
subject name in a given zone in a given context. A Singular Assertion with a
valid signature can be used as a positive answer to a query for a name. It is
represented as a CBOR map. The keys present in this map depend on whether the
Singular Assertion is contained in a Message, Shard or Zone.

Assertions contained directly within a Message's content value cannot inherit any
values from their containers, and therefore MUST contain the signatures (0),
subject-name (3), subject-zone (4), context (6), and objects (7) keys.

Assertions within a Shard or Zone can inherit values from their containers. A
contained Assertion MUST contain the subject-name (3), objects (7) keys. The
subject-zone (4) and context (6) keys MUST NOT be present. They are assumed to
have the same value as the corresponding values in the containing Shard or Zone
for signature generation and signature verification purposes; see
{{signature}}.

The value of the signatures (0) key, if present, is an array of one or more
Signatures as defined in {{signature}}. Signatures on a contained Assertion are
generated as if the inherited subject-zone and context values are present in the
Assertion, whether actually present or not. The signatures on the Assertion are
to be verified against the appropriate key for the Zone containing the Assertion
in the given context.

The value of the subject-name (3) key is a UTF-8 encoded {{!RFC3629}} string
containing the name of the subject of the assertion. The subject name may cover
multiple levels of hierarchy, separated by the '.' character. The
fully-qualified name of an Assertion is obtained by joining the subject-name to
the subject-zone with a '.' character. The subject-name must be valid according
to the nameset expression for the zone, if any (see {{obj-nameset}}).

The value of the subject-zone (4) key, if present, is a UTF-8 encoded string
containing the name of the zone in which the assertion is made and MUST end with
'.' (the root zone). If not present, the zone of the assertion is inherited from
the containing Shard or Zone.

The value of the context (6) key, if present, is a UTF-8 encoded string
containing the name of the context in which the assertion is valid. Both the
authority-part and the context-part MUST end with a '.'. If not present, the
context of the assertion is inherited from the containing Shard or Zone. See
{{assertion-context}} for more.

The value of the objects (7) key is an array of objects, as defined in
{{obj-types}}.

### Shards {#shards}

A Shard contain Singular Assertions for every object within a zone in a given
context whose subject name falls within a specified lexicographic range. A Shard
with a valid signature, within which a subject name should fall (i.e. appearing
within that Shard's range), but within which there is no Singular Assertion for
the specified subject name and object type, can therefore be taken as proof of a
negative query result for that subject name. Shards are used exclusively for
negative proof; the individual signatures on their contained Singular Assertions
are used for positive proof of the existence of an assertion.

The content of a Shard (in terms of the number of Singular Assertions it covers)
is chosen by the authority of the zone for which the Shard is valid. There is an
inherent tradeoff between the number of Assertions within a Shard and the size
of the Shard, and therefore the size of the Message that must be presented as
negative proof. P-Shards ("Probabalistic Shards", see {{#p-shards}}) allow a
different tradeoff, gaining space efficiency and coverage for a fixed,
predictable probability of a false positive (i.e., the possibility that the
P-Shard cannot be used to prove the nonexistence of a subject which does not, in
fact, exist).

A Shard is represented as a CBOR map. The keys present in this map depend on
whether the Shard is contained in a Message or Zone.

Shards contained in a Message's content value cannot inherit any values from a
contained Zone, and therefore MUST contain the  signatures (0),
subject-zone (4), context (6), range (11), and assertions (20) keys.

Shards contained within a Zone's shards value inherit zone and context values
from their containing Zone, and therefore  MUST contain the signatures (0),
range(11), and assertions (20) keys. The subject-zone (4) and context (6) keys
MUST NOT be present.

The value of the signatures (0) key is an array of one or more Signatures as
defined in {{signatures}}. If not present, the containing Zone MUST be signed.
Signatures on a Shard contained within a Zone are generated as if the inherited
subject-zone and context values are present in the Shard. The signatures on the
Shard are to be verified against the appropriate key for the Zone containing the
Shard in the given context.

The value of the subject-zone (4) key, if present, is a UTF-8 encoded string
containing the name of the zone in which the Assertions within the Shard is made
and MUST end with '.' (the root zone). If not present, the zone of the assertion
is inherited from the containing Zone.

The value of the context (6) key, if present, is a UTF-8 encoded string
containing the name of the context in which the Assertions within the Shard are
valid. Both the authority-part and the context-part MUST end with a '.'. If not
present, the context of the assertion is inherited from the containing Zone.

The value of the range (11) key is a two element array of strings or nulls
(subject-name A, subject-name B). A MUST lexicographically sort before B. If A
is null, the shard begins at the beginning of the zone. If B is null, the shard
ends at the end of the zone. The shard MUST NOT contain any assertions whose
subject names are equal to or sort before A, or are equal to or sort after B.

The value of the assertions (20) key is an array of Singular Assertions as
defined in {{assertions}}. These Singular Assertions MUST be sorted within the
in lexicographic order by subject name; the set of allowable Singular Assertions
is restricted by the range, as above.

### Zones {#zones}

A Zone contains Assertions for every object associated with every subject name
within a given zone in a given context, organized into Shards or singular
Assertions. A Zone with a valid signature can be used either as a positive
answer for a query about a name (when its contained assertions are not signed),
or as a negative answer to prove that a given object does not exist for a given
name.

Organizing Assertions into Zones allows operators of zones with few subject
names (e.g., used only for simple web hosting, as is the case with many zones in
the current Internet naming system) to minimize signing and zone management
overhead.

A Zone is represented as a CBOR map. Zones MUST contain the signatures (0),
subject-zone (4), and context (6) keys, and MUST contain one of the
shards (19) or assertions (20) keys.

The value of the signatures (0) key is an array of one or more Signatures on the
Zone as defined in {{signature}}. Signatures on the Zone are to be verified
against the appropriate key for the Zone in the given context.

The value of the subject-zone (4) key is a UTF-8 encoded string containing the
name of the Zone which MUST end with '.' (the root zone).

The value of the context (6) key is a UTF-8 encoded string containing the name
of the context for which the Zone is valid. Both the authority-part and the
context-part MUST end with a '.'. See {{assertion-context}}

The contents of the Zone are contained in the values of the shards (19) or
assertions (20) key. The value of the shards key is a CBOR array of Shards as
defined in {{shards}}. The value of the assertions key is a CBOR array of
Singular Assertions as defined in {{singular-assertions}}. A Zone may either be
organized into Shards, in which case the Shards within the Zone must cover the
entire lexicographic space of subject names within the Zone, or it may be
organized as Singular Assertions. 

Within the shards array, if present, the contained Shards MUST be sorted in
lexicographic order by shard range start. Within the assertions array, if
present, the contained Singular Assertions MUST be sorted in lexicographic order
by subject name.

### P-Shards {#p-shards}

Shards ({{shards}}) can be used as definitive proof of the nonexistence of a
name within a zone. P-Shards serve the same purpose, but offer only a
probabalistic guarantee of the non-existence of the name. Specifically, as they
are based on Bloom filters, a subject name which does not in fact exist may
appear in the P-Shard; in return for this uncertainty, they offer a much more
space-efficient way to demonstrate the non-existence of a subject within the
zone than Shards do. There is a tradeoff between the size of the bit string
storing the Bloom filter, the number of Assertions covered by the P-Shard, and
the false positive error rate. The zone authority can determine how to weight
them.

A P-Shard is represented as a CBOR map. This map MUST contain the signatures
(0), subject-zone (4), context (6), and content(23) keys. It MAY contain the
range(11) key.

The value of the signatures (0) key is an array of one or more Signatures as
defined in {{cbor-signature}}. The signatures on the P-Shard are to be verified
against the appropriate key for the Zone for which the P-Shard is cvalid in the
given context.

The value of the subject-zone (4) key is a UTF-8 encoded string containing the
name of the zone in which the Assertions in the P-Shard is made and MUST end
with '.' (the root zone).

The value of the context (6) key is a UTF-8 encoded string containing the name
of the context in which the Assertions in the P-Shard are valid. Both the
authority-part and the context-part MUST end with a '.'.

The value of the range (11) key, if present, is a two element array of strings
or nulls (subject-name A, subject-name B). A MUST lexicographically sort before
B. If A is null, the P-Shard begins at the beginning of the zone. If B is null,
the shard ends at the end of the zone. The P-Shard MUST NOT be used to check
the existence of any assertions whose subject names are equal to or sort before
A, or are equal to or sort after B. If the range (11) key is not present, the
P-shard covers then entire zone.

The value of the content (23) key is a three-element array. The first element
identifies the algorithm used for generating the bitstring. The second element
identifies the hash function in use for generating the bitstring. The third
element contains the bitstring itself, as an octet array. The size of the
bitstring must be 0 mod 8.

{{tabpsds}} enumerates supported generation algorithms; supported hash functions
are given in {{hash-functions}}.

{: #tabpsds title="P-shard generation algorithms"}

| Code  | Name         | Description                                |
|------:|--------------|--------------------------------------------|
| 1     | bloom-km-2   | KM-optimized bloom filter with nh=2        |
| 2     | bloom-km-4   | KM-optimized bloom filter with nh=4        |
| 3     | bloom-km-8   | KM-optimized bloom filter with nh=8        |

The bloom-km-2 and bloom-km-4 datastructures generates a bitstring using a
Bloom filter and the Kirsch-Mitzenmacher optimization {{BETTER-BLOOM-FILTER}}.

To add or verify an assertion to a bloom-km structure, the assertion is first
encoded as a four-element CBOR array. The first element is the subject name.
The second element is the subject zone. The third element is the subject
context. The fourth element is the type code as in {{tabobj}} in {{obj-types}}.
This encoded object is then hashed according to the specified hash algorithm.
The hash algorithm's output is then split into nh equal length parts (2 for
bloom-km-2, 4 for bloom-km-4, 8 for bloom-km-8), and these parts are used as
indexes into the bitstring modulo the bitstring length. To add the assertion,
all bits at the given indices are set to 1. To verify the assertion, all bits
at the given indices are checked, and the assertion is taken to be in the
filter if all bits are 1.

### Dynamic Assertion Validity {#assertion-dynamics}

For a given {subject, zone, context, type} tuple, multiple assertions can be
valid at a given point in time; the union of the object values of all of these
assertions is considered to be the set of valid values at that point in time.

### Semantic of nonexistence proofs {#antiassertions}

Shards, P-Shards and Zones can all be used to prove nonexistence during their
validity. However, real naming systems are dynamic: an assertion might be
created, altered, expired or revoked during the validity period of a shard,
P-Shard or zone, leading to an inconsistency. Thus, a section proving
nonexistence only captures the state at the point in time when it was signed.
If an authoritative proof of non-existence is necessary, a query for
nonexistence (see {{nonexistence-query}}) may be sent to a server identified as
an authority for the name.

### Context in Assertions {#assertion-context}

Assertion contexts are used to provide explicit inconsistency, while allowing
Assertions themselves to be globally valid regardless of the query to which they
are given in reply. Explicit inconsistency is the simultaneous validity of
multiple sets of Assertions for a single subject name at a given point in time.
Explicit inconsistency is implemented by using the context to select an
alternate chain of signatures to use to verify the validity of an Assertion, as
follows:

- The global context is identified by the special context name '.'. Assertions
  in the global context are signed by the authority for the subject name. For
  example, assertions about the name 'ethz.ch.' in the global context are only
  valid if signed by the relevant authority which is either 'ethz.ch.', 'ch.',
  or '.' depending on the value of the subject name of the assertion.
- A local context is associated with a given authority. The local context's name
  is divided into an  authority-part and a context-part by a context marker
  ('cx--'). The authority-part directly identifies the authority whose key was
  used to sign the assertion; assertions within a local context are only valid
  if signed by the identified authority. Authorities have complete control over
  how the contexts under their namespaces are arranged, and over the names
  within those contexts. Both the authority-part and the context-part must end
  with a '.'.

Some examples illustrate how context works:

- For the common split-DNS case, an enterprise could place names for machines on
  its local networks within a separate context. E.g., a workstation could be
  named 'simplon.cab.inf.ethz.ch.' within the context
  'staff-workstations.cx--inf.ethz.ch.' Assertions about this name would be
  signed by the authority for 'inf.ethz.ch.'. Here, the context serves simply as
  a marker, without enabling an alternate signature chain: note that the name
  'simplon.cab.inf.ethz.ch' could at the same time be validly signed in the
  global context by the authority over that name to allow external users access
  this workstation. The local context simply marks this assertion as internal.
  This allows a client making requests of local names to know they are local,
  and for local resolvers to manage visibility of assertions outside the
  enterprise: explicit context makes accidental leakage of both queries and
  assertions easier to detect and avoid.
- Contexts make captive-portal interactions more explicit: a captive portal
  resolver could respond to a query for a common website (e.g. www.google.ch)
  with a signed response directed at the captive portal, but within a context
  identifying the location as well as the ISP (e.g.
  sihlquai.zurich.ch.cx--starbucks.access.some-isp.net.). This response will be
  signed by the authority for 'starbucks.access.some-isp.net.'. This signature
  achieves two things: first, the client knows the result for www.google.ch is
  not globally valid; second, it can present the user with some indication as to
  the identity of the captive portal it is connected to.

Further examples showing how context can be used in queries as well are given
in {{context-in-queries}} below.

Developing conventions for assertion contexts for different situations will
require implementation and deployment experience, and is a subject for future
work.

### Zone-Reflexive Assertions

A zone may make an assertion about itself by using the string "@" as a subject
name. This facility can be used for any assertion type, but is especially useful
for self-signing root zones, and for a zone to make a subsequent key assertion
about itself. If an assertion of a given type about a zone is available both in
the zone itself and in the superordinate zone, the assertion in the
superordinate zone will take precedence.

### Address Assertions {#revassertions}

\[EDITOR'S NOTE move up from old content, below]

## Object Types and Encodings {#obj-types}

Each Object associated with a given subject name in a Singular Assertion (see
{{singular-assertions}}) is represented as a CBOR array, where the first element
is the type of the object, encoded as an integer in the following table:

{: #tabobj title="Object type codes"}

| Code  | Name         | Description                             | Reference     |
|------:|--------------|-----------------------------------------|---------------|
| 1     | name         | name associated with subject            | {{#obj-name}} |
| 2     | ip6-addr     | IPv6 address of subject                 | {{#obj-ip6}} |
| 3     | ip4-addr     | IPv4 address of subject                 | {{#obj-ip4}} |
| 4     | redirection  | name of zone authority server           | {{#obj-redir}} |
| 5     | delegation   | public key for zone delgation           | {{#obj-deleg}} |
| 6     | nameset      | name set expression for zone            | {{#obj-nameset}} | 
| 7     | cert-info    | certificate information for name        | {{#obj-cert}} | 
| 8     | service-info | service information for srvname         | {{#obj-srv}} | 
| 9     | registrar    | registrar information                   | {{#obj-regr}} | 
| 10    | registrant   | registrant information                  | {{#obj-regt}} | 
| 11    | infrakey     | public key for RAINS infrastructure     | {{#obj-infrakey}} | 
| 12    | extrakey     | external public key for subject         | {{#obj-extrakey}} | 
| 13    | nextkey      | next public key for subject             | {{#obj-nextkey}} |

Subsequent elements contain the object content, encoded as described in the
respective subsection below.

### Name Alias {#obj-name}

A name (1) object contains a name associated with a name as an alias. It is
represented as a three-element array. The second element is a fully-qualified
name as a UTF-8 encoded string. The third type is an array of object type codes
for which the alias is valid, with the same semantics as the query-types (9) key
in queries (see {{queries}}).

The name type is roughly equivalent to the DNS CNAME RRTYPE.

### IPv6 Address {#obj-ip6}

An ip6-addr (2) object contains an IPv6 address associated with a name. It is
represented as a two element array. The second element is a byte array of
length 16 containing an IPv6 address in network byte order.

The ip6-addr type is roughly equivalent to the DNS AAAA RRTYPE.

### IPv4 Address {#obj-ip4}

An ip4-addr (3) object contains an IPv4 address associated with a name. It is
represented as a two element array. The second element is a byte array of
length 4 containing an IPv4 address in network byte order.

The ip4-addr type is roughly equivalent to the DNS A RRTYPE.

### Redirection {#obj-redir}

A redirection (4) object contains the fully-qualified name of a RAINS authority
server for a named zone. It is represented as a two-element array. The second
element is a fully-qualified name of an RAINS authority server as a UTF-8
encoded string.

The redirection type is used to point to a "last-resort" server or server from
which assertions about a zone can be retrieved; it therefore approximately
replaces theD NS NS RRTYPE.

### Delegation {#obj-deleg}

A delegation (5) object contains a public key used to generate signatures on
assertions in a named zone, and by which a delegation of a name within a zone to
a subordinate zone may be verified. It is represented as an 4-element array. The
second element is a signature algorithm identifier as in {{signatures}}. The
third element is a key phase as in {{signatures}}. The fourth element is the
public key, formatted  
as defined in {{signatures}} for the given algorithm identifier and RAINS
delegation chain keyspace.

Delegations approximately replace the DNS DNSKEY RRTYPE.

### Nameset {#obj-nameset}

A nameset (6) object contains an expression defining which names are allowed
and which names are disallowed in a given zone. It is represented as a two-
element array. The second element is a nameset expression to be applied to
each name element within the zone without an intervening delegation.

The nameset expression is represented as a UTF-8 string encoding a modified
POSIX Extended Regular Expression format (see POSIX.2) to be applied to each
element of a name within the zone. A name containing an element that does not
match the valid nameset expression for a zone is not valid within the zone, and
the nameset assertion can be used to prove nonexistence.

The POSIX character classes :alnum:, :alpha:, :ascii:, :digit:, :lower:, and
:upper: are available in these regular expressions, where:

- :lower: matches all codepoints within the Unicode general category "Letter, lowercase"
- :upper: matches all codepoints within the Unicode general category "Letter, uppercase"
- :alpha: matches all codepoints within the Unicode general category "Letter".
- :digit: matches all codepoints within the Unicode general category "Number, decimal digit"
- :alnum: is the union of :alpha: and :digit:
- :ascii: matches all codepoints in the range 0x20-0x7f

In addition, each Unicode block is available as a character class, with the
syntax :ublkXXXX: where XXXX is a 4 or 5 digit, zero-prefixed hex encoding of
the first codepoint in the block. For example, the Cyrillic block is available
as :ublk0400:.

Unicode escapes are supported in these regular expressions; the sequence \uXXXX
where XXXX is a 4 or 5 digit, possibly zero-prefixed hex encoding of the
codepoint, is substituted with that codepoint.

Set operations (intersection and subtraction) are available on character
classes. Two character class or range expressions in a bracket expression joined
by the sequence && are equivalent to the intersection of the two character
classes or ranges. Two character class or range expressions in a bracket
expression joined by the sequence -- are equivalent to the subtraction of the
second character class or range from the first.

For example, the nameset expression:

\[\[:ublk0400:]&&\[:lower:]\[:digit:]]+

matches any name made up of one or more lowercase Cyrillic letters and digits.
The same expression can be implemented with a range instead of a character
class:

\[\u0400-\u04ff&&\[:lower:]\[:digit:]]+

Nameset expression support is experimental and subject to (radical) change in
future revisions of this specification.

### Certificate Information {#obj-cert}

A cert-info (7) object contains an expression binding a certificate or
certificate authority to a name, such that connections to the name must either
use the bound certificate or a certificate signed by a bound authority. It is
represented as an five-element array.

The second element is the protocol family specifier, describing the
cryptographic protocol used to connect, as defined in {{tabcertproto}}. The
protocol family defines the format of certificate data to be hashed. The third
element is the certificate usage specifier as in {{tabcertusage}}, describing
the constraint imposed by the assertion. These are defined to be compatible with
Certificate Usages in the TLSA RRTYPE for DANE {{?RFC6698}}. The fourth element
is the hash algorithm identifier, defining the hash algorithm used to generate
the certificate data, as in {{tabhash}}. The fifth item is the data itself,
whose format is defined by the protocol family and hash algorithm.

{: #tabcertproto title="Certificate information protocol families"}

| Code | Name     | Protocol family                            | Certificate format |
|-----:|----------|--------------------------------------------|--------------------|
|    0 | unspec   | Unspecified                                | Unspecified        |
|    1 | tls      | Transport Layer Security (TLS) {{!RFC8446}} | {{!RFC5280}}        |

Protocol family 0 leaves the protocol family unspecified; client validation
and usage of cert-info assertions, and the protocol used to connect, are up to
the client, and no information is stored in RAINS. Protocol family 1 specifies
Transport Layer Security version 1.3 {{!RFC8446}} or a subsequent version,
secured with PKIX {{!RFC5280}} certificates.

{: #tabcertusage title="Certificate information usage values"}

| Code | Name | Certificate usage                |
|-----:|------|----------------------------------|
|    2 | ta   | Trust Anchor Certificate         |
|    3 | ee   | End-Entity Certificate           |

A trust anchor certificate constraint specifies a certificate that MUST appear
as the trust anchor for the certificate presented by the subject of the
assertion on a connection attempt. An end-entity certificate constraint
specifies a certificate that MUST be presented by the subject of the assertion
on a connection attempt.

Certificate information be hashed using an appropriate hash function described
in {{hash-functions}}; hash functions are identified by a code as in
{{tabhash}}. Code 0 is used to store full certificates in RAINS assertions,
while other codes are used to store hashes for verification.

For example, in a cert-info object with values \[ 7, 1, 3, 3, (data) ], the
data would be a 48 SHA-384 hash of the ASN.1 DER-encoded X.509v3 certificate
(see Section 4.1 of {{?RFC5280}}) to be presented by the endpoint on a
connection attempt with TLS version 1.2 or later.

The cert-info type replaces the TLSA DNS RRTYPE.

### Service Information {#obj-srv}

A service-info (8) object gives information about a named service. Services
are named as in {{!RFC2782}}. It is represented as a four-element array. The
second element is a fully-qualified name of a host providing the named service
as a UTF-8 string. The third element is a transport port number as a positive
integer in the range 0-65535. The fourth element is a priority as a positive
integer, with lower numbers having higher priority.

The service-info type replaces the DNS SRV RRTYPE.

### Registrar Information {#obj-registrar}

A registrar (9) object gives the name and other identifying information of the
registrar (the organization which caused the name to be added to the namespace)
for organization-level names. It is represented as a two element array. The
second element is a UTF-8 string of maximum length 256 bytes containing
identifying information chosen by the registrar according to the registry's
policy.

### Registrant Information {#obj-registrant}

A registrant (10) object gives information about the registrant of an
organization-level name. It is represented as a two element array. The second
element is a UTF-8 string with a maximum length of 4096 bytes containing this
information, with a format chosen by the registrant according to the registry's
policy.

### Infrastructure Key {#obj-infrakey}

An infrakey (11) object contains a public key used to generate signatures on
messages by a named RAINS server, by which a RAINS message signature may be
verified by a receiver. It is identical in structure to a delegation object,
as defined in {{obj-deleg}}. Infrakey signatures are especially useful
for clients which delegate verification to their query servers to authenticate
the messages sent by the query server.

### External Key {#obj-extrakey}

An extrakey (12) object contains a public key used to generate signatures on
assertions in a named zone outside of the normal delegation chain. It is
represented as an 4-element array, where the second element is a signature
algorithm identifier, and the third element is keyspace identifier, as in
{{signatures}}. The fourth element is the public key, as defined in
{{signatures}} for the given algorithm identifier. An extrakey may be
matched with a public key obtained through other means for additional
authentication of an assertion. 

### Next Delegation Public Key {#obj-nextkey}

A nextkey (13) object contains the a public key that a zone owner would like its
superordinate to delegate to in the future. It is represented as an 6-element
array. The second element is a signature algorithm identifier as in
{{signatures}}.  The third element is a key phase as in {{signatures}}. The
fourth element is the public key, as defined in {{signatures}} for the given
algorithm identifier. The fifth element is the requested-valid-since time, and
the sixth element is the requested-valid-until time, formatted as for signatures
as in {{signatures}}. See {{public-key-management}} for more.

## Hash Functions {#hash-functions} 

Hash algorithms are used in several places in the RAINS data model:

- hashing certificate data in cert-info objects (see {{obj-cert}})
- hashing assertions into Bloom filters for probabalistic shards (see {{p-shards}})
- hashing assertion and message data as part of generating a MAC (see {{signatures}})
- hashing assertion data for a confirmation query (see {{confirmation}})

Where they are used in RAINS, hash functions are identified by a code given in
{{tabhash}}. In this table, applicability "C" means the hash is valid for use
in a certificate info object, "P" that it can be used for hashing assertions
for P-shards, "S" that it can be used for hashing assertions and messages for
signatures, and "Q" that it can be used for hashing assertions for confirmation queries.

{: #tabhash title="Hash algorithms"}

| Code | Name       | Reference         | Length | Applicability |
|-----:|------------|-------------------|--------|---------------|
| 0    | nohash     | (data not hashed) | var.   | C             |
| 1    | sha-256    | {{!RFC6234}}      | 32     | CPSQ          |
| 2    | sha-512    | {{!RFC6234}}      | 64     | CSQ           |
| 3    | sha-384    | {{!RFC6234}}      | 48     | CSQ           |
| 4    | shake256   | {{!RFC8419}}      | 32     | PSQ           |
| 5    | fnv-64     | {{!FNV=I-D.eastlake-fnv}} | 8 | P          |
| 6    | fnv-128    | {{!FNV=I-D.eastlake-fnv}} | 16 | P         |

## Queries {#queries}

Information about requests for information about names is carried in Queries. A
Query specifies the name and object type about which information is requested,
information about how long the querier is willing to wait for an answer, and
additional options indicating the querier's preferences about how the query
should be handled.

In contrast to Assertions, the subject in a Query is given as a fully-qualified
name - the subject name concatenated to the zone name with a '.', since a
querier may not know the zone name associated with a fully-qualified name.

There are two kinds of queries supported by the RAINS data model:

- Query (or Normal Query): a request for information of a given type about a
given subject, about which the querier expresses no prior information.

- Confirmation Query: a request for information of a given type about a given
subject, for which the querier already has a valid cached assertion or a valid
cached nonexistence proof, but for which the querier would like a new assertion
if available. Confirmation queries are covered in {{confirmation}}.

Both of queries are carried in a Query message section. Each Query section in a
Message represents a separate query.

A Query body is represented as a CBOR map. Queries MUST contain the query-name (8),
context (6), query-types (10), and query-expires (12) keys. Queries MAY contain
the query-opts (13), query-keyphase (17) keys, and/or current-time (14) keys. 

The value of the query-name (8) key is a UTF-8 encoded string containing the
name for which the query is issued and MUST end with a '.' (the root zone).

The value of the context (6) key is a UTF-8 encoded string containing the name
of the context to which a query pertains. A zero-length string indicates that
assertions will be accepted in any context.

The value of the query-types (10) key is an array of integers encoding the
type(s) of objects (as in {{cbor-object}}) acceptable in answers to the query.
All values in the query-type array are treated at equal priority: for example,
\[2,3] means the querier is equally interested in both IPv4 and IPv6 addresses
for the query-name. An empty query-types array indicates that objects of any
type are acceptable in answers to the query.

The value of the query-expires (12) key is a CBOR integer epoch timestamp
identified with tag value 1 and encoded as in section 2.4.1 of {{!RFC7049}}.
After the query-expires time, the query will have been considered not answered
by the original issuer and can be ignored.

The value of the query-keyphase (17) key, if present, is an array of integers
representing all key phases (see {{cbor-signature}}) desired in delegation and
nextkey answers to queries (see {{obj-deleg}} and {{obj-nextkey}}). The value
of the query-keyphase key is ignored for all queries where query-types does not
include delegation or nextkey. A query for a delegation or nextkey object that
does not contain a query-keyphase key should return information for all
available keyphases.

The value of the query-opts (13) key, if present, is an array of integers in
priority order of the querier's preferences in tradeoffs in answering the
query. See {{query-opts}}.

The value of the current-time (14) key, if present, is the timestamp of the
latest information available at the querier for the queried subject and object
type. See {{confirmation}} for details of how confirmation queries work.

### Query Options {#query-opts}

RAINS supports a set of query options to allow a querier to express
preferences. Query options are advisory.

{: #tabqopts title="Query Option Codes"}

| Code | Description                                                    |
|-----:|----------------------------------------------------------------|
| 1    | Minimize end-to-end latency                                    |
| 2    | Minimize last-hop answer size (bandwidth)                      |
| 3    | Minimize information leakage beyond first hop                  |
| 4    | No information leakage beyond first hop: cached answers only   |
| 5    | Expired assertions are acceptable                              |
| 6    | Enable query token tracing                                     |
| 7    | Disable verification delegation (client protocol only)         |
| 8    | Suppress proactive caching of future assertions                |
| 9    | Maximize freshness of result                                   |

Options 1-5 and 9 specify performance/privacy tradeoffs. Each server is free to
determine how to minimize each performance metric requested; however, servers
MUST NOT generate queries to other servers if "no information leakage" is
specified, and servers MUST NOT return expired assertions unless "expired
assertions acceptable" is specified.

Option 6 specifies that the token on the message containing the query (see
{{tokens}}) should be used on all queries resulting from a given query,
allowing traceability through an entire RAINS infrastructure. These resulting
queries SHOULD also carry Option 6. When Option 6 is not present, queries
sent by a server in response to an incoming query SHOULD use different tokens.

By default, a client service will perform verification of negative queries and
return a 404 No Assertion Exists for queries with a consistent proof of non-
existence, within a message signed by the query service's infrakey. Option 7
disables this behavior, and causes the query service to return the shard
proving nonexistence for verification by the client. It is intended to be used
with untrusted query services.

Option 8 specifies that a querier's interest in a query is strictly ephemeral,
and that future assertions related to this query SHOULD NOT be proactively
pushed to the querier.

Option 9 specifies that the querier would prefer a fresh result to one from the
server's cache. If the server is not running an authority service for the
queried subject, it can honor this request by issuing a query toward the
authority. As this could be used for denial-of-service-attacks, a server
honoring Option 9 SHOULD limit the rate of "freshness" queries it issues.

\[EDITOR'S NOTE: RAINS is missing a facility for "trusted" interserver
communication. Queries are meant to be as disjoint from their possible answers
as is specified in this document, since this allows maximum flexibility in
implementation in the query / but when two servers are configured to be more
closely coupled to each other, the query/assertion interface really doesn't
support the full expressiveness of this tight coupling. There are three ways
that I can see do deal with this: (1) ignore it and deal, (2) try to build a set
of query options to make queries more, (3) rely on heuristics and/or
configuration to specify different query answering behavior for trusted servers,
based on information about the connection itself. Leaning toward (3) now, which
would go down in the "operational considerations" section]

### Confirmation Queries {#confirmation}

\[EDITOR'S NOTE: make sure we actually need the hash - why is a timestamp not
sufficient here?]

A Query containing a current-time key is a confirmation query, used by a server
to refresh a cached query result. The querier passes the timestamp of the most
recent result it has cached, taken from the most recent start time of the
validity of the signature(s) on the assertion(s) that may answer it. If the
answer to a confirmation query is not newer than the given timestamp, the server
may answer with a notification of type 304 instead of with an assertion. answer
is older or would match the current answer is a notification of type 304 (see
{{notifications}}).

The value of the current-time key is represented as a CBOR integer epoch timestamp
identified with tag value 1 and encoded as in section 2.4.1 of {{!RFC7049}}.

### Context in Queries {#query-context}

Context is used in queries as it is in assertions (see
{{context-in-assertions}}). Assertion contexts in an answer to a query have to
match the context in the query in order to respond to a query. The Context
section of a query contains the context of desired assertions; a special "any"
context (represented by the empty string) indicates that assertions in any
context will be accepted.

Query contexts can also be used to provide additional information to RAINS
servers about the query. For example, context can provide a method for explicit
selection of a CDN server not based on either the client's or the resolver's
address (see {{?RFC7871}}). Here, the CDN creates a context for each of its
content zones, and an external service selects appropriate contexts for the
client based not just on client source address but passive and active
measurement of performance. Queries for names at which content resides can then
be made within these contexts, with the priority order of the contexts
reflecting the goodness of the zone for the client. Here, a context might be
'zrh.cx--cdn-zones.some-cdn.com.' for names of servers hosting content in a
CDN's Zurich data center. A client could represent its desire to find content
nearby by making queries in the zrh.cx--, fra.cx-- (Frankfurt), and ams.cx--
(Amsterdam) contexts of the 'cdn-zones.some-cdn.com.' authority. In all cases,
the assertions themselves will be signed by the authority for
'cdn-zones.some-cdn.com.', accurately representing that it is the CDN, not the
owner of the related name in the global context, that is making the assertion.

As with assertion contexts, developing conventions for query contexts for
different situations will require implementation and deployment experience,
and is a subject for future work.

### Address Queries {#revqueries}

\[EDITOR'S NOTE: write me]

## Notifications {#notifications}

Notifications contain information about the operation of the RAINS protocol
itself. A Notification Message Section body is represented as a CBOR map, which
MUST contain the token (2) and note-type (21) keys, and MAY contain the
note-data (22) key. 

The value of the token (2) key is a 16-byte array, which
MUST contain the token of the message or query to which the notification is a
response. See {{tokens}}.

The value of the note-type key is encoded as an integer as in the
{{tabnotify}}.

{: #tabnotify title="Notification Type Codes"}

| Code | Description                          | Reference        |
|-----:|--------------------------------------|------------------|
| 100  | Connection heartbeat                 | {{heartbeat}}    |
| 304  | Confirmation query has latest answer | {{confirmation}} |
| 399  | Send full capabilities               | {{capabilities}} |
| 400  | Bad message received                 | {{errors}}       |
| 403  | Inconsistent message received        | {{consistency}}  |
| 404  | No assertion exists                  | {{client-proto}} |
| 413  | Message too large                    | {{errors}}       |
| 500  | Unspecified server error             | {{errors}}       |
| 501  | Server not capable                   | {{capabilities}} |
| 504  | No assertion available               | {{client-proto}} |

Note that the status codes are chosen to be mnemonically similar to status
codes for HTTP {{?RFC7231}}.

The value of the note-data (22) key, if present, is a UTF-8 encoded string
with additional information about the notification, intended to be displayed
to an administrator to help debug the issue identified by the negotiation.

## Signatures {#signatures}

RAINS supports multiple signature algorithms and hash functions for signing
assertions for cryptographic algorithm agility {{?RFC7696}}. A RAINS signature
algorithm identifier specifies the signature algorithm; a hash function for
generating the HMAC and the format of the encodings of the signature
values in Assertions, Shards, Zones, and Messages, as well as of public key
values in delegation objects.

RAINS signatures have five common elements: the algorithm identifier, a keyspace
identifier, a key phase, a valid-since timestamp, and a valid-until
timestamp. Signatures are represented as an array of these five values followed
by additional elements containing the signature data itself, according to the
algorithm identifier.

The following algorithms are supported:

{: #tabsig title="Defined signature algorithms"}

| Alg ID | Signatures | Hash/HMAC | Format               |
|-------:|------------|-----------|----------------------|
| 1      | ed25519    | sha-512   | See {{eddsa-format}} |
| 2      | ed448      | shake256  | See {{eddsa-format}} |

As noted in {{eddsa-format}}, support for Algorithm 1, ed25519, is REQUIRED;
other algorithms are OPTIONAL.

The keyspace identifier associates the signature with a method for verifying
signatures. This facility is used to support signatures on assertions from
external sources (the extrakey object type). At present, one keyspace identifier
is defined, and support for it is REQUIRED.

| Keyspace ID | Name  | Signature Verification Algorithm               |
|------------:|-------|------------------------------------------------|
| 0           | rains | RAINS delegation chain; see {{cbor-signature}} |

Within the RAINS delegation chain keyspace, the key phase is an unbounded,
unsigned integer matching a signature's key phase to the delegation key phase.
Multiple keys may be valid for a delegation at a given point in time, in order
to support seamless rollover of keys, but only one per key phase and algorithm
may be valid at once. The third element of delegation objects and signatures is
the key phase.

Valid-since and valid-until timestamps are represented as CBOR integers
counting seconds since the UNIX epoch UTC, identified with tag value 1 and
encoded as in section 2.4.1 of {{!RFC7049}}. 

A signature in RAINS is generated over a byte stream representing the data
element to be signed. The signing process is defined as follows:

- Render the element to be signed into a canonical byte stream as specified in
  {{c14n}}.

- Generate a signature on the resulting byte stream according to the algorithm
  selected.

- Add the full signature to the signatures array at the appropriate point in
  the element.

To verify a signature, generate the byte stream as for signing, then verify
the signature according to the algorithm selected.

### Canonicalization {#c14n}

The byte stream representing a data element over which signatures are generated
and verified is a canonicalized CBOR object representing the data element. 

Signatures may be attached to any form of Assertion, as well as to Messages as a whole.

First, to canonicalize signature metadata to allow it to be protected by the
signature, regardless of the type of data element:

- recursively strip all signatures from the content of the data element.
- add a single-element signatures array at the level in the data structure where
  the generated signature will be attached, containing the information common to
  all signatures: the algorithm identifier, a keyspace identifier, a key phase,
  a valid-since timestamp, and a valid-until timestamp, but omitting any
  signature content.

Then follow the canonicalization steps below appropriate for the type of data
element to be signed:

To generate a canonicalized Singular Assertion:

- sort the objects array by ascending order of object type ({{tabobj}}), then by
  ascending numeric or lexicographic order of each subsequent array element in
  the object(s)' representation.
- sort the CBOR map by ascending order of its keys ({{tabmkey}}).

To generate a canonicalized Shard:

- sort the objects array in each assertion contained in the assertions array as
  for Singular Assertions, above.
- sort the assertions array by lexicographic order of the serialized
  canonicalized byte string representing the assertion. Note that this will
  cause the subject array to be sorted in lexicographic order of subject name,
  as well.
- sort the CBOR map by ascending order of its keys ({{tabmkey}}).

To generate a canonicalized Zone:

- sort the objects array in each assertion contained in the assertions array as
  for Singular Assertions, above.
- sort each shard contained in the shards array as for Shards, above.
- sort the assertions array by lexicographic order of the serialized
  canonicalized byte string representing the assertion. Note that this will
  cause the array to be sorted in lexicographic order of subject name,
  as well.
- sort the shards array by lexicographic order of the serialized canonicalized
  byte string representing the shard. Note that this will
  cause the array to be sorted in lexicographic order of shard range, as well.
- sort the CBOR map by ascending order of its keys ({{tabmkey}}).

To generate a canonicalized P-Shard:

- sort the CBOR map by ascending order of its keys ({{tabmkey}}).

To generate a canonicalized Message:

- preserve the order of Sections within the Message
- canonicalize each section as appropriate by following the canonicalization
  steps for the appropriate Section type, above.

### EdDSA signature and public key format {#eddsa-format}

EdDSA public keys consist of a single value, a 32-byte bit string generated as
in Section 5.1.5 of {{!RFC8032}} for Ed25519, and a 57-byte bit string generated
as in Section 5.2.5 of {{!RFC8032}} for Ed448. The fourth element in a RAINS
delegation object is this bit string encoded as a CBOR byte array. RAINS
delegation objects for Ed25519 keys with value k are therefore represented by
the array \[5, 1, phase, k]; and for Ed448 keys as \[5, 2, phase, k].

Ed25519 and Ed448 signatures are are a combination of two non-negative integers,
called "R" and "S" in sections 5.1.6 and 5.2.6, respectively, of {{!RFC8032}}. An
Ed25519 signature is represented as a 64-byte array containing the concatenation
of R and S, and an Ed448 signature is represented as a 114-byte array containing
the concatenation of R and S. RAINS signatures using Ed25519 are therefore the
array \[1, 0, phase, valid-since, valid-until, R|S]; using Ed448 the array \[2, 0,
phase, valid-since, valid-until, R|S].

Ed25519 keys are generated as in Section 5.1.5 of {{!RFC8032}}, and Ed448 keys
as in Section 5.2.5 of {{!RFC8032}}.  Ed25519 signatures are generated from a
normalized serialized CBOR object as in Section 5.1.6 of {{!RFC8032}}, and
Ed448 signatures as in section 5.2.6 of {{!RFC8032}}.

RAINS Server and Client implementations MUST support Ed25519 signatures for
delegation.


## Tokens {#tokens}

Messages and notifications contain an opaque token (2) key, whose
content is a 16-byte array, and is used to link Messages to the Queries they
respond to, and Notifications to the Messages they respond to. Tokens MUST be
treated as opaque values by RAINS servers.

A Message sent in response to a Query (normal and update) MUST contain the token
of the Message containing the Query. Otherwise, the Message MUST contain a token
selected by the server originating it, so that future Notifications can be
linked to the Message causing it. Likewise, a Notification sent in response to a
Message MUST contain the token from the Message causing it (where the new
Message contains a fresh token selected by the server). This allows sending
multiple Notifications within one Message and the receiving server to respond to
a Message containing Notifications (e.g. when it is malformed).

Since tokens are used to link queries to replies, and to link notifications to
messages, regardless of the sender or recipient of a message, they MUST be
chosen by servers to be hard to guess; e.g. generated by a cryptographic random
number generator.

When a server creates a new query to forward to another server in response to
a query it received, it MUST NOT use the same token on the delegated query
as on the received query, unless option 6 Enable Tracing is present in the
received, in which case it MUST use the same token.

## Capabilities {#capabilities}

The capabilities (1) key in a RAINS message allows a the sender of that message
to communicate its capabilities to its peer. Capabilities MUST be sent on the
first message sent from one peer to another. If a peer receives a message from a
counterpart for which it does not have capabilities, it can ask for the next
message to contain full capabilities by sending a message containing
notification 399.

A peer's capabilities can be represented in one of two ways:

- an array of uniform resource names specifying capabilities supported by the
  sending server, taken from the table below, with each name encoded as a
  UTF-8 string.
- a SHA-256 hash of the CBOR byte stream derived from normalizing such an
  array by sorting it in lexicographically increasing order, then serializing
  it.

This mechanism is inspired by {{XEP0115}}, and is intended to be used to
reduce the overhead in exposing common sets of capabilities. Each RAINS server
can cache a set of recently-seen or common hashes, 

The following URNs are presently defined; other URNs will specify future
optional features, support for alternate transport protocols and new signature
algorithms, and so on.

| URN                | Meaning                                                |
|--------------------|--------------------------------------------------------|
| urn:x-rains:tlssrv | Listens for TLS/TCP connections (see {{transport-tls}} |
| urn:x-rains:udpsrv | Accepts messages via UDP (see {{transport-udp}})       |

A RAINS server MUST NOT assume that a peer server supports a given capability
unless it has received a message containing that capability from that server. An
exception are the capabilities indicating that a server listens for connections
using a given transport protocol; servers and clients can also learn this
information from RAINS itself (given redirection and service-info assertions for
a named zone) or from external configuration.

# Zone File Format {#zonefiles}

\[EDITOR'S NOTE: derive this from the zonefile parser]

# RAINS Protocol {#protocol}

## Transport Bindings {#transport}

### TLS over TCP {#transport-tls}

### UDP {#transport-udp}

### Heartbeat Messages {#heartbeat}

## Protocol Dynamics {#base-proto}

\[EDITOR'S NOTE: things that must appear here listed below]

- MUST check that a P-Shard provides a negative proof for a query before returning it

## Client Protocol {#client-proto}

## Publication Protocol {#pub-proto}

## Enforcing Assertion Consistency {#consistency}

\[EDITOR'S NOTE: inconsistency is only unequivocal when P ^ !P for a given
assertion at the same instant T]

## Error Handling {#errors}

# Operational Considerations

# Security Considerations

# IANA Considerations

# Acknowledgments



# Old content below
<!---->


## Address to Object Mapping

In contrast to the current domain name system, information about addresses is
stored in a completely separate tree, keyed by address and prefix. An address
assertion consists of the following elements:

- Context: name of the context in which the assertion is valid;
  see {{context-in-address-assertions}}.
- Subject: address about which the assertion is made, consisting of an address
  family, address, and prefix length. A subject may be a network address (where
  the prefix length is less than the address length for the given address
  family) or a host address (where the prefix length is equal to the address
  length for the given address family)
- Type: the type of information about the Subject contained in the assertion.
  Each Assertion is about a single type of data.
- Object: the data of the indicated type associated with the Subject
- Signatures: one or more signatures generated by the authority for the
  Assertion. Signatures contain a time interval during which they are considered
  valid, as in {{signatures-in-assertions}}.

The following object types are available:

- Delegation: the authority associated with the subject network address. The
  Object contains a public key by which the authority can be identified. Only
  available for network address subjects.
- Redirection: The name(s) of one or more RAINS servers providing authority
  service for the authority associated with the subject network address. The
  Object contains a set of names. Only available for network address subjects.
- Name: one or more names associated with the subject network address.
  The Object contains a set of names. Only available for host address subjects.
- Zone-Registrant: Information about the organization responsible for a network.
  Only available for network address subjects.

Queries for addresses are similar to those for names, and consist of the following information elements:

- Context: Context in which the query is made; this must match the assertion
  context as in {{context-in-address-assertions}}.
- Subject: the address about which the query is made, consisting of an address
  family, address, and prefix length.
- Types: a set of assertion types the querier is interested in, as above.
- Valid-Until: an optional client-generated timestamp for the query after which
  it expires and should not be answered.
- Query Token: a client-generated token for the query, which can be used
  in the answer to refer to the query.

### Context in Address Assertions

Just as in forward Assertions, Assertion contexts are used in address
assertions to determine the scope of an address assertion, and the signature
chain used to verify it.

- The global addressing context for each address family is identified by the
  special context name '.'. For both IPv4 and IPv6 addresses, this is rooted at
  IANA, which delegates to the RIRs, which then delegates to LIRs and to
  address-holding registries.
- Local contexts associated with a given authority in a forward tree can also
  make assertions about addresses. As with contexts in forward assertions, the
  authority-part and the context-part of a local context name are divided by a
  context marker ('cx--'). The authority-part directly identifies the authority
  whose key was used to sign the address assertion; address assertions within a
  local context are only valid if signed by the identified authority.
  Authorities have complete control over how the contexts under their
  numberspaces are arranged, and over the addresses within those contexts.

Each local context may have a root address space zone (0/0), but these root
address spaces may only delegate addresses that are reserved for local use
{{!RFC1918}} {{!RFC4193}}. Local context assertions for other addresses are
invalid.

# CBOR Data Model {#cbor}

\[EDITOR'S NOTE: frontmatter moved, delete me]

## Symbol Table {#cbor-symtab}

\[EDITOR'S NOTE: content moved, delete me]

## Message {#cbor-message}

\[EDITOR'S NOTE: content moved, delete me]

## Assertion body {#cbor-assertion}

\[EDITOR'S NOTE: content moved, delete me]

## Shard body {#cbor-shard}

\[EDITOR'S NOTE: content moved, delete me]

### Sorting Shards {#cbor-shard-sorting}

Shards are sorted lexicographically by their cbor encoded byte string in
ascending order. That means, they are sorted by the following elements in the
mentioned order: zone name, context, range begin, range end, content, signature
meta data.

## P-Shard body {#cbor-P-Shard}

\[EDITOR'S NOTE: content moved, delete me]

### Sorting P-Shard {#cbor-P-Shard-sorting}

P-Shards are sorted lexicographically by their cbor encoded byte string in
ascending order. That means, they are sorted by the following elements in the
mentioned order: zone name, context, range begin, range end, hash family, number
of hash functions, filter, signature meta data.

## Zone body {#cbor-zone}

\[EDITOR'S NOTE: content moved, delete me]

## Query body {#cbor-query}

\[EDITOR'S NOTE: content moved, delete me]

## Address Assertion body {#cbor-revassert}

Assertions about addresses are similar to assertions about names, but keyed by
address and restricted in terms of the objects they can contain. An Address
Assertion body is a map which MUST contain the signatures (0), subject-addr (5),
context (6), and objects (7) keys.

The value of the signatures (0) key is an array of one or more Signatures as
defined in {{cbor-signature}}.

The value of the subject-addr (5) key is a three element CBOR array. The first
element of the array is the address family encoded as an object type, 2 for
IPv6 addresses and 3 for IPv4 addresses. The second element is the prefix
length encoded as an integer, 0-128 for IPv6 and 0-32 for IPv4. The third
element is the address, encoded as in {{cbor-object}}. Subject addresses with
the maximum prefix length for the address family are subject host addresses,
and are nameable; subject addresses with less than the maximum prefix length
are subject network addresses, and are delegatable.

The value of the context (6) key is a UTF-8 string containing the name of the
context in which the Address Assertion is valid. See
{{context-in-address-assertions}}.

The value of the objects (7) key is an array of objects, as defined in
{{cbor-object}}. Only object types redirection, delegation, and registrant are
available for subject network addresses, and only object type name is
available for subject host addresses.

## Address Query body {#cbor-revquery}

Queries for assertions about addresses are similar to queries for assertions
about names, but have semantic restrictions similar to those for Address
Assertions.

An Address Query body is a map. Queries MUST contain the subject-addr (5),
context (6), query-types (10), and query-expires (12) keys. Address Queries MAY
contain query-opts (13) key.

The value of the subject-addr (5) key is a three-element CBOR array. The first
element of the array is the address family encoded as an object type, 2 for
IPv6 addresses and 3 for IPv4 addresses. The second element is the prefix
length encoded as an integer, 0-128 for IPv6 and 0-32 for IPv4. The third
element is the address, encoded as in {{cbor-object}}.

The value of the context (6) key is a UTF-8 encoded string containing the name
of the context for which the Query is valid. Unlike queries for names, Address
Queries can only pertain to a single context. See
{{context-in-address-assertions}} for more.

The value of the query-types (10) key is an array of integers encoding the
type(s) of objects (as in {{cbor-object}}) acceptable in answers to the query.
All values in the query-type array are treated at equal priority: \[4,5] means
the querier is equally interested in both redirection and delegation for the
subject-addr. An empty query-types array indicates that objects of any type are
acceptable in answers to the query.

The value of the query-expires (12) key is a CBOR integer
counting seconds since the UNIX epoch UTC, identified with tag value 1 and
encoded as in section 2.4.1 of {{!RFC7049}}. After the query-expires time, the
query will have been considered not answered by the original issuer.

The value of the query-opts (13) key, if present, is an array of integers in
priority order of the querier's preferences in tradeoffs in answering the
query, as in {{tabqopts}}. See {{cbor-query}} for more.

An Address Assertion with a more-specific prefix is preferred over a
less-specific in response to a Address Query.

## Notification body {#cbor-notification}

\[EDITOR'S NOTE - content moved up]

## Object {#cbor-object}

\[EDITOR'S NOTE - content moved up]

## Signatures, delegation keys, and RAINS infrastructure keys {#cbor-signature}


## Capabilities {#cbor-capabilities}



# Canonical signing format {#signing-format}

\[EDITOR'S NOTE: to define, based on CBOR canonicalization, once this is implemented.]

# RAINS Protocol Definition {#protocol-def}

As noted in {{cbor}}, RAINS is a message-exchange protocol that uses CBOR
{{!RFC7049}} as its framing. Since CBOR is self-framing -- a CBOR parser can
determine when a CBOR object is complete at the point at which it has read its
final byte -- RAINS requires no external framing. It can therefore run over
any streaming, multistreaming, or message-oriented transport protocol. In
order to protect query confidentiality, and support rapid deployment over a
ubiquitously implemented transport, RAINS is defined in this document to run
over persistent TLS 1.3 connections {{!RFC8446}} over TCP {{!RFC0793}} with
mutual authentication between servers, and authentication of servers by
clients. The TLS certificates of RAINS server peers can be verified as
specified in the cert-info assertions for those servers.

RAINS servers MUST support this transport; future transports can be negotiated
using the capabilities mechanism after bootstrapping using TLS 1.3. As RAINS
is an experimental protocol, RAINS servers listen on port 1022 {{!RFC4727}} for
connections from other RAINS servers and clients. RAINS servers should strive
to keep connections open to peer servers, unless it is clear that no future
messages will be exchanged with those peers, or in the face of resource
limitations at either peer. If a RAINS server needs to send a message to
another RAINS server to which it does not have an open connection, it attempts
to open a connection with that server.

This section describes the operation of the protocol as used among RAINS
servers. A simplified version of the protocol for client access is described in
{{protocol-client}}, and a simplified version of the protocol for publication by
authorities is described in {{protocol-publish}}.

## Bootstrapping

At startup, a server performing recursive lookup MUST have access to at least
one of each of these three assertion types: a self-signed delegation assertion
of the root zone, a redirection assertion containing the name of an
authoritative root name server, and an ip4 or ip6 assertion of the root name
server mentioned in the redirection assertion. These assertions must be obtained
through a secure out of band mechanism. For a caching server, it is sufficient
to have a connection to a recursive resolver which does the lookup on its
behalf.

When a zone authority delegates a part of its namespace to a subordinate, it
MUST sign and serve the assertions of the three above mentioned types. This
information is necessary for a recursive resolver to determine in a recursive
lookup where to ask for a more specific answer and to validate the response.

## Allowed Inconsistencies

For RAINS to work in a highly dynamic environment, some time-bounded
inconsistencies are allowed to occur. On the one hand, an authority wants to
prove nonexistence of a name for a duration of time to make caching possible to
reduce query latency and reduce load on its naming servers. On the other hand,
an authority wants to add the name of a new delegation as quickly as possible
and also allow its customers to make changes available quickly. Assuming an
authority resigns sections every x seconds, then any inconcistency can occur at
most x seconds. At the point in time a section is signed, its content MUST
represent the state of the zone at that point in time. The following actions
result in allowed inconsistencies:

- Creation of a new assertion: Shards and P-Shards in range, and zones
  signed before this assertion was created and which are still valid, prove that
  this assertion is nonexistent, although it does now.
- Changed object value of an assertion: Shards in range and zones signed before
  this assertion was created and which are still valid, prove that this
  assertion is nonexistent, althoug it does now.
- Expiration of an assertion: Shards and P-Shards in range, and zones
  signed before this assertion has expired and which are still valid, prove that
  this assertion exists, although it does not anymore.
- Revocation of an assertion: Same inconsistencies as for expiration of an
  assertion with the addition, that the assertion itself might still be cached
  and served although it has been revoked.

Two sections for proving nonexistence (shard, P-Shard or zone) which have
an overlapping range and validity time where in between the signing of the two
sections any of the above mentioned actions leading to inconsistencies happend,
become inconsistent as well. One of them has the old view, while the newer one
has the updated view about the assertion. Note that there is no inconsistency
between a P-Shard and any other section proving nonexistence if only the
object value of the assertion has changed (a P-Shard does not store this
information).

Note that most assertions are consistent between each other as the union of them
is considered to be the valid state. However, there are few exceptions mentioned
in {{runtime-consistency-checking}}.

## Message processing {#protocol-processing}

Once a transport is established, any server may validly send a message with
any content to any other server. A client may send messages containing queries
to servers, and a server may sent messages containing anything other than
queries to clients.

Upon receipt of a message, a server or client attempts to parse it.

If the server or client cannot parse the message at all, it returns a 400 Bad
Message notification to the peer. This notification may have a null token if the
token cannot be retrieved from the message.

If the server or client can parse the message, it:

- notes the token on the message. This token MUST be present on any
  messages sent in reply to this message.
- processes any capabilities present, replacing the set of capabilities known
  for the peer with the set present in the message. If the present
  capabilities are represented by a hash that the server does not have in its
  cache, it prepares a notification of type 399 "Capability hash not
  understood" to send to its peer.
- splits the contents into its constituent message sections, and verifies that
  each is acceptable. Specifically, queries are not accepted by clients (see
  {{protocol-client}}), and 404 No Assertion Exists notifications are not
  accepted by servers. If a message contains an unacceptable section, the server
  or client returns a 400 Bad Message notification to its peer, and ceases
  processing of the message.

On receipt of an assertion, shard, P-Shard, or zone message section, a
server:

- verifies its consistency (see {{runtime-consistency-checking}}). If the
  section is not consistent, it prepares to send a notification of type 403
  Inconsistent Message to the peer, and discards the section. Otherwise, it:
- determines whether it answers an outstanding query; if so, it prepares to
  forward the section to the server that issued the query.
- determines whether it is likely to answer a future query, according to its
  configuration, policy, and query history; if so, it caches the section.

On receipt of an assertion, shard, P-Shard or zone message section, a
client:

- determines whether it answers an outstanding query; if so, it considers the
  query answered. It then:
- determines whether it is likely to answer a future query, according to its
  configuration, policy, and query history; if so, it caches the section.

On receipt of a query, a server:

1. determines whether it has expired by checking the query-expires value. If so,
   it drops the query silently. If not, it:
2. determines whether it has at least one stored assertion answering the query.
   If so, it returns the assertion(s) with the longest validUntil value that is
   already valid. If not, it:
3. checks whether the query specifies option 1 and/or 9. If so, it:
   - determines whether the chosen option is in compliance with the server's
     configuration and policy. If so, and:
     - option 9 is set and option 1 is not, it continues with step 4.
     - option 1 is set, it checks whether it has a cached section to
       proof nonexistence. If so:
       - and option 9 is not set, it returns the section with the shortest size
         or the signature of the longest remaining validity to the peer that
         issued the query depending on the server's policy. \[EDITOR'S NOTE: Add a
         query option for this decision?]
       - and option 9 is set, it might send a 211 notification back to the
         client, depending on the server's configuration. Independent of the
         previous decision it then continues with step 4. 
     If not, the server overwrites the query's option according to its 
     configuration and policy and processes it as above with the adapted option.
   If not, it:
4. checks to see whether the query specifies option 4 (cached answers only). If
   so, and if option 5 (expired assertions acceptable) is also specified, it
   then checks to see if it has any cached sections that answer the query on
   which signatures are expired; otherwise, processing stops, and the server
   returns a 504 No Assertion Available notification, as if the query had
   instantly expired. If the query does not specify option 4, delegation
   proceeds, and the server:
5. determines whether it has other non-authoritative servers it can forward the
   query to, according to its configuration and policy, and in compliance with
   any query options (see {{cbor-query}}). If so, it prepares to forward the
   query to those servers, noting the reply for the received query depends on
   the replies for the forwarded query. If not, it:
6. determines the responsible authority servers for the zone containing the
   query name in the query for the context requested, and forwards the query to
   those authority servers, noting the reply for the received query depends on
   the reply for the forwarded query.

If query delegation fails to return an answer within the maximum of the
valid-until time in the received query and a configured maximum timeout for a
delegated query, the server prepares to send a 504 No assertion available
response to the peer from which it received the query.

When a server creates a new query to forward to another server in response to
a query it received, it SHOULD NOT use the same token on the delegated query
as on the received query, unless option 6 Enable Tracing is present in the
received, in which case it MUST use the same token. The Enable Tracing option
is designed to allow debugging of query processing across multiple servers, It
SHOULD only be enabled by clients designed explicitly for debugging RAINS
itself, and MUST NOT be enabled by default by client resolvers.

When a server creates a new query to forward to another server in response to a
query it received, and the received query contains a query-expires time, the
delegated query MUST NOT have a query-expires time after that in the received
query. If the received query contains no query-expires time, the delegated query
MAY contain a query- expires time of the server's choosing, according to its
configuration.

On receipt of an assertion update query, a server:

- determines whether it has expired by checking the query-expires value. If so,
  it drops the query silently. If not, it:
- determines whether it is the authoritative server of the queried name. If so,
  - it checks if the hashed assertion is still the assertion currently valid
    with the highest validUntil time for the given name, context, type and
    object value. In that case it returns a 200 notification message which
    contains the hash value of the assertion. If not, it:
  - determines whether there is at least one assertion for the same name, type
    and object value which is already valid and its validUntil time is higher.
    If so, the assertion with the highest validUntil value is returned. If not:
  - the assertion must have been revoked in the meantime and either a 210
    notification message containing the hash value of the assertion or a section
    proofing nonexistence is returned. If such a section exists it MUST be
    returned.
  If not, it:
- determines whether it has other non-authoritative servers it can forward the
  query to, according to its configuration and policy. If so, it prepares to
  forward the query to those servers, noting the reply for the received query
  depends on the replies for the forwarded query. If not, it:
- determines the responsible authority servers for the zone containing the query
  name and forwards the query to those authority servers, noting the reply for
  the received query depends on the reply for the forwarded query.

If the server does not obtain an answer within the maximum of the valid-until
time in the received query and a configured maximum timeout for an assertion
update query, the server sends a 504 No assertion available response to the peer
from which it received the query.

When a server creates a new assertion update query to forward to another server
in response to an assertion update query it received, it SHOULD NOT use the same
token on the new query as on the received query, unless query option 6 Enable
Tracing is present in the received query, in which case it MUST use the same
token. The Enable Tracing option is designed to allow debugging of query
processing across multiple servers, It SHOULD only be enabled by clients
designed explicitly for debugging RAINS itself, and MUST NOT be enabled by
default by client resolvers.

When a server creates a new assertion update query to forward to another server
in response to an assertion update query it received, and the received query
contains a query-expires time, the new query MUST NOT have a query-expires time
after that in the received query. If the received query contains no
query-expires time, the new query MAY contain a query- expires time of the
server's choosing, according to its configuration.

On receipt of an nonexistence update query, a server:

- determines whether it has expired by checking the query-expires value. If so,
  it drops the query silently. If not, it:
- determines whether it is the authoritative server of the queried name. If so,
  - it checks if it has a valid assertion for the queried context, subject-name
    and type. In this case it returns the assertion. If not, it:
  - determines whether it has an already valid zone, P-Shard, or shard in
    the range of the queried fully-qualified name, in a matching context, and
    with a higher validUntil value. If so, the section with the highest
    validUntil value is returned. If not, it:
  - knows that the received shard, P-Shard, or zone is still the most
    recent one and a 200 notification message containing the hash value of the
    section is returned.
  If not, it:
- determines whether it has other non-authoritative servers it can forward the
  query to, according to its configuration and policy. If so, it prepares to
  forward the query to those servers, noting the reply for the received query
  depends on the replies for the forwarded query. If not, it:
- determines the responsible authority servers for the zone containing the query
  name and forwards the query to those authority servers, noting the reply for
  the received query depends on the reply for the forwarded query.

If the server does not obtain an answer within the maximum of the valid-until
time in the received query and a configured maximum timeout for an assertion
update query, the server sends a 504 No assertion available response to the peer
from which it received the query.

When a server creates a new nonexistence update query to forward to another
server in response to an nonexistence update query it received, it SHOULD NOT
use the same token on the new query as on the received query, unless query
option 6 Enable Tracing is present in the received query, in which case it MUST
use the same token. The Enable Tracing option is designed to allow debugging of
query processing across multiple servers, It SHOULD only be enabled by clients
designed explicitly for debugging RAINS itself, and MUST NOT be enabled by
default by client resolvers.

When a server creates a new nonexistence update query to forward to another
server in response to an nonexistence update query it received, and the
received query contains a query-expires time, the new query MUST NOT have a
query-expires time after that in the received query. If the received query
contains no query-expires time, the new query MAY contain a query- expires time
of the server's choosing, according to its configuration.

On receipt of a notification, a server's behavior depends on the notification type:

- For type 100 "Connection Heartbeat", the server does nothing: these null
  messages are used to keep long-lived connections open in the presence of
  network behaviors that may drop state for idle connections.
- For type 399 "Capability hash not understood", the server prepares to send a
  full capabilities list on the next message it sends to the peer.
- For type 504 "No assertion available", the server checks the token on the
  message, and prepares to forward the assertion to the associated query.
- For type 413 "Message too large" the server notes that large messages may not
  be sent to a peer and tries again (see {{protocol-limits}}), or logs the error
  along with the note-data content.
- For type 400 "Bad message", type 403 "Inconsistent message", type 500
  "Server error", or type 501 "Server not capable", the server logs the error
  along with the note-data content, as these notifications generally represent
  implementation or configuration error conditions which will require human
  intervention to mitigate.

On receipt of a notification, a client's behavior depends on the notification type:

- For type 100 "Connection Heartbeat", the client does nothing, as above.
- For type 399 "Capability hash not understood", the client prepares to send a
  full capabilities list on the next message it sends to the peer.
- For type 404 "No assertion exists", the client takes the query to be
  unanswerable. It may reissue the query with query option 7 to do the
  verification of nonexistence again, if the server from which it received the
  notification is untrusted.
- For type 413 "Message too large" the client notes that large messages may 
  not be sent to a peer and tries again (see {{protocol-limits}}), or logs
  the error along with the note-data content.
- For type 400 "Bad message", type 403 "Inconsistent message", type 500
  "Server error", or type 501 "Server not capable", the client logs the error
  along with the note-data content, as these notifications generally represent
  implementation or configuration error conditions which will require human
  intervention to mitigate.

The first message a server or client sends to a peer after a new connection is
established SHOULD contain a capabilities section, if the server or client
supports any optional capabilities. See {{cbor-capabilities}}.

If the server is configured to keep long-running connections open, due to the
presence of network behaviors that may drop state for idle connections, it
SHOULD send a message containing a type 100 Connection Heartbeat notification
after a configured idle time without any messages containing other content
being sent.

In general, servers should follow the principles laid out in
{{?I-D.iab-protocol-maintenance}}. A malformed message section, or a message
section with any invalid (but not expired) signature, should be dropped and log.
A malformed message section or invalid signature should not, however, result in
other sections in the same message being dropped, except as explicitly noted
above.

## Message Transmission

As noted in {{protocol-processing}} many messages are sent in reply to messages
received from peers. Servers may also originate messages on their own, based
on their configuration and policy:

- Proactive queries to retrieve assertions, shards, and zones for which all
  signatures have expired or will soon expire, for cache management purposes.
- Proactive push of assertions, shards, and zones to other servers, based on
  query history or other information indicating those servers may query for
  the assertions they contain.

## Message Limits {#protocol-limits}

RAINS servers MUST accept messages up to 65536 bytes in length, but MAY accept
messages of greater length, subject to resource limitations of the server. A
server with resource limitations MUST respond to a message rejected due to
length restrictions with a notification of type 413 (Message Too Large). A
server that receives a type 413 notification must note that the peer sending
the message only accepts messages smaller than the largest message it's
successfully sent that peer, or cap messages to that peer to 65536 bytes in
length.

Since a singular assertion with a single Ed25519 signature requires on the order of
180 bytes, it is clear that many full zones won't fit into a single minimum
maximum-size message. Authorities are therefore encouraged to publish zones
grouped into shards that will fit into 65536-byte messages, to allow servers
to reply using these shards when full-zone transfers are not possible due to
message size limitations.

## Runtime Consistency Checking

The data model used by the RAINS protocol allows inconsistent information to
be asserted, all resulting from misconfigured or misbehaving authority
servers. The following types of inconsistency are possible:

- A shard omits an assertion within its range which is valid at the same
  time as the shard.
- A zone omits an assertion within its zone which is valid at the same time
  as the zone.
- An address assertion contains an object that is not allowed (see {{cbor-revassert}})
- An assertion prohibited by its zone's nameset is valid at the same time
  as the zone's nameset assertion.
- A zone contains a valid reflexive assertion of a given object type at the same
  time that its superordinate zone contains a valid assertion of the same type.
- Delegations to more than one key are simultaneously valid for a given context,
  zone, signature algorithm, and key phase.

RAINS relies on runtime consistency checking to mitigate inconsistency: each
server receiving an assertion, shard, or zone SHOULD, subject to resource
constraints, ensure that it is consistent with other information it has, and if
not, discard all inconsistent assertions, shards, and zones in its cache, log
the error, and send a 403 Inconsistent Message to the source of the message.

## Integrity and Confidentiality Protection

Assertions are not valid unless they contain at least one signature that can
be verified from the chain of authorities specified by the name and context on
the assertion; integrity protection is built into the information model. The
infrastructure key object type allows keys to be associated with RAINS
servers in addition to zone authorities, which allows a client to
delegate integrity verification of assertions to a trusted query service (see
{{protocol-client}}).

Since the job of an Internet naming service is to provide publicly-available
information mapping names to information needed to connect to the services
they name, confidentiality protection for assertions is not a goal of the
system. Specifically, the information model and the mechanism for proving
nonexistence of an assertion is not designed to provide resistance against
zone enumeration.

On the other hand, confidentiality protection of query information is crucial.
Linking naming queries to a specific user can be nearly as useful to build a
profile of that user for surveillance purposes as full access to the clear
text of that client's communications {{?RFC7624}}. In this revision, RAINS uses
TLS to protect communications between servers and between servers and clients,
with certificate information for RAINS infrastructure stored in RAINS itself.
Together with hop-by-hop confidentiality protection, query options, proactive
caching, default use of non-persistent tokens, and redirection among servers
can be used to mix queries and reduce the linkability of query information to
specific clients.

## Cooperative Delegation Distribution

Regardless of any other configuration directive, a RAINS server MUST be prepared
to provide a full chain of delegation assertions from the appropriate delegation
root to the signature on any assertion it gives to a peer or a client, whether
as additional assertions on a message answering a query, or in reply to a
subsequent query. This property allows RAINS servers to maintain a full
delegation tree.

# RAINS Client Protocol {#protocol-client}

The protocol used by clients to issue queries to and receive responses from a
query service is a subset of the full RAINS protocol, with the following
differences:

- Clients only process assertion, shard, zone, and notification sections;
  sending a query to a client results in a 400 Bad Message notification.
- Clients never listen for connections; a client must initiate and maintain a
  transport session to the query server(s) it uses for name resolution.
- Servers only process query and notification sections when connected to
  clients; a client sending assertions to a server results in a 400 Bad
  Message notification.

Since signature verification is resource-intensive, clients delegate signature
verification to query servers by default. The query server signs the message
containing results for a query using its own key (published as an infrakey
object associated with the query server's name), and a validity time
corresponding to the signature it verified with the longest lifetime,
stripping other signatures from the reply. This behavior can be disabled by a
client by specifying query option 7, allowing the client to do its own
verification.

# RAINS Publication Protocol {#protocol-publish}

The protocol used by authorities to publish assertions to an authority service
is a subset of the full RAINS protocol, with the following differences:

- Servers only process assertion, shard, zone, and notification sections when
  connected to publishers; sending a query to a server via the publication
  procotol results in a 400 Bad Message notification. Servers only process
  notifications for capability negotiation purposes (see {{cbor-capabilities}}).
- Publishers only process notification sections; sending a query or assertion to
  a publisher results in a 400 Bad Message notification.

# Deployment Considerations

The following subsections discuss issues that must be considered in any
deployment of RAINS at scale.

## Assertion Lifetime Management

An assertion can contain multiple signatures, each with a different lifetime.
Signature lifetimes are equivalent to a time to live in the present DNS:
authorities should compute a new signature for each validity period, and make
these new signatures available when old ones are expiring.

Since assertion lifetime management is based on a real-time clock expressed in
UTC, RAINS servers MUST use a clock synchronization protocol such as NTP
{{?RFC5905}}.

RAINS servers MAY coalesce assertion lifetimes, e.g. using only the most recent
valid-until time in their cache management. This implies that an assertion with
valid signatures in time intervals (T1, T2) and (T3, T4) such that T3 > T2 may
be cached during the interval (T2, T3) as well. Authorites MUST NOT rely on
non-caching or non-availability of assertions during such intervals.

## Secret Key Management

The secret keys associated with public keys for each RAINS server (via
infrakey objects) must be available on that server, whether through a hardware
or software security device, so they can sign messages on demand; this is
particularly important for query servers. In addition, the secret keys
associated with TLS certificates for each server (published via certinfo
objects) must be available as well in order to establish TLS sessions.

However, storing zone secret keys (associated via delegation objects) on RAINS
servers would represent a more serious operational risk. To keep this from
being necessary, authority servers have an additional signer interface, from
which they will accept and cache any assertion, shard, or zone for which they
are authority servers until at least the end of validity of the last
signature, provided the signature is verifiable.

## Public Key Management

As signature lifetime is used to manage assertion lifetime, and key rotation
strategies may be used both for revocation as well as operational flexibility
purposes, RAINS presents a much more dynamic key management environment than
that presented by DNSSEC.

### Key Phase and Key Rotation

Each signature and public key in a RAINS message is associated with a key phase,
allowing multiple keys to be valid for a given authority at any given time. For
example, given two key phases and a key validity interval of one day, a phase 0
key would be valid from 00:00 on day 0 to 00:00 on day 1, and a phase 1 key
valid from 12:00 on day 0 to 12:00 on day 1. When the phase 0 key expires, it
would be replaced by a new phase 0 valid from 00:00 on day 1 to 00:00 on day 2,
and so on.

Since the end time of the validity of a signature on an assertion is the maximum
of the validity of the signatures on each of the delegations in the delegation
chain from the root, key rotation avoids mass expiration of assertions, at the
cost of requiring one valid signatures per key phase on at least all delegation
assertions. Key rotation schedules are a matter of authority operational policy,
but key validity intervals should be longer the closer in the delegation chain
an assertion is to the root.

### Next Key Assertions

Another problem this dyanmic envrionment raises is how a zone authority
communicates to its superordinate that it would like to begin using a new public
key to sign its assertions.

This can be done out of band, using private APIs provided by the superordinate
authority. Through the nextkey object type, RAINS provides a way for a future
public key to be shared with the superordinate authority (and all other
queriers) in-band. An authority that wishes to use a new key publishes a
reflexive nextkey assertion (i.e., in its own zone, with subject @) with the new
public key and a requested valid-since and valid-until time range. The
superordinate issues periodic queries for nextkey assertions from its
subordinate zone, or the subordinate pushes these assertions to an intermediate
service designated to receive them. When the superordinate receives a nextkey,
and it decides it wants to delegate to the new key, it creates and signs a
delegation assertion.

This process is not mandatory: the superordinate is free to ignore the request,
or to use a different time range, depending on its policy and/or the status of
its business relationship with the subordinate. The subordinate can discover
this, in turn, using its own RAINS queries, or through the delegation assertions
being similarly pushed to a designated intermediate service.

## Unsigned Contained Assertions

Although RAINS supports Shards and Zones containing unsigned assertions,
protecting the integrity of those Assertions by the signature on the Shard or
Zone, it is RECOMMENDED that authorities sign each Assertion, even those
contained within a Shard or Zone, in order to minimize the size of positive
answers to queries.

## Query Service Discovery

A client that will not do its own verification must be able to discover the
query server(s) it should trust for resolution. Integration with DHCP is left
to a future revision of this document.

In any case, clients MUST provide a configuration interface to allow a user to
specify (by address or name) and/or constrain (by certificate property) a
preferred/trusted query server. This would allow client on an untrusted network
to use an untrusted locally-available query server to discover a preferred query
server (doing key verification on its own for bootstrapping), before connecting
to that query server for normal name resolution.

## Transition using translation between RAINS and DNS information models {#dns-transition}

Full adoption of RAINS would require changes to every client device (replacing
DNS stub resolvers with RAINS clients) and name server on the Internet. In
addition, most client software would need to change, as well, to get the full
benefits of explicit context in name resolution. This is an unrealistic goal.

RAINS servers can, however, coexist with Domain Name System servers and
clients during an indefinite transition period. RAINS assertions can be
algorithmically translated into DNS answers, and RAINS queries can be
algorithmically translated into DNS queries, by RAINS to DNS gateways, given
the mostly compatible information models used by the two.

While DNSSEC and RAINS keys for equivalent ciphersuites are compatible with
each other, there is no equivalent to query option 7 for gateways, since the
RAINS signatures are generated over the RAINS byte stream for an assertion, not
the DNS byte stream. Therefore, RAINS to DNS gateways must provide verification
services for DNS clients. DNS over TLS {{?RFC7858}} SHOULD be used between the
DNS client and gateway to ensure confidentiality and integrity for queries and
answers.

Object type mappings are as follows:

- Objects of type name can (largely) be represented as CNAME RRs.
- Objects of type ip6-addr can be represented as AAAA RRs.
- Objects of type ip4-addr can be represented as A RRs.
- Objects of type redirection can be represented as NS RRs.
- Objects of type cert-info can be represented as TLSA RRs 
- Objects of type service-info can be represented as SRV RRs.

There are a few object types without mappings:

- Objects of type delegation can be represented as DS RRs, and signatures as
  RRSIG RRs, but since these keys are verified by the gateway, there is no need
  to represent this information to the client.
- Objects of type infrakey cannot be represented in DNS, but are irrelevant for
  DNS translation of RAINS messages, since DNS does not support server signing
  of responses.
- Objects of type registrar and registrant cannot be represented in DNS; clients
  can use WHOIS instead. In addition, RRTYPEs could be added for them in the
  future if RAINS sees significant deployment with DNS as a front-end protocol.
- Objects of type nameset cannot be represented in DNS; the current equivalent
  are the IDNA parameters maintained by IANA (for the DNS root zone only) at
  https://www.iana.org/assignments/idna-tables-6.3.0/idna-tables-6.3.0.xhtml.

When translating a DNS query from a client to a RAINS query for that client,
client options can be set on a per-server, per-client, or per-query basis
using some out of band configuration options.

When translating a RAINS assertion to a DNS answer, the gateway can use the
time to expiry for the verified signature as the TTL.

There is no method for exposing context information in a DNS query or answer.
Therefore, queries and answers at a RAINS gateway are only supported for the
global context ".".

# Experimental Design and Evaluation

The protocol described in this document is intended primarily as a prototype
for discussion, though the goal of the document is to specify RAINS completely
enough to allow independent, interoperable implementation of clients an
servers. The massive inertia behind the deployment of the present domain name
system makes full deployment as a replacement for DNS unlikely. Despite this,
there are some criteria by which the success of the RAINS experiment may be
judged:

First, deployment in simulated or closed networks, or in alternate Internet
architectures such as SCION, allows implementation experience with the
features of RAINS which DNS lacks (signatures as a first-order delegation
primitive, support for explicit contexts, explicit tradeoffs in queries,
runtime availability of registrar/registrant data, and nameset support),
which in turn may inform the specification and deployment of these features
on the present DNS.

Second, deployment of RAINS "islands" in the present Internet alongside DNS on
a per-domain basis would allow for comparison between operational and
implementation complexity and efficiency and benefits derived from RAINS'
features, as information for future development of the DNS protocol.

# IANA Considerations

The present revision of this document has no actions for IANA.

The authors have registered the CBOR tag 15309736 to identify RAINS messages
in the CBOR tag registry at 
https://www.iana.org/assignments/cbor-tags/cbor-tags.xhtml.

RAINS servers currently listen for connections from other servers on Port
1022. Future revisions of this document may specify a different port,
registered with IANA via Expert Review {{?RFC5226}}.

The symbol table in this document in {{cbor-symtab}}, the notification code
table in {{cbor-notification}}, and the signature algorithm table in 
{{cbor-signature}} may be candidates for IANA registries in future revisions 
of this document.

The urn:x-rains namespace used by the RAINS capability mechanism in 
{{cbor-capabilities}} may be a candidate for replacement with an IANA-registered
namespace in a future revision of this document.

# Security Considerations

This document specifies a new, experimental protocol for Internet name
resolution, with mandatory integrity protection for assertions about names
built into the information model, and confidentiality for query information
protected on a hop-by-hop basis. See especially {{signatures-in-assertions}},
{{integrity-and-confidentiality-protection}}, {{cbor-signature}},
{{cbor-certinfo}}, and {{secret-key-management}} for security-relevant details.

With respect to the resistance of the protocol itself to various attacks, we
consider a few potential attacks against RAINS servers and RAINS clients in the
subsections below:

## Server state exhaustion

\[EDITOR'S NOTE: detail this attack: attacker can create domain, use
long-validity queries to exhaust state at server. defense: server can consider
shorter validity time than that requested, but not longer. attack: attacker can
push garbage assertions proactively. defense: server doesn't accept assertions
it's never seen a query for. how to handle an attacker that pushes assertions
and queries? attack: attacker can push garbage delegations, exhausting
delegation chain cache. defense: server doesn't accept sigs for domains it
doesn't know about, but what about a domain with hundreds of valid delegations?
in all cases, blacklisting both clients and domains seems like a good idea.]

## Query relay attacks

\[EDITOR'S NOTE: detail this attack: attacker can cause traffic overload at a
targeted intermediate or authority service by crafting queries and sending them
via multiple query services. There is no amplification here, but a
concentration, with indirection that makes tracing difficult.]

# Acknowledgments

Thanks to Daniele Asoni, Laurent Chuat, Markus Deshon, Ted Hardie, Joe
Hildebrand, Tobias Klausmann, Steve Matsumoto, Adrian Perrig, Raphael Reischuk,
Wendy Seltzer, Andrew Sullivan, and Suzanne Woolf for the discussions leading to
the design of this protocol, and the definition of an ideal naming service on
which it is based. Thanks especially to Stephen Shirley for detailed feedback.

--- back

