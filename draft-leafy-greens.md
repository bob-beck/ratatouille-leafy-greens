---
title: "End Entity Name Restrictions"
abbrev: "EENR"
category: info
docname: draft-leafy-greens-latest
# This is not a PLANT pun.
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
# area: AREA
# workgroup: WG Working Group
keyword:
 - TLS
 - PKIX
 - X.509
 - name constraints
venue:
#  group: WG
#  type: Working Group
#  mail: WG@example.com
#  arch: https://example.com/WG
  github: "bob-beck/ratatouille-leafy-greens"
  latest: "https://bob-beck.github.io/ratatouille-leafy-greens/draft-leafy-greens.html"

author:
 -
    fullname: "Bob Beck"
    organization: OpenSSL
    email: "beck@obtuse.com"

normative:
  RFC5280:
  RFC9525:

informative:

...

--- abstract

The interaction of name constraint matching in {{RFC5280}} and
wildcard subject alternative names creates a gap in which an
excluded name constraint cannot be relied upon to prevent the
issuance of certificates usable for the excluded name. This
document defines End Entity Name Restrictions (EENR), a new
critical X.509 extension for CA certificates that constrains the
`dNSName` Subject Alternative Name entries which may appear in
end entity certificates issued beneath the CA. EENR specifies its
own matching semantics, including for wildcard `dNSName` entries,
so that it does not depend on application-defined interpretations.
The extension is scoped to use in certificate path validation for
TLS client and TLS server authentication.

--- middle

# Introduction

X.509 name constraints, defined in {{RFC5280}} section 4.2.1.10,
allow a CA certificate to restrict the names that may appear in
any certificate beneath it in a certification path. Enforcement
is performed during certification-path validation, per {{RFC5280}}
sections 6.1.3 and 6.1.4.

For `dNSName` entries, {{RFC5280}}'s matching algorithm is purely
lexical: *"Any DNS name that can be constructed by simply adding
zero or more labels to the left-hand side of the name satisfies
the name constraint."* Read literally, this treats `*` as an
ordinary label and makes `*.example.com` satisfy a constraint of
`example.com` exactly as `host.example.com` does. {{RFC5280}}
section 4.2.1.6 separately leaves the semantics of wildcards in
subject alternative names undefined, requiring any application
that uses them to define the semantics itself. {{RFC9525}} supplies
wildcard semantics for one application context, matching presented
identifiers against reference identifiers in TLS, but its
section 1.2 explicitly defers name constraints back to {{RFC5280}}.

For example, consider a CA certificate with an excluded constraint
of `foo.example.com`, signing an end entity certificate with a SAN
of `*.example.com`. The apparent intent of the constraint is to
prevent that CA from issuing certificates usable as
`foo.example.com`. The SAN does not fall within the excluded
subtree under {{RFC5280}}'s matching, and the certificate passes
name-constraint validation. The resulting end entity certificate
can then be presented to TLS clients as the server identity for
`foo.example.com`: {{RFC9525}} section 6.3 defines `*.example.com`
as a valid presented identifier matching the reference identifier
`foo.example.com`. The constraint does what {{RFC5280}} prescribes:
it rejects a literal SAN of `foo.example.com`. It does not,
however, prevent the CA from issuing certificates that can be
used as `foo.example.com`.

A future specification that defines wildcard semantics for name
constraint matching cannot remedy this for the general case.
Implementations conforming to {{RFC5280}} as written today would
remain conformant when applying the literal algorithm indefinitely,
and any verifier predating such a clarification would continue to
do so. Unless the CA can constrain its certificates to verifiers
known to apply specific semantics, it cannot ensure that the
verifiers eventually consuming them share any single interpretation.
Criticality does not remedy this: {{RFC5280}} deferred the wildcard
portion of `nameConstraints` semantics, so the meaning of this
critical extension can change without changing {{RFC5280}}, and a
verifier processing it conformantly may be applying semantics no
longer current.

A PKI that depends on `excludedSubtrees` for security therefore
cannot rely on those exclusions being enforced consistently across
the verifiers that will see its certificates. This document
defines End Entity Name Restrictions (EENR), a new critical
extension for CA certificates that constrains the `dNSName`
Subject Alternative Names which may appear in end entity
certificates issued beneath the CA. The extension is scoped to
TLS client and TLS server certificate validation only, so that
its matching semantics can be specified completely within this
document rather than deferred to other specifications.

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# The End Entity Name Restrictions Extension

## Syntax {#syntax}

The End Entity Name Restrictions extension is identified by the
following object identifier:

   id-pe-eenr OBJECT IDENTIFIER ::= { TBD }

The extension SHALL be encoded as follows:

   EndEntityNameRestrictions ::= SEQUENCE {
       permittedSubtrees       \[0]     GeneralSubtrees OPTIONAL,
       excludedSubtrees        \[1]     GeneralSubtrees OPTIONAL }

The `GeneralSubtrees`, `GeneralSubtree`, and `BaseDistance` types
are defined in {{RFC5280}} section 4.2.1.10 and are reused here
without modification.

The `base` field of each `GeneralSubtree` MUST be of type
`dNSName`; the use of any other general-name type renders the
extension malformed. The `minimum` field of each `GeneralSubtree`
MUST be 0 and the `maximum` field MUST be absent, matching the
constraint that {{RFC5280}} places on use of `GeneralSubtree` in
the `nameConstraints` extension.

The IANA registration request for `id-pe-eenr` appears in
{{iana-considerations}}.

## Use in CA Certificates

The End Entity Name Restrictions extension MUST be used only in a
CA certificate. Conforming CAs MUST mark the extension as critical.

Both `permittedSubtrees` and `excludedSubtrees` are OPTIONAL. If
both are absent, the extension is present but conveys no
name-space restriction; the restrictions on end entity
certificates defined in {{restrictions-on-end-entity-certificates}}
still apply by virtue of the extension's presence anywhere in the
certification path. If `permittedSubtrees` is provided, it MUST
contain at least one entry; if `excludedSubtrees` is provided, it
MUST contain at least one entry.

The `dNSName` value carried in the `base` field of each
`GeneralSubtree` MUST NOT contain the DNS wildcard character `*`.

## Propagation Through CA Certificates

This extension is designed so that delegation can only narrow,
never widen, the names a CA may authorize in subsequent end entity
certificates. The propagation rules below realize that property.

The extension MAY appear in any CA certificate. Where it appears,
`permittedSubtrees` and `excludedSubtrees` accumulate down the
certification path in the manner specified in
{{matching-semantics}}.

Only the first CA certificate to bear the extension in a given
path MAY use `permittedSubtrees`. In any CA certificate whose
certification path contains an ancestor bearing the extension,
the `permittedSubtrees` field MUST be absent; subsequent CAs MAY
narrow the name space further by providing `excludedSubtrees`.

## Relationship to RFC 5280 Name Constraints {#relationship-to-rfc5280-name-constraints}

The matching semantics defined by this document for `dNSName`
entries differ from those of {{RFC5280}} section 4.2.1.10, and a
certification path that combines both would be subject to the
matching ambiguity this extension is designed to eliminate. To
avoid that, in any certification path containing the End Entity
Name Restrictions extension, a CA certificate in that path MUST
NOT include a `nameConstraints` extension whose `permittedSubtrees`
or `excludedSubtrees` contains a `dNSName` form. A verifier MUST
reject any certification path that violates this restriction.

## Restrictions on End Entity Certificates {#restrictions-on-end-entity-certificates}

When the End Entity Name Restrictions extension is present in any
CA certificate in a certification path, every end entity certificate
in that path MUST satisfy each of the following restrictions:

 * The subject distinguished name MUST NOT contain a `commonName`
   (CN) attribute.

 * The extended key usage extension MUST be present and MUST
   contain exactly one of the following key purpose identifiers,
   and no other:

     * `id-kp-serverAuth` (1.3.6.1.5.5.7.3.1) for TLS server
       authentication, or
     * `id-kp-clientAuth` (1.3.6.1.5.5.7.3.2) for TLS client
       authentication.

 * The subject alternative name extension MUST be present and MUST
   contain at least one `dNSName` entry. The number of `dNSName`
   entries SHOULD NOT exceed 16; conforming applications MAY reject
   certificates with more.

 * Each `dNSName` entry in the subject alternative name extension
   MUST conform to {{RFC9525}} section 6.3.

A verifier MUST reject any certification path containing an end
entity certificate that fails any of these restrictions.

## Matching Semantics {#matching-semantics}

For each `dNSName` entry present in the end entity certificate's
subject alternative name extension, the verifier MUST evaluate the
restrictions accumulated from all End Entity Name Restrictions
extensions in the certification path as follows:

 * If any `permittedSubtrees` was provided, the `dNSName` entry
   MUST match at least one of those permitted subtree base values.

 * For every `excludedSubtrees` entry provided by any CA in the
   path that bears the extension, the `dNSName` entry MUST NOT
   match that excluded subtree base value.

A subtree base value (which contains no wildcard) is treated as
the reference identifier, and the end entity certificate's
`dNSName` entry (which may contain a wildcard in its leftmost
label) as the presented identifier, per {{RFC9525}} section 6.3.

A verifier MUST reject any certification path for which a
`dNSName` entry fails this evaluation.

# Security Considerations

The End Entity Name Restrictions extension is a critical X.509
extension. A verifier processing a certification path containing
the extension either implements the matching semantics defined
here exactly or rejects the path; no third option exists in which
the extension is silently misinterpreted. This is what allows a
CA to rely on it as a security control restricting the `dNSName`
Subject Alternative Names in end entity certificates issued
beneath it, for purposes of TLS client and TLS server
authentication.

# IANA Considerations  {#iana-considerations}

This document defines one new object identifier for the End Entity
Name Restrictions extension. IANA is requested to assign a value
from the "SMI Security for PKIX Certificate Extension" registry
(1.3.6.1.5.5.7.1) for the OID label `id-pe-eenr`. The assigned
value is to be substituted for the placeholder in {{syntax}}.

| Decimal | Description  | References     |
|--------:|--------------|----------------|
| TBD     | id-pe-eenr   | this document  |

The reference in the table is to be replaced with the published
RFC number upon publication.


--- back

# Alternatives Considered  {#alternatives-considered}

## Amending RFC 5280 or RFC 9525 to define wildcard semantics

The wildcard ambiguity could in principle be addressed by an
update to {{RFC5280}} section 4.2.1.10's matching algorithm, or
by an update to {{RFC9525}} to extend its wildcard semantics to
name constraint matching (which {{RFC9525}} section 1.2 currently
defers to {{RFC5280}}).

Either approach fails to give a CA reliable control over how its
certificates will be evaluated. A verifier conforming to the
specification as written today is not made non-conformant by any
future update; a CA issuing a certificate cannot ensure that the
verifiers consuming it have adopted the updated semantics, and
would have no way to detect a verifier still applying the
original algorithm.

## Prohibiting wildcards in chains with excluded dNSName constraints

The security failure caused by the matching ambiguity arises only
when both wildcard `dNSName` SAN entries and `dNSName`
`excludedSubtrees` are present in the same certification path:
literal {{RFC5280}} matching can accept a wildcard SAN whose TLS
expansion covers a name the excluded subtree was intended to
prevent. An {{RFC5280}} amendment could prohibit this combination.

Such an amendment suffers the same deployment problem as defining
new semantics: a verifier that has not adopted it will not enforce
the prohibition. Certificates issued under the old understanding
remain vulnerable in proportion to the verifier population that
has not been updated.

## The new critical extension approach

This document defines a new critical extension with fully
specified semantics. The criticality of {{RFC5280}}'s
`nameConstraints` extension guarantees the extension is processed
but not what semantics are applied, and those semantics may shift
with later specifications. A new extension with both properties
ties the two together: a verifier either rejects the certificate
because the extension is critical, or applies the rules in this
document because no other rules exist.

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
