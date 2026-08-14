# Attack Vectors Reference (6/7) — Language Semantics, Hidden State & Parsing

## V151 — Sensitive Data Exposure in On-Chain State or Messages

**What:** A contract treats message bodies, persistent cells, transaction data, or emulator-visible runtime values as confidential.

**Why it matters:** TON blockchain data is public and permanent. Secrets, credentials, private bids, salts, or unrevealed answers can be extracted and used before the intended reveal or authorization step.

**What to look for:**
- Plaintext secrets, private keys, passwords, answers, bids, or recovery material in messages or c4 storage
- Commitments without a strong salt or domain separation
- Access control that assumes a cell or internal message field cannot be observed

---

## V152 — Manual Throw of Success Exit Codes

**What:** Validation failure or an error path explicitly throws exit code `0` or `1`, which TVM treats as successful termination.

**Why it matters:** Callers, wrappers, indexers, or later protocol logic may interpret the failed operation as successful even though the intended state transition or response did not occur.

**What to look for:**
- `throw(0)`, `throw(1)`, `throw_if(0, ...)`, or equivalent generated behavior on failure branches
- Custom error constants equal to TVM success codes
- Off-chain success detection based only on transaction exit status

---

## V153 — Operation Code Reused for Incompatible Message Schemas

**What:** The same opcode identifies two or more message bodies with different field order, width, refs, or authorization semantics.

**Why it matters:** A body valid for one handler can be misparsed by another contract, version, fallback, or proxy, causing type confusion and unintended state changes.

**What to look for:**
- Duplicate opcode constants across structs, contracts, legacy handlers, or union members
- Upgrade code that keeps an opcode but changes its TL-B layout
- Routers forwarding an opcode without binding it to an exact schema and destination

---

## V154 — Unbounded Human-Format Parsing On Chain

**What:** A contract parses attacker-controlled strings, decimal numbers, domains, JSON-like data, comments, or other human-oriented formats without strict size and complexity limits.

**Why it matters:** Variable-length decoding, recursion, Unicode/byte ambiguity, and repeated scans can exhaust gas or create multiple interpretations of the same security-critical value.

**What to look for:**
- Loops over arbitrary strings, snake cells, labels, delimiters, or numeric digits
- SnakeString chains without exactly zero-or-one continuation ref per chunk, byte-aligned data, explicit termination, and a hard maximum depth/total byte length
- Domain/path normalization performed on chain without canonicalization and hard bounds
- Parsed human text used as an address, amount, identity, opcode, or authorization input

---

## V155 — Parent-Child Peer Authentication Failure

**What:** A parent trusts a claimed child identity, or a child trusts a claimed parent, without deriving and comparing the actual sender against trusted StateInit data.

**Why it matters:** A forged sibling or unrelated contract can spoof callbacks, deposits, completion messages, or administrative commands and corrupt parent-managed accounting.

**What to look for:**
- Parent/child address accepted from payload rather than `sender`/`in.senderAddress`
- Missing address derivation from code, data, owner, index, workchain, and salt
- Callback correlation that checks query ID but not the derived peer address

---

## V156 — Unsupported First-Time Counterparty Deployment Assumption

**What:** A flow assumes the destination wallet, child, or counterparty contract is already deployed, or attaches StateInit without validating its exact address, funding, and failure behavior.

**Why it matters:** The first interaction can fail, deploy the wrong code/data, consume user value, or commit local state while the expected remote account remains uninitialized.

**What to look for:**
- Sends to deterministic wallets/children without checking first-deployment cost or StateInit
- Different inputs used for destination derivation and attached code/data
- State credited/debited before deployment success with no bounce or retry recovery

---

## V157 — FunC Modifying vs Non-Modifying Method Confusion

**What:** FunC code calls a modifying helper with `.` instead of `~`, or uses `~` where the original value should remain unchanged.

**Why it matters:** The mutation may affect only a returned copy or unexpectedly overwrite the first argument, leaving parsers, dictionaries, balances, and counters inconsistent with later logic.

**What to look for:**
- Mutating slice/dictionary helpers invoked with `.` while their result is discarded
- `~` calls whose updated first return value is unintentionally written back
- Security checks performed on a modified copy while stale original state is saved

---

## V158 — FunC Storage Field Shadowing or Misbinding

**What:** A local variable, tuple component, parameter, or return value shadows an authoritative storage field or is passed to a save helper in the wrong position.

**Why it matters:** Authorization may validate the stored value while persistence writes an attacker-controlled or stale value, silently replacing owners, balances, or configuration.

**What to look for:**
- Local and global storage variables with identical or confusing names
- Large positional `load_data()`/`save_data()` tuples with reordered arguments
- Redeclaration in nested branches followed by persistence outside the branch

---

## V159 — Ignored Helper Result or Optional Failure

**What:** Code ignores a success flag, nullable lookup, mutation result, or low-level failure returned by a dictionary, map, storage, parser, or system helper.

**Why it matters:** Execution continues using a fabricated default, stale value, or assumed mutation after the underlying operation failed.

**What to look for:**
- Discarded dictionary delete/update success flags
- Tolk `MapLookupResult.isFound` or nullable results not checked before use
- Tact optional/native results force-unwrapped or replaced with privileged defaults

---

## V160 — Missing Termination After a Handled Branch

**What:** A handler completes a recognized branch but does not return or otherwise terminate before reaching default, error, or another operation branch.

**Why it matters:** The transaction can revert after queuing intended actions, execute a second conflicting path, or double-send and double-update state.

**What to look for:**
- Independent `if` dispatch blocks where a handled opcode falls through
- Tolk `match`/router helpers and Tact receivers that continue into a default throw
- Success paths followed by shared cleanup that repeats sends or persistence

---

## V161 — Tact Mutable Helper Does Not Persist State

**What:** A Tact mutable helper changes a temporary value or returns an updated value that the caller ignores instead of updating the contract's persistent `self` state.

**Why it matters:** Later sends and accounting assume the mutation happened while receiver-end serialization persists the old owner, balance, counter, or status.

**What to look for:**
- Mutable helpers operating on copied structs or locals rather than `self`
- Returned updated values discarded by callers
- Outgoing actions based on new values while getters/storage still expose old values

---

## V162 — Tact `setData()` Overwritten by Implicit State Save

**What:** A Tact receiver calls low-level `setData()` and then reaches normal receiver completion, where generated code serializes stale typed `self` over the manually replaced data.

**Why it matters:** Upgrade, migration, recovery, or administrative state replacement can be partially or completely undone, producing corrupted storage that may only fail on the next message.

**What to look for:**
- Direct `setData()` followed by ordinary return/fallthrough
- Typed fields not synchronized with the manually written cell
- No compiled-artifact test proving the final c4 value for every replacement branch

---

## V163 — Tact External Receiver Relies on Internal Context

**What:** A Tact external receiver or shared helper assumes internal-message `sender()`, `context()`, attached value, or bounce semantics are available.

**Why it matters:** External messages have no authenticated inbound sender/value context. The path may become unauthenticated, fail after accepting gas, or reuse a meaningless default identity.

**What to look for:**
- `external` handlers calling helpers that read sender or internal context
- Authorization derived from missing context instead of signatures and replay state
- `acceptMessage()` before discovering that internal-only data is unavailable

---

## V164 — Tact Default `Int` Serialization Mismatch

**What:** An externally serialized Tact `Int` omits an explicit width/format and uses the default signed 257-bit encoding where the protocol expects `coins`, `uint32`, `uint64`, `uint256`, or another shape.

**Why it matters:** Peers parse the wrong boundaries or signedness, and negative or oversized values can enter a business domain intended to be unsigned and bounded.

**What to look for:**
- Bare `Int` fields in messages, storage, or getters crossing a contract boundary
- Claimed standard structs without explicit `as uintX`, `intX`, or `coins`
- Wrapper and contract serializers disagreeing on the field width

---

## V165 — Tact Bulk Map or Complex State Import

**What:** A Tact message supplies a map, dictionary-like array, or complex config object that is assigned directly into persistent state.

**Why it matters:** Bulk replacement bypasses per-entry key/value validation, uniqueness, size limits, funding discipline, and invariants enforced by normal incremental updates.

**What to look for:**
- Persistent map/config assignment directly from an inbound message
- Missing per-entry canonicalization, bounds, authorization, and storage funding
- Replacement paths that bypass duplicate, ownership, quota, or reserved-key checks

---

## V166 — Tact Safety Checks Disabled on Critical Paths

**What:** Compiler/runtime safety options are disabled for arithmetic, parsing, or message handling relied on by security-critical code.

**Why it matters:** Optimizations can remove checks assumed by the source-level reasoning, turning malformed input or arithmetic boundaries into state corruption or unexpected exits.

**What to look for:**
- Security-related options disabled in `tact.config.json`
- No explicit replacement guards on every reachable critical path
- Tests covering only default compiler settings rather than the deployed configuration

---

## V167 — Tact Optional Address / `addr_none` Mismatch

**What:** A Tact message uses non-optional `Address` where the actual TL-B or claimed standard permits `addr_none`, `Maybe`, or another absent-address representation.

**Why it matters:** Valid standard messages can fail after earlier state effects, or fabricated fallback addresses can redirect refunds, ownership, discovery, and callback flows.

**What to look for:**
- Non-optional address fields for standard messages that permit absence
- `null` replaced with zero, sender, owner, or another privileged default
- Optional tag bits ignored or parsed with a fixed internal-address schema

---
