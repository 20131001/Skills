# Tolk Language-Specific Attack Vectors

These vectors depend on Tolk typed parsers, lazy loading, storage helpers, serialization, or control flow. They are bundled only for Tolk targets.

---

**TL1. Signed Integer Abuse in Asset or Voting Logic**

- **D:** Tolk integer fields or casts used for balances, shares, quotas, voting power, or message amounts can admit negative or boundary values when the business domain expects unsigned values.
- **FP:** Negative values are required and strict range validation prevents misuse.

**TL2. Ignored Result Flags or Optional Results from Mutating Helpers**

- **D:** Ignoring success flags, nullable results, or failure indicators from dictionary/storage/system helpers can continue after a failed lookup, mutation, or low-level operation.
- **FP:** Failure is intentionally handled later, the operation is optional/idempotent, or failure cannot affect security-relevant state.

**TL3. Incomplete Typed or Manual Parser Consumption**

- **D:** Tolk typed message/storage decoders, lazy-loaded fields, custom serializers, or manual slice parsers that do not prove exact layout can accept malformed trailing bits/refs or fail after earlier state effects.
- **FP:** The type explicitly supports extensions, all lazy/fallible fields are accessed before state effects, or the parser proves exact bit-and-ref shape before values influence state or outbound messages.

**TL4. Typed Serialization / Storage Layout Mismatch or Misbinding**

- **D:** Tolk `Storage.load`, `Storage.save`, typed structs, field order, custom codecs, or manual builders can mismatch the authoritative TL-B/schema and corrupt state or messages.
- **FP:** The storage/message schema is centralized, segmented, and load/save ordering is provably consistent.

**TL5. Missing Termination After a Handled Branch**

- **D:** Router or opcode branches that complete their intended logic but continue into later default/error logic can revert state/actions or execute conflicting logic.
- **FP:** Fallthrough is intentional and proven not to reach a failing or conflicting path.

**TL6. Lazy Validation Bypass Before Security-Relevant Effects**

- **D:** A `lazy` struct/union field is never accessed, so malformed data or an invalid security-critical value is not decoded before state mutation, gas acceptance, commit, or send. A permissive union `else` can similarly accept unknown/incomplete payloads.
- **FP:** Every field that affects authorization, accounting, routing, or exact-layout validation is forced and checked before effects; extensible unread fields are explicitly harmless.

**TL7. Unsafe Raw Message Construction or Inline/Reference Assumption**

- **D:** `sendRawMessage`, `UnsafeBodyNoRef`, manual builders, or `body: obj.toCell()` bypass compiler-managed layout and make body placement, StateInit, flags, or size differ from the peer's expected TL-B schema.
- **FP:** Raw construction is required for a documented schema, exact size/ref limits are proven, and the resulting BoC is tested against every producer/consumer.

**TL8. Address-Kind, Cast, or Nullable-Type Bypass**

- **D:** `any_address`, unsafe `as` casts, `unknown`, raw `slice/cell/builder`, or forced nullable unwraps admit external/none addresses, malformed payloads, or absent values where an internal authenticated address or typed cell is required.
- **FP:** Address kind, nullability, range, and TL-B type are checked before use, with no fallible cast/unwrap after state or value effects.

**TL9. Unsafe `commitContractDataAndActions()` Placement**

- **D:** Tolk commits persistent data/actions before all fallible validation, parsing, reserve, or outbound-message construction is complete, preserving replay state, accounting, or queued actions in an unsafe intermediate configuration.
- **FP:** The checkpoint contains only deliberately final replay protection or another proven-safe state, and every post-commit failure leaves authorization, accounting, and recovery invariants intact.

**TL10. Global Variable Used as Persistent Contract State**

- **D:** A Tolk global is treated as durable across messages instead of loading/saving c4 state, causing authorization, counters, caches, or configuration to reset or diverge from persistent storage.
- **FP:** Globals are invocation-local by design and all authoritative state is loaded and saved through the declared storage schema.
