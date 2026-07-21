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

informative:
  RFC8993:
  RFC8994:
  RFC8366:

--- abstract

This document describes how devices joining an autonomic control plane as defined
in RFC 8994 may use the bootstrapping mechanism defined in RFC 8995 even if they cannot
use a manufacturer-installed X.509 certificate. Instead, such devices may generate
a self-signed certificate embedding a unique token selected from a one-time pad.

--- middle

# Introduction        {#intro}

The Bootstrapping Remote Secure Key Infrastructure (BRSKI) mechanism is specified
in {{RFC8995}}. It relies on two elements. The first is an X.509v3 certificate
formatted as an IEEE 802.1AR IDevID,
installed in a device by its manufacturer. The second is a Manufacturer Authorized
Signing Authority (MASA), a server that can certify that an IDevID is valid.
During the operation of the BRSKI mechanism, a device attempting to join the
Autonomic Control Plane (ACP) {{RFC8994}} is known as a "pledge", and the purpose
of BRSKI is to authorize a pledge by obtaining a voucher {{RFC8366}} from the MASA.

In practice, it can happen that either the devices needing to connect do not
possess an IDevID, or that the network in question does not have access to
a suitable MASA.  This document describes a solution for this scenario, while
using the existing BRSKI protocol framework.

This solution could be applicable to a corporate network that does not use
manufacturer-installed IDevIDs at all. Alternatively, in a network
using BRKSI for devices with IDevIDs, the solution could be used
in a heterogeneous mode for a subset of pledges for which either an IDevID
or a MASA is unavailable. In the heterogeneous case, the normal BRSKI trust
model for the whole ACP (Section 7.1 of {{RFC8995}}) is altered as
described in {{trust}}.

# Terminology

{::boilerplate bcp14-tagged}

# Corporate Authorized Signing Authority (CASA)

This fills the role of the MASA for BRSKI purposes.

The CASA is initialized once and once only by creating a list of randomly
generated tokens. If the planned network size is N devices, there SHOULD be at
least 100N tokens, or at least a million tokens, whichever is greater.
This list is referred to as the OPADL (One-time-PAD List, pronounced Oh-Paddle).
It MUST be stored on long-term, backed-up and cryptographically secured storage.

The tokens MUST be hard to guess, with a minimum size of at least 64 bits.

# Authorized Installer

This is a person or agent that is trusted to authorize new devices to connect
to the network. Each Installer is given a batch of tokens randomly chosen
from the OPADL, called an APADL (Agent one-time-PAD List, pronounced "a Paddle").
It MUST be stored on secure storage, e.g., an encrypted memory stick in the possession
of the Installer.

When the CASA creates an APADL, a record of it MUST be made, along with the
identity of the Installer, for audit purposes. This record too MUST be
stored on long-term, backed-up and cryptographically secured storage.

If an APADL is lost or compromised, all the tokens in it MUST be marked as
"used" in the OPADL.

# Connecting a Pledge

When an Installer authorizes a new device to connect, the following steps occur:

1. The Installer picks a token from the APADL at random.
2. This token is installed in the pledge and marked as "used" in the APADL.
3. The pledge then executes code to create a key pair and an X.509v3 certificate
in IDevID format. It contains contains the token ("serial-number" in BRSKI terms)
and the pledge's new public key, and is self-signed. It is referred to as an ODevID
(One-time Device ID) but is in effect an LDevID.

These steps SHOULD be embedded in code stored on the Installer's
secure memory device, such that the token is never viewed by a human.

The pledge then starts the normal BRSKI process per {{RFC8995}},
using the ODevID in place of an IDevID.

# Authorization

The CASA will receive a voucher request from the BRKSI Registrar.
Instead of the checks normally carried out by a MASA, it will
extract the token ("serial-number") from the pledge's ODevID,
and check if it is present and unused in the OPADL. If yes,
the CASA will mark it as "used" in the OPADL, and issue the required voucher,
allowing the BRSKI process to complete.
If the token is not available in the OPADL, authorization will fail.

The action of checking and marking a token as "used" MUST
be an atomic operation.

Clearly, a bogus token will fail. In the unlikely event that
two agents try the same token, the second agent simply tries
again with another token from their APADL.

# Trust Model  {#trust}

Section 7.1 of {{RFC8995}} summarizes the BRSKI trust model.
The present document removes the requirement to trust equipment
manufacturers, the integrity of their IDevID creation, and their
MASA services. On the other hand, it introduces a need to operate
a CASA in a completely secure manner, and a need to trust the
authorized Installers, especially their operational security
practices that keep the APADLs secure. The risk of fraudulent pledges
due to a lost or compromised APADL is real, but can be traced after
the event using logs from the CASA.

<!-- # Implementation Status \[RFC Editor: please remove] -->


# Security Considerations

The security considerations of {{RFC8995}} apply in general. However, the trust model
is modified, as discussed in {{trust}}.

Also, sections 7.3 and 7.4 of {{RFC8995}} allow certain security reductions for BRSKI
registrars and MASAs. The mechanism described in the present document reduces the need for
such reductions, since it caters for devices without manufacturer or ownership credentials.
In addition, the CASA is under local control so could safely be placed on the local side
of an air gap. In some scenarios, this may be considered a security advantage.

More???


# IANA Considerations

No IANA actions are required by this document.

--- back

# Change Log

## Draft-00

- Original version

# Acknowledgements
{:numbered="false"}

Helpful comments were made by
Michael Richardson,
...
