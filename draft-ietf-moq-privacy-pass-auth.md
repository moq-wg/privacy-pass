---
title: "Privacy Pass Authentication for Media over QUIC (MoQ)"
abbrev: "Privacy Pass MoQ Auth"
category: std

docname: draft-ietf-moq-privacy-pass-auth-latest
submissiontype: IETF
ipr: trust200902
v: 3
area: "Web and Internet Transport"
workgroup: "Media Over QUIC"
keyword:
- media over quic
- privacy-pass
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  home: https://datatracker.ietf.org/wg/moq/
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  repo: https://github.com/moq-wg/privacy-pass
  latest: https://moq-wg.github.io/privacy-pass/draft-ietf-moq-privacy-pass-auth.html

author:
 -
    name: "Suhas Nandakumar"
    organization: "Cisco"
    email: "snandaku@cisco.com"
 -
    name: "Cullen Jennings"
    organization: "Cisco"
    email: "fluffy@iii.ca"
 -
    name: "Thibault Meunier"
    organization: "Cloudflare Inc."
    email: "ot-ietf@thibault.uk"



normative:
  ARC: I-D.draft-ietf-privacypass-arc-crypto
  MoQ-TRANSPORT: I-D.draft-ietf-moq-transport
  PRIVACYPASS-ARC: I-D.draft-ietf-privacypass-arc-protocol
  RFC2119:
  RFC9474:
  RFC9497:
  RFC9576:
  RFC9577:
  RFC9578:

informative:
  RFC9458:
  PRIVACYPASS-BATCHED: I-D.draft-ietf-privacypass-batched-tokens
  PRIVACYPASS-IANA:
    title: Privacy Pass IANA
    target: https://www.iana.org/assignments/privacy-pass/privacy-pass.xhtml
  PRIVACYPASS-REVERSE-FLOW: I-D.draft-meunier-privacypass-reverse-flow

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

- **Client**: The MoQ client requesting authorization to subscribe to, fetch,
or publish media content. The client is responsible for obtaining Privacy Pass
tokens through the attestation and issuance process, and presenting these
tokens when requesting MoQ operations such as SUBSCRIBE, FETCH, PUBLISH, or
ANNOUNCE.

- **MoQ Relay**: The MoQ relay server that forwards media content and verifies
that clients are authorized. The relay validates Privacy Pass tokens presented
by clients, enforces access policies, and forwards authorized requests to other
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

## Joint Attester and Issuer {#joint-issuer-attester}

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

In certain deployments the MoQ relay and Privacy Pass issuer may be
operated by the same entity to simplify key management and policy coordination.
This is the Privacy Pass deployment described in {{Section 4.2 of RFC9576}}.

## Shared Origin, Attester, Issuer with a Reverse Flow {#reverse-flow}

The flow described above can be used to bootstrap a shared origin-attester-issuer flow,
as described in {{Section 4.2 of RFC9576}}. The MoQ relay plays all roles (origin,
attester, and issuer), allowing it to use privately verifiable token types registered
in {{PRIVACYPASS-IANA}}.

In this scenario, the MoQ relay origin would accept tokens signed by two issuers:

1. Type `0x0002` token signed by the bootstrap issuer from {{joint-issuer-attester}}
2. Type `0x0001`, `0x0005`, or `0xE5AC` tokens signed by its own issuer.

This two-phase approach provides several advantages:

- **Bootstrapping**: The initial publicly verifiable token (`0x0002`) establishes
  trust without requiring the relay to share private keys with external verifiers.
- **Efficiency**: Subsequent privately verifiable tokens allow batched issuance,
  amortizing cryptographic costs across multiple operations.
- **Privacy**: Each token presentation is unlinkable, even when obtained from
  the same credential.

### Reverse Flow Overview

The reverse flow, as described in {{Section 6.3 of PRIVACYPASS-REVERSE-FLOW}},
allows a client to exchange a publicly verifiable token for privately verifiable
tokens (or credentials) issued directly by the MoQ relay.

~~~aasvg
+---------------------------------------.                      +--------------------------.
|  +---------------+ +---------------+   |      +--------+     |  +----------+ +--------+  |
|  | Origin Issuer | |   MoQ Relay   |   |      | Client |     |  | Attester | | Issuer |  |
|  +-------+-------+ +-------+-------+   |      +----+---+     |  +-----+----+ +----+---+  |
 `---------|-----------------|----------'            |          `-------|----------|------'
           |                 |                       |                  |          |
           |                 |   Phase 1: Bootstrap Token Acquisition  |          |
           |                 |                       |                  |          |
           |                 |<-- CLIENT_SETUP[] ---+                  |          |
           |                 +-- UNAUTHORIZED ------>|                  |          |
           |                 |   [TokenChallenge]    |                  |          |
           |                 |                       |<== Attestation ==>          |
           |                 |                       +------- TokenRequest ------->|
           |                 |                       |<------ TokenResponse -------+
           |                 |                       |                  |          |
           |                 |   Phase 2: Token Exchange via Reverse Flow         |
           |                 |                       |                  |          |
           |                 |<-- CLIENT_SETUP ------+                  |          |
           |                 |    [Token +           |                  |          |
           |                 |     CredentialRequest]|                  |          |
           |<-CredentialReq--+                       |                  |          |
           +--CredentialRes->|                       |                  |          |
           |                 +--- SERVER_SETUP ----->|                  |          |
           |                 |    [CredentialResponse]                  |          |
           |                 |                       |                  |          |
           |                 |   Phase 3: Normal Operations with Derived Tokens   |
           |                 |                       |                  |          |
           |                 |<-- SUBSCRIBE ---------+                  |          |
           |                 |    [Token from        |                  |          |
           |                 |     credential]       |                  |          |
           |                 +--- SUBSCRIBE_OK ----->|                  |          |
           |                 |                       |                  |          |
~~~
{: #fig-reverse-flow title="Complete Reverse Flow Authorization"}

### Detailed Reverse Flow Steps

**Phase 1: Bootstrap Token Acquisition**

1. The client initiates a connection with `CLIENT_SETUP` without authorization.
2. The MoQ relay responds with `UNAUTHORIZED` containing a `TokenChallenge`
   specifying a publicly verifiable token type (`0x0002`).
3. The client performs attestation with an external attester/issuer.
4. The client obtains a publicly verifiable token.

**Phase 2: Token Exchange via Reverse Flow**

5. The client sends `CLIENT_SETUP` with:
   - The publicly verifiable `Token` from Phase 1
   - A `CredentialRequest` (or `TokenRequest`) for a privately verifiable
     token type (`0x0001`, `0x0005`, or `0xE5AC`)

6. The MoQ relay validates the bootstrap token, then processes the credential
   request using its internal issuer.

7. The MoQ relay responds with `SERVER_SETUP` containing:
   - A `CredentialResponse` (or `TokenResponse`) with the privately verifiable
     credential/tokens

**Phase 3: Normal Operations**

8. For subsequent operations (SUBSCRIBE, PUBLISH, FETCH), the client presents
   tokens derived from the credential obtained in Phase 2.

9. The MoQ relay validates tokens locally using its private verification key.

### Credential Request/Response Encoding

When using the reverse flow, the `GenericBatchTokenRequest` in `ClientPrivateTokenAuth`
contains the credential or token request for the privately verifiable token type:

- For `0x0001` or `0x0005`: `TokenRequest` as defined in {{Section 5.1 of PRIVACYPASS-BATCHED}}
- For `0xE5AC`: `CredentialRequest` as defined in {{Section 7.1 of PRIVACYPASS-ARC}}

Similarly, `GenericBatchTokenResponse` in `ServerPrivateTokenAuth` contains:

- For `0x0001` or `0x0005`: `TokenResponse` as defined in {{Section 5.2 of PRIVACYPASS-BATCHED}}
- For `0xE5AC`: `CredentialResponse` as defined in {{Section 7.2 of PRIVACYPASS-ARC}}

## Trust Model

The architecture assumes the following trust relationships based on
{{Section 3 of RFC9576}}:

*  Relays trust issuers to properly validate client eligibility
before issuing tokens

*  Issuers trust attesters to accurately verify client eligibility


# Privacy Pass Token Integration

This section describes how Privacy Pass tokens are integrated into the MoQ
transport protocol to provide privacy-preserving authorization for various
media operations.

## Token Types for MoQ Authorization {#moq-token-types}

This specification uses the below existing Privacy Pass token types:

**Publicly verifiable token types**

- `0x0002 (Blind RSA (2048-bit))`: Defined in {{Section 6 of RFC9578}}. Uses
blind RSA signatures ({{RFC9474}}) for deployments requiring distributed validation
across multiple relays.

**Privately verifiable token types**

- `0x0001 (VOPRF(P-384, SHA-384))`: Defined in {{Section 6 of RFC9578}}. Uses VOPRF ({{RFC9497}}) for
deployments where the origin is the issuer. Issuance can be batched as defined in {{Section 5 of PRIVACYPASS-BATCHED}}.
- `0x0005 (VOPRF(ristretto255, SHA-512))`: Defined in {{Section 8.1 of PRIVACYPASS-BATCHED}}. Uses VOPRF ({{RFC9497}}) for
deployments where the origin is the issuer. Issuance can be batched as defined in {{Section 5 of PRIVACYPASS-BATCHED}}.
- `0xE5AC (ARC(P-256))`: Anonymous Rate Limit Credentials Token using {{ARC}}.
Tokens are presented by clients based on an issued credential and up to a `presentation_limit`.

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
{{Section 2.1 of RFC9577}} with MoQ-specific parameters in the origin_info field,
reproduced thereafter for convenience:

~~~
struct {
    uint16_t token_type;
    opaque issuer_name<1..2^16-1>;
    opaque redemption_context<0..32>;
    opaque origin_info<0..2^16-1>;
} TokenChallenge;
~~~

For MoQ usage, the `origin_info` field contains MoQ-specific authorization scope
information encoded as a UTF-8 string with the following format:

> TODO: Define origin_info to be binary format using TLS presentation language
> For anonymous credentials, it would make a lot more sense to use redemption_context
> instead of origin_info, as redemption_context is decided upon at presentation time
> rather than at issuance time.
> Question that I don't have the answer yet: are 32-bytes enough? we might need to say
> something like "first 2 bytes represent the type of rule, then data"

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

   - For Token Type `0x0001 (VOPRF(P-384, SHA-384))` and `0x0005 (VOPRF(ristretto255, SHA-512))`: Use the issuer's private validation key
   - For Token Type `0x0002 (Blind RSA(2048 bits))`: Use the issuer's public verification key

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

   - `SUBSCRIBE` operations require "subscribe" scope
   - `FETCH` operations require "fetch" scope
   - `PUBLISH` operations require "publish" scope
   - `ANNOUNCE` operations require "announce" scope

6.  Apply namespace/name matching rules based on the pattern type:

    - If Exact Match, the requested namespace/name MUST exactly equal the pattern
    - If Prefix Match, the requested namespace/name MUST start with the pattern prefix

Access is granded to the requested resource if and only if ALL of the following
conditions are met:

   - Token signature verification succeeds
   - Token nonce has not been previously seen (replay protection)
   - Token has not expired (if applicable)
   - Requested operation matches token operation scope
   - Requested namespace matches token namespace pattern
   - Requested track name matches token track name pattern (if specified)

else, authorization error is returned to the requesting client.

## Token in MOQ Messages

Privacy Pass tokens are provided to MoQ relays using the existing MoQ
authorization framework with the following adaptations:

### SETUP Message Authorization

For connection-level authorization, Privacy Pass tokens are included in the
SETUP message's authorization parameter ({{Section 9.3.1.5 of MoQ-TRANSPORT}}).

~~~
SETUP {
    Version = 1,
    Parameters = [
        {
            Type = AUTHORIZATION,
            Value = PrivateTokenAuth
        }
    ]
}

type PrivateTokenAuth = ClientPrivateTokenAuth | ServerPrivateTokenAuth;
~~~

For `CLIENT_SETUP`, the authorization value uses `GenericBatchTokenRequest`
as defined in {{Section 6.1 of PRIVACYPASS-BATCHED}} as follows:

~~~
struct {
    uint8_t auth_scheme = 0x01;
    Token token;
    GenericBatchTokenRequest token_requests;
} ClientPrivateTokenAuth;
~~~

For `SERVER_SETUP`, the authorization value uses `GenericBatchTokenResponse`
as defined in {{Section 6.2 of PRIVACYPASS-BATCHED}} as follows:

~~~
struct {
    uint8_t auth_scheme = 0x01;
    Token token;
    GenericBatchTokenResponse token_responses;
} ServerPrivateTokenAuth;
~~~

When batch issuance is not used, `token_requests` and `token_responses`
are empty (length = 0).

The `Token` structure is prepended by a two-byte token type identifier
as registered with IANA:

~~~
struct {
   uint16_t token_type; /* From the IANA Privacy Pass Token Types Registry */
   select (token_type) { /* Rest of the token */
     case (0x0001, 0x0002, 0x0005):
         uint8_t nonce[32];
         uint8_t challenge_digest[32];
         uint8_t token_key_id[Nid];
         uint8_t authenticator[Nk];
     case (other): /* Other token types from the IANA Privacy Pass Token Types Registry */
         opaque remainder<0..2^16-1>;
   }
} Token;
~~~

Where `Nk` is determined by token_type per the {{PRIVACYPASS-IANA}}.

Unknown token types MUST be rejected.

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
            Value = PrivateTokenAuth
        }
    ]
}
~~~

### Errors {#errors}

If the authentication fails for any reason, the server MUST send an error.
The error response includes a `TokenChallenge` to enable the client to obtain
a valid token and retry the operation.

#### SETUP Errors

If authentication fails during SETUP, the Relay MUST terminate the connection
with the `UNAUTHORIZED` (0x010C) Termination Error Code defined in
{{Section 3.4 of MoQ-TRANSPORT}}. The termination reason phrase MUST contain
a `MoQAuthChallenge` structure:

~~~
struct {
    TokenChallenge challenge;
    uint16_t supported_token_types<2..2^16-1>;
} MoQAuthChallenge;
~~~

The `supported_token_types` field lists token types the relay accepts, ordered
by preference (most preferred first). This allows clients to select an appropriate
issuance protocol.

#### Operation Errors

If authentication fails for an operation on an established connection (e.g.,
SUBSCRIBE, FETCH, PUBLISH), the Relay MUST send a `REQUEST_ERROR` message as
defined in {{Section 9.8 of MoQ-TRANSPORT}}.

The error code MUST be one of:

| Error Code | Name | Description |
|------------|------|-------------|
| 0x0100 | TOKEN_MISSING | No token provided when required |
| 0x0101 | TOKEN_INVALID | Token signature verification failed |
| 0x0102 | TOKEN_EXPIRED | Token has expired or been revoked |
| 0x0103 | TOKEN_REPLAYED | Token nonce has been seen before |
| 0x0104 | SCOPE_MISMATCH | Token scope does not authorize this operation |
| 0x0105 | ISSUER_UNKNOWN | Token issuer is not trusted by this relay |
{: #error-codes-table title="Privacy Pass Authorization Error Codes"}

The reason phrase in `REQUEST_ERROR` MUST contain a `MoQAuthChallenge` structure
when the client should retry with a new token.

#### TokenChallenge Construction

The `TokenChallenge` in `MoQAuthChallenge` MUST be constructed as follows:

- `token_type`: The preferred token type from `supported_token_types`
- `issuer_name`: The issuer name as configured by the relay
- `redemption_context`: A fresh 32-byte random value, or empty if the relay
  accepts tokens with any redemption context
- `origin_info`: The relay's origin identifier, optionally including the
  required authorization scope

When `origin_info` is empty, the relay accepts tokens with any scope and
performs authorization based solely on the token's embedded scope information.

#### Error Response Example

~~~
REQUEST_ERROR {
    Request_ID = 42,
    Error_Code = 0x0104,  /* SCOPE_MISMATCH */
    Reason = MoQAuthChallenge {
        challenge = TokenChallenge {
            token_type = 0x0002,
            issuer_name = "issuer.example.com",
            redemption_context = <32 random bytes>,
            origin_info = <authorization scope>
        },
        supported_token_types = [0x0002, 0xE5AC, 0x0001]
    }
}
~~~

#### Control Message Authorization Failures {#control-message-authz}

When authorization fails for MoQ control messages other than SETUP, the relay
returns a `REQUEST_ERROR` with the appropriate error code from {{error-codes-table}}.
The client MAY retry the operation with a valid token obtained using the
`TokenChallenge` from the error response.

As per {{Section 3.4.4 of MoQ-TRANSPORT}}, implementations MAY elevate
request-specific errors to session-level errors. This elevation is appropriate
when:

- The authorization failure indicates a systemic issue (e.g., all client tokens
  are from an untrusted issuer)
- Continuing the session would be futile due to policy restrictions
- The error represents a security concern requiring session termination

Implementations need to consider the impact on other outstanding subscriptions
before elevating to session-level errors.

# Example Authorization Flow

Below shows an example deployment scenario where the relay has been
configured with the necessary validation keys and content policies. The
relay can verify Privacy Pass tokens locally and deliver media directly
without contacting the Issuer. This example uses publicly verifiable tokens.

~~~~~aasvg
         +-----------+                        +--------+         +----------+ +--------+
         | MoQ Relay |                        | Client |         | Attester | | Issuer |
         +-----+-----+                        +---+----+         +----+-----+ +---+----+
               |                                  |                   |           |
               |<--------------- CLIENT_SETUP[] --+                   |           |
               |   UNAUTHORIZED (0x010C) [        |                   |           |
               +--   Reason=MoQAuthChallenge ---->|                   |           |
               |   ]                              |                   |           |
               |                                  |                   |           |
               |                                  |<== Attestation ==>|           |
               |                                  |                   |           |
               |                                  +--------- TokenRequest ------->|
               |                                  |<-------- TokenResponse -------+
               |                                  |                   |           |
               |                           FinalizeToken              |           |
               |                                  |                   |           |
               |            CLIENT_SETUP[{        |                   |           |
               |<----------    AUTHORIZATION,   --+                   |           |
               |               PrivateTokenAuth,  |                   |           |
               |            }]                    |                   |           |
               |                                  |                   |           |
 .-------------+--.                               |                   |           |
| Local validation |                              |                   |           |
 `-------------+--'                               |                   |           |
               |                                  |                   |           |
               +-- SERVER_SETUP[{  -------------->|                   |           |
               |     AUTHORIZATION,               |                   |           |
               |     PrivateTokenAuth,            |                   |           |
               |   }]                             |                   |           |
               |                           FinalizeToken              |           |
               |                                  |                   |           |
~~~~~
{: #direct-relay-authorization-flow title="Direct Relay Authorization Flow"}

The `MoQAuthChallenge` in the `UNAUTHORIZED` response contains:

- A `TokenChallenge` with the relay's issuer configuration
- A list of `supported_token_types` (e.g., `[0x0002, 0xE5AC]`)

This allows the client to select the appropriate issuance protocol based on
its capabilities and the available attesters/issuers.

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

## MoQ Privacy Pass Error Code Registry

IANA is requested to create a new registry titled "MoQ Privacy Pass Authorization
Error Codes" with the following initial contents:

| Value | Name | Reference |
|-------|------|-----------|
| 0x0100 | TOKEN_MISSING | {{errors}} |
| 0x0101 | TOKEN_INVALID | {{errors}} |
| 0x0102 | TOKEN_EXPIRED | {{errors}} |
| 0x0103 | TOKEN_REPLAYED | {{errors}} |
| 0x0104 | SCOPE_MISMATCH | {{errors}} |
| 0x0105 | ISSUER_UNKNOWN | {{errors}} |
| 0x0106-0x01FF | Unassigned | |
{: #error-code-registry title="MoQ Privacy Pass Error Codes Registry"}

New entries in this registry require Specification Required registration policy.
Values are allocated from the 0x0100-0x01FF range reserved for Privacy Pass
authorization errors.


--- back

# Acknowledgments

TODO acknowledge.

# Change Log

RFC Editor's Note: Please remove this section prior to publication of
a final version of this document.

## Since draft-ietf-moq-privacy-pass-auth-02

* Expanded reverse flow documentation with three-phase flow (bootstrap, exchange, operations)
* Defined MoQAuthChallenge structure for error responses with supported_token_types
* Added TokenChallenge construction requirements
* Added control message authorization failure handling section
* Documented credential request/response encoding for different token types

## Since draft-ietf-moq-privacy-pass-auth-01

* Define error handling
* Integrate privacy pass reverse flow within PrivateTokenAuth
* MoQ definition now follow draft-ietf-moq-transport-16
* Update dependencies
* Removed b64 encoding given MoQ can use bytes directly


## Since draft-ietf-moq-privacy-pass-auth-00

* Add Thibault Meunier as Coauthor
* Add support for Reverse flow to be deploy and scale friendly way to get tokens
