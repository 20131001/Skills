# TON Security Best Practices Reference

Use this file as the local summary of TON smart-contract security guidance for FunC, Tolk, and Tact audits. It combines official TON guidance with audit-proven TVM contract practices. It is complementary to `standards-and-lessons.md`: this file focuses on generic security anti-patterns, exploit mechanics, and development patterns rather than interface standards. Upstream source links are tracked in `source-links.md`; audit agents should use this local summary during normal audits for determinism.

## When To Use This Reference

Consult this file whenever the audited code includes any of the following:

- external message handlers, gas acceptance, signatures, `seqno`, nonce, or `valid_until`
- multi-contract workflows, message cascades, entrypoint handlers, or bounced-message recovery
- cross-contract request-response flows, callback handlers, or on-chain attempts to read another contract's state
- claimed TEP standard compatibility, including Jetton, NFT, DNS, SBT, royalty, metadata, source registry, normalized hash, compressed NFT, or scaled UI Jetton behavior
- `random()`, `rand()`, `randomize_lt()`, lotteries, raffles, NFT rarity, or other value-bearing randomness
- code upgrade paths such as `set_code`, `set_c3`, code refs, or explicit update/admin code handlers
- manual cell serialization or deserialization of balances, permissions, or config
- FunC `impure` helpers, `.` / `~` method calls, bitwise conditions, `load_data()` / `save_data()` layouts, and `end_parse()` parser boundaries
- Tolk typed/lazy message parsers, nullable or optional helper results, `Storage.load` / `Storage.save`, custom codecs, router branches, and signed value domains
- Tact message structs, optional fields, optional addresses, mutable helpers, external receivers, traits, parent-child deployments, `Int` serialization, fallback receivers, state collections, or `tact.config.json` safety settings
- large flat storage load/save layouts, storage shadowing, or storage-group refactors
- human-readable on-chain parsing such as strings, comments, base64, JSON-like payloads, or metadata blobs
- destructive send modes, especially `128 + 32`
- `send_raw_message`, fee accounting, cashback, excesses, or `raw_reserve()`

## Core Security Rules

### Message Flow Design

- Model every non-trivial workflow as an explicit message cascade with a clear entrypoint and consequence messages.
- Partial execution is normal in TON; never assume a later step will revert an earlier committed step.
- Put payload validation, sender checks, gas sufficiency, and phase checks at the entrypoint of the cascade whenever possible.
- Do not assume a property checked at stage one still holds at stage three. Reserve, lock, carry, or re-check balances, ownership, phase, and eligibility before each dependent state transition.
- Avoid storing unkeyed temporary request data for a later message. If a flow needs pending state, key it by request id and expected sender, consume it once, and reject stale or unsolicited replies.
- For master/child or factory-deployed groups, derive the expected child, master, or wallet address from trusted state and code before accepting peer messages.
- If one contract is the authoritative keeper of supply, liquidity, or entitlement, do not commit that authoritative state before the downstream wallet or peer-side consequence is known to have succeeded unless the authoritative layer also implements an explicit reconciliation path for bounces and failures.
- If a contract supports native-mode and jetton-mode operation, each mode-specific entrypoint must verify the active asset configuration or sentinel such as `HOLE_ADDRESS` / `addr_none` before accepting value.
- Administrative withdrawal, rescue, or "refund remaining" flows must compute the free balance after subtracting all reserved or unclaimed user obligations; never let an owner drain inventory that the contract still owes to claimants.
- Never authenticate a caller by reading an address from `in_msg_body` or other attacker-controlled payload fields; use the sender from the trusted inbound message envelope or parsed full-message context.
- Do not accept a caller-supplied nested wallet or peer message cell such as `master_msg` and forward it unchanged after parsing only one or two fields; rebuild the outbound body from trusted fields, or prove the nested opcode, amount, and layout match the authorized outer request exactly.
- Reject unsupported selector, enum, or role-type values explicitly; nested dispatchers on `child_op`-style fields should not silently succeed on unknown branches.
- In later consequence messages, `throw_*` checks should usually behave like assertions protecting invariants, not like late business-logic validation.
- Bounced messages are useful recovery signals, but they are not full protection:
  - they require enough gas to be produced and processed
  - they are not transitive, so a failure deep in the cascade may never be visible to the original entry contract
  - a bounced message cannot be bounced again
- Choose bounce mode from value ownership and recovery semantics. Use bounceable sends when the sender must recover or reconcile failure, and non-bounceable sends only when the destination should keep value or failure is harmless.

### External Messages

- External messages do not carry value for their own processing. After `accept_message()` or an equivalent `set_gas_limit()` increase, the contract pays from its own balance.
- Never call `accept_message()` or raise gas limits before signature, freshness, authorization, and message-shape checks.
- Replay protection should bind:
  - a nonce or `seqno`
  - expiration such as `valid_until`
  - a wallet-specific or deployment-specific domain when signatures may otherwise be reusable elsewhere
- For bridge verifiers, multisigs, and any threshold-signature flow, validate the signature set as a set: traverse the full container correctly, require the configured minimum number of valid signatures, and count only unique verifier identities.
- If a path can still throw after `accept_message()`, commit replay-protection state before the fallible work.
- Treat any manual commit checkpoint as a security boundary. It is appropriate for replay-protection state when later failures must not allow replay, but it is dangerous for partially validated business state, accounting, or queued actions before later fallible work.

### Async Communication And Callbacks

- A contract cannot synchronously call another contract's getter on-chain; every cross-contract read becomes a request-response flow.
- Prefer the carry-value pattern: move the authoritative value or state through the message flow instead of asking a remote contract to describe what it currently owns.
- Do not use on-chain balance-query request/response flows as authority for later settlement; by the time the response arrives, the balance may already have changed.
- Treat callback and response handlers as attacker-reachable until both sender authenticity and response correlation are verified.
- If multiple requests can be in flight, bind each reply to stored pending-request state such as the expected sender, phase, nonce, or query id.
- Unsolicited or stale replies must not move funds, finalize state, or overwrite authoritative data.
- If a request arrives together with TON, Jettons, NFTs, or another asset and the contract cannot fulfill it, return the received value to the authenticated sender instead of simply throwing and trapping it.
- Unexpected or unsupported inbound assets should be explicitly returned, or routed into a clearly defined rescue flow, rather than silently retained.
- For Jetton transfers into an application, validate `forward_payload` before crediting the request path; if the payload is empty or unsupported for that application protocol, return the Jettons to the authenticated sender instead of throwing after receipt.
- Do not use an outer empty `catch` or silent `return` on a value-bearing deposit path after the asset has already been credited locally; either refund the asset or reverse the optimistic accounting before exiting.
- Do not parse human-readable user data such as strings, comments, base64, or JSON-like payloads on-chain unless there is a strict bound and enough gas. Prefer compact binary formats and off-chain parsing.

### Randomness

- Treat single-block on-chain randomness as attacker-influenceable.
- Do not rely on `random()` or `randomize_lt()` for high-value lotteries, trait assignment, or privileged selection.
- Keep randomness out of external message handlers; validators can still exploit inclusion timing or seed control.
- Prefer multi-phase commit-reveal or off-chain/oracle-assisted entropy for value-bearing flows.

### Storage And Serialization

- Keep store/load types consistent:
  - width
  - signedness
  - address vs integer encoding
  - inline vs referenced cell layout
- Manual serialization bugs are security-relevant when they can corrupt balances, roles, config, or parser alignment.
- Flat storage layouts with very large load/save argument lists are high risk because they encourage argument swaps, field shadowing, and namespace pollution.
- When helper signatures, typed storage structs, or builder calls contain many same-typed fields, compare every call site against the authoritative schema; swapped adjacent integers or addresses are a real state-corruption risk, not a style issue.
- Prefer grouped or nested storage cells so unrelated state is not unpacked and repacked on every path.
- Require exact-layout proof on message bodies, storage sub-parsers, embedded cells, forwarded payloads, and typed message/storage decoders whenever the layout is expected to be fully consumed.
- In FunC this usually means `end_parse()` or equivalent bit-and-ref emptiness proof. In Tolk and Tact this means the language-specific typed parser, custom serializer, or `Slice` fallback must prove the same exact shape.
- A pure bit-count check is not enough when references may remain.
- Treat caller-supplied nested cells in role-gated flows the same way; a missing exact-layout proof on an admin-only or minter-only payload parser is still a parser-integrity bug, with confidence reduced for privilege rather than dropped as unreachable.
- Check return flags from dictionary and system helpers before assuming a mutation succeeded.

### FunC-Specific Checks

- Mark side-effecting helpers as `impure` when they throw, authorize, mutate state, send messages, or otherwise enforce security through effects. A side-effecting call whose result is unused can be optimized away if the function is not `impure`.
- Review every `.` and `~` method call on slices, builders, dictionaries, and state containers. `~` persists the modified first argument; `.` returns a copy and leaves the original unchanged.
- Do not use bitwise `~` as boolean negation. Use `ifnot(...)`, equality checks, or explicit TVM boolean conventions so paused, allowed, or success flags cannot become always-true conditions.
- Avoid shadowing loaded storage fields with local variables or helper parameters. Keep `load_data()` unpacking, local mutations, and `save_data()` arguments aligned by name, order, and type.
- Use `end_parse()` or an equivalent bit-and-ref emptiness proof for fixed-layout message payloads, nested refs, storage cells, and helper-returned slices.
- Keep integer domains explicit. Reject negative values for balances, voting power, timestamps, sequence numbers, and amounts; match signedness and widths between `load_*`, `store_*`, and standards.
- External FunC handlers must verify signature, owner, `seqno` or nonce, `valid_until` or timestamp, and domain separation before `accept_message()`. After gas is accepted, commit replay-protection state before fallible parsing, sends, or business execution that could throw.
- Replace magic numbers for opcodes, bit widths, send modes, and internal-message prefixes with named constants or helper builders, and verify the constants against the claimed TON standard.
- Validate address workchain and canonical form before value movement. For Jetton deposits or callbacks, derive the expected wallet from the trusted master and owner instead of trusting sender- or payload-supplied wallet addresses.

### Tolk-Specific Checks

- Audit typed message structs, lazy field access, manual slice/cell parsing, custom codecs, and storage decoders as fallible parsing boundaries. Fields first accessed after state mutation or value acceptance can still fail too late.
- Require exact-layout proof for fixed-layout message, config, permission, forwarded-payload, and storage parsers. Tolk typed decoders and custom serializers need the same bit-and-ref exactness expected from FunC `end_parse()`.
- Keep signed integer domains explicit. Reject negative values for balances, shares, quotas, voting power, timestamps, and amounts when the business domain is unsigned.
- Check nullable, optional, and success-flag results from dictionary, storage, and system helpers before assuming a lookup, deletion, update, or mutation succeeded.
- Compare `Storage.load`, `Storage.save`, struct packing, field order, signedness, address kinds, optional tags, references, and custom codecs against the authoritative TL-B/schema and peer contracts.
- Ensure router, enum, selector, and opcode branches terminate after handled work, and reject unsupported values explicitly instead of falling through or silently succeeding.
- Treat catch/suppressed-error paths after receiving or crediting value as high risk unless they refund or roll back the asset/accounting mutation.
- Check external-message handlers for signature validation, domain separation, replay protection, and gas acceptance only after validation.
- Check bounced-message handlers against TON bounce truncation; do not assume late fields from the original body are available.

### Tact-Specific Checks

- Keep Tact safety checks enabled for security-critical code unless every disabled check is replaced by explicit equivalent guards.
- Treat optional variables and optional fields as nullable until proven otherwise. Prefer explicit null checks and controlled rejection/refund paths before `!!`; unsafe force-unwraps are security-relevant when they can throw after value receipt, state mutation, or gas acceptance.
- Persistent state variables must have an explicit default value or be initialized in `init()`, except maps that are intentionally empty by default. Review zero, empty, and trait-initialized values as security state, not just deployment boilerplate.
- A contract inheriting a trait with persistent state variables needs its own `init()` to initialize that trait state. Verify inherited state and inherited receivers use the same authorization, phase, and value assumptions as the concrete contract.
- Review mutable helpers that are expected to persist state. In Tact, mutation must update `self`; updating a temporary value or ignoring a returned mutation can leave later sends or accounting based on state that was never saved.
- In `external` handlers, do not use internal-message assumptions such as `context()` or `sender()`. Authenticate with signed data, freshness, and explicit message fields before accepting gas or doing bounded work.
- Use optional address types for TON standard fields that may be `addr_none` or `Maybe ^MsgAddress`; a non-optional `Address` cannot safely represent those cases.
- Do not rely on default `Int` serialization for external ABI fields. Specify the exact width or coin serialization expected by the standard or peer contract, and range-check signed values used as unsigned business amounts.
- Audit `receive`, `external`, `bounced`, and fallback receivers separately. Empty, text, string, or `Slice` fallbacks should not silently accept unsupported value-bearing protocol commands.
- If a path sends with `bounce: true` and relies on recovering from rejection, provide a `bounced()` handler, trait-provided recovery, or another explicit reconciliation path keyed to the original operation.
- Bound attacker-controlled map, array, and per-user collection growth. State-size limits should be part of the protocol design, not only an implicit gas limit.
- Do not assign caller-supplied maps, dictionary-backed arrays, or bulk config objects directly into persistent state. Iterate through bounded entries, validate key and value domains, and charge or reserve for storage before saving.
- Enumerate trait-provided receivers and trait state as part of the public attack surface. Ownership, pause/resume, and custom trait hooks must not silently add admin or value-moving paths outside the concrete contract's intended guard model.
- In parent-child patterns, authenticate children and parents by recomputing the expected address from trusted `StateInit` inputs or stored state. Do not trust sequence numbers, parent addresses, or owner fields supplied in the callback body.
- For lazy deployment, build `to`, `code`, and `data` from the same `StateInit` value, include deployment funding in the gas budget, and reconcile parent state if child deployment or initialization can fail.
- Check that state mutations before a `try/catch`, send, reserve, or parsing operation remain safe if that later operation fails. Tact auto-persistence around receivers can make optimistic updates visible even when a later branch is suppressed.
- Verify trait-provided receivers and inherited methods do not expose admin or value-moving paths that the concrete contract forgot to gate.

### Upgrades And Code Execution

- Any code-update path must be strictly authorized and validate the new code or upgrade payload shape.
- Executing untrusted third-party code is dangerous even when the interface looks constrained; gas exhaustion and partial execution can still break invariants.

### Destruction And Lifecycle

- Destructive send modes such as `128 + 32` require a stronger proof of safety than normal sends.
- Do not destroy a contract while it may still have pending obligations, in-flight message chains, or user funds that rely on a later callback.
- If destruction is intended, gate it with explicit authorization and an invariant that no obligations remain.

### Gas And Excesses

- Underfunded message cascades can partially execute and leave state committed without the intended downstream effects.
- Budget gas per handler and per outgoing branch, not just as one coarse global margin for the whole workflow.
- Re-check `msg_value` assumptions whenever the number of consequence messages changes.
- Include `StateInit` deployment cost and any per-user helper-account credit message in the funding check when a path deploys or initializes downstream accounts.
- Returning excess value is the contract's responsibility.
- `SEND_MODE_CARRY_ALL_REMAINING_MESSAGE_VALUE` is safest in linear one-send flows; once a path also spends contract balance, emits external messages, attaches explicit value, or pays fees separately, the flow may silently subsidize callers.
- Avoid carry-all remaining value after previous sends, explicit-value sends, or fee-separate sends in the same handler unless a reserve protects the contract balance.
- Storage fees are paid from the contract balance, not from inbound `msg_value`; external-out messages, explicit attached values, and fee-separate sends can also spend contract balance. Reserve storage and budget these costs before returning all remaining inbound value.
- Avoid `SendRemainingBalance` or other whole-balance sends from user-triggerable paths unless the contract is intentionally being drained and storage rent obligations are already settled. Prefer carrying the inbound value, returning excess, and reserving storage balance first.
- Do not treat `SEND_MODE_IGNORE_ERRORS` as a substitute for preflight funding checks or rollback logic. If a downstream crediting or settlement send may be ignored, do not persist the upstream accounting mutation unless a later rollback path exists.
- Use `raw_reserve()` or an equivalent reserve discipline when optional sends or repeated callbacks could erode storage balance.

### On-Chain Data Exposure

- Message bodies, persisted cells, and runtime values recoverable through local emulation are public in practice. Never place secrets, passwords, or raw confidential material on-chain.

### Code Clarity And Builder Safety

- Prefer helper builders, wrappers, and named constants over manually repeating bit widths, message prefixes, send modes, and op codes.
- Reducing magic numbers lowers the chance of malformed message headers, incorrect body layouts, or unsafe flag combinations in future edits.

## Reporting Guidance

- Report the root cause, not every derived symptom.
- A best-practice deviation is reportable only when it yields a concrete exploit path, fund loss, fund lock, spoofing path, denial of service, or broken invariant.
- Drop pure style or documentation issues. For example, message-flow diagrams, helper wrappers, and named constants are strong engineering guidance, but their absence alone is not a finding unless it creates a concrete bug.
