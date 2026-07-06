---

title: 'OAuth 2.0 Delegated Authorization'
abbrev: 'Delegated-Auth'
docname: 'draft-li-oauth-delegated-authorization-latest'

stand_alone: true
wg: oauth
ipr: trust200902
submissiontype: IETF
cat: info

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
  RFC2119:
  RFC6749:
  RFC6750:
  RFC7515:
  RFC7516:
  RFC7517:
  RFC7519:
  RFC7636:
  RFC7662:
  RFC8174:
  RFC8259:
  RFC8392:
  RFC8414:
  RFC8615:
  RFC8693:
  RFC8705:
  RFC9110:
  RFC9396:
  RFC9449:
  RFC9700:
  RFC9728:
  I-D.lombardo-oauth-step-up-authz-challenge-proto:

informative:
  RFC8792:

--- abstract

Delegated authorization enables a client to delegate a subset of its granted privileges to a subordinate access token (also known as a delegated access token). This mechanism allows the client to securely delegate authorization to another application while maintaining fine-grained control over delegated permissions.

--- middle

# Introduction

OAuth 2.0 {{RFC6749}} provides a framework for authorizing third-party applications to access protected resources on behalf of a resource owner. However, in existing implementations, access tokens issued to clients often contain excessive permissions that exceed actual requirements, creating security vulnerabilities and potential data exposure risks.

This specification extends OAuth 2.0 with a delegated authorization framework that enables clients to create subordinate access tokens with restricted permissions. This approach addresses the problem of over-privileged access tokens by implementing a token-chain architecture that decouples the initial authorization from the final resource access, allowing delegated tokens to be created and used flexibly.

# Requirements Language

{::boilerplate bcp14-tagged}

# Terminology

This specification uses the following terms defined in OAuth 2.0 {{RFC6749}}: authorization server, client, resource server, and resource owner. A resource server can also act as a client when it accesses one or more downstream resource servers. This document refers to such an entity as an intermediary resource server: it is downstream from the client that invokes it, and acts as a client when making requests to resource servers that are farther downstream in the call chain. Intermediary resource servers can be chained.

The following additional terms are used throughout this document:

**Intermediary Resource Server**:
: A resource server that receives a request from a client and, to serve that request, acts as an OAuth client when accessing one or more downstream resource servers. It is downstream from its caller and upstream from the resource servers it invokes. A downstream resource server can itself be an intermediary resource server when it invokes another resource server farther downstream in the call chain.

**Target Resource Server**:
: The resource server that receives a delegated access token from an upstream intermediary resource server for a particular downstream request and hosts the protected resource requested in that hop. This role is relative to a particular request: a target resource server for one hop can also act as an intermediary resource server for a subsequent downstream hop. It is downstream from the intermediary resource servers that precede it in that call chain.

**Delegation Token**:
: A token issued by the authorization server for the client that enables the client to create delegated access tokens.

**Delegated Access Token**:
: A token created by the client using the delegation token, with permissions being a subset of the delegation token's privileges and a more limited lifespan.

**Delegation Key**:
: A cryptographic key bound to the delegation token, used by the client to sign delegated access tokens. The delegation key is presented in the token request as the `delegation_key` parameter.

# Overview

The delegated authorization framework introduces a hierarchical token structure where a client can obtain a delegation token from an authorization server and use it to issue subordinate access tokens with reduced permissions. This enables fine-grained access control while maintaining the security properties of the original authorization grant.

~~~aasvg
{::include diagrams/delegated-authorization-architecture.ascii-art}
~~~
{: #fig-architecture title="Delegated Authorization Framework Architecture" align="center"}

1. The client requests authorization from the resource owner. The client indicates in the authorization request that the requested authorization grant is for delegated authorization.

2. The client receives an authorization grant.

3. The client requests a delegation token by authenticating with the authorization server and presenting the authorization grant and its delegation key as defined in Section 3.

4. The authorization server authenticates the client and validates the authorization grant, and if valid, issues a delegation token.

5. The client calls an intermediary resource server's API, presenting the delegated access token generated from the delegation token. The delegated access token is issued by the client using the delegation key. The intermediary resource server can request protected resources from one or more downstream resource servers by presenting the delegated access token. A downstream resource server can itself be an intermediary resource server for a subsequent downstream request.

6. A target resource server for a downstream request validates the delegated access token, and if valid, serves the resource. Responses propagate back through the intermediary resource servers, each of which can optionally transform a downstream response into a service-specific response, and ultimately return a response to the client.

Both delegation token and delegated access token can be JSON Web Tokens (JWTs) {{RFC7519}} or CBOR Web Tokens (CWTs) {{RFC8392}}.

# Protected Resource Metadata Discovery

Before the client retrieves a delegation token and generates a delegated access token for an intermediary resource server, the client needs to determine the permissions required by that resource server for its own protected resources, the permissions required when accessing downstream resource servers, and the identity of those downstream resource servers. Such information can be manually configured into the client, or it can be dynamically discovered using OAuth 2.0 Protected Resource Metadata {{RFC9728}}.

Each resource server publishes its metadata at the standard well-known URI as defined in {{RFC9728}}. The metadata describes the resource server as a protected resource. A client can determine that a resource server is an intermediary resource server when the metadata contains the `delegated_resources` key. The `delegated_resources` metadata field identifies any downstream resource servers the publishing resource server accesses and the authorization capabilities for each. If a downstream resource server is itself an intermediary resource server, the client can also retrieve that resource server's metadata to discover subsequent downstream requirements.

The `authorization_requirements` metadata field identifies the authorization requirements for specific endpoints or operations. A client can use this field to request a delegation token and create delegated access tokens with the minimum permissions needed for the intended request. Discovery of protected resource metadata, including use of the `WWW-Authenticate` `resource_metadata` parameter, follows {{RFC9728}}.

## The `delegated_resources` Metadata Field

This specification defines a new protected resource metadata field, `delegated_resources`, that indicates the downstream resource servers and the authorization capabilities the publishing resource server needs for each.

**delegated_resources**:
: JSON object mapping downstream protected resource identifiers (as defined in {{RFC9728}}) to the authorization capabilities for each. Each downstream resource value is a JSON object containing the following members:

  **authorization_target**:
  : OPTIONAL. String identifying the protected resource to which the listed `scopes_supported` or `authorization_details_types_supported` apply. The value `self` indicates the downstream protected resource identified by the corresponding `delegated_resources` key. Any other value is a protected resource identifier as defined in {{RFC9728}}. If omitted, the default value is `self`.

  **scopes_supported**:
  : OPTIONAL. JSON array of scope values {{RFC6749}} that the publishing resource server may need at the `authorization_target`.

  **authorization_details_types_supported**:
  : OPTIONAL. JSON array of authorization details types {{RFC9396}} that the publishing resource server may need at the `authorization_target`.

A downstream resource value MUST contain at least one of `scopes_supported` or `authorization_details_types_supported`.

## The `authorization_requirements` Metadata Field

This specification defines a new protected resource metadata field, `authorization_requirements`, that indicates the authorization requirements for specific endpoints or operations of the publishing resource server. The requirements can describe permissions needed at the publishing resource server itself, permissions delegated to downstream resource servers, or both.

**authorization_requirements**:
: JSON array of authorization requirement objects. Each object describes one endpoint or operation and the permissions required to access it. The following members are defined:

  **methods**:
  : OPTIONAL. JSON array of HTTP methods to which this requirement applies. If omitted, the requirement applies independent of HTTP method.

  **path**:
  : OPTIONAL. Absolute path or path template identifying the endpoint to which this requirement applies. If omitted, the requirement applies to the operation as otherwise identified by the object.

  **operation**:
  : OPTIONAL. String identifying an application-level operation to which this requirement applies. This member is intended for APIs where the authorization requirement is associated with a named action rather than only with an HTTP path. The syntax and semantics of the operation identifier are deployment specific. For HTTP APIs where the method and path uniquely identify the protected action, `methods` and `path` are preferred.

  **scopes**:
  : OPTIONAL. JSON array of scope values {{RFC6749}} required for the identified endpoint or operation at the publishing resource server.

  **authorization_details_types**:
  : OPTIONAL. JSON array of authorization details types {{RFC9396}} required or accepted for the identified endpoint or operation at the publishing resource server.

  **authorization_details**:
  : OPTIONAL. JSON array of authorization detail objects {{RFC9396}} that describe the required permissions for the identified endpoint or operation at the publishing resource server. This member can be used when the authorization details type alone is not sufficient to describe the minimum required permission.

  **delegated_resources**:
  : OPTIONAL. JSON object mapping downstream protected resource identifiers to delegated authorization requirement objects. This member describes permissions that the publishing resource server needs to receive in a delegated access token in order to call the specified downstream resource servers while serving the identified local endpoint or operation. Each delegated authorization requirement object MAY contain `authorization_target`, with the same semantics as in the `delegated_resources` metadata field, and identifies required permissions using `scopes`, `authorization_details_types`, or `authorization_details`.

A top-level authorization requirement object MUST identify an endpoint or operation using `path`, `operation`, or both. Each top-level authorization requirement object MUST contain at least one of `scopes`, `authorization_details_types`, `authorization_details`, or `delegated_resources`. Each delegated authorization requirement object MUST contain at least one of `scopes`, `authorization_details_types`, or `authorization_details`. The `authorization_requirements` field is optional. Resource servers MAY omit it when authorization requirements are dynamic, confidential, or better conveyed through an authorization challenge.

When both `path` and `operation` are present, `path` identifies the endpoint and `operation` further identifies the action within that endpoint.

The `authorization_requirements` metadata is self-declared discovery information. It does not replace access control enforcement by resource servers, and clients MUST NOT assume that possession of the indicated permissions guarantees access to the identified endpoint or operation.

## Metadata Example

The following is a non-normative example of a resource server publishing protected resource metadata that describes both its downstream resource server capabilities and endpoint-specific authorization requirements:

~~~sourcecode
{
  "resource": "https://analytics.example.com",
  "scopes_supported": ["openid", "analytics:summary"],
  "authorization_servers": ["https://idp.example.com"],

  "delegated_resources": {
    "https://crm.example.com": {
      "authorization_target": "self",
      "scopes_supported": ["read:contacts", "read:reports"],
      "authorization_details_types_supported": ["data"]
    }
  },

  "authorization_requirements": [
    {
      "methods": ["GET"],
      "path": "/summary",
      "scopes": ["analytics:summary"]
    },
    {
      "methods": ["GET"],
      "path": "/reports/contacts",
      "delegated_resources": {
        "https://crm.example.com": {
          "scopes": ["read:contacts"],
          "authorization_details_types": ["data"]
        }
      }
    },
    {
      "methods": ["GET"],
      "path": "/reports/monthly",
      "delegated_resources": {
        "https://crm.example.com": {
          "authorization_target": "https://reports.example.com",
          "scopes": ["read:reports"]
        }
      }
    }
  ]
}
~~~

In this example the resource server (`https://analytics.example.com`) indicates that it supports its own `analytics:summary` scope and that it may need access to the CRM resource server (`https://crm.example.com`) with the `read:contacts` and `read:reports` scopes. The `authorization_requirements` field further indicates that `GET /summary` requires the local `analytics:summary` scope, `GET /reports/contacts` requires a delegated access token that covers the CRM `read:contacts` scope and `data`-type authorization details, and `GET /reports/monthly` requires a delegated access token presented to CRM that covers the `read:reports` scope at `https://reports.example.com`.

The client uses this information, and optionally the metadata of downstream intermediary resource servers, to:

1. Request a delegation token from the authorization server with appropriate permissions (see Section 7).
2. Create delegated access tokens with the correct permissions, audience, and downstream resource identifiers (see Section 8).

# Delegation Tokens and Delegated Access Tokens

This section defines the properties of delegation tokens and delegated access tokens. The delegated authorization framework uses a hierarchical token structure where tokens form a chain: a delegation token can be used to create subordinate tokens, which can either be another delegation token (to continue the chain) or a delegated access token (to access protected resources). The top-level token in the chain is issued by the authorization server, while subordinate tokens are created by clients.

## Delegation Tokens

A delegation token is a token that can be used to create subordinate tokens. It contains a `max_delegation_depth` claim that limits the maximum length of the delegation chain.

**delegation_key**:
: The cryptographic key bound to the delegation token, used by the client to sign subordinate tokens (either another delegation token or a delegated access token). The value is a JSON object representing the public key, using key format as defined in {{RFC7517}}. This claim MUST be present in delegation tokens. The corresponding private key is held by the client and used to create subordinate tokens. Subordinate tokens MUST include the parent delegation token in the `delegation_token` claim, creating a verifiable chain from the top-level token to the subordinate token.

**max_delegation_depth**:
: OPTIONAL. Integer value indicating the maximum depth of the delegation chain. If not present, the delegation chain is unrestricted. When a client creates a subordinate token from a delegation token:

- If the parent delegation token has a `max_delegation_depth` value, the subordinate delegation token MUST have a `max_delegation_depth` value smaller than the parent's value.
- If the parent delegation token does not have a `max_delegation_depth` value, the subordinate delegation token MAY set any `max_delegation_depth` value.

When `max_delegation_depth` reaches 1, the subordinate token MUST be a delegated access token (no further delegation is allowed).

The `scope` / `authorization_details`, `aud`, `exp`, and `nbf` claims MAY be present in delegation tokens and can be reduced in subordinate tokens.

**scope** / **authorization_details**:
: The scope (as defined in {{RFC8693}}) or authorization details (as defined in {{RFC9396}}) of the delegation token. When creating a subordinate token, the client MAY reduce the scope or authorization_details to a subset of the parent token's permissions.

**aud**:
: The audience of the delegation token. When creating a subordinate token, the client MAY narrow the audience to a subset of the parent token's audience.

**exp**:
: The expiration time of the delegation token. When creating a subordinate token, the client MUST set an expiration time that is no later than the parent token's expiration time.

**nbf**:
: The not-before time of the delegation token. When creating a subordinate token, the client MAY set a not-before time that is later than the parent token's not-before time.

### Top Level Delegation Token

The top-level delegation token is issued by the authorization server and MAY contain the `iss`, `sub`, and `jti` claims. If present, these claims apply to the entire token chain. These claims MUST NOT be present in subordinate delegation tokens or delegated access tokens.

**iss**:
: The issuer of the delegation token. This claim applies to all subordinate tokens in the chain.

**sub**:
: The subject of the delegation token, typically representing the resource owner or the client. This claim applies to all subordinate tokens in the chain.

**jti**:
: Unique identifier for the delegation token. This claim applies to all subordinate tokens in the chain.

Subordinate delegation tokens and delegated access tokens MUST NOT include the `iss`, `sub`, or `jti` claims. Verifiers (resource servers) SHALL consider these claims from the top-level delegation token when validating the entire token chain.

## Delegated Access Tokens

A delegated access token is a token used to access protected resources. It cannot be used to create subordinate tokens.

The `scope`, `aud`, `exp`, and `nbf` claims in delegated access tokens follow the same rules as in delegation tokens, as described in Section 6.1.

**max_delegation_depth**:
: Delegated access tokens do not need to include a `max_delegation_depth` claim, even if the parent delegation token has one. If a delegated access token includes `max_delegation_depth`, its value MUST be 0.

# Acquiring Delegation Tokens

The client requests a delegation token using standard OAuth 2.0 grant types with additional parameters to distinguish delegation requests from standard token requests.

## Authorization Code Grant

For authorization code grant type, the client MUST include a `delegation=true` parameter in the authorization request to indicate that the client is requesting a delegation token instead of an OAuth 2.0 access token.

Additionally, the authorization request MUST include either a `scope` parameter (as defined in {{RFC6749}} Section 3.3), an `authorization_details` parameter (as defined in "Rich Authorization Requests" {{RFC9396}}), or both, that define the permissions granted to the requested delegation token.

In the token request, the client MUST include a `delegation_key` parameter with the value of the delegation key. The delegation key is a public key for digital signature.

Additionally, the client MAY include in the token request either a `scope` parameter, an `authorization_details` parameter, or both. The client MAY also include a `delegation=true` parameter in the token request.

In the token response, the authorization server MUST include an `access_token` attribute whose value is the delegation token, and MUST include a `token_type` attribute valued `"Delegation"`, and MAY include a `refresh_token` attribute which is the refresh token for obtaining a new delegation token via the refresh token grant.

Other procedures of the authorization code grant are as described in {{RFC6749}}. Use of Proof Key for Code Exchange (PKCE) {{RFC7636}} is RECOMMENDED.

## Other Grant Types

Other OAuth 2.0 grant types, such as the refresh token grant or client credentials grant, MAY support delegated authorization by including the `delegation` and `delegation_key` parameters when applicable. The authorization server MUST validate that the client is authorized to request delegation tokens using the given grant type.

# Creating Delegated Access Tokens

The client creates delegated access tokens by:

1. Validating the delegation token's validity and permissions.
2. Generating a subordinate access token with (optionally) reduced privileges.
3. Applying cryptographic protection using the delegation key (digital signature).

The client MUST include the delegation token in the `delegation_token` attribute of the delegated access token.

The client MUST ensure that the delegated access token's scope, lifetime, audience, and other claims do not exceed those of the delegation token. The client MAY generate single-use delegated access tokens that the resource server or authorization server only consider valid when validating it for the first time.

The client is RECOMMENDED to "sender-constrain" the delegated access tokens by binding the delegated access tokens with public keys or certificates where the corresponding private keys are held by the resource servers that will present those tokens, via techniques similar to OAuth 2.0 mTLS {{RFC8705}} or OAuth 2.0 DPoP {{RFC9449}}.

# Using Delegated Access Tokens

When the client accesses an intermediary resource server endpoint that requires downstream access, the client MUST include the delegated access token as a bearer token {{RFC6750}} in the `Delegated-Authorization` header. The delegated access token is used by downstream resource servers to verify requests from upstream intermediary resource servers. The `Delegated-Authorization` header MAY be used in combination with an `Authorization` header used by the intermediary resource server to verify the request from the client.

For example:

~~~
GET /dp-resource HTTP/1.1
Host: analytics.example.com
Authorization: Bearer mF_9.B5g1234
Delegated-Authorization: Bearer mF_9.B5f-4.1JqM
~~~

Upon receiving a request with a `Delegated-Authorization` header, the intermediary resource server can send downstream requests to one or more resource servers for the respective protected resources. For each such downstream request, the invoked resource server is the target resource server for that hop and can itself act as an intermediary resource server for subsequent downstream requests. The intermediary resource server MUST include the received delegated access token as a bearer token in the `Authorization` header.

For example:

~~~
GET /target-resource HTTP/1.1
Host: resource.example.com
Authorization: Bearer mF_9.B5f-4.1JqM
~~~

# Verification of Delegated Access Tokens

Resource servers verify delegated access tokens through either local validation using pre-configured public keys or remote validation via token introspection {{RFC7662}} at the authorization server.

## Local Verification

The resource server verifies delegated access tokens by:

1. The resource server is pre-configured with the authorization server's public key, or it fetches the public key via the authorization server's JWKS endpoint {{RFC7517}}.
2. Checking the digital signature of the delegation token (part of the delegated access token) against the authorization server's public key.
3. Checking the digital signature of the delegated access token against the delegation key bound to the delegation token.
4. Verifying the delegated access token's permissions and validity are within the scope of the delegation token.
5. Verifying the delegated access token is within validity period, and the delegated access token's permissions cover the resource request.

## Token Introspection

The resource server sends the delegated access token to the authorization server via the token introspection endpoint {{RFC7662}}. The authorization server verifies the delegated access token against its keys.

# Privacy Considerations

This section describes the privacy properties of the local delegation approach defined in this specification.

## Privacy Benefits

When a client creates delegated access tokens locally using a delegation token, the authorization server only observes the initial delegation token request. After that, the client can independently issue subordinate tokens without further authorization server interaction. This provides the following privacy benefits:

- **Minimized Authorization Server Visibility**: The authorization server does not learn which intermediary resource servers are authorized to access downstream resource servers, what resources they access, or when they access them.
- **No Access Pattern Correlation**: The authorization server cannot track usage patterns or correlate activities across different resource servers.
- **Reduced Data Collection**: The authorization server stores only metadata about the initial delegation token, not about each delegated access token.
- **Network Traffic Reduction**: Fewer network round-trips to the authorization server reduce exposure to network surveillance.

## Relationship to Token Exchange

Token Exchange {{RFC8693}} provides a mechanism for delegating access by having the client exchange one token for another at the authorization server. That approach requires authorization server involvement for each delegation event, which necessarily exposes delegation metadata to the authorization server. The local delegation approach in this specification performs delegation entirely on the client side after the initial token issuance, providing an alternative with different privacy characteristics.

## Trade-offs

While local delegation provides significant privacy benefits, implementations should consider:

- **Client Responsibility**: The client must securely protect the delegation key and properly enforce delegation rules.
- **Auditability**: Deployments requiring delegation event audits can implement client-side logging rather than relying on authorization server monitoring.
- **Revocation**: If a delegation key is compromised, the authorization server may need to revoke the entire delegation token.

# Security Considerations

This specification extends OAuth 2.0 to support delegated authorization through hierarchical token issuance. While this enables fine-grained privilege delegation, it also introduces new trust and security considerations.

- Delegation tokens MUST NOT be sent to resource servers without subordinate delegated access tokens. Resource servers and authorization servers MUST NOT treat delegation tokens as regular OAuth access tokens.
- Clients MUST protect the delegation key, as compromise allows an attacker to mint valid delegated access tokens within the scope of the delegation token.
- Delegated access tokens SHOULD have short lifetimes and be bound to specific audiences, methods, and sender keys (e.g., via DPoP or mTLS) to mitigate replay and token leakage risks. Resource servers MUST validate both the delegation token and the delegated access token, ensuring the latter does not exceed the former’s permissions.
- Token introspection CAN be used in scenarios where the tokens are kept opaque from the resource servers. If employed, token introspection responses MUST NOT reveal sensitive internal information. Authorization servers SHOULD enforce rate limiting and audit token issuance and validation activities.
- Protected resource metadata published by resource servers is self-declared. Clients and authorization servers SHOULD verify that delegated access tokens remain within the intended scope of delegation. Clients SHOULD not trust `delegated_resources` or `authorization_requirements` metadata from untrusted resource servers without independent verification of the resource server's identity and authorization requirements.

# Operational Considerations

Deployments of this specification should consider the following operational aspects:

- **Key Management**: Clients MUST securely store and rotate delegation keys. Authorization servers SHOULD support key rotation for delegation tokens and provide mechanisms to revoke compromised keys.
- **Token Lifetimes**: Delegation tokens SHOULD have longer lifetimes than delegated access tokens to reduce authorization server load, and SHOULD be refreshable using refresh tokens.
- **Metadata Caching**: Protected resource metadata published by resource servers at `/.well-known/oauth-protected-resource` can be cached by clients; a reasonable default TTL (e.g., 24 hours) is RECOMMENDED.
- **Error Handling**: Resource servers SHOULD provide clear error responses (e.g., invalid token, insufficient scope) without exposing implementation details.
- **Interoperability**: Implementers SHOULD ensure compatibility with existing OAuth 2.0 features such as PKCE, Rich Authorization Requests, and sender-constrained tokens.

# IANA Considerations

## OAuth Parameters Registry

This specification registers the following parameters in the "OAuth Parameters" registry:

**Delegation:**

- **Name**: delegation
- **Parameter Usage Location**: authorization request, token request
- **Change Controller**: IETF
- **Reference**: [this document]

**Delegation Key:**

- **Name**: delegation_key
- **Parameter Usage Location**: token request
- **Change Controller**: IETF
- **Reference**: [this document]

## OAuth Access Token Types Registry

This specification registers the following parameters in the "OAuth Access Token Types" registry:

- **Name**: Delegation
- **Additional Token Endpoint Response Parameters**: (none)
- **HTTP Authentication Scheme(s)**: Bearer
- **Change Controller**: IETF
- **Reference**: [this document]

## HTTP Field Name Registry

This specification registers the following parameters in the "Hypertext Transfer Protocol (HTTP) Field Name" registry:

- **Field Name**: Delegated-Authorization
- **Status**: permanent
- **Structured Type**: (none)
- **Reference**: [this document]

## OAuth Protected Resource Metadata Registry

This specification registers the following entry in the "OAuth Protected Resource Metadata" registry defined in {{RFC9728}}:

- **Metadata Name**: delegated_resources
- **Metadata Description**: JSON object mapping downstream protected resource identifiers to the authorization capabilities for each. Each downstream resource value contains `authorization_target` and either `scopes_supported` or `authorization_details_types_supported`, indicating permissions that may be needed at that authorization target.
- **Change Controller**: IETF
- **Specification Document(s)**: [this document]

- **Metadata Name**: authorization_requirements
- **Metadata Description**: JSON array describing authorization requirements for specific endpoints or operations of the protected resource. Each entry can identify local requirements using `scopes`, `authorization_details_types`, or `authorization_details`, and can identify downstream delegated requirements using `delegated_resources`. Within `delegated_resources`, `authorization_target` distinguishes the protected resource to which the listed permissions apply.
- **Change Controller**: IETF
- **Specification Document(s)**: [this document]

--- back

# Token Format

The example tokens in this section are shown in Flattened JSON Serialization {{RFC7515}} {{RFC7516}}, un-base64url-encoded and with comments for ease of reading. When used as JWTs {{RFC7519}}, they should be represented in Compact Serialization {{RFC7515}} {{RFC7516}}. Similarly, they can be represented as CWTs {{RFC8392}}.

## Example 1

In this example, the delegation token is a JWS token signed with HS256, and the delegated access token is a JWS token signed with RS256.

**Delegation Token**:

~~~sourcecode
{::include diagrams/example1-delegation-token.jsonc}
~~~

**Delegated Access Token**:

~~~sourcecode
{::include diagrams/example1-delegated-access-token.jsonc}
~~~

## Example 2

In this example, the delegation token chain consists of three tokens: a top-level delegation token issued by the authorization server, a subordinate delegation token created by the client, and a delegated access token created from the subordinate token. The top-level delegation token has `max_delegation_depth` of 3, which allows for two levels of delegation.

**Top-Level Delegation Token**:

~~~sourcecode
{::include diagrams/example2-top-level-delegation-token.jsonc}
~~~

**Subordinate Delegation Token**:

~~~sourcecode
{::include diagrams/example2-subordinate-delegation-token.jsonc}
~~~

**Delegated Access Token**:

~~~sourcecode
{::include diagrams/example2-delegated-access-token.jsonc}
~~~

# Integration with Step-Up Authorization

This specification can be used together with step-up authorization {{I-D.lombardo-oauth-step-up-authz-challenge-proto}}. When a resource server returns an `insufficient_authorization` error, the client can create a new delegated access token with the required permissions from its existing delegation token.

This is particularly useful for AI agents that hold a delegation token. Upon receiving a step-up authorization challenge, the agent can automatically create a new delegated access token with the additional authorization_details required by the resource server, without requiring the user to re-authenticate.

Resource servers that support both delegated authorization and step-up challenges SHOULD use the `body_instructions` parameter to specify the exact authorization_details or scope required.

# Use Cases

## Delegating Subset of Access Rights to Specialized AI Agents

Enterprise Identity and Access Management systems often employ Role Based Access Control (RBAC) or Attribute Based Access Control (ABAC), assigning a set of minimal permissions to the employee based on their role, department, or other attributes. AI Agent can be an employee's personal assistant, or a virtual employee of a certain department in general. The permissions delegated to an AI agent CAN be long-term, but an AI agent MUST NOT directly inherit all its owner's access rights. Rather, they SHOULD be a subset of its owner, bound to specific service/API/database/codebase according to its specialty and dedicated workflow.

| Role | Service / Component |
| --- | --- |
| Resource Owner | an enterprise, individual or a department |
| Client | agent's client application |
| Intermediary Resource Server | CI-CD agent, test agent, DEV agent, research agent |
| Authorization Server | enterprise IAM system |
| Downstream / Target Resource Server(s) | enterprise IT systems |
| Downstream / Target Protected Resource(s) | DEV/STAGE/PROD environments, internal knowledge database |
{: title="AI Agents" align="center"}

## Third‑Party Analytics Platform Integrated in an Enterprise SaaS

In this scenario, a corporate customer uses a Software-as-a-Service (SaaS) Customer Relationship Management (CRM) application. The customer wishes to gain business insights by granting a specialized third-party analytics platform limited access to its CRM data.

The CRM application obtains a delegation token from the enterprise's identity provider. It then creates a narrowly scoped delegated access token for the analytics service. This token only permits read access to a predefined, non-sensitive subset of customer data (e.g., names and identifiers, but not personal email addresses). The analytics platform uses this token to pull data, generates an aggregated business intelligence report, and delivers it back to the CRM application for the corporate customer to view.

| Role | Service / Component |
| --- | --- |
| Resource Owner | company A (the tenant) |
| Client | SaaS CRM application |
| Intermediary Resource Server | analytics service |
| Authorization Server | enterprise IdP |
| Downstream / Target Resource Server(s) | CRM application server |
| Downstream / Target Protected Resource(s) | CRM application's data retrieval API |
{:  title="Enterprise-SaaS" align="center"}
