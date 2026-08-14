# Attack Vectors Reference (5/6) — Tolk, Audit Completeness & Action Semantics

## V121 — Tolk Lazy Validation Bypass

**What:** A security-critical field in a `lazy` message or storage value is never accessed, so its deserialization or validation does not run.

**Why it matters:** An attacker can submit malformed or forbidden data that appears covered by the type but reaches state-changing logic without the intended check.

**What to look for:**
- `lazy Union.fromSlice(...)` or lazy storage where authorization, amount, tag, or address fields are not read
- State writes, sends, or accepts before all critical lazy fields are accessed
- Validation helpers that receive a lazy object but inspect only a subset of security fields

---

## V122 — Tolk Permissive Union Fallback

**What:** A Tolk incoming-message union is incomplete, has duplicate opcodes, or uses a permissive `else` branch that accepts unknown or incomplete messages.

**Why it matters:** Unknown opcodes or truncated payloads can enter fallback logic, receive refunds, or mutate state without the checks applied to declared variants.

**What to look for:**
- Non-exhaustive `match` on a message union
- `else` branches that silently return, cashback, or continue into shared state logic
- Duplicate opcode prefixes or incomplete bodies accepted through lazy parsing

---

## V123 — Disabled End-of-Read Assertion

**What:** `assertEndAfterReading` or an equivalent full-consumption check is disabled without a protocol reason.

**Why it matters:** Trailing bits or refs create ambiguous messages and may smuggle fields past signatures, hashes, union dispatch, or wrapper expectations.

**What to look for:**
- `assertEndAfterReading: false` or permissive serializers
- Parsed slices with remaining bits/refs not treated as an explicit remainder payload
- Signed/hashed prefixes whose trailing attacker data is processed elsewhere

---

## V124 — Unsafe Tolk Address Type

**What:** `any_address`, `null`, or a default address is accepted where a valid internal `address` is required.

**Why it matters:** External/none addresses can bypass identity assumptions, break sends, or accidentally match an uninitialized privileged state.

**What to look for:**
- `any_address` in admin, owner, wallet, recipient, or sender-derived fields
- Force-unwrapped nullable addresses without validation
- Zero/default/null address used as an active privileged identity or initialization sentinel

---

## V125 — Unsafe Tolk Cast or Raw-Type Escape

**What:** Untrusted `slice`, `cell`, `builder`, or `unknown` data is cast with `as` or otherwise treated as a typed value without structural proof.

**Why it matters:** The cast bypasses type safety and hides malformed tags, widths, refs, addresses, or payload layouts until after security decisions.

**What to look for:**
- `as` casts from attacker-controlled raw types
- `unknown`, `builder`, `slice`, or `cell` passed into privileged or financial logic
- Nullable force unwrap (`!`) without a dominating null check

---

## V126 — Typed Cell / TL-B Layout Mismatch

**What:** `Cell<T>` or compiler-managed serialization does not match the actual TL-B layout expected by another producer, consumer, or deployed contract.

**Why it matters:** Typed source can still encode the wrong tag, field order, width, inline/ref choice, or optional layout, corrupting values across contract boundaries.

**What to look for:**
- `Cell<T>` used for opaque or differently encoded data
- Struct fields that do not match standard or legacy layouts
- Inline-versus-ref assumptions that differ between compiler-generated and manual parsers

---

## V127 — Tolk Sized-Integer Writeback Overflow

**What:** Arithmetic is performed in a wider integer domain and the result is written back to `uint32`, `uint64`, `coins`, or another sized field without a bound check.

**Why it matters:** Large attacker-controlled values can fail serialization, truncate assumptions, or DoS a critical state transition after earlier actions are prepared.

**What to look for:**
- Addition/multiplication before assignment to sized serialized fields
- Missing upper bounds before storage or message serialization
- Financial accumulators whose runtime range exceeds their declared encoded width

---

## V128 — Unsafe Raw Message Construction in Tolk

**What:** A contract uses manual builders or `sendRawMessage` where `createMessage` and typed serialization could enforce the intended shape.

**Why it matters:** Raw construction reintroduces header, mode, address, body, StateInit, bit-width, and ref-layout errors hidden by high-level source.

**What to look for:**
- `sendRawMessage` without a proxy/raw protocol requirement and strict validation
- Manually constructed ordinary messages
- Raw body or header fields copied directly from untrusted input

---

## V129 — Unproved `UnsafeBodyNoRef`

**What:** `UnsafeBodyNoRef` is used without proving the complete message body fits inline under every input.

**Why it matters:** Oversized attacker-controlled payloads can make serialization or action creation fail after state has changed.

**What to look for:**
- `UnsafeBodyNoRef` with dynamic strings, arrays, maps, refs, or forwarded payloads
- No exact worst-case bit/ref calculation
- State mutation or commit before constructing the unsafe body

---

## V130 — Forced `.toCell()` Changes Message Layout

**What:** `body: obj.toCell()` is used where `body: obj` should let Tolk choose a safe inline/ref representation.

**Why it matters:** Forced cell placement may break the expected wire layout, bounce truncation assumptions, or interoperability with standard parsers.

**What to look for:**
- `.toCell()` passed to `createMessage` for normally serializable typed bodies
- Receiver expecting inline fields while sender always stores a ref
- Bounce recovery assuming fields reside in the root body

---

## V131 — AutoDeployAddress / StateInit Mismatch

**What:** Deployment address derivation does not exactly match the code, data, owner, wallet code, workchain, or salt used in StateInit.

**Why it matters:** Funds or authorization may be directed to an attacker-controlled or non-existent contract instead of the intended deterministic child.

**What to look for:**
- Different code/data inputs between address computation and deployed StateInit
- Missing workchain, salt, domain, or owner binding
- Unjustified shard targeting or trusted address accepted from payload instead of recomputation

---

## V132 — Insufficient Tolk BounceMode Recovery Data

**What:** `NoBounce`, `Only256BitsOfBody`, `RichBounce`, or `RichBounceOnlyRootCell` is selected without ensuring the handler receives enough data to authenticate and restore state.

**Why it matters:** A failed stateful send can become unrecoverable, or a truncated bounce can be misattributed to the wrong operation.

**What to look for:**
- `NoBounce` on a send that debits or locks state
- Recovery identifiers located outside the returned body portion
- Rich/root-only parser assumptions that do not match the selected mode

---

## V133 — Mixed or Future-Incompatible Bounce Schemas

**What:** A bounce handler mixes legacy, root-only, and rich bounce bodies without explicit schema discrimination and bounds checks.

**Why it matters:** Reserved fields, original-body placement, phase, exit code, gas usage, or future extensions can shift parsing and restore the wrong operation.

**What to look for:**
- One parser used for multiple `BounceMode` variants
- No tag/mode/remaining-bits checks before reading rich fields
- Malformed or truncated bounce bodies silently ignored after state was debited

---

## V134 — Premature `commitContractDataAndActions()`

**What:** Tolk commits contract data/actions before all authorization, replay, bounds, and state-transition validation is complete.

**Why it matters:** Later failure can leave attacker-selected state permanently committed or make an invalid external message unreplayable only after causing damage.

**What to look for:**
- Commit before signature, seqno, valid-until, recipient, or amount validation
- Commit before dynamic parsing or action construction that may fail
- Committed state that assumes every later outbound action succeeds

---

## V135 — External Replay Through Commit/Action Ordering

**What:** External-message seqno/nonce and commit ordering allows replay or committed bad state when an outbound action fails.

**Why it matters:** Updating too late permits replay; committing too early may consume the nonce while skipping the intended action or preserve inconsistent accounting.

**What to look for:**
- `acceptExternalMessage()` before complete cheap authentication
- Seqno update not protected against action-phase failure
- Wallet order differing unsafely from parse → time → seqno → signature → accept → update → commit → actions

---

## V136 — Reserve Action Misordering

**What:** `RAWRESERVE`, `nativeReserve`, or an equivalent action uses the wrong mode or order relative to sends.

**Why it matters:** A later send can drain reserved funds, a reserve can unexpectedly starve required sends, or action failure can commit state without the expected balance protection.

**What to look for:**
- Reserve actions not inventoried with sends
- Exact/all-except/at-most modes used without insufficient-balance tests
- Reserve ordering or bounce-on-fail behavior inconsistent with later action assumptions

---

## V137 — Incorrect `+2` / `+16` Failure Model

**What:** Code treats `+2` and `+16` as independent protections or assumes `+2` suppresses every action error.

**Why it matters:** Malformed messages, invalid StateInit libraries, invalid external modes, or invalid mode combinations can still fail while state/action assumptions diverge.

**What to look for:**
- Comments or recovery logic claiming `+2 +16` always ignores and bounces
- No tests for non-ignored action errors
- Critical state committed before an action whose failure model is misunderstood

---

## V138 — Query ID Used as Authorization

**What:** A callback or bounce is trusted solely because its `query_id` matches or is globally unique.

**Why it matters:** Query IDs are correlation keys; an attacker can spoof a matching ID unless sender, operation, pending state, and amount are also verified.

**What to look for:**
- Lookup by `query_id` followed directly by credit/refund/finalization
- Missing sender/opcode/amount/pending-status checks
- Unnecessarily global uniqueness or unsafe reuse within concurrent in-flight operations

---

## V139 — Hidden Privileged Entry Point

**What:** Admin or state-changing logic is reachable through fallback, empty, bounced, deployment, upgrade, plugin, or callback paths that lack the primary handler's policy.

**Why it matters:** Auditors may verify named admin opcodes while an alternate entry point invokes the same mutation without authorization.

**What to look for:**
- Shared helpers called from both authenticated and unauthenticated handlers
- Empty/fallback/bounce branches that dispatch privileged opcodes
- Upgrade/plugin callbacks that can replace code, data, fee receivers, or accounting

---

## V140 — Stale Generated ABI or Wrapper

**What:** Tests and integrations use generated ABI, wrappers, source maps, or output files that were not rebuilt after source changes.

**Why it matters:** Tests may exercise old opcodes, layouts, getters, or code while the deployed artifact contains unreviewed behavior.

**What to look for:**
- Generated files older or inconsistent with Tolk source
- Narrow tests that skip the build/generation step
- Manual review relying on wrappers/source maps instead of emitted code and TVM effects

---

## V141 — Tooling Treated as Proof of Safety

**What:** Static analysis, symbolic execution, fuzzing, coverage, or generated wrappers are treated as complete security proof despite unsupported language or paths.

**Why it matters:** Tool blind spots can exclude bounce, action-phase, cross-contract, Tolk lazy, raw-cell, or generated-code behavior while reporting a clean result.

**What to look for:**
- No manual review of unsupported constructs or warnings
- Missing language/tool version compatibility check
- Coverage percentages used to dismiss untested failure and interleaving paths

---

## V142 — Missing Multi-Message Recovery State

**What:** A pending operation has no timeout, cancel, retry, recovery, or late-response policy.

**Why it matters:** Lost actions, bounces, or delayed messages can lock funds forever, while late or duplicated replies may finalize twice.

**What to look for:**
- Permanent `pending`/`locked` state without expiry
- Cancellation that does not invalidate late replies
- Retry paths that duplicate mint, withdrawal, claim, reward, or refund accounting

---

## V143 — Excess/Refund Correlation Failure

**What:** Excess or refund messages are credited without correlating sender, operation, query ID, pending state, and amount.

**Why it matters:** Attackers can fake deposits, cancel accounting, bypass slippage/deadline checks, or trigger double refunds with crafted callbacks.

**What to look for:**
- `excesses` (`0xd53276db`) or refund handlers trusting body fields alone
- Refund destination taken from unverified forwarded data
- Return messages that race with cancellation, bounce, retry, or another deposit

---

## V144 — High-Value Randomness Without Commit-Reveal

**What:** Validator-influenced on-chain randomness determines valuable traits, winners, allocations, prices, or rewards without commit-reveal.

**Why it matters:** Validators or message senders can influence seed, inclusion, timing, or ordering to bias the outcome.

**What to look for:**
- Randomness in external receivers or immediate mint/lottery outcomes
- No domain separation, reveal deadline, replay protection, or safe refund/cancel path
- Low-level random primitives used without required initialization/randomization

---

## V145 — Unbounded Action List or Batch Send

**What:** Attacker-controlled arrays, lists, maps, dictionaries, or message payloads determine an unbounded number of reserve/send actions.

**Why it matters:** The action-list or gas limit can be exceeded after state changes, causing denial of service, skipped sends, or partial accounting.

**What to look for:**
- Sends inside loops over user-controlled collections
- No hard batch size or pagination
- External messages accepted before expensive batch/action construction

---

## V146 — Storage Migration Not Tested on Live Layout

**What:** A code or schema upgrade is reviewed only against the new layout and not tested with the previous deployed data cell.

**Why it matters:** Field order, type, optional/ref placement, code cells, ABI, or wrapper changes can corrupt existing balances and permissions at upgrade.

**What to look for:**
- No old-state fixture or migration test
- Changed serialized structs loaded directly from existing c4 data
- Wallet/minter/trusted code cells replaced without compatibility checks

---

## V147 — Getter / Off-Chain Trust Mismatch

**What:** UIs, indexers, relayers, or oracles treat a getter or event/log as authoritative despite stale, manipulable, or implementation-specific assumptions.

**Why it matters:** Off-chain services may authorize, price, relay, or display actions based on data that is not synchronized with asynchronous on-chain state.

**What to look for:**
- Getter output assumed to be an on-chain synchronous dependency
- External outgoing logs/events used as on-chain truth
- ABI/wrapper/indexer assumptions not updated with contract changes

---

## V148 — External Message Partial Signing

**What:** A signature omits recipient, amount, opcode, query ID/seqno, validity, contract address, workchain, or protocol intent.

**Why it matters:** A valid signature can be replayed or repurposed across contracts, wallets, workchains, operations, or amounts.

**What to look for:**
- Signature over payload data without domain/contract binding
- Missing subwallet, recipient, workchain, or operation fields
- Same signed blob valid for multiple handlers or deployments

---

## V149 — Fee Constants Drift

**What:** Hardcoded gas, forward-fee, storage, deployment, or service-fee constants are not tested or recalibrated for current toolchain/network behavior.

**Why it matters:** Underestimated fees cause partial execution or contract subsidy; overestimated fees enable overcharging or stranded balances.

**What to look for:**
- Fixed fee constants with no worst-case gas snapshots
- Message chains or deployments close to the assumed minimum
- Toolchain changes without updated fee and excess/refund tests

---

## V150 — Missing Failure-Path Test Matrix

**What:** High-impact flows lack tests for failure, bounce, insufficient funds, invalid destination, malformed/oversized cells, replay, duplicate, late message, and interleaving.

**Why it matters:** Happy-path coverage misses the TON-specific states where compute succeeds but actions, downstream messages, or recovery fail.

**What to look for:**
- Unit tests cover handlers but not full multi-contract traces
- No race/replay/gas/migration/fuzz tests for lazy fields and send modes
- Audit conclusions based on coverage without concrete failure-path evidence

---
