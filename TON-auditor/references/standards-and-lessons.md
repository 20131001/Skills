# Standards And Lessons Reference

Use this file as the semantic source for TON standards and core FunC lessons.

## Goal

Compress the high-signal requirements from the TEPs and the FunC course into audit rules that are easy to apply during review.

## Consult This File When

- the code implements Jetton handlers or getters such as `transfer`, `burn`, `internal_transfer`, `burn_notification`, `get_wallet_data`, `get_jetton_data`, or `provide_wallet_address`
- the code implements NFT handlers or getters such as `transfer`, `get_static_data`, `get_nft_data`, `get_collection_data`, or `royalty_params`
- the code parses or constructs token metadata
- the code handles `recv_external`, signatures, `accept_message()`, or `commit()`
- the code serializes internal messages, handles bounces, uses `raw_reserve()`, or relies on `SEND_MODE_*`

## Source Map

### Jettons

- `TEP-74`: Jetton message schemas, opcodes, authorization rules, notification / excess behavior, getter ABI
- `TEP-89`: `provide_wallet_address`, `take_wallet_address`, and `Maybe ^MsgAddress` wallet discovery encoding
- `TEP-64`: token metadata content prefixes, snake / chunked serialization, metadata keys, `decimals`

### NFTs

- `TEP-62`: NFT item transfer flow, `ownership_assigned`, static-data flows, getter ABI
- `TEP-66`: royalty getter and message layouts
- `TEP-64`: collection and item metadata representation

### FunC Lessons

- short-form internal message serialization
- bounce semantics and rewritten fields
- external-message replay and `accept_message()` pitfalls
- carry-value gas accounting and partial execution hazards
- Jetton wallet/minter design expectations

## Audit Rules

### Standards Claims

- Treat `MUST` and `REQUIRED` requirements as hard interface or safety checks when the contract claims compatibility with the standard.
- Treat `SHOULD` deviations as reportable only when they create:
  - real fund loss or lock
  - spoofing or false-credit risk
  - broken wallet, explorer, bridge, marketplace, or other standard interoperability with concrete consequences
- If the contract is clearly closed-system and does not claim standard compatibility, many ABI mismatches are false positives.

### TL-B And Getter Discipline

- Field names are not enforced on-chain, but field order, field width, tagged unions, and address/value types are.
- Audit exact opcode, field order, and field types for standard messages.
- Audit exact tuple order and return types for standard getters.
- `Maybe` and `Either` tags matter. Do not treat inline and referenced encodings as interchangeable unless the schema allows both.

## Jetton Rules

- `transfer` and `burn` are owner-authorized operations.
- Jetton wallet balance must be checked before decrement.
- Standard transfers must account for receiver-wallet deployment if needed.
- `transfer_notification` is conditional on `forward_ton_amount > 0`.
- `excesses` should preserve `query_id` and return to `response_destination`.
- Burn flows should notify the master so supply stays aligned.
- Jetton master supply must stay aligned with actual wallet balances even when mint or burn consequence messages fail or bounce.
- Minter-provided nested wallet bodies such as `master_msg` should match the intended wallet opcode and amount.
- Jetton deposits should authenticate the sender as the expected wallet derived from the trusted minter and owner.
- Discovery flows should respect `include_address`, `addr_none`, and the `Maybe ^MsgAddress` encoding.
- Discovery and getter derivation must agree on the same wallet type and formula.

## NFT Rules

- NFT item `transfer` is owner-authorized.
- NFT transfer flows must preserve the expected reply path:
  - optional `ownership_assigned`
  - `excesses`
  - preserved `query_id`
- `get_static_data` should return `report_static_data` with the expected schema.
- `get_nft_data`, `get_collection_data`, `get_nft_address_by_index`, and `get_nft_content` are ABI-sensitive.
- Royalty contracts should preserve `TEP-66` getter and message layouts, and royalty fractions should be sane.

## Token Metadata Rules

- `TEP-64` uses:
  - `0x01` for off-chain layout
  - `0x00` for on-chain layout
  - `0x00` snake prefix for serialized chunks
  - `0x01` chunked-data prefix
- Semi-chain content merges on-chain and off-chain metadata, with on-chain values taking precedence on collisions.
- Jetton `decimals` is a UTF-8 string representation of a value from `0` to `255`.
- Metadata issues are worth reporting only when they can materially mislead users, break integrations, or violate an explicit immutability promise.

## Message And Bounce Notes

- In short-form internal message serialization, `0x18` is the common bounceable prefix when using `addr_none` as sender.
- `0x08` is only the bounce bit, not the full short-form header by itself.
- Validators rewrite `src`, `bounced`, `ihr_fee`, `fwd_fee`, `created_lt`, and `created_at` on outbound internal messages.
- A bounced body contains `0xffffffff` plus only the first `256` bits of the original body.
- Bounce handlers cannot safely recover arbitrary later fields from that truncated body.

## External Message Notes

- `recv_external` is dangerous by default.
- `accept_message()` should happen only after signature, freshness, and authorization checks.
- Replay protection should bind:
  - `seqno` or another nonce
  - expiration such as `valid_until`
  - wallet-specific or deployment-specific domain separation
- After `accept_message()`, commit replay-protection state before fallible parsing or sends, or a failed path can be replayed until balance is drained.

## Gas And Carry-Value Notes

- TON can partially execute a transaction if downstream consequences are underfunded.
- Entry points must validate `msg_value` against the whole cascade they trigger.
- Returning excess value is the developer's job.
- `SEND_MODE_CARRY_ALL_REMAINING_MESSAGE_VALUE` is safest in linear flows where each handler sends only one message.
- The following spend contract balance rather than the carried inbound value:
  - attached outbound value
  - `SEND_MODE_PAY_FEES_SEPARATELY`
  - storage fees
  - event-like external sends
- Prefer `raw_reserve()` or equivalent storage-balance protection when a path can otherwise leak reserve funds.

## Reporting Guidance

- Prefer the attacker-reachable root cause over a pure standards-diff note.
- If a standards mismatch is only cosmetic or closed-system, drop it.
- If a standards mismatch breaks how other standard participants authenticate, route, or reconcile assets, it is in scope.
