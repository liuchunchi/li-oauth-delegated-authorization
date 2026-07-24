---

title: 'OAuth 2.0 Delegated Authorization'
abbrev: 'Delegated-Auth'
docname: 'draft-li-oauth-delegated-authorization-latest'

stand_alone: true
wg: oauth
ipr: trust200902
submissiontype: IETF
cat: std

venue:
  group: WG
  type: Working Group
  mail: oauth@ietf.org
  arch: https://datatracker.ietf.org/wg/oauth/about/
  github: liuchunchi/li-oauth-delegated-authorization
  latest: https://liuchunchi.github.io/li-oauth-delegated-authorization/draft-li-oauth-delegated-authorization.html

author:
- ins: R. Li
  name: Ruochen Li
  org: Huawei Int. Pte Ltd
  email: li.ruochen@h-partners.com
- ins: H. Wang
  name: Haiguang Wang
  org: Huawei Int. Pte Ltd
  email: wang.haiguang.shieldlab@huawei.com
- ins: C. Liu
  name: Chunchi Peter Liu
  org: Huawei Technologies
  email: liuchunchi@huawei.com
- ins: T. Li
  name: Tieyan Li
  org: Huawei Int. Pte Ltd
  email: Li.Tieyan@huawei.com

normative:
  RFC6749:
  RFC6750:
  RFC7009:
  RFC7515:
  RFC7517:
  RFC7519:
  RFC7638:
  RFC7800:
  RFC8414:
  RFC8707:
  RFC8725:
  RFC9110:
  RFC9396:
  RFC9449:
  RFC9700:
  RFC9728:

informative:
  RFC8693:

--- abstract

This specification defines Delegated Authorization Tokens, key-bound tokens that enable an OAuth client to delegate a constrained subset of its authorization to another client without contacting the authorization server for each delegation. An authorization server issues the root token and binds it to a client key. That client can use the bound key either to prove possession when accessing a protected resource or to sign a further token bound to a delegate client's key. Resource servers validate the ordered token chain, the restrictions imposed at every delegation step, and a DPoP proof signed by the key bound to the leaf token.

--- middle

# Introduction

OAuth enables a client to obtain authorization to access protected resources. In systems composed of agents, services, and other independently operated components, that client may need to authorize another client to perform only part of the authorized work. Giving the second client the original access token or the original client's private key grants excessive authority and prevents the client from independently constraining each delegation step.

This specification defines a key-bound token called a Delegated Authorization Token. An authorization server initially issues such a token to an OAuth client and binds the token to a public key held by that client. The client can then locally issue another Delegated Authorization Token to a delegate client. Each client-issued token is signed with the private key bound to the preceding token, is bound to the delegate client's public key, and can only reduce the authorization conveyed by the preceding chain.

A client controlling the private key bound to a Delegated Authorization Token can use the token in either of two ways. It can sign a further token when delegation remains permitted, or it can access a protected resource by presenting the complete token chain together with a DPoP proof {{RFC9449}}. These uses rely on the same client-held key pair.

## Motivation and Goals

The mechanism is intended to:

* let a client delegate authorization without disclosing its private key or a reusable bearer credential;
* make every delegation cryptographically attributable to the client controlling the key bound to the preceding token;
* ensure that permissions, audiences, validity periods, and delegation depth can only be narrowed;
* let a resource server validate a delegation chain without contacting the authorization server for every delegation; and
* bind protected resource access to the client controlling the key bound to the leaf token.

## Use Cases

The following non-normative use cases illustrate where local, constrained delegation is useful.

### Specialized AI Agents

An enterprise or individual can use an orchestrator agent that coordinates specialized agents for research, software development, testing, or access to internal knowledge. The authorization server issues a root token to the orchestrator. For each task, the orchestrator issues a child token to the selected agent and limits the token to the resource servers, permissions, and delegation depth needed for that agent's role. A terminal agent can receive a token with `max_delegation_depth` set to `0`.

An agent does not thereby inherit all authorization conveyed to the orchestrator, much less all authority associated with the resource owner. For example, a research agent can receive read access to an internal knowledge service without receiving deployment privileges, while a deployment agent can receive access to one environment without receiving access to unrelated data services.

The token lifetime can be chosen for the agent's task and operational model, but remains finite and subject to the authorization server's configured maximum. Subject to authorization server policy, a long-running agent can use a token with a longer finite lifetime or periodically obtain a new root token through a supported OAuth grant type. Longer-lived tokens increase the consequences of key compromise and delayed revocation, so deployments using them need correspondingly strong private-key protection and a revocation-status distribution mechanism that meets their response-time requirements.

### Third-Party Analytics for Enterprise SaaS

A corporate tenant can authorize a SaaS CRM client to delegate limited CRM-data access to a third-party analytics client. The authorization server issues a root token to the CRM client. The CRM client then issues a child token bound to the analytics client's key, with an audience restricted to the CRM API, read-only permissions, authorization details limiting the accessible data, and `max_delegation_depth` set to `0`.

The analytics client presents the complete chain and a DPoP proof to the CRM resource server. It does not receive the CRM client's private key or the CRM client's full authorization, and the authorization server need not participate in each analytics task.

## Scope

This document specifies the token format, authorization server issuance, client-to-client delegation, ordered token-chain presentation and validation, and proof-of-possession access to protected resources.

The mechanism does not define how one client discovers another client's public key, how clients negotiate delegation out of band, how multiple Delegated Authorization Token Chains are delivered in one request, how delivered chains are bound to a request or task, the type-specific rules for comparing application-specific authorization details, or how revocation status is distributed to resource servers. These functions are outside the scope of this specification and can be defined by deployment policy or by extending this specification.

## Relationship to OAuth 2.0

This specification extends the OAuth 2.0 authorization framework {{RFC6749}}. It uses the roles, grants, endpoints, and authorization server metadata model established by OAuth 2.0. Deployments apply the requirements of the underlying OAuth 2.0 grant type together with the requirements in this document and the security guidance in {{RFC9700}}.

A Delegated Authorization Token is not a bearer token. Its use requires proof of possession of the key bound to the leaf token. This specification uses DPoP for that proof and defines the `DA` HTTP authentication scheme rather than the `Bearer` or `DPoP` schemes. An implementation MUST NOT treat a Delegated Authorization Token as a bearer access token.

# Requirements Language

{::boilerplate bcp14-tagged}

# Terminology

This specification uses the terms "authorization server", "client", "resource server", "resource owner", "authorization grant", and "protected resource" as defined by OAuth 2.0 {{RFC6749}}. It uses "DPoP proof" as defined by {{RFC9449}}.

**Delegation**:
: The act by which a delegating client issues a Client-Issued Delegated Authorization Token to a delegate client.

**Delegating Client**:
: A client that holds a Delegated Authorization Token and uses the corresponding private key to issue a Client-Issued Delegated Authorization Token to another client.

**Delegate Client**:
: A client to which a delegating client delegates authorization by issuing a Client-Issued Delegated Authorization Token. It holds the private key corresponding to the `cnf.jkt` value in that token. A delegate client can subsequently act as a delegating client.

**Delegated Authorization Token**:
: A signed JWT that conveys authorization, binds that authorization to a client key using `cnf.jkt`, and can be used as one element of a Delegated Authorization Token Chain. The token can authorize protected resource access and, subject to `max_delegation_depth`, further delegation.

**Authorization-Server-Issued Delegated Authorization Token**:
: A Delegated Authorization Token issued and signed by an authorization server. It is the root of a Delegated Authorization Token Chain.

**Client-Issued Delegated Authorization Token**:
: A Delegated Authorization Token issued by a delegating client, signed with the private key bound to the preceding token, and bound to the delegate client's key.

**Delegated Authorization Token Chain**:
: An ordered sequence containing one Authorization-Server-Issued Delegated Authorization Token followed by zero or more Client-Issued Delegated Authorization Tokens. The sequence is ordered from the root token to the leaf token.

**Root Token**:
: The first token in a Delegated Authorization Token Chain. It is issued by the authorization server.

**Leaf Token**:
: The last token in a Delegated Authorization Token Chain. Its `cnf.jkt` identifies the key whose corresponding private key is used to create a DPoP proof or to issue a further token.

# Delegated Authorization Model

## Overview

~~~aasvg
{::include diagrams/delegated-authorization-architecture.ascii-art}
~~~
{: #fig-architecture title="Delegated Authorization Model" align="center"}

Every client that receives a Delegated Authorization Token has an asymmetric key pair. The private key remains under that client's control. The token's `cnf.jkt` member identifies the corresponding public key by its JWK thumbprint {{RFC7638}}.

The authorization server creates the root token and signs it with one of its signing keys. When the client holding the root token delegates authorization, it creates a new token, places its own public key in the new token's protected `jwk` header parameter, binds the new token to the delegate client's key using `cnf.jkt`, and signs the new token with its own private key. This procedure can be repeated to form a chain.

For every adjacent pair in a valid chain, the JWK thumbprint of the child token's protected `jwk` header parameter equals the parent token's `cnf.jkt` value. The child signature therefore demonstrates that the client controlling the key bound to the parent token authorized the next delegation.

A client accesses a protected resource by presenting the complete ordered chain and a DPoP proof. The DPoP proof is signed with the private key identified by the leaf token's `cnf.jkt`. The resource server validates the root token's signature using an authorization server key, every client-issued token signature, key continuity between adjacent tokens, all downscoping restrictions, and the DPoP proof.

## Trust Model

A resource server trusts one or more authorization servers to establish root authorization. It obtains the authorization server's signing keys through configured or discovered means. The resource server does not need a pre-established trust relationship with every delegating client. A client's authority to sign the next token derives from the preceding token's `cnf.jkt`.

A delegating client is trusted only to exercise and further constrain the authorization conveyed by its token chain. It cannot validly increase that authorization. A verifier MUST reject a chain when it cannot establish that every child token is no broader than its parent.

Possession of a serialized token chain is insufficient to use it. A client presenting a serialized token chain MUST prove possession of the private key bound to the leaf token. A delegate client SHOULD validate a received chain before accepting it, particularly when it relies on the chain to perform security-sensitive work.

## Client Key Pairs

A client receiving a Delegated Authorization Token MUST control an asymmetric key pair suitable for digital signatures. The client MUST protect the private key and MUST NOT provide it to a delegating client, resource server, or authorization server.

The JWK thumbprint used in `cnf.jkt` MUST be computed according to {{RFC7638}}. The same bound key is used to sign a child Delegated Authorization Token when the client delegates and to sign a DPoP proof when the client accesses a protected resource. A deployment MAY rotate client keys by obtaining or receiving a new token bound to a new key, but changing a key does not alter an existing token's binding.

Clients SHOULD use a distinct key pair for each Delegated Authorization Token. This one-token-one-key model limits correlation, simplifies key lifecycle and revocation, and prevents a child token signed under one parent from being reusable with another parent token bound to the same key. When operational constraints make a distinct key per token impractical, a client MAY reuse a key for a small set of tokens used in the same security context or usage scenario. A client SHOULD NOT reuse a bound key across unrelated audiences, tenants, applications, or authorization contexts.

The means by which a delegating client obtains a delegate client's public key and establishes that it is associated with the intended delegate client is outside the scope of this specification. It can be provided by a local agent runtime, configuration, an authenticated application protocol, or another out-of-band mechanism.

## Initial Token Issuance by an Authorization Server

A client obtains an Authorization-Server-Issued Delegated Authorization Token using a supported OAuth grant type. The client indicates that it requests delegated authorization and supplies a DPoP proof at the token endpoint. The authorization server validates the authorization grant according to the requirements of the grant type, authenticates the client when required, and verifies the proof. It then places the thumbprint of the public JWK carried in the DPoP proof in `cnf.jkt` and signs the root token.

The authorization server determines the initial permissions, audience, validity period, and maximum delegation depth according to the authorization established by the grant, resource owner authorization, client policy, and server policy.

## Client-to-Client Delegation

A delegating client can issue a token only when the effective delegation depth of its current leaf token permits another level. The delegating client binds the new token to the delegate client's public key and signs it with the private key bound to its own leaf token.

The new token can narrow permissions, audience, validity period, and remaining delegation depth. Omission of `scope` or `authorization_details` in a child discards that permission component; other omitted restrictions are inherited as specified for each claim. The effective authorization at any point is the cumulative result of all tokens from the root through that point.

Client-to-client delegation does not require a request to the authorization server. The clients can use a protocol defined elsewhere or an out-of-band mechanism to exchange the delegate client's public key and deliver the resulting token chain.

## Accessing a Protected Resource

A client presents a Delegated Authorization Token Chain using the `DA` HTTP authentication scheme and sends a DPoP proof in the `DPoP` header field. The chain is serialized from root to leaf. The DPoP `ath` claim binds the proof to that exact serialized chain, and the JWK thumbprint of the proof's public JWK computed according to {{RFC7638}} MUST equal the leaf token's `cnf.jkt` value.

The resource server validates the chain and the proof before enforcing its application-specific authorization policy. A token with an effective `max_delegation_depth` of zero can still authorize access to a protected resource; the value only prevents further delegation.

# Delegated Authorization Token Format

## General JWT Requirements

A Delegated Authorization Token is a signed JWT {{RFC7519}} represented using JWS Compact Serialization {{RFC7515}}. It MUST be integrity protected and MUST NOT use the `none` algorithm. This specification does not define encrypted Delegated Authorization Tokens.

The octets of each compact JWT are significant when computing the token-chain serialization and the DPoP `ath` value. A verifier MUST NOT reserialize a JWT before performing that calculation.

A verifier MUST reject a Delegated Authorization Token if its protected JOSE header or JWT Claims Set contains duplicate member names. The last-member-wins processing option permitted by JWS and JWT MUST NOT be used. A verifier MUST also reject duplicate member names in any nested JSON object whose contents it processes for validation or authorization, including `cnf`, public JWKs, authorization detail objects, and extension claim values. Duplicate names MUST be detected before parser behavior that discards, overwrites, or combines them can affect processing.

## JOSE Header

### The `typ` Header Parameter

Every Delegated Authorization Token MUST include, in its protected header, a `typ` parameter with the value `da+jwt`. A verifier MUST compare this value using a case-sensitive comparison. The explicit type prevents a token from being confused with another kind of JWT.

### The `jwk` Header Parameter

A Client-Issued Delegated Authorization Token MUST include a `jwk` parameter in its protected header. The value MUST be a public JWK {{RFC7517}}, MUST NOT contain private key material, and MUST represent the key used to verify that token's signature.

The JWK thumbprint of this value MUST equal the `cnf.jkt` value of the immediately preceding token in the presented chain. An Authorization-Server-Issued Delegated Authorization Token MUST NOT contain the `jwk` header parameter. Its signature verification key MUST be obtained from trusted authorization server configuration or authenticated authorization server metadata.

### Algorithm Requirements

Token issuers and verifiers MUST use asymmetric digital signature algorithms appropriate for the supplied key. They MUST reject `none`, symmetric MAC algorithms, algorithms disallowed by local policy, and a token whose `alg` is inconsistent with the verification key. A verifier MUST select acceptable algorithms by policy and MUST NOT derive acceptance solely from the token's `alg` value.

Implementations MUST apply mutually exclusive validation rules for Delegated Authorization Tokens and other JWT types, as required by {{RFC8725}}. A JWT accepted as another token type MUST NOT be accepted as a Delegated Authorization Token solely because it otherwise satisfies generic JWT validation.

A resource server MUST maintain an allowlist for Client-Issued Delegated Authorization Token signature algorithms and an allowlist for DPoP proof algorithms. The two allowlists MAY be the same or different. Before issuing a child token, a client MUST select an algorithm accepted by every resource server expected to validate the chain, using authenticated protected resource metadata, configuration, or another authenticated mechanism. An authorization server's root-token signing metadata MUST NOT be interpreted as advertising algorithms accepted for client-issued tokens or protected resource DPoP proofs.

For an ECDSA signature, the token issuer MUST produce a canonical low-S signature: the integer `s` MUST be no greater than half the order of the curve's base point. A verifier MUST reject an ECDSA signature whose `s` value is greater than half that order. This requirement prevents an existing signature from being transformed into a bytewise-distinct valid signature and ensures that a token identified by a digest of its exact compact serialization cannot evade revocation through ECDSA signature malleability.

## Delegated Authorization Token Claims

### The `cnf` Claim and `jkt` Member

Every Delegated Authorization Token MUST contain a `cnf` claim as defined by {{RFC7800}}. The `cnf` claim MUST contain the `jkt` member defined by {{RFC9449}}. Its value is the base64url-encoded SHA-256 thumbprint of the bound public JWK, computed according to {{RFC7638}}.

The `cnf` claim MUST contain exactly one member, `jkt`. A verifier MUST reject a token whose `cnf` claim contains any other member. This specification does not define the use of multiple confirmation methods or any confirmation method other than `jkt`.

The client controlling the corresponding private key uses it to create a DPoP proof for protected resource access and, when the delegation depth permits, to sign a child token.

### The `max_delegation_depth` Claim

The `max_delegation_depth` claim is an integer in the range 0 to 65535, inclusive, specifying how many further client-issued levels the token permits beneath itself. An Authorization-Server-Issued Delegated Authorization Token MUST contain this claim.

For a child token whose parent's effective depth is a positive integer `m`, omission gives the child an effective depth of `m - 1`; an explicit value `n` MUST satisfy `0 <= n <= m - 1`. A token with an effective depth of zero MUST NOT have a child token.

The effective depth is evaluated for each adjacent pair of tokens. A value of zero prohibits further delegation but does not prohibit protected resource access.

### The `scope` Claim

The `scope` claim carries a space-delimited set of scope values using the syntax and semantics of the OAuth `scope` token response parameter {{RFC6749}}. A child token that contains `scope` MUST contain a subset of its parent's effective scope. If a child omits `scope`, it discards the scope permission component, and neither that child nor any descendant conveys scope authorization. If the parent has no effective scope, the child token MUST NOT contain `scope`.

If the root token omits `scope`, no scope authorization is conveyed by the chain.

### The `authorization_details` Claim

The `authorization_details` claim is a JSON array of authorization detail objects as defined by {{RFC9396}}. A child token that contains `authorization_details` MUST be no broader than its parent's effective authorization details. If a child omits `authorization_details`, it discards the authorization-details permission component, and neither that child nor any descendant conveys authorization through `authorization_details`. If the parent has no effective authorization details, the child token MUST NOT contain `authorization_details`. If the root token omits `authorization_details`, no authorization details are conveyed by the chain.

A specification that defines an authorization detail `type` for use with delegation MUST define how one value is determined to be a subset of another. Support for {{RFC9396}} alone is not sufficient to support delegation of an authorization detail type. A type can be retained in a Client-Issued Delegated Authorization Token only when its defining specification or a companion specification defines deterministic containment rules for delegation. An implementation that supports an authorization detail type for ordinary OAuth authorization does not thereby support that type in a Delegated Authorization Token Chain.

A root token can contain an authorization detail type without delegation containment rules and can be used as a single-token chain. A client-issued token can discard that permission component by omitting `authorization_details`. If a client-issued token retains `authorization_details`, the verifier MUST implement the applicable containment rules and MUST reject the chain if it cannot establish that every child authorization detail is covered by an effective parent authorization detail.

`scope` and `authorization_details` are separate permission components. When both are present, the resource server combines them according to deterministic semantics defined by the applicable authorization detail type, deployment profile, or resource server policy. Those semantics MUST be monotonic: removing either component, removing a scope value, or narrowing an authorization detail MUST NOT increase the set of authorized operations.

Omission of a permission component from a child discards that component. It MUST NOT be interpreted as removing a restriction on the remaining component. A verifier MUST reject a chain when the applicable composition semantics could cause omission or downscoping of a permission component to increase authorization.

### The `aud` Claim

The `aud` claim has the syntax defined by {{RFC7519}} and identifies the intended resource servers. Whenever present, the `aud` claim MUST identify at least one audience. The root token MUST contain an `aud` claim. Each audience in a child token MUST be included in the parent's effective audience. A child that omits `aud` imposes no additional audience restriction; the effective audience remains the parent's effective audience.

### The `exp` Claim

The `exp` claim has the syntax and processing rules defined by {{RFC7519}}. An Authorization-Server-Issued Delegated Authorization Token MUST contain `exp`.

A child token's `exp`, when present, MUST NOT be later than the parent's effective expiration time. A child that omits `exp` inherits the parent's effective expiration time. The effective expiration time of a chain prefix is the earliest `exp` value in that prefix. Consequently, every valid chain has a finite effective expiration time.

### The `nbf` Claim

The `nbf` claim has the syntax and processing rules defined by {{RFC7519}}. An `nbf` claim imposes an additional not-before restriction on the chain. Its omission means that the token imposes no additional not-before restriction.

If the parent has an effective not-before time, a child token's `nbf`, when present, MUST NOT be earlier than that time. If the parent has no effective not-before time, a child MAY contain an `nbf` value.

The effective not-before time of a chain prefix is the latest `nbf` value present in that prefix. If no token in the prefix contains `nbf`, the prefix has no effective not-before time.

### The `iss`, `sub`, `iat`, and `jti` Claims

An Authorization-Server-Issued Delegated Authorization Token MAY include `sub`, `iat`, and `jti` in accordance with {{RFC7519}} and authorization server policy. The `iss` claim MUST be present and identify the authorization server.

A Client-Issued Delegated Authorization Token MUST NOT contain `iss`, `sub`, `iat`, or `jti`. This specification provides no mechanism for a verifier to validate these claims when they are asserted by a delegating client; only their values in the authorization-server-issued root token are validated under the authorization server's policy.

### Extension Claims

An extension claim is a JWT payload claim whose semantics are not defined by this specification. An authorization server MAY include extension claims in a root token. A verifier MUST reject a root token containing an unrecognized extension claim unless trusted authorization server configuration or an applicable deployment profile explicitly permits that claim to be ignored. A verifier that ignores such a claim MUST NOT use it as a source of authorization or as an authorization restriction.

An authorization server MUST NOT rely on an extension claim to grant or restrict authorization unless every intended verifier is known to implement the claim's processing semantics. When an extension claim affects authorization, a verifier that does not implement those semantics MUST reject the token.

A Client-Issued Delegated Authorization Token MUST NOT contain an extension claim unless a specification defining that claim also defines its use in Delegated Authorization Tokens. Such a specification MUST define:

* whether the claim conveys authorization, imposes a restriction, or supplies contextual information;
* whether the claim is permitted in root tokens, client-issued tokens, or both;
* whether a client-issued token can introduce or reintroduce the claim;
* the effect of omission from a child token;
* when applicable, a deterministic procedure for establishing that a child value is no broader than its parent's effective value; and
* how the claim composes with the permission and restriction claims defined by this specification.

A verifier MUST reject a Client-Issued Delegated Authorization Token containing an extension claim for which it does not implement the delegation semantics required by the claim's defining specification. It MUST also reject a chain when it cannot establish that an authorization-affecting extension claim complies with those semantics. An extension claim MUST NOT expand the effective `scope`, `authorization_details`, audience, validity period, or delegation depth established by the chain.

## Authorization-Server-Issued Delegated Authorization Tokens

An Authorization-Server-Issued Delegated Authorization Token MUST:

* be signed by the authorization server;
* contain a protected `typ` header parameter with the value `da+jwt`;
* contain `iss` and `cnf.jkt`;
* contain an `exp` claim;
* contain a `max_delegation_depth` claim with a value in the range 0 to 65535, inclusive;
* contain an `aud` claim identifying one or more intended resource servers; and
* contain a non-empty `scope` claim, a non-empty `authorization_details` claim, or both.

Its signing key is resolved from trusted authorization server configuration or metadata. The token MAY contain extension claims subject to the rules in Section 5.3.9.

## Client-Issued Delegated Authorization Tokens

A Client-Issued Delegated Authorization Token MUST:

* be signed by the private key bound to the immediately preceding token;
* contain a protected `typ` header parameter with the value `da+jwt`;
* contain the signing public key in the protected `jwk` header parameter;
* contain `cnf.jkt` for the delegate client's key;
* contain a non-empty `scope` claim, a non-empty `authorization_details` claim, or both;
* comply with every inherited restriction; and
* omit `iss`, `sub`, `iat`, and `jti`.

The token does not identify a particular parent by embedding it or by carrying a parent-token hash. Its position in the presented chain, key continuity, signature, and downscoping relationships establish whether it is valid in that chain.

## Representing the Delegated Authorization Token Chain

A Delegated Authorization Token Chain is serialized as one or more JWS Compact Serialization values, as defined by {{RFC7515}}, joined by the tilde character (`~`). Tokens MUST appear in root-to-leaf order. The first token MUST be authorization-server-issued, and every later token MUST be client-issued. Each chain element MUST be non-empty and MUST be a valid JWS Compact Serialization.

A single-token chain consists only of an Authorization-Server-Issued Delegated Authorization Token, which is both root and leaf. Empty elements and trailing separators are invalid.

The chain itself is not a JWT and is not covered by the `da+jwt` media type. Its integrity follows from the individual signatures, adjacent key continuity, downscoping validation, and, during protected resource access, the DPoP proof's `ath` value over the exact chain serialization.

Implementations MUST enforce deployment-appropriate limits on both the number of tokens in a chain and its serialized size.

# Obtaining an Authorization-Server-Issued Delegated Authorization Token

## Authorization Server Support

An authorization server can issue Delegated Authorization Tokens using any grant type for which it has defined a suitable policy. Support is indicated by authorization server metadata as described in Section 6.8. A client MUST NOT assume that an authorization server supports Delegated Authorization Token issuance merely because it supports OAuth access token issuance.

The authorization server determines the clients and grant types for which it supports Delegated Authorization Tokens, as well as the supported resources, scopes, authorization detail types, signing algorithms, and delegation depths. It MUST apply the security requirements of the underlying grant type in addition to this specification.

## Authorization Request

For a grant type that uses an authorization endpoint, the client requests a Delegated Authorization Token by including the following parameter in the authorization request:

**delegation**:
: REQUIRED for such a request. Its value MUST be the case-sensitive string `true`.

**max_delegation_depth**:
: OPTIONAL. An integer in the range 0 to 65535, inclusive, specifying the exact maximum delegation depth requested by the client. The value MUST use the canonical decimal representation with no sign, whitespace, or leading zeros except for the value `0`. If omitted, the authorization server determines the value according to the grant, resource owner authorization, client policy, and server policy.

When the client includes `max_delegation_depth`, the authorization server MUST either authorize that exact value or reject the request. It MUST NOT issue a token with a smaller or larger value. When the client omits the parameter, the authorization server selects the depth covered by the authorization decision.

For an authorization flow in which authorization is obtained from a resource owner, the authorization decision MUST cover the client's ability to issue Client-Issued Delegated Authorization Tokens without further authorization server or resource owner interaction and the maximum delegation depth that will be issued. Resource owner authorization to issue an access token without delegation capability MUST NOT be interpreted as authorization for client-to-client delegation. Resource owner refusal of an explicitly requested depth MUST cause the request to fail and MUST NOT be interpreted as approval for a smaller depth, including zero. This specification does not define how the delegation capability and depth are presented to the resource owner.

The request MUST identify the requested authorization using `scope`, `authorization_details`, `resource` parameters as defined by {{RFC8707}} when supported, or a combination of these mechanisms accepted by the authorization server. When the client uses the `resource` parameter, the authorization server MUST process it according to {{RFC8707}} and MUST constrain the root token's `aud` claim to resources authorized for the request. Every audience included in the issued root token MUST correspond to a resource authorized for the request. The authorization server MUST bind both the delegated-authorization request and the authorized restrictions to the authorization code or other intermediate credential.

An authorization server MUST NOT interpret an omitted `delegation` parameter as `true`. An invalid or unsupported value results in an authorization error as described in Section 6.10.

## Token Request

The client sends the token request required by the selected grant type. The request MUST include:

* a `delegation` parameter with the value `true`; and
* a DPoP proof in the `DPoP` header field as specified by {{RFC9449}}.

The token request MAY include `scope`, `authorization_details`, `max_delegation_depth`, and `resource` parameters as defined by {{RFC8707}} when permitted by the grant type and authorization server policy. The syntax and exact-value semantics of `max_delegation_depth` are the same as in the authorization request. A `resource` parameter in the token request MUST NOT select a resource outside the authorization established by the grant. Such a request MUST NOT otherwise expand authorization established by the grant.

The client authenticates to the token endpoint when required by the underlying grant type. Client authentication and the DPoP proof have distinct purposes: client authentication establishes the OAuth client's identity, while the DPoP proof establishes possession of the key to which the resulting token is bound.

## Client Key Binding

The authorization server MUST validate the token-request DPoP proof according to {{RFC9449}}, including its signature, `htm`, `htu`, `iat`, `jti`, and nonce when a nonce is required. The proof's protected `jwk` header parameter supplies the client's public key and MUST contain no private key material.

After successful validation, the authorization server computes the JWK SHA-256 thumbprint of that public key according to {{RFC7638}} and places the result in the issued token's `cnf.jkt` member. This proves that the requesting client possessed the private key at issuance time.

The key used for client authentication can be the same as or different from the key used for this binding. A public key sent without a valid proof of possession is insufficient to establish the binding.

## Token Response

A successful response uses the OAuth token response format and contains:

**access_token**:
: REQUIRED. The JWS Compact Serialization of the Authorization-Server-Issued Delegated Authorization Token.

**token_type**:
: REQUIRED. Its value MUST be `DA`. As specified by {{RFC6749}}, clients MUST compare this value case-insensitively.

**expires_in**:
: RECOMMENDED. Its value is the remaining lifetime of the returned token, in seconds, and MUST be consistent with `exp`.

The response MAY contain other parameters permitted by the grant type. An authorization server conforming only to this specification MUST NOT issue a refresh token whose use can produce a Delegated Authorization Token. Such support requires an extension that defines refresh-token binding, authorization preservation, delegation-depth preservation, key rotation, and revocation semantics.

The authorization conveyed by the returned Delegated Authorization Token is authoritative. If the response includes the `scope` response parameter, the returned root token MUST contain a `scope` claim, and the two values MUST represent the same set of space-delimited scope values. The client MUST reject a response in which those values are inconsistent.

For example:

~~~json
{
  "access_token": "eyJ0eXAiOiJkYStqd3QiLCJhbGciOiJFUzI1NiJ9...",
  "token_type": "DA",
  "expires_in": 3600,
  "scope": "read write"
}
~~~

An authorization server MUST NOT return a Delegated Authorization Token with a `token_type` identifying the token as `Bearer`, `DPoP`, or any token type other than `DA`.

## Authorization Code Grant

In an authorization code grant, the authorization request and token request MUST both contain `delegation=true`. The authorization server MUST bind the authorization request's delegation indicator, whether `max_delegation_depth` was explicitly requested, the authorized depth, and the authorized permissions to the authorization code.

If `max_delegation_depth` was included in the authorization request, the token request MAY omit it or repeat the identical value. If it was omitted from the authorization request, the token request MUST NOT introduce it. The authorization server MUST reject a mismatched or newly introduced value with `invalid_grant`.

At the token endpoint, the authorization server MUST reject an attempt to use an authorization code issued for a request that did not request delegated authorization to obtain a Delegated Authorization Token. It MUST also reject an attempt to broaden the permissions, audience, or other restrictions associated with the code. All requirements of the authorization code grant type, including PKCE requirements applicable to the deployment, continue to apply.

## Other Grant Types

An authorization server MAY support Delegated Authorization Tokens with the client credentials grant type or an extension grant type. This specification does not define issuance through the refresh token grant. The server MUST publish or otherwise document which grant types are supported and MUST verify that the client is authorized to obtain a Delegated Authorization Token through the selected grant type.

For the client credentials grant type, the authorization server determines the root authorization from the client's own authority and policy rather than resource owner authorization. The token request MAY omit `max_delegation_depth`, in which case the authorization server selects the value according to client and server policy.

## Authorization Server Metadata

This specification defines the following authorization server metadata parameters for use in metadata defined by {{RFC8414}}:

**delegated_authorization_token_supported**:
: OPTIONAL. A Boolean value indicating whether the authorization server supports issuing Delegated Authorization Tokens. Omission is equivalent to `false`.

**delegated_authorization_signing_alg_values_supported**:
: OPTIONAL. A JSON array containing the JWS `alg` values the authorization server can use to sign Authorization-Server-Issued Delegated Authorization Tokens. It MUST be present when `delegated_authorization_token_supported` is `true` and MUST NOT contain `none` or a symmetric MAC algorithm.

**delegated_authorization_max_depth**:
: OPTIONAL. An integer in the range 0 to 65535, inclusive, identifying the greatest `max_delegation_depth` value the authorization server can issue. It MUST be present when `delegated_authorization_token_supported` is `true`. This value describes server capability and does not imply that a particular client is authorized to request that depth.

For example:

~~~json
{
  "issuer": "https://as.example.com",
  "token_endpoint": "https://as.example.com/token",
  "jwks_uri": "https://as.example.com/jwks.json",
  "delegated_authorization_token_supported": true,
  "delegated_authorization_signing_alg_values_supported": ["ES256"],
  "delegated_authorization_max_depth": 4
}
~~~

The metadata describes an authorization server's capabilities, not a particular client's entitlement to use them. The server can reject a request even when it advertises support. The `dpop_signing_alg_values_supported` authorization server metadata parameter defined by {{RFC9449}} identifies algorithms accepted for DPoP proofs at the token endpoint. The `delegated_authorization_signing_alg_values_supported` parameter identifies algorithms the authorization server can use to sign root tokens. Neither parameter identifies algorithms accepted by resource servers for client-issued tokens or protected resource DPoP proofs.

## Authorization Server Processing Rules

Before issuing a token, the authorization server MUST:

1. validate the underlying OAuth grant according to the requirements of its grant type and authenticate the client when required;
2. establish that delegated authorization was requested at every required protocol step;
3. confirm that the client and grant type are permitted to request it;
4. validate requested scopes, authorization details, resources, and audiences;
5. validate the token-request DPoP proof and establish the client key binding;
6. select an allowed asymmetric signing algorithm and key;
7. set the protected `typ` header parameter to `da+jwt`;
8. include `iss`, `cnf.jkt`, and an `aud` identifying one or more intended resource servers in the token;
9. include a non-empty `scope`, a non-empty `authorization_details`, or both;
10. set a finite `exp`;
11. set a `max_delegation_depth` in the range 0 to 65535, inclusive, that equals an explicitly requested value or, when the parameter was omitted, a value selected according to the grant, resource owner authorization, client policy, and server policy; and
12. apply appropriate lifetime and delegation-depth policies.

The authorization server MUST enforce a configured maximum lifetime for root tokens and SHOULD issue the shortest lifetime suitable for the delegated task. This specification does not define a universal maximum. The authorization server MUST enforce a configured maximum delegation depth and MUST reject an explicitly requested value above that maximum.

## Error Responses

Errors are returned using the mechanism defined by the underlying OAuth grant type. The authorization endpoint returns `access_denied`, `unsupported_response_type`, `invalid_scope`, `unauthorized_client`, or `invalid_request`, as applicable. A malformed or unsupported `max_delegation_depth` value results in `invalid_request`; resource owner refusal of the requested delegation depth results in `access_denied`. The token endpoint returns `invalid_request`, `invalid_grant`, `invalid_client`, `unauthorized_client`, or `invalid_scope`, as applicable.

A missing, malformed, or invalid DPoP proof is handled according to {{RFC9449}}, including the use of `use_dpop_nonce` when the authorization server requires a nonce. An unsupported `delegation=true` request SHOULD result in `invalid_request` unless a more specific existing OAuth error applies.

# Delegating Authorization to Another Client

## Preconditions

A delegating client requires:

* a valid Delegated Authorization Token Chain whose leaf token is bound to a key it controls;
* an effective delegation depth greater than zero;
* a public key established through an authenticated mechanism as associated with the intended delegate client; and
* sufficient authorization for the proposed child token.

The delegating client SHOULD validate the chain before relying on it. It MUST NOT issue a child token after the effective expiration time, before the effective not-before time, or when further delegation is prohibited.

## Constructing a Client-Issued Delegated Authorization Token

The delegating client constructs a JWT payload containing `cnf.jkt` for the delegate client's public key and restrictions for the delegation. It calculates inherited effective values from the current chain before selecting child values. The child MUST contain a non-empty `scope`, a non-empty `authorization_details`, or both.

Omission of `scope` or `authorization_details` discards that permission component. Omission of another restriction has the claim-specific effect defined in Section 5.3 and does not create unrestricted authority. The delegating client MUST NOT include `iss`, `sub`, `iat`, or `jti`. The delegating client MUST NOT include an extension claim unless its defining specification permits the claim in Client-Issued Delegated Authorization Tokens and the delegating client applies the delegation rules defined for that claim.

## Binding the Token to the Delegate Client

The delegating client MUST use a mechanism appropriate to the relationship and threat model of the clients to establish that the public key is associated with the intended delegate client. It computes the established key's thumbprint according to {{RFC7638}} and places it in the child token's `cnf.jkt`.

This specification does not require a separate proof-of-possession exchange during local issuance when the mechanism used to establish that association already establishes control of the corresponding private key, or when key substitution is not a relevant threat. Otherwise, the delegating client SHOULD obtain such proof. Regardless of the issuance mechanism, the delegate client MUST prove possession of the private key when using the token at a resource server or delegating further.

## Signing the Client-Issued Delegated Authorization Token

The delegating client creates a protected JOSE header containing `typ`, `alg`, and its public `jwk`, then signs the child token with the private key bound to the current leaf token. The JWK thumbprint of the protected `jwk` MUST equal the current leaf token's `cnf.jkt`.

The delegating client appends the child token in compact form to the existing chain. It MUST preserve the preceding compact tokens byte for byte and in their existing order.

## Delegation Depth Restrictions

The delegating client computes the parent's effective depth as specified in Section 5.3.2. If it is zero, the delegating client MUST NOT create a child token. If it is a positive integer `m`, an explicit child value MUST be between zero and `m - 1`, inclusive; omission yields `m - 1`.

A delegating client SHOULD impose the smallest depth sufficient for the intended workflow. When the delegating client intends to issue a terminal token that can be used for protected resource access but cannot be delegated further, it SHOULD set the child token's `max_delegation_depth` to `0`.

## Permission Downscoping

For `scope`, the delegating client treats the scope values as a set. If the child contains `scope`, every included scope value MUST occur in the parent's effective scope. If no effective parent scope exists, the child MUST NOT contain `scope`. Omission discards the scope permission component.

For `authorization_details`, the delegating client applies the subset rules defined for each authorization detail type. If the child contains `authorization_details`, every child detail MUST be covered by an effective parent detail. If no effective parent authorization details exist, the child MUST NOT contain `authorization_details`. If the delegating client cannot establish containment, it MUST NOT issue the child token. Omission discards the authorization-details permission component.

The child MUST retain at least one non-empty permission component. When both claims are present, the delegating client MUST apply the deterministic composition semantics applicable to the authorization detail types and intended resource servers. It MUST NOT issue a child if removing or narrowing either component could increase the child's effective authorization.

## Audience and Lifetime Restrictions

Every audience explicitly included in the child MUST occur in the parent's effective audience. Delegating clients that include an explicit child audience SHOULD select the narrowest subset appropriate for the resource servers that the delegate client is expected to access.

A child token's `exp` value MUST be no later than the parent's effective expiration time. If the parent has an effective not-before time, a child token's `nbf` value, when present, MUST be no earlier than that time. If the parent has no effective not-before time, the child MAY contain an `nbf` value. Omitting `nbf` imposes no additional not-before restriction. The delegating client SHOULD choose a short lifetime appropriate to the delegated task.

## Out-of-Band Delivery

The delegating client delivers the complete ordered chain, including the new child token, to the delegate client. The protocol for delivering the delegate public key, requesting delegation, and returning the chain is outside the scope of this specification.

An out-of-band mechanism MUST provide integrity protection and endpoint authentication appropriate to the sensitivity of the authorization. It SHOULD also provide confidentiality because token contents can reveal identities, resources, and permissions, even though possession of the private key is required to use the tokens.

## Delegate Client Validation

A delegate client receiving a chain SHOULD perform the validation in Section 8 before accepting it. At a minimum, it MUST confirm that the leaf `cnf.jkt` matches the thumbprint of its own public key before attempting to use the chain. It SHOULD also confirm that the effective permissions, audience, lifetime, and depth are suitable for the intended task.

# Delegated Authorization Token Chain Validation

## Validation Inputs

The inputs to the validation procedure are the exact serialized chain, trusted authorization server configuration, the current time, the accepted algorithms, and the authorization context in which validation is performed. For protected resource access, the context includes the target resource server and requested operation.

The verifier splits the serialization on `~` without modifying any compact token. It MUST reject an empty chain, an empty element, an element that is not a well-formed compact signed JWT, or a chain exceeding configured size or length limits.

## Locating the Authorization-Server-Issued Root Token

The first element is the root token. The verifier uses its `iss` claim and trusted configuration to identify an acceptable authorization server and its verification keys. The verifier MUST NOT accept a client-issued token as the first element or search later elements for an alternative root.

Only the first token can contain `iss`, `sub`, `iat`, or `jti`. The presence of any of these claims in a later token causes validation to fail.

## Validating the Authorization-Server-Issued Token

The verifier MUST reject a root token containing a `jwk` header parameter. It MUST validate the root token's JWS signature using an allowed authorization server key and algorithm. It MUST verify the protected `typ` header parameter, `iss`, `cnf.jkt`, the time claims, and all other claims required by authorization server policy. It MUST reject the root token if the issuer is untrusted, the signature is invalid, or the token is not currently valid.

Issuer identification alone is not sufficient to trust a key supplied by the token. Authorization server verification keys MUST come from trusted configuration or authenticated metadata.

## Validating Client-Issued Token Signatures

For each token after the root, in order, the verifier MUST:

1. parse the protected header and payload;
2. require the protected `typ` header parameter to equal `da+jwt`;
3. require a public `jwk` in the protected header;
4. ensure that `alg` and the JWK are allowed and compatible;
5. reject private key material in the JWK; and
6. verify the signature using that JWK.

A failure at any level invalidates the entire chain.

## Validating Key Continuity

For each adjacent parent and child, the verifier computes the SHA-256 JWK thumbprint of the child's protected `jwk` according to {{RFC7638}}. It MUST compare that value to the parent's `cnf.jkt` using a constant-time comparison. They MUST be equal.

The verifier MUST also require every child token to contain a syntactically valid `cnf.jkt`. This value establishes continuity to the next token, if any, or, for the leaf token, identifies the key to which the token is bound.

## Validating Delegation Depth

The verifier starts with the depth explicitly specified by the root token and MUST reject a root that omits the claim. For each child:

* if the parent's effective depth is zero, the verifier MUST reject the chain; or
* if the parent's effective depth is a positive integer `m`, an explicit child value MUST be in `[0, m-1]`, and omission produces `m-1`.

The verifier MUST reject a value outside the range 0 to 65535, a non-integer value, or a value that exceeds its configured maximum.

## Validating Permission Downscoping

The verifier initializes the effective scope and authorization details from the root token and MUST reject a root that contains neither a non-empty `scope` nor a non-empty `authorization_details`. While traversing the chain, it MUST reject a child that contains neither a non-empty `scope` nor a non-empty `authorization_details`. A permission claim present in a child becomes effective only after the verifier establishes that it is no broader than the corresponding effective parent value. If the child omits a permission claim, that component becomes absent from the effective authorization and MUST NOT reappear in a descendant. The verifier MUST reject a child that contains a permission claim for which its parent has no corresponding effective value.

Scope containment is determined by set inclusion. Containment of authorization details is determined by type-specific rules. If the comparison is unsupported, ambiguous, or unsuccessful, the verifier MUST reject the chain rather than assume containment.

When `scope` and `authorization_details` are combined, the verifier MUST apply the applicable deterministic composition semantics. It MUST reject the chain if those semantics are unavailable or if omission or downscoping of either component could increase effective authorization.

## Validating Extension Claims

For each extension claim in a Client-Issued Delegated Authorization Token, the verifier MUST apply the delegation semantics defined for that claim. These semantics determine whether the claim can be introduced or reintroduced, how omission is processed, whether a child value must be compared with an effective parent value, and how the claim affects authorization. If the required semantics are not defined or implemented, or if the claim does not comply with them, the verifier MUST reject the chain.

An unrecognized extension claim in the root token causes validation to fail unless trusted authorization server configuration or an applicable deployment profile explicitly permits that claim to be ignored. A verifier that ignores such a claim MUST NOT use it as a source of authorization or as an authorization restriction.

## Validating Audience Downscoping

The verifier treats the value of each `aud` claim as a set, including when the claim uses the single-string form defined by {{RFC7519}}. Whenever present, an `aud` claim MUST identify at least one audience. The verifier MUST reject a root token that omits `aud`. A child that omits `aud` adds no audience restriction, so the effective audience remains the parent's effective audience. Every audience explicitly included in a child MUST be a member of its parent's effective audience.

For protected resource access, the effective leaf audience MUST identify the resource server according to its audience-validation policy.

## Validating Time Restrictions

The verifier maintains the earliest expiration time and the latest not-before time encountered. A token that omits `nbf` adds no not-before restriction. The chain is invalid if a child's expiration time is later than its parent's effective expiration time or, when the parent has an effective not-before time, if the child's explicit not-before time is earlier than that value.

The verifier MUST ensure that the current time is before the effective expiration time and, when an effective not-before time exists, is not before that time, allowing only locally configured clock skew. It MUST also apply any additional time-claim policies required for the root token.

## Validation Result

Successful validation produces at least:

* the trusted authorization server and the validated identity claims from the root token;
* the leaf `cnf.jkt`;
* effective scope and authorization details;
* effective audience;
* an effective expiration time and, when present, an effective not-before time; and
* effective remaining delegation depth.

Chain validation alone does not establish that a protected resource request is authorized. The resource server also validates the DPoP proof, request binding, audience, and application-specific permissions as described in Section 9.


# Protected Resource Access

## Protected Resource Metadata

A resource server can publish its Delegated Authorization algorithm capabilities using OAuth Protected Resource Metadata as defined by {{RFC9728}}. The `dpop_signing_alg_values_supported` parameter defined by {{RFC9728}} identifies the algorithms accepted by the resource server for DPoP proofs.

This specification defines the following protected resource metadata parameter:

**delegated_authorization_client_signing_alg_values_supported**:
: OPTIONAL. A JSON array containing the JWS `alg` values accepted by the resource server for Client-Issued Delegated Authorization Token signatures. The array MUST NOT contain `none` or a symmetric MAC algorithm.

A client issuing a Client-Issued Delegated Authorization Token SHOULD select an algorithm listed by every resource server expected to validate the chain. When a chain has multiple intended resource servers, the client selects an algorithm from the intersection of their advertised values or uses an algorithm established through configuration or another authenticated mechanism. If the metadata parameter is omitted, the client MUST NOT infer support for any particular client-issued-token algorithm from the omission alone.

A resource server MUST apply its current local algorithm policy when validating a token or DPoP proof. Metadata does not override that policy or by itself require acceptance of a token.

## The `DA` Authentication Scheme

A client presents a Delegated Authorization Token Chain in the HTTP `Authorization` field using the `DA` authentication scheme. The scheme name is followed by one or more SP characters and the Delegated Authorization Token Chain. The chain is a syntactically restricted instance of the `token68` form defined by {{RFC9110}}.

As specified by {{RFC9110}}, the authentication scheme name is case-insensitive. A request using the `DA` scheme MUST contain exactly one `Authorization` field value with exactly one credential. A resource server MUST reject a request containing more than one `Authorization` field value or a combined field value containing more than one credential.

To extract the token chain, the resource server removes the scheme name and the one or more SP characters that immediately follow it. All remaining characters form the Delegated Authorization Token Chain. The chain MUST be non-empty, MUST NOT contain whitespace, and MUST conform to the chain serialization defined in Section 5.6. Leading or trailing whitespace in the chain is invalid.

The token-chain value is case-sensitive and MUST appear in the exact root-to-leaf order defined in Section 5.6. The `DA` scheme MUST be used with a DPoP proof and MUST NOT be used as a bearer authentication scheme.

## Compatibility with the Bearer Authentication Scheme

A Delegated Authorization Token MUST NOT be accepted as a bearer token. A client MUST present a Delegated Authorization Token Chain using the `DA` authentication scheme and MUST include the DPoP proof required by this specification.

A resource server receiving a Delegated Authorization Token or Delegated Authorization Token Chain using the `Bearer` authentication scheme MUST reject the request. It MUST also reject a bearer access token presented using the `DA` scheme. Validation rules for `DA` credentials and bearer credentials MUST be mutually exclusive so that a token accepted under one scheme cannot be accepted under the other.

An authorization server MUST issue a Delegated Authorization Token with the `DA` token type and MUST NOT issue the same token value for use as a bearer token. These requirements prevent downgrade to bearer-token semantics and ensure that possession of a serialized Delegated Authorization Token Chain without the leaf private key is insufficient for access.

## Creating the DPoP Proof

This specification uses the DPoP proof format and validation rules defined by {{RFC9449}}, except for the following substitutions for protected resource requests:

* the `DA` authentication scheme defined in Section 9.2 replaces the `DPoP` authentication scheme;
* the Delegated Authorization Token Chain replaces the single access token; and
* the `ath` claim is computed over the exact Delegated Authorization Token Chain serialization as specified below, rather than over a single access token.

All other applicable DPoP proof requirements, including the protected `typ` and `jwk` header parameters, signature algorithm requirements, `htm`, `htu`, `iat`, `jti`, proof freshness, replay detection, and nonce processing, remain unchanged unless this specification explicitly states otherwise.

The client creates a DPoP proof using these substitutions. The proof MUST contain `htm`, `htu`, `iat`, `jti`, and `ath`, and it MUST be signed with the private key identified by the leaf token's `cnf.jkt`. The proof includes the corresponding public JWK in its protected header.

For this scheme, the `ath` value is:

~~~
base64url(SHA-256(ASCII(DA-Token-Chain)))
~~~

where `DA-Token-Chain` is the exact character sequence extracted by the parsing procedure in Section 9.2. The scheme name and separating SP characters are not included. The resource server MUST NOT decode, normalize, or reserialize the chain or any token before computing `ath`. This binds the proof to every token in the chain and to the order of those tokens. DPoP nonce processing follows {{RFC9449}}.

## Protected Resource Request

A request carries both header fields. For example:

~~~http-message
GET /reports/123 HTTP/1.1
Host: resource.example.com
Authorization: DA eyJ...root...~eyJ...leaf...
DPoP: eyJ...proof...
~~~

A client MUST create a fresh DPoP proof for the target URI and method. It MUST NOT forward a proof created for a different request or use a private key other than the one bound to the leaf token.

## Resource Server Processing

Upon receiving a `DA` request, the resource server MUST:

1. parse the authentication scheme and exact chain serialization;
2. validate the Delegated Authorization Token Chain as specified in Section 8;
3. validate the DPoP proof as specified by {{RFC9449}} and this section;
4. verify that the DPoP proof key matches the leaf token's key binding;
5. verify `ath` over the exact chain serialization;
6. verify that the effective audience identifies this resource server; and
7. enforce effective scope, authorization details, and local access-control policy.

The resource server MUST complete all checks before fulfilling the protected resource request. It MAY cache validation results, subject to token expiration, key rotation, revocation policy, and replay defenses.

## Delegated Authorization Token Chain Verification

The resource server supplies the validation procedure in Section 8 with its trusted issuers, accepted algorithms, resource identifier, current time, and request context. It MUST reject the entire request if any token or adjacent relationship is invalid.

A resource server MAY use a trusted validation service to perform some or all chain checks. The protocol and response format for such a service are outside the scope of this specification. The validation result MUST include the effective restrictions and the leaf key binding. Regardless of how the chain is validated, the resource server MUST perform the request-specific DPoP and authorization checks in this section.

## DPoP Proof Verification

In addition to the checks required by {{RFC9449}}, the resource server MUST require an asymmetric proof signature, validate `htm` and `htu` against the current request, enforce an acceptable proof age, detect replay of `jti` as required by its policy, and validate a server-provided nonce when one was required.

The resource server MUST reject a missing DPoP proof. It MUST NOT accept a DPoP proof as defined by {{RFC9449}} that was created for a token sent under another authentication scheme unless all requirements of this specification, including the adapted `ath`, are met.

## Binding the DPoP Proof to the Leaf Delegated Authorization Token

The resource server computes the JWK thumbprint of the public JWK in the DPoP proof's protected header according to {{RFC7638}}. That value MUST equal the leaf token's `cnf.jkt`. The resource server also computes `ath` from the exact presented chain and compares it to the proof's `ath` using a constant-time comparison.

Together, these checks establish that the client presenting the request controls the private key bound to the leaf token and used it to authorize the request with this exact ordered chain. They do not replace validation of the signatures and downscoping relationships within the chain.

## Authorization Enforcement

After cryptographic validation, the resource server determines whether the effective authorization permits the requested operation. It MUST apply every effective restriction, including scope, authorization details, audience, time, and any application-specific claims recognized by its policy.

The resource server MUST ensure that its composition of `scope` and `authorization_details` is monotonic, such that removing or narrowing either permission component cannot authorize an operation that was not previously authorized.

Possession of a valid chain and its bound private key does not guarantee access. The resource server can deny a request due to resource owner policy, tenant policy, resource state, revoked authorization, or other local controls.

## `WWW-Authenticate` Challenges

A resource server challenges a client to use this scheme by returning a `WWW-Authenticate` header field with the `DA` scheme. The challenge MAY include `realm`, `error`, `error_description`, and `scope` parameters when applicable. Error descriptions are intended for developers and MUST NOT expose sensitive details.

For example:

~~~http-message
HTTP/1.1 401 Unauthorized
WWW-Authenticate: DA realm="reports", error="invalid_token"
~~~

When a fresh DPoP nonce is required, the resource server returns an HTTP 401 response containing the `DPoP-Nonce` header field and a `DA` challenge whose `error` parameter is `use_dpop_nonce`. This adapts the resource server nonce challenge defined by {{RFC9449}} to the `DA` authentication scheme.

~~~http-message
HTTP/1.1 401 Unauthorized
WWW-Authenticate: DA error="use_dpop_nonce"
DPoP-Nonce: eyJ7S_zG.eyJH0-Z.HX4w-7v
~~~

The client MUST create a new DPoP proof containing the nonce and retry the request with the same Delegated Authorization Token Chain. It MUST NOT reuse the rejected DPoP proof.

## Error Responses

The `DA` authentication scheme uses the `invalid_request`, `invalid_token`, and `insufficient_scope` error codes defined by {{RFC6750}}, with the adaptations specified in this section. These error codes are carried in a `DA` challenge; a resource server MUST NOT use the `Bearer` scheme for a Delegated Authorization Token.

* `invalid_request` indicates that the request is missing required authentication information or contains malformed DA credentials or a malformed DPoP proof;
* `invalid_token` indicates that the token chain, a signature, key continuity, a time restriction, the audience, or the DPoP binding is invalid; and
* `insufficient_scope` indicates that the chain and proof are valid but the effective authorization does not permit the requested operation.

A resource server MUST return HTTP 401 for `invalid_request` and `invalid_token`. It MUST return HTTP 403 for `insufficient_scope`. It SHOULD avoid identifying which internal chain element failed, because detailed failures can aid attackers.

# Token Revocation

## Revocation Request

This specification extends OAuth 2.0 Token Revocation {{RFC7009}} to accept a complete Delegated Authorization Token Chain. The revocation endpoint advertised by the authorization server accepts the following form parameters:

**token**:
: REQUIRED. The exact serialization of a Delegated Authorization Token Chain. The last token in the chain is the token to be revoked. A single-token chain therefore revokes an Authorization-Server-Issued Delegated Authorization Token, while a chain containing two or more tokens can revoke the last Client-Issued Delegated Authorization Token.

**token_type_hint**:
: OPTIONAL. When present, its value SHOULD be the case-sensitive string `delegated_authorization_token`.

The request MUST include a DPoP proof in the `DPoP` header field. The proof MUST contain `htm`, `htu`, `iat`, `jti`, and `ath`. The `ath` value MUST be:

~~~
base64url(SHA-256(ASCII(DA-Token-Chain)))
~~~

where `DA-Token-Chain` is the exact value of the `token` parameter after form decoding. The authorization server MUST NOT decode, normalize, or reserialize the chain or any token before computing `ath`.

The request and response otherwise follow {{RFC7009}}. In particular, the request uses an HTTP `POST`, the authorization server MUST require TLS, and the endpoint returns an HTTP 200 response for both a successful revocation and an invalid token.

## Revocation Authorization

The authorization server MUST validate the DPoP proof according to {{RFC9449}}, including its signature, `htm`, `htu`, `iat`, `jti`, `ath`, and nonce when a nonce is required. The proof MUST be bound to the revocation endpoint URI and the `POST` method.

After validating the chain as specified in Section 10.3, the authorization server MUST require the DPoP proof key to match one of the following keys:

* the key identified by the target token's `cnf.jkt`; or
* for a Client-Issued Delegated Authorization Token, the public `jwk` in that target token's protected header.

A match means that the JWK thumbprint of the DPoP proof's public key, computed according to {{RFC7638}}, is equal to the applicable thumbprint. Successful chain and DPoP validation under either rule is sufficient authorization to revoke the target token. This allows the client controlling the key identified by the target token's `cnf.jkt` to revoke it and allows the delegating client that issued a client-issued target token to revoke its direct child.

OAuth client authentication, if performed, does not by itself authorize revocation. Possession of a serialized chain without proof of possession of an applicable private key is insufficient.

## Revocation Validation

Revocation validation establishes the cryptographic provenance and downscoping validity of the target; it does not determine whether the chain is currently usable for protected resource access.

Before recording a revocation, the authorization server MUST:

1. parse the exact chain serialization and enforce configured token-count, size, and verification-cost limits;
2. require the root `iss` to identify that authorization server and validate the root signature using a trusted key;
3. validate the required token types, claim syntax, prohibited claims, and confirmation bindings;
4. validate every client-issued token signature and every adjacent key-continuity relationship;
5. validate delegation depth, permission containment, audience downscoping, and time-claim downscoping; and
6. apply the extension-claim delegation rules required to establish that the chain is valid.

The authorization server MUST reject the chain internally if any required cryptographic or downscoping check fails. Consistent with {{RFC7009}}, this failure does not change the external HTTP 200 response. If an authorization detail or extension claim requires containment rules that the authorization server does not implement, it cannot establish a valid chain and MUST NOT record the target as a valid revocation entry.

The current time being after an `exp` value or before an `nbf` value MUST NOT prevent revocation. Prior revocation of the target or any ancestor MUST NOT cause an error response. The authorization server MAY treat such a request as already effective and need not record redundant descendant state. Audience applicability and authorization for a particular protected resource operation are not evaluated during revocation validation.

The authorization server MAY accept a legacy signature algorithm for revocation validation after it has stopped accepting that algorithm for newly issued tokens or protected resource access, provided that the server can still verify it securely. It MUST NOT accept an algorithm whose signature verification is considered unsafe under current policy.

The server MUST identify the target using a collision-resistant digest of its exact compact serialization or another unambiguous identifier. It MUST NOT identify a Client-Issued Delegated Authorization Token only by its position, claims, or bound-key thumbprint.

## Revocation Semantics

Revoking an Authorization-Server-Issued Delegated Authorization Token invalidates every chain rooted in that token. Revoking a Client-Issued Delegated Authorization Token invalidates every chain containing that token, including every chain in which another token has been appended after it. Revocation of one client-issued token does not by itself revoke its parent, its siblings, or another token bound to the same key. An authorization server accepting revocation of a client-issued token MUST retain sufficient status information to recognize that token when it appears at any position in a subsequently evaluated chain.

A successful revocation request records revocation state at the authorization server; it does not by itself deliver that state to resource servers performing offline validation. This specification does not define a revocation-status distribution mechanism. Resource servers performing offline validation cannot learn of a revocation until they receive that status through a mechanism defined outside this specification. Short token lifetimes bound exposure when no online or promptly updated status mechanism is available.

# Privacy Considerations

Delegated Authorization Tokens can reduce authorization server visibility. After issuing the root token, the authorization server need not observe each client-to-client delegation or protected resource request. This prevents the authorization server from automatically learning every delegate client, downstream resource, access time, or multi-hop call pattern.

The ordered chain is visible to every resource server to which it is presented. Claims in the root and intermediate tokens can disclose the authorization server, subject, client relationships, audiences, permissions, and workflow structure. Clients SHOULD disclose a chain only to an intended audience, SHOULD minimize claims and chain length, and SHOULD use TLS for every transmission.

A stable `cnf.jkt`, root `jti`, or other persistent claim can enable correlation across requests and resource servers. Token issuers SHOULD avoid unnecessary identifiers and SHOULD use audience-specific, short-lived child tokens where unlinkability is important. DPoP proofs contain public keys and unique proof identifiers; implementations SHOULD follow the privacy guidance in {{RFC9449}}.

Local delegation changes the entities that have audit visibility; it does not eliminate such visibility. Deployments requiring delegation-event records can maintain privacy-conscious logs at delegating clients and resource servers. Such logs MUST be protected because complete chains and authorization details can contain sensitive information.

# Security Considerations

This specification extends the OAuth threat model by allowing clients to issue constrained child tokens. Implementations MUST follow OAuth security best current practice {{RFC9700}}, JSON Web Token best current practice {{RFC8725}}, and the DPoP security considerations in {{RFC9449}}.

## Resource Owner Authorization and Delegation

Local delegation changes the set of parties that can exercise an authorization without another interaction with the authorization server. For grants involving a resource owner, the authorization server MUST distinguish authorization for protected resource access from authorization for client-issued delegation.

The authorization server determines how delegation is presented to the resource owner. The presentation SHOULD make clear that the client can authorize other clients without another interaction with the authorization server or resource owner. It SHOULD also communicate the permissions, audiences, lifetime, and maximum delegation depth covered by the authorization decision, and SHOULD avoid implying that the authorization server will observe or approve each child issuance.

A client MUST NOT represent possession of a Delegated Authorization Token as resource owner approval of a particular delegate client beyond the authorization and depth encoded in the validated chain.

## Protection of Client Private Keys

A client private key authorizes both use of the current leaf token and creation of child tokens when delegation remains available. Clients MUST protect such keys in a manner appropriate to the granted authority, restrict signing operations to authorized code, and erase obsolete keys when operationally possible. Private JWK members MUST never appear in a token or DPoP protected header.

Using one key for token issuance and DPoP simplifies continuity but increases the impact of compromise. Implementations SHOULD use non-exportable keys and narrowly scoped signing APIs that distinguish token-signing and proof-signing inputs.

## Compromise of a Client Key

An attacker controlling a bound private key can use any unexpired chain ending at that key and can issue child tokens within its effective authorization and delegation depth. Short token lifetimes, finite depth, narrow audiences, minimal permissions, key rotation, and revocation reduce this exposure.

Revoking only a child key does not invalidate signatures already created by an uncompromised parent unless the deployment has a revocation mechanism that resource servers consult. Incident response should identify all affected chains and their descendants.

## Delegated Authorization Token Replay

A stolen chain cannot be used without the leaf private key, but replay remains possible after simultaneous compromise of chain and key or through a signing oracle. Resource servers MUST require DPoP, validate `ath`, enforce proof freshness, and apply replay detection as described by {{RFC9449}}. High-value operations can additionally require server nonces or application-level transaction binding.

## DPoP Proof Replay

A DPoP proof is bound to the HTTP method, target URI, complete chain, and signing key. Resource servers MUST validate those bindings and SHOULD store recently seen `jti` values for the proof acceptance window. They SHOULD use DPoP nonces where clock skew, pre-generated proofs, or active replay are material threats.

## Algorithm Confusion

Verifiers MUST enforce an allowlist of asymmetric algorithms, ensure algorithm and key-type compatibility, reject `none` and symmetric MACs, enforce the canonical ECDSA signature requirement in Section 5.2.3, and obtain root verification keys only from trusted authorization server configuration. A token-supplied client JWK is trusted only after its thumbprint is matched to the preceding `cnf.jkt`; it MUST NOT be treated as an independently trusted identity key.

## Token Chain Substitution

A client-issued token does not contain a parent identifier. Consequently, it can be valid in any chain in which the preceding token binds the same signing key and the effective restrictions of the chain encompass the child token's restrictions. This is an intentional property of the initial format.

The DPoP `ath` over the exact complete chain prevents an attacker from substituting, removing, adding, or reordering tokens in a captured protected resource request without obtaining a new proof from the leaf key. Verifiers MUST validate every adjacent pair and MUST NOT accept an unordered set of tokens.

A delegating client that needs a child token to be usable only with one particular parent chain MUST use distinct parent keys or apply an application-specific restriction. Future specifications can define stronger parent binding.

## Invalid Downscoping

The security of local delegation depends on rejecting any expansion of authorization. Verifiers MUST compute effective restrictions from the root in order and fail closed when containment cannot be established. Omission of `scope` or `authorization_details` discards that permission component for the child and its descendants; omission of other restrictions has the claim-specific effect defined in this specification.

Specifications defining authorization detail types SHOULD define deterministic semantic subset rules. Comparing raw JSON text or merely matching `type` values is generally insufficient.

## Excessive or Unbounded Delegation Depth

Every root token has a finite `max_delegation_depth`, but this does not replace implementation limits. Authorization servers MUST enforce a configured maximum depth, delegating clients SHOULD minimize remaining depth, and every implementation MUST limit chain count and serialized size. Such limits mitigate denial of service from expensive signature and authorization comparisons.

## Audience Confusion

Authorization servers MUST set an explicit root audience. A client that explicitly narrows `aud` in a child SHOULD select the smallest audience set suitable for the delegated task. Resource servers MUST verify that they are an effective audience.

## Authorization Details Comparison

An authorization detail can have semantics that are not evident from its generic JSON structure. The specification registering or defining its `type` SHOULD define containment for delegation. A verifier that does not implement that rule MUST reject a chain that requires the comparison. Implementations MUST account for omitted members, defaults, arrays, wildcards, resource identifiers, and actions according to type semantics.

## Revocation Security

Offline validation does not provide prompt revocation by itself. A successful revocation request records state at the authorization server but does not ensure that a resource server has received that state. Deployments requiring prompt revocation MUST use short-lived tokens together with an online or promptly updated status-distribution mechanism defined outside this specification.

Revocation endpoints and status-distribution mechanisms can reveal subjects, authorization relationships, and delegation topology. They MUST authenticate authorized participants, protect integrity and confidentiality as applicable, and limit information disclosure.

The DPoP proof on a revocation request prevents a party that merely observes a chain, including a resource server, from revoking its target. Authorization servers MUST apply DPoP freshness, replay-detection, and nonce requirements as they do for other DPoP-protected endpoint requests.

# Operational Considerations

## Key Rotation

Authorization server signing-key rotation follows normal JWT and JWKS practices. Old verification keys need to remain available for as long as valid root tokens signed with them can be presented.

A client rotates its bound key by obtaining or receiving a new token whose `cnf.jkt` identifies the new key. Existing tokens cannot be rewritten. During a transition, a client might need to retain old private keys until all corresponding chains expire or are revoked. Consistent with Section 4.3, clients SHOULD generate a distinct bound key for each token; limited key reuse SHOULD be confined to tokens in the same usage scenario.

## Token Lifetimes

Authorization servers MUST issue finite root lifetimes and enforce a configured maximum lifetime. They SHOULD use the shortest lifetime suitable for the delegated task, and delegating clients SHOULD shorten lifetimes further to match delegated tasks. Long-lived root tokens magnify key compromise and revocation risks. Effective expiration is the earliest expiration in the chain, regardless of omitted child claims.

## Token Size

Every delegation appends another compact JWT, so chain size grows linearly with depth. Authorization details and public JWK headers can be substantial. Implementations need to account for HTTP header limits imposed by clients, proxies, gateways, and servers. A deployment can require a smaller maximum than the protocol syntax permits.

## Chain Length Limits

Every parser and verifier MUST enforce limits on the token count, total serialized size, individual token size, and signature-verification cost. It SHOULD reject an oversized chain before performing expensive cryptographic work. Deployments SHOULD document these limits so that clients can avoid unusable delegation paths.

## Caching and Revocation

A resource server MAY cache validated signatures and effective restrictions keyed by a cryptographic digest of the exact chain. Cache entries MUST be invalidated or expire according to the effective expiration time, authorization server key status, local policy changes, and revocation information available through any configured status mechanism. DPoP proof freshness and replay checks are per request and MUST NOT be bypassed by chain-validation caching.

## Logging and Audit

Useful audit events include root issuance, local delegation, chain validation outcome, effective authorization, and protected resource decisions. Logs SHOULD identify chains by a one-way digest rather than store complete credentials. Implementations MUST redact `Authorization` and `DPoP` header values from routine logs and protect any recorded authorization details or subject identifiers.

# IANA Considerations

## Delegated Authorization Token Media Type

This specification requests registration of the following media type in the "Media Types" registry:

* Type name: application
* Subtype name: da+jwt
* Required parameters: n/a
* Optional parameters: n/a
* Encoding considerations: 8bit; Delegated Authorization Token values are encoded as a series of base64url-encoded values separated by period (`.`) characters
* Security considerations: Section 12 of this document
* Interoperability considerations: n/a
* Published specification: this document
* Applications that use this media type: OAuth clients, authorization servers, and resource servers
* Fragment identifier considerations: n/a
* Additional information: n/a
* Person and email address to contact for further information: IETF OAuth Working Group, oauth@ietf.org
* Intended usage: COMMON
* Restrictions on usage: n/a
* Author: IETF
* Change controller: IETF

The `typ` value `da+jwt` is the media type without the `application/` prefix, as recommended by {{RFC7515}}.

## OAuth Access Token Type

This specification requests registration in the "OAuth Access Token Types" registry:

* Name: `DA`
* Additional Token Endpoint Response Parameters: n/a
* HTTP Authentication Scheme(s): `DA`
* Change Controller: IETF
* Reference: this document

## OAuth Token Type Hint

This specification requests registration in the "OAuth Token Type Hints" registry. Following the naming convention of existing values such as `access_token` and `refresh_token`, the registered value uses lowercase words separated by underscores:

* Hint Value: `delegated_authorization_token`
* Change Controller: IETF
* Reference: this document

The hint indicates that the `token` parameter contains a complete Delegated Authorization Token Chain in the serialization defined by this specification.

## HTTP Authentication Scheme

This specification requests registration in the "HTTP Authentication Scheme Registry":

* Authentication Scheme Name: `DA`
* Reference: this document
* Notes: Presents an ordered Delegated Authorization Token Chain and requires a DPoP proof.

## JWT Claims Registration

This specification requests registration in the "JSON Web Token Claims" registry:

* Claim Name: `max_delegation_depth`
* Claim Description: Maximum number of further delegated authorization levels, from 0 to 65535
* Change Controller: IESG
* Specification Document(s): this document

The `cnf`, `scope`, `aud`, `exp`, `nbf`, `iss`, `sub`, `iat`, and `jti` names are already registered and are not re-registered by this document.

## OAuth Parameters Registration

This specification requests registration in the "OAuth Parameters" registry:

* Name: `delegation`
* Parameter Usage Location: authorization request, token request
* Change Controller: IETF
* Reference: this document

The parameter value `true` requests issuance of a Delegated Authorization Token.

This specification also requests registration of the following parameter:

* Name: `max_delegation_depth`
* Parameter Usage Location: authorization request, token request
* Change Controller: IETF
* Reference: this document

The parameter optionally requests an exact finite delegation depth.

This specification also requests registration in the "OAuth Authorization Server Metadata" registry:

* Metadata Name: `delegated_authorization_token_supported`
* Metadata Description: Boolean indicating support for issuance of Delegated Authorization Tokens
* Change Controller: IESG
* Specification Document(s): this document

and:

* Metadata Name: `delegated_authorization_signing_alg_values_supported`
* Metadata Description: JWS algorithms supported for Authorization-Server-Issued Delegated Authorization Tokens
* Change Controller: IESG
* Specification Document(s): this document

and:

* Metadata Name: `delegated_authorization_max_depth`
* Metadata Description: Greatest delegation depth the authorization server can issue
* Change Controller: IESG
* Specification Document(s): this document

## Protected Resource Metadata Registration

This specification requests registration in the "OAuth Protected Resource Metadata" registry:

* Metadata Name: `delegated_authorization_client_signing_alg_values_supported`
* Metadata Description: JWS algorithms accepted for Client-Issued Delegated Authorization Token signatures
* Change Controller: IETF
* Specification Document(s): this document

--- back

# Protocol Flow Examples

## Authorization Server Issues a Delegated Authorization Token

The client creates a DPoP proof with its key and requests a token after receiving an authorization code:

~~~http-message
POST /token HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2Iiwiam...

grant_type=authorization_code&
code=SplxlOBeZQQYbYS6WxSbIA&
redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb&
delegation=true&
max_delegation_depth=2
~~~

After validating the grant and proof, the authorization server returns a root token bound to the proof key:

~~~http-message
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "access_token": "eyJ0eXAiOiJkYStqd3QiLCJhbGciOiJFUzI1NiJ9...",
  "token_type": "DA",
  "expires_in": 3600,
  "scope": "records:read records:write"
}
~~~

## One-Level Client Delegation

Client A holds the root token, whose `cnf.jkt` identifies Client A's key. After authenticating Client B's public key, Client A:

1. computes the thumbprint of Client B's key;
2. creates a child with `scope` narrowed to `records:read`;
3. puts Client A's public JWK in the protected header;
4. signs the child with Client A's private key; and
5. delivers `root-token~child-token` to Client B.

The thumbprint of the child's protected `jwk` equals the root's `cnf.jkt`, while the child's `cnf.jkt` identifies Client B's key.

## Multi-Level Client Delegation

Client B can repeat the operation for Client C when the child token's effective depth permits it. The resulting chain is:

~~~
root-token~client-a-child-token~client-b-child-token
~~~

At each boundary:

~~~
thumbprint(child protected header jwk) = parent payload cnf.jkt
~~~

The final child in this example has `max_delegation_depth` equal to zero. Client C can use it to access the protected resource but cannot create another valid child.

## Protected Resource Request with DPoP

Client C computes `ath` over the exact ASCII chain serialization, creates a DPoP proof for the request, and signs it with the key identified by the leaf `cnf.jkt`:

~~~http-message
GET /records/123 HTTP/1.1
Host: api.example.com
Authorization: DA eyJ...root...~eyJ...child-a...~eyJ...child-b...
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2Iiwiam...
~~~

The resource server validates the root signature, both client signatures, both key-continuity relationships, all cumulative restrictions, the leaf key binding, and the DPoP request binding.

# Delegated Authorization Token Examples

The following examples show decoded protected headers and payloads for readability. They are signed and transmitted as JWS Compact Serialization. Placeholder public-key coordinates, thumbprints, and signatures are illustrative rather than cryptographically complete.

## Authorization-Server-Issued Delegated Authorization Token

~~~json
{::include diagrams/as-issued-delegated-authorization-token.jsonc}
~~~

The root token's signature under an authorization server key establishes the root. The token permits two further levels and binds the root authorization to Client A's key.

## Client-Issued Delegated Authorization Token

~~~json
{::include diagrams/client-issued-delegated-authorization-token.jsonc}
~~~

The protected `jwk` is Client A's public key, and its thumbprint equals the root token's `cnf.jkt`. The payload binds the child token to Client B and narrows the scope, lifetime, and delegation depth.

## Multi-Level Delegated Authorization Token Chain

A second client-issued token can be appended:

~~~json
{::include diagrams/leaf-delegated-authorization-token.jsonc}
~~~

The protected `jwk` is Client B's public key and its thumbprint equals the preceding token's `cnf.jkt`. The new `cnf.jkt` identifies Client C's key. On the wire, the three compact JWTs are joined with `~` in root-to-leaf order.

# Negative Validation Examples

The following examples describe modifications to an otherwise valid chain and the required validation result. They are validation examples, not serialized cryptographic test vectors.

| Modification | Required result |
|---|---|
| A child `aud` contains an audience absent from the effective parent audience. | Reject the chain for audience expansion. |
| A token contains an empty `aud` array. | Reject the chain because an explicit `aud` claim must identify at least one audience. |
| A child `exp` is later than the effective parent expiration time. | Reject the chain for expiration-time expansion. |
| A child `nbf` is earlier than the effective parent not-before time. | Reject the chain for not-before-time expansion. |
| The chain contains more client-issued tokens than permitted by the root `max_delegation_depth`. | Reject the chain for excessive delegation depth. |
| The JWK thumbprint of a child's protected `jwk`, computed according to {{RFC7638}}, does not equal the preceding token's `cnf.jkt`. | Reject the chain for broken key continuity. |
| The DPoP proof key does not match the leaf token's `cnf.jkt`. | Reject the protected resource request. |
| Two tokens in the chain are reordered. | Reject the chain because its signatures, key continuity, or cumulative restrictions do not validate in that order. |
| The DPoP `ath` does not equal the hash of the exact received chain serialization. | Reject the protected resource request. |
| A protected header or payload contains a duplicate JSON member name. | Reject the chain during parsing. |
| A child contains `authorization_details` for which the verifier cannot establish semantic containment in the effective parent authorization details. | Reject the chain because containment cannot be established. |

# Comparison with Related OAuth Mechanisms

## OAuth Token Exchange

OAuth Token Exchange {{RFC8693}} lets a client request another token from an authorization server or security token service. That model provides centralized policy, issuance records, and potentially prompt revocation checks, but reveals each exchange to the issuing service and requires an online round trip.

This specification instead authorizes the client controlling the key bound to a token to create constrained child tokens locally. The approaches are complementary; a deployment can use token exchange when centralized issuance is required and local delegation for bounded offline workflows.

## DPoP-Bound Access Tokens

DPoP {{RFC9449}} binds an OAuth access token to a client key and binds each proof to an HTTP request. This specification reuses DPoP for authorization server key binding and protected resource access, but adds client-issued token chains and cumulative downscoping. It uses the `DA` authentication scheme and computes `ath` over the complete ordered chain.

## Rich Authorization Requests

Rich Authorization Requests {{RFC9396}} provide structured `authorization_details`. This specification carries those details through a delegation chain and requires every explicit child value to be no broader than the effective parent value. Each authorization detail type needs semantic containment rules before it can be safely delegated.
