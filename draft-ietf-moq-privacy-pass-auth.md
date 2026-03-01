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

## Shared Origin, Attester, Issuer with a Reverse Flow

The flow described above can be used to bootstrap a shared origin-attester-issuer flow,
as described in {{Section 4.2 of RFC9576}}. The MoQ relay plays all role, allowing it
to use privately verifiable token types registered in {{PRIVACYPASS-IANA}}.

In this scenario, the MoQ relay origin would accept tokens signed by two issuers:

1. Type `0x0002` token signed by the bootstrap issuer from {{joint-issuer-attester}}
2. Type `0x0001`, `0x0005`, or `0xE5AC` tokens signed by its own issuer.

With {{PRIVACYPASS-ARC}}, the flow would look as follows

~~~aasvg
+----------------------------------.                          +--------------------------.
|  +---------------+ +-----------+  |         +--------+      |  +----------+ +--------+  |
|  | Origin Issuer | | MoQ Relay |  |         | Client |      |  | Attester | | Issuer |  |
|  +---+-----------+ +-----+-----+  |         +---+----+      |  +----+-----+ +---+----+  |
 `-----|-------------------|-------'              |            `------|-----------|------'
       |                   |                      |<-------- TokenResponse -------+
       |                   |<---- Request    -----+                   |           |
       |                   |     +Token           |                   |           |
       |                   |     +TokenRequestO   |                   |           |
       |<- TokenRequestO --+                      |                   |           |
       +- TokenResponseO ->|                      |                   |           |
       |                   +----- Response     -->|                   |           |
       |                   |     +TokenResponseO  |                   |           |
       |                   |                      |                   |           |
~~~

`TokenRequestO` and `TokenResponseO` are part of a reverse flow as described in {{Section 6.3 of PRIVACYPASS-REVERSE-FLOW}}. The client request a
new token/credential to the origin. It allows the client to exchange
its initial 0x0002 `Token` against a privately verifiable token
issued by the origin.

`TokenRequestO` should correspond to the associated privately verifiable token
definition. These are listed in {{moq-token-types}}.

All privately verifiable scheme allow to amortise token issuance cost, making them
more compelling in a streaming case. This is specified in {{Section 5 of PRIVACYPASS-BATCHED}}.

When using `0xE5AC`, `TokenRequestO` is a `CredentialRequest` defined in {{Section 7.1 of PRIVACYPASS-ARC}}.

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

For MoQ usage, authorization scope information can be encoded in either the
`origin_info` field (bound at issuance time) or the `redemption_context` field
(bound at presentation time), depending on the deployment model.

### MoQ Actions {#moq-actions}

MoQ operations are identified by the following action values, aligned with
MoQTransport control message types:

| Action | Value | Reference |
|--------|-------|-----------|
| CLIENT_SETUP | 0 | {{Section 9.3 of MoQ-TRANSPORT}} |
| SERVER_SETUP | 1 | {{Section 9.3 of MoQ-TRANSPORT}} |
| PUBLISH_NAMESPACE | 2 | {{Section 9.20 of MoQ-TRANSPORT}} |
| SUBSCRIBE_NAMESPACE | 3 | {{Section 9.25 of MoQ-TRANSPORT}} |
| SUBSCRIBE | 4 | {{Section 9.9 of MoQ-TRANSPORT}} |
| REQUEST_UPDATE | 5 | {{Section 9.11 of MoQ-TRANSPORT}} |
| PUBLISH | 6 | {{Section 9.13 of MoQ-TRANSPORT}} |
| FETCH | 7 | {{Section 9.16 of MoQ-TRANSPORT}} |
| TRACK_STATUS | 8 | {{Section 9.19 of MoQ-TRANSPORT}} |
{: #moq-actions-table title="MoQ Action Values"}

The default authorization policy is "blocked" - all actions are denied unless
explicitly permitted by a token scope.

### Match Types {#match-types}

Match rules for namespaces and track names support the following types:

| Match Type | Value | Description |
|------------|-------|-------------|
| MATCH_EXACT | 0 | Value must equal the pattern exactly |
| MATCH_PREFIX | 1 | Value must start with the pattern |
| MATCH_SUFFIX | 2 | Value must end with the pattern |
| MATCH_CONTAINS | 3 | Value must contain the pattern as substring |
{: #match-types-table title="Match Type Values"}

Track namespaces in MoQ are represented as ordered tuples of byte strings
(e.g., `["example.com", "live", "sports"]`). Match rules operate on these
tuples at tuple element boundaries. The pattern in a MatchRule is also a
tuple of byte strings, and matching is performed element-by-element.

No normalization is performed on namespace tuple elements or track name values
before matching. Comparisons are performed as byte-level operations on each
tuple element.

### Authorization Scope Structure (origin_info) {#scope-origin-info}

When authorization scope is bound at issuance time, the `origin_info` field
contains a binary-encoded `MoQAuthorizationInfo` structure:

~~~
enum {
    CLIENT_SETUP(0),
    SERVER_SETUP(1),
    PUBLISH_NAMESPACE(2),
    SUBSCRIBE_NAMESPACE(3),
    SUBSCRIBE(4),
    REQUEST_UPDATE(5),
    PUBLISH(6),
    FETCH(7),
    TRACK_STATUS(8),
    (255)
} MoQAction;

enum {
    MATCH_EXACT(0),
    MATCH_PREFIX(1),
    MATCH_SUFFIX(2),
    MATCH_CONTAINS(3),
    (255)
} MatchType;

struct {
    opaque element<0..2^16-1>;
} TupleElement;

struct {
    TupleElement elements<0..2^16-1>;
} NamespaceTuple;

struct {
    MatchType match_type;
    NamespaceTuple value;
} NamespaceMatchRule;

struct {
    MatchType match_type;
    opaque value<0..2^16-1>;
} TrackNameMatchRule;

struct {
    MoQAction actions<1..2^8-1>;
    NamespaceMatchRule namespace_match;
    TrackNameMatchRule track_name_match;
} MoQAuthScope;

struct {
    MoQAuthScope scopes<1..2^16-1>;
} MoQAuthorizationInfo;
~~~

A token MAY contain multiple `MoQAuthScope` entries to authorize different
combinations of actions and resource patterns. Authorization succeeds if
ANY scope in the token permits the requested operation.

### Compact Authorization Scope (redemption_context) {#scope-redemption-context}

For anonymous credentials where authorization scope should be bound at
presentation time rather than issuance time, the `redemption_context` field
(limited to 32 bytes) uses a compact encoding:

~~~
struct {
    uint16_t actions_bitmask;
    uint8_t namespace_match_type;
    uint8_t track_match_type;
    opaque namespace_pattern<0..13>;
    opaque track_pattern<0..13>;
} CompactMoQScope;
~~~

The `actions_bitmask` field encodes permitted actions as a bitmask where bit N
corresponds to action value N from {{moq-actions}}. For example, a bitmask of
`0x00D0` (binary `0000000011010000`) permits SUBSCRIBE (4), PUBLISH (6), and
FETCH (7).

When patterns exceed 13 bytes, implementations SHOULD use a truncated hash:

~~~
struct {
    uint16_t actions_bitmask;
    uint8_t namespace_match_type;
    uint8_t track_match_type;
    opaque namespace_hash[14];
    opaque track_hash[14];
} CompactMoQScopeHashed;
~~~

The hash values are the first 14 bytes of SHA-256 applied to the full pattern.
The relay MUST maintain a mapping of hash values to full patterns for
verification.

### Examples

The following examples illustrate authorization scope configurations:

Subscribe to live sports namespace (prefix match):

~~~
MoQAuthScope {
    actions = [SUBSCRIBE(4)],
    namespace_match = {
        match_type = MATCH_PREFIX(1),
        value = ["sports.example.com", "live"]
    },
    track_name_match = {
        match_type = MATCH_PREFIX(1),
        value = ""
    }
}
~~~

This matches namespace tuples like `["sports.example.com", "live", "soccer"]`
and `["sports.example.com", "live", "tennis", "finals"]`.

Publish to specific meeting track (exact match):

~~~
MoQAuthScope {
    actions = [PUBLISH(6)],
    namespace_match = {
        match_type = MATCH_EXACT(0),
        value = ["meetings.example.com", "meeting", "m123"]
    },
    track_name_match = {
        match_type = MATCH_PREFIX(1),
        value = "audio-"
    }
}
~~~

This matches only the exact namespace tuple `["meetings.example.com", "meeting", "m123"]`
with track names starting with "audio-".

Fetch video-on-demand with suffix matching:

~~~
MoQAuthScope {
    actions = [FETCH(7)],
    namespace_match = {
        match_type = MATCH_CONTAINS(3),
        value = ["vod", "movies"]
    },
    track_name_match = {
        match_type = MATCH_SUFFIX(2),
        value = ".mp4"
    }
}
~~~

This matches namespace tuples containing the contiguous subsequence
`["vod", "movies"]`, such as `["example.com", "vod", "movies", "action"]`.

## Track Namespace and Track Name Matching Rules

This specification defines matching rules for track namespaces and track names
to enable fine-grained access control while maintaining privacy. Both namespace
and track name matching use the same `MatchRule` structure and algorithm.

### Match Rule Evaluation

Given a `MatchRule` and a target value (namespace tuple or track name), the
match succeeds according to the following rules. For namespace matching, both
the pattern and target are tuples of byte strings; matching operates at tuple
element boundaries.

MATCH_EXACT (0):

The target MUST be identical to the pattern. For namespace
tuples, this means the same number of elements with each element byte-for-byte
identical. The pattern tuple `["example.com", "live"]` matches only
`["example.com", "live"]`, not `["example.com", "live", "sports"]`.

MATCH_PREFIX (1):

The target MUST start with the pattern at tuple element
boundaries. The pattern tuple `["example.com", "live"]` matches
`["example.com", "live", "sports"]` and `["example.com", "live", "news", "breaking"]`
but not `["example.com", "vod"]`. Note that `["example.com", "liv"]` does NOT
match `["example.com", "live"]` since matching is at element boundaries.

MATCH_SUFFIX (2):

The target MUST end with the pattern at tuple element
boundaries. The pattern tuple `["audio"]` matches `["meeting123", "audio"]` and
`["conference", "room1", "audio"]` but not `["audio", "opus"]`.

MATCH_CONTAINS (3):

The target MUST contain the pattern as a contiguous
subsequence of tuple elements. The pattern tuple `["live", "sports"]` matches
`["example.com", "live", "sports", "soccer"]` but the single-element pattern
`["sports"]` does NOT match `["live-sports", "channel"]` since "sports" is a
substring within an element, not a complete element.

Note: To match all values, use MATCH_PREFIX with an empty pattern (`[]` for
namespaces or `""` for track names). An empty pattern is a prefix of every value.

### Matching Algorithm {#matching-algorithm}

When a MoQ relay receives a request with a Privacy Pass token, it performs the
following validation steps to determine whether to authorize the requested
operation:

~~~aasvg
+------------------+
| Extract Token    |
| from MoQ Message |
+--------+---------+
         |
         v
+------------------+     +----------------+
| Verify Token     |---->| Authorization  |
| (Type-specific)  | No  | Failed         |
+--------+---------+     +----------------+
         | Yes
         v
+------------------+     +----------------+
| Check Replay     |---->| Authorization  |
| Protection       | No  | Failed         |
+--------+---------+     +----------------+
         | Yes
         v
+------------------+
| Extract Scope    |
| (origin_info or  |
| redemption_ctx)  |
+--------+---------+
         |
         v
+------------------+
| For each Scope:  |<---------+
+--------+---------+          |
         |                    |
         v                    |
+------------------+          |
| Action in        |--+       |
| scope.actions?   |  | No    |
+--------+---------+  |       |
         | Yes        |       |
         v            |       |
+------------------+  |       |
| Namespace Match  |--+       |
| Rule passes?     |  | No    |
+--------+---------+  |       |
         | Yes        |       |
         v            |       |
+------------------+  |       |
| Track Name Match |--+       |
| Rule passes?     |  | No    |
+--------+---------+  +-------+
         | Yes           ^
         v               | More scopes
+------------------+     |
| Authorization    |     |
| Granted          |     |
+------------------+     |
                         |
              No match --+
              in any     |
              scope      v
                  +----------------+
                  | Authorization  |
                  | Failed         |
                  +----------------+
~~~
{: #fig-matching-algorithm title="Token Validation and Matching Algorithm"}

1. **Token Extraction**: Extract the Privacy Pass token from the MoQ control
   message (SETUP, SUBSCRIBE, FETCH, PUBLISH, ANNOUNCE, or other operation).

2. **Token Verification**: Verify the token using the appropriate method for
   the token type:

   - Token Type `0x0001` or `0x0005` (VOPRF): Verify using the issuer's
     private validation key
   - Token Type `0x0002` (Blind RSA): Verify using the issuer's public
     verification key
   - Token Type `0xE5AC` (ARC): Verify the presentation proof using the
     issuer's public parameters

3. **Replay Protection**: Validate that the token has not been replayed:

   - Check token nonce uniqueness within the configured replay window
   - Verify token expiration timestamp if present in token metadata

4. **Scope Extraction**: Extract authorization scope from the token:

   - If using `origin_info`: Decode the `MoQAuthorizationInfo` structure
   - If using `redemption_context`: Decode the `CompactMoQScope` structure

5. **Scope Evaluation**: For each `MoQAuthScope` in the token, check if the
   requested operation is authorized:

   a. **Action Check**: Verify the requested MoQ action (from {{moq-actions}})
      is present in the scope's `actions` list or bitmask

   b. **Namespace Match**: Apply the `namespace_match` rule to the requested
      track namespace using the algorithm in {{match-types}}

   c. **Track Name Match**: Apply the `track_name_match` rule to the requested
      track name using the algorithm in {{match-types}}

   d. If all three checks pass, authorization succeeds for this scope

6. **Authorization Decision**: Access is granted if and only if:

   - Token verification succeeds (step 2)
   - Replay protection passes (step 3)
   - At least one scope in the token authorizes the operation (step 5)

If authorization fails, an error is returned as specified in {{errors}}.

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

### Continuous Authorization with Batched Tokens {#continuous-auth}

Long-lived MoQ sessions (such as live streaming or real-time communication)
require periodic re-authorization to ensure continued eligibility. Unlike
JWT-based approaches that use explicit revalidation intervals, Privacy Pass
achieves continuous authorization through batched token issuance.

During the initial SETUP exchange, clients can request multiple tokens via
`GenericBatchTokenRequest`. Each token in the batch is independently valid
and can be presented for subsequent operations or periodic re-authorization.

~~~
Batched Token Usage Timeline:

Time 0:     CLIENT_SETUP with Token_1, request batch of N tokens
            SERVER_SETUP with batch of N tokens

Time T:     SUBSCRIBE with Token_2 (from batch)

Time 2T:    Client presents Token_3 for continued authorization
            (proactive re-auth before relay requests it)

Time 3T:    Relay requests re-authorization
            Client presents Token_4
~~~

Relays MAY request periodic re-authorization by sending a `TokenChallenge`
in a `REQUEST_ERROR` message. Clients SHOULD present a fresh token from their
batch in response.

When using ARC tokens (`0xE5AC`), the credential's `presentation_limit` controls
how many times the client can present tokens from a single credential issuance.
This provides rate limiting while preserving unlinkability between presentations.

**Deployment Considerations**:

- Batch size SHOULD be sufficient for the expected session duration
- Relays SHOULD configure re-authorization intervals based on content
  sensitivity and trust requirements
- Clients SHOULD request new token batches before exhausting their supply
- For high-security deployments, shorter re-authorization intervals with
  smaller batches provide stronger revocation guarantees

### Errors {#errors}

If the authentication fails for any reason, the server MUST send an error.

If the error occurs during SETUP, the Relay MUST terminate the connection with
`UNAUTHORIZED` defined in {{Section 3.4 of MoQ-TRANSPORT}}.

If the error occurs over an establishhed connection, the Relay MUST send a `REQUEST_ERROR`
defined in {{Section 9.8 of MoQ-TRANSPORT}}.

In both cases, the Relay SHOULD provide a reason/message set to a `TokenChallenge`.

> TODO: reason tends to be a string. should TokenChallenge be encoded in base64, or even have a structure?

# Example Authorization Flow

Below shows an example deployment scenarios where the relay has been
configured with the necessary validation keys and content policies, the
relay can verify Privacy Pass tokens locally and deliver media directly
without contacting the Issuer. This uses token with public verifiability.

~~~~~aasvg
         +-----------+                        +--------+         +----------+ +--------+
         | MoQ Relay |                        | Client |         | Attester | | Issuer |
         +-----+-----+                        +---+----+         +----+-----+ +---+----+
               |                                  |                   |           |
               |<--------------- CLIENT_SETUP[] --+                   |           |
               |   UNAUTHORIZED [                 |                   |           |
               +--   Reason=TokenChallenge ------>|                   |           |
               |   ]                              |                   |           |
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

# Security Considerations

TODO: Add considerations for the security and privacy of the Privacy Pass
tokens.

* Token Replay
* Token harvest
* Key rotation
* Use of TLS

# IANA Considerations

## MoQ Privacy Pass Auth Scheme Registry

IANA is requested to create a new registry titled "MoQ Privacy Pass Auth
Schemes" with the following initial contents:

| Value | Name | Reference |
|-------|------|-----------|
| 0x00 | Reserved | This document |
| 0x01 | PrivateTokenAuth | This document |
{: #auth-scheme-registry title="MoQ Privacy Pass Auth Schemes"}

New entries in this registry require Specification Required registration policy.

## MoQ Action Registry

IANA is requested to create a new registry titled "MoQ Actions for Privacy
Pass Authorization" with the following initial contents:

| Value | Action | Reference |
|-------|--------|-----------|
| 0 | CLIENT_SETUP | {{moq-actions}} |
| 1 | SERVER_SETUP | {{moq-actions}} |
| 2 | PUBLISH_NAMESPACE | {{moq-actions}} |
| 3 | SUBSCRIBE_NAMESPACE | {{moq-actions}} |
| 4 | SUBSCRIBE | {{moq-actions}} |
| 5 | REQUEST_UPDATE | {{moq-actions}} |
| 6 | PUBLISH | {{moq-actions}} |
| 7 | FETCH | {{moq-actions}} |
| 8 | TRACK_STATUS | {{moq-actions}} |
| 9-254 | Unassigned | |
| 255 | Reserved | This document |
{: #moq-action-registry-table title="MoQ Actions Registry"}

New entries in this registry require Specification Required registration policy.
Values SHOULD align with MoQTransport control message types where applicable.

## MoQ Match Type Registry

IANA is requested to create a new registry titled "MoQ Match Types for Privacy
Pass Authorization" with the following initial contents:

| Value | Match Type | Reference |
|-------|------------|-----------|
| 0 | MATCH_EXACT | {{match-types}} |
| 1 | MATCH_PREFIX | {{match-types}} |
| 2 | MATCH_SUFFIX | {{match-types}} |
| 3 | MATCH_CONTAINS | {{match-types}} |
| 4-254 | Unassigned | |
| 255 | Reserved | This document |
{: #match-type-registry title="MoQ Match Types Registry"}

New entries in this registry require Specification Required registration policy.


--- back

# Acknowledgments

TODO acknowledge.

# Change Log

RFC Editor's Note: Please remove this section prior to publication of
a final version of this document.

## Since draft-ietf-moq-privacy-pass-auth-02

* Replace text-based moq-scope with binary TLS presentation language structures
* Add MoQ Actions registry aligned with MoQTransport control message types
* Add Match Types registry with exact, prefix, suffix, and contains matching
* Define MoQAuthorizationInfo structure for origin_info encoding
* Define CompactMoQScope structure for redemption_context encoding
* Add continuous authorization section using batched tokens
* Add IANA registries for auth schemes, actions, and match types

## Since draft-ietf-moq-privacy-pass-auth-01

* Define error handling
* Integrate privacy pass reverse flow within PrivateTokenAuth
* MoQ definition now follow draft-ietf-moq-transport-16
* Update dependencies
* Removed b64 encoding given MoQ can use bytes directly


## Since draft-ietf-moq-privacy-pass-auth-00

* Add Thibault Meunier as Coauthor
* Add support for Reverse flow to be deploy and scale friendly way to get tokens
