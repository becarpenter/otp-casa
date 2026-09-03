---
title: One-time Pad for Authorizing Device Identity
abbrev: One-time Pad for Device Identity
docname: draft-carpenter-anima-otp-casa-latest
submissiontype: IETF
ipr: trust200902
area: "Operations and Management"
workgroup: "Autonomic Networking Integrated Model and Approach"
kw:
 - BRSKI
 - IDevID
 - MASA
cat: std
venue:
  group: "Autonomic Networking Integrated Model and Approach"
  type: "Working Group"
  mail: "anima@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/anima/"
  github: "becarpenter/otp-casa"
  latest: "https://becarpenter.github.io/otp-casa/draft-carpenter-anima-otp-casa.html"

pi:
  toc: yes
  sortrefs:   # defaults to yes
  symrefs: yes

author:

      -
        ins: B. E. Carpenter
        name: Brian E. Carpenter
        org: The University of Auckland
        abbrev: Univ. of Auckland
        postal:
        - School of Computer Science
        - The University of Auckland
        - PB 92019
        - Auckland 1142
        country: New Zealand
        email: brian.e.carpenter@gmail.com

normative:
  RFC8990:
  RFC8995:
  RFC4086:

informative:
  RFC8993:
  RFC8994:
  RFC8366:

--- abstract

This document describes how devices joining an autonomic control plane as defined
in RFC 8994 may use the BRSKI onboarding mechanism defined in RFC 8995, even if they cannot
provide a manufacturer-installed X.509 IDevID certificate. Instead, such devices may generate
a self-signed certificate embedding a unique token selected from a one-time pad.

--- middle

# Introduction        {#intro}

The Bootstrapping Remote Secure Key Infrastructure (BRSKI) onboarding mechanism
is specified in {{RFC8995}}. It relies on two elements. The first is an X.509v3
certificate formatted as an IEEE 802.1AR IDevID,
installed in a device by its manufacturer. The second is a Manufacturer Authorized
Signing Authority (MASA), a server that can certify that an IDevID is valid.
During the operation of the BRSKI mechanism, a device attempting to join the
Autonomic Control Plane (ACP) {{RFC8994}} is known as a "pledge", and the purpose
of BRSKI is to authorize a pledge by obtaining a voucher {{RFC8366}} from the MASA.

In practice, it can happen that either the devices needing to connect do not
possess an IDevID, or that the network in question does not have access to
a suitable MASA.  This document describes a solution for this scenario, while
using much of the existing BRSKI protocol framework.

This solution could be applicable to a corporate network that does not use
manufacturer-installed IDevIDs at all. Alternatively, in a network
using BRSKI for devices with IDevIDs, the solution could be used
in a heterogeneous mode for a subset of pledges for which either an IDevID
or a MASA is unavailable. In the heterogeneous case, the normal BRSKI trust
model for the whole ACP (Section 7.1 of {{RFC8995}}) is altered as
described in {{trust}}.

# Terminology

{::boilerplate bcp14-tagged}

# Corporate Authorized Signing Authority (CASA)

This fills the role of the MASA for BRSKI purposes. It is in effect
a one-time pad.

The CASA is essentially based on a list of randomly generated
tokens. The tokens MUST be hard to guess, with a minimum size of at
least 64 bits. They SHOULD be cryptographically strong random or
pseudo-random numbers (see {{RFC4086}}, Section 6.2).

The list of tokens is referred to as the OPADL (One-time-PAD
List, pronounced Oh-Paddle). It MUST be stored on long-term, backed-up and
cryptographically secured storage.

# Authorized Installer

This is a person or agent that is trusted to authorize new devices to connect
to the network. Whenever needed, each Installer is given a new batch of random
tokens, called an APADL (Agent one-time-PAD List, pronounced "a Paddle").
These tokens are also added to the OPADL when the APADL is created. The APADL
MUST be stored on secure storage, e.g., an encrypted memory stick in the
possession of the Installer.

When the CASA creates an APADL, a record MUST be made, along with the
identity of the Installer, for audit purposes. This record  MUST be
associated with the OPADL, and stored on long-term, backed-up and
cryptographically secured storage.

If an APADL is lost or compromised, all the tokens in it MUST immediately
be marked as "claimed" in the OPADL.

# Connecting a Pledge

When an Installer authorizes a new device to connect, the following steps occur:

1. The Installer's software picks a token from the APADL.

2. This token is installed in the pledge and marked as "claimed" in the APADL.

3. The pledge then executes code to create and save a key pair and an X.509v3 certificate in IDevID format. It contains contains the token ("serial-number" in BRSKI terms) and the pledge's new public key, and is self-signed. It is referred to as an ODevID (One-time Device ID) but is in effect an LDevID.

These steps SHOULD be embedded in code stored on the Installer's
secure memory device, such that the token is never viewed by a human.

The pledge then starts the normal BRSKI process per {{RFC8995}},
using the ODevID in place of an IDevID. However, because the ODevID
is self-signed and thus has no CA issuer, the RFC8995 voucher request
is augmented by adding a `pledge-self-cert` binary element which
carries the ODevID certificate. This is used by the registrar to
verify the signed voucher request, and the registrar __SHOULD__ retain
this certificate (which includes the token, i.e. serial number).

TBD: update the YANG in RFC8995 accordingly. 

# Authorization

In practice, the CASA and the Registrar will be a single software
system, so no network protocol is needed between them.
When the Registrar receives a voucher request via EST, as per
{{RFC8995}}, it will pass the request directly to the CASA.
Instead of the checks normally carried out by a MASA, the CASA will
extract the token ("serial-number") from the pledge's ODevID,
and check if it is present and unused in the OPADL. If yes,
the CASA will mark it as "claimed" in the OPADL, and issue the
required voucher directly to the Registrar, allowing the BRSKI
process to complete. If the token is not available in the OPADL,
authorization will fail.

The action of checking and marking a token as "claimed" MUST
be an atomic operation.

Clearly, a bogus token will fail. In the highly unlikely event that
two pledges try the same token, the second Installer simply tries
again with another token from their APADL. The same would apply
if a voucher request failed in such a way that a token was marked
as "claimed" by the CASA but the voucher never reached the pledge.

# Trust Model  {#trust}

Section 7.1 of {{RFC8995}} summarizes the BRSKI trust model.
The present document removes the requirement to trust equipment
manufacturers, the integrity of their IDevID creation, and their
MASA services. It also removes any security exposures during
communication between the Registrar and the MASA.

On the other hand, it introduces a need to operate
a CASA in a completely secure manner, and a need to trust the
authorized Installers, especially their operational security
practices that keep the APADLs secure. The risk of fraudulent pledges
due to a compromised APADL is real, but can be traced after
the event using logs from the CASA. If an APADL should be
physically lost, all its tokens MUST immediately be marked as claimed
in the OPADL.

The ODevIDs are self-signed. This is acceptable because each ODevID
certificate includes a unique token from the OPADL, and so
can be trusted exactly to the extent that the Installer is trusted.
However, this means that the BRSKI-EST TLS connection cannot rely on
a CA-signed IDevID as described in Section 5.1 of {{RFC8995}}. It __SHOULD__
rely on whatever corporate or general PKI is already in place in the pledge.
In a stand-alone environment, an alternative is to accept self-signed CMS structures.

The Registrar and the CASA are trustworthy because they constitute
a corporate entity and can present an end-entity certificate satisfying corporate
security requirements.

# Implementation Status \[RFC Editor: please remove]

See [](https://github.com/becarpenter/graspy/blob/master/casa) for a proof of concept.
It's amateur code from a security point of view. DO NOT trust it in the slightest.

# Security Considerations

The security considerations of {{RFC8995}} apply in general. However, the trust model
is modified, as discussed in {{trust}}.

Also, sections 7.3 and 7.4 of {{RFC8995}} allow certain security reductions for BRSKI
registrars and MASAs. The mechanism described in the present document removes the need for
some of these reductions, since it caters for devices without manufacturer or ownership credentials.
For example, nonceless vouchers are never needed since the Registrar and the CASA are
colocated.

However, since the pledge is issued a voucher on the basis of a self-signed certificate,
there is a plausible man-in-the middle attack by a rogue BRSKI proxy, if it intercepts
a voucher request, extracts the token value, creates its own key pair, and simulates all
subsequent pledge actions. Similarly, a rogue registrar could accept any pledge without
checking that its token is known to the genuine CASA registrar. Only good operational
security can protect against such attacks.

The CASA is under local control so could safely be placed on the local side
of an air gap. In some scenarios, this may be considered a security advantage.

# IANA Considerations

No IANA actions are required by this document.

--- back

# Change Log \[RFC Editor: please remove]

## Draft-00

- Original version

## Draft-01

- Many changes after a proof-of-concept implementation

# Acknowledgements
{:numbered="false"}

Helpful comments were made by
Michael Richardson,
...
