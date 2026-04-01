# TON Security Best Practices Reference

Use this file as the local summary of TON smart-contract security guidance. It complements `standards-and-lessons.md`: this file focuses on generic exploit mechanics, message-flow failures, and development patterns rather than interface standards.

## Goal

Translate TON secure-programming guidance into compact audit rules for message flows, async settlement, serialization, upgrade safety, and gas-aware value handling.

## Consult This File When

- the code handles `recv_external`, `accept_message()`, `set_gas_limit()`, signatures, `seqno`, or `valid_until`
- the code uses multi-contract workflows, message cascades, callback handlers, or bounced-message recovery
- the code attempts cross-contract reads or request-response flows
- the code uses `random()`, `rand()`, `randomize_lt()`, or any value-bearing randomness
- the code contains code-upgrade paths such as `set_code`, `set_c3`, code refs, or explicit update/admin code handlers
- the code manually serializes or deserializes balances, permissions, or config
- the code has large flat `load_data()` / `save_data()` layouts
- the code uses destructive send modes such as `128 + 32`
- the code depends on `send_raw_message`, fee accounting, cashback, excess handling, or `raw_reserve()`

## Core Rules

### 1. Message Flow Design

- Model every non-trivial workflow as an explicit message cascade with one clear entrypoint and explicit consequence messages.
- Partial execution is normal in TON. Never assume a later step will revert an earlier committed step.
- Put payload validation, sender checks, gas sufficiency, and phase checks at the entrypoint whenever possible.
- If one contract is the authoritative keeper of supply, liquidity, or entitlement, do not commit that authoritative state before the downstream wallet or peer-side consequence is known to have succeeded unless the authoritative layer also reconciles bounces and failures.
- If a contract supports native-mode and jetton-mode operation, each mode-specific entrypoint must verify the active asset configuration or sentinel such as `HOLE_ADDRESS` or `addr_none` before accepting value.
- Administrative withdrawal, rescue, or “refund remaining” flows must compute free balance after subtracting reserved or unclaimed user obligations.
- Never authenticate a caller by reading an address from `in_msg_body` or other attacker-controlled payload fields.
- Do not accept a caller-supplied nested wallet or peer message cell such as `master_msg` and forward it unchanged after parsing only one or two fields; either rebuild the body from trusted fields or prove the nested opcode, amount, and layout match the authorized outer request.
- Reject unsupported selector, enum, or role-type values explicitly.
- In later consequence messages, `throw_*` checks should usually behave like invariant assertions, not late business-logic validation.
- Bounced messages help recovery, but they are not full protection:
  - they require gas to be produced and processed
  - they are not transitive
  - a bounced message cannot be bounced again

### 2. External Messages

- External messages do not carry value for their own processing. After `accept_message()` or an equivalent gas-limit raise, the contract pays from its own balance.
- Never call `accept_message()` or raise gas limits before signature, freshness, authorization, and message-shape checks.
- Replay protection should bind:
  - a nonce such as `seqno`
  - expiration such as `valid_until`
  - a wallet-specific or deployment-specific domain
- For bridge verifiers, multisigs, and threshold-signature flows, validate the signature set as a set:
  - traverse the full container correctly
  - require the configured minimum number of valid signatures
  - count only unique verifier identities
- If a path can still throw after `accept_message()`, commit replay-protection state before the fallible work.

### 3. Async Communication And Callbacks

- A contract cannot synchronously call another contract’s getter on-chain; every cross-contract read becomes a request-response flow.
- Prefer the carry-value pattern: move the authoritative value or state through the message flow instead of asking a remote contract to describe what it owns.
- Treat callback and response handlers as attacker-reachable until both sender authenticity and response correlation are verified.
- If multiple requests can be in flight, bind each reply to stored pending-request state such as expected sender, phase, nonce, or query id.
- Unsolicited or stale replies must not move funds, finalize state, or overwrite authoritative data.
- If a request arrives together with TON, Jettons, NFTs, or another asset and the contract cannot fulfill it, return the received value to the authenticated sender instead of trapping it.
- Unexpected or unsupported inbound assets should be explicitly returned or routed into a clearly defined rescue flow.
- Do not use an outer empty `catch` or silent `return` on a value-bearing deposit path after the asset has already been credited locally; either refund the asset or reverse the optimistic accounting.

### 4. Randomness

- Treat single-block on-chain randomness as attacker-influenceable.
- Do not rely on `random()` or `randomize_lt()` for high-value lotteries, privileged selection, or value-bearing trait assignment.
- Keep randomness out of `recv_external`.
- Prefer multi-phase commit-reveal or off-chain / oracle-assisted entropy for value-bearing flows.

### 5. Storage And Serialization

- Keep store/load types consistent:
  - width
  - signedness
  - address vs integer encoding
  - inline vs referenced cell layout
- Manual serialization bugs are security-relevant when they can corrupt balances, roles, config, or parser alignment.
- Flat `load_data()` / `save_data()` layouts with many parameters are high risk because they encourage argument swaps, field shadowing, and namespace pollution.
- When helper signatures contain many same-typed fields, compare each `save_data(...)`, `store_*`, or builder call site against the authoritative signature.
- Prefer grouped or nested storage cells so unrelated state is not unpacked and repacked on every path.
- Use `end_parse()` on message bodies and storage sub-parsers whenever the layout is expected to be fully consumed. It succeeds only when the slice has been fully consumed and throws if bits or refs still remain.
- Apply the same rule to:
  - top-level storage loaders such as `load_data()` built from `get_data().begin_parse()`
  - nested parsers created from refs or helper cells
  - reused payload slices such as `fwd_payload = in_msg_body`
- If you gate a fixed-layout payload with `slice_bits(...) == N`, also prove there are no leftover refs or finish with an equivalent emptiness check.
- Treat caller-supplied nested cells in role-gated flows the same way; a missing `end_parse()` on an admin-only or minter-only payload parser is still a parser-integrity bug.
- Check return flags from dictionary and system helpers before assuming a mutation succeeded.

### 6. Upgrades And Code Execution

- Any code-update path must be strictly authorized and validate the new code or upgrade payload shape.
- Executing untrusted third-party code is dangerous even when the interface looks constrained; gas exhaustion and partial execution can still break invariants.

### 7. Destruction And Lifecycle

- Destructive send modes such as `128 + 32` require a stronger proof of safety than normal sends.
- Do not destroy a contract while it may still have pending obligations, in-flight message chains, or user funds that rely on a later callback.
- If destruction is intended, gate it with explicit authorization and an invariant that no obligations remain.

### 8. Gas And Excesses

- Underfunded message cascades can partially execute and leave state committed without the intended downstream effects.
- Budget gas per handler and per outgoing branch, not just as one coarse global margin for the whole workflow.
- Re-check `msg_value` assumptions whenever the number of consequence messages changes.
- Include `StateInit` deployment cost and any per-user helper-account credit message in the funding check when a path deploys or initializes downstream accounts.
- Returning excess value is the contract’s responsibility.
- `SEND_MODE_CARRY_ALL_REMAINING_MESSAGE_VALUE` is safest in linear one-send flows; once a path also spends contract balance, emits external messages, attaches explicit value, or pays fees separately, the flow may silently subsidize callers.
- Do not treat `SEND_MODE_IGNORE_ERRORS` as a substitute for preflight funding checks or rollback logic.
- Use `raw_reserve()` or an equivalent reserve discipline when optional sends or repeated callbacks could erode storage balance.

### 9. On-Chain Data Exposure

- Message bodies, persisted cells, and runtime values recoverable through local emulation are public in practice.
- Never place secrets, passwords, or raw confidential material on-chain.

### 10. Code Clarity And Builder Safety

- Prefer helper builders, wrappers, and named constants over manually repeating bit widths, message prefixes, send modes, and op codes.
- Reducing magic numbers lowers the chance of malformed headers, incorrect body layouts, or unsafe flag combinations in future edits.

## Reporting Guidance

- Report the root cause, not every derived symptom.
- A best-practice deviation is reportable only when it yields a concrete exploit path, fund loss, fund lock, spoofing path, denial of service, or broken invariant.
- Drop pure style or documentation issues. Message-flow diagrams, helper wrappers, and named constants are good engineering practice, but their absence alone is not a finding unless it creates a concrete bug.
