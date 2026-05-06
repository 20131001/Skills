# Finding Validation

Every candidate finding must pass the false-positive gate before it can be scored or reported.

Score **certainty**, not severity. A catastrophic exploit with weak evidence should score lower than a modest exploit with a fully proven path.

This file supports two internal evidence states:

- **Finding candidate**: a scanner claims the path is complete enough to report.
- **Review Trail**: a scanner found a specific, source-backed risk signal, but the merge step has not proven every finding gate. Review Trails help reduce missed issues without turning uncertainty into a reportable vulnerability.

## FP Gate

Every finding must pass all three checks. If any check fails, do not score or report it as a finding.

1. **Concrete attack path exists.**
   You can trace a real path from caller -> reachable entrypoint -> vulnerable state change -> loss, lock, griefing, or broken invariant.
   Evaluate what the code actually allows, not what the deployer, admin, or integrator might choose to do off-chain.

2. **The attacker can reach the entrypoint.**
   Confirm the attacker can actually trigger the path after accounting for sender checks, access control, replay protection, init-phase restrictions, workchain checks, and any other gating logic.

3. **No existing guard already stops the exploit.**
   Check `throw_unless`, sender validation, `if`-revert branches, bounce filters, `seqno` checks, init flags, state locks, and equivalent guard logic.
   If a real guard blocks the exploit under the claimed path, drop the finding.

## Review Trail Handling

Use Review Trails for unresolved evidence, not for weaker findings.

- Keep a Review Trail only when it names a concrete file/location, reachable-looking entrypoint, case family, evidence trace, and unresolved blocker.
- Promote a Review Trail to a finding only after validating caller -> reachable entrypoint -> vulnerable state or trust edge -> concrete impact, and after checking every guard in the FP Gate.
- If two or more scanners independently flag the same unresolved path, use that convergence as a reason to re-check source, not as proof by itself.
- If a concrete guard blocks the path, reject it completely. Do not preserve source-refuted issues as Review Trails.
- If the only problem is missing comments, style, naming, tests, off-chain deployment assumptions, or a by-design privileged action, reject it instead of keeping a Review Trail.
- Do not attach confidence scores, `Fix` blocks, or finding numbers to Review Trails. They are audit work items, not vulnerabilities.

## Internal Merge Fields

These fields are for audit orchestration and deduplication only. Do not require them in the final report unless `report-formatting.md` explicitly includes a table for them.

- `ton_case_key`: `language | file | entrypoint-or-getter | case_family`.
- `case_family`: a short TON-specific root-cause label such as `sender-auth`, `parser-integrity`, `tep-abi`, `optimistic-accounting`, `bounce-recovery`, `external-replay`, `state-layout`, `tact-receiver`, or `tolk-lazy-parser`.
- `evidence_trace`: a compact source-backed trace, for example `caller -> recv_internal -> op::transfer -> save_data -> send_raw_message`.
- `unresolved_blocker`: the exact missing validation step that prevents a Review Trail from becoming a finding.

## Direct-Pattern Presumptions

The following code-level anti-patterns are not mere style issues. If caller-controlled input reaches them, including role-gated or otherwise privileged caller input, and they influence authorization, state mutation, or outbound message behavior, they usually satisfy FP gate 1 unless a real guard defeats the path. Language agents must map these root causes to their own concrete constructs. For authoritative storage parsers, caller control is indirect; keep those as weaker storage-layout integrity findings when multiple reachable state-changing paths rely on the parser and the layout is intended to be exact.

- authorization based on an address parsed from FunC `in_msg_body` or an equivalent attacker-controlled Tolk/Tact payload field instead of the trusted inbound sender context
- fixed-layout message, cell, slice, typed-struct, or storage parsers that never prove full consumption when the parsed shape is expected to be exact and its values influence authorization, state changes, or downstream messages
- optimistic accounting or liquidity mutation before a dependent ignored-error downstream crediting or deployment message
- a native-only or jetton-only entrypoint that never verifies the configured asset mode or sentinel before crediting value in that mode
- threshold-signature or verifier-set authorization that never proves a minimum number of unique valid signers, or traverses the signature container incorrectly
- adjacent same-typed state fields, typed message fields, or serialization helper arguments passed in a different order from the authoritative schema
- ignored nullable, optional, success-flag, or failure results from dictionary, storage, helper, or system operations that can change authorization, accounting, or state
- a value-bearing inbound path that accepts or credits the asset first, then swallows a later failure via `catch` / early `return` without refunding or rolling back
- a handled branch, receiver, router, or opcode case that falls through into a later default/error/conflicting branch after completing its intended action
- an owner/admin withdrawal or rescue path that uses gross balance instead of free balance after reserved or unclaimed obligations
- authoritative supply or settlement state committed before a dependent wallet-side mint, burn, or transfer consequence is confirmed, with no authoritative bounce recovery
- a caller-supplied nested internal message ref/body/cell such as `master_msg` forwarded to a wallet or peer contract without validating opcode, correlated amount, or exact schema
- a manual commit checkpoint that persists partially validated business state before later fallible parsing, reserve, or send work
- a standard-facing Tact field, Tolk typed field, or custom codec that cannot represent `addr_none`, optional tags, references, or the exact integer width/coin encoding required by the claimed ABI
- a fallback receiver that accepts unsupported value-bearing protocol messages without explicit rejection, refund, or safe no-op proof

These patterns may still fail FP gate 2 or 3 if the attacker cannot reach the entrypoint or another guard already neutralizes the bug. If the path is reachable only by a privileged caller, keep the finding and apply the privileged-caller confidence deduction instead of dropping it solely because the caller is role-gated.

For the second pattern, you do **not** need to prove a fully custom byte-level exploit beyond the leftover data itself. If the parser is fixed-layout, caller-influenced, and its decoded values participate in authorization, accounting, configuration, or outbound message construction, the unvalidated trailing data is itself a concrete integrity break and should normally be reportable. Broken schema or message-shape invariants count as a broken invariant even when the immediate consequence is malformed-state acceptance or unsafe downstream message construction rather than instant fund loss. In FunC, a pure `slice_bits(...) == N` check is not enough when refs may still remain; Tolk and Tact agents should apply the equivalent exact-layout proof for typed or custom parsers.

Do **not** apply the "Attack path is partial" deduction to this parser-integrity pattern merely because the leftover bytes or refs are not separately monetized or reinterpreted later in the same trace. If the fixed-layout parser is reachable and its decoded output influences state, authorization, configuration, or outbound message construction, the accepted malformed shape is already the completed invariant break.

For top-level storage parsers or typed storage decoders, the same root cause remains reportable at lower confidence when the helper is authoritative across reachable state-changing handlers and the layout is expected to be exact. The finding can score below the normal threshold, but do not drop it as mere style simply because the malformed shape resides in persisted storage rather than a caller message.

For the third and fourth patterns, you do not need a separate exotic exploit primitive beyond the reachable message path. If the contract saves a balance, liquidity, quota, or entitlement update before an ignored-error consequence send that is required to mirror that update, or if a mode-specific asset entrypoint accepts value without verifying the active asset configuration, that is normally a concrete broken-invariant path.

For the fifth pattern, it is enough to show that the signature gate can be satisfied without the intended `k-of-n` unique-verifier property. A broken traversal loop, missing threshold check, or duplicate-verifier acceptance is itself a concrete authorization bypass.

For the sixth pattern, it is enough to show that the call-site argument order no longer matches the authoritative helper signature and that the affected fields influence balances, fees, thresholds, addresses, or other security-relevant state. You do not need a second exotic exploit primitive beyond the reachable state mutation.

For the ignored-result pattern, it is enough to show that a failed lookup, deletion, update, helper call, or low-level operation can leave the code using a stale, absent, or unmodified value for authorization, accounting, serialization, or state mutation.

For the value-bearing swallowed-failure pattern, it is enough to show that the contract has already accepted or credited user value before the swallowed failure point, and that the failure path neither refunds the asset nor reverses the accounting. An empty or effectively no-op `catch` on a deposit path is not harmless if user value has already moved.

For the fallthrough pattern, it is enough to show that a handled branch, receiver, router case, or opcode path completes security-relevant work and can then execute a later default/error/conflicting branch that reverts, double-sends, skips validation, or corrupts state.

For the reserved-obligation withdrawal pattern, it is enough to show that the contract separately tracks reserved, unclaimed, or claimable obligations and that the withdrawal path ignores them when computing the transferable amount. A privileged caller requirement reduces confidence but does not turn the issue into a mere trust-model note when users can be locked out of already-accounted entitlements.

For the optimistic authoritative-state pattern, it is enough to show that the authoritative contract records supply, balance, liquidity, or entitlement before the dependent peer-side state transition is known to have succeeded, and that the authoritative layer either ignores the bounce or lacks another reconciliation path. You do not need a second exotic exploit primitive beyond the reachable message cascade.

For the nested-message forwarding pattern, it is enough to show that the forwarded nested cell remains caller-influenced and that the contract does not prove the forwarded opcode, amount, or layout matches the outer request it just authorized. A privileged mint or admin path reduces confidence but does not make this a style issue.

If the missing proof is only an explicit nested opcode check on a caller-influenced peer-message body, keep it as a weaker surviving finding when the contract still forwards that body into another contract or relies on its parsed fields. That subcase can score below the normal threshold, but it should not be dropped merely because the caller is privileged or because the stronger downstream exploit depends on additional conditions.

For manual `commit()` findings, it is enough to show that a reachable path commits business state or queued actions before a later operation can throw and that the committed checkpoint violates accounting, authorization, replay, or message-flow invariants after the failure.

For Tact/Tolk typed ABI-shape findings, it is enough to show the contract claims or depends on a standard/peer schema and the Tact/Tolk type, field order, optional tag, custom codec, or serialization annotation cannot encode or decode the legal on-chain shape. Closed-system custom ABIs reduce or remove reportability only when every participant is proven to use the same custom shape.

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
  Example: payload-derived sender authorization, missing exact-layout proof on a fixed-layout nested parser, and forwarding an unvalidated nested message cell are distinct root causes even if they occur in one handler.
  Example: a native-mode asset-configuration bypass and an optimistic ignored-error downstream credit desync are distinct root causes even if both occur in the same liquidity handler.
  Example: unvalidated forwarding of a caller-supplied nested `master_msg` and minter-side `total_supply` desync after wallet-side mint failure are distinct root causes even if both occur in the same `op::mint` handler.
- If two findings compound, keep both only when they are meaningfully distinct; otherwise keep the stronger root-cause finding.
- A linked-path finding is allowed only when two already validated findings form a stricter exploit chain and the combined impact is worse than either finding alone. Score it at the lower confidence of the two inputs. Do not add a linked-path finding when the interaction can be described clearly inside one existing finding.
- For findings at or above `[80]`, sanity-check the proposed fix against the original exploit path. A fix that creates an authorization bypass, replay opening, trapped-value path, storage-rent drain, unbounded gas path, bounce/excess loss, TEP ABI break, or cross-language message-schema mismatch is not acceptable.
- If you cannot explain the exploit path in concrete terms, drop the finding.
- For FC1-style `impure` findings, the concrete path is: caller reaches a handler -> handler calls a side-effecting authorization, throw, send, or state-update helper whose result is unused -> helper lacks `impure` and can be optimized away -> protected logic executes or required state/action is skipped.
- For FC2-style method-syntax findings, the concrete path is: caller reaches a handler -> code expects a `.` / `~` helper call to mutate or preserve a value -> the first argument is not updated or is unexpectedly overwritten -> later authorization, parsing, accounting, or serialization uses the wrong value.
- For FC4-style shadowing findings, the concrete path is: caller reaches a handler -> local variable, tuple component, or helper parameter shadows an authoritative storage field -> validation or `save_data()` uses the wrong value -> authorization, ownership, balance, config, or phase state is corrupted.
- For FC5/TL2/TA2-style ignored-result findings, the concrete path is: caller reaches a handler -> dictionary/storage/helper/system operation fails or returns null/none/false -> code ignores that result -> stale, absent, or unmodified data feeds authorization, accounting, state, or outbound messages.
- For TP-style TEP standard findings, the concrete path is: caller, wallet, marketplace, indexer, or peer contract interacts through an advertised or de facto TEP interface -> implementation violates the TEP's opcode, getter, sender, funding, reply, optional-field, or serialization requirement -> value, ownership, attribution, discovery, metadata, or trace identity becomes wrong, locked, spoofable, or unrecoverable.
- For FC6/TL3/TA3-style parser-integrity findings, the concrete path is: caller reaches a fixed-layout parser or typed decoder -> contract accepts malformed or trailing data without proving full consumption -> decoded values still influence authorization, state, configuration, or outbound message behavior. Do not require a second downstream exploit gadget before reporting it.
- For storage-variant FC6/TL3/TA3 findings, the concrete path is: caller reaches a state-changing handler -> handler relies on an authoritative fixed-layout storage parser or typed storage decoder -> parser silently accepts extra serialized trailing data or refs without proving exact layout. This is weaker than a caller-message parser break, but still reportable as a parser-integrity / storage-layout risk.
- For FC8/TL5/TA5-style fallthrough findings, the concrete path is: caller reaches a handled branch, receiver, router case, or opcode path -> intended work completes -> control continues into a later default/error/conflicting branch -> state, sends, replay protection, or availability is broken.
- For TL1/TA1/FC3-style signed-domain findings, the concrete path is: caller controls a signed or misdecoded value used as an unsigned balance, share, quota, voting power, timestamp, or amount -> negative or out-of-range value passes validation -> accounting, authorization, or serialization uses the invalid domain.
- For TL4/TA4/TA6/TA7-style typed ABI findings, the concrete path is: caller or peer sends a legal standard/claimed message shape -> typed field order, optional address/tag, reference layout, integer width, coin encoding, or custom codec cannot represent it exactly -> decode, routing, accounting, or reply behavior fails with concrete consequence.
- For TC54-style commit findings, the concrete path is: caller reaches a handler -> handler commits business state/actions -> later fallible work fails -> committed intermediate state violates an invariant.
- For TC58-style parent-child findings, the concrete path is: caller reaches a parent/child/factory-managed handler -> code trusts payload identity or an unverified address for the peer -> `sender` is not checked against a derived or stored expected peer -> spoofed peer message mutates state, releases value, or finalizes a flow.
- For TA10-style mutable-helper findings, the concrete path is: caller reaches a handler -> code expects a helper to persist state -> the helper does not update `self` or its mutation result is ignored -> later authorization, accounting, send, or replay logic assumes state changed and a concrete invariant breaks.
- For TA11-style external-context findings, do not report direct compiler errors alone. Report only when an external path reuses internal-message trust assumptions, accepts gas before validation, or otherwise creates a concrete unauthorized action, replay, balance-drain, or denial-of-service path.
- For TA12-style bounce-recovery findings, the concrete path is: caller reaches a handler -> contract commits state/value ownership -> sends a bounceable message that may reject -> no `bounced()` handler, trait recovery, or other reconciliation restores the invariant.
- For TA13-style unbounded-state findings, the concrete path is: attacker can create or grow maps, arrays, or per-user collections repeatedly -> no hard cap, pruning, or funding discipline exists -> gas, storage, or cell-size limits block normal operation or drain balance.
- For TA14-style bulk-state findings, the concrete path is: caller supplies a typed map, array-like dictionary, or bulk config object -> handler assigns or merges it into persistent state without per-entry validation and bounds -> attacker changes unauthorized keys, bloats state, or bypasses accounting/eligibility checks.
- For TA15-style trait findings, the concrete path is: inherited receiver, trait state, or overridden hook is reachable -> intended ownership, pause, phase, or value guard is missing or weaker than the concrete contract assumes -> unauthorized or stopped-state action becomes possible.
- For TA16-style parent-child findings, the concrete path is: parent or child receives a typed callback -> code trusts body fields such as parent, owner, child index, or sequence -> `sender()` is not checked against the derived/stored peer -> spoofed callback moves state or value.
- For TA17-style lazy-deployment findings, the concrete path is: handler builds or accepts child `StateInit` / destination data -> `to`, `code`, and `data` do not match or deployment can fail after parent mutation -> funds, child ownership, or parent accounting become incorrect.
- For FC9-style bitwise-negation findings, the concrete path is: caller reaches a handler -> guard uses `~flag` / `~status` as if it were logical negation -> TVM truthiness makes the branch reachable for unintended non-`-1` values -> authorization, pause, success, or phase logic is bypassed.

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
