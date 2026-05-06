# FunC Language-Specific Attack Vectors

These vectors depend on FunC syntax, parser APIs, storage helpers, or control-flow behavior. They are bundled only for FunC targets.

---

**FC1. Missing `impure` Modifier on Security-Critical Functions**

- **D:** A function that enforces checks, updates state, authorizes a sender, or throws only through side effects can be optimized away when its return value is unused unless marked `impure`.
- **FP:** The function is intentionally pure, has no security-relevant side effects, or every security-relevant call consumes the returned value rather than relying on an unused side-effecting call.

**FC2. Incorrect Method Call Syntax (`.` vs `~`)**

- **D:** Calling a mutating helper with `.` returns a modified copy but does not update the original variable; using `~` where a non-mutating copy was intended can overwrite the first argument. Either mistake can leave balances, dictionaries, counters, or parsers in the wrong state.
- **FP:** The original value is intentionally preserved or intentionally overwritten, and the returned value is stored or discarded consistently with that design.

**FC3. Signed / Unsigned Integer Domain Confusion**

- **D:** Mixing `load_int` / `store_int` and `load_uint` / `store_uint`, or accepting signed `int` values for balances, shares, quotas, voting power, timestamps, or amounts, can admit negative or boundary values where the business domain expects non-negative bounded integers.
- **FP:** Negative values are part of the protocol and strict range validation prevents arithmetic, voting, or accounting misuse.

**FC4. Storage Field Shadowing or Identifier Collision**

- **D:** A local variable, tuple component, helper parameter, or ambiguous FunC identifier can shadow an authoritative storage field or be parsed differently than intended, causing `save_data()` or security checks to use stale, attacker-controlled, or wrong values.
- **FP:** Naming rules, scoped unpacking, and call-site checks prove the authoritative storage field is the value being validated and saved.

**FC5. Ignored Return Values from Dictionary or System Operations**

- **D:** FunC/TVM helpers often return success flags; ignoring them can continue after failed delete, update, lookup, or low-level operations.
- **FP:** Failure is intentionally handled later, the operation is optional/idempotent, or failure cannot affect security-relevant state.

**FC6. Incomplete Message Parsing (`end_parse` Omission)**

- **D:** Fixed-layout slices, nested ref parsers, reused payload slices, or storage parsers that never call `end_parse()` or prove bit-and-ref emptiness can accept malformed trailing data.
- **FP:** Trailing extension data is explicitly supported, or equivalent full emptiness is enforced.

**FC7. Serialization / State Layout Mismatch or Misbinding**

- **D:** Large flat `load_data()` / `save_data()` patterns, helper argument swaps, inconsistent hardcoded widths/opcodes/send flags, or manual builders can corrupt ownership, balances, config, message shape, or parser alignment.
- **FP:** Storage and message layouts are segmented, constants/builders are authoritative, and call sites are provably consistent with the schema.

**FC8. Missing Early `return` After a Handled Branch**

- **D:** A handled branch that does not `return` can fall through into a default throw or conflicting branch, causing hidden revert or denial of service.
- **FP:** Fallthrough is intentional and cannot reach a failing/conflicting path.

**FC9. Bitwise NOT Used as Boolean Negation**

- **D:** FunC `~` is bitwise NOT, not logical negation. Conditions such as `if (~status)` are true for every value except `-1`, so paused flags, authorization flags, or success flags can be inverted incorrectly and expose protected branches.
- **FP:** The code intentionally uses TVM truthiness with `-1` as true and `0` as false, and the condition is proven equivalent to the intended guard.
