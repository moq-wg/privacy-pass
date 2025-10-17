---
title: "Privacy Pass Authentication for Media over QUIC (MoQ)"
abbrev: "Privacy Pass MoQ Auth"
category: std

docname: draft-jennings-moq-privacy-pass-auth-latest
submissiontype: IETF
ipr: trust200902
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Media Over QUIC"
keyword:
- media over quic
venue:
  group: MoQ
  type: Working Group
  home: https://datatracker.ietf.org/wg/moq/
  mail: moq@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/moq/
  github: suhasHere/moq-privacy-pass
  latest: https://suhasHere.github.io/moq-privacy-pass/draft-jennings-moq-privacy-pass-auth.html

author:
 -
    name: "Suhas Nandakumar"
    organization: "Cisco"
    email: "snandaku@cisco.com"
 -
    name: "Cullen Jennings"
    organization: "Cisco"
    email: "fluffy@iii.ca"


normative:
  MoQ-TRANSPORT: I-D.draft-ietf-moq-transport
  RFC2119:
  RFC9576:
  RFC9577:
  RFC9578:

informative:
  RFC9458:
  PRIVACYPASS-IANA:
    title: Privacy Pass IANA
    target: https://www.iana.org/assignments/privacy-pass/privacy-pass.xhtml

--- abstract

This document specifies the use of Privacy Pass architecture and issuance
protocols for authorization in Media over QUIC (MoQ) transport protocol. It
defines how Privacy Pass tokens can be integrated with MoQ's authorization
framework to provide privacy-preserving authentication for subscriptions,
fetches, publications, and relay operations while supporting fine-grained
access control through prefix-based track namespace and track name matching
rules.


--- middle

# Introduction

Media over QUIC (MoQ) {{MoQ-TRANSPORT}} provides a transport protocol for live
and on-demand media delivery, real-time communication, and interactive content
distribution over QUIC connections. The protocol supports a wide range of
applications including video streaming, video conferencing, gaming, interactive
broadcasts, and other latency-sensitive use cases. MoQ includes mechanisms for
authorization through tokens that can be used to control access to media
streams, interactive sessions, and relay operations.

Traditional authorization mechanisms often lack the privacy protection needed
for modern media distribution scenarios, where users' viewing patterns and
content preferences should remain private while still enabling fine-grained
access control, namespace restrictions, and operational constraints.

Privacy Pass {{RFC9576}} provides a privacy-preserving authorization
architecture that enables anonymous authentication through unlinkable tokens.
The Privacy Pass architecture consists of four entities: Client, Origin, Issuer,
and Attester, which work together to provide token-based authorization without
compromising user privacy. The issuance protocols {{RFC9578}} define how these
tokens are created and verified.

This document defines how Privacy Pass tokens can be integrated with MoQ's
authorization framework to provide comprehensive access control for media
streaming, real-time communication, and interactive content services while
preserving user privacy through unlinkable authentication tokens.

## Requirements Language

{::boilerplate bcp14-tagged}

# Privacy Pass Architecture for MoQ

Privacy Pass Terminology defined in {{Section 2 of RFC9576}} is reused here.
The Privacy Pass MoQ integration involves the following entities and their
interactions:

- **Client**: The MoQ client requesting access to media content. The client is
responsible for obtaining Privacy Pass tokens through the attestation and
issuance process, and presenting these tokens when requesting MoQ operations.

- **MoQ Relay/Origin**: The MoQ relay server or origin that forwards media
content and requires authorization. The relay validates Privacy Pass tokens,
enforces access policies, and forwards authorized requests to origins or other
relays. Relays maintain configuration for trusted issuers and validate token
signatures and metadata.

- **Privacy Pass Issuer**: The entity that issues Privacy Pass tokens to clients
after successful attestation. The issuer operates the token issuance protocol,
manages cryptographic keys. The issuer creates tokens with appropriate
MoQ-specific metadata.

- **Privacy Pass Attester**: The entity that attests to properties of clients
for the purposes of token issuance. The attester verifies client credentials,
subscription status, or other eligibility criteria. Common attestation methods
include username/password, OAuth, device certificates, or other authentication
mechanisms.

In the below deployment, the MoQ relay and Privacy Pass issuer are operated
by different entities to enhance privacy through separation of concerns.
This corresponds to {{Section 4.4 of RFC9576}}.

~~~aasvg
+-----------+            +--------+         +----------+ +--------+
| MoQ Relay |            | Client |         | Attester | | Issuer |
+-----+-----+            +---+----+         +----+-----+ +---+----+
      |                      |                   |           |
      |<------ Request ------+                   |           |
      +--- TokenChallenge -->|                   |           |
      |                      |<== Attestation ==>|           |
      |                      |                   |           |
      |                      +--------- TokenRequest ------->|
      |                      |<-------- TokenResponse -------+
      |<--- Request+Token ---+                   |           |
      |                      |                   |           |
~~~
{: #fig-overview title="Separated Issuer and Relay Architecture"}

However, in certain deployments the MoQ relay and Privacy Pass issuer may be
operated by the same entity to simplify key management and policy coordination.
This is the Privacy Pass deployment described in {{Section 4.2 of RFC9576}}.

## Trust Model

The architecture assumes the following trust relationships based on
{{Section 3 of RFC9576}}:

*  Issuer and attesters do not collude to guarantee
unlinkability properties

*  Relays trust issuers to properly validate client eligibility
before issuing tokens

*  Issuers trust attesters to accurately verify client eligibility


# Privacy Pass Token Integration

This section describes how Privacy Pass tokens are integrated into the MoQ
transport protocol to provide privacy-preserving authorization for various
media operations.

## Token Types for MoQ Authorization

This specification uses the existing Privacy Pass token types defined in
{{RFC9578}}:

- **Token Type 0x0001 (VOPRF(P-384, SHA-384))**: Privately verifiable tokens
using Verifiable Oblivious Pseudorandom Function for deployments requiring
issuer-only validation capability.

- **Token Type 0x0002 (Blind RSA (2048-bit))**: Publicly verifiable tokens
using blind RSA signatures for deployments requiring distributed validation
across multiple relays.

## Token Structure

Privacy Pass tokens used in MoQ MUST follow the structure defined in
{{Section 2.2 of RFC9577}} for the PrivateToken HTTP authentication scheme. The token
structure includes:

- **Token Type**: 2-byte identifier specifying the issuance protocol used
- **Nonce**: 32-byte client-generated random value for uniqueness
- **Challenge Digest**: 32-byte SHA-256 hash of the TokenChallenge
- **Token Key ID**: Variable-length identifier for the issuer's public key
- **Authenticator**: Variable-length cryptographic proof bound to the token

### Token Challenge Structure for MoQ

MoQ-specific TokenChallenge structures use the default format defined in
{{Section 2.1 of RFC9577}} with MoQ-specific parameters in the origin_info field:

~~~
struct {
    uint16_t token_type;
    opaque issuer_name<1..2^16-1>;
    opaque redemption_context<0..32>;
    opaque origin_info<0..2^16-1>;
} TokenChallenge;
~~~

For MoQ usage, the origin_info field contains MoQ-specific authorization scope
information encoded as a UTF-8 string with the following format:

TODO: Degfine origin_info to be binary format

~~~~
moq-scope = operation ":" namespace-pattern [":" track-pattern]
operation = "subscribe" / "fetch" / "publish" / "announce"
namespace-pattern = exact-match / prefix-match
track-pattern = exact-match / prefix-match
exact-match = namespace/name-string
prefix-match = namespace/name-string"*"
~~~~

Examples:

- `subscribe:sports.example.com/live/*` - Subscribe to any track under live
  sports
- `fetch:vod.example.com/movies/action*` - Fetch video-on-demand action content
- `publish:meetings.example.com/meeting/m123/audio/opus48000` - Publish content
  for meeting m123

## Track Namespace and Track Name Matching Rules

This specification defines prefix-based matching rules for track namespaces
and track names to enable fine-grained access control while maintaining
privacy.

### Namespace Matching

Track namespace matching supports three modes:

**Exact Match**:

- Pattern: `"example.com/live/sports/soccer"`

- Matches: Only the exact namespace `example.com/live/sports/soccer`

**Prefix Match**:

- Pattern: `"example.com/live/sports/*"`

- Matches: Any namespace starting with `example.com/live/sports/`

- Examples: `example.com/live/sports/soccer`,
  `example.com/live/sports/tennis`

### Track Name Matching

Track name matching within authorized namespaces follows the same pattern:

**Exact Match**:

- Pattern: `"video"`

- Matches: Only tracks named exactly `video`

**Prefix Match**:

- Pattern: `"video*"`

- Matches: Any track name starting with `video`

- Examples: `video-med`, `video-high`, `video-low`

### Matching Algorithm

When a MoQ relay receives a request with a Privacy Pass token, it performs the
following validation steps to determine whether to authorize the requested
operation:

1. Extract the Privacy Pass token from the MoQ control
message (SETUP, SUBSCRIBE, FETCH, PUBLISH, or ANNOUNCE)

2. Verify the token signature using the appropriate
issuer public key based on the token type:

   - For Token Type 0x0001 (VOPRF(P-384, SHA-384)): Use the issuer's private validation key
   - For Token Type 0x0002 (Blind RSA(2048 bits)): Use the issuer's public verification key

3. Validate that the token has not been replayed by checking:

   - Token nonce uniqueness within the issuer's replay window
   - Token expiration timestamp (if present in token metadata)

4. Extract the MoQ-specific authorization scope from the token's origin_info
field:

   - Authorized operation type (subscribe, fetch, publish, announce)
   - Namespace pattern (exact match or prefix match)
   - Track name pattern (exact match or prefix match, optional)

5. Verify that the requested MoQ operation matches the operation specified
in the token scope:

   - SUBSCRIBE operations require "subscribe" scope
   - FETCH operations require "fetch" scope
   - PUBLISH operations require "publish" scope
   - ANNOUNCE operations require "announce" scope

6.  Apply namespace/name matching rules based on the pattern type:

    - If Exact Match, the requested namespace/name MUST exactly equal the pattern
    - If Prefix Match, the requested namespace/name MUST start with the pattern prefix

Access is granded to the requested resource if and only if ALL of the following
conditions are met:

   - Token signature verification succeeds
   - Token nonce has not been previously used (replay protection)
   - Token has not expired (if applicable)
   - Requested operation matches token operation scope
   - Requested namespace matches token namespace pattern
   - Requested track name matches token track name pattern (if specified)
     specified)

else, authorzation error is returned to the requesting client.

## Token in MOQ Messages

Privacy Pass tokens are provided to MoQ relays using the existing MoQ
authorization framework with the following adaptations:

### SETUP Message Authorization

For connection-level authorization, Privacy Pass tokens are included in the
SETUP message's authorization parameter:

~~~
SETUP {
    Version = 1,
    Parameters = [
        {
            Type = AUTHORIZATION,
            Value = base64url(PrivateTokenAuth)
        }
    ]
}

struct {
    uint8_t auth_scheme = 0x01; // Privacy Pass
    opaque token_data<1..2^16-1>;
} PrivateTokenAuth;
~~~

### MoQ Operation-Level Authorization

For individual MoQ operation authorization, tokens are included in
operation-specific control messages:

~~~
SUBSCRIBE {
    Track_Namespace = "sports.example.com/live/soccer",
    Track_Name = "video",
    Parameters = [
        {
            Type = AUTHORIZATION,
            Value = base64url(PrivateTokenAuth)
        }
    ]
}
~~~

# Example Authorization Flow

Below shows an example deployment scenarios where the relay has been
configured with the necessary validation keys and content policies, the
relay can verify Privacy Pass tokens locally and deliver media directly
without contacting the Issuer. This uses token with public verifiability.

~~~~~aasvg
+-----------+                +--------+         +----------+ +--------+
| MoQ Relay |                | Client |         | Attester | | Issuer |
+-----+-----+                +---+----+         +----+-----+ +---+----+
      |                          |                   |           |
      |<- Request Service Access +                   |           |
      +--- TokenChallenge ------>|                   |           |
      |                          |<== Attestation ==>|           |
      |                          |                   |           |
      |                          +--------- TokenRequest ------->|
      |                          |<-------- TokenResponse -------+
      |                          |                   |           |
      |<---- SUBSCRIBE+Token ----+                   |           |
 .----+-----------.              |                   |           |
| Local validation |             |                   |           |
 `----+-----------'              |                   |           |
      +-- SUBSCRIBE_OK+Media --->|                   |           |
      |                          |                   |           |
~~~~~
{: #direct-relay-authorization-flow title="Direct Relay Authorization Flow"}


# Security Considerations

TODO: Add considerations for the security and privacy of the Privacy Pass
tokens.

* Token Replay
* Token harvest
* Key rotation
* Use of TLS

# IANA Considerations

TODO

* Register namespace?
* New registry for auth_scheme with 0x01 as the first registered auth_scheme


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
