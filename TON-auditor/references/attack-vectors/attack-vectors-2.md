# Attack Vectors Reference (2/2)

61 total attack vectors

---

**31. Forwarding Unvalidated Raw Internal Messages**

- **D:** If a contract forwards a caller-supplied or externally influenced message cell without rebuilding or validating its schema, opcode, correlated amount fields, value, and send-mode semantics, attackers can smuggle malformed bodies, mismatched transfer amounts, hostile `forward_ton_amount` values, or unexpected message flags into downstream flows. This includes minter or router paths that accept a nested ref such as `master_msg` and pass it directly into a wallet or peer contract after only partial parsing of one field. Even when the path is privileged and the only missing check is an explicit nested opcode assertion, the contract is still accepting and forwarding a peer-message shape it never proved matches the intended standard operation; that weaker subcase should still survive as a low-confidence or minor-style finding rather than being dropped as mere style.
- **FP:** The contract reconstructs the outgoing message from trusted fields, or fully validates the supplied cell before any send.

**32. Cross-Contract Accounting or Supply Desync in Multi-Step Settlement**

- **D:** If a contract updates supply, balances, liquidity, quotas, settlement state, or per-user entitlement before a dependent downstream mint, burn, transfer, provider-credit, or helper-account deployment is irrevocably confirmed, later failure or bounce can leave global accounting out of sync with actual holdings or user claims. This includes authoritative minter logic that increments or decrements `total_supply` before a wallet-side `internal_transfer` / mint / burn consequence is known to succeed, and then ignores or mishandles the bounce path at the authoritative layer. This is especially dangerous when the downstream step is sent with `SEND_MODE_IGNORE_ERRORS`, because the parent path may save state even though the dependent crediting message never executes.
- **FP:** The design reverses the optimistic update on failure, or only mutates the authoritative state after the dependent step is guaranteed complete.

**33. Serialization / State Layout Mismatch or Misbinding**

- **D:** Large flat `load_data()` / `save_data()` patterns and manual message builders make it easy to shadow fields, swap argument order, or serialize a field with one type and parse it back with another. A single misbinding can silently corrupt ownership, balances, configuration, or parser alignment in either storage or message flows, especially when the contract unpacks and repacks far more state than the handler actually needs. This includes adjacent same-typed arguments passed in the wrong order at a helper call site, such as calling `save_data(..., min_amount, swap_fee, ...)` when the signature expects `save_data(..., swap_fee, min_amount, ...)`.
- **FP:** The storage layout is segmented or otherwise structured so load/save ordering, types, and variable binding remain provably consistent across every path.

**34. Missing Entry-Point Validation in Asynchronous Cascades**

- **D:** In a multi-stage message flow, the first reachable entrypoint must enforce the critical invariants for all later stages. If payload checks, gas sufficiency, sender authentication, or phase checks are deferred and only assumed by consequence messages, the cascade can fail mid-flight or reach an inconsistent state after partial execution.
- **FP:** Each stage independently validates its own prerequisites, or downstream failure cannot corrupt state or strand third-party funds.

**35. Cell Bit-Capacity Overflow in Serialized State or Message Bodies**

- **D:** Packing too many addresses, coin values, fixed-width integers, or references into one ordinary cell without a worst-case size budget can exceed TVM limits (`1023` bits and `4` refs). Serialization will then throw, making deployment, state updates, or outbound message construction fail and potentially turning a user-triggerable path into a denial of service.
- **FP:** The full worst-case encoding is proven to fit, or the data is intentionally split across referenced cells.

**36. Balance-Subsidized Sends Due to Incorrect Send-Mode / Value Semantics**

- **D:** TON send modes interact with attached value, fee payment, and carry-all behavior in subtle ways. Combining carry-all modes with explicit non-zero value, fee-separate sends, cashback helpers, optional branches, or external log/event sends can make the contract spend its own TON reserve instead of only forwarding caller-funded value.
- **FP:** The additional spend is intentional, fully budgeted, and the contract reserves its minimum storage balance before the send.

**37. Incorrect Internal Message Header Prefix or Flag Serialization**

- **D:** Short-form internal message serialization is easy to encode incorrectly. Confusing a single flag bit with the full header prefix, composing the wrong bounce and source-address bits, or hand-packing opaque magic numbers into headers can silently make transfers non-bounceable, malformed, or semantically different from what the code intends.
- **FP:** The code uses a known-correct builder pattern or composes the final prefix from validated flag components before serialization.

**38. Bounce-Handler Parsing Beyond the Truncated Payload**

- **D:** Bounced messages include the `0xffffffff` marker plus only the first `256` bits of the original body. Recovery code that tries to read later fields such as a full address, owner identifier, or deep payload data can misparse truncated data, misroute refunds, or throw during the bounce path itself.
- **FP:** Every field read during bounce handling is guaranteed to fit inside the truncated prefix, or the contract tracks needed correlation data in state instead of reconstructing it from the bounce.

**39. Broken Threshold Signature-Set Validation**

- **D:** Any bridge, multisig, verifier-set, or threshold-signature flow that accepts a packet once enough signatures are seen must validate the signature set as a set, not as an unstructured byte stream. If the code fails to advance through a chained signature container correctly, never enforces a minimum threshold, or counts duplicate verifier identities multiple times, attackers can satisfy an intended `k-of-n` authorization with one signature, one repeated verifier, or a malformed signature list. The same root cause appears whether the contract tracks verifier indices, recovered public keys, signer addresses, or explicit signature counters.
- **FP:** The implementation traverses the full signature container correctly, rejects duplicate verifier identities, and enforces the required minimum number of unique valid signers before authorizing the action.

**40. Missing Early `return` After a Handled Branch**

- **D:** If a handler completes its intended logic but does not `return`, execution can fall through into a default `throw` or conflicting branch. In TON, that can discard pending state changes and outbound actions, turning a seemingly handled case into a hidden revert or denial-of-service condition.
- **FP:** The fallthrough is intentional and provably cannot reach a failing or conflicting path.

**41. Missing Domain Separation in Signed External Messages**

- **D:** If a signed payload does not bind a wallet-specific and deployment-specific domain such as wallet id, contract address, workchain, or network context, a valid signature can be replayed against another compatible wallet or another deployment controlled by the same keys.
- **FP:** The contract verifies a domain separator that uniquely ties the signature to this wallet and this deployment.

**42. Replay Drain When `accept_message()` Happens Before Replay State Is Committed**

- **D:** If `recv_external` calls `accept_message()` and then performs fallible parsing or sending before replay-protection state is committed, a later exception will consume gas from contract balance while reverting the nonce or `seqno` update. The same message can then be replayed repeatedly to drain the contract.
- **FP:** Replay-protection state is updated and committed before any fallible work after `accept_message()`, or the handler cannot throw after acceptance.

**43. Unprotected Contract Code Update Path**

- **D:** Any upgrade path that can change contract code or continuation pointers, such as `set_code`, `set_c3`, or an equivalent code-cell swap, must be strictly authenticated and shape-validated. If attacker-controlled input can reach that path, the contract can be replaced, bricked, or subverted in a single transaction.
- **FP:** The update path is unreachable to attackers, tightly authorized, and validates the upgrade payload before mutating code.

**44. Incorrect Refund Recipient Construction**

- **D:** Refund, cashback, and excess-return helpers must carry the correct response or claimant address through message construction. If a helper omits the dedicated refund address, reuses the transfer destination instead, or substitutes another caller-controlled slice parsed from an untrusted message body, excess TON or refunded assets can be misrouted irreversibly.
- **FP:** The protocol intentionally routes all refunds to a fixed, caller-independent recipient and documents that behavior.

**45. Missing State Consumption Update After Claim, Refund, or Settlement**

- **D:** After a claim, refund, or settlement path moves value or consumes entitlement, the contract must decrement the corresponding accounting buckets and persist the new state. If it forgets to reset a consumed field or deduct an unclaimed amount, later operations can double-count, fail permanently, or lock funds.
- **FP:** The consumed amount is tracked in another authoritative field and the apparent omission cannot desynchronize later execution.

**46. Missing Business-Logic Boundary or Eligibility Validation**

- **D:** Business-logic handlers must validate that every critical input falls inside an allowed domain at the point of execution. This includes non-zero amounts, exact or minimum `msg_value` funding for the full downstream cascade, rate existence, deadline ordering, phase boundaries, eligibility flags, available inventory, reserved or unclaimed obligations, mutually exclusive asset modes, and selector or enum inputs such as role types, child op codes, or mode values. Missing or off-by-one guards can let users buy for zero output, let an owner withdraw inventory that should remain reserved for user claims or refunds, call a native-only path while the contract is configured for a jetton mode, claim in the wrong phase, silently accept unsupported operations, or strand assets behind invalid states.
- **FP:** Equivalent invariant checks already run on the same reachable path and the dependent state cannot drift before execution.

**47. Validation Performed in the Wrong Asset or Unit Domain**

- **D:** Protocols that convert between input amounts, output amounts, rates, caps, and unclaimed buckets must validate in the same unit and asset domain actually being transferred or reserved. If the code checks one asset while issuing or crediting another, or checks a gross inventory balance while the real withdrawable amount is the net balance after reserved / unclaimed obligations, attackers can bypass caps, overcommit inventory, or force later claims and refunds to fail. This includes accepting native value while crediting liquidity, entitlement, or balances in a jetton-configured path, validating one asset mode while settling another, or allowing administrative withdrawal from `poolBalance` when only `poolBalance - totalUnclaimed` is actually free.
- **FP:** The conversion is provably exact and the checked variable is intentionally the authoritative unit for both validation and settlement.

**48. Inconsistent or Uninitialized Asset Configuration**

- **D:** When a contract stores both an asset identifier and its configured address, wallet, sentinel, or account fields, every mutation and settlement path must verify that the configuration exists and that the fields still match. Missing existence or consistency checks can settle against the wrong asset or empty configuration. This includes mode-selecting sentinels such as `HOLE_ADDRESS`, `addr_none`, or equivalent native-asset markers: native-only entrypoints must reject jetton-configured pools, and jetton-only entrypoints must reject native-mode pools unless the protocol explicitly supports both.
- **FP:** Asset configuration is derived from a single authoritative source and missing or mismatched entries are impossible by construction.

**49. Claimed Standard Command Body Schema Mismatch**

- **D:** A contract that claims compliance with a standard interface but uses different op codes, field order, field names, field types, or internal peer-message schemas can misread attacker-crafted bodies, reject standard callers, or route value using the wrong fields. This applies to user-facing commands and tightly coupled peer messages alike.
- **FP:** The contract intentionally uses a custom ABI and is not advertised as standard-compatible.

**50. Incorrect Tagged-Union or Optional-Field Parsing in Standard Messages**

- **D:** Standards often rely on exact `Maybe`, `Either`, optional-reference, and inline-vs-ref encodings. If the decoder ignores the tag bit, assumes a reference when the value is inline, or otherwise shifts parsing incorrectly, crafted payloads can alter subsequent control flow or integration data.
- **FP:** The implementation decodes the tagged unions exactly and accepts all standard encodings it claims to support.

**51. Optional Standard Fields Treated as Mandatory or Fabricated**

- **D:** Standard interfaces often permit `addr_none`, omitted owner fields, or optional response addresses. Force-unwrapping those values, rejecting a legal empty case, or fabricating a substitute address turns an optional field into a mandatory or misleading one and breaks compatible callers.
- **FP:** The protocol intentionally tightens the interface in a closed ecosystem and documents that non-standard restriction.

**52. Missing Balance or State Sufficiency Check on Standard Asset Operations**

- **D:** Standard asset operations such as transfer, burn, claim, or ownership reassignment often require rejecting requests when balance, ownership state, or entitlement is insufficient. Skipping that check can allow overspend, phantom burn, invalid ownership movement, or corrupted accounting.
- **FP:** Equivalent sufficiency checks occur before mutation and the numeric model rules out underflow or over-consumption.

**53. Missing Required Refund-Floor, Funding, or Cashback Guarantee in Standard Interfaces**

- **D:** Standard interfaces often require enough attached TON to fund deployment, replies, and a minimum excess return to a nominated address. If the implementation under-checks those guarantees, callers can lose TON, downstream sends can underfund, or the contract can end up subsidizing standard flows from its own balance.
- **FP:** Equivalent preflight accounting guarantees the same minimum refund and funding behavior under the contract's exact send modes and fee model.

**54. Unsupported First-Time Counterparty Deployment Assumption**

- **D:** Standard transfer flows may require the sender side to deploy the receiver-side wallet, item, or helper contract on first use. Implementations that only work for already deployed counterparties can fail on first transfer, break standard compatibility, or leave the cascade underfunded because deployment cost was never accounted for.
- **FP:** The protocol intentionally requires counterparties to be predeployed and documents that non-standard restriction.

**55. Required Standard Notification or Reply Path Missing, Malformed, or Misgated**

- **D:** Standards often require specific notifications or replies only under defined conditions, such as when forward value is non-zero, when a query is received, or when a burn completes. Omitting a required reply, sending one unconditionally, changing its schema, or routing it to the wrong recipient breaks standard settlement and can misattribute deposits or ownership changes.
- **FP:** The contract intentionally uses a different closed-system reply flow and every participant depends on that custom behavior.

**56. Correlation Identifier Not Preserved Across Standard Reply Paths**

- **D:** Standard reply messages commonly require the original `query_id` or correlation id to be echoed back. Replacing it, zeroing it, or generating a new id breaks caller matching logic and can cause wallets, bridges, marketplaces, or exchanges to mis-handle confirmations and refunds.
- **FP:** A different end-to-end correlation scheme is explicitly defined and enforced by every participant.

**57. Claimed Standard Getter or Discovery ABI Mismatch**

- **D:** Standards define exact getter tuple order, return types, and discovery semantics. Reordering fields, returning the wrong address type, using the wrong derivation formula, or serializing discovery responses differently can break wallet verification, explorer decoding, and address authenticity checks. This includes discovery handlers or getters that return the wrong wallet type for the asset being described, derive the address with the wrong wallet code/helper, or make `provide_wallet_address` / `take_wallet_address` disagree with `get_wallet_address` for the same owner.
- **FP:** The contract is intentionally non-standard and all consumers use a matching custom getter and discovery ABI.

**58. Non-Canonical Boolean or Sentinel Semantics**

- **D:** TVM and TON standards often rely on canonical sentinel values such as `-1` and `0` for booleans or specific flags. Returning `1`, raw bit masks, or other non-canonical values can pass simple truthiness checks in some branches while breaking bitwise logic, off-chain decoders, or standard compatibility.
- **FP:** The value is normalized before any security-relevant logic or external consumption, and no caller can observe the non-canonical form.

**59. Invalid Ratio, Fee, or Royalty Parameter Bounds**

- **D:** Fractions used for fees, royalties, rates, or share calculations must respect sane bounds and denominator rules. Exposing `numerator >= denominator`, zero or nonsensical denominators, or malformed payout destinations can cause overcharge, underflow, or broad integration failure.
- **FP:** The values are sanitized before exposure and no caller can observe an invalid tuple.

**60. Standard Metadata or Content Serialization Mismatch**

- **D:** Metadata and content standards depend on exact prefixes, layout selectors, and snake or chunked serialization rules. Using the wrong prefix, mixing encodings, or emitting malformed content cells can make wallets and explorers decode the wrong metadata or fail to verify identity.
- **FP:** The contract intentionally uses a custom metadata layout and does not claim compatibility with the relevant content standard.

**61. Unauthenticated or Uncorrelated Asynchronous Response Handling**

- **D:** TON request-response flows complete through later internal messages. If a callback handler accepts data, value, or confirmation replies without verifying the expected sender and binding the reply to a live outstanding request, attackers can spoof responses, replay stale responses, or finalize the wrong request with attacker-controlled data.
- **FP:** The handler verifies both sender authenticity and correlation against stored pending-request state, or unsolicited and stale replies are harmless by design.
