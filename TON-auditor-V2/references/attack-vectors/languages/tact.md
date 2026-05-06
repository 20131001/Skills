# Tact Language-Specific Attack Vectors

These vectors depend on Tact receivers, typed messages, fallback parsing, state persistence, or control flow. They are bundled only for Tact targets.

---

**TA1. Signed Integer Abuse in Asset or Voting Logic**

- **D:** Tact integer fields used for balances, shares, quotas, voting power, or message amounts can admit negative or boundary values when the business domain expects unsigned values.
- **FP:** Negative values are required and strict range validation prevents misuse.

**TA2. Unsafe Optional Handling or Failed Helper Outcomes**

- **D:** Optional variables and fields default to `null`. Unsafe `!!` unwraps, fabricated defaults, ignored nullable map lookups, failed helper outcomes, or unchecked native/low-level results can turn an absent value into a late throw, stale assumption, or incorrect state transition.
- **FP:** The absent/failure case is checked before unwrap, explicitly rejected or refunded, or proven unable to affect security-relevant state.

**TA3. Incomplete `Slice` Fallback or Embedded Payload Parsing**

- **D:** Tact typed messages are schema-driven, but `Slice` fallbacks, embedded cells, forwarded payloads, and custom parsers still need exact-layout proof before decoded values affect state or outbound messages.
- **FP:** The receiver explicitly supports extension data, exact shape is proven before use, or parsing failure occurs before any state, value, gas-acceptance, or outbound-message effect.

**TA4. Typed Message / Storage Layout Mismatch or Misbinding**

- **D:** Tact message structs, storage fields, traits, custom serialization, or getter schemas can mismatch the claimed TON schema and corrupt state, parsing, or integration.
- **FP:** The schema is centralized and field order/types are proven consistent with every peer and claimed standard.

**TA5. Missing Termination After a Handled Receiver Branch**

- **D:** Receiver or helper branches that perform their intended action but continue into later default/error logic can revert, double-send, or execute conflicting logic.
- **FP:** The fallthrough is intentional and proven not to reach a failing or conflicting path.

**TA6. Tact Optional Address / `addr_none` Schema Mismatch**

- **D:** Using non-optional `Address` where a claimed TON standard allows `addr_none` or `Maybe ^MsgAddress` can make otherwise valid Jetton, NFT, discovery, or reply messages fail mid-flow and strand assets.
- **FP:** The contract intentionally tightens the ABI in a closed system, or every peer is proven to never send `addr_none` for that field.

**TA7. Tact Default `Int` Serialization Mismatch**

- **D:** Omitting explicit serialization on externally communicated `Int` fields defaults to a signed 257-bit shape, which can mismatch standards expecting `coins`, `uint32`, `uint64`, `uint256`, or another exact width and can also admit negative business values.
- **FP:** The field is purely internal and never serialized across a trusted boundary, or the default signed width is explicitly required and range-checked.

**TA8. Tact Safety Options Disabled for Security-Critical Code**

- **D:** Disabling relevant `tact.config.json` safety checks for performance can remove runtime protections that security-critical arithmetic, parsing, or message handling depends on.
- **FP:** The disabled checks are irrelevant to reachable security-critical logic, or they are replaced by explicit equivalent guards on every reachable path and the performance tradeoff is documented.

**TA9. Tact Fallback Receiver Accepts Unsupported Value-Bearing Messages**

- **D:** Empty, text, string, or `Slice` fallback receivers that silently accept or cashback unsupported messages can hide malformed command delivery, trap non-TON assets, or bypass typed receiver validation.
- **FP:** The fallback is deployment-only or explicitly returns all attached value/assets and cannot mutate state or acknowledge unsupported protocol commands.

**TA10. Mutable Helper Does Not Persist Intended State Change**

- **D:** Tact mutable functions mutate by replacing `self` with the execution result. Helpers that update only a temporary value, fail to change `self`, or whose returned mutation is ignored can leave persistent state unchanged while later sends, accounting, or authorization logic assumes the mutation happened.
- **FP:** The mutation is intentionally local-only, or the caller persists the changed value before any security-relevant effect depends on it.

**TA11. External Receiver Relies on Missing Context or Sender**

- **D:** Tact external messages have no inbound sender/value context. External handlers or shared helpers that rely on `context()`, `sender()`, or internal-message assumptions can expose unauthenticated logic, fail after gas is accepted, or charge contract balance for attacker-controlled work.
- **FP:** The external path validates signatures, freshness, authorization, message shape, and bounded gas before acceptance, and it does not rely on internal-message context.

**TA12. Bounceable Send Without Matching `bounced` Recovery**

- **D:** A Tact contract that sends bounceable messages but lacks a matching `bounced()` handler, trait-provided bounce recovery, or other explicit reconciliation can leave committed state, value ownership, or supply/accounting unrecovered when the destination rejects.
- **FP:** The selected bounce mode is harmless for the value owner, or another explicit path reconciles every failed-message consequence.

**TA13. Unbounded Tact Map or Array State Growth**

- **D:** Attacker-controlled insertion into maps, arrays, or per-user collections without a hard cap, pruning rule, or funding discipline can grow persistent state until gas, storage, or cell-size limits block normal contract operation.
- **FP:** Growth is capped by protocol rules, prepaid and reserved per entry, pruned before limits are reachable, or limited to trusted operators.

**TA14. Attacker-Supplied Map Replacement or Bulk State Import**

- **D:** Tact messages can carry maps and complex typed values. Assigning a caller-supplied map, array-like dictionary, or bulk config object directly into persistent state can bypass per-entry validation, size limits, key-domain checks, duplicate-prevention assumptions, and storage funding discipline.
- **FP:** Every imported entry is bounded, validated, canonicalized, and funded before persistence, or the sender is a trusted operator whose authority includes replacing that full state.

**TA15. Trait-Injected Receiver or Guard Coverage Gap**

- **D:** Traits such as ownership, stoppable/resumable behavior, and custom traits can add persistent state, receivers, and helper guards to the concrete contract. Hidden or inherited receivers, missing `requireOwner()` / `requireNotStopped()` coverage, unsafe owner transfer targets, or overridden trait hooks can expose admin, pause, or value-moving paths that the concrete contract did not intend.
- **FP:** The concrete contract enumerates inherited receivers, initializes trait state safely, and applies the intended guards to every sensitive path, including trait-provided entrypoints.

**TA16. Parent-Child Derived Address Authentication Drift**

- **D:** Parent-child Tact patterns derive child addresses from `initOf` / `contractAddress` and often route by child sequence, owner, or parent fields. If a parent trusts child identity from message fields, or a child trusts a payload parent instead of `sender()` matching the derived address or stored parent, forged siblings or unrelated contracts can spoof callbacks and move parent-managed state.
- **FP:** Each direction authenticates the expected peer from trusted state and derived `StateInit`, and request/response messages are correlated before state changes.

**TA17. Lazy Deployment `StateInit` / Destination Drift**

- **D:** Tact lazy deployment sends `code` and `data` alongside a destination address derived from `StateInit`. If `to`, `code`, and `data` are built from different inputs, user-controlled init parameters are not constrained, or parent state is committed before deployment/funding success is guaranteed, the contract can deploy or fund the wrong child, strand value, or desync parent accounting.
- **FP:** The destination is recomputed from the exact `StateInit` being attached, init parameters are authorized and bounded, and parent state changes are reconciled if deployment or initialization fails.
