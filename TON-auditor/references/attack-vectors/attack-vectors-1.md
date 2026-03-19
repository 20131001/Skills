# Attack Vectors Reference (1/4)

170 total attack vectors

---

**1. Missing `impure` Modifier on Security-Critical Functions**

- **D:** If a function only throws via `throw_unless` and does not return a value, but is missing the `impure` modifier, the compiler may optimize the call away when its result is unused. A required authorization or validation step can therefore disappear entirely.
- **FP:** The function is intentionally pure and has no side effects beyond stack-local computation.

**2. Incorrect Method Call Syntax (`.` vs `~`)**

- **D:** In FunC, calling a mutating helper with `.` returns a modified copy but does not update the original variable. If the developer meant to mutate state in place, they must use `~` or explicitly reassign the returned value. Misusing `.` can leave balances, ledgers, or dictionaries unchanged.
- **FP:** The code intentionally preserves the original value and stores the returned copy elsewhere.

**3. Signed Integer Abuse in Asset or Voting Logic**

- **D:** Using signed `int` values for balances, shares, or voting power can let an attacker supply a negative amount. That can invert accounting logic, for example turning `balance += amount` into a decrement or `balance -= amount` into an increase.
- **FP:** Negative values are required by the business logic and strict range validation prevents misuse.

**4. Predictable On-Chain Randomness**

- **D:** Seeding randomness from `cur_lt()`, block time, or easily controlled message data is predictable enough for validators or attackers to bias outcomes. In value-bearing flows, this can let them selectively participate only when the result is favorable.
- **FP:** The randomness is explicitly documented as weak and is not used to allocate assets or privileged outcomes.

**5. Sensitive Data Exposure in Transaction History**

- **D:** Passing passwords, raw secret-derived values, or other confidential data in message bodies permanently exposes them on-chain. Anyone can recover the plaintext later by parsing historical transactions.
- **FP:** The transmitted data is intentionally public or is a properly salted, non-reversible commitment.

**6. Missing or Weak Bounce Handling**

- **D:** When a sent message fails and bounces, the sender must handle the bounced path explicitly. If it does not, funds or state may be debited on the sender side without the intended credit or state transition ever occurring on the recipient side.
- **FP:** The message is intentionally non-bounceable, or the destination is guaranteed to accept that exact message flow.

**7. Account Destruction via Asynchronous Race Conditions**

- **D:** If a contract destroys itself when its balance reaches zero, multiple concurrent withdrawals can race in TON's asynchronous model. The account may be destroyed before all pending flows settle, creating a window for stranded value or unsafe redeployment.
- **FP:** The contract is intentionally single-use, or destruction is gated by a separate delayed and authorized process.

**8. Execution of Untrusted Third-Party Code**

- **D:** Running external code via `EXECUTE` gives untrusted code direct influence over control flow and gas usage. Because out-of-gas failures cannot be safely caught, an attacker can force execution into an unexpected partially committed outcome.
- **FP:** Only governance-approved or pre-vetted code cells can ever be executed.

**9. Variable or Function Name Collision**

- **D:** FunC permits unusual identifier names, including characters that resemble operators. Without strict linting, code such as `var++` may be parsed as an identifier instead of an increment, causing silent logic errors.
- **FP:** The codebase enforces strict linting and naming conventions that forbid ambiguous identifiers.

**10. Manual Throw of Success Exit Codes (`0` or `1`)**

- **D:** Throwing exit codes `0` or `1` is dangerous because TVM treats them as successful execution. A failed validation that throws one of these codes can appear successful and allow downstream logic to proceed incorrectly.
- **FP:** The code intentionally uses a success exit to terminate an otherwise valid branch early.

---

---

**11. Synchronous Data Pull Assumption**

- **D:** TON contracts cannot synchronously read another contract's current state during the same transaction flow. Code that assumes an immediate response may operate on stale, missing, or attacker-invalidated data.
- **FP:** The design correctly uses a request-response flow or carries the needed state/value inside the message itself.

**12. Improper `recv_internal` vs `recv_external` Usage**

- **D:** Placing sensitive logic in the wrong entrypoint can break the intended trust model. In particular, external handlers without `accept_message()` discipline, replay protection, and authentication checks can expose admin or value-moving paths to unauthorized callers.
- **FP:** The contract intentionally omits `recv_external`, or its external interface is fully authenticated and replay-safe.

**13. Failure to Filter Bounced Messages**

- **D:** If `recv_internal` does not reject bounced messages up front, a bounced payload may be processed as if it were a fresh command. That can trigger phantom operations, duplicate accounting, or invalid state transitions.
- **FP:** The contract's logic is explicitly unaffected by bounced messages and no state is derived from them.

**14. Address Format and Workchain Mismatch**

- **D:** Accepting TON addresses without validating the expected workchain can send funds or control messages to unsupported or unintended destinations. Assets transferred to an unreachable workchain may be lost permanently.
- **FP:** The protocol intentionally supports multiple workchains and validates them elsewhere in the flow.

**15. Non-Bounceable Transfers Used for Value Movement**

- **D:** Sending valuable transfers with a non-bounceable flag means failures at an uninitialized or rejecting destination will not return the funds. This can permanently burn TON or tokens.
- **FP:** The transfer is intentionally fire-and-forget, or the transferred amount is trivial and the recipient path is known.

**16. Missing Replay Protection for External Messages**

- **D:** External messages are replayable unless the contract tracks freshness with a `seqno`, nonce, or equivalent mechanism. A captured signed message can otherwise be executed multiple times.
- **FP:** The operation is truly idempotent and repeated execution has no harmful effect.

**17. Message Cascade Race Conditions**

- **D:** Multi-step flows that span several blocks can validate a condition early and consume it later, after the underlying state has changed. An attacker can exploit that delay by starting conflicting flows in parallel.
- **FP:** The contract uses single-step logic only, or the state is reserved/locked before the asynchronous cascade begins.

**18. Carry-Value Pattern Violation**

- **D:** Querying another contract for critical balances or state instead of carrying the relevant value in the message creates a stale-state risk. By the time the response arrives, the referenced state may already have changed.
- **FP:** The queried information is non-critical and temporary inconsistency cannot affect financial correctness.

**19. Failure to Return Gas Excesses**

- **D:** If excess gas is not returned to the appropriate recipient, user funds accumulate as trapped dust inside the contract. Over time this can distort accounting and even threaten the contract's ability to pay storage fees correctly.
- **FP:** The protocol explicitly treats residual gas as revenue and documents that behavior.

**20. Ignored Return Values from Dictionary or System Operations**

- **D:** Many FunC and TVM helpers return a success flag that must be checked. Ignoring it can make the contract continue after a failed delete, update, or lookup, causing broken invariants and exploitable state drift.
- **FP:** The failure case is intentionally handled by later logic and cannot lead to an unsafe state transition.

---

---

**21. Fake Jetton Injection**

- **D:** A contract that accepts `op::internal_transfer` without verifying that the sender is the legitimate Jetton wallet for the expected minter can be tricked by a counterfeit Jetton. Attackers can then spoof deposits and extract real assets or credits.
- **FP:** The contract recomputes and verifies the expected wallet address from the official Jetton minter before accepting the transfer.

**22. Unconditional `ACCEPT` of External Messages**

- **D:** External messages do not bring gas; the contract pays from its own balance after `ACCEPT`. If `ACCEPT` is executed before strong validation, an attacker can spam the entrypoint and drain the contract through gas consumption alone.
- **FP:** The external handler sets a tightly bounded gas limit or is intentionally public and funded for that purpose.

**23. Unbounded Gas Consumption / Unhandled OOG**

- **D:** Out-of-gas exceptions cannot be recovered with normal error handling. Expensive loops, large dictionary scans, or attacker-inflated workloads can therefore abort execution mid-flow and make critical paths permanently unusable.
- **FP:** The code bounds work explicitly, pre-checks gas, or postpones all meaningful state changes until successful completion.

**24. Predictable Use of TON `random()`**

- **D:** TON's `random()` is derived from chain-controlled context and is not suitable for adversarial value allocation. In lotteries, NFT rarity, or gambling logic, an attacker may predict or bias outcomes enough to capture value.
- **FP:** The contract uses commit-reveal or another protocol where no single participant can unilaterally predict the final entropy.

**25. Unsigned Parameter Front-Running in External Messages**

- **D:** If only part of an external message is signed, an attacker who sees the message can alter the unsigned fields and rebroadcast it. That can redirect funds, change fees, or otherwise reuse a valid signature for a different action.
- **FP:** Every security-critical parameter is included inside the signed payload.

**26. Replayable Signed Messages**

- **D:** A valid signature is not enough by itself. If the signed payload lacks a nonce, `seqno`, or expiration that is actually enforced on-chain, the same signed command can be replayed repeatedly.
- **FP:** The operation is one-time by construction, or freshness fields are checked and consumed correctly.

**27. Missing Sender Validation on Protected Operations**

- **D:** Any state-changing or privileged handler that does not validate `sender_address` can be triggered by unauthorized parties. For contract-originated messages, the sender often must be verified against an expected wallet or StateInit-derived address.
- **FP:** The function is intentionally permissionless and its effects are safe for arbitrary callers.

**28. Improper Pre-Initialization vs Post-Initialization Logic**

- **D:** If a contract does not clearly separate pre-init and post-init behavior, callers may reach logic that assumes dependent addresses or state are already set. That can lead to null-address interactions, frozen assets, or inconsistent initialization.
- **FP:** A dedicated initialization flag or phase check gates every sensitive path.

**29. Incomplete Message Parsing (`end_parse` Omission)**

- **D:** Failing to call `end_parse()` after decoding a message allows trailing attacker-controlled data to remain unvalidated. That can break assumptions about message shape and create unexpected downstream behavior.
- **FP:** The contract intentionally supports trailing extension data and handles it explicitly.

**30. Incomplete Gas Provision for Message Chains**

- **D:** A multi-contract flow can succeed in compute but fail in the action phase if not enough gas is reserved for outgoing messages. Without explicit rollback or bounce-safe design, upstream state may remain changed while downstream effects never occur.
- **FP:** The contract accounts for worst-case gas and has a correct rollback or bounce-recovery path.