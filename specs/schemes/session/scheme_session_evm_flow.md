# Session Scheme: EVM Full Lifecycle Examples

# Lifecycle 1: Self-Contained Session

API charges **$0.10 per lookup** in USDC. The client opens a channel with a $1.00 deposit (enough for 10 requests), uses it, tops up when the deposit runs out and eventually closes.

## Actors & Constants

| Role                 | Value                                          |
| :------------------- | :--------------------------------------------- |
| Client (payer)       | `0xClientAddress`                              |
| Server (payee)       | `0xServerPayeeAddress`                         |
| Facilitator signer   | `0xFacilitatorSignerAddress`                   |
| USDC on Base         | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`  |
| Network              | `eip155:8453` (Base)                           |
| Price per request    | `100000` ($0.10 USDC, 6 decimals)              |

## Lifecycle Summary

| Step | Action        | Cumulative | Deposit   | Onchain              | Client Spend | Client Refund |
| :--- | :------------ | :--------- | :-------- | :------------------- | :----------- | :------------ |
| 1    | Initial 402   | —          | —         | —                    | —            | —             |
| 2    | Channel Open  | $0.10      | $1.00     | `openWithERC3009`    | $1.00        | —             |
| 3    | Voucher       | $0.20      | $1.00     | —                    | —            | —             |
| …    | Requests 3–10 | $1.00      | $1.00     | —                    | —            | —             |
| 4    | Top-Up        | $1.10      | $1.50     | `topUpWithERC3009`   | $0.50        | —             |
| 5    | Close         | $1.20      | $1.50     | `close`              | —            | $0.30         |

---

## Step 1: Initial 402 — Client Hits the API for the First Time

The client requests the API. The server has no channel for this client and returns 402. No `channelId` in `extra` signals that a new channel must be opened.

**Client request:**

```http
GET /api/resource?q=example HTTP/1.1
Host: api.example.com
```

**Server response — 402 with `PAYMENT-REQUIRED` header (base64-decoded):**

```json
{
  "x402Version": 2,
  "error": "PAYMENT-SIGNATURE header is required",
  "resource": {
    "url": "https://api.example.com/",
    "description": "API",
    "mimeType": "application/json"
  },
  "accepts": [
    {
      "scheme": "session",
      "network": "eip155:8453",
      "amount": "100000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0xServerPayeeAddress",
      "maxTimeoutSeconds": 3600,
      "extra": {
        "authorizedSettler": "0xFacilitatorSignerAddress"
      }
    }
  ]
}
```

> No `channelId` in `extra` → client must open a new channel.

---

## Step 2: Channel Open — Client Deposits $1.00, Signs First Voucher for $0.10

The client decides to deposit $1.00 (`1000000`) to cover ~10 requests. It signs two things off-chain:

1. An **ERC-3009 `receiveWithAuthorization`** transferring $1.00 USDC to the channel contract
2. An **EIP-712 voucher** for $0.10 cumulative (`100000`) — payment for this first request

### `PAYMENT-SIGNATURE` header (base64-decoded)

```json
{
  "x402Version": 2,
  "accepted": {
    "scheme": "session",
    "network": "eip155:8453",
    "amount": "100000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "payTo": "0xServerPayeeAddress",
    "maxTimeoutSeconds": 3600,
    "extra": {
      "authorizedSettler": "0xFacilitatorSignerAddress"
    }
  },
  "payload": {
    "type": "channelOpen",
    "channelOpen": {
      "payer": "0xClientAddress",
      "payee": "0xServerPayeeAddress",
      "token": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "deposit": "1000000",
      "salt": "0x...keccak256(abi.encode('x402-session', uint256(0)))",
      "authorizedSigner": "0x0000000000000000000000000000000000000000",
      "authorizedSettler": "0xFacilitatorSignerAddress",
      "erc3009Authorization": {
        "validAfter": 0,
        "validBefore": 1711929600,
        "nonce": "0xe4f8b1c2d3a4e5f6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0",
        "signature": "0x...ERC-3009 ReceiveWithAuthorization signature from client"
      }
    },
    "voucher": {
      "channelId": "0xabc123...channelId",
      "cumulativeAmount": "100000",
      "signature": "0x...EIP-712 Voucher signature from client"
    }
  }
}
```

### Server → Facilitator: `POST /verify` — validate deposit authorization + voucher

The server forwards the payload to the facilitator for validation. The facilitator verifies the ERC-3009 authorization parameters, checks that the client has sufficient token balance for the deposit, recovers the EIP-712 signer from the voucher, confirms it matches the `channelOpen` payer (or `authorizedSigner`), and validates the amount. No onchain transaction occurs at this stage — funds are not yet in escrow.

**Request:**

```json
{
  "x402Version": 2,
  "paymentPayload": {
    "x402Version": 2,
    "accepted": {
      "scheme": "session",
      "network": "eip155:8453",
      "amount": "100000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0xServerPayeeAddress",
      "maxTimeoutSeconds": 3600,
      "extra": {
        "authorizedSettler": "0xFacilitatorSignerAddress"
      }
    },
    "payload": {
      "type": "channelOpen",
      "channelOpen": {
        "payer": "0xClientAddress",
        "payee": "0xServerPayeeAddress",
        "token": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
        "deposit": "1000000",
        "salt": "0x...keccak256(abi.encode('x402-session', uint256(0)))",
        "authorizedSigner": "0x0000000000000000000000000000000000000000",
        "authorizedSettler": "0xFacilitatorSignerAddress",
        "erc3009Authorization": {
          "validAfter": 0,
          "validBefore": 1711929600,
          "nonce": "0xe4f8b1c2d3a4e5f6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0",
          "signature": "0x...ERC-3009 ReceiveWithAuthorization signature from client"
        }
      },
      "voucher": {
        "channelId": "0xabc123...channelId",
        "cumulativeAmount": "100000",
        "signature": "0x...EIP-712 Voucher signature from client"
      }
    }
  },
  "paymentRequirements": {
    "scheme": "session",
    "network": "eip155:8453",
    "amount": "100000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "payTo": "0xServerPayeeAddress",
    "maxTimeoutSeconds": 3600,
    "extra": {
      "authorizedSettler": "0xFacilitatorSignerAddress"
    }
  }
}
```

**Response:**

```json
{
  "isValid": true,
  "payer": "0xClientAddress"
}
```

### Server → Facilitator: `POST /settle` — open the channel onchain

After verification succeeds, the server calls `/settle`. The facilitator sees `payload.type: "channelOpen"` and calls `openWithERC3009()` on the channel contract, paying gas. This executes the ERC-3009 deposit and creates the channel onchain.

**Request:**

```json
{
  "x402Version": 2,
  "paymentPayload": "« same paymentPayload as /verify request above »",
  "paymentRequirements": {
    "scheme": "session",
    "network": "eip155:8453",
    "amount": "100000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "payTo": "0xServerPayeeAddress",
    "maxTimeoutSeconds": 3600,
    "extra": {
      "authorizedSettler": "0xFacilitatorSignerAddress"
    }
  }
}
```

**Response:**

```json
{
  "success": true,
  "transaction": "0x...openWithERC3009 txHash",
  "network": "eip155:8453",
  "payer": "0xClientAddress",
  "extra": {
    "channelId": "0xabc123...channelId",
    "cumulativeAmount": "100000",
    "deposit": "1000000"
  }
}
```

### `PAYMENT-RESPONSE` header (base64-decoded)

The server returns 200 with the resource and a `PAYMENT-RESPONSE` header containing the new channel state. The client stores `channelId`, `cumulativeAmount`, and `deposit` for future requests.

```json
{
  "success": true,
  "transaction": "0x...openWithERC3009 txHash",
  "network": "eip155:8453",
  "payer": "0xClientAddress",
  "extra": {
    "channelId": "0xabc123...channelId",
    "cumulativeAmount": "100000",
    "deposit": "1000000"
  }
}
```

> Channel state: deposit = $1.00, spent = $0.10, remaining = $0.90

---

## Step 3: Second Request — Client Signs Voucher for $0.20 Cumulative (No Onchain Transaction)

The client makes another request. The server returns a **generic 402** (no channel state) — just the price and requirements. The client already knows its channel state from the `PAYMENT-RESPONSE` in Step 2. It signs a new cumulative voucher incrementing by $0.10 and sends it.

The server reads the `channelId` from the client's payload, looks up its own per-channel state, and includes it in `paymentRequirements.extra` when forwarding to the facilitator. The facilitator verifies the implied base against the server's truth.

### `PAYMENT-REQUIRED` header (base64-decoded)

```json
{
  "x402Version": 2,
  "error": "PAYMENT-SIGNATURE header is required",
  "accepts": [
    {
      "scheme": "session",
      "network": "eip155:8453",
      "amount": "100000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0xServerPayeeAddress",
      "maxTimeoutSeconds": 3600,
      "extra": {
        "authorizedSettler": "0xFacilitatorSignerAddress"
      }
    }
  ]
}
```

> Generic 402 — no channel state. Client uses its own state from the last `PAYMENT-RESPONSE`:
> `channelId = "0xabc123..."`, `cumulativeAmount = 100000`, `deposit = 1000000`.
> Client computes: 100000 + 100000 = 200000 ≤ 1000000 (deposit) → no top-up needed.

### `PAYMENT-SIGNATURE` header (base64-decoded)

The client signs a voucher using its own channel state:

```json
{
  "x402Version": 2,
  "accepted": {
    "scheme": "session",
    "network": "eip155:8453",
    "amount": "100000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "payTo": "0xServerPayeeAddress",
    "maxTimeoutSeconds": 3600,
    "extra": {
      "authorizedSettler": "0xFacilitatorSignerAddress"
    }
  },
  "payload": {
    "type": "voucher",
    "channelId": "0xabc123...channelId",
    "cumulativeAmount": "200000",
    "signature": "0x...EIP-712 Voucher signature (cumulative $0.20)"
  }
}
```

### Server → Facilitator: `POST /verify` — cross-check + signature verification

The server reads `channelId` from the payload, looks up its own state (`lastCumulativeAmount = 100000`, `deposit = 1000000`), and includes it in `paymentRequirements.extra`:

**Request:**

```json
{
  "x402Version": 2,
  "paymentPayload": {
    "x402Version": 2,
    "accepted": {
      "scheme": "session",
      "network": "eip155:8453",
      "amount": "100000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0xServerPayeeAddress",
      "maxTimeoutSeconds": 3600,
      "extra": {
        "authorizedSettler": "0xFacilitatorSignerAddress"
      }
    },
    "payload": {
      "type": "voucher",
      "channelId": "0xabc123...channelId",
      "cumulativeAmount": "200000",
      "signature": "0x...EIP-712 Voucher signature (cumulative $0.20)"
    }
  },
  "paymentRequirements": {
    "scheme": "session",
    "network": "eip155:8453",
    "amount": "100000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "payTo": "0xServerPayeeAddress",
    "maxTimeoutSeconds": 3600,
    "extra": {
      "authorizedSettler": "0xFacilitatorSignerAddress",
      "channelId": "0xabc123...channelId",
      "cumulativeAmount": "100000",
      "deposit": "1000000"
    }
  }
}
```

> Facilitator checks:
> 1. `payload.cumulativeAmount (200000) == paymentRequirements.extra.cumulativeAmount (100000) + amount (100000)` ✓ correct increment (implies matching base)
> 2. `payload.cumulativeAmount (200000) <= paymentRequirements.extra.deposit (1000000)` ✓ within deposit

**Response:**

```json
{
  "isValid": true,
  "payer": "0xClientAddress"
}
```

> No `/settle` call — the server defers onchain settlement and accumulates vouchers.

### `PAYMENT-RESPONSE` header (base64-decoded)

```json
{
  "success": true,
  "network": "eip155:8453",
  "payer": "0xClientAddress",
  "extra": {
    "channelId": "0xabc123...channelId",
    "cumulativeAmount": "200000",
    "deposit": "1000000"
  }
}
```

> Channel state: deposit = $1.00, spent = $0.20, remaining = $0.80
>
> Requests 3–10 follow the same voucher pattern. Each increments `cumulativeAmount` by 100000.
> After request 10: `cumulativeAmount` = 1000000 = `deposit`.

---

## Step 4: Top-Up — Deposit Exhausted After 10 Requests, Client Adds $0.50

On the 11th request, the server returns a generic 402 (price only). The client knows from its own state (last `PAYMENT-RESPONSE`) that `cumulativeAmount = 1000000 = deposit = 1000000`. Adding another $0.10 would exceed the deposit, so it signs both an ERC-3009 authorization for a $0.50 top-up and a new voucher.

### `PAYMENT-REQUIRED` header (base64-decoded)

```json
{
  "x402Version": 2,
  "error": "PAYMENT-SIGNATURE header is required",
  "accepts": [
    {
      "scheme": "session",
      "network": "eip155:8453",
      "amount": "100000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0xServerPayeeAddress",
      "maxTimeoutSeconds": 3600,
      "extra": {
        "authorizedSettler": "0xFacilitatorSignerAddress"
      }
    }
  ]
}
```

> Generic 402 — no channel state. Client uses its own state:
> `cumulativeAmount = 1000000`, `deposit = 1000000`.
> Client computes: 1000000 + 100000 = 1100000 > 1000000 (deposit) → top-up required.
> Client decides to add $0.50 (500000), bringing deposit to $1.50 (1500000) for 5 more requests.

### `PAYMENT-SIGNATURE` header (base64-decoded)

```json
{
  "x402Version": 2,
  "accepted": {
    "scheme": "session",
    "network": "eip155:8453",
    "amount": "100000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "payTo": "0xServerPayeeAddress",
    "maxTimeoutSeconds": 3600,
    "extra": {
      "authorizedSettler": "0xFacilitatorSignerAddress"
    }
  },
  "payload": {
    "type": "topUp",
    "topUp": {
      "channelId": "0xabc123...channelId",
      "additionalDeposit": "500000",
      "erc3009Authorization": {
        "validAfter": 0,
        "validBefore": 1711933200,
        "nonce": "0xb2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3",
        "signature": "0x...ERC-3009 ReceiveWithAuthorization signature for $0.50 top-up"
      }
    },
    "voucher": {
      "channelId": "0xabc123...channelId",
      "cumulativeAmount": "1100000",
      "signature": "0x...EIP-712 Voucher signature (cumulative $1.10)"
    }
  }
}
```

### Server → Facilitator: `POST /verify` — validate top-up authorization + voucher

The server reads the `channelId` from the payload, looks up its own state, and includes it in `paymentRequirements.extra`. The facilitator verifies the ERC-3009 authorization parameters for the additional deposit, validates the amount increment against the server's truth, validates the voucher signature, and confirms the new cumulative amount does not exceed the deposit + top-up. No onchain transaction occurs at this stage.

**Request:**

```json
{
  "x402Version": 2,
  "paymentPayload": "« PAYMENT-SIGNATURE above »",
  "paymentRequirements": {
    "scheme": "session",
    "network": "eip155:8453",
    "amount": "100000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "payTo": "0xServerPayeeAddress",
    "maxTimeoutSeconds": 3600,
    "extra": {
      "authorizedSettler": "0xFacilitatorSignerAddress",
      "channelId": "0xabc123...channelId",
      "cumulativeAmount": "1000000",
      "deposit": "1000000"
    }
  }
}
```

**Response:**

```json
{
  "isValid": true,
  "payer": "0xClientAddress"
}
```

### Server → Facilitator: `POST /settle` — top up channel onchain

After verification succeeds, the server calls `/settle`. The facilitator sees `payload.type: "topUp"` and calls `topUpWithERC3009()`, depositing an additional $0.50 into the channel onchain.

**Request:**

```json
{
  "x402Version": 2,
  "paymentPayload": "« PAYMENT-SIGNATURE above »",
  "paymentRequirements": {
    "scheme": "session",
    "network": "eip155:8453",
    "amount": "100000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "payTo": "0xServerPayeeAddress",
    "maxTimeoutSeconds": 3600,
    "extra": {
      "authorizedSettler": "0xFacilitatorSignerAddress",
      "channelId": "0xabc123...channelId",
      "cumulativeAmount": "1000000",
      "deposit": "1000000"
    }
  }
}
```

**Response:**

```json
{
  "success": true,
  "transaction": "0x...topUpWithERC3009 txHash",
  "network": "eip155:8453",
  "payer": "0xClientAddress",
  "extra": {
    "channelId": "0xabc123...channelId",
    "cumulativeAmount": "1100000",
    "deposit": "1500000"
  }
}
```

### `PAYMENT-RESPONSE` header (base64-decoded)

```json
{
  "success": true,
  "transaction": "0x...topUpWithERC3009 txHash",
  "network": "eip155:8453",
  "payer": "0xClientAddress",
  "extra": {
    "channelId": "0xabc123...channelId",
    "cumulativeAmount": "1100000",
    "deposit": "1500000"
  }
}
```

> Channel state: deposit = $1.50, spent = $1.10, remaining = $0.40 (4 more requests)

---

## Step 5: Close — Client Signs Final Voucher for $1.20 and Requests Channel Closure

The client makes one more request and signals it is done by setting `requestClose: true`. The server returns a generic 402. The client knows from its own state that `cumulativeAmount = 1100000`, `deposit = 1500000`. The server verifies the voucher, serves the content, then tells the facilitator to close the channel. The contract settles $1.20 to the server and refunds $0.30 to the client.

### `PAYMENT-REQUIRED` header (base64-decoded)

```json
{
  "x402Version": 2,
  "error": "PAYMENT-SIGNATURE header is required",
  "accepts": [
    {
      "scheme": "session",
      "network": "eip155:8453",
      "amount": "100000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0xServerPayeeAddress",
      "maxTimeoutSeconds": 3600,
      "extra": {
        "authorizedSettler": "0xFacilitatorSignerAddress"
      }
    }
  ]
}
```

> Generic 402 — no channel state. Client uses its own state:
> `cumulativeAmount = 1100000`, `deposit = 1500000`.
> Client computes: 1100000 + 100000 = 1200000 ≤ 1500000 → no top-up needed.
> Client is done and sets `requestClose: true`.

### `PAYMENT-SIGNATURE` header (base64-decoded)

```json
{
  "x402Version": 2,
  "accepted": {
    "scheme": "session",
    "network": "eip155:8453",
    "amount": "100000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "payTo": "0xServerPayeeAddress",
    "maxTimeoutSeconds": 3600,
    "extra": {
      "authorizedSettler": "0xFacilitatorSignerAddress"
    }
  },
  "payload": {
    "type": "voucher",
    "channelId": "0xabc123...channelId",
    "cumulativeAmount": "1200000",
    "signature": "0x...EIP-712 Voucher signature (cumulative $1.20)",
    "requestClose": true
  }
}
```

### Server → Facilitator: `POST /verify` — voucher signature + amount increment

The server includes its channel state in `paymentRequirements.extra`:

**Request:**

```json
{
  "x402Version": 2,
  "paymentPayload": {
    "x402Version": 2,
    "accepted": {
      "scheme": "session",
      "network": "eip155:8453",
      "amount": "100000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0xServerPayeeAddress",
      "maxTimeoutSeconds": 3600,
      "extra": {
        "authorizedSettler": "0xFacilitatorSignerAddress"
      }
    },
    "payload": {
      "type": "voucher",
      "channelId": "0xabc123...channelId",
      "cumulativeAmount": "1200000",
      "signature": "0x...EIP-712 Voucher signature (cumulative $1.20)",
      "requestClose": true
    }
  },
  "paymentRequirements": {
    "scheme": "session",
    "network": "eip155:8453",
    "amount": "100000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "payTo": "0xServerPayeeAddress",
    "maxTimeoutSeconds": 3600,
    "extra": {
      "authorizedSettler": "0xFacilitatorSignerAddress",
      "channelId": "0xabc123...channelId",
      "cumulativeAmount": "1100000",
      "deposit": "1500000"
    }
  }
}
```

**Response:**

```json
{
  "isValid": true,
  "payer": "0xClientAddress"
}
```

### Server → Facilitator: `POST /settle` — close channel onchain

The facilitator sees `requestClose: true` in the voucher payload and calls `close(channelId, 1200000, voucherSignature)` as the `authorizedSettler`. The contract settles the final amount to the server and refunds the remainder to the client.

**Request:**

```json
{
  "x402Version": 2,
  "paymentPayload": "« PAYMENT-SIGNATURE above »",
  "paymentRequirements": {
    "scheme": "session",
    "network": "eip155:8453",
    "amount": "100000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "payTo": "0xServerPayeeAddress",
    "maxTimeoutSeconds": 3600,
    "extra": {
      "authorizedSettler": "0xFacilitatorSignerAddress",
      "channelId": "0xabc123...channelId",
      "cumulativeAmount": "1100000",
      "deposit": "1500000"
    }
  }
}
```

**Response:**

```json
{
  "success": true,
  "transaction": "0x...close txHash",
  "network": "eip155:8453",
  "payer": "0xClientAddress"
}
```

> Onchain result of `close()`:
> - $1.20 (1200000) settled to `0xServerPayeeAddress`
> - $0.30 (300000) refunded to `0xClientAddress`
> - Channel marked as `finalized`

### `PAYMENT-RESPONSE` header (base64-decoded)

```json
{
  "success": true,
  "transaction": "0x...close txHash",
  "network": "eip155:8453",
  "payer": "0xClientAddress"
}
```

> Session complete. Total: 12 requests, $1.20 paid, $0.30 refunded. Three onchain transactions total (open, topUp, close).

---
---

# Lifecycle 2: Resuming a Previous Session

The client returns days later. The channel from Lifecycle 1 was **not** closed (imagine the client skipped the close step, or this is a different prior session). The client has lost all in-memory state — it has no `channelId`, `cumulativeAmount`, or `deposit`.

## Actors & Constants (Same as Lifecycle 1)

| Role                 | Value                                          |
| :------------------- | :--------------------------------------------- |
| Client (payer)       | `0xClientAddress`                              |
| Server (payee)       | `0xServerPayeeAddress`                         |
| Facilitator signer   | `0xFacilitatorSignerAddress`                   |
| USDC on Base         | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`  |
| Network              | `eip155:8453` (Base)                           |
| Price per request    | `100000` ($0.10 USDC, 6 decimals)              |

## Prior Channel State (from a previous session)

| Field              | Value       | Description                         |
| :----------------- | :---------- | :---------------------------------- |
| `channelId`        | `0xdef456...` | Open channel from a prior session  |
| `deposit`          | `1000000`   | $1.00 deposited                     |
| `settled` (onchain)| `500000`    | $0.50 settled onchain by facilitator|
| Server's `lastCumulativeAmount` | `800000` | $0.80 — server holds unsettled vouchers for $0.30 |

## Lifecycle Summary

| Step | Action                                    | Outcome                                              |
| :--- | :---------------------------------------- | :--------------------------------------------------- |
| 1    | Generic 402                               | Client learns price, no channel state                 |
| 2a   | Contract Read + Voucher (stale → retry)   | Client discovers channel, anchors to `settled`, gets corrected |
| 2b   | SIWX-Assisted Resume (alternative)        | Client identified, gets channel state in 402 directly |
| 3    | Subsequent requests                       | Normal voucher flow (same as Lifecycle 1)             |

---

## Step 1: Generic 402

The client hits the API. The server returns a generic 402 (no channel state, because it cannot identify the client).

**Client request:**

```http
GET /api/resource?q=example HTTP/1.1
Host: api.example.com
```

**Server response — `PAYMENT-REQUIRED` header (base64-decoded):**

```json
{
  "x402Version": 2,
  "error": "PAYMENT-SIGNATURE header is required",
  "accepts": [
    {
      "scheme": "session",
      "network": "eip155:8453",
      "amount": "100000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0xServerPayeeAddress",
      "maxTimeoutSeconds": 3600,
      "extra": {
        "authorizedSettler": "0xFacilitatorSignerAddress"
      }
    }
  ]
}
```

> Generic 402 — no channel state. The client must discover its channel.

---

## Step 2a: Contract Read Path — Discover Channel, Anchor to Settled

The client uses the deterministic salt convention to compute potential channel IDs and calls `getChannelsBatch()` to discover open channels.

### Channel Discovery

```
salt_0 = keccak256(abi.encode("x402-session", uint256(0)))
salt_1 = keccak256(abi.encode("x402-session", uint256(1)))

channelId_0 = keccak256(abi.encode(
  0xClientAddress,
  0xServerPayeeAddress,
  0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913,
  salt_0,
  0x0000000000000000000000000000000000000000,
  0xFacilitatorSignerAddress,
  contractAddress,
  8453
))
// = 0xdef456... (matches the open channel from the prior session)
```

**Contract call: `getChannelsBatch([channelId_0, channelId_1])`**

```json
[
  {
    "payer": "0xClientAddress",
    "payee": "0xServerPayeeAddress",
    "token": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "deposit": "1000000",
    "settled": "500000",
    "finalized": false
  },
  {
    "payer": "0x0000000000000000000000000000000000000000",
    "finalized": false
  }
]
```

> Channel 0: `payer != 0x0` and `finalized == false` → **open channel**. Remaining: `1000000 - 500000 = 500000 >= 100000` ✓
> Channel 1: `payer == 0x0` and `finalized == false` → **never opened** → stop.

### Voucher Anchored to Onchain `settled`

The client anchors to the onchain `settled` amount (500000) since it has no other state:

**`PAYMENT-SIGNATURE` header (base64-decoded):**

```json
{
  "x402Version": 2,
  "accepted": {
    "scheme": "session",
    "network": "eip155:8453",
    "amount": "100000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "payTo": "0xServerPayeeAddress",
    "maxTimeoutSeconds": 3600,
    "extra": {
      "authorizedSettler": "0xFacilitatorSignerAddress"
    }
  },
  "payload": {
    "type": "voucher",
    "channelId": "0xdef456...",
    "cumulativeAmount": "600000",
    "signature": "0x...EIP-712 Voucher signature (cumulative $0.60)"
  }
}
```

> Client signs `payload.cumulativeAmount = 500000 (onchain settled) + 100000 = 600000`.

### Server → Facilitator: `POST /verify` — Stale Base Detected

The server reads `channelId` from the payload, looks up its own state (`lastCumulativeAmount = 800000`), and includes it in `paymentRequirements.extra`:

**Request:**

```json
{
  "x402Version": 2,
  "paymentPayload": "« PAYMENT-SIGNATURE above »",
  "paymentRequirements": {
    "scheme": "session",
    "network": "eip155:8453",
    "amount": "100000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "payTo": "0xServerPayeeAddress",
    "maxTimeoutSeconds": 3600,
    "extra": {
      "authorizedSettler": "0xFacilitatorSignerAddress",
      "channelId": "0xdef456...",
      "cumulativeAmount": "800000",
      "deposit": "1000000"
    }
  }
}
```

**Facilitator cross-check:**

1. `payload.cumulativeAmount (600000) == paymentRequirements.extra.cumulativeAmount (800000) + amount (100000) = 900000` ✗ **MISMATCH** (implied base 500000 ≠ server truth 800000)

**Response:**

```json
{
  "isValid": false,
  "invalidReason": "session_stale_cumulative_amount"
}
```

### Server → Client: Corrective 402

The server returns a 402 **with** its per-channel state so the client can retry:

**`PAYMENT-REQUIRED` header (base64-decoded):**

```json
{
  "x402Version": 2,
  "error": "session_stale_cumulative_amount",
  "accepts": [
    {
      "scheme": "session",
      "network": "eip155:8453",
      "amount": "100000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0xServerPayeeAddress",
      "maxTimeoutSeconds": 3600,
      "extra": {
        "authorizedSettler": "0xFacilitatorSignerAddress",
        "channelId": "0xdef456...",
        "cumulativeAmount": "800000",
        "deposit": "1000000"
      }
    }
  ]
}
```

### Client Retries with Correct Base

**`PAYMENT-SIGNATURE` header (base64-decoded):**

```json
{
  "x402Version": 2,
  "accepted": {
    "scheme": "session",
    "network": "eip155:8453",
    "amount": "100000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "payTo": "0xServerPayeeAddress",
    "maxTimeoutSeconds": 3600,
    "extra": {
      "authorizedSettler": "0xFacilitatorSignerAddress",
      "channelId": "0xdef456...",
      "cumulativeAmount": "800000",
      "deposit": "1000000"
    }
  },
  "payload": {
    "type": "voucher",
    "channelId": "0xdef456...",
    "cumulativeAmount": "900000",
    "signature": "0x...EIP-712 Voucher signature (cumulative $0.90)"
  }
}
```

### Server → Facilitator: `POST /verify` — Now Matches

**Request:**

```json
{
  "x402Version": 2,
  "paymentPayload": "« PAYMENT-SIGNATURE above »",
  "paymentRequirements": {
    "scheme": "session",
    "network": "eip155:8453",
    "amount": "100000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "payTo": "0xServerPayeeAddress",
    "maxTimeoutSeconds": 3600,
    "extra": {
      "authorizedSettler": "0xFacilitatorSignerAddress",
      "channelId": "0xdef456...",
      "cumulativeAmount": "800000",
      "deposit": "1000000"
    }
  }
}
```

> Facilitator checks:
> 1. `payload.cumulativeAmount (900000) == paymentRequirements.extra.cumulativeAmount (800000) + amount (100000)` ✓ correct increment (implies matching base)
> 2. `payload.cumulativeAmount (900000) <= deposit (1000000)` ✓

**Response:**

```json
{
  "isValid": true,
  "payer": "0xClientAddress"
}
```

### `PAYMENT-RESPONSE` header (base64-decoded)

```json
{
  "success": true,
  "network": "eip155:8453",
  "payer": "0xClientAddress",
  "extra": {
    "channelId": "0xdef456...",
    "cumulativeAmount": "900000",
    "deposit": "1000000"
  }
}
```

> Channel resumed. Client now has fresh state and subsequent requests proceed as in Lifecycle 1 (Step 3+).

---

## Step 2b: SIWX-Assisted Resume (Alternative to 2a)

If the server supports the [sign-in-with-x](../../extensions/sign-in-with-x.md) extension, the client can skip the contract read entirely.

### Client Request with SIWX

```http
GET /api/resource?q=example HTTP/1.1
Host: api.example.com
SIGN-IN-WITH-X: eyJ...base64-encoded SIWX token...
```

### Server Response — 402 WITH Channel State

The server recovers the client address from the SIWX token, looks up open channels, and includes channel state in the 402:

```json
{
  "x402Version": 2,
  "error": "PAYMENT-SIGNATURE header is required",
  "accepts": [
    {
      "scheme": "session",
      "network": "eip155:8453",
      "amount": "100000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0xServerPayeeAddress",
      "maxTimeoutSeconds": 3600,
      "extra": {
        "authorizedSettler": "0xFacilitatorSignerAddress",
        "channelId": "0xdef456...",
        "cumulativeAmount": "800000",
        "deposit": "1000000"
      }
    }
  ]
}
```

> Channel state included in 402 — no contract read needed, no stale-settled risk.

### Client Signs Correct Voucher Directly

**`PAYMENT-SIGNATURE` header (base64-decoded):**

```json
{
  "x402Version": 2,
  "accepted": {
    "scheme": "session",
    "network": "eip155:8453",
    "amount": "100000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "payTo": "0xServerPayeeAddress",
    "maxTimeoutSeconds": 3600,
    "extra": {
      "authorizedSettler": "0xFacilitatorSignerAddress",
      "channelId": "0xdef456...",
      "cumulativeAmount": "800000",
      "deposit": "1000000"
    }
  },
  "payload": {
    "type": "voucher",
    "channelId": "0xdef456...",
    "cumulativeAmount": "900000",
    "signature": "0x...EIP-712 Voucher signature (cumulative $0.90)"
  }
}
```

### `PAYMENT-RESPONSE` header (base64-decoded)

```json
{
  "success": true,
  "network": "eip155:8453",
  "payer": "0xClientAddress",
  "extra": {
    "channelId": "0xdef456...",
    "cumulativeAmount": "900000",
    "deposit": "1000000"
  }
}
```

> Channel resumed via SIWX. No extra roundtrips. Subsequent requests proceed as in Lifecycle 1.

---

## Step 3: Subsequent Requests

After either Step 2a or 2b, the client has a fresh `PAYMENT-RESPONSE` with channel state. All subsequent requests follow the same pattern as Lifecycle 1, Step 3 — generic 402 for price, client signs voucher using own state, server enriches `paymentRequirements.extra`.
