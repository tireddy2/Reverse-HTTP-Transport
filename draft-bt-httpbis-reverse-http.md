---
title: "Reverse HTTP Transport"
abbrev: "Reverse HTTP"
category: std
stream: IETF

docname: draft-bt-httpbis-reverse-http-latest
area: "ART"
workgroup: "HTTPBIS"
keyword: Internet-Draft
venue:
  group: "HTTPBIS"
  type: "Working Group"
  mail: "ietf-http-wg@w3.org"
  arch: "http://lists.w3.org/Archives/Public/ietf-http-wg/"

v: 3

author:
 -
    name: Benjamin M. Schwartz
    organization: Meta Platforms, Inc.
    email: ietf@bemasc.net
 -
    name: Tirumaleswar Reddy
    organization: Nokia
    email: kondtir@gmail.com
 -
    name: Mohamed Boucadair
    organization: Orange
    email: mohamed.boucadair@orange.com

normative:

informative:


--- abstract

This document defines a secure transport for HTTP in which the client and server roles are reversed. This arrangement improves the origin server's protection from Layer 3 and Layer 4 denial-of-service attacks when an intermediary is in use. It allows origin server's to be hosted without being publicly accessible but allows the clients to access these servers via an intermediary.


--- middle

# Introduction

The HTTP protocol has long supported the ability of clients to access origins via an intermediary. There are now a variety of well-defined intermediary types:

* Client-selected
   - HTTP request proxies (a.k.a. forward proxies)
   - Transport proxies (e.g., CONNECT (see Section 9.3.6 of {{?RFC9110}}), CONNECT-UDP {{?RFC9298}})
   - IP relays (e.g., CONNECT-IP {{?I-D.ietf-masque-connect-ip}})
   - Oblivious HTTP Relays {{?I-D.ietf-ohai-ohttp}}
* Origin-selected
   - HTTP gateways (a.k.a. reverse proxies)
   - Transport load balancers

Although these intermediaries differ widely in their functionality, they all generally act as a client when connecting to the origin.  Client-selected intermediaries reach the origin based on its hostname or IP address specified in the HTTP request, while origin-selected intermediaries first translate this destination into a "backend address".

One of the main advantages of origin-selected intermediaries is their ability to defend the origin from attacks, especially Denial of Service (DoS) and Distributed Denial-of-Service (DDoS) attacks in which an attacker floods the origin server with requests/packets.  To prevent attackers from simply bypassing the intermediary, common practices include keeping the backend address hidden and/or instituting filtering rules (ACL, typically) that only allow packets from the intermediary. These practices are reasonably effective with origin-selected intermediaries, but they cannot be used with client-selected intermediaries, as those intermediaries do not know the secret backend address, and the origin does not know their "exit" IP addresses.

This specification defines a protocol for HTTP connections between origins and arbitrary intermediaries that can limit the impact of Layer 3 and Layer 4 DoS/DDoS attacks. When this protocol is in use, origins have the ability to partition the infrastructure that serves each intermediary. This ensures that attacks targeting the origin's public IP address 
or attacks via one intermediary will not affect any other intermediaries. By partitioning the infrastructure, the impact of the attacks is contained within the affected intermediary or the origin's public IP address.


This protocol works by reversing the TLS or QUIC transport roles that supports an HTTP connection: the origin acts as the transport client, and the intermediary acts as the server.  Inside the secure transport, HTTP operates normally but with the client and server roles reversed.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Protocol {#protocol}

The new protocol is termed "Reverse HTTP".  The "Reverse HTTP/2" version is identified by the ALPN ID "h2-reverse", and "Reverse HTTP/3" by "h3-reverse" (see {{iana-alpn}} for registrations).  These protocols represent HTTP/2 and HTTP/3, operating with the roles reversed but otherwise unmodified, except as follows:

* The intermediary MUST send a TLS CertificateRequest message to indicate that certificate-based client authentication is required.
* The origin MUST respond with a valid certificate chain.
* Each party MUST send a SETTINGS frame as soon as the transport becomes writable to them.  This means that the intermediary will typically send its SETTINGS frame first.
* The origin MUST immediately send an ORIGIN frame identifying the origins it claims.  This frame is used as originally defined, except that wildcard names are permitted (contrary to {{Section 2.2 of !RFC8336}}).
  - Otherwise, use of Reverse HTTP with wildcard certificates would be impossible.
* The intermediary MUST check the ORIGIN frame contents against the provided TLS client certificate.

Reverse HTTP/1.1 is not defined, as the lack of multiplexing renders it unsuitable for this use.

Once this process has completed, the origin has proved ownership of the Origin Set and is ready to receive requests.  The intermediary SHOULD direct subsequent HTTP requests for this origin over this Reverse HTTP channel unless it appears to be overloaded.

~~~~~ aasvg
                                        .------------------------------.
+---------+           +----------+     |  +----------+    +----------+  |
| Client  |           | Relay    |     |  | Gateway  |    | Target   |  |
|         |           | Resource |     |  | Resource |    | Resource |  |
+----+----+           +----+-----+     |  +-----+----+    +----+-----+  |
     |                     |            `-------|--------------|-------'
     |                     | Initiate TCP       |              |
     |                     | handshake          |              |
     |                     |<-------------------+              |
     |                     | Initiate TLS with  |              |
     |                     | ALPN="h2-reverse"  |              |
     |                     |<-------------------+              |
     |                     | CertificateRequest,|              |
     |                     | HTTP SETTINGS      |              |
     |                     +------------------->|              |
     |                     | CertificateVerify, |              |
     |                     | SETTINGS, ORIGIN   |              |
     |                     |<-------------------+              |
     |  .-------------.    |                    |              |
     | | Validate      |   |                    |              |
     | | certificate & |+-+|                    |              |
     | | role reversal |   |                    |              |
     |  `-------------'    |                    |              |
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
{: title="Example: Reverse HTTP/2 for Oblivious HTTP"}

## Use with Transport Proxies

Transport proxies do not normally act as HTTP clients, so they cannot use Reverse HTTP directly.  Instead, if a transport proxy receives a request whose destination host and port appears in the Origin Set, the proxy establishes a transport proxy connection to this origin over the Reverse HTTP connection.  For TCP, this corresponds to an HTTP CONNECT request, and for UDP it corresponds to an extended-CONNECT request with the "connect-udp" protocol, using the registered .well-known URI template.

Note that transport destinations identified by an IP address can only use this mechanism if the origin's certificate includes that IP address explicitly.

For IP relays, the destination does not include a port number.  The intermediary MUST add a port number of 443 before attempting to forward packets using this procedure.

## Discovery of Client-selected Intermediaries

The HTTP "Via" header can indicate the presence of a client-selected intermediary.  If a "Via" header arrives with an unrecognized host, the origin MAY attempt a Reverse HTTP connection for use by future requests from this intermediary.  If the intermediary does not confirm the protocol via ALPN, the origin MUST close the connection.

# Service Binding Mapping {#svcb}

Intermediaries that support Reverse HTTP SHOULD indicate this by publishing a SVCB record with port-prefix naming, using the scheme "http-reverse" with a default port of 443.  Applicable SvcParamKeys include "alpn", "ipv4hint"/"ipv6hint", "port", and "ech".  There is no default ALPN value, so the "alpn" key is REQUIRED.

~~~ Zone
proxy.example.net.               IN HTTPS 1 . alpn=h2,h3 ech=ABC..123
_http-reverse.proxy.example.net. IN SVCB  1 proxy.example.net. \
      alpn=h2-reverse,h3-reverse ech=ABC..123
~~~
{: title="Example records for a forward proxy that supports Reverse HTTP"}

# Operational Considerations

## Efficiency

Reverse HTTP does not use appreciably more bandwidth or CPU time than ordinary HTTP on active connections.  However, it is much less efficient for idle connections, which use memory and other connection-related resources on the intermediary even when no requests are being processed.

## Connection Management

To accommodate loss of state in firewalls or translators, especially in the absence
of application traffic, the origin SHOULD use appropriate transport-level keepalives and MUST
re-establish a new connection when application-level communication has failed.

Origins backed by multiple servers MAY attempt to establish a separate Reverse HTTP connection from each one in order to tolerate equipment failures and support optimized path selection.  However, intermediaries SHOULD limit the number of idle Reverse HTTP connections operated on behalf of each registrable domain, in order to avoid resource exhaustion attacks.

For very high-traffic origins and multi-instance intermediaries, a disruption could occur if the intermediary immediately directs all user traffic onto the first Reverse HTTP connection.  Very large intermediaries SHOULD ensure that transitions to Reverse HTTP are gradual, so that large origins have time to establish multiple connections.

## Non-public Origins

In some cases, an HTTP origin may be intended exclusively for use via one or more client-selected intermediaries that are known to the origin.  In this situation, the publication of DNS records for the origin is OPTIONAL.

## Other Applications

Reverse HTTP is principally intended for use between intermediaries and origins.  It is not applicable to general HTTP clients, as it requires the origin to know that the client will issue a request before the request occurs.  However, in cases where the HTTP client is publicly reachable and produces frequent requests to one origin over a long period of time, Reverse HTTP may be applicable.

# Security Considerations {#security}

## Stolen Key Attacks {#stolen-keys}

As noted in {{Section 4 of !RFC8336}}, accepting ORIGIN frames without DNS confirmation facilitates the use of stolen keys, and thus increases the incentive to steal these keys.  The mitigations listed in that section also apply here, and are RECOMMENDED.  Intermediaries also MAY impose a DNS confirmation requirement, although this weakens the DoS/DDoS defense provided by Reverse HTTP ({{ip-leaks}}).

> QUESTION: Should we do more about this?  For example, we could define an OID to mark these certificates as Reverse HTTP-only, or we could commit to an IP range by placing a MAC in a DNS record and revealing the message via a SETTINGS value.

## IP Address Leaks {#ip-leaks}

One goal of Reverse HTTP is to prevent DoS/DDoS attackers from learning the IP addresses used by the origin to communicate with this intermediary.  This IP address can be leaked in various ways, requiring certain mitigations:

* In the ordinary DNS address records for the origin.
  - Origins SHOULD use different IPs for Reverse HTTP (unless the intermediary imposes a DNS confirmation requirement as described in {{stolen-keys}}).
* In the `Proxy-Status` HTTP header field's "next-hop" attribute.
  - Intermediaries MUST NOT populate the "next-hop" attribute when using Reverse HTTP to the origin.
* From the probes sent to intermediaries discovered from the "Via" header field.
  - Origins SHOULD use distinct, unrelated IP addresses to contact each intermediary.
* From connection monitoring of the ALPN values in ClientHellos.
  - Intermediaries MAY use a single Encrypted ClientHello configuration for HTTP and Reverse HTTP.
* From statistical analysis of traffic patterns.
  - Origins SHOULD regularly change the IP address that is used.

Even if the origin's Reverse HTTP IP addresses do leak, Reverse HTTP still provides significant protection by simplifying the firewall rules required to block unsolicited connections.

## Key Consistency with Oblivious HTTP

The security considerations for the Oblivious HTTP protocol (Section 8 of {{?OHTTP=I-D.draft-ietf-ohai-ohttp}}) as well as the security considerations for
Discovery of Oblivious Services via Service Binding Records (Section 6 of {{?OHAI-SVCB=I-D.draft-ietf-ohai-svcb-config}}) apply. {{?CONSISTENCY=I-D.ietf-privacypass-key-consistency}}
provides an analysis of the options for ensuring the key configurations are consistent between different clients. Clients MUST employ
some technique to mitigate key targeting attacks. Note that the option of confirming the key with a shared proxy as described in {{CONSISTENCY}}
will work if the Oblivious HTTP relay and the forward proxy operate from a single origin.


# IANA Considerations {#iana}

## ALPN IDs {#iana-alpn}

This document rquests IANA to add two new registrations in the "TLS Application-Layer Protocol Negotiation (ALPN) Protocol IDs" registry
{{!RFC7301}}:

| Protocol       | Identification Sequence                                          | Specification   |
|----------------|------------------------------------------------------------------|-----------------|
| Reverse HTTP/2 | 0x68 0x32 0x2d 0x72 0x65 0x76 0x65 0x72 0x73 0x65 ("h2-reverse") | (This document) |
| Reverse HTTP/3 | 0x68 0x33 0x2d 0x72 0x65 0x76 0x65 0x72 0x73 0x65 ("h3-reverse") | (This document) |

## New Scheme

Per {{?Attrleaf=RFC8552}}, IANA is requested to add the following entry to the DNS Underscore Global Scoped Entry Registry:

| RR TYPE | _NODE NAME    | Reference                 |
| ------- | ------------- | ------------------------- |
| SVCB    | _http-reverse | (This document, {{svcb}}) |

> TODO: Register the URI scheme as well.

--- back
