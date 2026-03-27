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

Each voucher carries a `cumulativeAmount` that MUST be strictly greater than the previous voucher's amount. The increment between consecutive vouchers MUST equal the per-request price. Only the highest voucher matters for settlement.

- Rationale: Eliminates double-spend risk without requiring nonce tracking per voucher.

### 2. Channel Deposit Model

Clients deposit funds into an onchain escrow (channel) before consuming resources. The deposit is refundable: upon channel close, the unsettled remainder returns to the client. Deposits can be topped up without closing the channel.

- Rationale: Guarantees the server can always settle up to the deposit amount without further client cooperation.

### 3. Batched Onchain Settlement

Settlement is deferred at the server's discretion. The server accumulates vouchers off-chain and settles when economically optimal (e.g. threshold-based, periodic, or on close).

- Rationale: Amortizes gas costs across many requests.

### 4. Facilitator Verification

The server is the source of truth for per-channel state and provides it to the facilitator via `paymentRequirements.extra`. The facilitator cross-checks the client's signed voucher against this server-provided state: signature validity, channel existence, payee/token match, correct cumulative amount increment, and deposit sufficiency. Concrete verification checklists are defined in network-specific specs.

### 5. Server-Authorized Close

Channel closure (`close`) MUST require an authorization signature from the payee or `authorizedSettler` (if designated). This prevents unauthorized parties from closing a channel and refunding the deposit before the server has settled earned funds. Settlement (`settle`) has no caller restriction — a valid client-signed voucher is the only requirement, and funds can only flow to the payee. Concrete signature types are defined in network-specific specs.

### 6. Client State Verification

The client MUST verify server-provided cumulative amounts before signing each voucher. In-session: confirm the returned `cumulativeAmount` incremented by exactly the request price. On recovery (after state loss): verify that the server-provided `lastSignature` recovers to the client's own key for the claimed `cumulativeAmount`. Concrete field-level rules are defined in network-specific specs.

---

## Protocol Flow

### First Request (Channel Open)

The server returns a 402 with no `channelId`, signaling a new channel must be opened.

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

The 402 is generic by default -- it does NOT contain per-client channel state. The client tracks its own state from the last `PAYMENT-RESPONSE` or via a contract read after state loss.

```
Client                      Server                   Facilitator
  |-- GET /resource -------->|                              |
  |<-- 402 + PaymentRequired-|                              |
  |   (scheme:session, amount, authorizedSettler [server key] — no channel state)
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

When `cumulativeAmount + amount > deposit`, the client signs a token authorization for the additional deposit and a new voucher.

```
Client                      Server                   Facilitator           Blockchain
  |-- GET /resource -------->|                              |                     |
  |<-- 402 + PaymentRequired-|                              |                     |
  |   (amount, authorizedSettler [server key] — no channel state)        |                     |
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

The client includes `requestClose: true` in a voucher payload. The server signs a close authorization and forwards it to the facilitator.

```
Client                      Server                   Facilitator           Blockchain
  |-- GET /resource + PAYMENT-SIGNATURE -------------->|                      |
  |   (payload.type = voucher, requestClose = true)    |                      |
  |                          |-- POST /verify -------->| (signature check)    |
  |                          |<-- {isValid} -----------|                      |
  |                          |                         |                      |
  |  [server signs CloseAuthorization with payee key or authorizedSettler]    |
  |                          |-- POST /settle -------->|-- close channel ---->|
  |                          |  (+ closeAuthorization) |                      |
  |                          |<-- {tx} ---------------|                       |
  |<-- 200 + resource -------|                         |                      |
```

### Channel Resume (After State Loss)

When a client returns after losing state, it rediscovers its channel via a network-specific mechanism (e.g. contract read on EVM) and anchors the voucher to the onchain `settled` amount. If the server has unsettled vouchers above `settled`, the facilitator rejects with `session_stale_cumulative_amount`, and the server returns a **corrective 402** with its per-channel state and `lastSignature`. The client verifies the signature and retries. At most one extra roundtrip.

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
  |<-- 402 WITH channelId + cumulativeAmount + deposit + lastSignature           |
  | [verifies lastSignature recovers to own key for cumulativeAmount]           |
  | [signs voucher for cumulativeAmount + amount]           |                     |
  |-- GET /resource + PAYMENT-SIGNATURE ----------------->|                      |
  |                          |-- POST /verify ----------->|                      |
  |                          |<-- isValid: true ----------|                      |
  |<-- 200 + PAYMENT-RESPONSE|                              |                     |
```

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

Within a workflow, if the price is constant AND the client has channel state from a prior `PAYMENT-RESPONSE`, the client MAY skip the 402 entirely:

- The client reuses the previous `accepted` requirements and computes `cumulativeAmount` from `PAYMENT-RESPONSE.extra.cumulativeAmount + amount`.
- The server MUST accept proactive vouchers (no 402 required) if the voucher is valid.
- If the proactive voucher has the wrong amount (e.g., the price changed), the server returns a 402 with the correct price and the client retries.

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
| `session_invalid_close_authorization` | Close authorization signature does not recover to the expected signer (payee or authorizedSettler) |
| `session_channel_expired`           | Channel close grace period has elapsed                             |
| `session_deposit_insufficient`      | Channel deposit is too low to cover another request                |
| `session_stale_cumulative_amount`  | Voucher's base cumulative amount does not match the server's last known value |

---

## Network-Specific Implementation

Network-specific rules and implementation details are defined in the per-network scheme documents:

- EVM chains: See [`scheme_session_evm.md`](./scheme_session_evm.md)
