# Attack Vectors Reference (1/2)

61 total attack vectors

## How To Use This Catalog

- Treat each vector as a generalized root-cause template, not as a project-specific signature.
- Preserve concrete code patterns as aliases of the nearest generalized vector when they are what actually triggered the match.
- Keep the strongest generalized match when multiple vectors describe the same root cause.
- Keep separate findings only when the exploit mechanism or the local fix is materially different.
- Do not merge away parser-integrity findings such as V25 merely because the same parser or cell also participates in another bug.
- `D` describes the defect pattern; `FP` describes the main false-positive boundary.
- Numbering is global across both files; the split is only for bundle size and prompt efficiency.

---

**1. Missing `impure` Modifier on Security-Critical Functions**

- **D:** If a function only enforces checks through `throw_*`-style side effects and does not return a value, omitting `impure` can let the compiler optimize the call away when its result is unused. An authorization or invariant check can therefore disappear entirely.
- **FP:** The function is intentionally pure and has no security-relevant side effects.

**2. Incorrect Method Call Syntax (`.` vs `~`)**

- **D:** In FunC, calling a mutating helper with `.` returns a modified copy but does not update the original variable. If the code meant to mutate state in place, balances, dictionaries, or counters may remain unchanged while later logic assumes the mutation happened.
- **FP:** The code intentionally keeps the original value and stores the returned copy elsewhere.

**3. Signed Integer Abuse in Asset or Voting Logic**

- **D:** Using signed `int` values, or mixing signed and unsigned representations, for balances, shares, quotas, or voting power can let attackers supply negative or boundary values that reinterpret incorrectly. That can invert accounting logic, bypass minimum and maximum checks, or trigger unsafe overflow and underflow behavior.
- **FP:** Negative values are required by the protocol and strict range validation prevents misuse.

**4. Predictable or Attacker-Influenced On-Chain Randomness**

- **D:** Seeding randomness from `cur_lt()`, block time, validator-controlled context, `random()`, or attacker-shaped message data is predictable enough for adversaries to bias value-bearing outcomes. Lotteries, allocations, rarity assignment, and especially randomness used inside `recv_external` can therefore be gamed.
- **FP:** The randomness is explicitly non-security-critical, or the protocol uses a construction such as commit-reveal where no single participant can control the final entropy.

**5. Sensitive Data Exposure in Transaction History**

- **D:** Putting passwords, raw secrets, confidential business data, or secret-derived values into message bodies, persistent cells, or otherwise on-chain-readable execution paths exposes them permanently. Anyone can later recover the data from transaction history, contract state, or local emulation.
- **FP:** The transmitted data is intentionally public or is a properly salted, non-reversible commitment.

**6. Incorrect Bounce Handling**

- **D:** Bounce logic must both recover from failed outbound sends and reject bounced bodies before treating them as fresh commands. If a contract ignores bounces, or processes a bounced payload as a new request, it can debit one side without crediting the other, duplicate actions, or mis-handle recovery. Bounce handling is also not complete protection: bounce generation and processing require gas, and deeper cascade failures may never reach the original entry contract.
- **FP:** The message is intentionally non-bounceable, or the contract proves that bounced messages cannot affect state or accounting.

**7. Account Destruction via Asynchronous Race Conditions**

- **D:** If a contract destroys itself when balance reaches zero, or uses destructive send paths such as mode `128 + 32`, concurrent withdrawals or callbacks can race in TON's asynchronous model. The account may disappear before all pending flows settle, stranding value or enabling unsafe redeployment assumptions.
- **FP:** The contract is intentionally single-use, or destruction is gated by a separate, delayed, and authorized shutdown process.

**8. Execution of Untrusted Third-Party Code**

- **D:** Running external code via `EXECUTE`, `set_c3`, or equivalent continuation control gives untrusted code direct influence over control flow and gas usage. Because out-of-gas failures cannot be safely recovered, an attacker can force unexpected partially committed outcomes.
- **FP:** Only governance-approved or pre-vetted code cells can ever reach the execution path.

**9. Variable or Function Name Collision**

- **D:** FunC permits unusual identifiers, including names that visually resemble operators. Without strict linting, code such as `var++` may be parsed as an identifier instead of an increment, causing silent logic errors in security-critical paths.
- **FP:** The codebase enforces naming rules and linting that eliminate ambiguous identifiers.

**10. Manual Throw of Success Exit Codes (`0` or `1`)**

- **D:** Throwing exit codes `0` or `1` is dangerous because TVM treats them as successful execution. A failed validation that throws one of these codes can look like success and allow later logic or off-chain systems to assume the operation completed normally.
- **FP:** The code intentionally uses a success exit to stop an otherwise valid branch early.

**11. Unsafe Asynchronous State Assumptions / Carry-Value Violation**

- **D:** TON contracts cannot synchronously call another contract's getter or pull authoritative state during the same business flow. Designs that rely on a later reply instead of carrying the needed value or state in the message can operate on stale assumptions after the world has changed.
- **FP:** The queried data is informational only, or the protocol correctly carries the authoritative value through the message cascade.

**12. Improper `recv_internal` vs `recv_external` Trust Model**

- **D:** Putting sensitive logic in the wrong entrypoint can break the intended trust boundary. In particular, external handlers without strict authentication, replay protection, and gas discipline can expose admin or value-moving flows to unauthorized callers.
- **FP:** The external interface is intentionally public and fully replay-safe, or the sensitive logic is only reachable from authenticated internal paths.

**13. Address Format and Workchain Mismatch**

- **D:** Accepting or emitting TON addresses without validating the expected workchain, address kind, or canonical format can send funds or control messages to unintended destinations. Assets sent to unsupported workchains or malformed addresses may be unrecoverable.
- **FP:** The protocol intentionally supports multiple workchains and validates them elsewhere before any value movement.

**14. Non-Bounceable Transfers Used for Value Movement**

- **D:** Sending meaningful value with a non-bounceable flag means failures at an uninitialized or rejecting destination will not return the funds. TON, Jettons, or other assets can then be burned or stranded permanently.
- **FP:** The transfer is intentionally fire-and-forget and the destination path is known to be safe for that exact flow.

**15. Replayable External or Signed Messages**

- **D:** External and signed commands are replayable unless the contract enforces freshness with a `seqno`, nonce, expiration, or equivalent mechanism. A captured message can otherwise be executed multiple times, even if the signature itself is valid.
- **FP:** The operation is truly idempotent, or freshness fields are checked and consumed correctly on every reachable path.

**16. Message Cascade Race Conditions**

- **D:** Multi-step flows that validate a condition early and consume it later can be invalidated by concurrent TON messages before the later step executes. Attackers can exploit the gap by acting as a practical man-in-the-middle of the message flow, starting conflicting cascades in parallel and forcing stale assumptions.
- **FP:** The protocol reserves or locks the relevant state before the asynchronous cascade continues, or the flow is single-step only.

**17. Failure to Return Gas Excesses**

- **D:** If excess TON is not returned to the correct recipient, user funds accumulate as trapped dust inside the contract or are redirected incorrectly. Over time this can distort accounting and threaten storage-fee safety.
- **FP:** Residual TON is intentionally retained as protocol revenue and that behavior is explicitly documented.

**18. Ignored Return Values from Dictionary or System Operations**

- **D:** Many FunC and TVM helpers return success flags that must be checked. Ignoring them can make the contract continue after a failed delete, update, lookup, or low-level operation, causing silent state drift and broken invariants.
- **FP:** The failure case is intentionally handled later and cannot affect any security-relevant path.

**19. Counterfeit Token or Wallet Injection**

- **D:** A contract that accepts deposits or callbacks from token wallets without deriving and verifying the legitimate sender can be tricked by counterfeit wallets or fake assets. Attackers can then spoof deposits, credits, or settlement confirmations.
- **FP:** The implementation recomputes and verifies the expected wallet or peer address from the trusted master or derivation formula before accepting the message.

**20. Unconditional `ACCEPT` / `SETGASLIMIT` of External Messages**

- **D:** External messages do not bring gas; the contract pays from its own balance after `ACCEPT` or a widened gas limit. If either happens before strong validation, attackers can spam the entrypoint and drain the contract through gas consumption alone.
- **FP:** The handler applies a tightly bounded gas limit after validation, or the public external interface is intentionally subsidized and budgeted for that purpose.

**21. Unbounded Gas Consumption / Unhandled OOG**

- **D:** Out-of-gas exceptions cannot be safely recovered with normal error handling. Expensive loops, large dictionary scans, or attacker-inflated workloads can therefore abort execution mid-flow and make critical paths unusable or leave partially updated state assumptions.
- **FP:** The code explicitly bounds work, pre-checks gas, or postpones all meaningful state changes until the expensive work has completed.

**22. Unsigned Parameter Front-Running in External Messages**

- **D:** If only part of an external message is signed, an observer can alter the unsigned fields and rebroadcast it. That can redirect funds, change fees, or reuse a valid signature for a different action.
- **FP:** Every security-critical parameter is included inside the signed payload and enforced on-chain.

**23. Missing Sender or Owner Validation on Protected Operations or Callbacks**

- **D:** Any privileged or state-sensitive handler that does not validate `sender_address`, owner identity, or the expected derived counterparty from the trusted message context can be triggered by unauthorized parties. This includes the concrete FunC anti-pattern of loading a supposed caller with `in_msg_body~load_msg_addr()` or another attacker-controlled payload field and using it for authorization, instead of deriving the real sender from the inbound message envelope such as `in_msg_full.begin_parse()` / `cs~load_msg_addr()`. The same root cause appears in standard wallet commands, burn notifications, settlement callbacks, and other contract-originated messages.
- **FP:** Equivalent sender authentication is enforced before any state change, or the function is intentionally permissionless and safe for arbitrary callers.

**24. Improper Pre-Initialization vs Post-Initialization Logic**

- **D:** If a contract does not clearly separate pre-init and post-init behavior, callers may reach logic that assumes dependent addresses, code cells, or configuration are already set. That can lead to null-address interactions, frozen assets, or inconsistent initialization.
- **FP:** A dedicated initialization flag or phase guard protects every sensitive path.

**25. Incomplete Message Parsing (`end_parse` Omission)**

- **D:** Failing to fully consume a fixed-layout slice after decoding it allows trailing caller-controlled or malformed data to remain unvalidated. This includes nested parsers created with `begin_parse()` such as `permission_cs = ctx_permissions.begin_parse()` and `master_msg_cs = master_msg.begin_parse()`, but it also includes reused payload slices such as `fwd_payload = in_msg_body`, `forward_payload`, helper-returned slices, and authoritative storage parsers such as `slice ds = get_data().begin_parse(); return (...)` in `load_data()` helpers when the storage layout is expected to be exact. A check like `slice_bits(payload) == 267` or `== 534` is not by itself equivalent to `end_parse()`, because extra refs can still remain unless the code also proves `slice_refs(payload) == 0` or otherwise empties the slice. Role-gated or admin-only payload refs still qualify: a missing full-consumption check on a minter/admin-controlled cell is a parser-integrity bug, with confidence reduced for privilege rather than dropped as unreachable. The leftover data can break assumptions about message shape, branch selection, phase logic, stored layout compatibility, or outbound message contents, and the accepted malformed fixed-layout message or storage shape is itself the invariant break even when the immediate impact is not instant fund loss. When a reachable fixed-layout parser lacks `end_parse()` or an equivalent emptiness proof and there is no explicit trailing-extension design, this should normally survive to a standalone finding rather than being absorbed into another issue.
- **FP:** The contract intentionally supports trailing extension data and handles it explicitly, or it enforces full emptiness by equivalent means such as exact bit-and-ref checks or `slice_empty?()` after parsing.

**26. Underfunded Message Cascades / Insufficient `msg_value`**

- **D:** In TON, the entry message must fund the full cascade it creates, including the downstream branches each handler may emit, any `StateInit` deployment cost, and the gas needed for consequence messages to succeed. If the handler accepts too little `msg_value` and still mutates state, later compute or action phases can fail while earlier state changes remain committed, causing partial execution and locked funds. This remains true even when the downstream send uses `SEND_MODE_IGNORE_ERRORS`; ignored action failures do not make an underfunded optimistic state update safe.
- **FP:** The contract explicitly checks that `msg_value` covers worst-case downstream costs, or later failures cannot violate any invariant because the flow is pure carry-value.

**27. Wrong `SEND_MODE` Causing Unintended Rollback**

- **D:** Using rollback-triggering send modes for optional or best-effort messages can make an irrelevant downstream failure revert the entire transaction. A harmless notification or helper transfer failure then becomes a state rollback vulnerability.
- **FP:** The downstream message is critical to correctness and the whole operation should revert if it cannot be delivered.

**28. Unsafe Rejection of Value-Bearing Inbound Requests After Receipt**

- **D:** If a contract receives a request together with TON, Jettons, NFTs, or another asset and then throws, rejects late, or otherwise lacks a safe return path when the payload is malformed, the asset is unsupported, the gas is insufficient, or post-receipt logic fails, value may become stuck or misrouted after receipt. This includes deposit or `transfer_notification` flows that first credit local accounting, then hit a late `throw`, or fall into an outer `catch` / early `return` that suppresses the failure without refunding the user or rolling back the optimistic balance update.
- **FP:** The contract validates all fallible conditions before value moves, or explicitly returns the received value to the authenticated sender on every unsupported or failure path.

**29. Reusing the Same `op` for Different Message Schemas**

- **D:** Assigning the same operation code to different body layouts creates ambiguity for parsers, indexers, relays, and sometimes the contract itself. A handler may decode one schema while receiving another and misinterpret attacker-controlled fields.
- **FP:** The schemas are truly identical and remain security-equivalent across every path that uses that `op`.

**30. Mutable Asset Metadata Contrary to Integrator Expectations**

- **D:** If a token or NFT exposes metadata that integrators expect to be stable, but the contract still allows privileged mutation, an admin or compromised key can change name, symbol, decimals, content, or description to mislead users and downstream systems.
- **FP:** The asset is explicitly documented as mutable and that trust model is part of the intended design assumptions.
