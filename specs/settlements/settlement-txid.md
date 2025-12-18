# TXID (Transaction-Reference) Settlement Specification

## Overview

TXID (Transaction-Reference) Settlement is a settlement mechanism designed to enable x402 payments using **any transferable asset on any blockchain**, without requiring the asset to implement a specialized "transfer-with-authorization" or permit interface.

In TXID settlement, the payer executes an on-chain transfer directly using the native transaction mechanics of the underlying network. The client then proves payment to the resource server by presenting:

- a **transaction reference** (`txRef`) identifying the on-chain transfer, and
- a **payer signature** demonstrating that the entity requesting the resource controls the account that funded the transaction.

This settlement model allows x402 to support:

- tokens and native assets that do not implement standardized authorization-based transfer interfaces,
- non-EVM chains and heterogeneous execution environments,
- payment flows where the payer submits the transaction and pays network fees directly, including with the native token of the chain, although existing gas subsidy tools can still be used.

TXID settlement deliberately shifts responsibility from protocol-enforced guarantees to facilitator-side correctness. In exchange for supporting payments in any transferable asset on any chain, TXID requires facilitators to correctly implement additional verification and state-management logic. Unlike facilitated signed-transfer settlements—where replay protection for fund movement is enforced by on-chain mechanisms, TXID requires facilitators to enforce replay protection for **service delivery**. This includes:

- maintaining persistent state to de-duplicate accepted transaction references,
- aligning transaction acceptance rules with transaction-reference retention policies.

In return, TXID enables resource servers to accept payments for assets that do not support standardized authorization-based transfer interfaces, including:

- native assets of a chain (e.g. ETH),
- tokens without permit or transfer-with-authorization functionality.

## Status, Evolution, and Forward Compatibility

TXID is specified here as a settlement mechanism for x402. The x402 ecosystem is expected to introduce additional settlement mechanisms over time (e.g., permit-style and other authorization-based approaches). Supporting these safely may require changes to the protocol's settlement layering model (for example, moving settlement-specific parameters into a dedicated container, scheme-specific payload structures, or a standardized extensions namespace).

Accordingly:

- **Wire shape and field placement are not considered stable** and may change to match the x402 canonical settlement architecture once standardized.
- **Behavioral requirements are stable**: the verification rules, payer-binding requirements, replay-prevention requirements, and confirmation/finality policy requirements in this document are normative and MUST be implemented as written, independent of serialization details.
- Implementers SHOULD design with forward compatibility in mind and SHOULD treat unknown settlement-specific fields as unsupported rather than attempting best-effort interpretation.

## Protocol Integration

This section describes how TXID settlement is represented within x402 message structures. TXID settlement is compatible with both x402 v1 and v2 protocols.

### Settlement Identifier

TXID settlement is identified by the `settlement` field in both payment requirements and payment payloads:

```
settlement: "txid"
```

The `scheme` field (e.g., `"exact"`) identifies the payment method family, while `settlement` specifies the exact settlement mechanism.

---

### Payment Requirements (Server → Client)

When a resource server requires payment via TXID settlement, it responds with payment requirements containing TXID-specific fields.

#### x402 v1 Payment Requirements Example

```json
{
  "x402Version": 1,
  "error": "Payment required",
  "accepts": [
    {
      "scheme": "exact",
      "settlement": "txid",
      "network": "eip155:1",
      "maxAmountRequired": "10000",
      "asset": "native",
      "payTo": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
      "resource": "https://api.example.com/premium-data",
      "description": "Access to premium market data",
      "mimeType": "application/json",
      "outputSchema": null,
      "maxTimeoutSeconds": 60
    }
  ]
}
```

#### x402 v2 Payment Requirements Example

In v2, the resource information is separated into a top-level `resource` object, network identifiers use CAIP-2 format, and the amount field is renamed:

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
      "settlement": "txid",
      "network": "eip155:1",
      "amount": "10000",
      "asset": "native",
      "payTo": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
      "maxTimeoutSeconds": 60
    }
  ],
  "extensions": {}
}
```

#### TXID Payment Requirements Fields (v1)

| Field Name          | Type     | Required | Description                                                              |
| ------------------- | -------- | -------- | ------------------------------------------------------------------------ |
| `scheme`            | `string` | Required | Payment scheme identifier (e.g., "exact")                                |
| `settlement`        | `string` | Required | Settlement type identifier ("txid" for TXID settlement)                  |
| `network`           | `string` | Required | Blockchain network identifier (e.g., "base-sepolia", "eip155:1")         |
| `maxAmountRequired` | `string` | Required | Required payment amount in atomic token units (e.g.- Wei or Lamport)     |
| `asset`             | `string` | Required | Token contract address, or "native" for the native token of "network"    |
| `payTo`             | `string` | Required | Recipient wallet address for the payment                                 |
| `resource`          | `string` | Required | URL of the protected resource                                            |
| `description`       | `string` | Required | Human-readable description of the resource                               |
| `mimeType`          | `string` | Optional | MIME type of the expected response                                       |
| `outputSchema`      | `object` | Optional | JSON schema describing the response format                               |
| `maxTimeoutSeconds` | `number` | Optional | Maximum time allowed for payment completion (payment offer expiration)   |
| `extra`             | `object` | Optional | Scheme-specific additional information                                   |

#### TXID Payment Requirements Fields (v2 Differences)

v2 uses the same fields as v1 with the following differences:

| Field Name          | Change                                                                                      |
| ------------------- | ------------------------------------------------------------------------------------------- |
| `network`           | Uses CAIP-2 format (e.g., "eip155:84532") instead of human-readable identifiers             |
| `amount`            | Replaces `maxAmountRequired`                                                                |
| `resource`          | Moved to top-level `resource` object (not in each payment requirement)                      |
| `description`       | Moved to top-level `resource.description`                                                   |
| `mimeType`          | Moved to top-level `resource.mimeType`                                                      |
| `outputSchema`      | Removed from payment requirements                                                           |
| `maxTimeoutSeconds` | Now Required (was Optional in v1)                                                           |

**Note on `maxTimeoutSeconds`**: For TXID settlement, `maxTimeoutSeconds` is an advisory payment offer lifetime and is not cryptographically enforced, because the resource server does not commit to the quoted terms on-chain or via signature. It indicates the window during which the client can reasonably expect the quoted terms to remain valid; after this window, the resource server MAY reject the payment proof if pricing or policy has changed.

### TXID Payment Payload (Client → Server)

For TXID settlement, the client returns a payment proof containing the transaction reference and a signature binding it to the payment terms.

#### x402 v1

##### Payment Payload Fields (v1)

| Field Name    | Type     | Required | Description                                                              |
| ------------- | -------- | -------- | ------------------------------------------------------------------------ |
| `x402Version` | `number` | Required | Protocol version identifier (1)                                          |
| `scheme`      | `string` | Required | Payment scheme identifier (e.g., "exact")                                |
| `settlement`  | `string` | Required | Settlement type identifier ("txid" for TXID settlement)                  |
| `network`     | `string` | Required | Blockchain network identifier (e.g., "base-sepolia", "eip155:1") |
| `payload`     | `object` | Required | Payment proof object                                                     |

In v1, the client includes the `paymentRequirements` directly in the payment proof payload. This is the same object the resource server supplies to the facilitator `/verify` API. The field is named `offer` in the payload; this refers directly to the `paymentRequirements` object and does not introduce a new semantic layer.

The `payload` field for v1 contains:

| Field Name            | Type     | Required | Description                                                             |
| --------------------- | -------- | -------- | ----------------------------------------------------------------------- |
| `type`                | `string` | Required | Proof type (`"payment-proof"` for TXID)                                 |
| `alg`                 | `string` | Required | Cryptographic algorithm used for signing (e.g., "ES256K", "Ed25519")    |
| `format`              | `string` | Required | Signing convention/serialization (e.g., "eip712", "solana-signmessage") |
| `txRef`               | `string` | Required | Transaction reference linking to the on-chain payment                   |
| `from`                | `string` | Required | Payer's wallet address                                                  |
| `offer`               | `object` | Required | The PaymentRequirements object as defined in Verify Request Example (v1)|
| `signature`           | `string` | Required | Serialized signature over `txRef` and `paymentRequirements`             |

The 'signature' field is the result of the 'from' private key signing the following object (see Signature Binding for encoding details):

```json
{
  "txRef": "<txRef>",
  "from": "0x857b06519E91e3A54538791bDbb0E22373e36b66",
  "offer": { /* the paymentRequirements object */ }
}
```

##### Payment Payload Example (v1)

```json
{
  "x402Version": 1,
  "scheme": "exact",
  "settlement": "txid",
  "network": "eip155:1",
  "payload": {
    "type": "payment-proof",
    "alg": "ES256K",
    "format": "eip712",
    "txRef": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef",
    "from": "0x857b06519E91e3A54538791bDbb0E22373e36b66",
    "offer": {
      "scheme": "exact",
      "settlement": "txid",
      "network": "eip155:1",
      "maxAmountRequired": "10000",
      "resource": "https://api.example.com/premium-data",
      "description": "Access to premium market data",
      "mimeType": "application/json",
      "payTo": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
      "maxTimeoutSeconds": 60,
      "asset": "native"
    },
    "signature": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef01"
  }
}
```

#### x402 v2

In v2, the payment proof payload is minimal—it contains only `txRef`, `from`, and `signature`. The `resource` and `accepted` fields are already present at the top level of the payment payload (per base x402 v2), so they are not duplicated inside the proof.

##### Payment Payload Fields (v2 Differences)

v2 restructures the payment payload with the following differences:

| Field Name    | Type     | Required | Description                                                              |
| ------------- | -------- | -------- | ------------------------------------------------------------------------ |
| `x402Version` | `number` | Required | Protocol version identifier (2)                                          |
| `resource`    | `object` | Required | Top-level object with `url`, `description`, `mimeType`                   |
| `accepted`    | `object` | Required | Top-level field echoing the chosen entry from `accepts[]`                |
| `payload`     | `object` | Required | Payment proof object                                                     |
| `extensions`  | `object` | Optional | New field for protocol extensions data                                   |

The `payload` field for v2 contains:

| Field Name            | Type     | Required | Description                                                             |
| --------------------- | -------- | -------- | ----------------------------------------------------------------------- |
| `type`                | `string` | Required | Proof type (`"payment-proof"` for TXID)                                 |
| `alg`                 | `string` | Required | Cryptographic algorithm used for signing (e.g., "ES256K", "Ed25519")    |
| `format`              | `string` | Required | Signing convention/serialization (e.g., "eip712", "solana-signmessage") |
| `txRef`               | `string` | Required | Transaction reference linking to the on-chain payment                   |
| `from`                | `string` | Required | Payer's wallet address                                                  |
| `signature`           | `string` | Required | Serialized signature over `txRef` and `paymentRequirements`             |

The 'signature' field is the result of the 'from' private key signing the following object (see Signature Binding for encoding details):

```json
{
  "txRef": "<txRef>",
  "from": "0x857b06519E91e3A54538791bDbb0E22373e36b66",
  "resource": { /* top-level resource object */ },
  "accepted": { /* top-level accepted object */ }
}
```

##### Payment Payload Example (v2)

```json
{
  "x402Version": 2,
  "resource": {
    "url": "https://api.example.com/premium-data",
    "description": "Access to premium market data",
    "mimeType": "application/json"
  },
  "accepted": {
    "scheme": "exact",
    "settlement": "txid",
    "network": "eip155:1",
    "amount": "10000",
    "asset": "native",
    "payTo": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
    "maxTimeoutSeconds": 60
  },
  "payload": {
    "type": "payment-proof",
    "alg": "ES256K",
    "format": "eip712",
    "txRef": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef",
    "from": "0x857b06519E91e3A54538791bDbb0E22373e36b66",
    "signature": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef01"
  },
  "extensions": {}
}
```

### Signature Binding

For TXID settlement, the client MUST include a payer signature that cryptographically commits to the transaction reference and the payment terms. The signing object differs between v1 and v2.

Including `from` in the signing object prevents identity substitution and ensures the signature is bound to the payer account referenced by the on-chain transaction.

#### Supported Signature Formats

The `format` field specifies how the signing object is encoded and signed. The required format is determined by the network's virtual machine type:

| Network Type | Required Format       | Required Algorithm |
| ------------ | --------------------- | ------------------ |
| EVM          | `eip712`              | `ES256K`           |
| Solana       | `solana-signmessage`  | `Ed25519`          |

Clients MUST use the format corresponding to the `network` specified in the payment requirements. Facilitators MUST reject payment proofs that use an incorrect format for the network.

#### Canonicalization

Implementations MUST use deterministic canonicalization when signing. The canonicalization method depends on the signing format:

| Format              | Canonicalization Method                                                |
| ------------------- | ---------------------------------------------------------------------- |
| `eip712`            | EIP-712 typed data encoding (native EIP-712 canonicalization)          |
| `solana-signmessage`| RFC 8785 (JCS) canonical JSON, then UTF-8 encode                       |

**Important**: EIP-712 provides its own deterministic canonicalization and domain separation. JSON canonicalization (RFC 8785 / JCS) MUST NOT be applied when using EIP-712. The two signing paths are distinct and MUST NOT be combined.

##### EIP-712 (Ethereum Structured Data Signing)

**Format identifier**: `"eip712"`

**Domain:**

```javascript
{
  name: "x402 Payment Proof",
  version: "1",
  chainId: <network chainId>
}
```

**Primary Type and Message Structure (v1):**

```javascript
{
  types: {
    EIP712Domain: [
      { name: "name", type: "string" },
      { name: "version", type: "string" },
      { name: "chainId", type: "uint256" }
    ],
    PaymentProof: [
      { name: "txRef", type: "string" },
      { name: "from", type: "address" },
      { name: "offer", type: "Offer" }
    ],
    Offer: [
      { name: "scheme", type: "string" },
      { name: "settlement", type: "string" },
      { name: "network", type: "string" },
      { name: "maxAmountRequired", type: "string" },
      { name: "asset", type: "string" },
      { name: "payTo", type: "address" },
      { name: "resource", type: "string" },
      { name: "description", type: "string" },
      { name: "mimeType", type: "string" },
      { name: "maxTimeoutSeconds", type: "uint256" }
    ]
  },
  primaryType: "PaymentProof",
  message: {
    txRef: "<txRef>",
    from: "<from address>",
    offer: { /* the offer object */ }
  }
}
```

**Primary Type and Message Structure (v2):**

```javascript
{
  types: {
    EIP712Domain: [
      { name: "name", type: "string" },
      { name: "version", type: "string" },
      { name: "chainId", type: "uint256" }
    ],
    PaymentProof: [
      { name: "txRef", type: "string" },
      { name: "from", type: "address" },
      { name: "resource", type: "Resource" },
      { name: "accepted", type: "Accepted" }
    ],
    Resource: [
      { name: "url", type: "string" },
      { name: "description", type: "string" },
      { name: "mimeType", type: "string" }
    ],
    Accepted: [
      { name: "scheme", type: "string" },
      { name: "settlement", type: "string" },
      { name: "network", type: "string" },
      { name: "amount", type: "string" },
      { name: "asset", type: "string" },
      { name: "payTo", type: "address" },
      { name: "maxTimeoutSeconds", type: "uint256" }
    ]
  },
  primaryType: "PaymentProof",
  message: {
    txRef: "<txRef>",
    from: "<from address>",
    resource: { /* top-level resource object */ },
    accepted: { /* top-level accepted object */ }
  }
}
```

**Signature encoding**: Hex-encoded ECDSA signature (0x-prefixed, 65 bytes: r + s + v)

**Verification**: Use EIP-712 typed data hashing and ECDSA signature verification. Support EIP-1271 for smart contract wallets.

##### Solana signMessage (JCS-based)

**Format identifier**: `"solana-signmessage"`

For non-EVM networks like Solana, the signing object is serialized as JSON, canonicalized using JCS (RFC 8785), and then signed.

**Encoding**: The signing object is serialized using JCS (RFC 8785) and UTF-8 encoded with a prefix:

```
x402 Payment Proof\n<JCS(signing object)>
```

**Signing object (v1):**
```json
{
  "txRef": "<txRef>",
  "from": "<from address>",
  "offer": { /* the offer/paymentRequirements object */ }
}
```

**Signing object (v2):**
```json
{
  "txRef": "<txRef>",
  "from": "<from address>",
  "resource": { /* top-level resource object */ },
  "accepted": { /* top-level accepted object */ }
}
```

**Signature encoding**: Base58-encoded Ed25519 signature (64 bytes)

**Verification**: Use Ed25519 signature verification with the public key derived from `from`.

---

### Payload Verification

When a resource server receives a TXID payment proof from a client, it MUST perform the facilitator verification procedure (described below) before delivering the requested service. The resource server SHOULD implement this verification by interfacing with a **facilitator service**. x402 defines two REST APIs for facilitators and this section describes how these APIs can be used with TXID settlement:

| Endpoint       | Purpose                                                                 | Required for TXID |
| -------------- | ----------------------------------------------------------------------- | ----------------- |
| `POST /verify` | Validates the payment proof                                             | **Required**      |
| `POST /settle` | Ensures payment transactions are finalized                              | No          |

**TXID vs EIP-3009**: Unlike EIP-3009 settlement—where the resource server calls both `/verify` (to validate the signature) and `/settle` (to execute the on-chain transfer)—TXID settlement only requires `/verify`. The on-chain transfer has already been executed by the client, so there is no settlement transaction for the facilitator to submit. The `/settle` endpoint is available for resource servers that want explicit settlement tracking, but `/verify` alone is sufficient for TXID.

#### POST /verify

Verifies a TXID payment proof. The resource server calls this endpoint with both the payment payload (from the client) and the original payment requirements (that the server issued).

##### Verify Request Example (v1)

```json
{
  "paymentPayload": {
    "x402Version": 1,
    "scheme": "exact",
    "settlement": "txid",
    "network": "eip155:1",
    "payload": {
      "type": "payment-proof",
      "alg": "ES256K",
      "format": "eip712",
      "txRef": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef",
      "from": "0x857b06519E91e3A54538791bDbb0E22373e36b66",
      "offer": {
        "scheme": "exact",
        "settlement": "txid",
        "network": "eip155:1",
        "maxAmountRequired": "10000",
        "asset": "native",
        "payTo": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
        "resource": "https://api.example.com/premium-data",
        "description": "Access to premium market data",
        "mimeType": "application/json",
        "maxTimeoutSeconds": 60
      },
      "signature": "0x1234567890abcdef..."
    }
  },
  "paymentRequirements": {
    "scheme": "exact",
    "settlement": "txid",
    "network": "eip155:1",
    "maxAmountRequired": "10000",
    "resource": "https://api.example.com/premium-data",
    "description": "Access to premium market data",
    "mimeType": "application/json",
    "payTo": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
    "maxTimeoutSeconds": 60,
    "asset": "native"
  }
}
```

##### Verify Request Example (v2)

```json
{
  "paymentPayload": {
    "x402Version": 2,
    "resource": {
      "url": "https://api.example.com/premium-data",
      "description": "Access to premium market data",
      "mimeType": "application/json"
    },
    "accepted": {
      "scheme": "exact",
      "settlement": "txid",
      "network": "eip155:1",
      "amount": "10000",
      "asset": "native",
      "payTo": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
      "maxTimeoutSeconds": 60
    },
    "payload": {
      "type": "payment-proof",
      "alg": "ES256K",
      "format": "eip712",
      "txRef": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef",
      "from": "0x857b06519E91e3A54538791bDbb0E22373e36b66",
      "signature": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef01"
    }
  },
  "paymentRequirements": {
    "scheme": "exact",
    "network": "eip155:1",
    "amount": "10000",
    "asset": "native",
    "payTo": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
    "maxTimeoutSeconds": 60
  }
}
```

##### Verification Steps

The verification implementation performs the following steps. The `paymentRequirements` object is provided by the resource server and represents the original payment terms.

1. **Reserve the txRef (atomic)**
   - Atomically attempt to reserve `paymentPayload.payload.txRef`
   - If `txRef` is already reserved or consumed, reject immediately
   - This MUST happen first to prevent race conditions where concurrent requests with the same proof could both proceed past this check
   - If any subsequent step fails, release the reservation

2. **Validate signature format matches network**
   - `paymentPayload.payload.format` MUST be `eip712` for EVM networks, `solana-signmessage` for Solana
   - `paymentPayload.payload.alg` MUST be `ES256K` for EVM, `Ed25519` for Solana

3. **Construct the signing object**
   - **For v1**: Construct `{ "txRef": <txRef>, "from": <from>, "offer": <payload.offer> }`
   - **For v2**: Construct `{ "txRef": <txRef>, "from": <from>, "resource": <top-level resource>, "accepted": <top-level accepted> }`

4. **Verify the payment proof signature**
   - **For EVM (EIP-712)**: Encode the signing object as EIP-712 typed data (do NOT use JCS). Compute the EIP-712 hash. Use `ecrecover` with the hash and `paymentPayload.payload.signature` to recover the signer address. The recovered address MUST match `paymentPayload.payload.from`.
   - **For Solana (JCS-based)**: Canonicalize the signing object using JCS (RFC 8785), UTF-8 encode with prefix. Decode `paymentPayload.payload.from` to get the public key. Verify `paymentPayload.payload.signature` against the hash using `ed25519_verify`. Verification MUST succeed.

5. **Validate payment terms binding**
   - **For v1**: `paymentPayload.payload.offer` MUST match the `paymentRequirements` provided by the resource server
   - **For v2**: The top-level `accepted` object MUST match the payment requirements
   - `settlement` MUST be `"txid"`
   - `resource` MUST match the expected resource identifier
   - Payment-relevant fields (`network`, `asset`, `payTo`, amount) MUST match

6. **Fetch and validate the on-chain transaction**
   - Locate the transaction identified by `paymentPayload.payload.txRef`
   - Transaction MUST be confirmed to the required confirmation depth
   - Transaction payer/authority (e.g., `tx.from` on EVM, fee payer or equivalent on other networks) MUST match`paymentPayload.payload.from`

7. **Validate the payment transfer against payment terms**
   - Transaction MUST have transferred at least the amount specified in `paymentRequirements.maxAmountRequired` (v1) or `accepted.amount` (v2)
   - Asset MUST match `paymentRequirements.asset` (v1) or `accepted.asset` (v2)
   - Transfer recipient MUST match `paymentRequirements.payTo` (v1) or `accepted.payTo` (v2)
   - Transaction timestamp MUST be within the retention window (see Replay and Idempotency Requirements)

8. **Mark txRef as consumed**
   - If all checks pass, mark the reserved `txRef` as permanently consumed
   - The `txRef` record is subject to the retention policy

##### /verify Response

The facilitator /verify API MUST respond with either 'success' or 'error' depending on the verification result.

**Successful Response:**

```json
{
  "isValid": true,
  "payer": "0x857b06519E91e3A54538791bDbb0E22373e36b66"
}
```

**Error Response:**

```json
{
  "isValid": false,
  "invalidReason": "payment_not_found", // see below for additional error reasons
  "payer": "0x857b06519E91e3A54538791bDbb0E22373e36b66"
}
```

#### POST /settle

For TXID settlement, the `/settle` endpoint is optional. The on-chain transaction has already been executed by the client, so there is no settlement transaction for the facilitator to submit.

Resource servers MAY implement `/settle` for TXID as an internal operation (e.g., logging, tracking prepaid balances, or finalizing service delivery), but it is not required since `/verify` is sufficient.

### Payment Payload Response

After successful verification, the resource server delivers the requested resource to the client— the service or data the client paid for. The response format is defined by the resource server, not by this specification.

**Success example:**

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "data": { ... }  // The actual resource content the client paid for
}
```

**Verification failure example:**

```
HTTP/1.1 402 Payment Required
Content-Type: application/json

{
  "error": "payment_not_found",
  "accepts": [ ... ]  // Payment requirements for retry
}
```

TXID settlement defines how payment verification works. Once payment is verified, the resource server delivers whatever service the client paid for according to the resource's own response format.

## TXID Settlement Properties

TXID settlement has the following defining properties:

- **Payer-executed on-chain transfer**
  The payer submits the on-chain transaction that moves funds.

- **Transaction-reference proof**
  The client proves payment by providing `txRef` and a payer signature proving that the resource requester controls the account that funded the transaction.

- **Facilitator verification (no facilitator-submitted transaction)**
  The facilitator verifies the payment proof and confirms the on-chain transaction, but does not submit a transaction. The resource server MAY self-host the facilitator.

- **Freshness via retention policy**
  Facilitators enforce transaction freshness through retention window policies and transaction timestamp checks, rather than cryptographically-bound quote windows.

- **Client Identity**
The **payer** that actually funded the on-chain transfer referenced by `txRef` (i.e., the transaction sender or equivalent chain-specific payer/authority) also provides the payment proof signature.  This essentially ties the identity of the client to the payer.

## Replay and Idempotency Requirements

Because TXID relies on a transaction reference rather than a one-time on-chain authorization primitive, replay protection for **service delivery** is essential.

### Responsibility

The **resource server** is ultimately responsible for ensuring correct payment verification before delivering service—it is the party providing the service and booking the revenue. The x402 specification integrates a facilitator service, but this delegation does not transfer responsibility. If the resource server uses a facilitator, it is responsible for choosing a trustworthy facilitator and ensuring the facilitator implements the required behavior correctly.

### Replay Scope and Consequences

Replay of TXID payment proofs can only result in duplicate service delivery. Replay cannot cause additional fund transfers without payer consent—the on-chain transaction has already been executed and cannot be replayed on-chain.

### Required Behavior

For the **exact** payment scheme (one payment = one service delivery), the verification implementation (whether self-hosted or via facilitator) MUST:

1. **De-duplicate by transaction reference**
   A given `txRef` MUST be accepted at most once for successful service authorization. Facilitators MUST track consumed `txRef` values and MUST reject reuse of `txRef` values. Facilitators MUST mark `txRef` as consumed before returning success.

2. **Enforce retention window**
   Define a retention window `T` (e.g., 1–24 hours) for storing consumed `txRef` values. The facilitator MUST reject any `txRef` whose on-chain timestamp is older than `T` relative to verification time.

**Note**: Other payment schemes (e.g., `deferred`, `prepaid`) may define different replay semantics where a single `txRef` can authorize multiple service deliveries by tracking and decrementing a balance. Such schemes are outside the scope of this specification.

## Transport Security

TXID payment proofs MUST be transmitted over a secure, authenticated transport (e.g., HTTPS). Payment proofs MUST NOT be transmitted over cleartext or unauthenticated channels.

TXID's resistance to service theft relies on the payment proof remaining off-chain and private. Because TXID payment proofs are bearer-like at the application layer (a valid proof can be replayed unless the facilitator enforces idempotency), transport-layer security is essential.

## TXID Security Posture

### Strengths

- **Broad asset and chain support**: does not require specialized token contract interfaces.
- **No facilitator gas requirement**: payer submits the transaction and pays network fees.
- **Private proof delivery** (transport-protected): the payment proof is not inherently published on-chain as structured authorization data.
- **No additional fund transfer risk**: replay of payment proofs cannot cause additional fund transfers—only duplicate service delivery.

### Tradeoffs

- **Facilitator-side replay risk for service delivery**: facilitators must implement `txRef` de-duplication and time-window enforcement.
- **Indexing/lookup dependency**: verification requires reliable access to chain transaction data and transfer parsing.
- **Proof format complexity**: multi-format signature verification introduces algorithm and canonicalization risks that must be constrained by whitelists and strict encoding rules.

### Responsibility and Risk

TXID security depends on correct facilitator implementation. Facilitators choose their own risk tolerance for reorgs, replay windows, and storage duration. Incorrect implementation can cause duplicate service delivery but cannot cause unauthorized fund transfers.

## Use Cases (Informative)

TXID settlement addresses several important use cases that are difficult or impossible with authorization-based settlement mechanisms like EIP-3009.

### Economically Truthful Payment Model

With EIP-3009 settlement, the facilitator submits the on-chain transaction and pays gas fees—even though the facilitator typically does not receive the payment revenue. This creates an economic mismatch where facilitators must subsidize gas costs or pass them on indirectly.

TXID settlement eliminates this mismatch: the payer submits the transaction and pays gas directly. The facilitator only performs verification, which requires no on-chain transaction and no gas expenditure. This aligns costs with the party who benefits from the service.

### Native Token Payments

TXID settlement allows payers to use native tokens (ETH, SOL, etc.) to pay for both:
- the x402 service itself, and
- the gas/transaction fees required to execute the payment.

With EIP-3009, payments are limited to tokens that implement the `transferWithAuthorization` interface. Native tokens cannot be used. TXID removes this limitation, enabling payments in any transferable asset.

### Token Utility and Ecosystem Support

TXID settlement enables any token to be used for x402 payments, bringing additional utility and demand to tokens that may not have EIP-3009 support. This is particularly valuable for:
- tokens that fund decentralized projects or DAOs,
- community tokens that want to enable real-world utility,
- newer tokens that haven't implemented authorization-based transfer interfaces.

For example, a project like OMA3 (the author of this specification) would be able to accept x402 payments in the native token of its OMAChain Ethereum rollup.

### Simplified Multi-Chain Support

With authorization-based settlement, each blockchain/VM requires a specific signed-transfer standard (EIP-3009 for EVM, SPL Token extensions for Solana, etc.). Resource servers must implement and maintain verification logic for each standard.

TXID settlement simplifies multi-chain support: the core verification logic (signature verification + on-chain transaction lookup) follows the same pattern across chains. Resource servers can more easily support a wide range of VMs without requiring each chain to implement a specific authorization-based transfer mechanism.

## Version History

| Version | Date       | Changes                                                                 | Author     |
| ------- | ---------- | ----------------------------------------------------------------------- | ---------- |
| v0.1    | 2025-12-18 | Initial draft                                                           | Alfred Tom |
