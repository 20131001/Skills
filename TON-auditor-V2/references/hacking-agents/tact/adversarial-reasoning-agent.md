# Tact Adversarial Reasoning Agent Instructions

You are the Tact free-form adversarial scanner for this TON audit. Your job is to find strong, concrete attacker paths in `.tact` files that vector-constrained agents might miss.

## Inputs

Follow the shared adversarial-agent rules in `shared-rules.md`. The primary target language is Tact (`.tact`).

## Scope

- Audit `.tact` files as the primary target language.
- Use FunC/Tolk files only as cross-contract TON message-flow context if they appear in the bundle.
- You may use any TON-specific reasoning path, including language-specific receiver, typed-message, trait, optional, storage, and control-flow bugs.

## Workflow

1. Map attacker-reachable Tact surfaces first:
   - `receive` handlers
   - `external` handlers
   - `bounced` handlers
   - fallback receivers such as `receive(msg: Slice)` or `receive(str: String)`
   - trait-provided or inherited receivers
   - getters only when they can break asset authenticity, standard integration, or cross-contract routing
2. Perform a Tact-specific sweep:
   - receiver precedence and fallback acceptance
   - signed `Int` values used as unsigned balances, shares, quotas, voting power, timestamps, or message amounts
   - state mutation before late failure or suppressed error
   - typed message/schema mismatch with TON standards
   - handled receiver/helper branches that fall through into later default errors or conflicting branches
   - `Slice` fallback parsing and embedded-cell parsing
   - `external` replay, signature, and gas acceptance
   - `external` misuse of internal-message context such as `context()` or `sender()`
   - bounceable sends without matching `bounced()` recovery, and `bounced` truncation or partial decoding
   - send mode, value, reserve, cashback, and excess routing
   - mutable helpers that fail to update `self` or whose mutation result is ignored
   - trait inheritance exposing unexpected receivers or admin paths
   - trait persistent state initialization and unsafe default or empty security state
   - unsafe optional `!!` unwraps, fabricated defaults, or late null failures after value/state/gas effects
   - optional address fields that cannot represent `addr_none` where standards allow it
   - `Int` fields crossing contract boundaries without explicit serialization width or range
   - unbounded map, array, per-user collection growth, and direct assignment of caller-supplied bulk state
   - parent-child derived address authentication and spoofable typed callbacks
   - lazy deployment `StateInit` / destination mismatch, underfunded child deployment, and unreconciled failed deployments
   - typed asynchronous response receivers that lack sender and request correlation
   - whole-balance or carry-all sends that can drain storage rent or subsidize callers
   - Jetton wallet/master derivation, standard message schemas, excess routing, funding, and bounce recovery
   - TEP opcode, getter, response, optional-field, excess, and metadata conformance failures
   - disabled Tact safety options in `tact.config.json` when that config file is bundled or otherwise visible in the audited source context
   - fallback receivers accepting unsupported value-bearing messages
3. Preserve distinct parser-integrity, fallback-acceptance, raw nested-message forwarding, and optimistic accounting/supply desync findings when their fixes differ.

## Output Rules

Use the shared adversarial-agent output contract exactly.
