# Shared TON/TVM Attack Vectors

These vectors apply across FunC, Tolk, and Tact because the root cause comes from TON/TVM execution, message flow, standards, gas, bounce behavior, or cross-contract semantics.

---

**TC1. Predictable or Attacker-Influenced On-Chain Randomness**

- **D:** Randomness seeded from `cur_lt()`, block time, validator-controlled context, `random()`, or attacker-shaped messages can be biased in value-bearing flows.
- **FP:** The randomness is non-security-critical or uses a construction such as commit-reveal where no single participant controls final entropy.

**TC2. Sensitive Data Exposure in Transaction History**

- **D:** Secrets placed in message bodies, persistent cells, or emulation-visible runtime values are permanently public.
- **FP:** The data is intentionally public or is only a salted, non-reversible commitment.

**TC3. Incorrect Bounce Handling**

- **D:** Failing to inspect the inbound bounced flag, ignoring bounces, processing bounced payloads as fresh commands, or relying on incomplete bounce recovery can duplicate actions, strand value, or desync accounting.
- **FP:** Bounced messages cannot affect state/accounting, or the flow intentionally uses safe non-bounceable behavior.

**TC4. Account Destruction via Asynchronous Race Conditions**

- **D:** Destroying an account or using destructive send modes while messages or obligations are in flight can strand value or break redeployment assumptions.
- **FP:** Destruction is single-use by design or gated by an authorized shutdown with no pending obligations.

**TC5. Execution of Untrusted Third-Party Code**

- **D:** Executing untrusted code cells or continuations can give an attacker control over flow and gas, causing partially committed outcomes.
- **FP:** Only vetted, authorized code can reach the execution path.

**TC6. Manual Throw of Success Exit Codes**

- **D:** Throwing TVM success exit codes `0` or `1` for failed validation can make a failure look successful to later logic or off-chain systems.
- **FP:** The success exit is an intentional early stop of an otherwise valid branch.

**TC7. Unsafe Asynchronous State Assumptions / Carry-Value Violation**

- **D:** TON contracts cannot synchronously pull current remote state during a business flow; later replies, including balance-query replies, can be stale unless the authoritative value is carried through the cascade.
- **FP:** The queried data is informational only, the result is revalidated before settlement, or the protocol carries authoritative state through messages.

**TC8. Improper Internal vs External Message Trust Model**

- **D:** Sensitive logic exposed through unauthenticated external handlers, or trusted internal-only assumptions applied to external messages, can expose admin or value-moving flows.
- **FP:** The external surface is fully authenticated, replay-safe, and gas-safe.

**TC9. Address Format and Workchain Mismatch**

- **D:** Accepting or emitting addresses without validating workchain, kind, or canonical form can send funds or control messages to unsupported destinations.
- **FP:** Multiple workchains are intentionally supported and validated elsewhere before value movement.

**TC10. Non-Bounceable Transfers Used for Value Movement**

- **D:** Meaningful value sent non-bounceably can be burned or stranded when the deterministic destination address is uninitialized, frozen, missing code, or rejects.
- **FP:** The transfer is intentionally fire-and-forget and the destination is proven safe for that exact flow.

**TC11. Replayable External or Signed Messages**

- **D:** External or signed commands without consumed freshness can be replayed multiple times.
- **FP:** The operation is idempotent or freshness fields are checked and consumed on every path.

**TC12. Message Cascade Race Conditions**

- **D:** Multi-step flows that validate a property early and consume it later, even in a later stage of the same logical contract flow, can be invalidated by concurrent TON messages before the later step executes.
- **FP:** The relevant state is reserved/locked before the cascade continues, or the flow is single-step.

**TC13. Failure to Return Gas Excesses**

- **D:** In standard or user-funded flows, excess TON not returned to the expected recipient can trap user funds, distort accounting, or threaten storage-fee safety.
- **FP:** Residual TON is documented protocol revenue, explicitly credited to the user elsewhere, negligible dust by design, or outside any standard/user-refund obligation.

**TC14. Counterfeit Token or Wallet Injection**

- **D:** Deposit or callback handlers that do not derive and verify the legitimate token wallet or peer can be spoofed by counterfeit wallets/assets.
- **FP:** The expected wallet or peer is recomputed from trusted master/owner data before acceptance.

**TC15. Unconditional Gas Acceptance of External Messages**

- **D:** External messages do not bring gas; accepting or widening gas before validation lets attackers drain contract balance.
- **FP:** Gas is accepted only after validation or the public subsidy is explicitly bounded.

**TC16. Unbounded Gas Consumption / Unhandled OOG**

- **D:** Attacker-inflated loops, dictionary scans, or workloads can abort mid-flow and leave committed state or denial of service.
- **FP:** Work is bounded, gas is prechecked, or state changes happen only after expensive work.

**TC17. Unsigned Parameter Front-Running in External Messages**

- **D:** If a signature covers only part of an external payload, observers can mutate unsigned fields and rebroadcast.
- **FP:** Every security-critical parameter is signed and enforced on-chain.

**TC18. Missing Sender or Owner Validation on Protected Operations or Callbacks**

- **D:** Privileged or state-sensitive handlers that trust payload-derived identities instead of the inbound sender or expected peer can be called by unauthorized parties.
- **FP:** Equivalent sender/counterparty authentication is enforced before state change, or the function is safely permissionless.

**TC19. Improper Pre-Initialization vs Post-Initialization Logic**

- **D:** If pre-init and post-init behavior are not separated, callers may reach logic that assumes unset addresses, code cells, or config.
- **FP:** A dedicated init/phase guard protects every sensitive path.

**TC20. Underfunded Message Cascades / Insufficient Message Value**

- **D:** If an entry message underfunds the full downstream cascade but state is still mutated, later phases can fail after earlier changes commit.
- **FP:** Worst-case downstream cost is checked, or later failure cannot violate invariants.

**TC21. Wrong Send Mode Causing Unintended Rollback**

- **D:** Rollback-triggering modes on optional sends can let irrelevant downstream failures revert otherwise valid state changes.
- **FP:** The downstream message is correctness-critical and the whole operation should fail if delivery fails.

**TC22. Unsafe Rejection of Value-Bearing Inbound Requests After Receipt**

- **D:** Throwing, silently returning, or swallowing errors after accepting value can trap or misroute TON, Jettons, NFTs, or other assets; unsupported value-bearing requests should return the received asset to the authenticated sender rather than reject after receipt.
- **FP:** All fallible conditions are validated before value moves, or received value is returned on every unsupported/failure path.

**TC23. Reusing the Same Operation Code for Different Message Schemas**

- **D:** One opcode used for multiple layouts lets parsers or peers decode one schema while receiving another.
- **FP:** The schemas are identical and security-equivalent across every path.

**TC24. Mutable Asset Metadata Contrary to Integrator Expectations**

- **D:** Privileged metadata mutation can mislead wallets, users, or downstream systems when the asset claims stable identity, immutable metadata, or standard-compatible display semantics.
- **FP:** Metadata mutability is explicit in the protocol/user trust model, cannot affect on-chain accounting or entitlement, and does not violate an advertised immutability or compatibility promise.

**TC25. Forwarding Unvalidated Raw Internal Messages**

- **D:** Forwarding caller-supplied message cells without rebuilding or validating opcode, amount, destination, value, layout, and send-mode semantics lets attackers smuggle unsafe downstream actions.
- **FP:** The outbound message is rebuilt from trusted fields or fully validated before send.

**TC26. Cross-Contract Accounting or Supply Desync in Multi-Step Settlement**

- **D:** Authoritative supply, balance, liquidity, quota, or entitlement saved before dependent downstream success can desync on failure or bounce.
- **FP:** The design mutates authoritative state only after the required consequence is guaranteed within the same transaction/action semantics, or it has explicit bounce/failure reconciliation that restores the invariant.

**TC27. Missing Entry-Point Validation in Asynchronous Cascades**

- **D:** Critical gas, payload, sender, or phase checks deferred to later stages can fail mid-cascade after value or state has already moved.
- **FP:** Each stage validates its prerequisites, or downstream failure cannot corrupt state or strand funds.

**TC28. Cell Bit-Capacity Overflow in Serialized State or Message Bodies**

- **D:** Packing too many bits or refs into one ordinary cell can exceed TVM limits and make deployment, updates, or outbound messages fail.
- **FP:** Worst-case encoding is proven to fit or intentionally split across refs.

**TC29. Balance-Subsidized Sends Due to Incorrect Send-Mode / Value Semantics**

- **D:** Combining carry-all, explicit value, fee-separate sends, cashback, optional branches, storage-fee assumptions, or external-out logs can spend contract balance instead of caller-funded value.
- **FP:** The spend is intentional, budgeted, and protected by reserve discipline.

**TC30. Incorrect Internal Message Header Prefix or Flag Serialization**

- **D:** Incorrect short-form internal message flags can make messages non-bounceable, malformed, or semantically different from intended.
- **FP:** A known-correct builder or validated flag composition is used.

**TC31. Bounce-Handler Parsing Beyond the Truncated Payload**

- **D:** Bounced bodies contain only a prefix of the original body; reading later fields can misroute refunds or throw during recovery.
- **FP:** Every read field fits the bounce prefix or correlation data is stored elsewhere.

**TC32. Broken Threshold Signature-Set Validation**

- **D:** Threshold flows that do not traverse, deduplicate, and count signers correctly can authorize with too few or repeated signatures.
- **FP:** The implementation rejects duplicates and enforces the required unique valid signer threshold.

**TC33. Missing Domain Separation in Signed External Messages**

- **D:** Signatures not bound to wallet/deployment/workchain/network context can replay across compatible contracts.
- **FP:** A domain separator uniquely ties the signature to this contract and deployment.

**TC34. Replay Drain When Gas Is Accepted Before Replay State Is Committed**

- **D:** If gas is accepted and later work can fail before replay state is committed, the same external message can be replayed to drain balance.
- **FP:** Replay state is committed before any fallible work after gas acceptance, or no such work can fail.

**TC35. Unprotected Contract Code Update Path**

- **D:** Attacker-controlled code update paths can replace, brick, or subvert the contract.
- **FP:** Upgrade is unreachable to attackers, authorized, and validates payload shape.

**TC36. Incorrect Refund Recipient Construction**

- **D:** Refund, cashback, and excess helpers that use the wrong recipient can irreversibly misroute funds.
- **FP:** Routing to a fixed recipient is intentional and documented.

**TC37. Missing State Consumption Update After Claim, Refund, or Settlement**

- **D:** Moving value without decrementing or clearing the consumed entitlement can double-count, lock funds, or break later claims.
- **FP:** Another authoritative field consumes the entitlement and prevents desync.

**TC38. Missing Business-Logic Boundary or Eligibility Validation**

- **D:** Missing checks for nonzero amounts, deadlines, rates, phase, eligibility, inventory, selector values, or obligations can allow invalid execution or stranded assets.
- **FP:** Equivalent invariant checks already run and cannot drift before execution.

**TC39. Validation Performed in the Wrong Asset or Unit Domain**

- **D:** Checking one asset/unit while transferring or reserving another can bypass caps, overcommit inventory, or block later claims.
- **FP:** Unit transformation is exact and the checked variable is authoritative for both validation and settlement.

**TC40. Inconsistent or Uninitialized Asset Configuration**

- **D:** Missing checks that asset identifiers, sentinels, configured wallets, or account fields exist and match can settle in the wrong mode or asset.
- **FP:** Asset configuration is derived from one authoritative source and mismatch is impossible.

**TC41. Claimed Standard Command Body Schema Mismatch**

- **D:** Standard-compatible contracts using wrong opcodes, field order, types, or peer-message schemas can misread callers or break routing.
- **FP:** The ABI is intentionally custom and not advertised as standard-compatible.

**TC42. Incorrect Tagged-Union or Optional-Field Parsing in Standard Messages**

- **D:** Ignoring `Maybe`, `Either`, inline/reference tags, or optional-field encoding can shift parsing and alter flow.
- **FP:** The implementation decodes all claimed standard encodings exactly.

**TC43. Optional Standard Fields Treated as Mandatory or Fabricated**

- **D:** Force-unwrapping, rejecting, or fabricating legal optional fields breaks compatible callers or misleads recipients.
- **FP:** The interface is intentionally tightened in a closed ecosystem.

**TC44. Missing Balance or State Sufficiency Check on Standard Asset Operations**

- **D:** Missing sufficiency checks on transfer, burn, claim, or ownership reassignment can overspend or corrupt ownership/accounting.
- **FP:** Equivalent checks occur before mutation and the numeric model prevents over-consumption.

**TC45. Missing Required Refund-Floor, Funding, or Cashback Guarantee**

- **D:** Under-checking a claimed standard, user-funded cascade, deployment, or excess-return path can lose user TON, underfund downstream sends, or subsidize execution from contract balance.
- **FP:** Preflight accounting guarantees equivalent funding under the actual send modes and fee model, or the protocol explicitly documents that the caller must predeploy/fund the counterparty and no user value can be stranded.

**TC46. Unsupported First-Time Counterparty Deployment Assumption**

- **D:** Standard flows that only work for predeployed counterparties can fail first transfers or underfund deployment.
- **FP:** Predeployment is an intentional documented protocol requirement.

**TC47. Required Standard Notification or Reply Path Missing, Malformed, or Misgated**

- **D:** In claimed standard interfaces, missing, unconditional, malformed, or misrouted standard replies can break settlement, attribution, or caller reconciliation.
- **FP:** A closed-system custom reply flow is intentionally used by every participant.

**TC48. Correlation Identifier Not Preserved Across Standard Reply Paths**

- **D:** Replacing or dropping `query_id` or correlation identifiers breaks caller matching and confirmation/refund handling.
- **FP:** A different end-to-end correlation scheme is explicitly enforced.

**TC49. Claimed Standard Getter or Discovery ABI Mismatch**

- **D:** Getter/discovery order, types, address derivation, or response encoding mismatches break wallet/explorer verification and address authenticity.
- **FP:** The contract is intentionally non-standard and consumers use a matching custom ABI.

**TC50. Non-Canonical Boolean or Sentinel Semantics**

- **D:** Non-canonical booleans or sentinels can pass truthiness checks but break bitwise logic, decoders, or standards.
- **FP:** Values are normalized before security use and external observation.

**TC51. Invalid Ratio, Fee, or Royalty Parameter Bounds**

- **D:** Invalid numerators, zero denominators, out-of-range fee bounds, or unusable payout destinations can overcharge, underflow, trap payments, or break integrations.
- **FP:** Values are sanitized before use and exposure, or an unusual economic range is explicitly documented and cannot underflow, exceed available proceeds, or break supported integrations.

**TC52. Standard Metadata or Content Serialization Mismatch**

- **D:** Wrong metadata/content prefixes, layout selectors, snake encoding, or chunked encoding in claimed standard metadata can make wallets/explorers misdecode identity.
- **FP:** The contract intentionally uses a custom metadata layout, does not claim compatibility, and supported consumers use the same custom decoder.

**TC53. Unauthenticated or Uncorrelated Asynchronous Response Handling**

- **D:** Callback handlers that do not verify expected sender and live request correlation can accept spoofed or stale responses.
- **FP:** Sender authenticity and correlation are verified, or unsolicited/stale replies are harmless.

**TC54. Unsafe Manual `commit()` Before Fallible Work**

- **D:** Calling `commit()` after partially updating persistent state or queued actions, then performing fallible parsing, validation, sends, or reserve work, can preserve an unsafe intermediate state even when later compute fails.
- **FP:** The committed state is only replay-protection or another deliberately safe checkpoint, and all post-commit failures are proven not to violate accounting, authorization, or message-flow invariants.

**TC55. On-Chain Human-Format Parsing / Unbounded Decoding**

- **D:** Parsing base64, strings, comments, JSON-like formats, metadata blobs, or other human-friendly encodings on-chain can create attacker-controlled gas costs, OOG denial of service, or partial execution after earlier state changes.
- **FP:** The parser is strictly bounded, pre-funded, and runs before any meaningful state mutation or value acceptance; or parsing is intentionally off-chain with compact binary input on-chain.

**TC56. Bounce Mode Mismatch With Recovery or Value Ownership**

- **D:** Sending a message bounceably when the destination should keep attached value, or non-bounceably when the sender must roll back state on rejection, can trap value, block Jettons, or desync supply/balances.
- **FP:** The selected bounce mode matches both value ownership and recovery semantics, and all failed-message consequences are either harmless or explicitly reconciled.

**TC57. Cross-Message Temporary State Reuse**

- **D:** Saving temporary request, pending, rollback, query-id, claim, sale, stake, vote, mint, burn, tax, escrow, or callback state for a later message lets concurrent, stale, successful, bounced, or unrelated flows consume it out of order. Missing success cleanup, query-id-only matching, or rollback from mutable current state can corrupt amounts, recipients, entitlements, storage, or completed accounting.
- **FP:** The state is keyed by a unique request id plus expected sender/destination/message type, consumed or invalidated on every terminal path, and stale or concurrent replies cannot affect another flow.

**TC58. Parent-Child Peer Authentication Failure**

- **D:** Master/child contract groups that trust payload-supplied parent, child, owner, index, or wallet fields instead of recomputing or checking the expected peer address can process spoofed messages from fake children, fake parents, or unrelated contracts.
- **FP:** Every parent-child direction authenticates `sender` against a peer derived from trusted code/state/index or stored state, and replies are correlated to an outstanding request before state changes.

**TC59. Tax, Fee, or Net-Amount Accounting Mismatch**

- **D:** A contract records or validates gross token amounts while a wallet, master, tax window, fee-on-transfer rule, burn split, royalty, redistribution leg, or transfer hook delivers a lower net amount or diverts part of the value. This can underpay users, overstate rewards, desync supply, or make redistribution silently incomplete.
- **FP:** The protocol uses a non-taxed transfer path for internal payouts, explicitly books the net amount received by each beneficiary, or reconciles every fee/tax/burn/redistribution leg on failure.

**TC60. Business Math Boundary or Rounding Error**

- **D:** Vesting, staking, sale, governance, reward, fee, or cap math rounds periods, ratios, decimals, quorum, thresholds, or tranche counts in a direction that releases value early, blocks a legitimate claim/exit/vote, overcharges, underpays, or corrupts authoritative accounting.
- **FP:** Boundary cases are explicitly handled, rounding direction is documented and safe for the protected invariant, and the final period/cap/quorum/threshold reaches the intended exact state.
