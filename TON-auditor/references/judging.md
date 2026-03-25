# Finding Validation

Every candidate finding must pass the false-positive gate before it can be scored or reported.

Score **certainty**, not severity. A catastrophic exploit with weak evidence should score lower than a modest exploit with a fully proven path.

## FP Gate

Every finding must pass all three checks. If any check fails, drop the finding immediately. Do not score it. Do not report it.

1. **Concrete attack path exists.**
   You can trace a real path from caller -> reachable entrypoint -> vulnerable state change -> loss, lock, griefing, or broken invariant.
   Evaluate what the code actually allows, not what the deployer, admin, or integrator might choose to do off-chain.

2. **The attacker can reach the entrypoint.**
   Confirm the attacker can actually trigger the path after accounting for sender checks, access control, replay protection, init-phase restrictions, workchain checks, and any other gating logic.

3. **No existing guard already stops the exploit.**
   Check `throw_unless`, sender validation, `if`-revert branches, bounce filters, `seqno` checks, init flags, state locks, and equivalent guard logic.
   If a real guard blocks the exploit under the claimed path, drop the finding.

## Direct-Pattern Presumptions

The following exact code-level anti-patterns are not mere style issues. If caller-controlled input reaches them, including role-gated or otherwise privileged caller input, and they influence authorization, state mutation, or outbound message behavior, they usually satisfy FP gate 1 unless a real guard defeats the path. For authoritative storage parsers such as top-level `load_data()` helpers built from `get_data().begin_parse()`, caller control is indirect; keep those as weaker storage-layout integrity findings when multiple reachable state-changing paths rely on the parser and the layout is intended to be exact.

- authorization based on an address parsed from `in_msg_body` or another attacker-controlled payload field instead of the trusted inbound sender context
- fixed-layout slice parsers that never prove full consumption when the parsed cell or sub-slice is expected to be exact and its values influence authorization, state changes, or downstream messages
- optimistic accounting or liquidity mutation before a dependent ignored-error downstream crediting or deployment message
- a native-only or jetton-only entrypoint that never verifies the configured asset mode or sentinel before crediting value in that mode
- threshold-signature or verifier-set authorization that never proves a minimum number of unique valid signers, or traverses the signature container incorrectly
- adjacent same-typed state or serialization helper arguments passed in a different order from the helper signature
- a value-bearing inbound path that accepts or credits the asset first, then swallows a later failure via `catch` / early `return` without refunding or rolling back
- an owner/admin withdrawal or rescue path that uses gross balance instead of free balance after reserved or unclaimed obligations
- authoritative supply or settlement state committed before a dependent wallet-side mint, burn, or transfer consequence is confirmed, with no authoritative bounce recovery
- a caller-supplied nested internal message ref such as `master_msg` forwarded to a wallet or peer contract without validating opcode, correlated amount, or exact schema

These patterns may still fail FP gate 2 or 3 if the attacker cannot reach the entrypoint or another guard already neutralizes the bug. If the path is reachable only by a privileged caller, keep the finding and apply the privileged-caller confidence deduction instead of dropping it solely because the caller is role-gated.

For the second pattern, you do **not** need to prove a fully custom byte-level exploit beyond the leftover data itself. If the parser is fixed-layout, caller-influenced, and its decoded values participate in authorization, accounting, configuration, or outbound message construction, the unvalidated trailing data is itself a concrete integrity break and should normally be reportable. Broken schema or message-shape invariants count as a broken invariant even when the immediate consequence is malformed-state acceptance or unsafe downstream message construction rather than instant fund loss. A pure `slice_bits(...) == N` check is not enough when refs may still remain.

Do **not** apply the "Attack path is partial" deduction to this parser-integrity pattern merely because the leftover bytes or refs are not separately monetized or reinterpreted later in the same trace. If the fixed-layout parser is reachable and its decoded output influences state, authorization, configuration, or outbound message construction, the accepted malformed shape is already the completed invariant break.

For top-level storage parsers such as `load_data()` from `get_data().begin_parse()`, the same root cause remains reportable at lower confidence when the helper is authoritative across reachable state-changing handlers and the layout is expected to be exact. The finding can score below the normal threshold, but do not drop it as mere style simply because the malformed shape resides in persisted storage rather than a caller message.

For the third and fourth patterns, you do not need a separate exotic exploit primitive beyond the reachable message path. If the contract saves a balance, liquidity, quota, or entitlement update before an ignored-error consequence send that is required to mirror that update, or if a mode-specific asset entrypoint accepts value without verifying the active asset configuration, that is normally a concrete broken-invariant path.

For the fifth pattern, it is enough to show that the signature gate can be satisfied without the intended `k-of-n` unique-verifier property. A broken traversal loop, missing threshold check, or duplicate-verifier acceptance is itself a concrete authorization bypass.

For the sixth pattern, it is enough to show that the call-site argument order no longer matches the authoritative helper signature and that the affected fields influence balances, fees, thresholds, addresses, or other security-relevant state. You do not need a second exotic exploit primitive beyond the reachable state mutation.

For the seventh pattern, it is enough to show that the contract has already accepted or credited user value before the swallowed failure point, and that the failure path neither refunds the asset nor reverses the accounting. An empty or effectively no-op `catch` on a deposit path is not harmless if user value has already moved.

For the eighth pattern, it is enough to show that the contract separately tracks reserved, unclaimed, or claimable obligations and that the withdrawal path ignores them when computing the transferable amount. A privileged caller requirement reduces confidence but does not turn the issue into a mere trust-model note when users can be locked out of already-accounted entitlements.

For the ninth pattern, it is enough to show that the authoritative contract records supply, balance, liquidity, or entitlement before the dependent peer-side state transition is known to have succeeded, and that the authoritative layer either ignores the bounce or lacks another reconciliation path. You do not need a second exotic exploit primitive beyond the reachable message cascade.

For the tenth pattern, it is enough to show that the forwarded nested cell remains caller-influenced and that the contract does not prove the forwarded opcode, amount, or layout matches the outer request it just authorized. A privileged mint or admin path reduces confidence but does not make this a style issue.

If the missing proof is only an explicit nested opcode check on a caller-influenced peer-message body, keep it as a weaker surviving finding when the contract still forwards that body into another contract or relies on its parsed fields. That subcase can score below the normal threshold, but it should not be dropped merely because the caller is privileged or because the stronger downstream exploit depends on additional conditions.

## Confidence Score

Confidence measures how certain you are that the finding is real and exploitable.

- Start every surviving finding at **100**.
- Apply **all** deductions that fit.
- Confidence is independent of severity.
- Report the final score as `[N]`.

Examples: `[95]`, `[75]`, `[60]`.

### Deductions

- Privileged caller required (owner, admin, multisig, governance) -> **-25**
- Attack path is partial (the idea is plausible, but you cannot fully write caller -> call -> state change -> outcome) -> **-20**
- Impact is self-contained (only the attacker's own funds or position are affected, with no broader user or protocol spillover) -> **-15**

## Threshold

Default confidence threshold: **75**

- Findings at or above the threshold get a normal finding entry with a `Fix` section.
- Findings below the threshold may still be listed, but they do **not** get a `Fix` section.

## Reporting Rules

- Report the **root cause**, not every downstream symptom.
- Do not report the same root cause twice under slightly different titles.
- Do not merge distinct code-level root causes merely because they appear in the same function, parse the same message, or participate in the same business flow.
- Preserve separate findings when the exploit mechanism or fix is different.
  Example: payload-derived sender authorization, missing `end_parse()` on a fixed-layout nested parser, and forwarding an unvalidated nested message cell are distinct root causes even if they occur in one handler.
  Example: a native-mode asset-configuration bypass and an optimistic ignored-error downstream credit desync are distinct root causes even if both occur in the same liquidity handler.
  Example: unvalidated forwarding of a caller-supplied nested `master_msg` and minter-side `total_supply` desync after wallet-side mint failure are distinct root causes even if both occur in the same `op::mint` handler.
- If two findings compound, keep both only when they are meaningfully distinct; otherwise keep the stronger root-cause version.
- If you cannot explain the exploit path in concrete terms, drop the finding.
- For V25-style parser-integrity findings, the concrete path is: caller reaches fixed-layout parser -> contract accepts malformed or trailing data without proving full consumption -> decoded values still influence authorization, state, configuration, or outbound message behavior. Do not require a second downstream exploit gadget before reporting it.
- For storage-variant V25 findings, the concrete path is: caller reaches a state-changing handler -> handler relies on an authoritative fixed-layout storage parser such as `load_data()` -> parser silently accepts extra serialized trailing data or refs without proving exact layout. This is weaker than a caller-message parser break, but still reportable as a parser-integrity / storage-layout risk.

## Do Not Report

- Anything a linter, compiler, or experienced reviewer would dismiss: style issues, naming, formatting, NatSpec, redundant comments, or minor code hygiene.
- Gas micro-optimizations or efficiency notes with no concrete exploit path.
- Missing logs, events, or observability-only concerns.
- Pure centralization or trust observations with no distinct exploit path.
  Example: "owner could rug" is not a vulnerability unless the code exposes a specific unsafe mechanism beyond the intended trust model.
- By-design privileged actions such as owner/admin ability to set parameters, fees, pause state, or upgrade configuration.
- Theoretical issues that require implausible assumptions.
  Examples: compromised compiler, malicious validator majority, corrupted toolchain, or attacker control over unrealistic external preconditions.
- Issues that depend on integrations behaving incorrectly when the audited contract itself enforces the relevant safety property.
- Duplicate findings that restate the same bug from another angle without a distinct exploit path.
