---
title: "Secondary Certificate Authentication of HTTP Clients"
abbrev: "Secondary Clients Certificates"
category: std

docname: draft-rosomakho-httpbis-secondary-client-certs-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: ""
workgroup: "HTTP"
keyword:
 - exported authenticators
 - masque
venue:
  group: "HTTP"
  type: ""
  mail: "ietf-http-wg@w3.org"
  arch: "https://lists.w3.org/Archives/Public/ietf-http-wg/"
  github: "yaroslavros/httpbis-secondary-client-certs"
  latest: "https://yaroslavros.github.io/httpbis-secondary-client-certs/draft-rosomakho-httpbis-secondary-client-certs.html"

author:
 -
    fullname: Yaroslav Rosomakho
    organization: Zscaler
    email: yrosomakho@zscaler.com

 -
    fullname: Jonathan Hoyland
    organization: Cloudflare
    email: jonathan.hoyland@gmail.com

normative:
  H2:
    =: RFC9113
    display: HTTP/2
  H3:
    =: RFC9114
    display: HTTP/3

informative:


--- abstract

This document defines a mechanism for HTTP/2 and HTTP/3 clients to provide additional certificate-based credentials after the TLS handshake has completed, using TLS Exported Authenticators. Unlike traditional client authentication during the TLS handshake, this mechanism allows clients to present multiple certificates over the lifetime of a session.

--- middle

# Introduction

{{!TLS=RFC8446}} provides an option for client authentication during the initial handshake, but this approach has several limitations. It requires early commitment to a single identity and can expose identifying information prematurely.

This document defines a protocol extension that allows {{H2}} and {{H3}} clients to provide certificate-based credentials after the TLS handshake, using TLS Exported Authenticators {{!EXPORTED-AUTH=RFC9261}}. This enables clients to authenticate in a flexible and dynamic manner, for example when accessing specific resources, elevating privileges, or switching identities.

An important use case for this mechanism is the ability for a client to provide multiple identities or contextual information distributed across several certificates, such as separate device and user credentials.

This mechanism builds on and complements the secondary certificate support for HTTP servers defined in {{!SECONDARY-SERVER=I-D.ietf-httpbis-secondary-server-certs}}.

## Authentication Initiation

Client authentication using this mechanism can be initiated by either the client or the server:

- A client-initiated authentication begins with the client sending an `REQUEST_CLIENT_AUTH` frame. This signals its intent to provide certificate-based credentials and requests one or more challenge contexts from the server.
- A server-initiated authentication begins with the server sending an `AUTHENTICATOR_REQUESTS` frame unsolicited. This allows the server to request client credentials proactively.

In both cases, the server provides a set of authenticator request contexts using the `AUTHENTICATOR_REQUESTS` frame. The client replies with one or more `CERTIFICATE` frames, each containing an Exported Authenticator constructed using one of the received contexts.

## Certificate Validation and Authorization

This protocol does not define certificate validation or access control policies. The interpretation and acceptance of client-provided certificates are delegated entirely to the application.

Servers are expected to evaluate the received credentials according to their policy and decide whether to:

- Accept the certificate and associate it with the client session or with a specific set of future HTTP requests;
- Ignore the certificate and proceed without adjusting the client's authorization state;
- Terminate the connection if the certificate is unacceptable or appears malicious.

This mechanism does not define any explicit protocol-level signaling for certificate rejection. Applications may implement their own logic to determine how authentication success or failure affects access to resources, but such decisions are made outside the scope of this protocol extension.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Negotiating Support for HTTP-layer Client Certificate Authentication

Support for HTTP-layer client certificate authentication is negotiated via a `SETTINGS` parameter. An endpoint MUST NOT send frames related to client authentication (`REQUEST_CLIENT_AUTH`, `AUTHENTICATOR_REQUESTS`, or client-initiated `CERTIFICATE`) unless the peer has explicitly advertised support via the `SETTINGS` parameter defined in {{settings-http-client-cert-auth}}.

This restriction does not apply to server-initiated `CERTIFICATE` frames, which are governed by {{SECONDARY-SERVER}}.

## SETTINGS_HTTP_CLIENT_CERT_AUTH {#settings-http-client-cert-auth}

A new `SETTINGS` parameter is defined for both HTTP/2 and HTTP/3 to indicate support for HTTP-layer client certificate authentication. The value of this parameter MUST be either 0 or 1.

Each endpoint must advertise its own support. A client MUST NOT send `REQUEST_CLIENT_AUTH` or `CERTIFICATE` frames unless both parties have advertised support for this mechanism. Similarly, a server MUST NOT send an `AUTHENTICATOR_REQUESTS` frame unless both parties have advertised support.

Endpoints MUST NOT send a `SETTINGS_HTTP_CLIENT_CERT_AUTH` parameter with a value of 0 after previously sending a value of 1. Once support for this extension has been advertised, it is considered enabled for the lifetime of the connection.

### HTTP/2 Definition

In HTTP/2, this parameter is defined as an entry in the `SETTINGS` frame as specified in {{Section 6.5.2 of H2}}.

- Identifier: `SETTINGS_HTTP_CLIENT_CERT_AUTH` (0xTBD)
- Value: 0 indicates that the sender does not support this extension; 1 indicates support for receiving client certificate authentication frames.

### HTTP/3 Definition

In HTTP/3, this parameter is defined as a `SETTINGS` parameter in accordance with {{Section 7.2.4.1 of H3}}.

- Identifier: `SETTINGS_HTTP_CLIENT_CERT_AUTH` (0xTBD)
- Value: 0 indicates no support; 1 indicates support.

# Client Request for Server Authenticators

When a client wishes to initiate HTTP-layer certificate authentication, it begins by sending an `REQUEST_CLIENT_AUTH` frame. This frame signals the client's intent to provide one or more certificates and asks the server to generate a corresponding number of authenticator request contexts, which are returned in an `AUTHENTICATOR_REQUESTS` frame.

A client MUST NOT send an `REQUEST_CLIENT_AUTH` frame unless both endpoints have negotiated support for the HTTP-layer client certificate authentication mechanism via the `SETTINGS_HTTP_CLIENT_CERT_AUTH` parameter as described in {{settings-http-client-cert-auth}}.

If a server receives an `REQUEST_CLIENT_AUTH` frame and has not negotiated this extension, it MUST treat the event as a connection error. In HTTP/3, the connection MUST be closed with an `H3_FRAME_UNEXPECTED` error according to {{Section 8 of H3}}. In HTTP/2, the connection MUST be closed with a `PROTOCOL_ERROR` according to {{Section 5.4.1 of H2}}.

## REQUEST_CLIENT_AUTH Frame

The `REQUEST_CLIENT_AUTH` frame is used by a client to request one or more authenticator request contexts from the server. This frame is only valid if both endpoints have negotiated support for HTTP-layer client certificate authentication.

This frame is used exclusively by clients and is considered a control frame. Its format and transport differ depending on the HTTP version in use.

### HTTP/2 Definition

The `REQUEST_CLIENT_AUTH` frame is sent on stream 0.

~~~~~~~~~~ ascii-art
REQUEST_CLIENT_AUTH Frame {
  Length (24),
  Type (8) = 0xTBD,
  Unused Flags (8),

  Reserved (1),
  Stream Identifier (31) = 0,

  Authenticator Count (i),
}
~~~~~~~~~~
{: title="HTTP/2 REQUEST_CLIENT_AUTH Frame"}

The `Authenticator Count` field is a variable-length integer that specifies how many authenticator request contexts the client is prepared to respond to. The value of this field MUST be greater than zero. Variable-length integers are encoded as described in {{Section 16 of ?QUIC=RFC9000}}.

If the server receives an `REQUEST_CLIENT_AUTH` frame with a Authenticator Count set to zero, it MUST treat it as a connection error of type `PROTOCOL_ERROR` according to {{Section 5.4.1 of H2}}.

If this frame is received on a stream other than stream 0, the recipient MUST treat it as a connection error of type `PROTOCOL_ERROR`.

### HTTP/3 Definition

The `REQUEST_CLIENT_AUTH` frame is sent on the control stream.

~~~~~~~~~~ ascii-art
REQUEST_CLIENT_AUTH Frame {
  Length (i),
  Type (i) = 0xTBD,

  Authenticator Count (i),
}
~~~~~~~~~~
{: title="HTTP/3 REQUEST_CLIENT_AUTH Frame"}

The Authenticator Count field is a variable-length integer that specifies how many authenticator request contexts the client is prepared to respond to. The value of this field MUST be greater than zero. Variable-length integers are encoded as described in {{Section 16 of QUIC}}.

If the server receives an `REQUEST_CLIENT_AUTH` frame with a Authenticator Count set to zero, it MUST treat it as a connection error of type `H3_MESSAGE_ERROR` according to {{Section 8 of H3}}.

If this frame is received on a stream other than the control stream, the recipient MUST treat it as a connection error of type `H3_FRAME_UNEXPECTED`.

## Server Behavior and Request Handling

Upon receiving an `REQUEST_CLIENT_AUTH` frame, the server MUST respond with an `AUTHENTICATOR_REQUESTS` frame (as defined in {{authenticator-requests}}). The `AUTHENTICATOR_REQUESTS` frame carries a list of authenticator requests that the client is expected to satisfy. This list MAY be empty.

If the server is not prepared to generate any authenticator requests, or does not wish to accept client certificate authentication at this time, it MUST respond with an `AUTHENTICATOR_REQUESTS` frame that contains an empty list of authenticators.

The server MAY include fewer authenticator requests than the number requested by the client. This allows the server to limit the state it must retain while waiting for the client to respond with corresponding `CERTIFICATE` frames.

A client MUST NOT send a new `REQUEST_CLIENT_AUTH` frame until all of the following conditions are met:

- It has received an `AUTHENTICATOR_REQUESTS` frame from the server in response to the previous request;
- It has sent a `CERTIFICATE` frame for each authenticator included in that response.

If the server receives an `REQUEST_CLIENT_AUTH` frame that violates these conditions, such as being sent before the previous exchange has completed, it MUST treat this as a connection error. In HTTP/2, the server MUST terminate the connection with a `PROTOCOL_ERROR` according to {{Section 5.4.1 of H2}}. In HTTP/3, it MUST terminate the connection with H3_FRAME_UNEXPECTED according to {{Section 8 of H3}}.

# Authentication Procedure

The authentication procedure begins when the server sends an `AUTHENTICATOR_REQUESTS` frame to the client. This may occur in response to a `REQUEST_CLIENT_AUTH` frame previously sent by the client, or independently, if the server wishes to initiate client authentication based on its own policy.

The `AUTHENTICATOR_REQUESTS` frame contains a list of authenticator requests, as defined in {{Section 4 of EXPORTED-AUTH}}. Each request presents a cryptographic challenge for the client to use in the construction of a corresponding Exported Authenticator. The list of authenticator requests MAY be empty, indicating that the server is not currently accepting client authentication but acknowledges the `REQUEST_CLIENT_AUTH` request.

The client responds by sending a `CERTIFICATE` frame for each authenticator request provided. Each `CERTIFICATE` frame carries an Exported Authenticator that includes a certificate chain and proof of possession, as specified in {{Section 5 of EXPORTED-AUTH}}. A client may send an empty authenticator as defined in {{Section 6 of EXPORTED-AUTH}} in cases where it declines to authenticate for a given request.

This section defines the structure of the `AUTHENTICATOR_REQUESTS` frame and outlines how it and the `CERTIFICATE` frames, as defined in {{SECONDARY-SERVER}}, are exchanged, ordered, and validated during the authentication process.

## AUTHENTICATOR_REQUESTS Frame {#authenticator-requests}

The `AUTHENTICATOR_REQUESTS` frame is sent by the server to deliver authenticator requests to the client. It may be used in response to a `REQUEST_CLIENT_AUTH` frame or sent independently. The list of authenticator requests MAY be empty, in which case the frame serves to acknowledge the client's request without initiating authentication.

Each authenticator request is used by the client to construct an Exported Authenticator as defined in {{Section 5 of EXPORTED-AUTH}}. This frame is sent only after support for HTTP-layer client certificate authentication has been negotiated via `SETTINGS_HTTP_CLIENT_CERT_AUTH` as described in {{settings-http-client-cert-auth}}. It is a control frame and MUST be sent on stream 0 in HTTP/2 or the control stream in HTTP/3.

### HTTP/2 Definition

~~~~~~~~~~ ascii-art
AUTHENTICATOR_REQUESTS Frame {
  Length (24),
  Type (8) = 0xTBD,
  Unused Flags (8),

  Reserved (1),
  Stream Identifier (31) = 0,

  Authenticator Request (..) ...,
}
~~~~~~~~~~
{: title="HTTP/2 AUTHENTICATOR_REQUESTS Frame"}

If this frame is received by a server on any stream, or by a client on a stream other than stream 0, the recipient MUST treat it as a connection error of type `PROTOCOL_ERROR` according to {{Section 5.4.1 of H2}}.

If the frame payload is malformed or contains malformed authenticator requests (e.g., incomplete length-prefixes or invalid encoding), the recipient MUST treat it as a connection error of type `PROTOCOL_ERROR`.

### HTTP/3 Definition

~~~~~~~~~~ ascii-art
AUTHENTICATOR_REQUESTS Frame {
  Type (i) = 0xTBD,
  Length (i),

  Authenticator Request (..) ...,
}
~~~~~~~~~~
{: title="HTTP/3 AUTHENTICATOR_REQUESTS Frame"}

If this frame is received by a server on any stream, or by a client on a stream other than the control stream, the recipient MUST treat it as a connection error of type `H3_FRAME_UNEXPECTED` according to {{Section 8 of H3}}.

If the frame payload is malformed or contains malformed authenticator requests, the recipient MUST treat it as a connection error of type `H3_MESSAGE_ERROR`.

### Authenticator Request Format

Each Authenticator Request is an encoded structure consisting of a variable-length integer length and the corresponding binary value of the request:

~~~~~~~~~~ ascii-art
Authenticator Request {
  Length (i),
  ClientCertificateRequest (..),
}
~~~~~~~~~~
{: title="Authenticator Request structure"}

- Length (i): A variable-length integer specifying the length, in bytes, of the ClientCertificateRequest.
- ClientCertificateRequest: A structure as defined in {{Section 4 of EXPORTED-AUTH}}.

The payload of the `AUTHENTICATOR_REQUESTS` frame consists of a sequence of these Authenticator Request elements. The list MAY be empty. Clients parse the payload by reading the length, then interpreting the following bytes as a ClientCertificateRequest structure.

### Request Ordering and Pacing

Clients MUST respond to each authenticator request context with a corresponding `CERTIFICATE` frame, in the same order as the authenticator requests appeared in the `AUTHENTICATOR_REQUESTS` frame.

A client MUST NOT accept a new `AUTHENTICATOR_REQUESTS` frame while a previous one is still outstanding. Specifically, if the client has received an `AUTHENTICATOR_REQUESTS` frame and has not yet sent a `CERTIFICATE` frame for each contained request, it MUST treat any additional `AUTHENTICATOR_REQUESTS` frame as a connection error.

In HTTP/2, the connection MUST be closed with a PROTOCOL_ERROR according to {{Section 5.4.1 of H2}}. In HTTP/3, the connection MUST be closed with `H3_FRAME_UNEXPECTED` according to {{Section 8 of H3}}.

## CERTIFICATE Frame

The `CERTIFICATE` frame is used by the client to respond to each authenticator request previously delivered by the server in an `AUTHENTICATOR_REQUESTS` frame. Each `CERTIFICATE` frame carries a single Exported Authenticator structure constructed using the TLS Exported Authenticator mechanism defined in {{Section 5.2 of EXPORTED-AUTH}}.

The format and semantics of the `CERTIFICATE` frame are defined in {{SECONDARY-SERVER}}, and are reused without modification for client authentication.

Clients MUST send exactly one `CERTIFICATE` frame for each authenticator request context received. These frames MUST be sent in the same order as the corresponding authenticator requests appeared in the `AUTHENTICATOR_REQUESTS` frame.

A `CERTIFICATE` frame MAY carry an empty authenticator, as described in {{Section 6 of EXPORTED-AUTH}}. This allows the client to decline authentication for a particular request context without terminating the connection.

Clients MUST NOT send `CERTIFICATE` frames unless they are in response to an outstanding `AUTHENTICATOR_REQUESTS` frame. If a server receives a `CERTIFICATE` frame when no authenticator request is outstanding, it MUST treat this as a connection error.

Servers MUST NOT send `CERTIFICATE` frames unless the secondary certificate authentication mechanism for HTTP servers has been negotiated according to {{SECONDARY-SERVER}}. If a client receives a server-generated `CERTIFICATE` frame without such negotiation, it MUST treat this as a connection error of type `PROTOCOL_ERROR` (HTTP/2) or `H3_FRAME_UNEXPECTED` (HTTP/3).

### Certificate Validation and Authorization

The validation of client-provided certificates and any associated authorization logic are outside the scope of this specification. The server is responsible for interpreting the `CERTIFICATE` frames it receives and deciding how they impact the session state or access control.

A server MAY associate an accepted certificate with the connection, use it for access control decisions on subsequent requests, or ignore it entirely. The application determines whether authentication is required, optional, or sufficient for specific operations.

If a server receives a certificate that it considers invalid, untrusted, or inappropriate, it MAY respond by limiting the client’s access, rejecting individual requests, or closing the connection. This specification does not define protocol-level error signaling for certificate rejection.

# Security Considerations

This mechanism builds on TLS Exported Authenticators {{EXPORTED-AUTH}}, which provide cryptographic binding to the underlying TLS connection and strong guarantees of authenticity, integrity, and freshness. However, several considerations remain relevant for implementers and deployers.

## Impersonation and Trust

Servers MUST carefully validate client-provided certificates and ensure that they are trusted, appropriate for the context, and bound to the intended identity. Failure to enforce proper certificate validation could allow an attacker to impersonate another client or gain unauthorized access.

Authorization decisions based on certificates are application-specific and must be handled with care. A certificate's presence does not imply entitlement.

## Replay Prevention

Exported Authenticators are constructed using unique per-session cryptographic context and include binding to the TLS connection via the handshake transcript. This provides replay protection across different sessions, provided the ClientCertificateRequest includes a fresh context value. Reuse of identical request contexts across connections SHOULD be avoided.

## Exposure of Client Identity

Certificates often contain long-lived identifiers (such as names, device serial numbers, or email addresses). When a client presents such certificates, it exposes identity information to the server. Clients should avoid presenting identifying certificates unless necessary for the resource or context being accessed.

If client authentication is not required, clients may prefer to respond with an empty Exported Authenticator.

## Binding to the Connection

Exported Authenticators are cryptographically bound to the TLS session in which they are sent. A certificate authenticated using this mechanism applies to the entire HTTP connection and all streams within it. Servers MUST NOT assume that certificate-based identity can vary between streams on the same connection.

Applications MUST ensure that any authenticated identity is scoped to the connection lifetime, and not reused across connections unless explicitly permitted by policy.

## Denial-of-Service Risk

The process of generating authenticator requests, tracking their state, and verifying the resulting authenticators can be computationally expensive for the server. A malicious or misbehaving client could send repeated or excessive `REQUEST_CLIENT_AUTH` frames to force the server to consume memory, perform cryptographic operations, or hold open authentication state.

Servers SHOULD apply limits on the number of authenticator requests they are willing to issue in response to a single request. Servers MAY respond with fewer `AUTHENTICATOR_REQUESTS` than requested or with an empty list. Implementations SHOULD bound memory and compute usage per connection to mitigate such attacks.

## Timing Side Channels

Applications should consider the potential for timing side channels in certificate validation and authenticator generation. Variations in processing time may reveal information about internal policies, trust anchors, or user behavior. Constant-time validation and randomized delays are potential mitigations in sensitive deployments.

# IANA Considerations

This document registers the `REQUEST_CLIENT_AUTH` and `AUTHENTICATOR_REQUESTS` frame types and `SETTINGS_HTTP_CLIENT_CERT_AUTH` for both {{H2}} and {{H3}}.

## HTTP/2 Frame Types

This specification registers the following entries in the "HTTP/2 Frame Types" registry defined in {{H2}}:

`REQUEST_CLIENT_AUTH` frame:

- Code: 0xTBD
- Frame Type: REQUEST_CLIENT_AUTH
- Reference: This document

`AUTHENTICATOR_REQUESTS` frame:

- Code: 0xTBD
- Frame Type: AUTHENTICATOR_REQUESTS
- Reference: This document

## HTTP/3 Frame Types

This specification registers the following entries in the "HTTP/3 Frame Types" registry defined in {{H3}}:

`REQUEST_CLIENT_AUTH` frame:

- Value: 0xTBD
- Frame Type: REQUEST_CLIENT_AUTH
- Status: provisional (permanent if this document is approved)
- Reference: This document
- Change Controller: Yaroslav Rosomakho (IETF if this document is approved)
- Contact: yrosomakho@zscaler.com (HTTP_WG; HTTP working group; ietf-http-wg@w3.org if this document is approved)
- Notes: None

`AUTHENTICATOR_REQUESTS` frame:

- Value: 0xTBD
- Frame Type: AUTHENTICATOR_REQUESTS
- Status: provisional (permanent if this document is approved)
- Reference: This document
- Change Controller: Yaroslav Rosomakho (IETF if this document is approved)
- Contact: yrosomakho@zscaler.com (HTTP_WG; HTTP working group; ietf-http-wg@w3.org if this document is approved)
- Notes: None

## HTTP/2 Setting

This specification registers the following entry in the "HTTP/2 Settings" registry defined in {{H2}}:

- Code: 0xTBD
- Name: SETTINGS_HTTP_CLIENT_CERT_AUTH
- Initial Value: 0
- Reference: This document

## HTTP/3 Setting

This specification registers the following entry in the "HTTP/3 Settings" registry defined in {{H3}}:

- Code: 0xTBD
- Setting Name: SETTINGS_HTTP_CLIENT_CERT_AUTH
- Default: 0
- Status: provisional (permanent if this document is approved)
- Reference: This document
- Change Controller: Yaroslav Rosomakho (IETF if this document is approved)
- Contact: yrosomakho@zscaler.com (HTTP_WG; HTTP working group; ietf-http-wg@w3.org if this document is approved)
- Notes: None

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
