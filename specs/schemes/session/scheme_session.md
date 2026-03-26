# Scheme: `session`

## Summary

The `session` scheme enables high-frequency, pay-as-you-go payments over unidirectional payment channels. Clients deposit funds into an onchain escrow contract, then sign off-chain **cumulative vouchers** as they consume resources. The server (facilitator) verifies vouchers with pure signature checks and settles periodically in batches via the facilitator.

## Example Use Cases

- An AI agent making repeated tool calls at a fixed price per call
- High-frequency data feeds with per-query pricing
- Content access metered by page or document

## Design Rationale

The `exact` scheme settles every request onchain. This is appropriate for one-off purchases but introduces per-request latency and gas costs that become prohibitive at scale. The `session` scheme amortizes onchain costs across many requests by separating **authorization** (off-chain voucher signatures) from **settlement** (onchain batch transactions).


## Core Properties (MUST)

The `session` scheme MUST enforce the following properties across ALL network implementations:

### 1. Cumulative Monotonic Vouchers

Each voucher carries a `cumulativeAmount` that MUST be strictly greater than the previous voucher's amount. The increment between consecutive vouchers MUST equal the per-request price. Vouchers are not individually redeemable, only the highest voucher matters for settlement.

### 2. Channel Deposit Model

Clients deposit funds into an onchain escrow (channel) before consuming resources. The deposit is refundable: upon channel close, the unsettled remainder returns to the client. Deposits can be topped up without closing the channel.

### 3. Batched Onchain Settlement

Settlement is deferred at the server's discretion. The server accumulates vouchers off-chain and settles when economically optimal (e.g. threshold-based, periodic, or on close).

---

## Protocol Flow

### First Request (Channel Open)

The server returns a 402 with no `channelId`, signaling a new channel must be opened. The client signs a token authorization (for the deposit) and a first voucher, then retries with a `channelOpen` payload.

```
Client                      Server                   Facilitator           Blockchain
  |-- GET /resource -------->|                             |                     |
  |<-- 402 + PaymentRequired-|                             |                     |
  |   (scheme:session, no channelId)                       |                     |
  |                          |                             |                     |
  | [signs token authorization + first voucher]            |                     |
  |-- GET /resource + PAYMENT-SIGNATURE ----------------->|                      |
  |   (payload.type = channelOpen)                         |                     |
  |                          |-- POST /verify ------------>|  (validate deposit  |
  |                          |<-- {isValid} ---------------|   auth + voucher)   |
  |                          |-- POST /settle ------------>|-- open channel ---->|
  |                          |<-- {channelId, tx} ---------|                     |
  |<-- 200 + resource -------|                             |                     |
  |   + PAYMENT-RESPONSE (channelId, cumulativeAmount)     |                     |
```

### Subsequent Requests (Voucher)

The server returns a 402 with `PaymentRequirements` containing the `amount` for this request and `extra.authorizedSettler`. The 402 is **generic by default** -- it does NOT contain per-client channel state (`channelId`, `cumulativeAmount`, `deposit`). The client is responsible for knowing its own channel state, either from the last `PAYMENT-RESPONSE` (within a workflow) or from a contract read (after state loss). See [Channel Resume](#channel-resume-after-state-loss) for the state-loss case.

The client signs a new cumulative voucher incrementing by `amount`. After receiving the client's `PAYMENT-SIGNATURE`, the server reads the `channelId` from the payload, looks up its own per-channel state, and includes it in `paymentRequirements.extra` when forwarding to the facilitator.

> **Optional convenience**: If the server can identify the client (e.g. via the [sign-in-with-x](../../extensions/sign-in-with-x.md) extension), it MAY include channel state (`channelId`, `cumulativeAmount`, `deposit`) in the 402 `extra` as a convenience. This is NOT required for the core flow.

```
Client                      Server                   Facilitator
  |-- GET /resource -------->|                              |
  |<-- 402 + PaymentRequired-|                              |
  |   (scheme:session, amount, authorizedSettler — no channel state)
  |                          |                              |
  | [knows channel state from last PAYMENT-RESPONSE]        |
  | [signs voucher: cumulativeAmount = own cumulativeAmount + amount]
  |-- GET /resource + PAYMENT-SIGNATURE ----------------->|
  |   (payload.type = voucher)                             |
  |                          |                              |
  | [server reads channelId from payload, looks up own state]
  | [includes channelId + cumulativeAmount + deposit in paymentRequirements.extra]
  |                          |-- POST /verify ----------->| (cross-check + sig)
  |                          |<-- {isValid} --------------|
  |<-- 200 + resource -------|                             |
  |   + PAYMENT-RESPONSE (cumulativeAmount updated)        |
```

### Top-Up (Deposit Exhausted)

When the client's own channel state shows `cumulativeAmount + amount > deposit`, it knows a top-up is required. It signs a token authorization for the additional deposit and a new voucher, then retries with a `topUp` payload. The server includes its channel state in `paymentRequirements.extra` when forwarding to the facilitator.

```
Client                      Server                   Facilitator           Blockchain
  |-- GET /resource -------->|                              |                     |
  |<-- 402 + PaymentRequired-|                              |                     |
  |   (amount, authorizedSettler — no channel state)        |                     |
  | [knows cumulativeAmount + amount > deposit from own state — top-up required]  |
  | [signs token authorization + new voucher]               |                     |
  |-- GET /resource + PAYMENT-SIGNATURE ------------------>|                      |
  |   (payload.type = topUp)                               |                      |
  |                            |                            |                     |
  | [server reads channelId, includes own state in paymentRequirements.extra]     |
  |                            |-- POST /verify ---------->| (validate top-up    |
  |                            |<-- {isValid} ------------|   auth + voucher)    |
  |                            |-- POST /settle ---------->|-- top up channel -->|
  |                            |<-- {tx} -----------------|                      |
  |<-- 200 + resource ---------|                           |                     |
  |   + PAYMENT-RESPONSE (updated deposit, cumulativeAmount)|                     |
```

### Close (Client-Initiated via Payload Flag)

The client includes `requestClose: true` in a voucher payload. The server processes the request normally, then instructs the facilitator to close the channel:

```
Client                      Server                   Facilitator           Blockchain
  |-- GET /resource + PAYMENT-SIGNATURE -------------->|                      |
  |   (payload.type = voucher, requestClose = true)    |                      |
  |                          |-- POST /verify -------->| (signature check)    |
  |                          |<-- {isValid} -----------|                      |
  |                          |-- POST /settle -------->|-- close channel ---->|
  |                          |<-- {tx} ---------------|                       |
  |<-- 200 + resource -------|                         |                      |
```

### Channel Resume (After State Loss)

When a client returns after losing state (e.g. browser restart, days later), it has no `PAYMENT-RESPONSE` to reference. It must rediscover its open channel before sending a voucher.

The channel resume flow:

1. Client receives a generic 402 (no channel state)
2. Client discovers its channel via a network-specific mechanism (e.g. contract read on EVM -- see [`scheme_session_evm.md`](./scheme_session_evm.md))
3. Client sends a voucher anchored to the onchain `settled` amount (`payload.cumulativeAmount = settled + amount`)
4. Server reads the `channelId` from the payload, includes its own channel state in `paymentRequirements.extra`
5. Facilitator checks `payload.cumulativeAmount == paymentRequirements.extra.cumulativeAmount + paymentRequirements.amount`
6. **If match**: the server had no unsettled vouchers (or recently settled). Facilitator accepts.
7. **If mismatch**: the server holds unsettled vouchers above the onchain `settled` amount. The implied base (`payload.cumulativeAmount - amount`) does not match the server's truth. Facilitator rejects with `session_stale_cumulative_amount`.
8. On rejection: server returns a **corrective 402** that includes its per-channel state (`channelId`, `cumulativeAmount`, `deposit`) so the client can retry with the correct base. At most one extra roundtrip.

```
Client                      Server                   Facilitator           Blockchain
  |-- GET /resource -------->|                              |                     |
  |<-- 402 (generic) --------|                              |                     |
  |                          |                              |                     |
  | [discovers channel via contract read]                   |                     |
  |----------------------------------------------- getChannel() -------------->|
  |<--------------------------------------------- channel state (settled) -----|
  |                          |                              |                     |
  | [signs voucher for settled + amount]                    |                     |
  |-- GET /resource + PAYMENT-SIGNATURE ----------------->|                      |
  |                          |                              |                     |
  | [server includes own state in paymentRequirements.extra]|                     |
  |                          |-- POST /verify ----------->|                      |
  |                          |   (paymentRequirements.extra.cumulativeAmount = server's truth)
  |                          |                              |                     |
  |               [facilitator: payload.cumulativeAmount == paymentRequirements.extra.cumulativeAmount + amount?]
  |                          |                              |                     |
  |            alt match ----|<-- isValid: true ----------|                      |
  |<-- 200 + PAYMENT-RESPONSE|                              |                     |
  |                          |                              |                     |
  |        alt mismatch -----|<-- isValid: false, session_stale_cumulative_amount |
  |<-- 402 WITH channelId + cumulativeAmount + deposit      |                     |
  | [signs voucher for cumulativeAmount + amount]           |                     |
  |-- GET /resource + PAYMENT-SIGNATURE ----------------->|                      |
  |                          |-- POST /verify ----------->|                      |
  |                          |<-- isValid: true ----------|                      |
  |<-- 200 + PAYMENT-RESPONSE|                              |                     |
```

> **Alternative: SIWX-assisted resume.** If the server supports the [sign-in-with-x](../../extensions/sign-in-with-x.md) extension and the client provides a `SIGN-IN-WITH-X` header, the server identifies the client and includes channel state in the 402 directly. This skips the contract read and avoids the potential stale-settled roundtrip. See the EVM spec for details.

### Channel Closure Recommendation

If no identification extensions are in use and the client does not persist state, it SHOULD close the channel at the end of its workflow (`requestClose: true`). This avoids the resume complexity (contract reads, potential stale-settled roundtrips).

---

## State Management

The server is the sole owner of session state. The facilitator is stateless.

### Server State

The server MUST maintain the following per open channel:

| State Field            | Type      | Description                                                |
| :--------------------- | :-------- | :--------------------------------------------------------- |
| `channelId`            | `bytes32` | Channel identifier                                         |
| `lastCumulativeAmount` | `uint128` | Highest cumulative amount from a verified voucher          |
| `lastSignature`        | `bytes`   | Signature corresponding to `lastCumulativeAmount`          |
| `deposit`              | `uint128` | Current channel deposit (updated on top-up)                |
| `settled`              | `uint128` | Amount already settled onchain                             |

### Server → Facilitator: `paymentRequirements.extra`

When forwarding a payment to the facilitator for verification, the server includes its per-channel state in `paymentRequirements.extra`. This enables the facilitator to cross-check the client's claimed base against the server's truth.

**For `channelOpen` payloads** (first request), `paymentRequirements.extra` contains only the `authorizedSettler`:

```json
{
  "authorizedSettler": "0xFacilitator..."
}
```

**For `voucher` and `topUp` payloads** (subsequent requests), the server includes its per-channel state:

```json
{
  "authorizedSettler": "0xFacilitator...",
  "channelId": "0xabc...",
  "cumulativeAmount": "500000",
  "deposit": "1000000"
}
```

The server populates this data as follows:
- When client identity is known at 402 time (via SIWX): the same data that appears in the 402 `accepts[].extra` is forwarded in `paymentRequirements.extra`.
- When client identity is NOT known (generic 402): the server reads the `channelId` from the client's `PAYMENT-SIGNATURE` payload, looks up its own state, and appends channel info to `paymentRequirements.extra` before forwarding.

### 402 Response to Client

The 402 response to the client includes channel state in `extra` **ONLY** when the server can identify the client (e.g. via SIWX). Otherwise, the 402 is generic -- containing only `authorizedSettler` (and optionally `assetTransferMethod`).

---

## Verification Rules (MUST)

A facilitator verifying a `session`-scheme payment MUST enforce:

1. **Signature validity**: The voucher signature MUST recover to an authorized signer for the channel.
2. **Channel existence**: For `voucher` and `topUp` payloads, the channel MUST exist and not be finalized. For `channelOpen`, the channel MUST NOT already exist.
3. **Payee match**: The channel payee MUST equal `paymentRequirements.payTo`.
4. **Token match**: The channel token MUST equal `paymentRequirements.asset`.
5. **Settler match**: The channel's authorized settler MUST equal the facilitator's own signer address.
6. **Amount increment (base cross-check)**: `payload.cumulativeAmount` MUST equal `paymentRequirements.extra.cumulativeAmount + paymentRequirements.amount`. This single check validates both the correct per-request increment and that the client's implied base (`payload.cumulativeAmount - amount`) matches the server's truth. If the implied base does not match (e.g. anchored to onchain `settled` while the server has unsettled vouchers), the facilitator rejects with `session_stale_cumulative_amount`.
7. **Deposit sufficiency**: `payload.cumulativeAmount` MUST be ≤ `paymentRequirements.extra.deposit` (the server's known deposit). For `topUp` payloads, `payload.cumulativeAmount` MUST be ≤ `paymentRequirements.extra.deposit + topUp.additionalDeposit`.
8. **Channel not expired**: If a close has been requested and the grace period has elapsed, the channel MUST be rejected.

These checks are security-critical. The server is the source of truth via `paymentRequirements.extra`. The facilitator validates; the server provides data. Implementations MAY introduce stricter limits but MUST NOT relax the above constraints.

---

## Settlement Strategy

The resource server controls when and how often onchain settlement occurs:

| Strategy            | Description                                                    | Trade-off                          |
| :------------------ | :------------------------------------------------------------- | :--------------------------------- |
| **Periodic**        | Settle every N minutes                                         | Predictable gas costs              |
| **Threshold**       | Settle when unsettled amount exceeds T                         | Bounds server's risk exposure      |
| **On close**        | Settle only when closing the channel                           | Minimum gas, maximum risk window   |

---

## Skip-402 Optimization (Optional)

Within a workflow, the client uses the last `PAYMENT-RESPONSE` for channel state and the 402 for price. If the price is constant AND the client has channel state from a prior `PAYMENT-RESPONSE`, the client MAY skip the 402 entirely:

- The client reuses the previous `accepted` requirements and computes `cumulativeAmount` from `PAYMENT-RESPONSE.extra.cumulativeAmount + amount`.
- The server MUST accept proactive vouchers (no 402 required) if the voucher is valid.
- If the proactive voucher has the wrong amount (e.g., the price changed), the server returns a 402 with the correct price and the client retries.

This is a narrow optimization for fixed-price workflows. The primary flow always includes a 402 roundtrip to learn the current price.

---

## Error Codes

In addition to the standard x402 error codes, the `session` scheme defines:

| Error Code                          | Description                                                        |
| :---------------------------------- | :----------------------------------------------------------------- |
| `session_channel_not_found`         | No open channel exists for the given `channelId`                   |
| `session_channel_finalized`         | Channel has been closed and finalized                              |
| `session_amount_not_increasing`     | Voucher `cumulativeAmount` is not greater than the last known value |
| `session_amount_exceeds_deposit`    | Voucher `cumulativeAmount` exceeds the channel's deposit           |
| `session_invalid_increment`         | Delta does not equal the required `amount` per request             |
| `session_invalid_voucher_signature` | Voucher signature does not recover to an authorized signer         |
| `session_payee_mismatch`            | Channel payee does not match `payTo` in requirements               |
| `session_token_mismatch`            | Channel token does not match `asset` in requirements               |
| `session_settler_mismatch`          | Channel `authorizedSettler` does not match the facilitator's signer |
| `session_channel_expired`           | Channel close grace period has elapsed                             |
| `session_deposit_insufficient`      | Channel deposit is too low to cover another request                |
| `session_stale_cumulative_amount`  | Voucher's base cumulative amount does not match the server's last known value. The client should retry with the corrective 402 channel state. |

---

## Network-Specific Implementation

Network-specific rules and implementation details are defined in the per-network scheme documents:

- EVM chains: See [`scheme_session_evm.md`](./scheme_session_evm.md)
