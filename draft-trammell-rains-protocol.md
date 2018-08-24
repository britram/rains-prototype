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
    I-D.trammell-inip-pins:
    RFC0793:
    RFC1918:
    RFC2119:
    RFC2782:
    RFC3629:
    RFC4193:
    RFC4727:
    RFC5246:
    RFC5280:
    RFC7049:
    FIPS-186-3:
      author:
        -
          ins: NIST
      title: Digital Signature Standard FIPS 186-3
      date: June 2009
    RFC8032:

informative:
    RFC1035:
    RFC4291:
    RFC4632:
    RFC5226:
    RFC5905:
    RFC6605:
    RFC6698:
    RFC7231:
    RFC7624:
    RFC7696:
    RFC7858:
    RFC7871:
    I-D.thomson-postel-was-wrong:
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

--- abstract

This document defines an alternate protocol for Internet name resolution,
designed as a prototype to facilitate conversation about the evolution or
replacement of the Domain Name System protocol. It attempts to answer the
question: "how would we design DNS knowing what we do now," on the
background of the properties of an ideal naming service described in 
{{I-D.trammell-inip-pins}}.

--- middle

# Introduction

This document defines an experimental protocol for providing Internet name
resolution services, as a replacement for DNS, called RAINS (RAINS, Another
Internet Naming Service). It is designed as a prototype to facilitate
conversation about the evolution or replacement of the Domain Name System
protocol, and was developed as a name resolution system for the SCION
("Scalability, Control, and Isolation on Next-Generation Networks") future
Internet architecture {{SCION}}. It attempts to answer the
question: "how would we design the DNS knowing what we do now," on the
background of the properties of an ideal naming service described in 
{{I-D.trammell-inip-pins}}.

Its architecture ({{architecture}}) and information model 
({{information-model}}) are largely compatible with the existing 
Domain Name System. However, it does take several radical departures 
from DNS as presently defined and implemented:

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
  delegations of any prefix length, in accordance with CIDR {{RFC4632}} and
  the IPv6 addressing architecture {{RFC4291}}.

Instead of using a custom binary framing as DNS, RAINS uses Concise Binary
Object Representation {{RFC7049}}, partially in an effort to make
implementations easier to verify and less likely to contain potentially
dangerous parser bugs {{PARSER-BUGS}}. Like DNS, CBOR messages can be carried
atop any number of substrate protocols; RAINS is presently defined to use TLS
over persistent TCP connections (see {{protocol-def}}).

# Terminology

The terms MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY, when they appear in
all-capitals, are to be interpreted as defined in {{RFC2119}}.

In addition, the following terms are used in this document as defined:

- Authority: An entity which may make assertions about names in a zone, by
  virtue of holding a secret key which can generate signatures verifiable using
  a public key associated with a delegation to the zone.
- Assertion: A mapping between a name and object(s) of the same type describing
  the name, signed by an authority for the zone containing the subject name. See
  {{assertion}}.
- Subject: The name to which an assertion pertains.
- Object: A type/value pair of information about a name within an assertion.
- Query: An expression of interest in certain types of objects pertaining to a
  subject name in one or more contexts. See {{query}}.
- Context: Additional information about the scope in which an assertion or query
  is valid. See {{context-in-assertions}} and {{context-in-queries}}.
- Shard: A group of assertions common to a zone and valid at a given point in
  time, with common signatures, which must be lexicographically complete for
  purposes of proving nonexistence of an assertion. See {{shards-and-zones}}.
- Zone: A group of all assertions valid at a given point in time, with common
  signatures, for a given level of delegation and context within the namespace.
  See {{shards-and-zones}}.
- Assertion Update Query: An expression of interest about the current validity
  status of an unexpired assertion one already has.
- Non-existence Update Query: An expression of interest about the current
  validity status of an unexpired shard or zone one already has.
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

# Architecture

The RAINS architecture is simple, and resembles the architecture of DNS. A
RAINS Server is an entity that provides transient and/or permanent storage for
assertions about names, and a lookup function that finds assertions for a
given query about a name, either by searching local storage or by delegating
to another RAINS server. RAINS servers can take on any or all of three roles:

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
queries and assertions. RAINS Clients use a subset variant of the RAINS Protocol
(called the RAINS Client Protocol) to interact with RAINS Servers providing
query services on their behalf.

# Information Model

The RAINS Protocol is based on an information model built around two kinds of
information: Assertions and Queries. An Assertion contains some information
about a name or address, and a Query contains a request for information about a
name or address. The information model in this section omits information
elements required by the resolution mechanism itself; these are defined in more
detail in {{cbor}} and {{protocol-def}}.

## Assertion

An Assertion is a signed statement about a mapping from a subject name to one or
several object values of the same type, and consists of the following elements:

- Context: name of the context in which the assertion is valid;
  see {{context-in-assertions}} below.
- Subject: name about which the assertion is made.
- Zone: name of the zone in which the assertion is made. The fully qualified
  name of the subject is made by appending the zone name to the subject name
  with a domain name separator ('.').
- Type: the type of information about the Subject contained in the assertion.
  Each Assertion is about a single type of data.
- Object: the data of the indicated type associated with the Subject
- Signatures: one or more signatures generated by the authority for the
  Assertion. Signatures contain a time interval during which they are considered
  valid. See {{signatures-in-assertions}} below.

The Types supported for each assertion are:

- Delegation: the authority associated with the zone identified by the name
  (roughly equivalent to the DNSSEC DS RRTYPE). The Object contains a public
  key by which the authority can be identified.
- Redirection: The name(s) of one or more RAINS servers providing authority
  service for the authority associated with the zone (roughly equivalent to
  the DNSSEC NS RRTYPE, but not always consulted directly during resolution).
  The Object contains a set of names.
- Address: one or more addresses associated with the name (replaces DNS A and
  AAAA RTYPEs). The Object contains a set of Addresses. An Address is an
  {address-family, value} tuple.
- Service-Info: one or more layer 4 ports and hostnames associated with a
  service name (replaces DNS SRV RRTYPE). The object contains a {hostname,
  port-number, priority} tuple.
- Name: one or more names associated with the name (roughly equivalent to DNS
  CNAME). The Object contains a set of names.
- Certificate: a certificate which must appear at a specified location in the
  certificate chain presented on a connection attempt with the named entity
  (roughly equivalent to DNS TLSA).
- Zone-Nameset: an expression of the set of names allowed within a zone; e.g.
  Unicode scripts or codepages in which names in the zone may be issued. This
  allows a zone to set policy on names in support of the distinguishability
  property in {{I-D.trammell-inip-pins}} that can be checked by RAINS servers at
  runtime. An assertion about a Subject within a Zone whose name is not allowed
  by a valid signed Zone-Nameset expression is taken to be invalid, even if it
  has a valid signature.
- Zone-Registrar: Information about the organization that caused a Subject name
  to exist, for registrant-level names.
- Zone-Registrant: Information about the organization responsible for a Subject
  name, for registrant-level names.
- Infrastructure Key: Information about public keys used for object security
  within the RAINS infrastructure itself. The Object contains a public key by
  which a named RAINS server can be identified.
- External Key: Information about public keys used for additional signatures
  on assertions. The external key is usually discovered outside RAINS, and can
  be verified by comparison with the key stored in a RAINS assertion. The
  Object contains an external public key.
- Subsequent Key: Assertions about delegations are made by a zone's
  superordinate. A zone may request that its superordinate delegate to a new
  public key by publishing a subsequent key assertion (replacing the mechanism
  implemented by CDS/CDNSKEY in DNS).

For a given {subject, type} tuple, multiple assertions can be valid at a given
point in time; the union of the object values of all of these assertions is
considered to be the set of valid values at that point in time.

### Context in Assertions

Assertion contexts are used to determine the validity of the signature by the
declared authority as follows:

- The global context is identified by the special context name '.'. Assertions
  in the global context are signed by the authority for the subject name. For
  example, assertions about the name 'ethz.ch.' in the global context are only
  valid if signed by the relevant authority which is either 'ethz.ch.', 'ch.',
  or '.' depending on the value of the subject name of the assertion.
- A local context is associated with a given authority. The authority-part and
  the context-part of a local context name are divided by a context marker
  ('cx--'). The authority-part directly identifies the authority whose key was
  used to sign the assertion; assertions within a local context are only valid
  if signed by the identified authority. Authorities have complete control over
  how the contexts under their namespaces are arranged, and over the names
  within those contexts. Both the authority-part and the context-part MUST end
  with a '.'.

Assertion context is the mechanism by which RAINS provides explicit
inconsistency (see section 5.3.2 of {{I-D.trammell-inip-pins}}). Some
examples illustrate how context works:

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

### Signatures in Assertions

A signature over an assertion contains the following information elements:

- Algorithm: identifier of the algorithm used to generate the signature.
- Keyspace: identifier of the key space used to generate the signature, i.e. how
  the key to verify the signature should be retrieved. RAINS supports an
  internal keyspace, but allows signatures using externally obtained keys to
  appear on assertions for additional security.
- Keyphase: phase of the key used to generate the signature. Since multiple keys
  may be valid for a given authority at a given point in time, this allows the
  correct key to be retrieved directly.
- Valid-Since: a timestamp of the start of validity of this signature.
- Valid-Until: a timestamp of the end of validity of this signature.
- Signature: the cryptographic signature itself, whose format is determined by
  the algorithm used.

The signature protects all the information in an assertion as well as its own
algorithm identifier, keyspace, keyphase, valid-since, and valid-until values;
it does not protect other signatures on the assertion.

### Shards and Zones

Assertions may also be grouped and signed as a group. A shard is a set of
assertions within the same zone and context, protected by one or more signatures
over all assertions within the shard. Shards have an exclusive lexicographic
range, and contain all assertions for names within a zone within that range.
This lexicographic completeness leads to the property that given a subject and
an authenticated shard, it can be shown that either an assertion with a given
name, type and object value exists within the shard or does not exist at all.

A shard has the following information elements:

- Context: name of the context in which the assertions in the shard are valid;
  see {{context-in-assertions}} above.
- Zone: name of the zone in which the assertions are made.
- Content: a lexicographically sorted set of assertions sharing the context and
  zone.
- Signatures: one or more signatures generated by the authority for the
  shard; see {{signatures-in-assertions}}.

For efficiency's sake, information elements within a shard common to all
assertions (zone, context, signature) within the shard may be omitted from the
assertions themselves.

A zone is the entire set of shards and assertions subject to a given
authority within a given context.

A zone has the following information elements:

- Context: name of the context in which the assertions in the zone are valid;
  see {{context-in-assertions}} above.
- Zone: name of the zone.
- Content: a set of assertions and/or shards sharing the context and zone.
- Signatures: one or more signatures generated by the authority for the
  zone; see {{signatures-in-assertions}}.

### Zone-Reflexive Assertions

A zone may make an assertion about itself by using the string "@" as a subject
name. This facility can be used for any assertion type, but is especially useful
for self-signing root zones, and for a zone to make a subsequent key assertion
about itself. If an assertion of a given type about a zone is available both in
the zone itself and in the superordinate zone, the assertion in the
superordinate zone will take precedence.

## Query

A query is a request for a set of assertions supporting a conclusion about a
given subject-object mapping. It consists of the following information
elements:

- Context: the context(s) in which assertions answering the query will be
  accepted; see {{context-in-queries}} below.
- Qualified-Subject: the name about which the query is made. The subject name
  in a query must be fully-qualified.
- Types: a set of assertion types the querier is interested in.
- Valid-Until: an optional client-generated timestamp for the query after which
  it expires and should not be answered.
- Query Token: a client-generated token for the query, which can be used in the
  answer to refer to the query.
- Options: a set of options by which a client may specify tradeoffs (e.g.
  privacy for performance).

A query expresses interest about all the given types of assertion in all the
specified contexts; more complex expressions of which types in which contexts
must be asked using multiple queries. Preferences for tradeoffs (freshness,
bandwidth efficiency, latency, privacy preservation) in servicing a query may
be bound to the query using query options.

### Context in Queries

Context is used in queries as it is in assertions (see
{{context-in-assertions}}). Assertion contexts in an answer to a query have to
match the context in the query in order to respond to a query. The Context
section of a query contains the context of desired assertions; a special "any"
context (represented by the empty string) indicates that assertions in any
context will be accepted.

Query contexts can also be used to provide additional information to RAINS
servers about the query. For example, context can provide a method for explicit
selection of a CDN server not based on either the client's or the resolver's
address (see {{RFC7871}}). Here, the CDN creates a context for each of its
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

### Answers to Queries

An answer consists of a set of assertions, shards, and/or zones which respond
to a query. If the query contained a token, it is bound to that query via the
token.

The content of an answer depends on whether the answer is positive or negative.
A positive answer contains the information requested in the smallest atomic
container that can be found, usually a single assertion. A negative answer
contains the information used to verify it; either a Shard, an entire Zone, or a
Zone-Nameset assertion showing the name is illegal within the zone.

A query is taken to have an inconclusive answer when no answer returns to the
querier before the query's Valid-Until time.

## Assertion Update Query

An assertion update query is a request for an updated version of a specified
assertion. It consists of the following information elements:

- Qualified-Subject: the fully-qualified name of the assertion.
- Hash function: the hash function used to hash the assertion.
- Hashed Assertion: The hash value obtained by hashing the assertion.
- Valid-Until: an optional client-generated timestamp for the query after which
  it expires and should not be answered.
- Query Token: a client-generated token for the query, which must be used in the
  answer to refer to the query.

### Answers to Assertion Update Queries

An answer consists of an assertion, a shard, a zone, or a notification which
responds to an assertion update query. If the update query contained a token, it
is bound to that query via the token.

The content of an answer depends on whether there is a newer version of the
assertion that is already valid. If the hashed assertion is still the most
recent one, a 200 notification message is returned. In case there is an
assertion for the same name, type and object value with a longer valid signature
it is returned. Otherwise a shard, zone or 210 notification is returned. A shard
or zone is preferred over the notification answer.

An update query is taken to have an inconclusive answer when no answer returns
to the querier before the update query's Valid-Until time.

## Non-existence Update Query

A non-existence update query is a request for an updated version of a previously
non-existent name proven through a shard or zone. It consists of the following
information elements:

- Context: the context(s) in which sections answering the update query will be
  accepted; see {{context-in-queries}} above.
- Qualified-Subject: the fully-qualified name within the range of the shard or
  the zone for which a non-existence proof is requested.
- Type: the assertion type the querier is interested in.
- Hash function: the hash function used to hash the shard or the zone.
- Hashed Section: the hash value obtained by hashing the shard or the zone.
- Valid-Until: an optional client-generated timestamp for the query after which
  it expires and should not be answered.
- Query Token: a client-generated token for the query, which must be used in the
  answer to refer to the query.

### Answers to Non-existence Update Queries

An answer consists of an assertion, a shard, a zone, or a notification which
responds to an update query. If the update query contained a token, it is bound
to that query via the token.

The content of an answer depends on whether there is a new assertion for the
queried context, subject-name and type or a newer version of the hashed shard or
zone which is already valid. If a new assertion exists, it is returned. In case
there is no matching assertion and there is a zone or a shard in the range of
the FQDN with a longer valid signature that is already valid, the section with
the highest validUntil value is returned. Otherwise, the shard or zone is still
the most recent one and a 200 notification message is returned.

An update query is taken to have an inconclusive answer when no answer returns
to the querier before the query's Valid-Until time.

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
{{RFC1918}} {{RFC4193}}. Local context assertions for other addresses are
invalid.

# CBOR Data Model {#cbor}

The RAINS data model is a relatively straightforward mapping of the
information model in {{information-model}} to the Concise Binary Object
Representation (CBOR) {{RFC7049}}, with an outer message type providing a
mechanism for future capabilities-based versioning and recognition of a
message as a RAINS message.

Messages, assertions, shards, zones, queries, and notifications are each
represented as a CBOR map of integer keys to values, which allows each of
these types to be extended in the future, as well as the addition of non-
standard, application-specific information to RAINS messages and data items. A
common registry of map keys is given in {{tabmkey}}. RAINS implementations
MUST ignore map keys they do not understand. Integer map keys in the range -22
to +23 are reserved for the use of future versions or extensions to the RAINS
protocol.

Message contents, signatures and object values are implemented as type-
prefixed CBOR arrays with fixed meanings of each array element; the structure
of these lower-level elements can therefore not be extended. Message section
types are given in {{tabsection}}, object types in {{tabobj}}, and signature
algorithms in {{tabsig}}.

## Symbol Table {#cbor-symtab}

The meaning of each of the integer keys in message, zone, shard, assertion,
and notification maps is given in the symbol table below:

{: #tabmkey title="CBOR Map Keys used in RAINS"}

| Code | Name           | Description                                   |
|-----:|----------------|-----------------------------------------------|
| 0    | signatures     | Signatures on a message or section            |
| 1    | capabilities   | Capabilities of server sending message        |
| 2    | token          | Token for referring to a data item            |
| 3    | subject-name   | Subject name in an assertion, shard or zone   |
| 4    | subject-zone   | Zone name in an assertion, shard or zone      |
| 5    | subject-addr   | Subject address in address assertion          |
| 6    | context        | Context of an assertion, shard, zone or query |
| 7    | objects        | Objects of an assertion                       |
| 8    | query-name     | Fully qualified name for a query              |
| 10   | query-types    | Acceptable object types for query             |
| 11   | shard-range    | Lexical range of Assertions in Shard          |
| 12   | query-expires  | Absolute timestamp for query expiration       |
| 13   | query-opts     | Set of query options requested                |
| 14   | hash-type      | Hash function used in an update query         |
| 15   | hash-value     | Value of a hashed assertion, shard or zone    |
| 16   | nuquery-type   | Object type in non-existence update query     |
| 21   | note-type      | Notification type                             |
| 22   | note-data      | Additional notification data                  |
| 23   | content        | Content of a message, shard, or zone          |

## Message {#cbor-message}

All interactions in RAINS take place in an outer envelope called a Message,
which is a CBOR map tagged with the RAINS Message tag (hex 0xE99BA8, decimal
15309736).

A Message map MAY contain a signatures (0) key, whose value is an array of
Signatures over the entire message as defined in {{cbor-signature}}, to be
verified against the infrastructure key for the RAINS Server originating the
message.

A Message map MAY contain a capabilities (1) key, whose value is described in
{{cbor-capabilities}}.

A Message map MUST contain a token (2) key, whose value is a 16-byte array.
See {{cbor-tokens}} for details.

A Message map MUST contain a content (23) key, whose value is an array of
Message Sections; a Message Section is either an Assertion, Shard, Zone, Query,
or Notification.

## Message Section header

Each Message Section in the Message's content value MUST be a two-element array.
The first element in the array is the message section type, encoded as an
integer as in {{tabsection}}. The second element in the array is a message
section body, a CBOR map defined as in the subsections
{{cbor-assertion}}-{{cbor-notification}}

{: #tabsection title="Message Section Type Codes"}

| Code | Name         | Description                                       |
|-----:|--------------|---------------------------------------------------|
| 1    | assertion    | Assertion (see {{cbor-assertion}})                |
| -1   | revassertion | Address Assertion (see {{cbor-revassert}})        |
| 2    | shard        | Shard (see {{cbor-shard}})                        |
| 3    | zone         | Zone (see {{cbor-zone}})                          |
| 4    | query        | Query (see {{cbor-query}})                        |
| -4   | revquery     | Address Query (see {{cbor-revquery}})             |
| 5    | auquery      | Assertion update query (see {{cbor-auquery}})     |
| 6    | nuquery      | Non-existence update query (see {{cbor-nuquery}}) |
| 23   | notification | Notification (see {{cbor-notification}})          |

## Assertion body {#cbor-assertion}

An Assertion body is a map. The keys present in this map depend on whether the
Assertion is contained in a Message Section or in a Shard or Zone.

Assertions contained in a Message's content value are "bare Assertions". Since
they cannot inherit any values from their containers, they MUST contain the
signatures (0), subject-name (3), subject-zone (4), context (6), and objects (7)
keys.

Assertions within a Shard or Zone are "contained Assertions", and can inherit
values from their containers. A contained Assertion MUST contain the subject-
name (3) and objects (7) keys. The subject-zone (4) and context (6) keys MUST
NOT be present. They are assumed to have the same value as the corresponding
values in the containing Shard or Zone for signature generation and signature
verification purposes; see {{cbor-signature}}.

A contained Assertion SHOULD contain the signatures (0) key, since an unsigned
contained Assertion cannot be used by a RAINS server to answer a query; it
must be returned in a signed Shard or Zone.

The value of the signatures (0) key, if present, is an array of one or more
Signatures as defined in {{cbor-signature}}. If not present, the containing
Shard or Zone MUST be signed. Signatures on a contained Assertion are generated
as if the inherited subject-zone and context values are present in the
Assertion, whether actually present or not. The signatures on the Assertion are
to be verified against the appropriate key for the Zone containing the Assertion
in the given context, as described in {{signatures-in-assertions}}.

The value of the subject-name (3) key is a UTF-8 encoded {{RFC3629}} string
containing the name of the subject of the assertion. The subject name MAY
contain dot(s) '.'. The subject name never contains the zone in which the
subject name is registered; the fully-qualified name is obtained by joining the
subject-name to the subject-zone with a '.' character. The subject-name must be
valid according to the nameset expression for the zone, if any.

The value of the subject-zone (4) key, if present, is a UTF-8 encoded string
containing the name of the zone in which the assertion is made and MUST end with
'.' (the root zone). If not present, the zone of the assertion is inherited from
the containing Shard or Zone.

The value of the context (6) key, if present, is a UTF-8 encoded string
containing the name of the context in which the assertion is valid. If not
present, the context of the assertion is inherited from the containing Shard
or Zone.

The value of the objects (7) key is an array of objects, as defined in 
{{cbor-object}}.

## Shard body {#cbor-shard}

A Shard body is a map. The keys present in the map depend on whether the Shard
is contained in a Message Section or in a Zone.

Shards contained in a Message's content value are "bare Shards". Since they
cannot inherit any values from their contained Zone, they MUST contain the
content (23), signatures (0), subject-zone (4), context (6), and shard-range
(11) keys.

Shards within a Zone are "contained Shards", and can inherit values from their
containing Zone. A contained Shard MUST contain the shard-range(11) and content
(23) keys. The subject-zone (4) and context (6) keys MUST NOT be present. They
are assumed to have the same value as the corresponding values in the containing
Zone for signature generation and signature verification purposes; see
{{cbor-signature}}.

A contained Shard SHOULD contain the signatures (0) key if it also contains a
shard-range (11) key, since an unsigned contained Shard cannot be used by a
RAINS server to answer a query for nonexistence; it must be returned in a
signed Zone.

The value of the content (23) key is an array of Assertion bodies as defined in
{{cbor-assertion}}. Assertions within a Shard MUST be sorted on subject names by
ordering Unicode codepoints in ascending order. If the name of two assertions
are equal, they are sorted by the code number of their type {{cbor-object}} in
ascending order. In case they also have the same type, they are sorted in
ascending order by their object value(s).

The value of the signatures (0) key, if present, is an array of one or more
Signatures as defined in {{cbor-signature}}. If not present, the containing Zone
MUST be signed. Signatures on a contained Shard are generated as if the
inherited subject-zone and context values are present in the Shard, whether
actually present or not. The signatures on the Shard are to be verified against
the appropriate key for the Zone containing the Shard in the given context, as
described in {{signatures-in-assertions}}.

The value of the subject-zone (4) key, if present, is a UTF-8 encoded string
containing the name of the zone in which the Assertions within the Shard is made
and MUST end with '.' (the root zone). If not present, the zone of the assertion
is inherited from the containing Zone.

The value of the context (6) key, if present, is a UTF-8 encoded string
containing the name of the context in which the Assertions within the Shard
are valid. If not present, the context of the assertion is inherited from the
containing Zone.

Shards are lexicographically complete within the range described in the
shard-range value: a subject-name within the shard-range that is not contained
in the shard is asserted to not exist.

The shard-range value MUST be a two element array of strings or nulls
(subject-name A, subject-name B). A must lexicographically sort before B, but
neither subject name need be present in the shard's contents. If A is null, the
shard begins at the beginning of the zone. If B is null, the shard ends at the
end of the zone. The shard MUST NOT contain any assertions whose subject names
sort before A or after B. In addition, the authority the shard belongs to MUST
NOT make any assertions during the period of validity of the shard's signatures
that would fall between subject-name A and subject-name B inclusive that are not
contained within the shard (see {{runtime-consistency-checking}}).

## Zone body {#cbor-zone}

A Zone body is a map. Zones MUST contain the content (23), signatures (0),
subject-zone (4), and context (6) keys.

Signatures on the Zone are to be verified against the appropriate key for the
Zone in the given context, as described in {{signatures-in-assertions}}.

The value of the content (23) key is an array of Shard bodies as defined in
{{cbor-shard}} and/or Assertion bodies as defined in {{cbor-assertion}}. Shards
and Assertions in the content array MUST be sorted. Assertions are sorted before
shards and are sorted as described in the shard's content value {{cbor-shard}}.
Shards are lexicographically sorted by their range where the start of the range
takes precedence over the end of the range. If the range is equal, the contained
assertions determine the ordering, which is again ascending.

The value of the subject-zone (4) key is a UTF-8 encoded string containing the
name of the Zone which MUST end with '.' (the root zone).

The value of the context (6) key is a UTF-8 encoded string
containing the name of the context for which the Zone is valid.

## Query body {#cbor-query}

A Query body is a map. Queries MUST contain the query-name (8),
context (6), query-types (10), and query-expires (12) keys. Queries MAY contain
the query-opts (13) keys.

The value of the query-name (8) key is a UTF-8 encoded string containing the
name for which the query is issued and MUST end with a '.' (the root zone).

The value of the context (6) key is a UTF-8 encoded string containing the name
of the context to which a query pertains. A zero-length string indicates that
assertions will be accepted in any context.

The value of the query-types (10) key is an array of integers encoding the
type(s) of objects (as in {{cbor-object}}) acceptable in answers to the query.
All values in the query-type array are treated at equal priority: [2,3] means
the querier is equally interested in both IPv4 and IPv6 addresses for the
query-name. An empty query-types array indicates that objects of any type are
acceptable in answers to the query.

The value of the query-expires (12) key, is a CBOR integer counting seconds
since the UNIX epoch UTC, identified with tag value 1 and encoded as in section
2.4.1 of {{RFC7049}}. After the query-expires time, the query will have been
considered not answered by the original issuer.

The value of the query-opts (13) key, if present, is an array of integers in
priority order of the querier's preferences in tradeoffs in answering the
query, as in {{tabqopts}}.

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

Options 1-5 specify performance/privacy tradeoffs. Each server is free to
determine how to minimize each performance metric requested; however, servers
MUST NOT generate queries to other servers if "no information leakage" is
specified, and servers MUST NOT return expired assertions unless "expired
assertions acceptable" is specified.

Option 6 specifies that a given token (see {{cbor-tokens}}) should be used on
all queries resulting from a given query, allowing traceability through an
entire RAINS infrastructure. It is meant for debugging purposes. 

By default, a client service will perform verification of negative queries and
return a 404 No Assertion Exists for queries with a consistent proof of non-
existence, within a message signed by the query service's infrakey. Option 7
disables this behavior, and causes the query service to return the shard
proving nonexistence for verification by the client. It is intended to be used
with untrusted query services.

Option 8 specifies that a querier's interest in a query is strictly ephemeral,
and that future assertions related to this query SHOULD NOT be proactively
pushed to the querier.

## Assertion Update Query body {#cbor-auquery}

An Assertion Update Query body is a map. Assertion Update Queries MUST contain
the query-name (8), hash-type (14), hash-value (15), query-expires (12) and
token (2) keys. Assertion Update Queries MAY contain the query-opts (13) keys.

The value of the query-name (8) key is a UTF-8 encoded string containing the
FQDN for which the update query is issued and MUST end with a '.' (the root
zone).

The value of the hash-type (14) key is an integer specifying a hash function
identifier used to generate the hash-value of the assertion, as in
{{tabuqhash}}.

The value of the hash-value (15) key is the hash of the assertion for which an
update is requested. The hash is generated over a byte stream representing the
assertion in a canonical signing format {{cbor-signature}} (The signature itself
is not hashed). The format is defined by the hash-type.

The value of the query-expires (12) key, is a CBOR integer counting seconds
since the UNIX epoch UTC, identified with tag value 1 and encoded as in section
2.4.1 of {{RFC7049}}. After the query-expires time, the update query will have
been considered not answered by the original issuer.

The value of the token (2) key is a 16-byte array which MUST be part of the
response. See {{cbor-tokens}} for details.

The value of the query-opts (13) key, if present, is an array of integers in
priority order of the querier's preferences in tradeoffs in answering the
assertion update query, as in {{tabqopts}}.

{: #tabuqhash title="Update Query Hash Function Codes"}

| Code | Name      | Notes                                  |
|-----:|-----------|----------------------------------------|
| 1    | sha-256   | Value contains SHA-256 hash (32 bytes) |
| 2    | sha-512   | Value contains SHA-512 hash (64 bytes) |
| 3    | sha-384   | Value contains SHA-384 hash (48 bytes) |

## Non-existence Update Query body {#cbor-nuquery}

A Non-existence Update Query body is a map. Non-existence Update Queries MUST
contain the query-name (8), context (6), nuquery-type (16), hash-type (14),
hash-value (15), query-expires (12) and token (2) keys. Non-existence Update
Queries MAY contain the query-opts (13) keys.

The value of the query-name (8) key is a UTF-8 encoded string containing the
FQDN for which the update query is issued and MUST end with a '.' (the root
zone).

The value of the context (6) key is a UTF-8 encoded string containing the name
of the context to which an update query pertains. A zero-length string indicates
that assertions will be accepted in any context.

The value of the nuquery-type (16) key is an integer encoding the type of
objects (as in {{cbor-object}}) acceptable in answers to the update query.

The value of the hash-type (14) key is an integer specifying a hash function
identifier used to generate the hash-value of the assertion, as in
{{tabuqhash}}.

The value of the hash-value (15) key is the hash of the assertion for which an
update is requested. The hash is generated over a byte stream representing the
assertion in a canonical signing format {{cbor-signature}} (The signature itself
is not hashed). The format is defined by the hash-type.

The value of the query-expires (12) key, is a CBOR integer counting seconds
since the UNIX epoch UTC, identified with tag value 1 and encoded as in section
2.4.1 of {{RFC7049}}. After the query-expires time, the update query will have
been considered not answered by the original issuer.

The value of the token (2) key is a 16-byte array which MUST be part of the
response. See {{cbor-tokens}} for details.

The value of the query-opts (13) key, if present, is an array of integers in
priority order of the querier's preferences in tradeoffs in answering the
non-existence update query, as in {{tabqopts}}.

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
All values in the query-type array are treated at equal priority: [4,5] means
the querier is equally interested in both redirection and delegation for the
subject-addr. An empty query-types array indicates that objects of any type are
acceptable in answers to the query.

The value of the query-expires (12) key is a CBOR integer
counting seconds since the UNIX epoch UTC, identified with tag value 1 and
encoded as in section 2.4.1 of {{RFC7049}}. After the query-expires time, the
query will have been considered not answered by the original issuer.

The value of the query-opts (13) key, if present, is an array of integers in
priority order of the querier's preferences in tradeoffs in answering the
query, as in {{tabqopts}}. See {{cbor-query}} for more.

An Address Assertion with a more-specific prefix is preferred over a
less-specific in response to a Address Query.

## Notification body {#cbor-notification}

Notification Message Sections contain information about the operation of the
RAINS protocol itself. A Notification Message Section body is a map which MUST
contain the token (2) and note-type (21) keys and MAY contain the note-data
(22) key. The value of the note-type key is encoded as an integer as in the
{{tabnotify}}.

{: #tabnotify title="Notification Type Codes"}

| Code | Description                                                    |
|-----:|----------------------------------------------------------------|
| 100  | Connection heartbeat                                           |
| 200  | The hashed section in an update query is still fine            |
| 210  | The hashed assertion has been revoked and is no longer valid   |
| 399  | Capability hash not understood                                 |
| 400  | Bad message received                                           |
| 403  | Inconsistent message received                                  |
| 404  | No assertion exists (client protocol only)                     |
| 413  | Message too large                                              |
| 500  | Unspecified server error                                       |
| 501  | Server not capable                                             |
| 504  | No assertion available                                         |

Note that the status codes are chosen to be mnemonically similar to status
codes for HTTP {{RFC7231}}. Details of the meaning of each status code are
given in {{protocol-def}}.

The value of the token (2) key is a 16-byte array, which
MUST contain the token of the message or query to which the notification is a
response. See {{cbor-tokens}}.

The value of the note-data (22) key, if present, is a UTF-8 encoded string
with additional information about the notification, intended to be displayed
to an administrator to help debug the issue identified by the negotiation.

## Object {#cbor-object}

Objects are encoded as arrays in CBOR, where the first element is the type of
the object, encoded as an integer in the following table:

{: #tabobj title="Object type codes"}

| Code  | Name         | Description                             |
|------:|--------------|-----------------------------------------|
| 1     | name         | name associated with subject            |
| 2     | ip6-addr     | IPv6 address of subject                 |
| 3     | ip4-addr     | IPv4 address of subject                 |
| 4     | redirection  | name of zone authority server           |
| 5     | delegation   | public key for zone delgation           |
| 6     | nameset      | name set expression for zone            |
| 7     | cert-info    | certificate information for name        |
| 8     | service-info | service information for srvname         |
| 9     | registrar    | registrar information                   |
| 10    | registrant   | registrant information                  |
| 11    | infrakey     | public key for RAINS infrastructure     |
| 12    | extrakey     | external public key for subject         |
| 13    | nextkey      | next public key for subject             |

A name (1) object contains a name associated with a name as an alias. It is
represented as a three-element array. The second element is a fully-qualified
name as a UTF-8 encoded string. The third type is an array of object type
codes for which the alias is valid, with the same semantics as the query-types
(9) key in queries (see {{cbor-query}}).

An ip6-addr (2) object contains an IPv6 address associated with a name. It is
represented as a two element array. The second element is a byte array of
length 16 containing an IPv6 address in network byte order.

An ip4-addr (3) object contains an IPv4 address associated with a name. It is
represented as a two element array. The second element is a byte array of
length 4 containing an IPv4 address in network byte order.

A redirection (4) object contains the fully-qualified name of a RAINS
authority server for a named zone. It is represented as a two-element array.
The second element is a fully-qualified name of an RAINS authority server as a
UTF-8 encoded string.

A delegation (5) object contains a public key used to generate signatures on
assertions in a named zone, and by which a delegation of a name within a zone to
a subordinate zone may be verified. It is represented as an N-element array. The
second element is a signature algorithm identifier as in {{cbor-signature}}.
Additional elements are as defined in {{cbor-signature}} for the given algorithm
identifier and keyspace.

A nameset (6) object contains an expression defining which names are allowed
and which names are disallowed in a given zone. It is represented as a two-
element array. The second element is a nameset expression to be applied to
each name element within the zone without an intervening delegation, as
defined in {{cbor-nameset}}

A cert-info (7) object contains an expression binding a certificate or
certificate authority to a name, such that connections to the name must either
use the bound certificate or a certificate signed by a bound authority. It is
represented as an five-element array, as defined in {{cbor-certinfo}}.

A service-info (8) object gives information about a named service. Services
are named as in {{RFC2782}}. It is represented as a four-element array. The
second element is a fully-qualified name of a host providing the named service
as a UTF-8 string. The third element is a transport port number as a positive
integer in the range 0-65535. The fourth element is a priority as a positive
integer, with lower numbers having higher priority.

A registrar (9) object gives the name and other identifying information of the
registrar (the organization which caused the name to be added to the namespace)
for organization-level names. It is represented as a two element array. The
second element is a UTF-8 string of maximum length 256 bytes containing
identifying information chosen by the registrar according to the registry's
policy.

A registrant (10) object gives information about the registrant of an
organization-level name. It is represented as a two element array. The second
element is a UTF-8 string with a maximum length of 4096 bytes containing this
information, with a format chosen by the registrar according to the registry's
policy.

An infrakey (11) object contains a public key used to generate signatures on
messages by a named RAINS server, by which a RAINS message signature may be
verified by a receiver. It is identical in structure to a delegation object,
as defined in {{cbor-signature}}. Infrakey signatures are especially useful
for clients which delegate verification to their query servers to authenticate
the messages sent by the query server.

An extrakey (12) object contains a public key used to generate signatures on
assertions in a named zone outside of the normal delegation chain. It is
represented as an 4-element array, where the second element is a signature
algorithm identifier, and the third element is keyspace identifier, as in
{{cbor-signature}}. The fourth element is the public key, as defined in
{{cbor-signature}} for the given algorithm identifier. An extrakey may be
matched with a public key obtained through other means for additional
authentication of an assertion. Extrakeys are different from delegation keys in
that they may not be used in the delegation chain: an extrakey signature is
valid only on assertions of object types other than delegation.

A nextkey (13) object contains the a public key that a zone owner would like its
superordinate to delegate to in the future. It is represented as an 5-element
array The second element is a signature algorithm identifier as in
{{cbor-signature}}. The third element is the public key, as defined in
{{cbor-signature}} for the given algorithm identifier. The fourth element is the
requested-valid-since time, and the fifth element is the requested-valid-until
time, formatted as for signatures as in {{cbor-signature}}. See
{{public-key-management}} for more.

### Certificate information format {#cbor-certinfo}

A cert-info object contains information about the certificate(s) that can be
used to authenticate a transport-layer association with a named entity. It is
encoded as a file-element array. The first element is the RAINS object type
(7). The second element is the protocol family specifier, describing the
cryptographic protocol used to connect, as defined in {{tabcertproto}}. The
protocol family defines the format of certificate data to be hashed. The third
element is the certificate usage specifier as in {{tabcertusage}}, describing
the constraint imposed by the assertion. These are defined to be compatible
with Certificate Usages in the TLSA RRTYPE for DANE {{RFC6698}}. The fourth
element is the hash algorithm identifier, defining the hash algorithm used to
generate the certificate data. The fifth item is the data itself, whose format
is defined by the protocol family and hash algorithm.

{: #tabcertproto title="Certificate information protocol families"}

| Code | Name     | Protocol family                            | Certificate format |
|-----:|----------|--------------------------------------------|--------------------|
|    0 | unspec   | Unspecified                                | Unspecified        |
|    1 | tls      | Transport Layer Security (TLS) {{RFC5246}} | {{RFC5280}}        |

Protocol family 0 leaves the protocol family unspecified; client validation
and usage of cert-info assertions, and the protocol used to connect, are up to
the client, and no information is stored in RAINS. Protocol family 1 specifies
Transport Layer Security version 1.2 {{RFC5246}} or a subsequent version,
secured with PKIX {{RFC5280}} certificates.

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

{: #tabcerthash title="Certificate information hash algorithms"}

| Code | Name      | Notes                                 |
|-----:|-----------|---------------------------------------|
| 0    | full      | Data contains full certificate        |
| 1    | sha-256   | Data contains SHA-256 hash (32 bytes) |
| 2    | sha-512   | Data contains SHA-512 hash (64 bytes) |
| 3    | sha-384   | Data contains SHA-384 hash (48 bytes) |

Code 0 is used to store full certificates in RAINS assertions, while other
codes are used to store hashes for verification.

For example, in a cert-info object with values [ 7, 1, 3, 3, (data) ], the
data would be a 48 SHA-384 hash of the ASN.1 DER-encoded X.509v3 certificate
(see Section 4.1 of {{RFC5280}}) to be presented by the endpoint on a
connection attempt with TLS version 1.2 or later.

### Name expression format {#cbor-nameset}

The nameset expression is represented as a UTF-8 string encoding a modified POSIX
Extended Regular Expression format (see POSIX.2) to be applied to each element
of a name within the zone. A name containing an element that does not match
the valid nameset expression for a zone is not valid within the zone, and the
nameset assertion can be used to prove nonexistence.

The POSIX character classes :alnum:, :alpha:, :ascii:, :digit:, :lower:, and :upper: are available in these regular expressions, where:

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

Unicode escapes are supported in these regular expressions; the sequence
\uXXXX where XXXX is a 4 or 5 digit, possibly zero-prefixed hex encoding of
the codepoint, is substituted with that codepoint.

Set operations (intersection and subtraction) are available on character
classes. Two character class or range expressions in a bracket expression
joined by the sequence && are equivalent to the intersection of the two
character classes or ranges. Two character class or range expressions in a
bracket expression joined by the sequence -- are equivalent to the subtraction
of the second character class or range from the first.

For example, the nameset expression:

[[:ublk0400:]&&[:lower:][:digit:]]+

matches any name made up of one or more lowercase Cyrillic letters and digits. The same expression can be implemented with a range instead of a character class:

[\u0400-\u04ff&&[:lower:][:digit:]]+

## Tokens in queries and messages {#cbor-tokens}

Messages and notifications contain an opaque token (2) key, whose
content is a 16-byte array, and is used to link Messages to the Queries they
respond to, and Notifications to the Messages they respond to. Tokens MUST be
treated as opaque values by RAINS servers.

A Message sent in response to a Query MUST contain the token of the Message
containing the Query. Otherwise, the Message MUST contain a token selected by
the server originating it, so that future Notifications can be linked to the
Message causing it. Likewise, a Notification sent in response to a Message MUST
contain the token from the Message causing it (where the new Message contains a
fresh token selected by the server). This allows sending multiple Notifications
within one Message and the receiving server to respond to a Message containing
Notifications (e.g. when it is malformed).

Since tokens are used to link queries to replies, and to link notifications to
messages, regardless of the sender or recipient of a message, they MUST be
chosen by servers to be hard to guess; e.g. generated by a cryptographic random
number generator.

When a server creates a new query to forward to another server in response to
a query it received, it MUST NOT use the same token on the delegated query
as on the received query, unless option 6 Enable Tracing is present in the
received, in which case it MUST use the same token.

## Signatures, delegation keys, and RAINS infrastructure keys {#cbor-signature}

RAINS supports multiple signature algorithms and hash functions for signing
assertions for cryptographic algorithm agility {{RFC7696}}. A RAINS signature
algorithm identifier specifies the signature algorithm; a hash function for
generating the HMAC and the format of the encodings of the signature
values in Assertions, Shards, Zones, and Messages, as well as of public key
values in delegation objects.

RAINS signatures have five common elements: the algorithm identifier, a keyspace
identifier, a keyphase identifier, a valid-since timestamp, and a valid-until
timestamp. Signatures are represented as an array of these five values followed
by additional elements containing the signature data itself, according to the
algorithm identifier.

The following algorithms are supported:

{: #tabsig title="Defined signature algorithms"}

| Alg ID | Signatures | Hash/HMAC | Format               |
|-------:|------------|-----------|----------------------|
| 1      | ed25519    | sha-512   | See {{eddsa-format}} |
| 2      | ed448      | shake256  | See {{eddsa-format}} |
| 3      | ecdsa-256  | sha-256   | See {{ecdsa-format}} |
| 4      | ecdsa-384  | sha-384   | See {{ecdsa-format}} |

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
encoded as in section 2.4.1 of {{RFC7049}}. A signature MUST have a
valid-until timestamp. If a signature has no specified valid-since time (i.e.,
is valid from the beginning of time until its valid-until timestamp), the
valid-since time MAY be null (as in Table 2 in Section 2.3 of {{RFC7049}}).

A signature in RAINS is generated over a byte stream representing the message in
a canonical signing format. The signing process is defined as follows:

- Parse the object to be signed into a byte stream according to the format
specified in {{signing-format}}.

- Generate a signature on the resulting byte stream according to the algorithm
  selected.

- Add the full signature to the signatures array at the appropriate point in
  the object.

To verify a signature, generate the byte stream as for signing, then verify
the signature according to the algorithm selected.

### EdDSA signature and public key format {#eddsa-format}

EdDSA public keys consist of a single value, a 32-byte bit string generated as
in Section 5.1.5 of {{RFC8032}} for Ed25519, and a 57-byte bit string generated
as in Section 5.2.5 of {{RFC8032}} for Ed448. The fourth element in a RAINS
delegation object is this bit string encoded as a CBOR byte array. RAINS
delegation objects for Ed25519 keys with value k are therefore represented by
the array [5, 1, phase, k]; and for Ed448 keys as [5, 2, phase, k].

Ed25519 and Ed448 signatures are are a combination of two non-negative integers,
called "R" and "S" in sections 5.1.6 and 5.2.6, respectively, of {{RFC8032}}. An
Ed25519 signature is represented as a 64-byte array containing the concatenation
of R and S, and an Ed448 signature is represented as a 114-byte array containing
the concatenation of R and S. RAINS signatures using Ed25519 are therefore the
array [1, 0, phase, valid-since, valid-until, R|S]; using Ed448 the array [2, 0,
phase, valid-since, valid-until, R|S].

Ed25519 keys are generated as in Section 5.1.5 of {{RFC8032}}, and Ed448 keys
as in Section 5.2.5 of {{RFC8032}}.  Ed25519 signatures are generated from a
normalized serialized CBOR object as in Section 5.1.6 of {{RFC8032}}, and
Ed448 signatures as in section 5.2.6 of {{RFC8032}}.

RAINS Server and Client implementations MUST support Ed25519 signatures for
delegation.

### ECDSA signature and public key format {#ecdsa-format}

ECDSA public keys consist of a single value, called "Q" in {{FIPS-186-3}}. Q
is a simple bit string that represents the uncompressed form of a curve point,
concatenated together as "x | y". The fourth element in a RAINS delegation
object is the Q bit string encoded as a CBOR byte array. RAINS delegation
objects for ECDSA-256 public keys are therefore represented as the array 
[5, 3, phase, Q]; and for ECDSA-384 public keys as [5, 4, phase, Q].

ECDSA signatures are a combination of two non-negative integers, called "r" and
"s" in {{FIPS-186-3}}. A Signature using ECDSA is represented using a
four-element CBOR array, with the fourth element being "r | s" such that r is
represented as a byte array as described in Section C.2 of {{FIPS-186-3}}, and s
represented as a byte array as described in Section C.2 of {{FIPS-186-3}}. For
ECDSA-256 signatures, each integer MUST be represented as a 32-byte array. For
ECDSA-384 signatures, each integer MUST be represented as a 48-byte array. RAINS
signatures using ECDSA-256 are therefore the array [3, 0, phase, valid-since,
valid-until, r|s]; and for ECDSA-384 the array [4, 0, phase, valid-since,
valid-until, r|s].

ECDSA-256 signatures and public keys use the P-256 curve as defined in {{FIPS-186-3}}.
ECDSA-384 signatures and public keys use the P-384 curve as defined in {{FIPS-186-3}}.

ECDSA-256 and ECDSA-384 support are primarily meant for compatibility with and
migration from existing DNSSEC deployments; see {{dns-transition}}.

## Capabilities {#cbor-capabilities}

When a RAINS server or client sends the first message in a stream to a peer, it
MUST expose its configured capabilities to its peer using the capabilities (1)
key. This key contains either:

- an array of uniform resource names specifying capabilities supported by the
  sending server, taken from the table below, with each name encoded as a
  UTF-8 string.
- a SHA-256 hash of the CBOR byte stream derived from normalizing such an
  array by sorting it in lexicographically increasing order, then serializing
  it.

This mechanism is inspired by {{XEP0115}}, and is intended to be used to
reduce the overhead in exposing common sets of capabilities. Each RAINS server
can cache a set of recently-seen or common hashes, and only request the full
URN set (using notification code 399) on a cache miss.

The following URNs are presently defined; other URNs will specify future
optional features, support for alternate transport protocols and new signature
algorithms, etc.

| URN                | Meaning                                                           |
|--------------------|-------------------------------------------------------------------|
| urn:x-rains:tlssrv | Listens for connections on TLS over TCP from other RAINS servers. |

Since there are only two defined capabilities at this time, RAINS servers can be
implemented with two hard-coded hashes to determine whether a peer is listening
or not. The hash presented by a server supporting urn:x-rains:tlssrv is
e5365a09be554ae55b855f15264dbc837b04f5831daeb321359e18cdabab5745; the hash
presented by a client or a server supporting no capabilities (not listening) is
76be8b528d0075f7aae98d6fa57a6d3c83ae480a8469e668d7b0af968995ac71.

Servers MAY piggyback capability negotiation on other messages, or use dedicated
messages for capability negotiation.

A RAINS server MUST NOT assume that a peer server supports a given capability
unless it has received a message containing that capability from that server.
An exception are the capabilities indicating that a server listens for
connections using a given transport protocol; servers and clients can also
learn this information from RAINS itself (given a redirection assertion for a
named zone) or from external configuration values.

# Canonical signing format {#signing-format}

\[EDITOR'S NOTE: to define, based on CBOR canonicalization, once this is implemented.]

# RAINS Protocol Definition {#protocol-def}

As noted in {{cbor}}, RAINS is a message-exchange protocol that uses CBOR
{{RFC7049}} as its framing. Since CBOR is self-framing -- a CBOR parser can
determine when a CBOR object is complete at the point at which it has read its
final byte -- RAINS requires no external framing. It can therefore run over
any streaming, multistreaming, or message-oriented transport protocol. In
order to protect query confidentiality, and support rapid deployment over a
ubiquitously implemented transport, RAINS is defined in this document to run
over persistent TLS 1.2 connections {{RFC5246}} over TCP {{RFC0793}} with
mutual authentication between servers, and authentication of servers by
clients. The TLS certificates of RAINS server peers can be verified as
specified in the cert-info assertions for those servers.

RAINS servers MUST support this transport; future transports can be negotiated
using the capabilities mechanism after bootstrapping using TLS 1.2. As RAINS
is an experimental protocol, RAINS servers listen on port 1022 {{RFC4727}} for
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

On receipt of an assertion, shard, or zone message section, a server:

- verifies its consistency (see {{runtime-consistency-checking}}). If the
  section is not consistent, it prepares to send a notification of type 403
  Inconsistent Message to the peer, and discards the section. Otherwise, it:
- determines whether it answers an outstanding query; if so, it prepares to
  forward the section to the server that issued the query.
- determines whether it is likely to answer a future query, according to its 
  configuration, policy, and query history; if so, it caches the section.

On receipt of an assertion, shard, or zone message section, a client:

- determines whether it answers an outstanding query; if so, it considers the
  query answered. It then:
- determines whether it is likely to answer a future query, according to its 
  configuration, policy, and query history; if so, it caches the section.

On receipt of a query, a server:

- determines whether it has expired by checking the query-expires value. If so,
  it drops the query silently. If not, it:
- determines whether it has a stored assertion, shard, and/or zone message
  section which answers the query. If so, it prepares to return the most
  specific such section (i.e., if it has both a shard and an assertion that
  would answer the query, it returns the assertion) with the signature of the
  longest remaining validity to the peer that issued the query. If not, it:
- checks to see whether the query specifies option 4 (cached answers only). If
  so, and if option 5 (expired assertions acceptable) is also specified, it then
  checks to see if it has any cached sections that answer the query on which
  signatures are expired; otherwise, processing stops, and the server returns a
  504 No Assertion Available notification, as if the query had instantly
  expired. If the query does not specify option 4, delegation proceeds, and the
  server:
- determines whether it has other non-authoritative servers it can forward the
  query to, according to its configuration and policy, and in compliance with
  any query options (see {{cbor-query}}). If so, it prepares to forward the
  query to those servers, noting the reply for the received query depends on
  the replies for the forwarded query. If not, it:
- determines the responsible authority servers for the zone containing the
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
  it checks if the hashed assertion is still the assertion currently valid with
  the highest validUntil time for the given name, context, type and object
  value. In that case it returns a notfication message containing the query's
  token and 'ok' as the content. 

If not, it: it has a stored assertion, shard, and/or zone message
  section which answers the query. If so, it prepares to return the most
  specific such section (i.e., if it has both a shard and an assertion that
  would answer the query, it returns the assertion) with the signature of the
  longest remaining validity to the peer that issued the query. If not, it:
- checks to see whether the query specifies option 4 (cached answers only). If
  so, and if option 5 (expired assertions acceptable) is also specified, it then
  checks to see if it has any cached sections that answer the query on which
  signatures are expired; otherwise, processing stops, and the server returns a
  504 No Assertion Available notification, as if the query had instantly
  expired. If the query does not specify option 4, delegation proceeds, and the
  server:

If the hashed assertion is still the most
recent one, a notification with 'ok' is returned. In case there is an assertion
for the same name, type and object value with a longer valid signature it is
returned. Otherwise a shard, zone or notification with content 'non-existent' is
returned. A shard or zone is preferred over the notification answer.

- determines whether it has other non-authoritative servers it can forward the
  query to, according to its configuration and policy, and in compliance with
  any query options (see {{cbor-query}}). If so, it prepares to forward the
  query to those servers, noting the reply for the received query depends on
  the replies for the forwarded query. If not, it:
- determines the responsible authority servers for the zone containing the
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

In general, servers should follow the principles laid out in Sections 4.1 and
4.2 of {{I-D.thomson-postel-was-wrong}}. A malformed message section, or a
message section with any invalid (but not expired) signature, should be dropped
and log. A malformed message section or invalid signature should not, however,
result in other sections in the same message being dropped, except as explicitly
noted above.

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

Since a bare assertion with a single Ed25519 signature requires on the order of
180 bytes, it is clear that many full zones won't fit into a single minimum
maximum-size message. Authorities are therefore encouraged to publish zones
grouped into shards that will fit into 65536-byte messages, to allow servers
to reply using these shards when full-zone transfers are not possible due to
message size limitations.

## Runtime Consistency Checking

The data model used by the RAINS protocol allows inconsistent information to
be asserted, all resulting from misconfigured or misbehaving authority
servers. The following types of inconsistency are possible:

- A shard omits an assertion within its shard-range which is valid at the same
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
non-existence of an assertion is not designed to provide resistance against
zone enumeration.

On the other hand, confidentiality protection of query information is crucial.
Linking naming queries to a specific user can be nearly as useful to build a
profile of that user for surveillance purposes as full access to the clear
text of that client's communications {{RFC7624}}. In this revision, RAINS uses
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
{{RFC5905}}.

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
services for DNS clients. DNS over TLS {{RFC7858}} SHOULD be used between the
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
registered with IANA via Expert Review {{RFC5226}}.

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
{{cbor-certinfo}}, and {{secret-key-management}} for security-relevant 
details.

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
Andrew Sullivan, and Suzanne Woolf for the discussions leading to the design of
this protocol. Thanks especially to Stephen Shirley for detailed feedback, and
to Christian Fehlmann for extensive implementation experience which has informed
the further development of the protocol.

--- back

# Directions for future experimentation

The following features were suggested during the design of RAINS, but have
been left out of the current revision of the specification to allow additional
experimentation with them before they are completely specified.

## Revocation based on hash chains {#hash-chain-rev}

RAINS assertions are scoped in temporal validity by the lifetimes on their
signatures. This is operationally equivalent to TTL in the current DNS. An
assertion which becomes invalid can simply not be renewed by its authority.
However, very dynamic infrastructures may require impractical numbers of
signatures, and could benefit from longer validity times. Allowing an assertion
to be revoked would make this possible. This entails adding a field to every
signature:

- Revocation-Token: an optional revocation token for this signature, which 
  allows a signature to be replaced or removed before the end of its validity.
  Revocation tokens are generally based on hash chains, meaning that a
  signature with a revocation token "down" the chain from a given token
  supercedes it. The format and mechanism used by the revocation token is
  determined by the alogrithm used. 

Hash-chain based revocation allows a signature (and the Assertion, Shard, or
Zone it protects) to be replaced before it expires. To use hash-chain based
revocation, a signing entity generates a hash chain from a known seed using
the hash function specified by the signature algorithm in use, and places the
Nth value derived therefrom in the hash chain revocation token on a signature.
When used, this token appears as a byte array after the signature data in the
signature array.

A revocation can be issued by generating a new section and signing it,
revealing the N-1st value from the hash chain in the revocation token. To
allow a recipient of a revoked section to verify the revocation, the following
restrictions on what can replace what apply:

- An Assertion can only be replaced by another Assertion with the same 
  Subject within the same Context and Zone, containing an Objects array 
  of the same length containing the same types of Objects. To delete Object 
  values, those values can be replaced with Null in the replacing Assertion.
- A Shard can only be replaced by another Shard with an identical shard-range
  key, within the same Context and Zone.
- A Zone can only be replaced by another Zone with an identical name within 
  the same Context.

Four codepoints have been reserved to support experimentation with this
mechanism, as shown in {{tabsigrev}}. 

{: #tabsigrev title="Defined signature algorithms"}

| Code | Signatures | Hash/HMAC | Format               | Revocation |
|-----:|------------|-----------|----------------------|------------|
| 24   | ed25519    | sha-512   | See {{eddsa-format}} | hash-chain |
| 25   | ed448      | shake256  | See {{eddsa-format}} | hash-chain |
| 26   | ecdsa-256  | sha-256   | See {{ecdsa-format}} | hash-chain |
| 27   | ecdsa-384  | sha-384   | See {{ecdsa-format}} | hash-chain |

The main open question for experimentation is how to ensure that a revocation
is properly propagated through a RAINS infrastructure; this may require
protocol changes to work reliably.

To support this experiment, a server must additionally evaluate an assertion
it receives to determine whether it replaces any information presently in its
cache. If so, it discards the old information, and caches the new section.

# Open Questions and Issues

- A method for clients to discover local oracles needs to be specified.
- Consider making negative answers less expensive by allowing a hash of a
  shard with a negative answer proof to be sent back, and checked with a "no
  hashed negative answers" query option. This would increase complexity
  somewhat, because it would require the (re-)addition of an Answer section,
  which could contain such a beast.
- Consider adding semantics to note-data for automated reaction to an error.
  Specifically, notification codes 400, 403, and 413 could use additional data.
- Do we really need 504 No Assertion Available? We know (at the client) when the
  query timed out.
- Revocation is hard; it might be made more tractable (and allow some
  operational control) by designing a revocation mechanism that can *only*
  revoke delegations)
  