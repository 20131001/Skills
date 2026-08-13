# PositiveSecurity Runtime Audit Checklist

Source: `PositiveSecurity/ton-audit-guide@55827b9b4201f1b12a4586ad8bf66743feb4dadd`, `README.md`.

Every row is a mandatory runtime checklist item. Coverage bundle generation must route each item by `family` and `language`; coverage agents must emit one matching `SRC-CHECK:` result. The text is retained with its upstream line number so refreshes are diffable.

| ID | Upstream line | Family | Language | Checklist item |
|---|---:|---|---|---|
| PS-L11 | 11 | assurance-evidence | shared | The contract language and compiler version are fixed: Tolk / FunC / Tact. |
| PS-L12 | 12 | assurance-evidence | shared | TVM/version assumptions, toolchain version are fixed. |
| PS-L13 | 13 | assurance-evidence | shared | Commit hash, build commands, compiler flags, stdlib imports, generated wrappers, ABI, and source maps are fixed. |
| PS-L14 | 14 | assurance-evidence | tolk | For Tolk: version changelog changes are reviewed, especially `lazy`, `createMessage`, `BounceMode`, `address`, `array<T>`, `string`, `unknown`, ABI, wrappers, and source maps. |
| PS-L15 | 15 | parser-standards | func | For FunC: manual serialization, `impure`, modifying/non-modifying methods, storage packing, `set_data`, and `set_code` are reviewed. |
| PS-L16 | 16 | assurance-evidence | tact | For Tact: generated wrappers, traits, receivers, fallback behavior, and bounced-message behavior are fixed. |
| PS-L17 | 17 | economic-protocol | shared | A list of external contracts, minter/wallet code cells, trusted contracts, oracle/indexer assumptions, and off-chain services is prepared. |
| PS-L18 | 18 | economic-protocol | shared | A list of roles is prepared: admin, owner, governance, operator, wallet user, relayer, indexer, validator, attacker. |
| PS-L19 | 19 | assurance-evidence | shared | Tool availability and language support are checked per project; no static or symbolic tool is treated as complete proof of safety. |
| PS-L25 | 25 | assurance-evidence | shared | All message-flow diagrams are drawn: internal, external, bounced, deployment, upgrade. |
| PS-L26 | 26 | lifecycle-finality | shared | A complete call/message graph is built: all `recv_internal`, `recv_external`, `onInternalMessage`, `onExternalMessage`, bounce/fallback/get handlers, deployment, upgrade, and child-contract paths. |
| PS-L27 | 27 | parser-standards | shared | For every handler, the following are documented: authorization, parsed fields, state writes, outbound sends, gas/value assumptions, possible bounce, and possible action-phase failure. |
| PS-L28 | 28 | parser-standards | shared | For every operation, value-flow is described: who pays gas, who receives TON/Jetton/NFT, where excesses/refunds are returned. |
| PS-L29 | 29 | parser-standards | shared | For every operation, state transitions are described: pending → sent → success/bounce/finalized/cancelled. |
| PS-L30 | 30 | economic-protocol | shared | Invariants are defined: total supply, balances, reserves, ownership, fee accounting, LP supply, debt, pending withdrawals, locked funds, pending claims. |
| PS-L31 | 31 | lifecycle-finality | shared | Invariants are verified under partial execution and interleaving of several message chains. |
| PS-L32 | 32 | parser-standards | shared | The contract does not assume synchronous getter calls to another contract. In TON, on-chain interaction between contracts happens through messages. |
| PS-L33 | 33 | surface-auth | shared | Every step of a multi-message flow revalidates state and does not trust a check made in a previous message. |
| PS-L34 | 34 | accounting-math | shared | Downstream failure does not lead to irreversible loss of funds, incorrect accounting, or a permanently stuck state. |
| PS-L35 | 35 | economic-protocol | shared | Protocol assumptions are separated from implementation assumptions: validators, oracles, indexers, relayers, bridges, and wallets are not silently trusted. |
| PS-L41 | 41 | lifecycle-finality | shared | All entry points are found: `recv_internal`, `recv_external`, `onInternalMessage`, `onExternalMessage`, `onBouncedMessage`, empty receiver, fallback, get methods. |
| PS-L42 | 42 | parser-standards | shared | For each entry point, it is clear who can call it, which fields are parsed, and which checks must happen before state is changed. |
| PS-L43 | 43 | lifecycle-finality | shared | Entry-point inventory includes hidden paths through fallback, empty messages, bounced messages, deployment messages, upgrade messages, and plugin/callback-like flows. |
| PS-L44 | 44 | surface-auth | shared | Empty body is handled intentionally: top-up/deploy/cashback or reject. |
| PS-L45 | 45 | economic-protocol | shared | Unknown opcode is handled explicitly: reject or intentional ignore; silent accept is forbidden without a clear reason. |
| PS-L46 | 46 | parser-standards | shared | Unknown opcode handling does not change value-flow in a dangerous way and does not allow griefing through accepted junk messages. |
| PS-L47 | 47 | parser-standards | shared | Bounced messages are not parsed as ordinary internal messages. |
| PS-L48 | 48 | lifecycle-finality | func | For FunC: the bounced flag in `int_msg_info` is checked before parsing op/body. |
| PS-L49 | 49 | parser-standards | tolk | For Tolk: the incoming message union is matched exhaustively; the `else` branch does not silently accept unknown opcodes. |
| PS-L50 | 50 | lifecycle-finality | tact | For Tact: fallback receivers and bounced receivers cover the expected message types. |
| PS-L51 | 51 | economic-protocol | shared | Get methods are checked for unsafe assumptions if their output is consumed by off-chain services, UIs, indexers, or relayers. |
| PS-L57 | 57 | surface-auth | shared | Every state-changing handler has an explicit caller/identity policy: privileged paths restrict `sender` / `in.senderAddress`, while permissionless paths still validate all sender-derived assumptions. |
| PS-L58 | 58 | economic-protocol | shared | Admin/governance functions have strict authorization, preferably multisig/timelock for upgrade/drain operations. |
| PS-L59 | 59 | accounting-math | shared | Blast radius is reviewed: whether a single admin key can withdraw all funds, change code, replace wallet code, change fee receiver, pause withdrawals, or alter accounting. |
| PS-L60 | 60 | lifecycle-finality | shared | Privileged operations cannot be reached through fallback, bounced, empty, deployment, plugin, callback, or upgrade paths. |
| PS-L61 | 61 | surface-auth | shared | For Jetton/NFT: sender is not just “some wallet”; it is the computed expected wallet/collection/minter/item address. |
| PS-L62 | 62 | surface-auth | shared | Forwarded addresses, payload fields, and notification bodies are not trusted as identity unless the sender contract is verified. |
| PS-L63 | 63 | surface-auth | shared | For workchain-sensitive logic: workchain is validated explicitly. |
| PS-L64 | 64 | accounting-math | shared | External messages: signature, seqno, valid_until, subwallet_id, and chain separation are checked before `accept_message()` / `acceptExternalMessage()`. |
| PS-L65 | 65 | lifecycle-finality | shared | Signed data includes recipient, amount, op, query_id/seqno, valid_until, contract address, workchain, and protocol-specific intent. There is no partial signing. |
| PS-L66 | 66 | lifecycle-finality | shared | Seqno/nonce is updated so replay is impossible even under action-phase failure. |
| PS-L67 | 67 | surface-auth | shared | Replay between contracts, cross-wallet replay, cross-workchain replay, signature reuse, and front-running are checked. |
| PS-L68 | 68 | surface-auth | shared | Any `null` / zero / default address state is reviewed so it cannot accidentally become a privileged identity. |
| PS-L74 | 74 | surface-auth | shared | There is no unconditional `accept_message()` / `acceptExternalMessage()`. |
| PS-L75 | 75 | surface-auth | shared | Cheap checks are performed before accept: format, op, seqno, valid_until, signature, subwallet, and basic bounds. |
| PS-L76 | 76 | storage-gas | shared | Heavy parsing, dictionary traversal, signature-independent hashing, and dynamic allocations do not happen before cheap rejection where avoidable. |
| PS-L77 | 77 | storage-gas | shared | After accept, there are no unbounded loops, heavy parsing, attacker-controlled batch sends, or unbounded dictionary operations. |
| PS-L78 | 78 | parser-standards | shared | For wallet-like contracts, the order is verified: parse → time check → seqno → signature → accept → update seqno → commit → actions. |
| PS-L79 | 79 | surface-auth | tolk | For Tolk, the use of `acceptExternalMessage()` and, where needed, `commitContractDataAndActions()` is reviewed. |
| PS-L80 | 80 | surface-auth | tolk | `commitContractDataAndActions()` placement does not create replay, committed-bad-state, or action-failure bugs. |
| PS-L81 | 81 | surface-auth | shared | External messages do not rely on randomness. |
| PS-L82 | 82 | assurance-evidence | shared | Repeated external messages, old seqno, expired `valid_until`, wrong subwallet, and wrong recipient are tested. |
| PS-L88 | 88 | parser-standards | shared | For each multi-message flow, the following scenarios are considered: delayed message, message lost because of action failure, bounce, duplicate, retry, timeout, cancellation, and parallel flow. |
| PS-L89 | 89 | parser-standards | shared | There is no logic like “balance was checked in step 1, therefore it is still the same in step 3.” |
| PS-L90 | 90 | surface-auth | shared | The carry-value pattern is used: value is passed in the message, not requested synchronously from another contract. |
| PS-L91 | 91 | lifecycle-finality | shared | State changes before outbound send are either reversible through bounce/recovery or do not break invariants on failure. |
| PS-L92 | 92 | lifecycle-finality | shared | Partial execution is checked with send modes in mind: with ignored/suppressed action errors, especially `+2` / `SendIgnoreErrors`, compute-phase state and some actions may be committed while one or more outbound sends are skipped. |
| PS-L93 | 93 | lifecycle-finality | shared | Pending states have timeout/cancel/retry/recovery paths. |
| PS-L94 | 94 | economic-protocol | shared | Idempotency: a repeated or late message does not credit reward, withdrawal, mint, claim, or refund twice. |
| PS-L95 | 95 | lifecycle-finality | shared | Query IDs are unique within the relevant in-flight/pending-operation scope, or the protocol explicitly proves that reuse is harmless. |
| PS-L96 | 96 | lifecycle-finality | shared | Query IDs are checked together with sender, expected operation, pending state, and amount; they are not used as the only trust anchor. |
| PS-L97 | 97 | economic-protocol | shared | Parallel deposits/withdrawals/liquidations/swaps/mints/burns/claims do not overwrite shared variables incorrectly. |
| PS-L98 | 98 | assurance-evidence | shared | Multi-step flows are tested in different interleavings, including success → bounce, bounce → retry, duplicate response, and late response after cancellation. |
| PS-L104 | 104 | lifecycle-finality | shared | All bounceable outgoing messages have a corresponding bounce handler, or it is explicitly proven that ignoring bounce is safe. |
| PS-L105 | 105 | economic-protocol | shared | The bounce handler restores state: balances, supply, locked amount, pending status, debt, LP accounting, claim status, or protocol-specific reserved value. |
| PS-L106 | 106 | lifecycle-finality | shared | Jetton mint/transfer/burn: failed `internal_transfer`, transfer, mint, or burn notification restores totalSupply/balance/pending accounting where applicable. |
| PS-L107 | 107 | lifecycle-finality | tolk,tact | If the bounce body is truncated, critical fields are located in the first bits; for Tact, the 224 useful bits limit is considered; for Tolk, the correct `BounceMode` is selected. |
| PS-L108 | 108 | parser-standards | tolk | For Tolk: `BounceMode.NoBounce`, `Only256BitsOfBody`, `RichBounce`, and `RichBounceOnlyRootCell` are selected intentionally. |
| PS-L109 | 109 | parser-standards | shared | Rich/new bounce-body schema compatibility is reviewed where relevant: root-cell-only vs full-body bounce, reserved/unknown fields, original body, phase, exit code, gas used, and parser behavior for future schema changes. |
| PS-L110 | 110 | lifecycle-finality | shared | Different bounce modes are not mixed without explicit parsing logic. |
| PS-L111 | 111 | lifecycle-finality | shared | Bounced messages are not used as a source of trusted authorization data without context verification. |
| PS-L112 | 112 | lifecycle-finality | shared | The bounce handler does not trust body fields without checking sender, query_id, expected pending state, expected operation, and expected amount/value. |
| PS-L118 | 118 | parser-standards | shared | All send sites are listed separately: destination, value, bounce flag, send mode, body, stateInit, deploy behavior, and follow-up assumptions. |
| PS-L119 | 119 | lifecycle-finality | shared | All reserve actions (`RAWRESERVE`, `nativeReserve`, or language equivalents) are listed like send sites: amount, reserve mode, ordering relative to sends, bounce-on-fail behavior, and failure impact. |
| PS-L120 | 120 | surface-auth | shared | There are no magic numbers for flags/modes; named constants are used. |
| PS-L121 | 121 | surface-auth | shared | `mode=64`, `mode=128`, `+1`, `+2`, `+16`, `+32`, and their combinations are checked. |
| PS-L122 | 122 | lifecycle-finality | shared | For outgoing external/log messages, only modes permitted for external messages are used; `+16` is avoided unless explicitly justified, because there is no ordinary sender to receive a bounce. |
| PS-L123 | 123 | surface-auth | shared | `+2` and `+16` interaction is reviewed: `+16` matters only for failures not suppressed by `+2`; `+2` ignores many, but not all, send/action errors. |
| PS-L124 | 124 | lifecycle-finality | shared | `+2` does not mask a critical failure after which state remains inconsistent; non-ignored cases such as malformed messages, invalid StateInit libraries, invalid external-message modes, and invalid mode combinations are tested where relevant. |
| PS-L125 | 125 | surface-auth | shared | `SendRemainingValue` / carry remaining value does not break later logic and does not make subsequent sends meaningless. |
| PS-L126 | 126 | lifecycle-finality | shared | `mode=128 + 32` / account destruction is available only to authorized flows and only when there are no pending obligations. |
| PS-L127 | 127 | accounting-math | shared | Carry-all-balance and account-destruction paths are reviewed as drain paths, not as ordinary sends. |
| PS-L128 | 128 | lifecycle-finality | shared | Action-phase failure is reviewed separately: under ignored/suppressed action errors, compute state may be applied while some outbound messages are skipped; otherwise the transaction may roll back depending on the failure and mode. |
| PS-L129 | 129 | storage-gas | shared | The number of actions does not exceed limits; batch loops are bounded. |
| PS-L130 | 130 | storage-gas | shared | No message send depends on a user-controlled unbounded loop, list, map, or dictionary traversal. |
| PS-L131 | 131 | lifecycle-finality | shared | State after failed action is either still safe or recoverable through a later message/bounce/retry path. |
| PS-L132 | 132 | surface-auth | shared | External outgoing logs/events are not used as a source of on-chain truth. |
| PS-L138 | 138 | storage-gas | shared | Gas is measured for each handler on the worst-case path. |
| PS-L139 | 139 | storage-gas | shared | `msg_value` covers compute fee, forward fee, action fee, storage reserve, deployment fee for child contracts, and excess return. |
| PS-L140 | 140 | parser-standards | shared | Fee accounting includes forward fee, action fee, storage reserve, deploy fee, excesses, refunds, and protocol-specific service fees. |
| PS-L141 | 141 | storage-gas | shared | Storage fees are accounted for; the contract maintains a positive reserve. |
| PS-L142 | 142 | storage-gas | shared | Reservation logic is tested: exact reserve, reserve-all-except, reserve-at-most, insufficient balance, action-list limit, and interaction with later sends. |
| PS-L143 | 143 | storage-gas | shared | Freeze/delete thresholds and storage debt scenarios are checked. |
| PS-L144 | 144 | lifecycle-finality | shared | There is no unbounded storage growth: maps/dicts/lists/history arrays/pending operations have a cap, pruning, pagination, rent mechanism, or sharding. |
| PS-L145 | 145 | parser-standards | shared | Attacker-controlled storage griefing is checked: fake pending operations, dust positions, spam deposits, metadata growth, and history growth. |
| PS-L146 | 146 | storage-gas | shared | Loops have a hard bound; dictionary traversal is limited. |
| PS-L147 | 147 | parser-standards | shared | Loop griefing is checked: attacker-controlled number of sends, iterations, refs, or parsed items cannot lead to denial of service. |
| PS-L148 | 148 | storage-gas | shared | There is no message sending from a user-controlled unbounded loop. |
| PS-L149 | 149 | storage-gas | shared | Excess gas is returned correctly, usually through `excesses` op `0xd53276db` or a protocol-specific equivalent. |
| PS-L150 | 150 | parser-standards | shared | Excesses/refunds return to the correct address and do not break accounting, replay protection, or pending-state logic. |
| PS-L151 | 151 | lifecycle-finality | shared | Returning excesses does not create reentrancy-like / race issues in the async model. |
| PS-L152 | 152 | assurance-evidence | shared | Fee constants are not hardcoded without tests; recalibration is needed when toolchain/network fees change. |
| PS-L158 | 158 | storage-gas | shared | Storage schema is documented and compatible with current on-chain data. |
| PS-L159 | 159 | parser-standards | shared | Every storage layout upgrade has a migration path and tests on the old state. |
| PS-L160 | 160 | parser-standards | func | For FunC: the order of `load_*` / `store_*` matches; `end_parse()` is used where needed. |
| PS-L161 | 161 | parser-standards | tolk,tact | For Tolk/Tact: auto-serialization is not bypassed with raw cells without a reason. |
| PS-L162 | 162 | parser-standards | shared | Cell size/depth limits are considered: 1023 bits / 4 refs per cell, message/c4/c5 depth limits. |
| PS-L163 | 163 | parser-standards | shared | After parsing, `slice` is fully consumed except for explicitly allowed payload refs. |
| PS-L164 | 164 | parser-standards | shared | Raw cell/slice parsing checks size, remaining bits, refs, type tags, opcodes, and `end_parse()` / assert-end behavior. |
| PS-L165 | 165 | surface-auth | shared | Extra payload tolerance is explicitly justified; disabled `assertEndAfterReading` / equivalent is treated as a red flag until proven safe. |
| PS-L166 | 166 | surface-auth | shared | Incorrect type handling is ruled out: uint written → uint read; address/internal/any address are used consistently. |
| PS-L167 | 167 | surface-auth | shared | Return values from functions that return a success flag are not ignored. |
| PS-L168 | 168 | storage-gas | shared | There is no storage collision, namespace collision, name shadowing, or confusing identifiers. |
| PS-L169 | 169 | parser-standards | shared | Storage schema, wallet code cells, minter code cells, and trusted code cells cannot be silently replaced with incompatible versions. |
| PS-L175 | 175 | surface-auth | shared | Signed integers are used only when truly needed. |
| PS-L176 | 176 | accounting-math | shared | All sums/balances/amounts are validated: `amount > 0`, `amount <= balance`, `amount <= max`. |
| PS-L177 | 177 | accounting-math | shared | There is no negative amount spoofing. |
| PS-L178 | 178 | accounting-math | shared | There is no division before multiplication where it creates precision loss. |
| PS-L179 | 179 | accounting-math | shared | Rounding direction is selected and documented: who receives dust. |
| PS-L180 | 180 | economic-protocol | shared | Fee/slippage math is checked for min/max, zero liquidity, dust, overflow/underflow, precision loss. |
| PS-L181 | 181 | parser-standards | shared | Sized integer serialization is checked: values do not exceed width. |
| PS-L182 | 182 | accounting-math | tolk | For Tolk: arithmetic on sized fields and writing back into `uint32/uint64/coins` is checked explicitly. |
| PS-L183 | 183 | surface-auth | tact | For Tact: `Int as uintX/intX` is checked against the expected range. |
| PS-L184 | 184 | surface-auth | shared | Unsafe casts from untrusted data have explicit range/type proofs. |
| PS-L185 | 185 | storage-gas | shared | Custom error codes do not use 0/1 and do not conflict with reserved ranges. |
| PS-L191 | 191 | surface-auth | shared | Randomness is not used for high-value decisions without commit-reveal or an equivalent scheme. |
| PS-L192 | 192 | surface-auth | shared | The contract does not use randomness in external receivers. |
| PS-L193 | 193 | lifecycle-finality | shared | Validator influence is considered: seed choice, message inclusion, ordering. |
| PS-L194 | 194 | surface-auth | shared | For low-value randomness, the language-specific initialization/randomize function is used correctly. |
| PS-L195 | 195 | lifecycle-finality | shared | Time-based logic (`now`, `valid_until`, deadlines) handles clock/ordering edge cases. |
| PS-L196 | 196 | accounting-math | shared | Expiration/cancel flows do not allow funds to be locked forever. |
| PS-L197 | 197 | parser-standards | shared | Commit-reveal schemes include domain separation, deadline, reveal validation, anti-replay, and safe refund/cancel paths. |
| PS-L203 | 203 | surface-auth | shared | Every `set_code`, `set_data`, `contract.setCodePostponed`, and `contract.setData` has strict authorization. |
| PS-L204 | 204 | surface-auth | tact | Tact direct `setData()` usage is audited for implicit end-of-receiver state save; manual state replacement branches intentionally terminate and are covered by tests. |
| PS-L205 | 205 | parser-standards | shared | New code hash/code cell is validated, preferably through allowlist/timelock/multisig. |
| PS-L206 | 206 | parser-standards | shared | Upgrade does not break storage layout; migration tests with the old data cell exist. |
| PS-L207 | 207 | parser-standards | shared | Upgrade corruption is checked: new version does not break storage layout, code hash assumptions, wallet code, minter code, trusted code cells, ABI, wrappers, or indexer assumptions. |
| PS-L208 | 208 | lifecycle-finality | shared | Timing is considered: code update is applied after the current execution/action phase; data replacement may be immediate depending on the primitive. |
| PS-L209 | 209 | economic-protocol | shared | Governance cannot bypass user rights to withdrawal/claim. |
| PS-L210 | 210 | surface-auth | shared | Emergency pause does not let the owner steal funds or permanently block users. |
| PS-L211 | 211 | lifecycle-finality | shared | Upgrade functions are not accessible through bounced/fallback/empty messages. |
| PS-L212 | 212 | surface-auth | shared | Third-party code / plugin / arbitrary code execution is absent, or external code cells and continuations are strictly validated and tested. |
| PS-L213 | 213 | surface-auth | shared | Admin key compromise scenarios are documented together with blast radius and recovery path. |
| PS-L221 | 221 | assurance-evidence | tolk | Tolk version and changelog are checked against the used features. |
| PS-L222 | 222 | assurance-evidence | tolk | ABI/source maps/wrappers are used for tests and review, but do not replace manual checking of TVM effects. |
| PS-L223 | 223 | parser-standards | tolk | Low-level assembler/Fift/raw cells are used only with justification and tests. |
| PS-L224 | 224 | assurance-evidence | tolk | Generated output visibility is checked: tests rebuild generated wrappers/output files before executing narrow test suites. |
| PS-L228 | 228 | surface-auth | tolk | `onInternalMessage` uses modern `InMessage`, not legacy manual parsing, unless there is a reason. |
| PS-L229 | 229 | parser-standards | tolk | `incomingMessages` union is complete and opcodes are unique. |
| PS-L230 | 230 | parser-standards | tolk | `lazy Union.fromSlice(in.body)` does not hide acceptance of an unknown opcode through a permissive `else`. |
| PS-L231 | 231 | surface-auth | tolk | Unknown or incomplete payloads do not pass into state-changing logic through a lazy union or permissive fallback branch. |
| PS-L232 | 232 | surface-auth | tolk | Empty message is intentionally ignored/cashbacked. |
| PS-L233 | 233 | lifecycle-finality | tolk | Bounced messages are handled in `onBouncedMessage` and are not mixed with internal messages. |
| PS-L237 | 237 | surface-auth | tolk | Security-critical lazy fields are actually read before the check. |
| PS-L238 | 238 | parser-standards | tolk | There is no “lazy validation bypass”: a field exists in a struct but is not accessed, so validation/deserialization never happens. |
| PS-L239 | 239 | surface-auth | tolk | Delayed validation does not allow state write / accept / send before the critical field has been read and validated. |
| PS-L240 | 240 | accounting-math | tolk | Partial update through lazy does not break invariants and does not overwrite fields incorrectly. |
| PS-L241 | 241 | surface-auth | tolk | `assertEndAfterReading` / equivalent is not disabled without a reason. |
| PS-L245 | 245 | surface-auth | tolk | `address` is used for internal addresses; `any_address` is used only when external/none addresses are truly allowed. |
| PS-L246 | 246 | surface-auth | tolk | External/none addresses do not pass where an internal address is expected. |
| PS-L247 | 247 | surface-auth | tolk | There are no unsafe `as` casts from untrusted `slice/cell/builder`. |
| PS-L248 | 248 | surface-auth | tolk | Nullable values are not force-unwrapped (`!`) without a prior null check. |
| PS-L249 | 249 | accounting-math | tolk | `null` is not used as a valid privileged state without a clear invariant. |
| PS-L250 | 250 | parser-standards | tolk | `unknown`, `builder`, `slice`, and `cell` are not used to bypass type safety or hide untrusted raw payload. |
| PS-L251 | 251 | parser-standards | tolk | `Cell<T>` / typed cells match the expected TL-B layout. |
| PS-L255 | 255 | parser-standards | tolk | `createMessage()` is used instead of raw building unless there is a proxy/raw reason. |
| PS-L256 | 256 | parser-standards | tolk | Compiler-managed serialization is used where raw body layout is not required. |
| PS-L257 | 257 | parser-standards | tolk | `body: obj.toCell()` is not passed when the compiler should be allowed to choose inline/ref; `body: obj` is passed instead. |
| PS-L258 | 258 | parser-standards | tolk | `UnsafeBodyNoRef` is used only after a size proof. |
| PS-L259 | 259 | surface-auth | tolk | `sendRawMessage` is allowed only for validated raw/proxy messages. |
| PS-L260 | 260 | parser-standards | tolk | `AutoDeployAddress` / StateInit: code+data match the expected address, shard targeting is justified. |
| PS-L261 | 261 | parser-standards | tolk | StateInit, deploy address, wallet code, owner address, workchain, and salt/domain separation are checked. |
| PS-L265 | 265 | lifecycle-finality | tolk | For stateful sends, a bounce mode with enough recovery data is selected. |
| PS-L266 | 266 | lifecycle-finality | tolk | `RichBounce` is used where full original body / exitCode / gasUsed is needed. |
| PS-L267 | 267 | parser-standards | tolk | `Only256BitsOfBody` / root-cell-only behavior is sufficient for the parser and recovery logic. |
| PS-L268 | 268 | lifecycle-finality | tolk | `NoBounce` is used only when loss of the downstream message is safe. |
| PS-L269 | 269 | parser-standards | tolk | Bounce parser compatibility is tested for selected bounce mode and malformed/truncated body. |
| PS-L273 | 273 | surface-auth | tolk | `commitContractDataAndActions()` is not called before validation is complete. |
| PS-L274 | 274 | surface-auth | tolk | Its placement is checked for replay protection and action failure. |
| PS-L275 | 275 | surface-auth | tolk | Committed state cannot become permanently wrong if an outbound action later fails. |
| PS-L276 | 276 | surface-auth | tolk | Global variables are not used as persistent state. |
| PS-L282 | 282 | surface-auth | func | All state-changing / send / random / `set_code` / `set_data` functions have `impure`. |
| PS-L283 | 283 | surface-auth | func | There is no confusion between modifying (`~`) and non-modifying (`.`) methods. |
| PS-L284 | 284 | parser-standards | func | Storage is manually parsed/packed in the same order everywhere. |
| PS-L285 | 285 | parser-standards | func | `end_parse()` is used for payload/storage where applicable. |
| PS-L286 | 286 | storage-gas | func | Redeclaration/shadowing does not mask storage variables or return values. |
| PS-L287 | 287 | surface-auth | func | `accept_message()` is placed only after cheap validation. |
| PS-L288 | 288 | lifecycle-finality | func | `send_raw_message` modes are documented and tested. |
| PS-L289 | 289 | surface-auth | func | `set_code`/`set_data` are gated and tested; data replacement is reviewed for immediate-vs-postponed effects and migration safety. |
| PS-L290 | 290 | storage-gas | func | Third-party code execution is absent or code is validated; the `COMMIT` + out-of-gas scenario is analyzed. |
| PS-L291 | 291 | assurance-evidence | func | Manual raw-cell parsing has tests for malformed slices, missing refs, extra refs, oversized payloads, and trailing garbage. |
| PS-L297 | 297 | assurance-evidence | tact | For Tact projects: although TON Docs mark Tact as deprecated in favor of Tolk, existing Tact contracts must still be audited against the current Tact compiler, generated wrappers, and dedicated Tact documentation. |
| PS-L298 | 298 | lifecycle-finality | tact | Receiver coverage: `receive`, `external`, `bounced`, fallback. |
| PS-L299 | 299 | parser-standards | tact | Bounced payload limit is considered; important fields are first or a fallback Slice parser is used. |
| PS-L300 | 300 | accounting-math | tact | Traits do not allow bypassing auth/pausable/stoppable invariants. |
| PS-L301 | 301 | surface-auth | tact | Optional values are used intentionally; there are no unsafe nullable patterns. |
| PS-L302 | 302 | surface-auth | tact | Native/asm functions are audited manually. |
| PS-L303 | 303 | surface-auth | tact | Direct `setData()` usage is treated as dangerous: every branch using manual state replacement prevents Tact's implicit final state save from overwriting or corrupting the intended data. |
| PS-L304 | 304 | accounting-math | tact | `myBalance()` / context balance assumptions are not used after sends as the actual live balance. |
| PS-L305 | 305 | lifecycle-finality | tact | After sends, stale assumptions about contract balance are not used for later accounting decisions. |
| PS-L306 | 306 | parser-standards | tact | Exit codes, default codes, and generated wrappers are checked. |
| PS-L307 | 307 | storage-gas | tact | Gas best practices do not conflict with security best practices. |
| PS-L315 | 315 | surface-auth | shared | Jetton wallet authenticity is checked by recomputing the address from master/wallet code/owner. |
| PS-L316 | 316 | surface-auth | shared | Sender of transfer notification is checked. |
| PS-L317 | 317 | accounting-math | shared | Fake Jetton deposit is checked: payload may look correct, but sender must be the expected Jetton wallet for master+owner. |
| PS-L318 | 318 | lifecycle-finality | shared | Untrusted notification is checked: `transfer_notification` / callback must come from the expected wallet/contract. |
| PS-L319 | 319 | lifecycle-finality | shared | Minter mint flow handles bounce and corrects totalSupply. |
| PS-L320 | 320 | lifecycle-finality | shared | Wallet transfer handles bounce and restores balance. |
| PS-L321 | 321 | lifecycle-finality | shared | Burn flow handles notification/bounce/failure and keeps totalSupply and wallet balance consistent. |
| PS-L322 | 322 | parser-standards | shared | Forward payload/ref constraints are checked. |
| PS-L323 | 323 | parser-standards | shared | TEP-74 message formats/opcodes/query_id are followed. |
| PS-L327 | 327 | surface-auth | shared | Ownership transfer authorization is checked. |
| PS-L328 | 328 | surface-auth | shared | Collection/item address derivation is checked. |
| PS-L329 | 329 | economic-protocol | shared | Fake NFT transfer is checked: item address must be derived from collection/index/code and not accepted only from payload claims. |
| PS-L330 | 330 | economic-protocol | shared | Mint indexing cannot be race-attacked or skipped. |
| PS-L331 | 331 | parser-standards | shared | Metadata mutability/admin controls are checked. |
| PS-L332 | 332 | surface-auth | shared | Random traits use a safe randomness model. |
| PS-L336 | 336 | parser-standards | shared | False deposit/top-up attack is checked: inbound transfer + outbound refund/bounce correlation. |
| PS-L337 | 337 | economic-protocol | shared | Slippage/min_out/deadline are applied on every path. |
| PS-L338 | 338 | accounting-math | shared | Fee collection happens on every path. |
| PS-L339 | 339 | economic-protocol | shared | LP supply math is symmetric for add/remove liquidity. |
| PS-L340 | 340 | economic-protocol | shared | Oracle/indexer assumptions are documented; stale price is handled. |
| PS-L341 | 341 | economic-protocol | shared | Bridge messages include domain separation, source chain, destination, nonce, amount, token id. |
| PS-L342 | 342 | lifecycle-finality | shared | Pending claims cannot be double-spent through parallel traces. |
| PS-L343 | 343 | economic-protocol | shared | DEX/vault/lending race scenarios are tested: slippage, deadline, min_out, fee, liquidity, reserves, debt, and liquidation state are preserved under parallel traces. |
| PS-L344 | 344 | economic-protocol | shared | Refund/excess logic cannot be used to fake a deposit, cancel accounting, or bypass slippage/deadline checks. |
| PS-L350 | 350 | assurance-evidence | shared | Unit tests cover all handlers, including empty/unknown/bounced/fallback. |
| PS-L351 | 351 | assurance-evidence | shared | Integration tests cover full multi-contract traces. |
| PS-L352 | 352 | assurance-evidence | shared | There are explicit tests for action-phase failure, bounce, insufficient funds, invalid destination, oversized message, malformed cell, wrong sender, and wrong workchain. |
| PS-L353 | 353 | lifecycle-finality | shared | Race-condition simulations: two flows interleaved in different orders. |
| PS-L354 | 354 | assurance-evidence | shared | Replay tests: same external message twice, old seqno, expired valid_until, wrong subwallet, wrong contract address, wrong op, and wrong workchain. |
| PS-L355 | 355 | assurance-evidence | shared | Gas snapshots exist for each handler; worst-case path is tested. |
| PS-L356 | 356 | assurance-evidence | shared | Storage migration tests run with the previous deployed data layout. |
| PS-L357 | 357 | assurance-evidence | shared | Fuzzing/mutation tests cover message bodies, amounts, opcodes, slices, refs, map sizes, lazy fields, and send modes. |
| PS-L358 | 358 | assurance-evidence | tolk | Tools are considered: Misti, TON Symbolic Analyzer, tolk-less, Universalmutator, custom linters. |
| PS-L359 | 359 | assurance-evidence | shared | Tool output is reviewed manually; coverage is checked, but is not treated as proof of security. |
| PS-L360 | 360 | assurance-evidence | shared | For each high-impact scenario, tests include success path, failure path, bounce path, insufficient funds path, replay path, and duplicate/late message path where relevant. |
| PS-L361 | 361 | assurance-evidence | shared | The audit report marks which items were checked by code review, which by tests, which by tooling, and which only by manual reasoning. |
| PS-L369 | 369 | surface-auth | shared | `accept_message()` / `acceptExternalMessage()` is called before signature/seqno/valid_until checks. |
| PS-L370 | 370 | accounting-math | shared | `mode=128`, `mode=128+32`, or carry-all-balance is used outside strict authorization. |
| PS-L371 | 371 | lifecycle-finality | shared | A bounce handler is missing for a stateful outgoing message. |
| PS-L372 | 372 | lifecycle-finality | shared | State is updated before outbound send and there is no recovery on bounce/action failure. |
| PS-L373 | 373 | parser-standards | shared | Unknown opcode is silently accepted. |
| PS-L374 | 374 | surface-auth | shared | A state-changing handler has no explicit caller/identity policy. |
| PS-L375 | 375 | surface-auth | shared | Sender identity is taken from payload/forwarded address instead of the actual verified sender contract. |
| PS-L376 | 376 | accounting-math | shared | Jetton deposit trusts sender without recomputing the expected wallet address. |
| PS-L377 | 377 | surface-auth | shared | NFT transfer trusts payload without deriving the expected item address. |
| PS-L378 | 378 | surface-auth | func | `set_code` / `set_data` / `contract.setCodePostponed` are used without multisig/timelock/allowlist. |
| PS-L379 | 379 | parser-standards | shared | Raw `sendRawMessage` / raw cell parsing is used without tests. |
| PS-L380 | 380 | surface-auth | shared | Unsafe `as` cast or nullable force unwrap is applied to untrusted data. |
| PS-L381 | 381 | parser-standards | shared | `unknown`, `slice`, `cell`, or `builder` is used to bypass type safety without strict parsing. |
| PS-L382 | 382 | storage-gas | shared | Unbounded loop/storage growth. |
| PS-L383 | 383 | surface-auth | shared | Randomness is used for a high-value outcome without commit-reveal. |
| PS-L384 | 384 | lifecycle-finality | shared | Ignored/suppressed action-phase failure can leave state inconsistent, or the code incorrectly assumes that action failure always rolls back state. |
| PS-L385 | 385 | surface-auth | shared | `assertEndAfterReading` is disabled / extra payload is accepted without a reason. |
| PS-L386 | 386 | lifecycle-finality | shared | A privileged operation can be reached through fallback, bounce, empty message, upgrade, or plugin-like path. |
| PS-L387 | 387 | assurance-evidence | shared | Tests use generated wrappers/output files without rebuilding them after source changes. |
| PS-L393 | 393 | assurance-evidence | shared | Message-flow diagrams. |
| PS-L394 | 394 | assurance-evidence | shared | Value-flow diagrams. |
| PS-L395 | 395 | surface-auth | shared | Complete call/message graph. |
| PS-L396 | 396 | parser-standards | shared | Storage layout and migration notes. |
| PS-L397 | 397 | surface-auth | shared | List of trusted roles and admin powers. |
| PS-L398 | 398 | parser-standards | shared | Send-site table: dest/value/mode/bounce/body/stateInit. |
| PS-L399 | 399 | lifecycle-finality | shared | Entry-point table: caller/auth/state changes/outgoing messages/gas/bounce/action-failure assumptions. |
| PS-L400 | 400 | storage-gas | shared | Gas table for each handler. |
| PS-L401 | 401 | assurance-evidence | shared | Test matrix and missing tests. |
| PS-L402 | 402 | accounting-math | shared | Exploit scenarios or “not applicable” rationale for high-impact checklist areas. |
| PS-L403 | 403 | accounting-math | shared | Known accepted risks and rationale. |
| PS-L404 | 404 | assurance-evidence | shared | Classification of evidence: checked by code review, checked by tests, checked by tooling, or checked only by manual reasoning. |
| PS-L410 | 410 | lifecycle-finality | shared | `+2` / `SendIgnoreErrors` ignores many send/action errors but does not make every malformed or invalid action safe. The non-ignored cases are part of the test matrix. |
| PS-L411 | 411 | lifecycle-finality | shared | `+16` / bounce-on-action-failure is meaningful only for failures that are not suppressed by `+2`; do not describe `+2 +16` as two independent protections. |
| PS-L412 | 412 | lifecycle-finality | shared | Action-phase failure is not a single outcome: distinguish rollback, skipped action, ignored error, bounce, and successful later actions. |
| PS-L413 | 413 | lifecycle-finality | shared | `query_id` is a correlation key, not an authorization primitive. Its uniqueness requirement is scoped to the relevant pending/in-flight operation unless a standard or protocol requires more. |
| PS-L414 | 414 | surface-auth | shared | Permissionless state-changing handlers are allowed, but only when their caller/identity policy is explicit and all sender-derived assumptions are verified. |
| PS-L415 | 415 | surface-auth | tact | Tact `setData()` is exceptional and dangerous; audit it together with Tact's implicit receiver-end state save. |
| PS-L416 | 416 | lifecycle-finality | shared | Reserve actions are first-class action-phase effects and must be reviewed like sends. |

