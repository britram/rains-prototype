---
title: RAINS (Another Internet Naming Service) Protocol Specification
abbrev: RAINS
docname: draft-trammell-rains-protocol
date: 
category: exp

ipr: trust200902
area: Internet Architecture Board
workgroup: Names and Identifiers Program
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: B. Trammell
    name: Brian Trammell
    organization: ETH Zurich NetSec
    street: Universitaetstrasse 6
    city: Zurich
    code: 8092
    country: Switzerland
    email: ietf@trammell.ch

normative:
    I-D.trammell-inip-pins:
    RFC0793:
    RFC2119:
    RFC2782:
    RFC3629:
    RFC4727:
    RFC5246:
    RFC7049:
    FIPS-186-3:
      author:
        -
          ins: NIST
      title: Digital Signature Standard FIPS 186-3
      date: June 2009

informative:
    RFC1035:
    RFC5226:
    RFC5905:
    RFC6605:
    RFC7231:
    RFC7624:
    RFC7696:
    RFC7858:
    RFC7871:
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
question: "how would we design the DNS knowing what we do now," on the
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

Its architecture ({{architecture}}) and information model ({{information-
model}}) are largely compatible with the existing Domain Name System. However,
it does take several radical departures from DNS as presently defined and
implemented:

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
  determining which assertions answer which queries; see {{context-in-
  queries}}.
- There is an explicit separation between registrant-level names and sub-
  registrant-level names, and explicit information about registrars and
  registrants available in the naming system at runtime.
- Sets of valid characters and rules for valid names are defined on a per-zone
  basis, and can be verified at runtime.

Instead of using a custom binary framing as DNS, RAINS uses Concise Binary
Object Representation {{RFC7049}}, partially in an effort to make
implementations easier to verify and less likely to contain potentially
dangerous parser bugs {{PARSER-BUGS}}. Like DNS, CBOR messages can be carried
atop any number of substrate protocols; RAINS is presently defined to use TLS
over persistent TCP connections (see {{protocol-def}}).

# Terminology

The terms MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY, when they appear in
all-capitals, are to be interpreted as defined in {{RFC2119}}.

In addition, the following terms are used in this document as defined [EDITOR'S NOTE: define and normalize definitions/capitalizations, move definitions here]

- Authority: see {{assertion}} [EDITOR'S NOTE: not really, needs def]
- Assertion: see {{assertion}}
- Context: see {{context-in-assertions}} and {{context-in-queries}}
- Shard: see {{shards-and-zones}}
- Zone: see {{shards-and-zones}}
- Query: see {{query}}
- Notification: see {{cbor-notification}}
- RAINS Message: see {{cbor-message}}
- Authority Service: see {{architecture}}
- Query Service: see {{architecture}}
- Intermediary Service: see {{architecture}}
- RAINS Server: see {{architecture}}
- RAINS Client: see {{architecture}} and {{protocol-client}}

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

Messages in the RAINS Protocol are made up of two kinds of elements: Assertion
and Query. A third type of element, Answer, binds a Query to a set of
Assertions in response to a Query.

The information model in this section omits information elements required by
the resolution mechanism itself; these are defined in more detail in 
{{cbor}} and {{protocol-def}}.

## Assertion

An Assertion is a signed statement about a mapping from a subject name to an
object value, and consists of the following elements:

- Context: name of the context in which the assertion is valid;
  see {{context-in-assertions}} below.
- Subject: name about which the assertion is made. 
- Zone: name of the zone in which the assertion is made. The fully qualified
  name of the subject is made by appending the zone name to the subject name
  with a domain name separator ('.').
- Type: the type of information about the Subject contained in the 
  assertion. Each Assertion is about a single type of data.
- Object: the data of the indicated type associated with the Subject
- Signatures: one or more signatures generated by the authority for the
  Assertion. Signatures contain a time interval during which they are considered
  valid, and may contain a revocation token allowing them to be revoked before
  the end of the time interval. See {{signatures-in-assertions}} below.

The Types supported for each assertion are:

- Delegation: the authority associated with the zone identified by the name
  (roughly equivalent to the DNSSEC DS RRTYPE). The Object contains a public
  key by which the authority can be identified. 
- Redirection: The name(s) of one or more a RAINS servers providing authority
  service for the authority associated with the zone (roughly equivalent to
  the DNSSEC NS RRTYPE, but not always consulted directly during resolution).
  The Object contains a set of names.
- Address: one or more addresses associated with the name (replaces DNS A and
  AAAA RTYPEs). The Object contains a set of Addresses. An Address is an 
  {address-family, value} tuple.
- Service-Info: one or more layer 4 ports and hostnames associated with a
  service name (replaces DNS SRV RRTYPE). The object contains a {hostname,
  port-number, priority tuple}.
- Name: one or more names associated with the name (roughly equivalent to DNS
  CNAME). The Object contains a set of names.
- Certificate: a certificate which must appear at a specified location in the
  certificate chain presented on a connection attempt with the named entity
  (roughly equivalent to DNS TLSA). The details of this type will be described 
  in a separate document.
- Zone-Nameset: an expression of the set of names allowed within a zone; e.g.
  Unicode scripts or codepages in which names in the zone may be issued. This
  allows a zone to set policy on names in support of the distinguishability
  property in {{I-D.trammell-inip-pins}} that can be checked by RAINS 
  servers at runtime. An assertion about a Subject within a Zone whose
  name is not allowed by a valid signed Zone-Nameset expression is taken to be
  invalid, even if it has a valid signature. The details of this type will be
  described in a separate document.
- Registrar: Information about the organization that caused a Subject name 
  to exist, for registrant-level names.
- Registrant: Information about the organization responsible for a Subject name, 
  for registrant-level names.

For a given {subject, type} tuple, multiple assertions can be valid at a given
point in time; the union of the object values of all of these assertions is
considered to be the set of valid values at that point in time.

### Context in Assertions

Assertion contexts are used to determine the validity of the signature by the
declared authority as follows:

- The global context is identified by the special context name `.'. Assertions
  in the global context are signed by the authority for the subject name. For
  example, assertions about the name simplon.inf.ethz.ch in the global context
  are only valid if signed by the relevant authority inf.ethz.ch.
- A local context is associated with a given authority. The authority-part and
  the context-part of a local context name are divided by a context marker 
  ('cx--'). The authority-part directly identifies the authority whose key was
  used to sign the assertion; assertions within a local context are only valid
  if signed by the identified authority. Authorities have complete control
  over how the contexts under their namespaces are arranged, and over the names 
  within those contexts.

Assertion context is the mechanism by which RAINS provides explicit
inconsistency (see section 5.3.2 of {{I-D.trammell-inip-pins}}). Some
examples illustrate how context works:

- For the common split-DNS case, an enterprise could place names for machines
  on its local networks within a separate context. E.g., a workstation could
  be named simplon.cab.inf.ethz.ch within the context 
  staff-workstations.cx--.inf.ethz.ch. Assertions about this name would 
  be signed by the authority
  for inf.ethz.ch. Here, the context serves simply as a marker, without enabling
  an alternate signature chain: note that the name simplon.cab.inf.ethz.ch can
  be validly signed by the authority for inf.ethz.ch if no delegation exists
  for cab.inf.ethz.ch. The context simply marks this assertion as internal. This
  allows a client making requests of local names to know they are local, and
  for local resolvers to manage visibility of assertions outside the
  enterprise: explicit context makes accidental leakage of both queries and
  assertions easier to detect and avoid.
- Contexts make captive-portal interactions more explicit: a captive portal 
  resolver could respond to a query for a common website (e.g. www.google.ch)
  with a signed response directed at the captive portal, but within a context
  identifying the location as well as the ISP (e.g. 
  sihlquai.zurich.ch.cx--.starbucks.access.some-isp.net.). This response will
  be signed by the authority for starbucks.access.some-isp.net. This
  signature achieves two things: first, the client knows the result for
  www.google.ch is not globally valid; second, it can present the user with
  some indication as to the identity of the captive portal it is connected to.
 
Further examples showing how context can be used in queries as well are given
in {{context-in-queries}} below.

Developing conventions for assertion contexts for different situations will
require implementation and deployment experience, and is a subject for future
work.

### Signatures in Assertions

A signature over an assertion contains the following information elements:

- Algorithm: identifier of the algorithm used to generate the signature.
- Valid-Since: a timestamp of the start of validity of this signature.
- Valid-Until: a timestamp of the end of validity of this signature.
- Signature: the cryptographic signature itself, whose format is determined by
  the algorithm used.
- Revocation-Token: an optional revocation token for this signature, which 
  allows a signature to be replaced or removed before the end of its validity.
  Revocation tokens are generally based on hash chains, meaning that a
  signature with a revocation token "down" the chain from a given token
  supercedes it. The format and mechanism used by the revocation token is
  determined by the alogrithm used.

The signature protects all the information in an assertion as well as its own
valid-since and valid-until values and the revocation token; it does not
protect other signatures on the assertion.

### Shards and Zones

Assertions may also be grouped and signed as a group. A shard is a set of
assertions subject to the same authority within the same context, protected by
one or more signatures over all assertions within the shard. A shard may have
an additional property that given a subject and an authenticated shard, it can
be shown that either an assertion with a given name and type exists within the
shard or does not exist at all. 

A shard has the following information elements:

- Context: name of the context in which the assertions in the shard are valid;
  see {{context-in-assertions}} above.
- Zone: name of the zone in which the assertions are made.
- Content: a set of assertions sharing the context and zone.
- Signatures: one or more signatures generated by the authority for the
  shard; see {{signatures-in-assertions}}.
- Complete-Flag: if true, the shard is lexicographically complete, and subject
  names that sort such that they would be within the shard if they existed,
  but are not in the shard, can be assumed not to exist.

For efficiency's sake, information elements within a shard common to all
assertions (zone, context, signature) within the shard may be omitted from the
assertions themselves.

A zone is the entire set of shards subject to a given
authority within a given context. There are three kinds of zones;
treating these zones differently may allow lookup protocol
optimizations:

- Zones containing only delegation assertions are delegation-only zones.
  Delegation-only zones are not relevant as part of an assertion lookup, other
  than for discovering and verifying authority. Top-level domains are
  generally delegation-only.
- Zones containing no delegation assertions are final zones. Final zones are
  not relevant as part of an authority discovery.
- Zones containing at least one delegation assertion and at least one
  assertion that is not a delegation assertion are mixed zones. No
  optimizations are available for mixed zones.

A zone has the following information elements:

- Context: name of the context in which the assertions in the zone are valid;
  see {{context-in-assertions}} above.
- Zone: name of the zone.
- Content: a set of assertions and/or shards sharing the context and zone.
- Signatures: one or more signatures generated by the authority for the
  shard; see {{signatures-in-assertions}}.
- Kind: delegation-only, final, or mixed; see above.

## Query

A query is a request for a set of assertions supporting a conclusion about a
given subject-object mapping. It consists of the following information
elements:

- Contexts: an expression of the context(s) in which assertions answering the
  query will be accepted; see {{context-in-queries}} below.
- Qualified-Subject: the name about which the query is made. The subject name
  in a query must be fully-qualified. 
- Types: a set of assertion types the querier is interested in.
- Valid-Until: an optional client-generated timestamp for the query after 
  which it expires and should not be answered.
- Token: an optional client-generated token for the query, which can be used
  in the answer to refer to the query (instead of the answer containing the
  query itself).

A query expresses interest about all the given types of assertion in all the
specified contexts; more complex expressions of which types in which contexts
must be asked using multiple queries.

TODO: provide mechanisms for privacy/performance tradeoffs in queries; are
infomodel changes required here?

### Context in Queries

Contexts are used in queries as they are in assertions (see {{context-in-
assertions}}). Assertion contexts in an answer to a query have to match some
context in the query in order to respond to a query. However, there are a few
additional considerations. An assertion can only exist with a specific
context, while queries may accept answers in multiple contexts. The Contexts
part of a query is a sequence of context specifiers taken to be in order of
decreasing priority. A special null context (represented by the empty string)
indicates that assertions in any context will be accepted. Any context in the
Contexts part of a query may additionally be negated, in order to note that
assertions in those contexts are not acceptable. Negated context name
appearing in the Contexts part of a query before the null context expresses
"any context except these".

Query contexts can also be used to provide additional information to RAINS
servers about the query. For example, contexts can provide a method for
explicit selection of a CDN servers not based on either the client's or the
resolver's address (see {{RFC7871}}). Here, the CDN creates a context for
each of its content zones, and an external service selects appropriate
contexts for the client based not just on client source address but passive
and active measurement of performance. Queries for names at which content
resides can then be made within these contexts, with the priority order of
the contexts reflecting the goodness of the zone for the client. Here, a
context might be zrh.cx--.cdn-zones.some-cdn.com for names of servers
hosting content in a CDN's Zurich data center, and a client could represent
its desire to find content nearby by making queries in the zrh.cx--,
fra.cx-- (Frankfurt), and ams.cx-- (Amsterdam) contexts within cdn-zones
.some-cdn.com. In all cases, the assertions themselves will be signed by the
authority for cdn-zones.some-cdn.com, accurately representing that it is
the CDN, not the owner of the related name in the global context, that is
making the assertion.

As with assertion contexts, developing conventions for query contexts for
different situations will require implementation and deployment experience,
and is a subject for future work.

## Answer

An answer consists of a set of assertions, shards, and/or zones which respond
to a query, bound to that query. It consists of the following information elements:

- Query: the query this answer applies to. If the query was issued with a
  token, the query in the answer may omit all content except the token.
- Content: a set of assertions and/or shards answering the query.

The content of an answer content depends on whether the answer is positive or
negative. A positive answer contains the information requested in the smallest
atomic container that can be found, usually a single assertion. A negative
answer contains the information used to verify it; either a shard with the
Complete-Flag set, an entire Zone, or a Zone-Nameset assertion showing the
name is illegal within the zone.

A query is taken to have an inconclusive answer when no answer returns to the
querier before the query's Valid-Until time.

# Data Model {#cbor}

The RAINS data model is a relatively straightforward mapping of the
information model in {{information-model}} to the Concise Binary Object
Representation (CBOR) {{RFC7049}}, with an outer message type providing a
mechanism for future capabilities-based versioning and recognition of a
message as a RAINS message.

## Symbol Table {#cbor-symtab}

Each CBOR object in a RAINS message is implemented as maps of integer keys to
values, which
implements a good tradeoff between efficiency of representation and
flexibility. The meaning of each of the integer keys is given in the symbol
table below:

| Code | Name           | Description                                   |
|-----:|----------------|-----------------------------------------------|
| 0    | content        | Content of a message, shard, or zone          |
| 1    | capabilities   | Capabilities of server sending message        |
| 2    | signatures     | Signatures on a message or section            |
| 3    | subject-name   | Subject name in an assertion                  |
| 4    | subject-zone   | Zone name in an assertion                     |
| 5    | query-name     | Qualified subject name in a query             |
| 6    | context        | Context of an assertion                       |
| 7    | objects        | Objects of an assertion                       |
| 8    | token          | Token for referring to a data item            |
| 9    | reserved       | Reserved for future use in RAINS              |
| 10   | reserved       | Reserved for future use in RAINS              |
| 11   | shard-range    | Lexical range of Assertions in Shard          |
| 12   | reserved       | Reserved for future use in RAINS              |
| 13   | query-contexts | Reserved for future use in RAINS              |
| 14   | query-types    | acceptable object types for query             |
| 15   | reserved       | Reserved for future use in RAINS              |
| 16   | reserved       | Reserved for future use in RAINS              |
| 17   | note-type      | Notification type                             |
| 18   | reserved       | Reserved for future use in RAINS              |
| 19   | reserved       | Reserved for future use in RAINS              |
| 20   | reserved       | Reserved for future use in RAINS              |
| 21   | reserved       | Reserved for future use in RAINS              |
| 22   | query-opts     | Set of query options requested                |
| 23   | note-data      | Additional notification data                  |

## Message {#cbor-message}

All interactions in RAINS take place in an outer envelope called a Message,
which is a CBOR map tagged with the RAINS Message tag (hex 0xE99BA8, decimal
15309736). 

A Message map MUST contain a content (0) key, whose value is an array of
Message Sections; a Message Section is either an Assertion, Shard, Zone, or
Query.

A Message map MAY contain a capabilities (1) key, whose value is described in
{#cbor-capabilities}.

A Message map MAY contain a signatures (2) key, whose value is an array of
Signatures over the entire message as defined in {{cbor-signature}}, to be verified against the 

A Message map MAY contain a token (8) key, whose value is either an integer or
a UTF-8 string of maximum byte length 32. The token key may be used to refer
to the message in future messages, or may refer to a past message or query by
token. If the message is in response to a message or query containing a token,
the message MUST contain that token.

## Message Section header

Each Message Section in the Message's content value MUST be a two-element
array. The first element in the array is the message section type, encoded as
an integer as in {{cbor-symtab}}. The second element in the array is the
message section body, defined as in {{cbor-assertion}}, {{cbor-shard}},
{{cbor-zone}}, {{cbor-query}}, or {{cbor-notification}}.

Section types are as in the following table, taken from {{cbor-symtab}}:

| Code | Name         | Description                                   |
|-----:|--------------|-----------------------------------------------|
| 1    | assertion    | Assertion (see {{cbor-assertion})             |
| 2    | shard        | Shard (see {{cbor-shard})                     |
| 3    | zone         | Zone (see {{cbor-zone}})                      |
| 4    | query        | Query (see {{cbor-query}})                    |
| 23   | notification | Notification (see {{cbor-notification}})      |

## Assertion body {#cbor-assertion}

An Assertion body is a map. The keys present in this map depend on whether the
Assertion is contained in a Message Section or in a Shard or Zone.

Assertions contained in Message Sections are "bare Assertions". Since they
cannot inherit any values from their containers, they MUST contain the
signatures (2), subject-name (3), subject-zone (4), context (6), and objects
(7) keys.

Assertions within a Shard or Zone are "contained Assertions", and can inherit
values from their containers. A contained Assertion MAY contain the signatures
(2) key and MUST contain the subject-name (3) and objects (7) keys. It MAY
contain subject-zone (4) and context (6) keys, but in this case the values of
these keys MUST be identical to the values in the containing Shard or Zone.

The value of the signatures (2) key, if present, is an array of one or more
Signatures as defined in {{cbor-signature}}. If not present, the containing
Shard or Zone MUST be signed. Signatures on a contained Assertion are
generated as if the inherited values are present in the Assertion, whether
actually present or not.

The value of the subject-name (3) key is a UTF-8 encoded {{RFC3629}} string
containing the name of the subject of the assertion. The subject name never
contains the zone in which the subject name; the fully-qualified name is
obtained by joining the subject-name to the subject-zone with a '.' character.
The subject-name must be valid according to the nameset expression for the
zone, if any.

The value of the subject-zone (4) key, if present, is a UTF-8 encoded string
containing the name of the zone in which the assertion is made. If not
present, the zone of the assertion is inherited from the containing Shard or Zone.

The value of the context (6) key, if present, is a UTF-8 encoded string
containing the name of the context in which the assertion is valid. If not
present, the context of the assertion is inherited from the containing Shard
or Zone.

The value of the objects (7) key is an array of objects, as defined in {{cbor-
object}}.

## Shard body {#cbor-shard}

A Shard body is a map. The keys present in the map depend on whether the Shard
is contained in a Message Section or in a Zone.

Shards contained in Message Sections are "bare Shards". Since they cannot
inherit any values from their contained Zone, they MUST contain the content
(0), signatures (2), subject-zone (4), and context (6) keys.

Shards within a Zone are "contained Shards", and can inherit values from their
containing Zone. A contained Shard MUST contain the content (0) key, and MAY
contain the signatures (2) key and shard-range (11) keys. It MAY contain
subject-zone (4) and context (6) keys, but in this case the values of these
keys MUST be identical to the values in the containing Zone.

The value of the content (0) key is an array of Assertion bodies as defined in
{#cbor-assertion}.

The value of the signatures (2) key, if present, is an array of one or more
Signatures as defined in {{cbor-signature}}. If not present, the containing
Zone MUST be signed. Signatures on a contained Shard are generated as if the
inherited values are present in the Shard, whether actually present or not.

The value of the subject-zone (4) key, if present, is a UTF-8 encoded string
containing the name of the zone in which the Assertions within the Shard is
made. If not present, the zone of the assertion is inherited from the
containing Zone.

The value of the context (6) key, if present, is a UTF-8 encoded string
containing the name of the context in which the Assertions within the Shard
are valid. If not present, the context of the assertion is inherited from the
containing Zone.

If the shard-range (11) key is present, the shard is lexicographically
complete within the range described in its value: a mapping for a (subject-
name, object-type) pair that should be between the two values given in the
range but is not is asserted to not exist. Lexicographic sorting is done on
subject names by ordering Unicode codepoints in ascending order; ordering on
object types is done via their code values in {{cbor-object}} in ascending order

The shard-range value MUST be a four element array of (subject-name A, object-
type A, subject-name B, object type B) where A does not necessarily need to
sort before B, and the (subject-name, object-type) pairs need not exist in the
shard. The shard MUST NOT contain any assertions for subject-names outside the
range.

If the shard-range key is not present, the shard is not lexicographically
complete and MUST NOT be used to make assertions about nonexistance.

## Zone Message Section body {#cbor-zone}

A Zone body is a map. Zones MUST contain the content (0), signatures (2),
subject-zone (4), and context (6) keys.

The value of the content (0) key is an array of Shard bodies as defined in
{#cbor-shard} and/or Assertion bodies as defined in {#cbor-assertion}.

The value of the subject-zone (4) key is a UTF-8 encoded string
containing the name of the Zone.

The value of the context (6) key is a UTF-8 encoded string
containing the name of the context for which the Zone is valid.

TODO: determine if Zones MUST contain all the valid assertions within the
Zone. I think so. This leads (as with inconsistent Shards) to the question of
"what happens if not", and defending against malicious inconsistency.

## Query Message Section body {#cbor-query}

A Query body is a map. Queries MUST contain the query-name (5), query-contexts (13),
and query-type (14) keys. Queries MAY contain the token(8) key and the 
query-opts (22) key.

The value of the query-name (5) key is a UTF-8 encoded string containing the
fully qualified name that is the subject of the query.

The value of the query-contexts (13) key is an allowable context expression, as an
array of context names as UTF-8 encoded strings. The allowable context
expression is evaluated in-order, as follows:

- Context names appearing earlier in the expression are given priority over
  context names appearing later in the expression.
- A context name may be negated by prepending the context negation marker 
  'cx--0-.' to the context name; a negated context name means the named context
  is not acceptable in answers to this query.
- The special context name '.' refers to the global context.
- The special context name 'cx--any-' means 'any context is acceptable'.

Some examples:

- ['cx--.inf.ethz.ch', 'cx--any-'] means that answers in the 
  'cx--.inf.ethz.ch' context are preferred, but any context is acceptable; 
- ['.', 'cx--.inf.ethz.ch'] means that only answers in the
  'cx--.inf.ethz.ch' or global contexts are acceptable, with the global
  context preferred;
- ['.', cx--0-.cx--.inf.ethz.ch', 'cx--any-'] means that answers in any 
  context except 'cx--.inf.ethz.ch' are acceptable, with the global context
  preferred.

An empty context array in a query is taken to be equivalent to an array
containing only ['.', 'cx--any-']; i.e. any context acceptable, global context
preferred.

The value of the query-type (14) key is an array of integers encoding the
type(s) of objects (as in {{cbor-object}}) acceptable in answers to the query.
All values in the query-type array are treated at equal priority: [2,3] means
the querier is equally interested in both IPv4 and IPv6 addresses for the
query-name.

The value of the token (8) key, if present, is either an integer or a UTF-8
string of maximum byte length 32. Future messages or notifications containing
answers to this query MUST contain this token, if present.

The value of the query-opts (22) key, if present, is an array of integers in
priority order of the querier's preferences in tradeoffs in answering the
query.

| Code | Description                                                    |
|-----:|----------------------------------------------------------------|
| 1    | Minimize end-to-end latency                                    |
| 2    | Minimize last-hop answer size (bandwidth)                      |
| 3    | Minimize information leakage beyond first hop                  |
| 4    | No information leakage beyond first hop: cached answers only   |
| 5    | Expired assertions are acceptable                              |
| 6    | Enable token tracing                                           |
| 7    | Disable verification delegation (client protocol only)         |

Each server is free to determine how to minimize each performance metric
requested; however, servers MUST NOT generate queries to other servers if "no
information leakage" is specified, and servers MUST NOT return expired
assertions unless "expired assertions acceptable" is specified.

## Notification Message Section body {#cbor-notification}

Notification Message Sections contain information about the operation of the
RAINS protocol itself. A Notification Message Section body is a map which MUST
contain the note-type (17) key and MAY contain the token (8) and note-data (23) keys. The
value of the note-type key is encoded as an integer as in the following table:

| Code | Description                                                    |
|-----:|----------------------------------------------------------------|
| 100  | Connection heartbeat                                           |
| 399  | Capability hash not understood                                 |
| 400  | Malformed message received                                     |
| 403  | Inconsistent message received                                  |
| 404  | No assertion exists (client protocol only)                     |
| 413  | Message too large                                              |
| 500  | Unspecified server error                                       |
| 501  | Server not capable                                             |
| 504  | No assertion available                                         |

Note that the status codes are chosen to be mnemonically similar to status
codes for HTTP {{RFC7231}}. Details of the meaning of each status code are
given in {{protocol-def}}.

The value of the token (8) key, if present, is either an integer or a UTF-8
string of maximum byte length 32. If the notification is in response to a
message or query containing a token, the notification MUST contain that token.

The value of the note-data (23) key, if present, is a UTF-8 encoded string
with additional information about the notification, intended to be displayed
to an administrator to help debug the issue identified by the negotiation.

## Object {#cbor-object}

Objects are encoded as arrays in CBOR, where the first element is the type of
the object, encoded as an integer in the following table:

| Code  | Name         | Description                                   |
|------:|--------------|-----------------------------------------------|
| 1     | name         | name associated with subject                  |
| 2     | ip6-addr     | IPv6 address of subject                       |
| 3     | ip4-addr     | IPv4 address of subject                       |
| 4     | redirection  | name of zone authority server                 |
| 5     | delegation   | public key for zone delgation                 |
| 6     | nameset      | name set expression for zone                  |
| 7     | cert-info    | certificate information for name              |
| 8     | service-info | service information for srvname               |
| 9     | registrar    | registrar information                         |
| 10    | registrant   | registrant information                        |
| 11    | infrakey     | public key for RAINS infrastructure           |
| 12-23 | reserved     | Reserved for future use in RAINS              |

A name (1) object contains a name associated with a name as an alias. It is
represented as a two-element array. The second element is a fully-qualified
name as a UTF-8 encoded string.

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

A delegation (5) object contains the public key used to generate signatures
on assertions in a named zone, and by which a delegation of a name within a
zone to a subordinate zone may be verified. It is represented as an N-element
array. The second element is a signature algorithm identifier as in 
{{cbor-signature}}. Additional elements are as defined in {{cbor-signature}} 
for the given algorithm identifier.

A nameset (6) object contains an expression defining which names are allowed
and which names are disallowed in a given zone. It is represented as an N-
element array, as defined in {{cbor-nameset}}

A cert-info (7) object contains an expression binding a certificate or
certificate authority to a name, such that connections to the name must either
use the bound certificate or a certificate signed by a bound authority. It is
represented as an N-element array, as defined in {{cbor-certinfo}}.

A service-info (8) object gives information about a named service. Services
are named as in {{RFC2782}}. It is represented as a four-element array. The
second element is a fully-qualified name of a host providing the named service
as a UTF-8 string. The third element is a transport port number as a positive
integer in the range 0-65535. The fourth element is a priority as a positive
integer, with lower numbers having higher priority.

A registrar (9) object gives the name and other identifying information of
the registrar (the organization which caused the name to be added to the
namespace) for organization-level names. It is represented as a UTF-8 string
containing identifying information chosen by the registrar according to the
registry's policy.

A registrant (10) object gives information about the registrant of an
organization-level name. It is represented as a UTF-8 string containing this
information, with a format chosen by the registrar according to the registry's
policy.

An infrakey (11) object contains the public key used to generate signatures on
messages by a named RAINS server, by which a RAINS message signature may be
verified by a receiver. It is identical in structure to a delegation object,
as defined in {{cbor-signature}}.

## Signatures, delegation keys, and RAINS infrastructure keys {#cbor-signature}

RAINS supports multiple signature algorithms and hash functions for signing
assertions for cryptographic algorithm agility {{RFC7696}}. A RAINS signature
algorithm identifier specifies the signature algorithm; a hash function for
generating the HMAC; a method for generating, verifying, and interpreting hash
chain tokens in signatures; and the format of the encodings of the signature
values in Assertions, Shards, Zones, and Messages, as well as of public key
values in delegation objects.

RAINS signatures have four common elements: the algorithm identifier, a valid-
since timestamp, a valid-until timestamp, and a hash chain token. Signatures
are represented as an array of these four values followed by additional
elements containing the signature data itself, according to the algorithm
identifier.

Valid-since and valid-until timestamps are represented as CBOR integers
counting seconds since the UNIX epoch UTC, identified with tag value 1 and
encoded as in section 2.4.1 of {{RFC7049}}. A signature MUST have a valid-
until timestamp. If a signature has no specified valid-since time (i.e., is
valid from the beginning of time until its valid-until timestamp), the valid-
since time MAY be null (as in Table 2 in Section 2.3 of {{RFC7049}}).

Hash chain tokens and their use are specified in the appropriate subsection of
this section for the given algorithm identifier.

Signatures in RAINS are generated over a normalized serialized CBOR object (a
Message; or an Assertion, Shard, or Zone section body). To normalize and
serialize an object for sigining or verifying:

- Remove all signatures from the object.
- Add a new "blank" signature to the object, containing Null in the place 
  of any elements containing signature data. 
- Serialize the object, emitting all keys in CBOR maps in ascending numerical
  order. Note that when serializing anything with a Content array, the order 
  of the content array is preserved. If the serialized object is a Message, 
  it should be tagged with the RAINS tag.
- Generate a signature on the resulting byte stream according to the 
  algorithm selected, and fill the resulting values into the "blank" 
  signature; or verify the signature of the resulting byte stream according 
  to the algorithm selected.

The following algorithms are supported:

| Code | Signatures | Hash/HMAC | Format               | Token                  |
|-----:|------------|-----------|----------------------|------------------------|
| 2    | ecdsa-256  | sha-256   | See {{ecdsa-format}} | See {{hash-chain-rev}} |
| 3    | ecdsa-384  | sha-384   | See {{ecdsa-format}} | See {{hash-chain-rev}} |

### ECDSA signature and public key format {#ecdsa-format}

ECDSA public keys consist of a single value, called "Q" in {{FIPS-186-3}}. Q
is a simple bit string that represents the uncompressed form of a curve point,
concatenated together as "x | y". The third element in a RAINS delegation
object is the Q bit string encoded as a CBOR byte array. RAINS delegation
objects for ECDSA-256 public keys are therefore represented as the array 
[5, 2, Q]; and for ECDSA-384 public keys as [5, 3, Q].

ECDSA signatures are a combination of two non-negative integers, called "r"
and "s" in {{FIPS-186-3}}. A Signature using ECDSA is represented using a six
element CBOR array, with the fifth element being r represented as a byte array
as described in Section C.2 of {{FIPS-186-3}}, and the sixth being s
represented as a byte array as described in Section C.2 of {{FIPS-186-3}}. For
ECDSA-256 signatures, each integer MUST be represented as a 32-byte array. For
ECDSA-384 signatures, each integer MUST be represented as a 48-byte array.
RAINS signatures using ECDSA-256 are therefore the array [2, valid-from,
valid-until, token, r, s]; and for ECDSA-384 the array [3, valid-from,
valid-until, token, r, s].

ECDSA-256 signatures and public keys use the P-256 curve as defined in {{FIPS-186-3}}.
ECDSA-384 signatures and public keys use the P-384 curve as defined in {{FIPS-186-3}}.

All RAINS servers MUST implement ECDSA-256 and ECDSA-384.

[EDITOR'S NOTE: make sure this is appropriately modern, check work in CURDLE.]

### Hash-chain based revocation {#hash-chain-rev}

Hash-chain based revocation allows a signature (and the Assertion, Shard, or
Zone it protects) to be replaced before it expires. To use hash-chain based
revocation, a signing entity generates a hash chain from a known seed using
the hash function specified by the signature algorithm in use, and places the
Nth value derived therefrom in the hash chain revocation token on a signature.

A revocation can be issued by generating a new section and signing it,
revealing the N-1st value from the hash chain in the revocation token. To
allow a recipient of a revoked section to verify the revocation, the following
restrictions on what can replace what apply:

- An Assertion can only be replaced by another Assertion with the same 
  Subject within the same Context and Zone, containing an Objects array 
  of the same length containing the same types of Objects. To delete Object 
  values, those values can be replaced with Null in the replacing Assertion.
- A Shard can only be replaced by another Shard with an identical shard-range 
  key, within the same Context and Zone. Incomplete Shards cannot be replaced.
- A Zone can only be replaced by another Zone with an identical name within 
  the same Context.

Signing entities can decline to use hash-chain based revocation by replacing
the revocation token with Null.

## Certificate information format {#cbor-certinfo}

[EDITOR'S NOTE: TODO. this should be largely in line with TLSA; determine if there's
any guidance from implementation experience we should consider, as well.]

## Name expression format {#cbor-nameset}

[EDITOR'S NOTE: Write me.]

## Capabilities {#cbor-capabilities}

When a RAINS server or client sends the first message in a stream to a peer, it MAY expose optional
capabilities to its peer using the capabilities (1) key. This key contains either:

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

Since there are only two defined capabilities at this time, RAINS servers can
be implemented with two hard-coded hashes to determine whether a peer is
listening or not. The hash presented by a server supporting urn:x-rains:tlssrv
is e5365a09be554ae55b855f15264dbc837b04f5831daeb321359e18cdabab5745; the hash
presented by a server supporting no capabilities is
76be8b528d0075f7aae98d6fa57a6d3c83ae480a8469e668d7b0af968995ac71.

A RAINS server MUST NOT assume that a peer server supports a given capability
unless it has received a message containing that capability from that server.
An exception are the capabilities indicating that a server listens for
connections using a given transport protocol; servers and clients can also
learn this information from RAINS itself (given a redirection assertion for a
named zone) or from external configuration values.

# RAINS Protocol Definition {#protocol-def}

As noted in {{cbor}}, RAINS is a message-exchange protocol that uses
CBOR {{RFC7049}} as its framing. Since CBOR is self-framing -- a CBOR parser
can determine when a CBOR object is complete at the point at which it has read
its final byte -- RAINS requires no external framing. It can therefore run
over any streaming, multistreaming, or message-oriented transport protocol. In
order to protect query confidentiality, and support rapid deployment over a
ubiquitously implemented transport, RAINS is defined in this document to run
over persistent TLS 1.2 connections {{RFC5246}} over TCP {{RFC0793}} with
mutual authentication. The TLS certificates of RAINS server peers can be
verified as specified in the certificate assertions for those servers.

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
servers. A simplified version of the protocol for client access is described
in {{protocol-client}}.

## Message processing {#protocol-processing}

Once a transport is established, any server may validly send a message with
any content to any other server. Upon receipt of a message, a server parses it, and:

- notes the token on the message. If present, the token MUST be present on any
  messages sent in reply to this message.
- processes any capabilities present, replacing the set of capabilities known
  for the peer with the set present in the message. If the present
  capabilities are represented by a hash that the server does not have in its
  cache, it prepares a notification of type 399 "Capability hash not
  understood" to send to its peer.
- splits the contents into its constituent message sections, processing them
  independently.

On receipt of an assertion, shard, or zone message section, the server:

- verifies its consistency (see {{runtime-consistency-checking}}). If the
  section is not consistent, it prepares to send a notification of type 403
  Inconsistent Message to the peer, and discards the section. Otherwise, it:
- determines whether it answers an outstanding query; if so, it prepares to
  forward the section to the server that issued the query.
- determines whether it replaces any information presently in its cache (see
  {{hash-chain-rev}}). If so, it discards the old information, and caches the
  new section.
- determines whether it is likely to answer a future query, according to its 
  configuration, policy, and query history; if so, it caches the section.

On receipt of a query, the server:

- determines whether it has a stored assertion, shard, and/or zone message
  section which answers the query. If so, it prepares to return the most
  specific such section with the signature of the longest remaining validity to the
  peer that issued the query. If not, it:
- checks to see whether the query specifies option 4 (cached answers only). If
  so, and if option 5 (expired assertions acceptable) is also specified, it
  then checks to see if it has any cached sections that answer the query on
  which signatures are expired; otherwise, processing stops. If the query does
  not specify option 4, delegation proceeds as follows:
- determines whether it has other non-authoritative servers it can forward the
  query to, according to its configuration and policy, and in compliance with
  any query options (see {{cbor-query}}). If so, it prepares to forward the
  query to those servers, noting the reply for the received query depends on
  the replies for the forwarded query. If not, it:
- determines the responsible authority servers for the zone containing the
  query name in the query for contexts requested, and forwards the query to
  those authority servers, noting the reply for the received query depends on
  the reply for the forwarded query.

If query delegation fails to return an answer within a configured timeout for
a delegated query, the server prepares to send a 504 No assertion available
response to the peer from which it received the query.

When a server creates a new query to forward to another server in response to
a query it received, it SHOULD NOT use the same token on the delegated query
as on the received query, unless option 6 Enable Tracing is present in the
received , in which case it MUST use the same token. The Enable Tracing option
is designed to allow debugging of query processing across multiple servers,
and MUST NOT be enabled by default.

On receipt of a notification, the server's behavior depends on the notification type:

- For type 100 "Connection Heartbeat", the server does nothing: these null
  messages are used to keep long-lived connections open in the presence of
  network behaviors that may drop state for idle connections.
- For type 399 "Capability hash not understood", the server prepares to send a
  full capabilities list on the next message it sends to the peer.
- For type 504 "No assertion available", the server checks the token on the
  message, and prepares to forward the assertion to the associated query.
- For type 413 "Message too large" the server notes that large messages may 
  not be sent to a peer and tries again (see {{protocol-limits}}), or logs
  the error along with the note-data content.
- For type 400 "Malformed message", type 403 "Inconsistent message", type 500
  "Server error", or type 501 "Server not capable", the server logs the error
  along with the note-data content, as these notifications generally represent
  implementation or configuration error conditions which will require human
  intervention to mitigate.

The first message a server sends to a peer after a new connection is
established SHOULD contain a capabilities section, if the server supports any
optional capabilities. See {{cbor-capabilities}}.

If the server is configured to keep long-running connections open, due to the
presence of network behaviors that may drop state for idle connections, it
SHOULD send a message containing a type 100 Connection Heartbeat notification
after a configured idle time without any messages containing other content
being sent.

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
length restrictions with a notification of type 414 (Message Too Large). A
server that receives a type 413 notification must note that the peer sending
the message only accepts messages smaller than the largest message it's
successfully sent that peer, or cap messages to that peer to 65536 bytes in
length.

Since a bare assertion with a single ECDSA signature requires on the order of
180 bytes, it is clear that many full zones won't fit into a single minimum
maximum-size message. Authorities are therefore encouraged to publish zones
grouped into shards that will fit into 65536-byte messages, to allow servers
to reply using these shards when full-zone transfers are not possible due to
message size limitations.

## Runtime consistency checking

The data model used by the RAINS protocol allows inconsistent information to
be asserted, all resulting from misconfigured or misbehaving authority
servers. For example, a shard valid at a given point in time can be marked
lexically complete, but not contain an assertion within its lexical range,
which is also valid at that point. This would allow both proof of the presence
and absence of an assertion at the same time, which is clearly nonsensical.

RAINS relies on runtime consistency checking to mitigate this problem: each
server receiving an assertion, shard, or zone SHOULD, subject to resource
constraints, ensure that it is consistent with other information it has, and
if not, discard all assertions, shards, and zones in its cache, log the error,
and send a 403 Inconsistent Message to the source of the message.

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
system. Specifically, the information model and the mechanism for proving non-
existence of an assertion is not designed to provide resistance against zone
enumeration.

On the other hand, confidentiality protection of query information in crucial.
Linking naming queries to a specific user can be nearly as useful to build a
profile of that user for surveillance purposes as full access to the clear
text of that client's communications {{RFC7624}}. In this revision, RAINS uses
TLS to protect communications between servers and between servers and clients,
with certificate information for RAINS infrastructure stored in RAINS itself.
Together with hop-by-hop confidentiality protection, query options, proactive
caching, default use of non-persistent tokens, and redirection among servers
can be used to mix queries and reduce the linkability of query information to
specific clients.

# RAINS Client Protocol {#protocol-client}

The protocol used by clients to issue queries to and receive responses from an
query service is a subset of the full RAINS protocol, with the following differences:

- Clients only process assertion, shard, zone, and notification sections;
  sending a query to a client results in a 400 Malformed Message notification.
- Clients never listen for connections; a client must initiate and maintain a
  transport session to the query server(s) it uses for name resolution.
- Servers only process query and notification sections when connected to
  clients; a client sending assertions to a server results in a 400 Malformed
  Message notification.

Since signature verification is resource-intensive, clients delegate signature
verification to query servers by default. The query server signs the message
containing results for a query using its own key (published as an infrakey
object associated with the query server's name), and a validity time corresponding
to the signature it verified with the longest lifetime, stripping other
signatures from the reply. This behavior can be disabled by a client by
specifying query option 7, allowing the client to do its own verification.

# Deployment Considerations

The following subsections discuss issues that must be considered in any
deployment of RAINS at scale.

## Assertion lifetime management

An assertion can contain multiple signatures, each with a different lifetime.
Signature lifetimes are equivalent to a time to live in the present DNS:
authorities should compute a new signature for each validity period, and make
these new signatures available when old ones are expiring.

If an unexpected change to an assertion is necessary, the hash chain based
replacement mechanism described in {{hash-chain-rev}} provides a way for an
authority to replace signed assertions with new information or with Null
objects, in the case of deletion.

Since assertion lifetime management is based on a real-time clock expressed in
UTC, RAINS servers MUST use a clock synchronization protocol such as NTP {{RFC5905}}.

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

## Query Service Discovery

A client that will not do its own verification must be able to discover the
oracle server(s) it should trust for resolution. Integration with e.g. DHCP or
selection of a local multicast discovery method are left to a future revision
of this document.

In any case, clients MUST provide a configuration interface to allow a user to
specify (by address or name) and/or constrain (by certificate property) a
preferred/trusted oracle. This would allow client on an untrusted network to
use an untrusted locally-available oracle to discover a preferred oracle
(doing key verification on its own for bootstrapping), before connecting to
that oracle for normal name resolution.

## Transition using translation between RAINS and DNS information models

Full adoption of RAINS would require changes to every client device (replacing
DNS stub resolvers with RAINS clients) and name server on the Internet. In
addition, most client software would need to change, as well, to get the full
benefits of explicit context in name resolution. This is a wholly unrealistic goal.

RAINS servers can, however, coexist with Domain Name System servers and
clients during an indefinite transition period. RAINS assertions can be
algorithmically translated into DNS answers, and RAINS queries can be
algorithmically translated into DNS queries, by RAINS to DNS gateways, given
the mostly compatible information models used by the two.

While DNSSEC and RAINS keys for equivalent ciphersuites are compatible with
each other, there is no equivalent to query option 7 for gateways, since the
RAINS signatures are generated over the RAINS bytestream for an assertion, not
the DNS bytestream. Therefore, RAINS to DNS gateways must provide verification
services for DNS clients. DNS over TLS {{RFC7858}} SHOULD be used between the
DNS client and gateway to ensure confidentiality and integrity for queries and
answers.

Object type mappings are as follows:

- Objects of type name can (largely) be represented as CNAME RRs.
- Objects of type ip6-addr can be represented as AAAA RRs.
- Objects of type ip4-addr can be represented as A RRs.
- Objects of type redirection can be represented as NS RRs.
- Objects of type cert-info can be represented as TLSA RRs 
  [EDITOR'S NOTE make sure we design them so this is true].
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

The urn:x-rains namespace used by the RAINS capability mechanism in {{cbor-
capabilities}} may be a candidate for replacement with an IANA-registered
namespace in a future revision of this document.

# Security Considerations

This document specifies a new, experimental protocol for Internet name
resolution, with mandatory integrity protection for assertions about names
built into the information model, and confidentiality for query information
protected on a hop-by-hop basis. See especially {{signatures-in-assertions}},
{{integrity-and-confidentiality-protection}}, {{cbor-signature}}, 
{{cbor-certinfo}}, and {{secret-key-management}} for security-relevant 
details.

# Acknowledgments

Thanks to Daniele Asoni, Laurent Chuat, Ted Hardie, Joe Hildebrand, Steve
Matsumoto, Adrian Perrig, Raphael Reischuk, Stephen Shirley, Andrew Sullivan,
and Suzanne Woolf for the discussions leading to the design of this protocol.

--- back

# Open Issues

- The format of the nameset and certinfo object types needs to be specified.
- A method for clients to discover local oracles needs to be specified.
