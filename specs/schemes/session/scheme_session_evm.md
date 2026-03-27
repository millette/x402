# Scheme: `session` on `EVM`

## Summary

The `session` scheme on EVM uses a modified **TempoStreamChannel** contract for onchain escrow, settlement and channel lifecycle management. Settlement (`settle`) requires a valid client-signed voucher but has no caller restriction; channel closure (`close`) additionally requires a `CloseAuthorization` EIP-712 signature from the server (payee or `authorizedSettler`). The facilitator sponsors gas for all onchain operations.

| AssetTransferMethod | Use Case                                                    | Recommendation              |
| :------------------ | :---------------------------------------------------------- | :-------------------------- |
| **`eip3009`**       | Tokens with `receiveWithAuthorization` (e.g., USDC)         | **Recommended** (simplest, truly gasless)     |
| **`permit2`**       | Tokens without EIP-3009                                     | **Universal Fallback** (Works for any ERC-20) |

Default: `eip3009` if `extra.assetTransferMethod` is omitted.

---

## Session Channel Contract

The session channel contract is a unidirectional payment channel where the client (payer) deposits funds and the server (payee) can settle or close at any time using signed cumulative vouchers.

### Contract Interface Summary

| Function              | Caller                     | Description                                                              |
| :-------------------- | :------------------------- | :----------------------------------------------------------------------- |
| `open`                | Payer                      | Deposit tokens and create a channel (payer pays gas)                     |
| `openWithERC3009`     | Anyone (facilitator)       | Gasless channel open via ERC-3009 signature                              |
| `settle`              | Anyone (valid voucher required) | Claim funds using a signed cumulative voucher                   |
| `topUp`               | Payer                      | Add funds to an existing channel (payer pays gas)                        |
| `topUpWithERC3009`    | Anyone (facilitator)       | Gasless top-up via ERC-3009 signature                                    |
| `requestClose`        | Payer                      | Begin the grace period for unilateral channel closure                    |
| `close`               | Anyone (requires CloseAuthorization) | Close channel with server-signed authorization, settle final voucher, refund remainder |
| `withdraw`            | Payer                      | Withdraw remaining funds after the close grace period                    |

In the x402 session flow, the **facilitator** is the sole gas-paying entity:

- **Client** signs ERC-3009 authorizations (off-chain) → facilitator submits `openWithERC3009` / `topUpWithERC3009`
- **Server** forwards vouchers + `CloseAuthorization` signatures to the facilitator → facilitator submits `settle` (no caller restriction) / `close` (with server's `CloseAuthorization`)

> **Requirement**: This contract MUST be deployed to the same address across all supported EVM chains using `CREATE2`.

See [`scheme_session_evm_contract.md`](./scheme_session_evm_contract.md) for the full contract specification.

### Voucher EIP-712 Type

Vouchers are signed using the EIP-712 typed data standard.

**Domain:**

```
name:    "Tempo Stream Channel"   (deployed contract constant)
version: "1"
```

**Type:**

```
Voucher(bytes32 channelId, uint128 cumulativeAmount)
```

### CloseAuthorization EIP-712 Type

Close authorizations use the same EIP-712 domain as vouchers.

**Type:**

```
CloseAuthorization(bytes32 channelId, uint128 cumulativeAmount)
```

Signed by the **payee** (if `authorizedSettler == address(0)`) or the **authorizedSettler** (server delegate). Required by `close()` to authorize the final settlement amount.

### Channel ID

The `channelId` is computed deterministically:

```
channelId = keccak256(abi.encode(payer, payee, token, salt, authorizedSigner, authorizedSettler, contractAddress, chainId))
```

### Contract Constants

| Constant             | Value      | Description                                              |
| :------------------- | :--------- | :------------------------------------------------------- |
| `CLOSE_GRACE_PERIOD` | 60 minutes | Time between a payer's close request and withdrawal eligibility |

Increased from 15 minutes to give the server sufficient time to settle via time-tracking alone, without monitoring onchain events (see [Server Settlement Timing](#channel-lifecycle-notes)). |

---

## 402 Response (PaymentRequirements)

The 402 is the entry point of every session interaction. The server returns it when the client has not yet paid or needs to pay again.

### Generic 402 (Default)

By default, the 402 contains only the pricing terms and the server's `authorizedSettler` address (close authorization signing key). No per-client channel state is included. The client determines its own channel state from its last `PAYMENT-RESPONSE` (within a workflow) or via a contract read (see [Channel Discovery](#channel-discovery)).

```json
{
  "scheme": "session",
  "network": "eip155:8453",
  "amount": "100000",
  "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  "payTo": "0xServerPayeeAddress",
  "maxTimeoutSeconds": 3600,
  "extra": {
    "authorizedSettler": "0xServerSettlerAddress",
    "name": "USDC",
    "version": "2"
  }
}
```

### Enriched 402 (Client Identified or Corrective)

When the server can identify the client (e.g. via [sign-in-with-x](../../extensions/sign-in-with-x.md)), it includes the client's channel state in `extra`. This lets the client skip channel discovery.

When channel state is included, the server MUST also include `lastSignature` to enable the client to cryptographically verify the server's claimed state (see [Client Verification Rules](#client-verification-rules-must)).

```json
{
  "scheme": "session",
  "network": "eip155:8453",
  "amount": "100000",
  "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  "payTo": "0xServerPayeeAddress",
  "maxTimeoutSeconds": 3600,
  "extra": {
    "authorizedSettler": "0xServerSettlerAddress",
    "name": "USDC",
    "version": "2",
    "channelId": "0xabc123...",
    "cumulativeAmount": "500000",
    "deposit": "1000000",
    "lastSignature": "0x...voucher signature for cumulativeAmount"
  }
}
```

### `extra` Field Reference

| Field                       | Type      | Required | Description                                                                 |
| :-------------------------- | :-------- | :------- | :-------------------------------------------------------------------------- |
| `extra.authorizedSettler`   | `string`  | yes      | Server's close authorization signing address (delegate). If `address(0)`, payee signs close authorizations directly. Used as `authorizedSettler` when opening channels. |
| `extra.assetTransferMethod` | `string`  | optional | `"eip3009"` (default) or `"permit2"` (future). Omit to use default.         |
| `extra.name`                | `string`  | yes      | EIP-712 domain name of the token contract (e.g., `"USDC"`)                  |
| `extra.version`             | `string`  | yes      | EIP-712 domain version of the token contract (e.g., `"2"`)                  |
| `extra.channelId`           | `string`  | enriched | Channel identifier                                                          |
| `extra.cumulativeAmount`    | `string`  | enriched | Server's `lastCumulativeAmount`                                             |
| `extra.deposit`             | `string`  | enriched | Server's known deposit                                                      |
| `extra.lastSignature`       | `string`  | enriched | Client's voucher signature for `cumulativeAmount`                           |

---

## Client: Payment Construction

After receiving a 402, the client constructs a `PaymentPayload` containing its signed commitment. The payload type depends on the channel state:

- **`channelOpen`** — no channel exists yet; client signs a token authorization and first voucher
- **`voucher`** — channel exists with sufficient balance; client signs a new cumulative voucher
- **`topUp`** — channel exists but balance exhausted; client signs a token authorization and voucher

### Asset Transfer Method: EIP-3009

For tokens that support `receiveWithAuthorization` (e.g., USDC), the session channel contract provides gasless channel operations.

#### Channel Open: `openWithERC3009()`

The client signs an ERC-3009 `receiveWithAuthorization` for the deposit amount. The facilitator submits the `openWithERC3009()` transaction, paying gas:

```solidity
function openWithERC3009(
    address payer,              // client's address
    address payee,              // PaymentRequirements.payTo
    address token,              // PaymentRequirements.asset
    uint128 deposit,            // ≥ PaymentRequirements.amount
    bytes32 salt,               // deterministic salt (see Channel Discovery)
    address authorizedSigner,   // 0x0 to use payer's own address, or a delegate
    address authorizedSettler,  // PaymentRequirements.extra.authorizedSettler (server delegate)
    uint256 validAfter,         // ERC-3009 authorization start time
    uint256 validBefore,        // ERC-3009 authorization expiry time
    bytes32 nonce,              // ERC-3009 authorization nonce
    bytes calldata signature    // ERC-3009 ReceiveWithAuthorization signature from payer
) external returns (bytes32 channelId)
```

**Authorized Signer**: Allows the client to delegate voucher signing to a different key (e.g., a session key or delegate). If set to `address(0)`, vouchers must be signed by the payer address.

**Authorized Settler**: MUST be set to the server's close authorization delegate address (from `PaymentRequirements.extra.authorizedSettler`). This designates which key must sign `CloseAuthorization` messages to authorize channel closure. If `address(0)`, the payee signs close authorizations directly.

#### Top-Up: `topUpWithERC3009()`

Same pattern as channel open. The ERC-3009 token validates that the signature was produced by `channel.payer`, ensuring only the original payer can authorize additional deposits:

```solidity
function topUpWithERC3009(
    bytes32 channelId,          // channel to top up
    uint256 additionalDeposit,  // amount to add (must match ERC-3009 value)
    uint256 validAfter,         // ERC-3009 authorization start time
    uint256 validBefore,        // ERC-3009 authorization expiry time
    bytes32 nonce,              // ERC-3009 authorization nonce
    bytes calldata signature    // ERC-3009 ReceiveWithAuthorization signature from payer
) external
```

If a close request was pending, the top-up cancels it and the session continues uninterrupted.

#### ERC-3009 Authorization Fields

The `channelOpen` and `topUp` payloads include an `erc3009Authorization` object:

| Field         | Type     | Description                                              |
| :------------ | :------- | :------------------------------------------------------- |
| `validAfter`  | `number` | Authorization start time (unix timestamp)                |
| `validBefore` | `number` | Authorization expiry time (unix timestamp)               |
| `nonce`       | `string` | Random nonce for replay protection                       |
| `signature`   | `string` | ERC-3009 `ReceiveWithAuthorization` signature from payer |

### PaymentPayload Examples

**Type: `channelOpen`**

```json
{
  "x402Version": 2,
  "accepted": {
    "scheme": "session",
    "network": "eip155:8453",
    "amount": "1000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "payTo": "0xServerPayeeAddress",
    "maxTimeoutSeconds": 3600,
    "extra": {
      "authorizedSettler": "0xServerSettlerAddress",
      "name": "USDC",
      "version": "2"
    }
  },
  "payload": {
    "type": "channelOpen",
    "channelOpen": {
      "payer": "0xClientAddress",
      "payee": "0xServerPayeeAddress",
      "token": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "deposit": "100000",
      "salt": "0x...keccak256(abi.encode('x402-session', uint256(0)))",
      "authorizedSigner": "0x0000000000000000000000000000000000000000",
      "authorizedSettler": "0xServerSettlerAddress",
      "erc3009Authorization": {
        "validAfter": 0,
        "validBefore": 1679616000,
        "nonce": "0x...random nonce",
        "signature": "0x...ERC-3009 ReceiveWithAuthorization signature from payer"
      }
    },
    "voucher": {
      "channelId": "0xabc123...computed channelId",
      "cumulativeAmount": "1000",
      "signature": "0x...EIP-712 voucher signature"
    }
  }
}
```

**Type: `voucher`**

```json
{
  "x402Version": 2,
  "accepted": {
    "scheme": "session",
    "network": "eip155:8453",
    "amount": "1000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "payTo": "0xServerPayeeAddress",
    "maxTimeoutSeconds": 3600,
    "extra": {
      "authorizedSettler": "0xServerSettlerAddress",
      "name": "USDC",
      "version": "2"
    }
  },
  "payload": {
    "type": "voucher",
    "channelId": "0xabc123...channelId",
    "cumulativeAmount": "5000",
    "signature": "0x...65-byte EIP-712 signature",
    "requestClose": false
  }
}
```

**Type: `topUp`**

```json
{
  "x402Version": 2,
  "accepted": {
    "scheme": "session",
    "network": "eip155:8453",
    "amount": "1000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "payTo": "0xServerPayeeAddress",
    "maxTimeoutSeconds": 3600,
    "extra": {
      "authorizedSettler": "0xServerSettlerAddress",
      "name": "USDC",
      "version": "2"
    }
  },
  "payload": {
    "type": "topUp",
    "topUp": {
      "channelId": "0xabc123...channelId",
      "additionalDeposit": "50000",
      "erc3009Authorization": {
        "validAfter": 0,
        "validBefore": 1679616000,
        "nonce": "0x...random nonce",
        "signature": "0x...ERC-3009 ReceiveWithAuthorization signature from payer"
      }
    },
    "voucher": {
      "channelId": "0xabc123...channelId",
      "cumulativeAmount": "101000",
      "signature": "0x...EIP-712 voucher signature"
    }
  }
}
```

---

## Server: State & Facilitator Forwarding

The server is the sole owner of session state. The facilitator is stateless.

### Per-Channel State

The server MUST maintain the following per open channel:

| State Field            | Type      | Description                                                |
| :--------------------- | :-------- | :--------------------------------------------------------- |
| `channelId`            | `bytes32` | Channel identifier                                         |
| `payer`                | `address` | Client address                                             |
| `lastCumulativeAmount` | `uint128` | Highest cumulative amount from a verified voucher          |
| `lastSignature`        | `bytes`   | Signature corresponding to `lastCumulativeAmount`          |
| `deposit`              | `uint128` | Current channel deposit (updated on top-up)                |
| `settled`              | `uint128` | Amount already settled onchain                             |
| `lastRequestTimestamp` | `uint64`  | Timestamp of the last paid request on this channel         |

### Enriching `paymentRequirements.extra` for the Facilitator

When forwarding a payment to the facilitator, the server includes its per-channel state in `paymentRequirements.extra`.

**For `channelOpen` payloads** (first request):

```json
{
  "authorizedSettler": "0xServerSettler..."
}
```

**For `voucher` and `topUp` payloads** (subsequent requests):

```json
{
  "authorizedSettler": "0xServerSettler...",
  "channelId": "0xabc...",
  "cumulativeAmount": "500000",
  "deposit": "1000000",
  "lastSignature": "0x...voucher signature for cumulativeAmount"
}
```

---

## Facilitator Interface

The session scheme uses the standard x402 facilitator interface (`/verify`, `/settle`, `/supported`). The facilitator is stateless and derives session context from `payload` (client's signed commitment), `paymentRequirements.extra` (server's truth), and onchain channel state.

### POST /verify

Verifies a payment payload without onchain interaction. The server enriches `paymentRequirements.extra` with its per-channel state for `voucher` and `topUp` payloads before forwarding.

Verification logic is defined in [Verification Rules](#verification-rules-must).

**Response:**

```json
{
  "isValid": true,
  "payer": "0xPayerAddress",
  "extra": {
    "closeRequestedAt": 0
  }
}
```

`extra.closeRequestedAt` is read from `channel.closeRequestedAt` (already available from the `getChannel()` call in verification rule #2). A non-zero value indicates the client has initiated a unilateral close and the server SHOULD settle immediately upon receiving this signal.

### POST /settle

Performs onchain operations. The facilitator infers the action from the payload:

| `settleAction` | Payload Type   | Onchain Operation                                             | When Used                                       |
| :------------- | :------------- | :------------------------------------------------------------ | :---------------------------------------------- |
| `"open"`       | `channelOpen`  | `openWithERC3009()`                                           | First request — server opens the channel        |
| `"topUp"`      | `topUp`        | `topUpWithERC3009()`                                          | Client sent a top-up payload                    |
| `"settle"`     | `voucher`      | `settle(channelId, amount, sig)`                              | Server batches settlement at its discretion     |
| `"close"`      | `voucher`      | `close(channelId, amount, voucherSig, closeAuth)`             | Client requested close or server-initiated close |

**Request (voucher settle example):**

```json
{
  "x402Version": 2,
  "paymentPayload": {
    "x402Version": 2,
    "accepted": {
      "scheme": "session",
      "network": "eip155:8453",
      "amount": "1000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0xServerPayeeAddress",
      "maxTimeoutSeconds": 3600,
      "extra": {
        "authorizedSettler": "0xServerSettlerAddress",
        "name": "USDC",
        "version": "2"
      }
    },
    "payload": {
      "type": "voucher",
      "channelId": "0xabc123...",
      "cumulativeAmount": "5000",
      "signature": "0x...EIP-712 voucher signature",
      "requestClose": false
    }
  },
  "paymentRequirements": {
    "scheme": "session",
    "network": "eip155:8453",
    "amount": "1000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "payTo": "0xServerPayeeAddress",
    "maxTimeoutSeconds": 3600,
    "extra": {
      "authorizedSettler": "0xServerSettlerAddress",
      "name": "USDC",
      "version": "2",
      "channelId": "0xabc123...",
      "cumulativeAmount": "4000",
      "deposit": "100000",
      "lastSignature": "0x...EIP-712 Voucher signature for cumulativeAmount 4000"
    }
  }
}
```

**Settlement Logic:**

- **`channelOpen`**: Submit `openWithERC3009()` using `payload.channelOpen` parameters. Returns the `channelId` and transaction hash.
- **`topUp`**: Submit `topUpWithERC3009()` using `payload.topUp` parameters. Returns the transaction hash.
- **`voucher`**: Submit `settle(channelId, cumulativeAmount, signature)` using the highest voucher. The contract transfers the delta between the onchain `settled` amount and `cumulativeAmount` to the payee. `settle` has no caller restriction — a valid voucher is the only requirement.
- **`voucher` + `requestClose: true`**: Submit `close(channelId, cumulativeAmount, voucherSignature, closeAuthorization)` using the highest voucher and the `CloseAuthorization` signature provided by the server. The contract settles the final amount and refunds the remainder to the payer.

**`closeAuthorization` Field**: When the server requests a close (either because `requestClose: true` or server-initiated), it MUST include `paymentRequirements.extra.closeAuthorization` — an EIP-712 `CloseAuthorization(channelId, cumulativeAmount)` signature from the payee or `authorizedSettler`. The facilitator passes this to the `close()` contract call.

**Close Request Detection**: During `/settle`, if the facilitator reads `channel.closeRequestedAt != 0`, it MUST proceed with onchain settlement to protect the server's funds before the grace period expires.

**Response:**

```json
{
  "success": true,
  "transaction": "0x...transactionHash",
  "network": "eip155:8453",
  "payer": "0xPayerAddress",
  "extra": {
    "channelId": "0xabc123...",
    "cumulativeAmount": "5000",
    "deposit": "100000",
    "closeRequestedAt": 0
  }
}
```

The `extra` field contains updated session state. `closeRequestedAt` is non-zero if the client has initiated a unilateral close. The server uses this to populate the `PAYMENT-RESPONSE` header and to update its own per-channel state. For `channelOpen` payloads, `extra.channelId` contains the newly created channel ID.

### GET /supported

```json
{
  "kinds": [
    { "x402Version": 2, "scheme": "session", "network": "eip155:8453" },
    { "x402Version": 2, "scheme": "exact", "network": "eip155:8453" }
  ],
  "extensions": [],
  "signers": {
    "eip155:*": ["0xFacilitatorSignerAddress"]
  }
}
```

### Verification Rules (MUST)

A facilitator verifying a `session`-scheme payment on EVM MUST enforce:

1. **Signature validity**: Compute the EIP-712 digest for `Voucher(channelId, cumulativeAmount)` using the `TempoStreamChannel` domain separator. Recover the signer via `ecrecover`. The recovered signer MUST match `channel.payer` or `channel.authorizedSigner` (if non-zero). For `channelOpen` payloads, verify against the `channelOpen` parameters.
2. **Channel existence**: For `voucher` and `topUp` payloads, read `TempoStreamChannel.getChannel(channelId)` -- the channel MUST exist (`payer != address(0)`) and not be finalized. For `channelOpen`, the channel MUST NOT already exist.
3. **Payee match**: `channel.payee` MUST equal `paymentRequirements.payTo`.
4. **Token match**: `channel.token` MUST equal `paymentRequirements.asset`. The contract MUST be on the correct chain.
5. **Balance check** (`channelOpen` and `topUp` only): Verify the client has sufficient token balance (`≥ deposit` for opens, `≥ additionalDeposit` for top-ups). For `voucher` payloads this is not needed as funds are already in escrow.
6. **Amount increment (base cross-check)**: `payload.cumulativeAmount` MUST equal `paymentRequirements.extra.cumulativeAmount + paymentRequirements.amount`. If the implied base does not match, reject with `session_stale_cumulative_amount`.
7. **Deposit sufficiency**: `payload.cumulativeAmount` MUST be ≤ `paymentRequirements.extra.deposit`. For `topUp` payloads, `payload.cumulativeAmount` MUST be ≤ `paymentRequirements.extra.deposit + topUp.additionalDeposit`.
8. **Close request detection**: If `channel.closeRequestedAt != 0` (already available from the `getChannel()` read in rule 2), the facilitator MUST include `closeRequestedAt` in the `/verify` and `/settle` responses. During `/settle`, if `closeRequestedAt != 0`, the facilitator MUST proceed with onchain settlement to protect the server's funds before the grace period expires.

These checks are security-critical. The server provides truth via `paymentRequirements.extra`; the facilitator validates. Implementations MAY introduce stricter limits but MUST NOT relax the above constraints.

---

## Channel Discovery

When a client has lost state and receives a generic 402 (no channel state), it must rediscover open channels via the contract. This requires **deterministic salt** so the client can compute channel IDs without server cooperation.

### Deterministic Salt Convention (MUST)

The `channelId` is computed from `(payer, payee, token, salt, authorizedSigner, authorizedSettler, contractAddress, chainId)`. For a stateless client to rediscover channels via contract reads, the salt MUST be deterministic:

```
salt = keccak256(abi.encode("x402-session", uint256(sequenceIndex)))
```

Where `sequenceIndex` starts at 0 and increments for concurrent channels between the same `(payer, payee, token, authorizedSigner, authorizedSettler)` tuple.

See [`scheme_session_evm_contract.md`](./scheme_session_evm_contract.md) for the salt convention in the contract context.

### Discovery Algorithm

1. Client reads `payTo`, `asset`, `authorizedSettler` from the 402 response
2. Client computes `channelId` values for indices 0, 1, 2, ... using the deterministic salt formula
3. Client calls `getChannelsBatch([id0, id1, id2, ...])` -- single RPC call
4. For each result, classify:
   - `payer != address(0)` and `finalized == false` → **open channel** (usable)
   - `payer == address(0)` and `finalized == true` → **closed channel** (skip, continue scanning)
   - `payer == address(0)` and `finalized == false` → **never opened** (stop iterating)
5. From open channels, select one with sufficient remaining balance (`deposit - settled >= amount`)
6. Use the lowest never-opened index when opening a new channel

### Resume After Discovery

After discovering an open channel, the client anchors its voucher to the onchain `settled` amount (`cumulativeAmount = settled + amount`). See [scheme_session.md -- Channel Resume](./scheme_session.md#channel-resume-after-state-loss) for the full resume flow including corrective-402 handling.

If the server supports the [sign-in-with-x](../../extensions/sign-in-with-x.md) extension and the client provides a `SIGN-IN-WITH-X` header, the server includes channel state in the 402 directly, skipping the contract read and potential stale-settled roundtrip. The client MUST verify `lastSignature` before using server-provided state (see [Client Verification Rules](#client-verification-rules-must)).

---

## Client Verification Rules (MUST)

The facilitator's verification rules protect the server. The following rules protect the **client** from a misbehaving server. They apply to the `PAYMENT-RESPONSE` header the server returns after each successful request (populated from the facilitator's settle response `extra` field).

### In-Session Verification

Before using `PAYMENT-RESPONSE` values as the base for the next voucher, the client MUST check:

1. **Cumulative amount increment**: `PAYMENT-RESPONSE.extra.cumulativeAmount` MUST equal `previousCumulativeAmount + requestAmount`. If the server inflates `cumulativeAmount`, the client would sign vouchers authorizing more than the actual cost.
2. **Deposit consistency**: For non-topup responses, `PAYMENT-RESPONSE.extra.deposit` MUST equal the client's last known deposit. For topup responses, it MUST equal `previousDeposit + additionalDeposit`.
3. **Channel ID consistency**: `PAYMENT-RESPONSE.extra.channelId` MUST match the channel the client is operating on.

If any check fails, the client MUST NOT sign further vouchers and SHOULD initiate channel closure.

### Recovery Verification

When the client has lost state and receives a corrective 402 (or SIWX-enriched 402) containing channel state, the server MUST include `lastSignature` -- the client's own voucher signature for the claimed `cumulativeAmount`.

1. **Voucher signature**: Compute the EIP-712 `Voucher(channelId, cumulativeAmount)` digest, recover the signer from `lastSignature`, confirm it matches the client's own address (or `authorizedSigner`).
2. **Onchain consistency**: `extra.cumulativeAmount` MUST be ≥ the onchain `settled` amount. `extra.deposit` MUST match the onchain deposit.

If the signature does not verify, the client MUST NOT sign based on the server's claimed state and SHOULD fall back to unilateral channel closure.

---

## Channel Lifecycle Notes

**Reusing Existing Channels**: If a client has an open, non-finalized channel to the same `(payee, token)` with sufficient remaining balance (`deposit - settled ≥ amount`), it SHOULD reuse it rather than opening a new one. Servers MUST support receiving vouchers for any open channel where they are the payee.

**Cooperative Close**: The client includes `requestClose: true` in a voucher payload. The server processes the request normally, then signs a `CloseAuthorization(channelId, lastCumulativeAmount)` with its payee key or `authorizedSettler` delegate and includes it in the `/settle` request to the facilitator. The facilitator sees `requestClose: true` in the payload and calls `TempoStreamChannel.close(channelId, cumulativeAmount, voucherSignature, closeAuthorization)`. The contract verifies the `CloseAuthorization` signature, settles the final amount to the payee, and refunds the remainder to the payer. If no identification extensions are in use and the client does not persist state, it SHOULD close the channel at the end of its workflow to avoid resume complexity.

**Unilateral Close (Escape Hatch)**: If the server becomes unresponsive and the client cannot initiate a cooperative close, the client calls `TempoStreamChannel.requestClose(channelId)` directly onchain, paying gas themselves. This starts the `CLOSE_GRACE_PERIOD` (1 hour). The server can still settle outstanding vouchers via the facilitator during this period. After the grace period, the client calls `withdraw()` to reclaim all unsettled funds. This mechanism is intentionally a direct blockchain transaction, not gasless — the client does not have the facilitator's address in the unilateral path, and adding EIP-712 signature flows for an escape hatch would introduce unnecessary friction.

**Server Settlement Timing**: The server SHOULD settle (or close) outstanding vouchers for a channel within `CLOSE_GRACE_PERIOD` of the last client request on that channel. This ensures the server captures earned funds even if the client initiates a unilateral close after the session goes idle. The facilitator provides an additional safety net: since it reads `channel.closeRequestedAt` from onchain state during `/verify` and `/settle` (see [Verification Rules](#verification-rules-must) rule 8), the server is alerted if a close request is already in progress and can settle immediately. Together, time-based settlement and facilitator detection protect the server without requiring onchain event monitoring. Servers managing many concurrent channels MAY additionally monitor onchain `CloseRequested` events (e.g. via RPC polling or an indexer) for more proactive awareness.

**Channel Rotation Requirement**: Servers MUST close all active channels before changing either `payTo` or `authorizedSettler`. Both values are part of the `channelId` computation. Changing them while channels are open would leave clients with channels pointing to stale server keys. After closing all channels, the server updates its 402 response with the new values and clients open fresh channels on the next request.

---

## Version History

| Version | Date       | Changes                                                     | Author    |
| :------ | :--------- | :---------------------------------------------------------- | :-------- |
| v0.1    | 2025-03-21 | Initial draft                                               | @phdargen |
