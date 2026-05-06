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
