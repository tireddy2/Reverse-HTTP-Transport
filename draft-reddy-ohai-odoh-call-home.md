---
title: "Oblivous DoH Call Home"
abbrev: "ODoH Call Home"
category: std
stream: IETF

docname: draft-reddy-ohai-odoh-call-home-latest
area: "Security"
workgroup: "Oblivious HTTP Application Intermediation"
keyword: Internet-Draft
venue:
  group: "Oblivious HTTP Application Intermediation"
  type: "Working Group"
  mail: "ohai@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/ohai/"

v: 3

author:
 -
    name: Tirumaleswar Reddy
    organization: Nokia
    email: kondtir@gmail.com

normative:

informative:


--- abstract

This document defines a mechanism that can be used by networks like
ISPs to host oblivious DoH servers without being publically accessible
but allows the clients to access the oblivious DoH servers via trusted
Oblivious Relay Resources. It protects the oblivious DoH servers from
Layer 3 and Layer 4 DDoS attacks from the Internet.


--- middle

# Introduction

Oblivious HTTP {{!OHTTP=I-D.draft-ietf-ohai-ohttp}} allows clients to encrypt
messages exchanged with an Oblivious Target Resource (target). The messages
are encapsulated in encrypted messages to an Oblivious Gateway Resource
(gateway), which gates access to the target. The gateway is accessed via an
Oblivious Relay Resource (relay), which proxies the encapsulated messages
to hide the identity of the client. Overall, this architecture is designed
in such a way that the relay cannot inspect the contents of messages, and
the gateway and target cannot discover the client's identity.

({{!OHAI-SVCB=I-D.draft-ietf-ohai-svcb-config}}) discusses discovering of
oblivious DoH servers and their assoicated gateways more dynamically.
A network might operate a DoH resolver that provides more optimized or
more relevant DNS answers and is accessible using Oblivious HTTP,
and might want to advertise support for Oblivious HTTP via mechanisms
like DHCP and Router Advertisement Options for the Discovery
of Network-designated Resolvers ({{!DNR=I-D.draft-ietf-add-ddr}}) and Discovery of
Designated Resolvers ({{!DDR=I-D.draft-ietf-add-dnr}}).

Clients can use trusted relays to access these gateways and oblivious DoH servers.
For deployments that support this kind of discovery, the gateway and target
resources need to be located on the same host. In order for DoH servers to
function as oblivious targets, their associated gateways need to be
accessible via an oblivious relay. DoH servers used with the discovery mechanisms
in {{OHAI-SVCB}} can either be publicly accessible, or specific to a network.
In general, only publicly accessible DoH servers will work as oblivious
targets, unless there is a coordinated deployment with an oblivious relay
that is also hosted within a network. If the DoH resolver is only accessible to
the devices attached to the network and is not publicly accessible, it cannot be
reached by the relay. If DoH resolver needs be publically accessible to support
Oblivious HTTP, it is a drastic change to the way it is operated is some deployments.
Most importantly, it needs to be protected from Layer 3 and Layer 4
DDoS attacks from the Internet. In some deployments, initiating a TCP connection
from the Internet to a oblivious DoH server is complicated because of the presence of
translators and firewalls.

This document defines a way to host oblivious DoH servers without
being publically accessible but allows the clients to access the
oblivious DoH servers via a trusted relay.  Most importantly, it
protects the oblivious DoH server from Layer 3 and Layer 4 DDoS
attacks from the Internet.  However, the oblivious DoH server still
needs to be protected from Layer 7 DDoS attacks from the clients
connecting via the Oblivious relay (see Section 8.3 of
({{!ORELAY-FEEDBACK=I-D.draft-rdb-ohai-feedback-to-proxy}})).

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Solution Overview {#solution}

The Oblvious DOH server initally acts as TCP/TLS client and the relay
acts as TCP/TLS server and then roles are reversed. The Oblvious DOH
server initiates a secure connection to the relay, signals itself as an
Oblivious target and triggers role reversal at the TLS layer.

The Oblivious DoH server is configured with the FQDNs of the relay. 

The Oblivious DOH server would use the Application-Layer Protocol Negotiation (ALPN) token "ODoH" ({{iana}})
in the TLS handshake to indicate to the relay that it supports this specification, and they
perform mutual TLS authentication.  After the TLS handshake completes, the roles are reversed
and the target and Oblivious DoH server exchange encapsulated DNS requests and responses.


# Oblivous DoH Call Home Procedure

The following figure illustrates a sample Oblivous DoH Call Home message flow :

~~~~~ aasvg
                                         .------------------------------.
+---------+            +----------+     |  +----------+    +----------+  |
| Client  |            | Relay    |     |  | Gateway  |    | Target   |  |
|         |            | Resource |     |  | Resource |    | Resource |  |
+----+----+            +----+-----+     |  +-----+----+    +----+-----+  |
     |                     |            `-------|--------------|-------'
     |                     |                    |              |
     |                     | Initiate TCP       |              |
     |                     | handshake          |              |
     |                     |<-------------------|              |
     |                     | Initiate TLS       |              |
     |                     | handshake          |              |
     |                     |<-------------------|              |
     |                     |                    |              |
     |                     | Certificiate       |              |
     |                     | Request            |              |
     |                     |------------------->|              |
     |                     |                    |              |
     |                     | Certificiate       |              |
     |                     |<-------------------|              |
     | .---------------.   |                    |              |
     | | Validate      |   |                    |              |
     | | certificate & |+-+|                    |              |
     | | role reversal |   |                    |              |
     | .---------------.   |                    |              |
     |                     |                    |              |
     | Relay               |                    |              |
     | Request             |                    |              |
     | [+ Encapsulated     |                    |              |
     |    Request ]        |                    |              |
     +-------------------->| Gateway            |              |
     |                     | Request            |              |
     |                     | [+ Encapsulated    |              |
     |                     |    Request ]       |              |
     |                     +------------------->| Request      |
     |                     |                    +------------->|
     |                     |                    |              |
     |                     |                    |     Response |
     |                     |            Gateway |<-------------+
     |                     |           Response |              |
     |                     |    [+ Encapsulated |              |
     |                     |         Response ] |              |
     |           Relay     |<-------------------+              |
     |        Response     |                    |              |
     | [+ Encapsulated     |                    |              |
     |      Response ]     |                    |              |
     |<--------------------+                    |              |
     |                     |                    |              |
~~~~~



The gateway would iterate through client configurations to identify pre-configured relays. The Oblivous DoH Call Home procedure is
illustrated as follows for each of the relays:

1. The gateway resource begins by initiating a TCP connection to the relay.  Once connected, the
   gateway continues to initiate a TLS connection to the relay.

2. The gateway uses ALPN token "ODoH" in the TLS handshake to indicate to the relay that it is a target and not a client.

3. The relay using the ALPN token "ODoH" would identify that the client is a gateway and sends the CertificateRequest message
   to indicate that the certificate-based client authentication is required. The gateway sends the client certificiate 
   which will be verified by the relay based on PKIX validation {{!RFC5280}}. The subjectAltName extension of type dNSName
   in the client certificate will be used to record the identity of the gateway. 

4. If the certificate validation is successful, role-reversal is triggered for the gateway to act as a server
   and the relay to act as client.

5. If any client desires to use the gateway, the relay will use the prior-established secure connection
   to forward the encapsulated requests from clients to the gateway and forwards the encapsulated responses from
   the gateway to clients. 

## TCP Hearbeat Mechanism

To accomodate loss of state in firewalls or translators especially in the absence
of application traffic, the gateway SHOULD perform periodic keepalives and MUST
re-establish a new TLS connection when application-level communication has failed.

The frequency of such keepalive and of timeouts is TBD.

# Security Considerations {#security}

The security considerations for the Oblivious HTTP protocol (Section 8 of {{OHTTP}}) as well as the security considerations for
Discovery of Oblivious Services via Service Binding Records (Section 6 of {{OHAI-SVCB}}) apply.



# IANA Considerations {#iana}

This document creates a new registration for the identification of DoQ in the "TLS Application-Layer Protocol Negotiation (ALPN) Protocol IDs" registry
{{!RFC7301}}.

The "ODoH" string identifies Oblivous DoH Call Home:

Protocol:

    ODoH

Identification Sequence:


    0x4f 0x44 0x6f 0x48 ("ODoH")

Specification:

    This document

--- back