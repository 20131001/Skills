# Attack Vectors Reference — Additional Vectors (5/5)

> Part 5 · Vectors 101–107
> Covers: CPI ownership reassignment, deserialization, orphan accounts, discriminator matching, Token-2022 sizing, custom numeric ordering, and flag decoding

---

## V101 — Account Ownership Reassignment via CPI

**Detect:** `system_instruction::assign` or `system_instruction::allocate` in CPI where target account's signer privilege is forwarded from caller.

**Vulnerable:**
```rust
pub fn exploit(accounts: &[AccountInfo]) -> ProgramResult {
    let user_account = &accounts[0];  // signer privilege forwarded from caller
    let ix = system_instruction::assign(user_account.key, &malicious_program_id);
    invoke(&ix, &[user_account.clone()])?;  // steals ownership
    Ok(())
}
```

**Exploit:** Attacker's program receives forwarded signer privilege and reassigns account ownership. Attacker can then modify account data freely.

**Secure:**
```rust
// Never forward user signer to untrusted programs (V32)
// Post-CPI: require!(*account.owner == expected_program_id, OwnershipChanged);
```

---

## V102 — Unsafe Deserialization and Ambiguous Input Consumption

**Detect:** Manual slicing before bounds checks; allocation driven by unbounded length prefixes; `BorshDeserialize::deserialize` used without confirming the cursor was fully consumed; or custom decoders that accept truncated/trailing data ambiguously.

Do not flag `try_from_slice` solely because there is no separate `data.len()` check: it normally returns an error for malformed input and requires full input consumption. Trace the actual decoder behavior and an attacker-reachable panic, compute-exhaustion path, or parser differential.

**Vulnerable:**
```rust
let tag = data[0];                  // panics when data is empty
let mut cursor = &data[1..];
let input = MyInput::deserialize(&mut cursor)?;
// BUG: trailing bytes remain and are interpreted differently downstream
```

**Exploit:** Crafted input panics a critical instruction, forces excessive allocation, or passes one validation layer while a downstream parser interprets different data.

**Secure:**
```rust
require!(data.len() >= MIN_SIZE && data.len() <= MAX_SIZE, InvalidInputLength);
let input = MyInput::try_from_slice(data)?; // full-consumption decoder
```

---

## V103 — Orphan Account from Parent-Child Lifecycle

**Detect:** Parent account closeable while child accounts still reference it. No `active_children` counter or cascade-close logic.

**Vulnerable:**
```rust
pub fn close_pool(ctx: Context<ClosePool>) -> Result<()> {
    // UserStake accounts with seeds = [b"stake", pool.key()] still exist
    // Users can't unstake — pool gone, has_one = pool fails
    Ok(())
}
```

**Exploit:** Parent closed, child accounts orphaned. User funds locked permanently.

**Secure:**
```rust
require!(pool.active_positions == 0, PoolHasActivePositions);
```

---

## V104 — Partial Discriminator or Selector Matching

**Detect:** Code declares or receives a multi-byte selector/discriminator but validates only a prefix, suffix, low byte, or masked subset. Compare the validation width with the protocol's actual schema.

Do not flag a native program merely for intentionally defining a one-byte instruction enum. A one-byte tag is safe when it is the complete canonical schema, every value maps unambiguously, and the remaining payload is strictly decoded.

**Vulnerable:**
```rust
// Input length was validated before this point.
let supplied_selector = u64::from_le_bytes(data[..8].try_into()?);
require!((supplied_selector as u8) == EXPECTED_SELECTOR_LOW_BYTE, WrongType);
// High 56 bits are attacker-controlled but ignored.
```

**Exploit:** An attacker supplies a different full selector with the accepted partial value, reaching an unintended decoder, message route, or account type.

**Secure:**
```rust
let selector: [u8; 8] = data[..8].try_into()?;
require!(selector == EXPECTED_SELECTOR, WrongType);
```

---

## V105 — Dynamic Token-2022 Account Size via Extensions

**Detect:** Hardcoded `space = TokenAccount::LEN` (165 bytes) for Token-2022 accounts. Missing extension size calculation.

**Vulnerable:**
```rust
#[account(init, payer = user, space = TokenAccount::LEN)]
pub vault_token: InterfaceAccount<'info, TokenAccount>,
// Token-2022 with extensions needs more space — creation fails
```

**Exploit:** Cannot create token accounts for mints with extensions. DoS on deposits/withdrawals.

**Secure:**
```rust
let space = ExtensionType::try_calculate_account_len::<Token2022Account>(&required_extensions)?;
```

---

## V106 — Custom Numeric Ordering Mismatch

**Detect:** `Ord` or `PartialOrd` is derived for a custom numeric representation whose field order is not its mathematical order. Prioritize little-endian limb arrays, signed-magnitude values, fixed-point wrappers, and structs where metadata fields precede the numeric magnitude.

**Vulnerable:**
```rust
#[derive(Clone, Copy, Eq, Ord, PartialEq, PartialOrd)]
struct U256([u64; 4]); // little-endian: limb 0 is least significant

let balance = U256([u64::MAX, 0, 0, 0]); // 2^64 - 1
let amount  = U256([0, 1, 0, 0]);        // 2^64
require!(balance >= amount, InsufficientFunds);
// Derived lexicographic ordering compares limb 0 first and incorrectly passes.
```

**Exploit:** Incorrect comparison bypasses an underflow, health-factor, limit, price, or authorization guard and permits a value-moving operation that the true numeric ordering should block.

**Secure:** Implement semantic comparison explicitly, normally comparing the most-significant limb first, and test boundary pairs where lexicographic and numeric ordering disagree. Report only when the bad ordering reaches a security-relevant decision.

---

## V107 — Non-Canonical Flag Decoding / Parser Differential

**Detect:** Manual decoders accept non-canonical boolean or flag values; validate only selected low bits while preserving attacker-controlled reserved bits; or interpret the same flag byte differently across validation, signature/hash construction, and execution.

**Vulnerable:**
```rust
let approved = data[0] != 0; // accepts 0xFF as true
require!(approved, NotApproved);
let role = data[0];          // later parser treats 0xFF as privileged sentinel
apply_role(role)?;
```

**Exploit:** A non-canonical encoding passes one parser or signature check but has a stronger meaning in another, bypassing role, message-type, signer, writable, or replay validation.

**Secure:** Accept only canonical values (`0` or `1` for booleans), reject unknown/reserved bits, and use one decoder and one canonical byte representation across validation and execution. Non-canonical acceptance without a security-relevant parser difference is not a finding.
