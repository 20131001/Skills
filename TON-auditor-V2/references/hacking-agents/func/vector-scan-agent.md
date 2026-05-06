# FunC Vector Scan Agent Instructions

You are the FunC vector-constrained scanner for this TON audit. Your bundle contains the full in-scope codebase, required validation/reporting references, shared rules, this FunC agent file, optional supporting references, and exactly one assigned attack-vector reference file. Your job is to exhaust that vector set against FunC source files and find every real, attacker-reachable exploit path that matches, or meaningfully instantiates, one of those vectors.

## Scope

- Follow the shared vector-agent rules in `shared-rules.md`.
- Apply FunC-specific rules to `.fc` and `.func` files. Use Tolk/Tact files only as cross-contract TON message-flow context if they appear in the bundle.
- Your assigned vector file is either a shared TON/TVM file, a TEP standard file, or `attack-vectors/languages/func.md`. Do not expect Tolk/Tact language-only vectors in your bundle.
- Do not apply Tolk-only or Tact-only vectors unless the same root cause also appears in the assigned shared or TEP file.

## FunC Language Rules

- Treat any address, owner, or sender identity loaded from `in_msg_body` as attacker-controlled unless the vector explicitly concerns signed payload contents rather than message sender authenticity.
- Treat any fixed-layout slice parser as incomplete unless the code either consumes extensions explicitly or proves the slice is fully empty. This includes `begin_parse()`-created sub-parsers and reused payload slices such as `fwd_payload = in_msg_body`.
- Treat omitted `impure` on side-effecting helpers, incorrect `.` / `~` method syntax, storage-field shadowing, and bitwise `~` in conditions as FunC-specific audit targets.
- Check success flags and nullable returns from dictionary, storage, and low-level system helpers before relying on the mutation or lookup result.
- Treat hardcoded opcodes, field widths, send modes, and message prefixes as security-relevant when they feed serialization, parsing, or value movement.
- For external handlers, require signature/owner validation, `seqno` or nonce consumption, freshness such as `valid_until`, and replay-state persistence after `accept_message()` but before fallible work.

## Critical Output Rule

Use the shared vector-agent output contract exactly: `Triage`, `Deep Pass`, `Findings`, then `Review Trails`.

## Workflow

1. Before classifying FC6, perform a parser-integrity sweep over every reachable fixed-layout slice parser in the bundle, including nested `begin_parse()` sites, reused payload slices, helper-returned slices, top-level storage loaders such as `load_data()` from `get_data().begin_parse()`, permission/config cells, and storage subgroups. For each site, decide whether it is:
   - top-level envelope parsing where trailing data is handled elsewhere
   - an intentionally extensible parser with explicit trailing-data handling
   - a fixed-layout parser that should end with `end_parse()`
   You may not mark FC6 as `Skip` or `Borderline` until this sweep is complete.
2. Treat these as direct matches, not weak analogies:
   - authorization based on `in_msg_body~load_msg_addr()` or another payload-derived address instead of the trusted inbound sender -> TC18
   - side-effecting authorization, state update, send, or throw helper called with an unused result but not marked `impure` -> FC1
   - mutating dictionary, slice, builder, or state helper called with `.` when `~` was required, or `~` used where mutation was not intended -> FC2
   - signed/unsigned parser or serializer mismatch, negative amount/vote/timestamp acceptance, or missing non-negative bounds -> FC3
   - local variable or helper parameter shadows a storage field and the wrong value is validated or saved -> FC4/FC7
   - ignored success flag, optional result, or nullable lookup from a dictionary, storage, or system helper whose failure changes authorization, accounting, or state -> FC5
   - a fixed-layout payload slice, ref cell, or top-level storage loader without a matching `end_parse()` or equivalent full-emptiness proof -> FC6
   - optimistic liquidity, balance, or entitlement mutation before a dependent ignored-error downstream credit or deployment send -> strongest of TC20 and TC26
   - a native-only or jetton-only handler that never verifies the configured asset mode or sentinel such as `HOLE_ADDRESS` before crediting value in that mode -> strongest of TC39 and TC40
   - a verifier or multisig signature gate that never enforces a minimum threshold of unique valid signers, or iterates the signature container incorrectly -> TC32
   - a `save_data(...)`, `load_data()`, or `store_*` helper call that passes adjacent same-typed arguments in a different order from the helper signature -> FC7
   - a `provide_wallet_address` / `take_wallet_address` / `get_wallet_address` path that derives the wrong wallet type or disagrees with the contract's own wallet getter formula -> TC49
   - a value-bearing deposit or `transfer_notification` path that credits the asset first, then hits an outer `catch` / silent `return` with no refund or rollback -> strongest of TC22 and TC27
   - an owner/admin withdrawal or "refund remaining" path that transfers from gross inventory without subtracting reserved or unclaimed obligations -> strongest of TC38 and TC39
   - nested selector dispatch such as `child_op` without rejecting unsupported values -> TC38
   - a minter or other authoritative contract that commits `total_supply`, balance, liquidity, or entitlement before a dependent wallet-side mint / burn / `internal_transfer` is confirmed, and has no authoritative bounce reconciliation -> TC26
   - a caller-supplied nested ref such as `master_msg` forwarded directly into a wallet or peer message without validating opcode, correlated amount, or exact schema -> TC25
   - a privileged mint or admin path that parses fields from a nested peer-message body but never explicitly checks that the nested opcode matches the intended standard operation before forwarding it -> weaker TC25
   - hardcoded magic opcode, width, message flag, or send mode differs from the claimed schema or intended bounce/value behavior -> FC7/TC30/TC41
   - handled branch does its work but misses an early `return` and falls into a conflicting branch or default throw -> FC8
   - bitwise `~` used as a boolean negation in a condition such as `if (~flag)` or `if (~status)` -> FC9
   - external wallet or admin message lacks `seqno`/nonce, timestamp/`valid_until`, or consumed replay state -> TC11/TC33
   - `accept_message()` or `set_gas_limit()` before signature, freshness, authorization, and message-shape verification -> TC15
   - replay state is saved only after fallible parsing, sends, or execution following gas acceptance -> TC34/TC54
   - value-moving address path omits workchain/canonical-address validation such as `force_chain(...)` -> TC9
   - Jetton deposit or callback trusts the sender/payload wallet instead of deriving the expected user wallet from the trusted minter/master and owner -> TC14
   - parent/child or factory-managed callback trusts body fields instead of deriving the expected peer address and checking the inbound sender -> TC58/TC18/TC53
3. For FC6 specifically, if the fixed-layout parser influences authorization, accounting, configuration, outbound message construction, or authoritative storage decoding, treat the missing full-consumption check as a standalone surviving finding rather than merely supporting evidence for another issue. Privilege affects confidence, not whether the pattern survives triage. A `slice_bits(...) == N` guard is not sufficient if refs can still remain.
4. During deep pass, preserve FC6, TC25 nested-message forwarding, asset-mode mismatch, and optimistic accounting/supply desync as distinct findings when their exploit mechanics or local fixes differ.
5. For confirmed findings, apply all `judging.md` deductions. If a finding was triggered by a preserved concrete alias, name that concrete pattern in the title or first sentence of `Description`; if you confirm FC6, name the exact parser site or reused payload variable.
