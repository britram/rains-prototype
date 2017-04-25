---
title: RAINS Parameters for SCION
abbrev: RAINS
docname: draft-trammell-rains-scion-00
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

normative:
    I-D.trammell-inip-pins:
    I-D.trammell-rains-protocol:

informative:
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


--- abstract

This document defines additional functionality atop the RAINS naming service
specific to the SCION Internet architecture.

--- middle


# SCION Address Objects

SCION addresses are based on IPv6 addresses, adding an Isolation Domain (ISD)
and Autonomous System (AS) number. These two additional address components are
used to select paths between a source and destination of traffic. RAINS for
SCION adds an additional object data type to those listed in section 5.12 of
{{I-D.trammell-rains-protocol}} to support SCION addresses.

| Code  | Name         | Description                             |
|------:|--------------|-----------------------------------------|
| 23    | scion-addr   | SCION address of subject                |

A scion-addr (23) object contains an SCION address associated with a name.  It
is represented as a four element array.  The second element is an ISD number as
an integer less than or equal to 2^16. The third element is an AS number as an
integer less than or equal to 2^32. The fourth element is a byte array of length
16 containing an IPv6 address in network byte order.

 ## Address to name mappings for SCION addresses

[EDITOR'S NOTE: This is an important open issue to discuss.]

# ARPKI certificates in RAINS Certificate objects

{: #tabcertusage title="Additional certificate information usage values"}

| Code | Certificate usage          |
|-----:|----------------------------|
|   23 | Policy binding certificate |

[EDITOR'S NOTE: write me, discuss which types we need here.]

# ARPKI keyspace

[EDITOR'S NOTE: write me, define how to do an SCP ]

| Keyspace ID | Name  | Signature Verification Algorithm                 |
|------------:|-------|--------------------------------------------------|
| 1           | arpki | ARPKI SCP signature; see {{arpki-keyspace}}      |


# RAINS servers over the SCION Socket Protocol (SSP)

[EDITOR'S NOTE: write me, once SSP is defined to the point we have something to cite.]