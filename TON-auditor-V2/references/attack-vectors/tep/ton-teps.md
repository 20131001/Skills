# TEP-Derived Standard Attack Vectors

These vectors are extracted from `ton-blockchain/TEPs/text` and bundled for every target language. They cover TEP standard conformance issues that can create fund loss, fund lock, spoofing, denial of service, broken interoperability, or misleading client behavior.

---

**TP1. TEP:2 Address Encoding, Workchain, or Bounce Flag Mismatch**

- **D:** Contracts, wallets, or helper builders that accept or emit TON addresses without preserving raw/user-friendly encoding semantics, workchain id, 256-bit address width, bounceable/non-bounceable tag, testnet-only flag, or CRC validation can send value to unsupported destinations or mislead users about where value will go.
- **FP:** Address parsing is delegated to a proven standard library and the contract explicitly validates every workchain and bounce-mode assumption before value movement.

**TP2. TEP:62 NFT Transfer Authorization, Funding, and Excess Violation**

- **D:** NFT `transfer#5fcc3d14` handlers that do not authenticate the current owner, do not preflight storage/gas/forward funding, commit ownership without reconciling failed consequence messages, omit the `ownership_assigned` notification when `forward_amount > 0`, or fail to return `excesses#d53276db` with the original `query_id` can transfer or lock NFTs and TON incorrectly.
- **FP:** Owner, funding, notification, and excess paths are enforced under the actual send modes, and every failed consequence is harmless or explicitly preserves/restores ownership and excess routing.

**TP3. TEP:62 NFT Getter, Static Data, or Collection Derivation Mismatch**

- **D:** NFT implementations with wrong `get_nft_data()`, `get_collection_data()`, `get_nft_address_by_index()`, `get_nft_content()`, or `report_static_data#8b771735` ordering, address kind, index semantics, collection address, `addr_none` handling, or send-mode `64` behavior can break ownership discovery, collection authenticity, or wallet/marketplace attribution.
- **FP:** Getter tuple order, collection/item derivation, `query_id`, and content composition are proven identical to the advertised TEP-62 surface, or the contract is explicitly custom and not consumed as a TEP-62 NFT.

**TP4. TEP:64 Token Content Prefix, Dictionary, Snake, or Chunked Encoding Mismatch**

- **D:** Token metadata that uses wrong off-chain/on-chain/semi-chain prefixes, wrong SHA256 dictionary keys, malformed Snake or Chunked data, non-byte-aligned chunks, or incorrect semi-chain merge precedence can make wallets display forged, missing, or attacker-controlled token identity.
- **FP:** Metadata cells are generated/tested against the TEP-64 layouts and key-hash rules, or the asset uses a documented custom metadata format and does not claim TEP-64 compatibility.

**TP5. TEP:64 Jetton Metadata `decimals`, `amount_style`, or `render_type` Drift**

- **D:** Mutable, off-chain-only, invalid, or inconsistent Jetton `decimals`, `amount_style`, or `render_type` metadata can cause clients to display different balances, percentages, hidden assets, or game-style quantities than the on-chain accounting represents.
- **FP:** Display metadata is immutable or governed/documented, changes cannot affect contract-side accounting, and supported clients cannot be deceived about value units or asset identity.

**TP6. TEP:66 Royalty Getter, Report, or Ratio Mismatch**

- **D:** Royalty extensions that omit `royalty_params()`, return the wrong tuple order, allow zero denominators or ratios that exceed documented proceeds, use malformed `report_royalty_params#a8cb00ad`, drop `query_id`, or misroute `destination` can overcharge, underpay, lock royalty flows, or break marketplace support.
- **FP:** Royalty tuple order, ratio bounds, response opcode, response send-mode, and destination address are enforced and interoperable with supported TEP-66 clients.

**TP7. TEP:74 Jetton Transfer Authorization, Funding, and Excess Violation**

- **D:** Jetton wallet `transfer#0f8a7ea5` handlers that do not verify owner sender, balance sufficiency, receiver-wallet deployment cost, storage/gas floor, forward TON, or required excess return can burn TON, strand transfers, subsidize callers, or create sender/receiver balance desync.
- **FP:** The wallet rejects underfunded transfers before mutation, or every committed mutation is reconciled on failed deployment/credit/notification/excess paths under the exact send modes used.

**TP8. TEP:74 Jetton Transfer Notification or Forward Payload Mismatch**

- **D:** Receiver wallets that send `transfer_notification#7362d09c` when `forward_ton_amount == 0`, omit it when `forward_ton_amount > 0`, corrupt `query_id`, `amount`, `sender`, or `forward_payload`, accept an empty payload where the downstream protocol requires a request payload, or mis-handle text/binary comment prefixes can break downstream payment attribution or cause contracts to parse attacker-shaped payloads as trusted payment metadata.
- **FP:** Notification emission is exactly gated by `forward_ton_amount`, fields are copied from the transfer request, and downstream payload parsing accepts empty payloads only for documented no-op/comment flows or rejects/refunds unsupported payloads before credit.

**TP9. TEP:74 Jetton Burn and Master Supply Desync**

- **D:** `burn#595f07bc` handlers or master burn notifications that do not authenticate owner/wallet, do not check jetton balance and TON floor, decrement wallet/master supply optimistically without confirmed notification handling, or fail to return `excesses` can desync total supply or lock user funds.
- **FP:** Burn state changes are authenticated, funded, correlated by `query_id`, and every wallet/master supply mutation is either guaranteed under the send/action semantics used or reconciled on failure.

**TP10. TEP:74 Jetton Getter and Wallet Derivation Mismatch**

- **D:** Incorrect `get_wallet_data()`, `get_jetton_data()`, or `get_wallet_address(owner)` tuple order, signedness, code cell, owner/master address, mintable flag, or derivation formula can let counterfeit wallets pass checks or make legitimate wallets undiscoverable.
- **FP:** Wallet address derivation and getter tuples are centralized, tested against the advertised ABI, and match every deposit/callback authenticity check; custom ABIs are not advertised as TEP-74 compatible.

**TP11. TEP:81 DNS Resolve Semantics Mismatch**

- **D:** DNS implementations or clients that mishandle internal domain representation, root vs non-root leading null byte, category `uint256`, `(8m, record)` return semantics, `(0, null)` failure, partial-resolution recursion, or `dns_next_resolver` lookup can resolve the wrong record or stop at an attacker-controlled resolver.
- **FP:** Domain normalization, prefix length, category hashing, null/self handling, and recursive resolver behavior exactly follow TEP-81.

**TP12. TEP:81 DNS Record Schema or Category Mismatch**

- **D:** DNS records with wrong category hash, record opcode, `MsgAddressInt`, ADNL bits, flags, capability/protocol list, or `HashmapE 256 ^DNSRecord` encoding can spoof wallet/site records or make clients resolve invalid destinations.
- **FP:** Record serialization is generated from the TEP-81 TL-B schema and unsupported categories or flags are rejected or ignored safely.

**TP13. TEP:85 SBT Transferability Bypass**

- **D:** SBT contracts that implement TEP-62 but allow `transfer` to succeed, expose an alternate ownership-transfer path, or keep a transferable wrapper owner can turn a non-transferable credential into a sellable asset.
- **FP:** Every transfer path is rejected or permanently disabled, and mint/wrapper assumptions cannot transfer effective ownership to a third party.

**TP14. TEP:85 SBT Ownership Proof or Owner Info Mismatch**

- **D:** `prove_ownership#04ded148` or `request_owner#d0c3bfea` handlers that do not authenticate the owner where required, corrupt `query_id`, `item_id`, `owner`, `initiator`, `revoked_at`, `content`, or arbitrary `data`, or misroute `dest` can forge or break credential proofs.
- **FP:** Proof and owner-info messages preserve request fields, authenticate the required sender, include revocation state, and treat `forward_payload` as untrusted data.

**TP15. TEP:85 SBT Destroy, Revoke, or Authority Semantics Mismatch**

- **D:** SBT `destroy#1f04537a`, `revoke#6f89f5e3`, `get_authority_address()`, or `get_revoked_time()` implementations that omit owner/authority checks, allow double revoke, fail to set owner/authority to null on destroy, or fail to return `addr_none` when no authority exists can leave credentials valid or revocable by the wrong party.
- **FP:** Owner and authority paths are disjoint, revocation is one-way and timestamped, destroyed SBTs clear authority/owner, and getters reflect the exact terminal state.

**TP16. TEP:89 Jetton Wallet Discovery Response Mismatch**

- **D:** `provide_wallet_address#2c76b973` handlers that ignore the required funding floor, derive wallets for unsupported workchains instead of returning `addr_none`, omit `owner_address` when `include_address` is true, corrupt `query_id`, or send a malformed `take_wallet_address#d1735400` response can mislead deposits and fake-wallet checks.
- **FP:** Discovery rejects underfunded requests, derives the same wallet as TEP-74, returns `addr_none` for invalid owners, and preserves `query_id` and optional owner inclusion exactly.

**TP17. TEP:91 Source Registry Code-Hash, Admin, or Quorum Bypass**

- **D:** Source registry contracts or verifiers that index by contract address instead of code hash, let non-admins update/remove verifiers, accept insufficient or duplicate verifier signatures, fail to bind signatures to `(code_hash, sources_json_url)`, or publish malformed `sources.json` can cause explorers to display unverified or fraudulent source code.
- **FP:** Code hash, verifier id, admin, quorum public keys, source URLs, compiler config, and signatures are all bound and verified before registry updates.

**TP18. TEP:122 Onchain Reveal Sender, Mode, Count, or Correlation Mismatch**

- **D:** Reveal collection or item handlers that accept reveal batches from non-owner/editor, accept NFT reveal requests from addresses not derived from the requested index, skip reveal-mode locking, do not enforce the gas floor, fail to decrement revealable count on success, or corrupt `query_id` / optional `new_content` can reveal wrong content or permanently lock reveal state.
- **FP:** Collection and item authenticate each other by derived address, enforce mode transitions `1 -> 2 -> 3` or rollback, preserve `query_id`, and reconcile count/content on failure.

**TP19. TEP:126 Compressed NFT Claim Proof or Item Data Mismatch**

- **D:** Compressed NFT collections that return item data without `metadata.owner`, `metadata.individual_content`, or `index`, provide a proof for a different item, do not cap list `count`, mismatch `last_index` or collection `address`, or accept invalid `claim#013a3ca6` proofs can mint NFTs to wrong owners or allow unauthorized claims.
- **FP:** Proofs bind item index, owner, individual content, and collection address; returned item data is bounded and consistent with collection state, and `claim` rejects invalid proofs before minting.

**TP20. TEP:160 Dispatch Queue Ordering Assumption**

- **D:** Protocols that assume strict relative ordering across messages from different accounts can break under Dispatch Queue behavior, where per-account lt order is preserved but cross-account branches can be delayed or interleaved.
- **FP:** The protocol uses a synchronization account, explicit phase/correlation state, or another ordering proof before consuming messages from different branches.

**TP21. TEP:467 Normalized Message Hash Misuse**

- **D:** Wallets, relayers, or contracts that compute normalized hashes without setting `src = addr_none`, `import_fee = 0`, empty `init`, and body-as-reference, or that treat normalized hash as globally unique outside the stated wallet/IGNORE_ERRORS/non-deleted-wallet conditions, can misattribute traces or accept replayed-looking identifiers as final proof.
- **FP:** Normalization exactly follows TEP-467 and the hash is used only as a trace identifier under the required uniqueness assumptions, not as sole authorization.

**TP22. TEP:526 Scaled UI Jetton Multiplier Mismatch**

- **D:** Scaled UI Jetton masters that omit `get_display_multiplier()`, return zero numerator or denominator, calculate displayed balances without `muldivr`, ignore metadata `decimals`, or convert user-entered display amounts with the wrong inverse ratio can make UIs show or send materially wrong amounts.
- **FP:** Multipliers are nonzero, getter values are authoritative, display and input amount transforms use the TEP-526 formulas, and on-chain accounting remains in unscaled base units.

**TP23. TEP:526 Display Multiplier Change Event Mismatch**

- **D:** Jetton masters that advertise TEP-526 and change display multiplier values without emitting `display_multiplier_changed#ac392598`, emit fields that do not match the post-transaction getter values, change values between event transactions, or omit the initialization event can make indexers and wallets display stale or misleading balances.
- **FP:** Every advertised multiplier change and initialization emits the external-out event with the exact post-transaction numerator and denominator, or the token does not claim event-driven TEP-526 compatibility.

**TP24. Multi-TEP Query ID, Reply, and Excess Correlation Loss**

- **D:** TEP-62, TEP-66, TEP-74, TEP-85, TEP-89, and TEP-122 require response messages to preserve request `query_id` and route excesses or reports to the expected destination. Replacing, dropping, or fabricating correlation fields can break refunds, callbacks, attribution, or reconciliation.
- **FP:** Every standard request-response path preserves `query_id`, expected sender/recipient, and outstanding request state until the reply is consumed.

**TP25. Multi-TEP Opcode, TL-B, Getter, or Send-Mode Schema Mismatch**

- **D:** Any TEP standard implementation that advertises compatibility but uses wrong opcodes, field order, field widths, signedness, optional tags, reference-vs-inline encoding, getter tuple order, response send-mode, or message prefix can misroute value or break clients.
- **FP:** Schemas are generated or tested against the exact TEP TL-B and getter definitions, and any intentional custom ABI is not advertised as standard-compatible.

**TP26. Multi-TEP Optional, Maybe, Either, or `addr_none` Handling Mismatch**

- **D:** TEP standards use optional payloads, `Maybe ^Cell`, `Either Cell ^Cell`, optional owner addresses, `addr_none`, null DNS records, and optional content cells. Treating legal absence as malformed, fabricating defaults, or ignoring tag bits can shift parsers, reject compatible peers, or spoof recipients.
- **FP:** Optional and tagged fields are decoded exactly, absence is handled per the specific TEP, and stricter closed-system assumptions are documented and unreachable from public standard peers.
