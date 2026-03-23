# TON Security Best Practices Reference

Use this file as the local summary of TON smart-contract security guidance. It combines the official `docs.ton.org` guidance with selected audit-proven FunC practices distilled from CertiK's TON security writeup. It is complementary to `standards-and-lessons.md`: this file focuses on generic security anti-patterns, exploit mechanics, and development patterns rather than interface standards.

## When To Use This Reference

Consult this file whenever the audited code includes any of the following:

- `recv_external`, `accept_message()`, `set_gas_limit()`, signatures, `seqno`, or `valid_until`
- multi-contract workflows, message cascades, entrypoint handlers, or bounced-message recovery
- cross-contract request-response flows, callback handlers, or on-chain attempts to read another contract's state
- `random()`, `rand()`, `randomize_lt()`, lotteries, raffles, NFT rarity, or other value-bearing randomness
- code upgrade paths such as `set_code`, `set_c3`, code refs, or explicit update/admin code handlers
- manual cell serialization or deserialization of balances, permissions, or config
- large flat `load_data()` / `save_data()` layouts, storage shadowing, or storage-group refactors
- destructive send modes, especially `128 + 32`
- `send_raw_message`, fee accounting, cashback, excesses, or `raw_reserve()`

## Core Security Rules

### Message Flow Design

- Model every non-trivial workflow as an explicit message cascade with a clear entrypoint and consequence messages.
- Partial execution is normal in TON; never assume a later step will revert an earlier committed step.
- Put payload validation, sender checks, gas sufficiency, and phase checks at the entrypoint of the cascade whenever possible.
- If one contract is the authoritative keeper of supply, liquidity, or entitlement, do not commit that authoritative state before the downstream wallet or peer-side consequence is known to have succeeded unless the authoritative layer also implements an explicit reconciliation path for bounces and failures.
- If a contract supports native-mode and jetton-mode operation, each mode-specific entrypoint must verify the active asset configuration or sentinel such as `HOLE_ADDRESS` / `addr_none` before accepting value.
- Administrative withdrawal, rescue, or “refund remaining” flows must compute the free balance after subtracting all reserved or unclaimed user obligations; never let an owner drain inventory that the contract still owes to claimants.
- Never authenticate a caller by reading an address from `in_msg_body` or other attacker-controlled payload fields; use the sender from the trusted inbound message envelope or parsed full-message context.
- Do not accept a caller-supplied nested wallet or peer message cell such as `master_msg` and forward it unchanged after parsing only one or two fields; rebuild the outbound body from trusted fields, or prove the nested opcode, amount, and layout match the authorized outer request exactly.
- Reject unsupported selector, enum, or role-type values explicitly; nested dispatchers on `child_op`-style fields should not silently succeed on unknown branches.
- In later consequence messages, `throw_*` checks should usually behave like assertions protecting invariants, not like late business-logic validation.
- Bounced messages are useful recovery signals, but they are not full protection:
  - they require enough gas to be produced and processed
  - they are not transitive, so a failure deep in the cascade may never be visible to the original entry contract
  - a bounced message cannot be bounced again

### External Messages

- External messages do not carry value for their own processing. After `accept_message()` or an equivalent `set_gas_limit()` increase, the contract pays from its own balance.
- Never call `accept_message()` or raise gas limits before signature, freshness, authorization, and message-shape checks.
- Replay protection should bind:
  - a nonce or `seqno`
  - expiration such as `valid_until`
  - a wallet-specific or deployment-specific domain when signatures may otherwise be reusable elsewhere
- For bridge verifiers, multisigs, and any threshold-signature flow, validate the signature set as a set: traverse the full container correctly, require the configured minimum number of valid signatures, and count only unique verifier identities.
- If a path can still throw after `accept_message()`, commit replay-protection state before the fallible work.

### Async Communication And Callbacks

- A contract cannot synchronously call another contract's getter on-chain; every cross-contract read becomes a request-response flow.
- Prefer the carry-value pattern: move the authoritative value or state through the message flow instead of asking a remote contract to describe what it currently owns.
- Treat callback and response handlers as attacker-reachable until both sender authenticity and response correlation are verified.
- If multiple requests can be in flight, bind each reply to stored pending-request state such as the expected sender, phase, nonce, or query id.
- Unsolicited or stale replies must not move funds, finalize state, or overwrite authoritative data.
- If a request arrives together with TON, Jettons, NFTs, or another asset and the contract cannot fulfill it, return the received value to the authenticated sender instead of simply throwing and trapping it.
- Unexpected or unsupported inbound assets should be explicitly returned, or routed into a clearly defined rescue flow, rather than silently retained.
- Do not use an outer empty `catch` or silent `return` on a value-bearing deposit path after the asset has already been credited locally; either refund the asset or reverse the optimistic accounting before exiting.

### Randomness

- Treat single-block on-chain randomness as attacker-influenceable.
- Do not rely on `random()` or `randomize_lt()` for high-value lotteries, trait assignment, or privileged selection.
- Keep randomness out of `recv_external`; validators can still exploit inclusion timing or seed control.
- Prefer multi-phase commit-reveal or off-chain/oracle-assisted entropy for value-bearing flows.

### Storage And Serialization

- Keep store/load types consistent:
  - width
  - signedness
  - address vs integer encoding
  - inline vs referenced cell layout
- Manual serialization bugs are security-relevant when they can corrupt balances, roles, config, or parser alignment.
- Flat storage layouts with very large `load_data()` / `save_data()` argument lists are high risk because they encourage argument swaps, field shadowing, and namespace pollution.
- When helper signatures contain many same-typed fields, compare each `save_data(...)`, `store_*`, or builder call site against the authoritative signature; swapped adjacent integers or addresses are a real state-corruption risk, not a style issue.
- Prefer grouped or nested storage cells so unrelated state is not unpacked and repacked on every path.
- Use `end_parse()` on message bodies and storage sub-parsers whenever the layout is expected to be fully consumed.
- Apply the same rule to nested parsers created from refs or helper cells, such as permission dictionaries, embedded payload cells, or op-specific message refs.
- Apply the same rule to reused slices that were not created by `begin_parse()`, such as `fwd_payload = in_msg_body` or helper-returned payload slices.
- If you gate a fixed-layout payload with `slice_bits(...) == N`, also prove there are no leftover refs, or finish with an equivalent emptiness check; exact bit length alone does not prove exact layout.
- Treat caller-supplied nested cells in role-gated flows the same way; a missing `end_parse()` on an admin-only or minter-only payload parser is still a parser-integrity bug, with confidence reduced for privilege rather than dropped as unreachable.
- Check return flags from dictionary and system helpers before assuming a mutation succeeded.

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
