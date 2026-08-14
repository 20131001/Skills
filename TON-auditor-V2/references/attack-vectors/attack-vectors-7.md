# Attack Vectors Reference (7/7) — Tolk Runtime, Bounce Recovery & Action Integrity

## V168 — Tolk Bitwise and Logical Operator Confusion

**What:** Security logic uses bitwise `&` / `|` where logical short-circuit `&&` / `||` is required, or assumes the right-hand side will not execute.

**Why it matters:** Both operands of bitwise expressions are evaluated and integer bit patterns are combined rather than reduced through boolean short-circuit semantics. A supposedly guarded parse, division, lookup, or side effect can execute on invalid input, while non-canonical truth values can select the wrong branch.

**What to look for:**
- `&` or `|` in authorization, null, bounds, parsing, or initialization conditions
- A dangerous right-hand expression that is expected to be skipped when the left condition decides the result
- Integer flags combined bitwise without explicit normalization to the intended boolean domain

---

## V169 — Uninitialized Tolk Global Read

**What:** A Tolk global is read before every reachable entry point assigns it, even though TVM globals begin as NULL rather than a declared business default.

**Why it matters:** A getter, bounce, empty-message, system, or error path can observe NULL and fail, bypass a comparison, or pass an invalid value into serialization or privileged logic.

**What to look for:**
- Globals initialized only in the main internal-message path but read by getters, bounces, external messages, tick-tock, or helpers
- Globals treated as persistent storage or assumed to retain a value across transactions
- Reads that rely on zero, false, empty tuple, or another default without explicit assignment and validation

---

## V170 — Fragile Initialization-State Detection

**What:** Initialization is inferred from cell shape, reference count, remaining bits, balance, or another incidental property instead of an explicit authenticated state field.

**Why it matters:** Valid schema evolution or attacker-crafted cells can resemble an initialized or uninitialized state, enabling takeover, repeated initialization, skipped migration, or permanent lockout.

**What to look for:**
- `refsCount`, remaining-bit checks, empty-cell checks, or exact storage shape used as the sole initialization guard
- First-message initialization that does not authenticate the initializer and set an explicit state atomically
- Upgrade or migration logic whose new layout changes the property used as the initialization sentinel

---

## V171 — Unsafe Manual Bounced-Message Policy

**What:** `@on_bounced_policy("manual")` or equivalent manual routing allows bounced messages to reach ordinary command parsing without a mandatory bounced discriminator and dedicated policy.

**Why it matters:** A bounced copy of an outbound command can be interpreted as a fresh inbound command, invoke privileged logic, double-account value, or bypass sender assumptions because the source and body have different semantics after bouncing.

**What to look for:**
- Manual bounce policy without an early, unconditional check of the bounced flag
- Shared dispatch for bounced and non-bounced bodies before authentication and schema separation
- Ordinary opcode handlers reachable with truncated bounce bodies or the bounce prefix still present

---

## V172 — Unrecoverable Bounce Failure

**What:** A stateful outbound operation assumes a bounce will restore state even when the original message cannot fund the bounce, the bounce handler can fail, or the bounced message itself cannot bounce again.

**Why it matters:** Debit, lock, supply, or pending-operation state can remain permanently inconsistent after delivery failure. Recovery logic that throws may discard the only compensation opportunity.

**What to look for:**
- Outbound value too small to cover forwarding, receiver execution, and bounce fees under adverse conditions
- Bounce handlers with complex parsing, unbounded work, sends, or failure branches before recovery is persisted
- Recovery designs that rely on a failed bounced message producing another bounce or retry automatically

---

## V173 — Library-Cell Resolution and Liveness Failure

**What:** Contract execution depends on library-referenced code whose resolution environment, hash, publication, or lifetime is not guaranteed and verified.

**Why it matters:** A missing or mismatched library can freeze the host contract, while an unexpected local/global resolution environment can execute code different from what reviewers, wrappers, or deployers assumed.

**What to look for:**
- StateInit or code cells containing library references without checking exact hashes and deployment environment
- No operational plan to keep required public libraries available and funded for the full contract lifetime
- Emulator/tests injecting local libraries that are absent, overridden, or different in the target network environment

---

## V174 — C5 Action-List Injection or Corruption

**What:** A wallet, extension, proxy, or plugin accepts or constructs an untrusted C5 action list without validating every action and the linked-list structure.

**Why it matters:** Malformed chains can fail the action phase, while unauthorized send, reserve, code-change, or library-change actions can bypass the high-level command policy and act with the contract's authority.

**What to look for:**
- Raw action-list cells accepted from signatures, plugins, extensions, or forwarded payloads
- Missing allowlist and field validation for every action opcode, mode, destination, value, StateInit, and library/code mutation
- Cyclic, excessively deep, malformed, trailing, or multi-root action chains not rejected before commit

---

## V175 — Silent External-Action Failure Consumes Seqno

**What:** An external-message handler advances and commits replay state, then catches or suppresses action construction/execution failure and returns success without the intended action.

**Why it matters:** The signed request becomes unreplayable even though its transfer or administrative action did not occur. In wallets this can silently drop user orders and desynchronize off-chain state; in protocols it can strand pending operations.

**What to look for:**
- Empty `catch` blocks or broad failure suppression around action construction after seqno update
- Seqno committed before all user-requested actions are structurally validated and safely queued
- Success exit status emitted when the signed intent was only partially or never executed

---

## V176 — Suppressed Cell-Overflow Safety Without Size Proof

**What:** `@overflow1023_policy("suppress")` or an equivalent option disables cell-capacity protection without an exact proof that every reachable serialization stays within 1023 bits and 4 refs per cell.

**Why it matters:** Attacker-controlled or future-expanded data can be truncated, misserialized, or fail later than expected, producing ambiguous messages, lost fields, inconsistent hashes, or state/action divergence.

**What to look for:**
- Overflow suppression on structs containing dynamic data, optional branches, nested values, refs, or forwarded payloads
- No worst-case bit/ref calculation covering all variants and compiler-added tags
- Protocol fields, signatures, hashes, or authorization data located near a capacity boundary where omission changes meaning

---
