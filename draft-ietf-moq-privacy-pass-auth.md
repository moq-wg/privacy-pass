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
PUBLISH_NAMESPACE.

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

The reverse flow, as described in {{Section 4 of PRIVACYPASS-REVERSE-FLOW}},
allows a client to exchange a publicly verifiable token for privately verifiable
tokens (or credentials) issued directly by the MoQ relay.

~~~aasvg
+---------------+  +------------+        +--------+        +----------+  +--------+
| Origin Issuer |  |  MoQ Relay |        | Client |        | Attester |  | Issuer |
+-------+-------+  +------+-----+        +----+---+        +-----+----+  +----+---+
        |                 |                   |                  |           |
        |                 |                   |                  |           |
        :     Phase 1: Bootstrap Token Acquisition               :           :
        |                 |                   |                  |           |
        |                 |<- CLIENT_SETUP[] -+                  |           |
        |                 +-- UNAUTHORIZED -->|                  |           |
        |                 |  [TokenChallenge] |                  |           |
        |                 |                   |                  |           |
        |                 |                   |<== Attestation ==>           |
        |                 |                   |                  |           |
        |                 |                   +---- TokenRequest ----------->|
        |                 |                   |<--- TokenResponse -----------+
        |                 |                   |                  |           |
        :     Phase 2: Token Exchange via Reverse Flow           :           :
        |                 |                   |                  |           |
        |                 |<- CLIENT_SETUP ---+                  |           |
        |                 |   [Token +        |                  |           |
        |                 |    CredentialReq] |                  |           |
        |                 |                   |                  |           |
        |<-CredentialReq--+                   |                  |           |
        +--CredentialRes->|                   |                  |           |
        |                 |                   |                  |           |
        |                 +-- SERVER_SETUP -->|                  |           |
        |                 |  [CredentialResp] |                  |           |
        |                 |                   |                  |           |
        :     Phase 3: Normal Operations with Derived Tokens     :           :
        |                 |                   |                  |           |
        |                 |<-- SUBSCRIBE -----+                  |           |
        |                 |   [Token from     |                  |           |
        |                 |    credential]    |                  |           |
        |                 +-- SUBSCRIBE_OK -->|                  |           |
        |                 |                   |                  |           |
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

For MoQ usage, authorization scope information can be encoded by the origin
within `origin_info` field. This is encoded in the Token at issuance time when
types `0x0001`, `0x0002`, `0x0005` are used. When clients present a credential
such as with {{ARC}}, the scope may be restricted at presentation time.

Origins MAY use `redemption_context` to scope token use to properties of the client
session. As described in {{Section 2.1.1.2 of RFC9577}}, `redemption context` can
be set to 32-byte random nonce, to the hash of a specific time window, or even
derived from the client's ASN.

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

`MoQAction` wire representation is as follows

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
~~~

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
tuples at tuple element boundaries. The pattern in a `MatchRule` (defined in {{scope-origin-info}}) is also a
tuple of byte strings, and matching is performed element-by-element.

As for track names, match rules can be applied directly given there is a single
tuple element.

No normalization is performed on namespace tuple elements or track name values
before matching. Comparisons are performed as byte-level operations on each
tuple element.

`MatchType` wire representation is as follows

~~~
enum {
    MATCH_EXACT(0),
    MATCH_PREFIX(1),
    MATCH_SUFFIX(2),
    MATCH_CONTAINS(3),
    (255)
} MatchType;
~~~

### Authorization Scope Structure (origin_info) {#scope-origin-info}

When authorization scope is bound at issuance time, the `origin_info` field
contains a binary-encoded `MoQAuthorizationInfo` structure:

~~~
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
    MoQAuthScope scopes<1..2^8-1>;
} MoQAuthorizationInfo;
~~~

A token MAY contain multiple `MoQAuthScope` entries to authorize different
combinations of actions and resource patterns. Authorization succeeds if
ANY scope in the token permits the requested operation.

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

`MATCH_EXACT (0)`:

The target MUST be identical to the pattern. For namespace
tuples, this means the same number of elements with each element byte-for-byte
identical. The pattern tuple `["example.com", "live"]` matches only
`["example.com", "live"]`, not `["example.com", "live", "sports"]`.

`MATCH_PREFIX (1)`:

The target MUST start with the pattern at tuple element
boundaries. The pattern tuple `["example.com", "live"]` matches
`["example.com", "live", "sports"]` and `["example.com", "live", "news", "breaking"]`
but not `["example.com", "vod"]`. Note that `["example.com", "liv"]` does NOT
match `["example.com", "live"]` since matching is at element boundaries.

`MATCH_SUFFIX (2)`:

The target MUST end with the pattern at tuple element
boundaries. The pattern tuple `["audio"]` matches `["meeting123", "audio"]` and
`["conference", "room1", "audio"]` but not `["audio", "opus"]`.

`MATCH_CONTAINS (3)`:

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
         | Yes
         v
+------------------+     +----------------+
| Check Replay     |---->| Authorization  |
| Protection       | No  | Failed         |
+--------+---------+     +----------------+
         |
         v
+------------------+     +----------------+
| Verify Token     |---->| Authorization  |
| (Type-specific)  | No  | Failed         |
+--------+---------+     +----------------+
         | Yes
         v
+------------------+
| Extract Scope    |
| (origin_info)    |
+--------+---------+
         |
         v
+------------------+                 +----------------+
|                  +---------------->| Authorization  |
| For each Scope   | No more scopes  | Failed         |
|                  |<---------+      +----------------+
+--------+---------+          |
         |                    |
         v                    |
+------------------+          |
| Action in        |----------+
| scope.actions?   |    No    |
+--------+---------+          |
         | Yes                |
         v                    |
+------------------+          |
| Namespace Match  |----------+
| Rule passes?     |    No    |
+--------+---------+          |
         | Yes                |
         v                    |
+------------------+          |
| Track Name Match |----------+
| Rule passes?     |
+--------+---------+
         | Yes
         v
+------------------+
| Authorization    |
| Granted          |
+------------------+
~~~
{: #fig-matching-algorithm title="Token Validation and Matching Algorithm"}

1. **Token Extraction**: Extract the Privacy Pass token from the MoQ control
   message (SETUP, SUBSCRIBE, FETCH, PUBLISH, PUBLISH_NAMESPACE, or other operation).

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

5. **Scope Evaluation**: For each `MoQAuthScope` in the token, check if the
   requested operation is authorized:

   a. **Action Check**: Verify the requested MoQ action (from {{moq-actions}})
      is present in the scope's `actions` list

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

### Continuous Authorization with Batched Tokens {#continuous-auth-batched}

Long-lived MoQ sessions (such as live streaming or real-time communication)
require periodic re-authorization to ensure continued eligibility. Unlike
JWT-based approaches that use explicit revalidation intervals, Privacy Pass
can achieve continuous authorization through batched token issuance.

During the initial SETUP exchange, clients can request multiple tokens via
`GenericBatchTokenRequest` (defined in {{Section 6.1 of PRIVACYPASS-BATCHED}}).
Each token in the batch is independently valid
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
batch in response if any satisfy the new `TokenChallenge`. If not, they SHOULD
perform a new issuance process.

When using {{ARC}} tokens (`0xE5AC`), the credential's `presentation_limit` controls
how many times the client can present tokens from a single credential issuance.
This provides rate limiting while preserving unlinkability between presentations.

**Deployment Considerations**:

- Batch size SHOULD be sufficient for the expected session duration
- Relays SHOULD configure re-authorization intervals based on content
  sensitivity and trust requirements
- Clients SHOULD request new token batches before exhausting their supply
- For high-security deployments, shorter re-authorization intervals with
  smaller batches provide stronger revocation guarantees

### Continuous Authorization with Reverse Flow {#continuous-auth-reverse}

If the client and the relay support it, a Relay MAY perform continuous
authentication using a reverse flow.

To do so, when presenting `PrivateTokenAuth`, a client MUST send at least one
`GenericBatchTokenRequest`. The Relay then acts as a reverse issuer, and issues
the corresponding number of `GenericBatchTokenResponse`.

Tokens obtained this way can be presented by the Client to maintain
the continuity of the session without linkability.

~~~
Reverse Flow Token Usage Timeline:

Time 0:     CLIENT_SETUP with Token_1, request batch of 1 token
            SERVER_SETUP with batch of 1 token

Time T:     SUBSCRIBE with Token_2 (from batch), request batch of 1 token
            Relay responds with batch of 1 token

Time 2T:    Client presents Token_3 (from batch of time T), request batch of 1 token
            Relay responds with batch of 1 token
~~~

### Errors {#errors}

If the authentication fails for any reason, the server MUST send an error.
The error response includes a `TokenChallenge` to enable the client to obtain
a valid token and retry the operation.

#### SETUP Errors

If authentication fails during SETUP, the Relay MUST terminate the connection
with the `UNAUTHORIZED` (0x02) Termination Error Code defined in
{{Section 3.4 of MoQ-TRANSPORT}}. The termination reason phrase MUST contain
a `MoQAuthChallenge` structure:

~~~
struct {
    TokenChallenge challenge;
    uint16_t supported_token_types<1..2^16-1>;
} MoQAuthChallenge;
~~~

The `supported_token_types` field lists token types the relay accepts, ordered
by preference (most preferred first). This allows clients to select an appropriate
issuance protocol. Relay MUST set at least one supported token type.

#### Operation Errors

If the error occurs over an established connection, the Relay MUST send a `REQUEST_ERROR`
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
| 0x0106 | TOKEN_MALFORMED | Token cannot be parsed correctly |
{: #error-codes-table title="Privacy Pass Authorization Error Codes"}

The reason phrase in `REQUEST_ERROR` MUST contain a `MoQAuthChallenge` structure
when the client should retry with a new token, encoded as a byte-string.

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
               |   UNAUTHORIZED (0x2)    [        |                   |           |
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
| 0x0106 | TOKEN_MALFORMED | {{errors}} |
| 0x0107-0x01FF | Unassigned | |
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

* Replace text-based moq-scope with binary TLS presentation language structures
* Add MoQ Actions registry aligned with MoQTransport control message types
* Add Match Types registry with exact, prefix, suffix, and contains matching
* Define MoQAuthorizationInfo structure for origin_info encoding
* Add continuous authorization section using reverse flow
* Add continuous authorization section using batched tokens
* Add IANA registries for auth schemes, actions, and match types
* Define error handling
* Integrate privacy pass reverse flow within PrivateTokenAuth
* MoQ definition now follow draft-ietf-moq-transport-16
* Update dependencies
* Removed b64 encoding given MoQ can use bytes directly


## Since draft-ietf-moq-privacy-pass-auth-00

* Add Thibault Meunier as Coauthor
* Add support for Reverse flow to be deploy and scale friendly way to get tokens
