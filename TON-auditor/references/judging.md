# Finding Validation

Use this file as the validation contract for whether a candidate becomes a reportable finding.

## Inputs

- a candidate root-cause finding
- the reachable execution map for the claimed path
- any relevant semantic backing from `standards-and-lessons.md` or `security-best-practices.md`

## Goal

Keep only attacker-reachable root-cause findings that survive the FP gate, then score certainty, not severity.

## Output

- drop the candidate, or
- keep one deduplicated finding with a confidence score and threshold-aware reporting behavior

## FP Gate

Every finding must pass all three checks. If any check fails, drop it immediately. Do not score it. Do not report it.

### 1. Concrete Attack Path Exists

You must be able to trace:

`caller -> reachable entrypoint -> vulnerable state change or forged trust edge -> loss / lock / grief / broken invariant`

Rules:

- Evaluate what the code actually allows, not what the deployer, admin, or integrator might choose to do off-chain.
- For logic and async-cascade findings, concrete means following the real reachable chain through local helpers and any in-scope consequence or bounce handlers that matter to the exploit.
- Do not stop at an outbound send when the callee is in scope.

### 2. The Attacker Can Reach The Entrypoint

Confirm the attacker can trigger the path after accounting for:

- sender checks
- access control
- replay protection
- init-phase restrictions
- workchain checks
- equivalent gating logic

### 3. No Existing Guard Already Stops The Exploit

Check:

- `throw_unless`
- sender validation
- `if`-revert branches
- bounce filters
- `seqno` checks
- init flags
- state locks
- equivalent guard logic

If a real guard blocks the exploit under the claimed path, drop the finding.

## Direct-Pattern Presumptions

The following exact code-level anti-patterns are not mere style issues. If they are reachable and influence authorization, state mutation, or outbound message behavior, they usually satisfy FP gate 1 unless a real guard defeats the path.

### P1. Payload-Derived Sender Authorization

- Authorization based on an address parsed from `in_msg_body` or another attacker-controlled payload field instead of the trusted inbound sender context.

### P2. Fixed-Layout Parser Without Full-Consumption Proof

- A fixed-layout slice parser never proves full consumption even though the decoded data influences authorization, state, configuration, outbound messages, or authoritative storage decoding.
- This includes:
  - nested `begin_parse()` parsers
  - reused payload slices
  - top-level storage loaders such as `load_data()` from `get_data().begin_parse()`

Rules:

- You do **not** need a second custom exploit beyond the leftover data itself.
- Broken schema or message-shape integrity is already a broken invariant when the decoded values affect security-relevant behavior.
- `slice_bits(...) == N` is not enough when refs may still remain.
- Do **not** apply the `Attack path is partial` deduction merely because leftover bytes or refs are not separately monetized later.
- For authoritative storage parsers such as top-level `load_data()` helpers, caller control is indirect. Keep these as weaker storage-layout integrity findings when multiple reachable state-changing paths rely on the parser and the layout is intended exact.
- If a reachable fixed-layout parser lacks `end_parse()` or an equivalent emptiness proof and there is no explicit extension design, default to reporting the issue; do not require a second exploit gadget, later branch confusion proof, or separate fund-loss story before it survives.

### P3. Optimistic Accounting Before Ignored-Error Consequence Send

- Balance, liquidity, quota, or entitlement is updated before a dependent ignored-error downstream crediting or deployment message.

### P4. Missing Asset-Mode Or Sentinel Verification

- A native-only or jetton-only entrypoint never verifies the configured asset mode or sentinel before crediting value in that mode.

### P5. Broken Threshold Signature-Set Validation

- A verifier-set or multisig authorization flow never proves a minimum number of unique valid signers, or traverses the signature container incorrectly.

### P6. Misordered Same-Typed Helper Arguments

- Adjacent same-typed state or serialization helper arguments are passed in a different order from the helper signature.

### P7. Accepted Value Followed By Swallowed Failure

- A value-bearing inbound path accepts or credits the asset first, then swallows a later failure via `catch` or early `return` without refunding or rolling back.

### P8. Gross-Balance Withdrawal Ignores Reserved Obligations

- An owner/admin withdrawal or rescue path uses gross balance instead of free balance after reserved or unclaimed obligations.

### P9. Authoritative Accounting Before Peer-Side Confirmation

- Supply, balance, liquidity, or entitlement is committed before a dependent wallet-side mint, burn, or transfer consequence is confirmed, with no authoritative bounce recovery.

### P10. Forwarded Nested Message Body Not Fully Validated

- A caller-supplied nested internal message ref such as `master_msg` is forwarded to a wallet or peer contract without validating opcode, correlated amount, or exact schema.

Rules:

- A privileged mint or admin path reduces confidence but does not make this a style issue.
- If the missing proof is only an explicit nested opcode check on a caller-influenced peer-message body, keep it as a weaker surviving finding when the contract still forwards or relies on that body.

## Pattern-Specific Notes

- **P3/P4**: You do not need a second exotic exploit primitive beyond the reachable message path.
- **P5**: It is enough to show that the signature gate can be satisfied without the intended `k-of-n` unique-verifier property.
- **P6**: It is enough to show that the call-site order no longer matches the authoritative helper signature and that the affected fields influence security-relevant state.
- **P7**: It is enough to show that the contract already accepted or credited user value before the swallowed failure point, and that the failure path neither refunds nor reverses the accounting.
- **P8**: A privileged caller requirement reduces confidence but does not turn the issue into a mere trust-model note when users can be locked out of already-accounted entitlements.
- **P9**: It is enough to show that the authoritative contract records state before the peer-side transition is known to have succeeded and lacks reconciliation.
- **P10**: The forwarded nested cell must remain caller-influenced and unproven against the intended outer request.

## Confidence Score

Confidence measures how certain you are that the finding is real and exploitable.

- Start every surviving finding at **100**.
- Apply **all** deductions that fit.
- Confidence is independent of severity.
- Report the final score as `[N]`.

Examples: `[95]`, `[75]`, `[60]`

### Deductions

- Privileged caller required (owner, admin, multisig, governance) -> **-25**
- Attack path is partial (the idea is plausible, but you cannot fully write caller -> call -> state change -> outcome) -> **-20**
- Impact is self-contained (only the attacker's own funds or position are affected, with no broader user or protocol spillover) -> **-15**

## Threshold

Default confidence threshold: **75**

- Findings at or above the threshold get a normal finding entry with a `Fix` section.
- Findings below the threshold must still be listed when they are confirmed, but they do **not** get a `Fix` section.
- The threshold affects presentation and recommended remediation detail, not whether a confirmed finding exists.
- Do not drop a confirmed finding solely because it is below the threshold.

## Reporting Rules

- Report the **root cause**, not every downstream symptom.
- Do not report the same root cause twice under slightly different titles.
- Do not merge distinct code-level root causes merely because they appear in the same function, parse the same message, or participate in the same business flow.
- Preserve separate findings when the exploit mechanism or local fix is different.
- If two findings compound, keep both only when they are meaningfully distinct; otherwise keep the stronger root-cause version.
- If you cannot explain the exploit path in concrete terms, drop the finding.

Examples of distinct root causes:

- payload-derived sender authorization, missing `end_parse()` on a fixed-layout nested parser, and forwarding an unvalidated nested message cell are distinct even if they occur in one handler
- a native-mode asset-configuration bypass and an optimistic ignored-error downstream credit desync are distinct even if both occur in one liquidity handler
- unvalidated forwarding of a caller-supplied nested `master_msg` and minter-side `total_supply` desync after wallet-side mint failure are distinct even if both occur in one `op::mint` handler

For V25-style parser-integrity findings, the concrete path is:

- caller reaches fixed-layout parser
- contract accepts malformed or trailing data without proving full consumption
- decoded values still influence authorization, state, configuration, outbound message behavior, or authoritative storage decoding

Do not require a second downstream exploit gadget before reporting that pattern.
Do not treat that path as partial merely because the residual bytes or refs are not later read explicitly.

For storage-variant V25 findings, the concrete path is:

- caller reaches a state-changing handler
- handler relies on an authoritative fixed-layout storage parser such as `load_data()`
- parser silently accepts extra serialized trailing data or refs without proving exact layout

This is weaker than a caller-message parser break, but still reportable as a parser-integrity / storage-layout risk.
Once that path is shown, the finding should normally be kept rather than merged away into a broader logic or accounting issue.

## Do Not Report

- style issues, naming, formatting, NatSpec, redundant comments, or minor code hygiene
- gas micro-optimizations with no concrete exploit path
- missing logs, events, or observability-only concerns
- pure centralization or trust observations with no distinct exploit path
- by-design privileged actions such as owner/admin ability to set parameters, fees, pause state, or upgrade configuration
- theoretical issues that require implausible assumptions
- issues that depend on integrations behaving incorrectly when the audited contract itself enforces the relevant safety property
- duplicate findings that restate the same bug from another angle without a distinct exploit path
