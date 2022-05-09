---
title: Automated Certificate Management Environment (ACME) Onion Identifier Validation Extension
abbrev: ACME-ONION
docname: draft-suchan-acme-onionv3-00
date: 2022-05-09
category: std

stand_alone: yes

ipr: trust200902
area: General
workgroup: ACME Working Group
submissionType: IETF

coding: us-ascii
pi:    # can use array (if all yes) or hash here
#  - toc
#  - sortrefs
#  - symrefs
  toc: yes
  sortrefs:   # defaults to yes
  symrefs: yes

author:
  -
    ins: Seo Suchan
    name: Seo Suchan
    email: tjtncks@gmail.com

normative:
  RFC2119:
  RFC4086:
  RFC4648:
  RFC5280:
  RFC7686:
  RFC8174:
  RFC8555:
  RFC8737:
  cabr:
    title: CA/B forum baseline requirement
    author:
      name:  CA/Browser Forum
    date: 2022-04-26
    target: https://cabforum.org/wp-content/uploads/CA-Browser-Forum-BR-1.8.4.pdf
  Toraddr:
    title: Tor address spec
    author:
      name: Nick Mathewson
    date: 2021-08-24
    target: https://github.com/torproject/torspec/blob/main/address-spec.txt

--- abstract

This document specifies identifiers and challenges required to enable the Automated Certificate Management Environment (ACME) to issue certificates for Tor Project's onion V3 addresses.

--- middle

# Introduction

While onion addresses {{RFC7686}} are in form of DNS address, they aren't in part of ICANN hierarchy, and onion name's self-verifying construction warrents different verification, duce different identfier type for them is worth consider.

# Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

# Onion Identifier

{{RFC8555}} only defines the identifier type "dns", that assumes it's on public CA/B hirearuchy, so to make clear Onion addresses are not using normal DNS valification methods, we assign a new identifier type for onion v3 addresses, "onion-v3".

This document only handles V3 version of onion address as defined in {{Toraddr}}, identified by 56 letters base domain name ends with d. 

This document doesn't handle about velification of version 2 of onion addresses, as they are retired already

An identifier for the onion address acmeulkebl5..4zcuv5hk7fqwad.onion would be formatted like:

~~~~~~~~~~
{"type": "onion-v3", "value": "acmeulkebl5...z4zcuv5hk7fqwad.onion"}
~~~~~~~~~~

Keep mind in CSR this address still treated as DNS.

# Validation Challenges for onion address

Onion-v3 identifiers MAY be used with the existing "http-01" and "tls-alpn-01" challenges from {{RFC8555}} Section 8.3 and {{RFC8737}} Section 3 respectively. To use Onion identifiers with these challenges their initial DNS resolution step MUST be skipped and the approperate Tor daemon that in control of CA MUST used to proxy such request.

The exsisting "dns-01" challange MUST NOT be used to validate onion addresses.

In addition to challanges earlier RFC defined, there 

## CSR signed with Onion public key challagne

With Onion-csr validation, the client in an ACME transaction proves its control of oninon address by proving the possesion of onion hidden service identity key. The ACME server challenges the client to sign CSR that includes the nonce it gave with.

The Onion-csr ACME challenge object has the following format:

type (required, string):
: The string "onion-v3-csr

token (required, string):
: A random value that uniquely identifies the challenge. This value MUST have at least 128 bits of entropy. It MUST NOT contain any characters outside the base64url alphabet as described in {{RFC4648}} Section 5. Trailing '=' padding characters MUST be stripped. See {{!RFC4086}} for additional information on randomness requirements.

The client prepares for validation by constructing a self-signed CSR that MUST contain an cabf caSigningNonce Attribute and a subjectAlternativeName extension {{!RFC5280}}. The subjectAlternativeName extension MUST contain a single dNSName entry where the value is the domain name being validated. The cabf caSigningNonce Attribute MUST contain the token string as ascii encoded for the challenge.

The cabf caSigningNonce Attribute is identified by the cabf-caSigningNonce object identifier (OID) in the cabf arc {{!RFC5280}}. conseurt {{cabr}} appendex B for how to construct CSR itself in detail.

~~~~~~~~~~
cabf OBJECT IDENTIFIER ::= 2.23.140 
{ joint-iso-itu-t(2) internationalorganizations(23) ca-browser-forum(140) }

caSigningNonce ATTRIBUTE ::= {
WITH SYNTAX OCTET STRING
EQUALITY MATCHING RULE octetStringMatch
SINGLE VALUE TRUE
ID { cabf-caSigningNonce }
}
cabf-caSigningNonce OBJECT IDENTIFIER ::= { cabf 41 }

applicantSigningNonce ATTRIBUTE ::= {
WITH SYNTAX OCTET STRING
EQUALITY MATCHING RULE octetStringMatch
SINGLE VALUE TRUE
ID { cabf-applicantSigningNonce }
}
cabf-applicantSigningNonce OBJECT IDENTIFIER ::= { cabf 42 }

~~~~~~~~~~

A client fulfills this challenge by construct the challange CSR from the "token" value provided in the challange, then POST on challange URL with crafted CSR as payload to request validated by the server.

~~~~~~~~~~
POST /acme/authz/1234/1
Host: example.com
Content-Type: application/jose+json

{
  "protected": base64url({
    "alg": "ES256",
    "kid": "https://example.com/acme/acct/1",
    "nonce": "XaYcb5XoTUgRWbTWw_NwkcP",
    "url": "https://example.com/acme/authz/1234/1"
  }),
  "payload": base64url({
       "csr": "MIIBBzCBugIBADAAMCo...gRYTMAhRP8nIH",
     }),
  "signature": "0wSzBJBgNVHREEQ...gYJKoZIhvcNAQkOM"
}
~~~~~~~~~~

On receiving this request from client, the server verifies client's control over the onion address by verifiy that CSR is crafted with expected properties:

1. CSR is signed with private part of identity key the requested onion address made from.
2. A caSigningNonce attribute that contains token Value that challenge object provided.
3. A applicantSigningNonce attribute that contains client-genetarted random value. This value SHOULD include at least 64bits of entropy. (CA/BR requirement)




# IANA Considerations

## Identifier Types

Adds a new type to the Identifier list defined in Section 9.7.7 of {{RFC8555}} with the label "onion-v3" and reference I-D.ietf-acme-onion.

## Challenge Types

in the Validation Methods list defined in Section 9.7.8 of {{RFC8555}}:

Adds the raw "onin-challange-csr" to the Validation Methods.

Adds the value "onion-v3-csr" to the Identifier Type column  for the "http-01", "onion-challange-csr", and "tls-alpn-01" challenges.

# Security Considerations

As onion addresses are able to generated in massive quantity without financial cost, it bypasses the normal ratelimit CAs imposess. CAs SHOULD adapt some mesure to prevent DoSing the CA by create hugh amount of request for onion address. For exemple, imposing limit per ACME account or require order to have at least one non-onion domain.

# note
this doc is made as documentation of my pebble tree does: process may change in track