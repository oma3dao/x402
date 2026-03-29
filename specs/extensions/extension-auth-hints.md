# Extension: `auth-hints`

## Summary

The `auth-hints` extension provides scheme-level authentication hints within x402 payment requirements. It enables clients to discover authentication requirements for specific payment schemes and complete registration and token acquisition *before* submitting a payment payload, avoiding an unnecessary round trip.

This extension addresses a specific gap: when a `402 Payment Required` response includes multiple payment schemes and only some of them require authentication, the client needs a way to know which schemes require auth and how to obtain credentials — before committing to a scheme.

This is a **Server ↔ Client** extension. The Facilitator is not involved in authentication.

---

## Interaction with `WWW-Authenticate`

On an HTTP transport, the `WWW-Authenticate` header (RFC 9110) is the standard mechanism for authentication challenges. x402 is fully compatible with this header. The following table describes how `WWW-Authenticate` and the `auth-hints` extension interact:

| Scenario               | `WWW-Authenticate` | `auth-hints` extension | Meaning                                                                                                     |
|------------------------|--------------------|------------------------|-------------------------------------------------------------------------------------------------------------|
| Resource requires auth | Present            | Absent                 | Auth is mandatory for all schemes. Standard HTTP flow.                                                      |
| Scheme requires auth   | Absent             | Present                | Auth is required only for specific schemes. Client uses extension hints.                                    |
| Both                   | Present            | Present                | Auth is mandatory for the resource AND the extension provides scheme-specific hints (e.g. different scopes).|
| Neither                | Absent             | Absent                 | No authentication required.                                                                                 |

---

## Authentication Without Hints

x402 is fully compatible with existing authentication mechanisms without this extension. Authentication and payment are parallel concerns- authentication identifies the client, payment authorizes the transfer of value.

When a resource server requires authentication, either for the resource itself or for a specific payment scheme, the client can discover this through standard HTTP challenge mechanisms:

1. The server returns a `402 Payment Required` response. If authentication is required for the resource, the server includes a `WWW-Authenticate` header on this response.
2. If the client selects a payment scheme that requires authentication and submits a `PaymentPayload` without credentials, the server responds with `401 Unauthorized` and a `WWW-Authenticate` header.
3. The client discovers the authorization server via `WWW-Authenticate` parameters or RFC 8414 (OAuth 2.0 Authorization Server Metadata).
4. The client registers via RFC 7591 (Dynamic Client Registration) if needed, obtains a token from the token endpoint, and retries with both authentication credentials and the payment payload.

On an HTTP transport, the retry includes both the `Authorization` header and the x402 `PAYMENT-SIGNATURE` header. For example, with DPoP:

```
GET /premium-data HTTP/1.1
Host: api.example.com
Authorization: DPoP <access-token>
DPoP: <proof-jwt>
PAYMENT-SIGNATURE: <base64-encoded-payment-payload>
```

Schema-based authorization without hints works but requires an extra round trip when the client doesn't know upfront that a scheme requires authentication. The `auth-hints` extension eliminates this by providing the authentication metadata in the `402` response.

On non-HTTP transports (MCP, A2A, etc.), the mechanism for discovering authentication requirements and presenting credentials is transport-specific. This specification defines behavior for HTTP. Other transports will define their own mechanisms in the future (see PaymentPayload below).

---

## Authentication With Hints

Some payment schemes require authentication even when the resource itself does not. For example, a `deferred` scheme uses off-chain vouchers against an escrow deposit, so the server needs to verify the client's identity to match vouchers to the correct escrow account and track accumulated value.  Without this extension, the client has no way to know which schemes require authentication until it tries and gets rejected. The `auth-hints` extension solves this by including authentication metadata in the `402` response, mapped to specific payment schemes.

### PaymentRequired

A resource server advertises scheme-level auth requirements by including the `auth-hints` extension in the `extensions` object of the `402 Payment Required` response.

The extension uses `schemeAuth` to map `accepts[]` entries to their authentication requirements.

```json
{
  "x402Version": 2,
  "error": "Payment required",
  "resource": {
    "url": "https://api.example.com/premium-data",
    "description": "Access to premium market data",
    "mimeType": "application/json"
  },
  "accepts": [
    {
      "scheme": "exact",
      "network": "eip155:8453",
      "amount": "10000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
      "maxTimeoutSeconds": 60
    },
    {
      "scheme": "deferred",
      "network": "eip155:8453",
      "amount": "10000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
      "maxTimeoutSeconds": 60
    }
  ],
  "extensions": {
    "auth-hints": {
      "info": {
        "schemeAuth": [
          {
            "acceptIndex": 1,
            "scheme": "deferred",
            "methods": [
              {
                "type": "oauth2",
                "tokenType": "DPoP",
                "authorizationServer": "https://as.example.com",
                "tokenEndpoint": "https://as.example.com/token",
                "registrationEndpoint": "https://as.example.com/register"
              }
            ]
          }
        ]
      },
      "schema": {
        "$schema": "https://json-schema.org/draft/2020-12/schema",
        "type": "object",
        "properties": {
          "schemeAuth": {
            "type": "array",
            "items": {
              "type": "object",
              "properties": {
                "acceptIndex": {
                  "type": "integer",
                  "description": "Index into the accepts[] array"
                },
                "scheme": {
                  "type": "string",
                  "description": "Payment scheme identifier for fallback matching"
                },
                "methods": {
                  "type": "array",
                  "items": {
                    "type": "object",
                    "properties": {
                      "type": {
                        "type": "string",
                        "description": "Authentication method type"
                      }
                    },
                    "required": ["type"]
                  }
                }
              },
              "required": ["methods"],
              "anyOf": [
                { "required": ["acceptIndex"] },
                { "required": ["scheme"] }
              ]
            }
          }
        },
        "required": ["schemeAuth"]
      }
    }
  }
}
```

In this example, `accepts[0]` (exact) requires no authentication. `accepts[1]` (deferred) requires OAuth 2.0 with DPoP.

#### `acceptIndex` and `scheme` Matching

Each `schemeAuth` entry identifies which `accepts[]` entries it applies to via `acceptIndex`, `scheme`, or both. At least one MUST be present.

When resolving which auth requirements apply to a chosen `accepts[]` entry, clients SHOULD:
1. If `acceptIndex` is present and in-range, use it (fast path)
2. If `acceptIndex` is absent or out of range, fall back to matching by `scheme` name against an `accepts[]` entry's `scheme` field

If both are present, `acceptIndex` takes precedence and `scheme` serves as the fallback. When both are present, the `scheme` value MUST match the `scheme` field of the referenced `accepts[acceptIndex]` entry. If they do not match, the client MUST treat the `schemeAuth` entry as invalid and ignore it.

#### Server-Declared PaymentRequired Fields

##### `schemeAuth[]`

| Field         | Type    | Required | Description                                                                                       |
|---------------|---------|----------|---------------------------------------------------------------------------------------------------|
| `acceptIndex` | integer | No*      | Index into the `accepts[]` array identifying which payment scheme requires auth                   |
| `scheme`      | string  | No*      | Payment scheme identifier (e.g., `"exact"`). Applies to all `accepts[]` entries with this scheme. |
| `methods`     | array   | Yes      | Supported authentication methods for this scheme. Client picks one.                               |

\* At least one of `acceptIndex` or `scheme` MUST be present.

If a `schemeAuth` entry applies to a selected `accepts[]` entry, authentication is REQUIRED for use of that payment scheme. The hints are not advisory- they indicate a mandatory authentication step.

##### Authentication Method Types

The `methods` array contains one or more authentication methods. Each method has a `type` field that determines the remaining fields. The following types are currently defined.

###### Type: `oauth2`

OAuth 2.0 authentication. The client obtains an access token from the authorization server and presents it in the request to the resource server. In HTTP, the token is sent in the `Authorization` header on the same request that carries the x402 payment.

| Field                 | Type   | Required | Description                                                                          |
|-----------------------|--------|----------|--------------------------------------------------------------------------------------|
| `type`                | string | Yes      | `"oauth2"`                                                                           |
| `tokenType`           | string | Yes      | Token type for presentation: `"Bearer"` or `"DPoP"`                                  |
| `authorizationServer` | string | Yes      | Base URL of the authorization server                                                 |
| `tokenEndpoint`       | string | Yes      | URL of the token endpoint for obtaining access tokens                                |
| `registrationEndpoint`| string | No       | URL of the DCR endpoint (RFC 7591). If present, dynamic client registration is available. If absent, the client must already have a `client_id`. |

The `tokenEndpoint` and `registrationEndpoint` serve different purposes. The registration endpoint (RFC 7591) is called once to register the client and obtain a `client_id`. The token endpoint is called for each session to obtain a time-limited access token using that `client_id`. Registration is optional and one-time; token acquisition is required and recurring.

###### Type: `sign-in-with-x`

Wallet-based authentication via the `sign-in-with-x` extension (CAIP-122). The client proves wallet ownership by signing a challenge.

| Field  | Type   | Required | Description         |
|--------|--------|----------|---------------------|
| `type` | string | Yes      | `"sign-in-with-x"`  |

When a client encounters this method type, it MUST look at the `sign-in-with-x` extension on the same `PaymentRequired` response for the full challenge parameters, supported chains, and schema. The auth-hint serves only as a pointer- all SIWX metadata is defined by the `sign-in-with-x` extension.

#### Client Behavior

When a client receives a `402` response containing the `auth-hints` extension:

1. The client evaluates all available payment schemes in `accepts[]`, including their authentication requirements from `schemeAuth`.
2. The client selects a payment scheme based on its capabilities, preferences, and the authentication burden of each option.
3. If the chosen scheme has a matching `schemeAuth` entry, the client completes the authentication flow before submitting the payment:
   - For `oauth2`: register if needed and obtain an access token
   - For `sign-in-with-x`: sign the SIWX challenge per the `sign-in-with-x` extension specification
4. The client submits the payment with both the authentication credentials and the x402 payment payload.

If no `schemeAuth` entry exists for the chosen scheme, no authentication is needed for that scheme.

### PaymentPayload

How the client presents authentication credentials alongside the payment payload depends on the transport.

#### HTTP

On HTTP transports, authentication credentials are sent as standard HTTP headers alongside the x402 `PAYMENT-SIGNATURE` header. The credentials do not travel inside the x402 `PaymentPayload`.

OAuth 2.0 — Bearer:

```
Authorization: Bearer <access-token>
PAYMENT-SIGNATURE: <base64-encoded-payment-payload>
```

OAuth 2.0 — DPoP:

```
Authorization: DPoP <access-token>
DPoP: <proof-jwt>
PAYMENT-SIGNATURE: <base64-encoded-payment-payload>
```

Sign-In With X:

```
SIGN-IN-WITH-X: <base64-encoded-siwx-proof>
PAYMENT-SIGNATURE: <base64-encoded-payment-payload>
```

#### Other Transports

Credential presentation for non-HTTP transports (MCP, A2A, etc.) is defined by the respective transport specifications. The `auth-hints` extension provides discovery metadata only; it does not define transport-specific presentation mechanisms.

### Server Verification

When the resource server receives a request with both authentication credentials and a payment payload, it validates both independently:

1. **Validate authentication:**
   - For `oauth2`: validate the access token (signature, audience, expiry, DPoP binding if applicable)
   - For `sign-in-with-x`: verify the SIWX proof per the `sign-in-with-x` extension specification (signature, nonce, domain, expiration)
2. **Validate payment:** forward the `PaymentPayload` to the facilitator (or verify locally) for payment verification and settlement
3. If both succeed, fulfill the request

The facilitator is not involved in authentication validation.

Authentication identity and payment identity are independent. The authenticated client MAY use one or more payer wallets, subject to server policy. This extension does not require the authentication identity to equal the payer address in the payment.

---

### Example Flow: OAuth 2.0 with DCR and DPoP

This example walks through the complete flow on an HTTP transport when a client encounters a `deferred` scheme that requires OAuth 2.0 authentication with DPoP.

#### Step 1 — Client Requests Resource

```
GET /premium-data HTTP/1.1
Host: api.example.com
```

#### Step 2 — Server Responds with 402

The server returns payment requirements. The `auth-hints` extension indicates that `accepts[1]` (deferred) requires OAuth 2.0 with DPoP.

```
HTTP/1.1 402 Payment Required
Content-Type: application/json

{
  "x402Version": 2,
  "resource": {
    "url": "https://api.example.com/premium-data",
    "description": "Access to premium market data",
    "mimeType": "application/json"
  },
  "accepts": [
    {
      "scheme": "exact",
      "network": "eip155:8453",
      "amount": "10000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
      "maxTimeoutSeconds": 60
    },
    {
      "scheme": "deferred",
      "network": "eip155:8453",
      "amount": "10000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
      "maxTimeoutSeconds": 60
    }
  ],
  "extensions": {
    "auth-hints": {
      "info": {
        "schemeAuth": [
          {
            "acceptIndex": 1,
            "scheme": "deferred",
            "methods": [
              {
                "type": "oauth2",
                "tokenType": "DPoP",
                "authorizationServer": "https://as.example.com",
                "tokenEndpoint": "https://as.example.com/token",
                "registrationEndpoint": "https://as.example.com/register"
              }
            ]
          }
        ]
      },
      "schema": { "..." : "..." }
    }
  }
}
```

#### Step 3 — Client Registers via DCR

The client chooses `accepts[1]` (deferred), sees the auth requirement, and registers with the authorization server via Dynamic Client Registration (RFC 7591). This is a one-time step — the client reuses the `client_id` for subsequent token requests. See RFC 7591 for the registration request and response format.

#### Step 4 — Client Obtains DPoP-Bound Access Token

Using the `client_id` from registration, the client requests an access token from the token endpoint (RFC 6749 §4.4). The client generates a DPoP key pair and includes a DPoP proof JWT per RFC 9449. See the respective RFCs for the token request and response format.

#### Step 5 — Client Submits Payment with Authentication

The client retries the original request with both the DPoP-bound access token and the x402 payment payload:

```
GET /premium-data HTTP/1.1
Host: api.example.com
Authorization: DPoP eyJ...
DPoP: <dpop-proof-jwt-for-resource>
PAYMENT-SIGNATURE: <base64-encoded-payment-payload>
```

#### Step 6 — Server Validates and Fulfills

The resource server validates the DPoP-bound access token, verifies the x402 payment (via facilitator or locally), and returns the requested resource.

---

## Security Considerations

- Use sender-constrained tokens (DPoP) when possible to prevent token theft and replay
- Validate token audience (`aud`) to ensure the token was issued for the correct resource server
- Validate token expiry to limit the window of use
- Protect token endpoints with TLS
- Authorization servers SHOULD implement replay protection for DPoP proofs (RFC 9449)
- For `sign-in-with-x`, servers MUST validate nonce uniqueness to prevent replay attacks

---

## References

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110) — `WWW-Authenticate` header semantics
- [RFC 8414: OAuth 2.0 Authorization Server Metadata](https://www.rfc-editor.org/rfc/rfc8414) — Authorization server discovery
- [RFC 7591: OAuth 2.0 Dynamic Client Registration](https://www.rfc-editor.org/rfc/rfc7591) — DCR protocol
- [RFC 9449: OAuth 2.0 Demonstrating Proof of Possession (DPoP)](https://www.rfc-editor.org/rfc/rfc9449) — Sender-constrained tokens
- [CAIP-122: Sign-In With X](https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-122.md) — Wallet-based authentication
- [Extension: `sign-in-with-x`](sign-in-with-x.md) — x402 SIWX extension specification
- [Core x402 Specification](../x402-specification-v2.md)
