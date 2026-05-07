# Shared TON Audit Rules

Apply these rules to every TON audit agent bundle regardless of source language.

## Reading

- Treat the bundle as the source of truth for scope, references, and agent instructions.
- Read the bundled source code before classifying findings.
- Apply language-specific rules only to the target-language files named in the bundle.
- Use other in-scope language files as TON cross-contract message-flow context.
- The assigned vector file is either a shared TON/TVM file from `attack-vectors/shared/`, a TEP standard file from `attack-vectors/tep/`, or the target language file from `attack-vectors/languages/`.
- Never classify vectors from another language's `attack-vectors/languages/*.md` file; those are intentionally excluded from the bundle.
- In vector-agent examples, cross-file IDs are aliases only. Classify and report only IDs that exist in the assigned vector file; if the same root cause is real but has no assigned ID, leave it for the bundle that owns that ID or for the adversarial agent.
- Interpret attack-vector ID prefixes by file: `TC` = TON-chain shared, `TP` = TEP standards, `FC` = FunC, `TL` = Tolk, `TA` = Tact. Each file starts numbering from 1.
- Do not inspect files outside the declared scope unless the parent prompt explicitly allows it.
- Use `judging.md` for false-positive gating and confidence scoring.
- Use `report-formatting.md` for finding block shape.
- Use `standards-and-lessons.md` when it is bundled and code depends on TON standards, message schemas, getter ABIs, bounce behavior, or gas semantics.
- Use `security-best-practices.md` when it is bundled and code depends on message-flow design, external-message safety, callbacks, randomness, upgrades, serialization, reserve discipline, or safe value return.
- Use `source-links.md` only when it is bundled, and only as a maintenance/source map. Do not browse external links during a normal audit unless the user explicitly asks to refresh or verify upstream material.

## Vector Agents

- Search aggressively, but confirm conservatively. Do not accept `No findings.` until every assigned vector has been classified. Do not invent issues that fail the FP gate.
- Audit only the bundle content you were given, and use only the assigned vector set in that bundle.
- Final response only; do not write files or emit interim findings.
- Consider alternate manifestations of a vector, not just literal syntax matches.
- Treat vector titles as generalized root-cause templates. A Jetton, NFT, multisig, sale, bridge, or wallet-specific manifestation still counts when the same underlying bug exists.
- Treat legacy concrete patterns as preserved aliases of generalized vectors. Do not drop a match only because the old concrete title no longer exists verbatim in the library.
- Do not scan for unrelated vulnerabilities outside the assigned vector set. The adversarial agent handles free-form exploration.
- Read the bundle in parallel 1000-line chunks on the first turn. The line count is in the prompt; compute offsets, issue all reads at once, and do not read without a limit.
- Output exactly these sections, in order: `Triage`, `Deep Pass`, `Findings`, `Review Trails`.
- Classify every vector in the assigned attack-vector file as exactly one of:
  - `Skip`: both the named construct and generalized exploit concept are absent in target-language scope.
  - `Borderline`: a weak analogy exists, but there is no exact state-changing path and exploit sentence.
  - `Surviving`: the literal construct exists, or a target-language construct clearly instantiates the same root cause.
- Output `Triage` as `Skip: ...`, `Borderline: ...`, `Surviving: ...`, and `Total: N classified`; verify `N` matches the number of vectors in the assigned reference.
- For each surviving vector, trace caller -> target-language entrypoint -> vulnerable state change or trust edge -> impact, then apply `judging.md` immediately.
- In `Deep Pass`, use one-line verdicts in this shape: `{ID}: path: ... | guard: ... | evidence: ... | ton_case_key: ... | verdict: CONFIRM [score]` or `REJECT (...)`. Use `REVIEW_TRAIL` only when a concrete location and risk signal exist but the complete impact path is still unresolved.
- In `Findings`, output only formatted finding blocks sorted by confidence, or exactly `No findings.` Do not output a full report wrapper.
- In `Review Trails`, include only unresolved source-backed trails using `RT-1: location: ... | case_family: ... | ton_case_key: ... | evidence_trace: ... | unresolved_blocker: ...`, or exactly `None.`
- After the deep pass, stop. Do not revisit eliminated vectors, scan outside the assigned vector set, or pad the output with speculative issues.

## Adversarial Agents

- The parent prompt gives one bundle file path and its line count. Read the bundle fully and use bundled judging, formatting, shared rules, attack-vector references, and optional references.
- Audit the target-language files as primary scope. Use other in-scope languages only as TON cross-contract message-flow context.
- Map attacker-reachable entrypoints, receivers, bounced handlers, routers, getters with security impact, and admin paths reachable through attacker-controlled contracts or relays before proposing findings.
- Build concrete paths in this shape: caller -> reachable entrypoint -> vulnerable state mutation or forged trust edge -> loss / lock / griefing / spoofing / broken invariant.
- You may use any TON-specific reasoning path, including language-specific parser, storage, serialization, receiver, trait, optional, lazy-loading, and control-flow bugs.
- Do not report style issues, centralization-by-design, or non-exploitable compatibility nits.
- Apply the FP gate from `judging.md` immediately. If you cannot trace a concrete attacker path, drop the issue.
- Deduplicate by root cause, preserving distinct parser-integrity, fallback-acceptance, raw nested-message forwarding, standards-mismatch, and optimistic accounting/supply-desync findings when their fixes differ.
- Put high-signal risks with a concrete source location but incomplete impact path in `Review Trails`; do not include source-refuted paths.
- Return exactly two sections: `Findings`, then `Review Trails`.
- Under `Findings`, return only formatted finding blocks per `report-formatting.md`, or exactly `No findings.`
- Under `Review Trails`, return unresolved source-backed trails as `RT-1: location: ... | case_family: ... | ton_case_key: ... | evidence_trace: ... | unresolved_blocker: ...`, or exactly `None.`
- Do not output a report wrapper, analysis notes, triage, or chain-of-thought.
- Sort findings by confidence, use placeholder sequential numbering, keep `Description` to one short sentence, and include `Fix` blocks only when confidence is at or above the threshold from `judging.md`.

## Coverage Agents

- The parent prompt assigns a coverage bundle and may assign a coverage family such as surface/auth, lifecycle/finality, parser/standards, accounting/math, or storage/gas.
- Coverage is a completeness pass, not a summary pass. Emit one `COV:` line for every checklist item in the assigned bundle.
- Emit one `COV-PROBE:` line for every semantic probe in the assigned bundle, and one `COV-SUMMARY:` line for the assigned family.
- Treat every nontrivial receiver, entrypoint, parser, helper, getter, storage map, send, bounce path, and business formula as independently auditable even when a broader invariant finding already exists.
- For lifecycle items, trace create -> local action failure -> remote bounce -> success callback -> excess/callback -> timeout/no-response -> stale/unrelated message -> cleanup. Missing one terminal path is a concrete signal.
- For lifecycle items, also test duplicate query ids and cross-flow query-id reuse. A stale pending sale, claim, unstake, mint, burn, tax, or withdrawal record can be security-relevant when a later unrelated bounce or callback can match it.
- For parser and standards items, check exact layout, trailing bits/refs, optional tags, address forms, integer widths, opcode tags, getter tuple order, nested-cell schemas, Merkle/proof helpers, empty proof, single-leaf proof, two-leaf proof, and producer/consumer agreement across languages.
- For accounting and math items, test zero, one unit, equality at caps, cap plus one, non-divisible duration, final tranche, fee/tax active and inactive windows, failed split legs, quorum/threshold equality, denominator boundaries, decimal-scale conversion, and gross-booked vs net-received amounts for every protocol payout.
- Mark checklist items as `finding` only when a complete reportable path exists. Mark them as `review_trail` when source-backed risk remains but one FP-gate element is unresolved. Mark them as `audited` only when the relevant guards or harmlessness proof are named in the note.
- Do not let a global optimistic-accounting finding hide a distinct post-credit rejection, local ignored-send finality, remote bounce compensation gap, stale pending cleanup, mutable rollback, query-id correlation, cross-flow stale pending collision, helper edge case, tax/net-amount mismatch, protocol payout underpayment, or math/rounding issue.
- Do not downgrade a probe to `review_trail` because off-chain prose is unavailable when the on-chain code defines the relevant invariant. Stored fields and state names such as vesting end/duration, snapshot root, total voting power, reward pool, pending state, and supply/accounting counters are sufficient on-chain evidence when the path is reachable and not source-refuted.
- Treat these as direct finding signals unless a concrete source guard refutes them: full vesting entitlement claimable before the stored end time, accepted single-leaf-compatible Merkle roots whose empty proof cannot be verified, protocol payout state booked at gross while taxed/fee-on-transfer delivery gives the recipient net, and stale pending entries that a later unrelated bounce/callback can match.

## TON Chain Model

- Model every non-trivial workflow as an asynchronous message cascade.
- For each broad invariant issue, also inspect the receiver/helper-level edge that creates it. A global "optimistic accounting" finding does not cover a distinct late-rejection, stale-pending, tax/net-amount, or bounce-correlation bug when the fix must be applied in a different local path.
- Partial execution is normal on TON; never assume a later contract action reverts an earlier committed state update.
- Treat the inbound message envelope sender and value as the trusted transport context. Treat fields decoded from message bodies, nested cells, callbacks, and forwarded payloads as attacker-controlled until validated.
- Authenticate callbacks and peer messages by both expected sender and outstanding-request correlation.
- In parent-child or factory-deployed contract groups, authenticate peers by recomputing the expected child/master/wallet address from trusted state, code, and index, or by checking stored peer state; do not trust body fields that name a parent or child.
- Verify sender-wallet authenticity by deriving the expected wallet address from the trusted master and owner when handling Jetton or wallet-like callbacks.
- Re-check or reserve every property that can change across messages. A balance, phase, entitlement, or eligibility check at the first stage does not prove the property still holds in a later stage.
- Validate enough value for the whole cascade before committing authoritative supply, balance, liquidity, quota, or entitlement state.
- Return or refund unsupported value-bearing messages instead of trapping assets through late throws, silent returns, or swallowed errors.
- Reserve storage balance before optional sends or branches that could subsidize callers from contract balance.
- Distinguish local action-phase send failure from remote bounce. Ignored local send failure may produce no outbound message and no bounce, so a bounce handler alone does not prove recovery.
- Trace pending-operation state through create, success, bounce/failure, timeout/no-response, and unrelated/stale-message paths. A pending map is safe only when every terminal path consumes or invalidates the exact entry it created.
- For taxed, fee-on-transfer, burn-split, or redistribution assets, reason about the net amount received by the beneficiary and every split leg. A payout path can be finality-safe but still underpay if the sender books gross amounts while the wallet delivers net amounts. Enumerate every application-level payout path, not only the tax-settlement handler.
- Audit business arithmetic at boundary values: zero, one unit, non-divisible durations, final tranche, cap equality, decimal scale, ratio denominator, and rounding direction. Treat time/rounding mistakes as security-relevant when they release, block, or burn entitlements.

## Standards And Serialization

- Treat claimed Jetton, NFT, DNS, SBT, royalty, metadata, source-registry, normalized-hash, compressed-NFT, scaled-UI-Jetton, wallet, and discovery interfaces as ABI-sensitive.
- Apply TEP-derived vectors when a contract claims compatibility with a TEP, implements the relevant opcode/getter, or is consumed as that standard by wallets, marketplaces, explorers, bridges, indexers, or peer contracts.
- Check opcodes, field order, field widths, address kinds, optional tags, referenced vs inline fields, getter tuple order, and response routing exactly against the claimed TON standard.
- Require exact-layout proof for fixed-shape messages, storage, config, and nested payloads unless the design explicitly supports extensions. Each language agent defines the concrete API-level proof for its language.
- Do not accept caller-supplied nested peer messages or wallet bodies unless the contract validates opcode, correlated amount, layout, destination, value, and send-mode semantics or rebuilds the message from trusted fields.
- Preserve `query_id` or equivalent correlation identifiers across standard reply paths.
- Prefer compact binary on-chain payloads over human-readable encodings. If string/comment/metadata parsing is unavoidable, bound it and perform it before state mutation or value acceptance.

## External Messages And Bounce

- External messages are attacker-replayable unless the contract enforces signature, freshness, domain separation, and consumed replay state.
- Gas acceptance or gas-limit increases should happen only after authentication and message-shape checks.
- If a path can still fail after gas acceptance, replay-protection state should be committed before the fallible work.
- Do not use manual commit checkpoints to preserve business state before later fallible parsing, sends, or reserve work unless the post-commit failure cases are proven harmless.
- Account for costs paid from contract balance, including storage fees, external-out messages, explicit attached value, and fee-separate send modes.
- Bounced messages contain only a truncated prefix of the original body. Do not reconstruct late fields unless the contract stored correlation data elsewhere.
- A bounce path is not complete recovery unless it is reachable, funded, authenticated, and reverses the exact authoritative state mutation.

## Validation

- Confirm only concrete, attacker-reachable paths.
- Trace caller -> reachable entrypoint -> vulnerable state change or trust edge -> loss, lock, griefing, spoofing, or broken invariant.
- Check every sender guard, replay guard, phase guard, init guard, workchain check, state lock, and equivalent branch before reporting.
- Drop style, lint, naming, gas micro-optimization, observability-only, centralization-by-design, and speculative findings.
- Confidence measures certainty, not severity.
- Use `Review Trails` for specific source-backed risk signals that are not source-refuted but do not yet satisfy every finding gate. A Review Trail must name the location, case_family, evidence_trace, and unresolved blocker.
- Do not use missing off-chain prose as the unresolved blocker when the on-chain code defines the invariant and the path is reachable. Promote or refute source-backed probes for vesting end/duration math, single-leaf/empty-proof verification, gross-vs-net taxed payouts, and stale pending cross-flow rollback.
- Use `ton_case_key` only as internal merge metadata in the form `language | file | entrypoint-or-getter | case_family`. Do not put internal keys into the final report unless the requested output format has a field for them.

## Deduplication

- Deduplicate by root cause, not title wording.
- Preserve distinct findings when exploit mechanics or local fixes differ, even inside the same handler.
- Preserve late validation after asset receipt separately from outbound transfer finality. The first needs refund/rejection before or after receipt; the second needs payout confirmation, recovery, or compensation.
- Preserve stale pending-entry cleanup separately from optimistic accounting. The first needs state lifecycle cleanup/correlation; the second needs mutation ordering or reconciliation.
- Preserve tax/net-amount mismatch separately from generic transfer failure. The fix must account for fee/tax behavior or bypass taxed paths for protocol payouts.
- Preserve taxed protocol payout underpayment separately from tax redistribution delivery failure.
- Preserve schedule/math boundary bugs separately from message-cascade failures.
- Preserve helper/proof edge-case bugs separately from broader governance or authorization findings when the helper needs its own local fix.
- Preserve cross-flow stale pending collisions separately from generic pending-map storage growth.
- Preserve language-specific root causes separately when the same TON-level flow is exposed through different language entrypoints or requires different code-local fixes.
- Do not merge unvalidated nested-message forwarding into optimistic accounting or supply desync when both exist.
- Do not merge a parser-integrity issue into an auth, mint, forwarding, or standards-mismatch finding unless the exploit path and local fix are identical.
- If findings compound, keep the distinct root causes and mention the interaction once in the stronger finding when useful.

## Output

- Output only the sections requested by the parent prompt or agent instructions.
- For formatted findings, use placeholder numbering and sort by confidence descending.
- Include `Fix` blocks only for findings at or above the threshold from `judging.md`.
- If nothing survives the false-positive gate, put exactly `No findings.` in the `Findings` section.
- If no unresolved trails remain, put exactly `None.` in the `Review Trails` section when that section is requested.
