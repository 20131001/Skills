# Attack Vectors Reference (2/4)

170 total attack vectors

---

**31. Wrong `SEND_MODE` Causing Unintended Rollback**

- **D:** Using `SEND_MODE_BOUNCE_ON_ACTION_FAIL` for optional or best-effort messages can make an irrelevant transfer failure revert the entire transaction. That turns a harmless notification failure into a state rollback vulnerability.
- **FP:** The message is critical to correctness and must revert the full operation if it cannot be delivered.

**32. Permanent Jetton Loss from Malformed `transfer_notification`**

- **D:** If a contract receives a Jetton `transfer_notification`, accepts the tokens, and then throws while parsing the `forward_payload`, the business flow may fail after the Jettons have already landed in the wallet. Without a recovery or refund path, those Jettons can become permanently stuck.
- **FP:** The contract includes an explicit rescue or refund mechanism for malformed incoming token transfers.

**33. Reusing the Same `op` for Different Message Schemas**

- **D:** Assigning the same `op` code to different message layouts creates ambiguity for parsers, indexers, and sometimes the contract itself. A handler may decode one schema while receiving another and misinterpret attacker-controlled fields.
- **FP:** The schemas are truly identical and remain security-equivalent across every path that uses that `op`.

**34. Incorrect `TakeWalletAddress` Serialization**

- **D:** TEP-89 expects `owner_address` in the wallet discovery response to be encoded as a reference (`Maybe ^MsgAddress`). Serializing it inline instead of by reference breaks standard parsers and can make integrations mis-handle the response.
- **FP:** The contract does not claim TEP-89 compatibility and is only used in a closed ecosystem with custom decoders.

**35. `is_bounced()` Returns `int` Instead of Canonical `bool`**

- **D:** Returning `msg_flags & 1` yields `0` or `1`, while FunC commonly represents true as `-1`. If that result is later combined with bitwise logic that assumes canonical booleans, the check can behave incorrectly.
- **FP:** The value is only used in plain truthiness checks such as `if (is_bounced())`.

**36. Mutable Jetton Metadata**

- **D:** If the Jetton minter allows metadata updates after deployment, an admin or compromised admin key can change name, symbol, decimals, or description to mislead users and integrators.
- **FP:** The token is explicitly documented as mutable and that governance model is part of the intended trust assumptions.

**37. Unsafe Admin-Supplied Message in `jetton-minter::op::mint`**

- **D:** If `mint` forwards an admin-supplied `master_msg` without validating its structure and gas semantics, the minter may emit malformed internal transfers, mis-handle `forward_ton_amount`, or lose funds through incorrect send modes.
- **FP:** The implementation rebuilds or fully validates the outgoing mint message before sending it.

**38. Unhandled Bounce of Minter `internal_transfer`**

- **D:** When minting, the minter usually increases `total_supply` before the wallet receives the tokens. If the outbound `internal_transfer` bounces and the minter does not reverse the supply increase, reported supply diverges from actual circulation.
- **FP:** The mint destination is guaranteed correct and bounce handling is unnecessary by construction.

**39. Wrong Authorization in `TokenBurnNotification` Handler**

- **D:** The burn handler must verify that the sender is the legitimate Jetton wallet that burned the tokens. If it validates some other address, such as `response_destination`, an attacker may spoof burn notifications and manipulate supply accounting.
- **FP:** The sender is independently verified against the expected wallet derivation for that holder.

**40. Forcing `response_destination` When `addr_none` Is Valid**

- **D:** TEP-74 allows `response_destination` to be `addr_none`. Code that force-unwraps or rejects that case turns an optional field into a mandatory one and breaks compatibility with standard callers.
- **FP:** The protocol intentionally requires a return address and documents that deviation from the standard.

---

---

**41. Wrong Mode for Cashback / Excess Return**

- **D:** Returning excess gas with a mode that pays fees from the contract's own balance instead of the carried value lets attackers repeatedly trigger refunds that slowly drain the contract's TON. This is especially dangerous in hot paths that send cashback on every call.
- **FP:** The contract intentionally subsidizes those transfers and has enough balance for that design.

**42. Local Variable Shadowing of Storage Fields**

- **D:** Large `load_data()` / `save_data()` patterns make it easy to accidentally reuse or shadow a storage variable with a local value derived from message input. If the wrong variable is later passed into `save_data()`, attacker-controlled data can overwrite persistent configuration or accounting state.
- **FP:** Storage is segmented into nested cells or otherwise structured so local business variables cannot be confused with raw storage fields.

**43. Storage Fee Depletion through Improper Excess Handling**

- **D:** Using `SEND_MODE_CARRY_ALL_REMAINING_MESSAGE_VALUE` does not prevent the contract from paying other fees from its own balance. If the same flow also uses fee-separate sends, logs, or other balance-funded actions, each call can leak a small amount of TON from storage reserve until the contract is frozen or deactivated.
- **FP:** The contract reserves its minimum storage balance up front, for example with `raw_reserve()`, before sending excesses or optional messages.

**44. Insufficient `msg_value` for Message Consequences**

- **D:** In TON, the entry message must fund the full cascade it creates. If the handler accepts too little `msg_value` and still mutates state, downstream messages may fail in later compute or action phases while the earlier state changes remain committed, causing partial execution and locked funds.
- **FP:** The contract explicitly checks that `msg_value` covers worst-case downstream costs, or the flow is a simple carry-value chain where a later failure cannot violate invariants.

**45. Missing Entry-Point Validation in Asynchronous Cascades**

- **D:** In a multi-stage message flow, the first reachable entrypoint must enforce the critical invariants for all later stages. If those checks are deferred or only assumed by consequence messages, the cascade can fail mid-flight or reach an inconsistent state after partial execution.
- **FP:** Each stage independently validates its prerequisites, or the carry-value design ensures that downstream failure cannot corrupt state or strand third-party funds.
