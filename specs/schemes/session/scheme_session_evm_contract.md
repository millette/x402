# Session Scheme: EVM Contract Specification

Adapted from the audited [TempoStreamChannel](https://github.com/tempoxyz/tempo/blob/main/tips/ref-impls/src/TempoStreamChannel.sol) for standard EVM ERC-20 chains. Adds gasless `openWithERC3009` and `topUpWithERC3009` via ERC-3009 `receiveWithAuthorization`, `authorizedSettler` for server-cosigned close authorization and caller-unrestricted `settle`.

## Changes from TempoStreamChannel

1. **ERC-20 token interface**: `ITIP20` / `TempoUtilities` replaced with `IERC20` / `IERC3009`
2. **`openWithERC3009`**: Gasless channel open: anyone (facilitator) submits tx, payer authorizes via ERC-3009 signature
3. **`topUpWithERC3009`**: Gasless top-up 
4. **`authorizedSettler`**: Delegates close authorization signing from the payee to a designated address (e.g. a server delegate). Mirrors the existing `authorizedSigner` pattern. `authorizedSigner` delegates voucher signing from the payer side, `authorizedSettler` delegates close authorization signing from the payee side.
5. **No caller restriction on `settle`**: `settle` has no `msg.sender` check. Any caller can submit, but a valid client-signed voucher is still required. Funds can only flow to the payee.
6. **Server-cosigned `close`**: `close` requires a `CloseAuthorization` EIP-712 signature from the payee or `authorizedSettler` (if non-zero), preventing unauthorized channel closure.

Existing `open` and `topUp` remain unchanged in behavior. Gasless and self-submitted operations produce identical channel state and are fully interoperable on the same channel.

### Why `receiveWithAuthorization`

Both new functions use `receiveWithAuthorization` (not `transferWithAuthorization`). With `receiveWithAuthorization`, only the `to` address (`address(this)` — the channel contract) can execute the authorization. This prevents front-running: a griefier cannot consume the ERC-3009 nonce by calling `transferWithAuthorization` before the channel's open/topUp executes, which would strand tokens in the contract without creating a channel.

## Contract

```diff
 // SPDX-License-Identifier: MIT
 pragma solidity ^0.8.20;

-import { TempoUtilities } from "./TempoUtilities.sol";
-import { ITIP20 } from "./interfaces/ITIP20.sol";
+import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
+import { IERC3009 } from "./interfaces/IERC3009.sol";
 import { ITempoStreamChannel } from "./interfaces/ITempoStreamChannel.sol";
 import { ECDSA } from "solady/utils/ECDSA.sol";
 import { EIP712 } from "solady/utils/EIP712.sol";

 /**
  * @title TempoStreamChannel
  * @notice Unidirectional payment channel escrow for streaming payments.
- * @dev Users deposit TIP-20 tokens, sign cumulative vouchers, and servers
+ * @dev Users deposit ERC-20 tokens, sign cumulative vouchers, and servers
  *      can settle or close at any time. Channels have no expiry - they are
  *      closed either cooperatively by the server or after a grace period
  *      following a user's close request.
  */
 contract TempoStreamChannel is ITempoStreamChannel, EIP712 {

     // --- Constants ---

     bytes32 public constant VOUCHER_TYPEHASH =
         keccak256("Voucher(bytes32 channelId,uint128 cumulativeAmount)");

+    bytes32 public constant CLOSE_AUTHORIZATION_TYPEHASH =
+        keccak256("CloseAuthorization(bytes32 channelId,uint128 cumulativeAmount)");

-    uint64 public constant CLOSE_GRACE_PERIOD = 15 minutes;
+    uint64 public constant CLOSE_GRACE_PERIOD = 60 minutes;

     // --- State ---

     mapping(bytes32 => Channel) public channels;

     // --- EIP-712 Domain ---

     function _domainNameAndVersion()
         internal
         pure
         override
         returns (string memory name, string memory version)
     {
         name = "Tempo Stream Channel";
         version = "1";
     }

     // --- External Functions ---

     /**
      * @notice Open a new payment channel with escrowed funds.
      * @param payee Address authorized to withdraw (server)
-     * @param token TIP-20 token address
+     * @param token ERC-20 token address
      * @param deposit Amount to deposit
      * @param salt Random salt for channel ID generation
      * @param authorizedSigner Address authorized to sign vouchers (0 = use msg.sender)
+     * @param authorizedSettler Address authorized to call settle/close (0 = payee only)
      * @return channelId The unique channel identifier
      */
     function open(
         address payee,
         address token,
         uint128 deposit,
         bytes32 salt,
-        address authorizedSigner
+        address authorizedSigner,
+        address authorizedSettler
     )
         external
         override
         returns (bytes32 channelId)
     {
         if (payee == address(0)) {
             revert InvalidPayee();
         }
-        if (!TempoUtilities.isTIP20(token)) {
-            revert InvalidToken();
-        }
         if (deposit == 0) {
             revert ZeroDeposit();
         }

-        channelId = computeChannelId(msg.sender, payee, token, salt, authorizedSigner);
+        channelId = computeChannelId(msg.sender, payee, token, salt, authorizedSigner, authorizedSettler);

         if (channels[channelId].payer != address(0) || channels[channelId].finalized) {
             revert ChannelAlreadyExists();
         }

         channels[channelId] = Channel({
             payer: msg.sender,
             payee: payee,
             token: token,
             authorizedSigner: authorizedSigner,
+            authorizedSettler: authorizedSettler,
             deposit: deposit,
             settled: 0,
             closeRequestedAt: 0,
             finalized: false
         });

-        bool success = ITIP20(token).transferFrom(msg.sender, address(this), deposit);
+        bool success = IERC20(token).transferFrom(msg.sender, address(this), deposit);
         if (!success) {
             revert TransferFailed();
         }

         emit ChannelOpened(channelId, msg.sender, payee, token, authorizedSigner, salt, deposit);
     }

+    /**
+     * @notice Open a new payment channel using ERC-3009 receiveWithAuthorization (gasless for payer).
+     * @param payer Address depositing funds (must match ERC-3009 `from` signer)
+     * @param payee Address authorized to withdraw (server)
+     * @param token ERC-3009 compatible token address (e.g. USDC)
+     * @param deposit Amount to deposit (must match ERC-3009 `value`)
+     * @param salt Random salt for channel ID generation
+     * @param authorizedSigner Address authorized to sign vouchers (0 = use payer)
+     * @param authorizedSettler Address authorized to call settle/close (0 = payee only)
+     * @param validAfter ERC-3009 authorization start time
+     * @param validBefore ERC-3009 authorization expiry time
+     * @param nonce ERC-3009 authorization nonce
+     * @param signature ERC-3009 ReceiveWithAuthorization signature from payer
+     * @return channelId The unique channel identifier
+     */
+    function openWithERC3009(
+        address payer,
+        address payee,
+        address token,
+        uint128 deposit,
+        bytes32 salt,
+        address authorizedSigner,
+        address authorizedSettler,
+        uint256 validAfter,
+        uint256 validBefore,
+        bytes32 nonce,
+        bytes calldata signature
+    )
+        external
+        returns (bytes32 channelId)
+    {
+        if (payer == address(0)) {
+            revert InvalidPayer();
+        }
+        if (payee == address(0)) {
+            revert InvalidPayee();
+        }
+        if (deposit == 0) {
+            revert ZeroDeposit();
+        }
+
+        channelId = computeChannelId(payer, payee, token, salt, authorizedSigner, authorizedSettler);
+
+        if (channels[channelId].payer != address(0) || channels[channelId].finalized) {
+            revert ChannelAlreadyExists();
+        }
+
+        channels[channelId] = Channel({
+            payer: payer,
+            payee: payee,
+            token: token,
+            authorizedSigner: authorizedSigner,
+            authorizedSettler: authorizedSettler,
+            deposit: deposit,
+            settled: 0,
+            closeRequestedAt: 0,
+            finalized: false
+        });
+
+        IERC3009(token).receiveWithAuthorization(
+            payer,
+            address(this),
+            deposit,
+            validAfter,
+            validBefore,
+            nonce,
+            signature
+        );
+
+        emit ChannelOpened(channelId, payer, payee, token, authorizedSigner, salt, deposit);
+    }
+
     /**
      * @notice Settle funds using a signed voucher.
      * @param channelId The channel to settle
      * @param cumulativeAmount Total amount authorized by the voucher
      * @param signature EIP-712 signature from the payer/authorizedSigner
      */
     function settle(
         bytes32 channelId,
         uint128 cumulativeAmount,
         bytes calldata signature
     )
         external
         override
     {
         Channel storage channel = channels[channelId];

         if (channel.finalized) {
             revert ChannelFinalized();
         }
         if (channel.payer == address(0)) {
             revert ChannelNotFound();
         }
-        if (msg.sender != channel.payee) {
-            revert NotPayee();
-        }
         if (cumulativeAmount > channel.deposit) {
             revert AmountExceedsDeposit();
         }
         if (cumulativeAmount <= channel.settled) {
             revert AmountNotIncreasing();
         }

         bytes32 structHash = keccak256(abi.encode(VOUCHER_TYPEHASH, channelId, cumulativeAmount));
         bytes32 digest = _hashTypedData(structHash);
         address signer = ECDSA.recoverCalldata(digest, signature);

         address expectedSigner =
             channel.authorizedSigner != address(0) ? channel.authorizedSigner : channel.payer;

         if (signer != expectedSigner) {
             revert InvalidSignature();
         }

         uint128 delta = cumulativeAmount - channel.settled;
         channel.settled = cumulativeAmount;

-        bool success = ITIP20(channel.token).transfer(channel.payee, delta);
+        bool success = IERC20(channel.token).transfer(channel.payee, delta);
         if (!success) {
             revert TransferFailed();
         }

         emit Settled(
             channelId, channel.payer, channel.payee, cumulativeAmount, delta, channel.settled
         );
     }

     /**
      * @notice Add more funds to a channel.
      * @param channelId The channel to top up
      * @param additionalDeposit Amount to add
      */
     function topUp(bytes32 channelId, uint256 additionalDeposit) external override {
         Channel storage channel = channels[channelId];

         if (channel.finalized) {
             revert ChannelFinalized();
         }
         if (channel.payer == address(0)) {
             revert ChannelNotFound();
         }
         if (msg.sender != channel.payer) {
             revert NotPayer();
         }

         if (additionalDeposit == 0) {
             revert ZeroDeposit();
         }

         if (additionalDeposit > type(uint128).max - channel.deposit) {
             revert DepositOverflow();
         }
         channel.deposit += uint128(additionalDeposit);

         bool success =
-            ITIP20(channel.token).transferFrom(msg.sender, address(this), additionalDeposit);
+            IERC20(channel.token).transferFrom(msg.sender, address(this), additionalDeposit);
         if (!success) {
             revert TransferFailed();
         }

         if (channel.closeRequestedAt != 0) {
             channel.closeRequestedAt = 0;
             emit CloseRequestCancelled(channelId, channel.payer, channel.payee);
         }

         emit TopUp(channelId, channel.payer, channel.payee, additionalDeposit, channel.deposit);
     }

+    /**
+     * @notice Add more funds to a channel using ERC-3009 receiveWithAuthorization (gasless for payer).
+     * @param channelId The channel to top up
+     * @param additionalDeposit Amount to add (must match ERC-3009 `value`)
+     * @param validAfter ERC-3009 authorization start time
+     * @param validBefore ERC-3009 authorization expiry time
+     * @param nonce ERC-3009 authorization nonce
+     * @param signature ERC-3009 ReceiveWithAuthorization signature from payer
+     */
+    function topUpWithERC3009(
+        bytes32 channelId,
+        uint256 additionalDeposit,
+        uint256 validAfter,
+        uint256 validBefore,
+        bytes32 nonce,
+        bytes calldata signature
+    )
+        external
+    {
+        Channel storage channel = channels[channelId];
+
+        if (channel.finalized) {
+            revert ChannelFinalized();
+        }
+        if (channel.payer == address(0)) {
+            revert ChannelNotFound();
+        }
+        if (additionalDeposit == 0) {
+            revert ZeroDeposit();
+        }
+        if (additionalDeposit > type(uint128).max - channel.deposit) {
+            revert DepositOverflow();
+        }
+
+        channel.deposit += uint128(additionalDeposit);
+
+        IERC3009(channel.token).receiveWithAuthorization(
+            channel.payer,
+            address(this),
+            additionalDeposit,
+            validAfter,
+            validBefore,
+            nonce,
+            signature
+        );
+
+        if (channel.closeRequestedAt != 0) {
+            channel.closeRequestedAt = 0;
+            emit CloseRequestCancelled(channelId, channel.payer, channel.payee);
+        }
+
+        emit TopUp(channelId, channel.payer, channel.payee, additionalDeposit, channel.deposit);
+    }
+
     /**
      * @notice Request early channel closure.
      * @dev Starts a grace period after which the payer can withdraw.
      * @param channelId The channel to close
      */
     function requestClose(bytes32 channelId) external override {
         Channel storage channel = channels[channelId];

         if (channel.finalized) {
             revert ChannelFinalized();
         }
         if (channel.payer == address(0)) {
             revert ChannelNotFound();
         }
         if (msg.sender != channel.payer) {
             revert NotPayer();
         }

        // Only set if not already requested
        if (channel.closeRequestedAt == 0) {
            channel.closeRequestedAt = uint64(block.timestamp);
            emit CloseRequested(
                channelId, channel.payer, channel.payee, block.timestamp + CLOSE_GRACE_PERIOD
            );
        }
    }

     /**
-     * @notice Close a channel immediately (server only).
+     * @notice Close a channel immediately (requires CloseAuthorization).
      * @dev Settles any outstanding voucher and refunds remainder to payer.
      * @param channelId The channel to close
      * @param cumulativeAmount Final cumulative amount (0 if no payments)
      * @param signature EIP-712 signature (empty if cumulativeAmount == 0 or same as settled)
+     * @param closeAuthorization EIP-712 CloseAuthorization signature from payee or authorizedSettler
      */
     function close(
         bytes32 channelId,
         uint128 cumulativeAmount,
         bytes calldata signature
+        bytes calldata closeAuthorization
     )
         external
         override
     {
         Channel storage channel = channels[channelId];

         if (channel.finalized) {
             revert ChannelFinalized();
         }
         if (channel.payer == address(0)) {
             revert ChannelNotFound();
         }
-        if (msg.sender != channel.payee) {
-            revert NotPayee();
-        }
+
+        // Verify CloseAuthorization from payee or authorizedSettler
+        bytes32 closeHash = keccak256(abi.encode(CLOSE_AUTHORIZATION_TYPEHASH, channelId, cumulativeAmount));
+        bytes32 closeDigest = _hashTypedData(closeHash);
+        address closeSigner = ECDSA.recoverCalldata(closeDigest, closeAuthorization);
+        address expectedCloseSigner =
+            channel.authorizedSettler != address(0) ? channel.authorizedSettler : channel.payee;
+        if (closeSigner != expectedCloseSigner) {
+            revert InvalidCloseAuthorization();
+        }

         address token = channel.token;
         address payer = channel.payer;
         address payee = channel.payee;
         uint128 deposit = channel.deposit;

         uint128 settledAmount = channel.settled;
         uint128 delta = 0;

        // If cumulativeAmount > settled, validate the voucher
        if (cumulativeAmount > settledAmount) {
            if (cumulativeAmount > channel.deposit) {
                revert AmountExceedsDeposit();
            }

             bytes32 structHash =
                 keccak256(abi.encode(VOUCHER_TYPEHASH, channelId, cumulativeAmount));
             bytes32 digest = _hashTypedData(structHash);
             address signer = ECDSA.recoverCalldata(digest, signature);

             address expectedSigner =
                 channel.authorizedSigner != address(0) ? channel.authorizedSigner : channel.payer;

             if (signer != expectedSigner) {
                 revert InvalidSignature();
             }

             delta = cumulativeAmount - settledAmount;
             settledAmount = cumulativeAmount;
         }

        // Effects before interactions
        uint128 refund = deposit - settledAmount;
        _clearAndFinalize(channelId);

        // Interactions
        if (delta > 0) {
-            bool success = ITIP20(token).transfer(payee, delta);
+            bool success = IERC20(token).transfer(payee, delta);
            if (!success) {
                revert TransferFailed();
            }
        }

         if (refund > 0) {
-            bool success = ITIP20(token).transfer(payer, refund);
+            bool success = IERC20(token).transfer(payer, refund);
             if (!success) {
                 revert TransferFailed();
             }
         }

         emit ChannelClosed(channelId, payer, payee, settledAmount, refund);
     }

     /**
      * @notice Withdraw remaining funds after close grace period.
      * @param channelId The channel to withdraw from
      */
     function withdraw(bytes32 channelId) external override {
         Channel storage channel = channels[channelId];

         if (channel.finalized) {
             revert ChannelFinalized();
         }
         if (channel.payer == address(0)) {
             revert ChannelNotFound();
         }
         if (msg.sender != channel.payer) {
             revert NotPayer();
         }

         address token = channel.token;
         address payer = channel.payer;
         address payee = channel.payee;
         uint128 deposit = channel.deposit;
         uint128 settledAmount = channel.settled;

        // Check if eligible to withdraw
        bool closeGracePassed = channel.closeRequestedAt != 0
            && block.timestamp >= channel.closeRequestedAt + CLOSE_GRACE_PERIOD;

        if (!closeGracePassed) {
            revert CloseNotReady();
        }

         uint128 refund = deposit - settledAmount;
         _clearAndFinalize(channelId);

         if (refund > 0) {
-            bool success = ITIP20(token).transfer(payer, refund);
+            bool success = IERC20(token).transfer(payer, refund);
             if (!success) {
                 revert TransferFailed();
             }
         }

         emit ChannelExpired(channelId, payer, payee);
         emit ChannelClosed(channelId, payer, payee, settledAmount, refund);
     }

     // --- View Functions ---

     /**
      * @notice Get channel state.
      */
     function getChannel(bytes32 channelId) external view override returns (Channel memory) {
         return channels[channelId];
     }

     /**
      * @notice Compute the channel ID for given parameters.
      * @param payer Address that deposited funds
      * @param payee Address authorized to withdraw
-     * @param token TIP-20 token address
+     * @param token ERC-20 token address
      * @param salt Random salt
      * @param authorizedSigner Address authorized to sign vouchers
+     * @param authorizedSettler Address authorized to call settle/close
      */
     function computeChannelId(
         address payer,
         address payee,
         address token,
         bytes32 salt,
-        address authorizedSigner
+        address authorizedSigner,
+        address authorizedSettler
     )
         public
         view
         override
         returns (bytes32)
     {
         return keccak256(
-            abi.encode(payer, payee, token, salt, authorizedSigner, address(this), block.chainid)
+            abi.encode(payer, payee, token, salt, authorizedSigner, authorizedSettler, address(this), block.chainid)
         );
     }

     /**
      * @notice Get the EIP-712 domain separator.
      */
     function domainSeparator() external view override returns (bytes32) {
         return _domainSeparator();
     }

     /**
      * @notice Compute the digest for a voucher (for off-chain signing).
      */
     function getVoucherDigest(
         bytes32 channelId,
         uint128 cumulativeAmount
     )
         external
         view
         override
         returns (bytes32)
     {
         bytes32 structHash = keccak256(abi.encode(VOUCHER_TYPEHASH, channelId, cumulativeAmount));
         return _hashTypedData(structHash);
     }

+    /**
+     * @notice Compute the digest for a close authorization (for off-chain signing).
+     */
+    function getCloseAuthorizationDigest(
+        bytes32 channelId,
+        uint128 cumulativeAmount
+    )
+        external
+        view
+        returns (bytes32)
+    {
+        bytes32 structHash = keccak256(abi.encode(CLOSE_AUTHORIZATION_TYPEHASH, channelId, cumulativeAmount));
+        return _hashTypedData(structHash);
+    }

     /**
      * @notice Read multiple channel states in a single call.
      * @param channelIds Array of channel IDs to query
      * @return channelStates Array of Channel structs
      */
     function getChannelsBatch(bytes32[] calldata channelIds)
         external
         view
         override
         returns (Channel[] memory channelStates)
     {
         uint256 length = channelIds.length;
         channelStates = new Channel[](length);

         for (uint256 i = 0; i < length; ++i) {
             channelStates[i] = channels[channelIds[i]];
         }
     }

     // --- Internal Functions ---

     function _clearAndFinalize(bytes32 channelId) internal {
         delete channels[channelId];
         channels[channelId].finalized = true;
     }

 }
```

## Notes

### Interoperability

A channel opened via `openWithERC3009` has `channel.payer` set to the actual client address (from the ERC-3009 `from` field). This is identical to what `open()` sets (`msg.sender`). Therefore:

- A channel opened with `open()` can be topped up with `topUpWithERC3009()` (and vice versa)
- `channelId` is identical regardless of which open method was used
- `settle`, `close`, `requestClose`, `withdraw` work unchanged on either type of channel



### New Errors

- `InvalidPayer()`: Introduced by `openWithERC3009` (for `payer == address(0)` check). The original `open()` does not need this because `msg.sender` can never be `address(0)`.
- `InvalidCloseAuthorization()`: Introduced by `close`. Reverts when the `CloseAuthorization` signature does not recover to the expected signer (payee or `authorizedSettler`).

### ERC-3009 Failure Mode

`receiveWithAuthorization` reverts on failure (invalid signature, expired authorization, used nonce) rather than returning a bool. No explicit success check is needed — the entire `openWithERC3009` / `topUpWithERC3009` transaction reverts atomically if the token transfer fails.

### `topUpWithERC3009` Authorization

`topUpWithERC3009` passes `channel.payer` as the `from` address to `receiveWithAuthorization`. The ERC-3009 token validates that the signature was produced by this address. This ensures only the original channel payer can authorize additional deposits, even though any address can submit the transaction.
