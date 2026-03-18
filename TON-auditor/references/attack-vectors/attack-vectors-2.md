# Attack Vectors Reference (2/4)

170 total attack vectors

---

**43. Missing `impure` Modifier on State-Changing Functions**

- **D:** If a function (such as an authorization check authorize) only throws an exception via `throw_unless` without a return value and lacks the impure modifier, the compiler may optimize away the entire function call because its result is unused. This bypasses the security check.
- **FP:** The function truly does not modify contract state, send messages, or change variables outside the stack (e.g., a pure mathematical calculation).

**44. Incorrect Method Call Syntax (`.` vs `~`)**

- **D:** In FunC, calling a method with `.` (e.g., `dict.udict_delete?()`) does not modify the original variable; you must use `~` (e.g., `dict~udict_delete?()`) to reassign the modified value back to the variable. Misusing `.` results in the contract state (like ledgers or balances) not being updated as expected.
- **FP:** The developer explicitly intends to keep the original variable unchanged and assigns the returned new copy to a different variable.

**45. Signed Integer Abuse in Asset/Voting Power**

- **D:** Using `int` (signed) to handle assets or voting power. An attacker can pass a negative value; executing `balance += amount` would decrease the balance, or `balance -= amount` would cause the balance to increase infinitely due to double negatives.
- **FP:** The business logic specifically requires handling negative numbers (e.g., profit/loss statistics) and has strict boundary checks in place.

**46. Predictable On-chain Randomness**

- **D:** Using logical time `cur_lt()`, block time, or message bodies as a seed for randomness. Validators or attackers can brute-force the random result by predicting the sequential logical time within the same block to guarantee a "win.".
- **FP:** Randomness is used for non-core features or the protocol explicitly informs users that the randomness is "weak" and not tied to asset distribution.

**47. Sensitive Data Exposure in Transaction History**

- **D:** Sending passwords, raw private key hashes, or other sensitive information as message parameters. Since TON stores all transaction history, attackers can easily retrieve plaintext sensitive info by parsing historical message bodies.
- **FP:** Data being sent is a salted hash, or the data is intended to be public and transparent.

**48. Missing or Weak Bounced Message Handling**

- **D:** When a contract sends a message that fails and triggers a "Bounce," the sending contract must have logic to handle the bounced flag. Failure to do so leads to assets being deducted from the sender but never credited to the recipient, causing asset loss or logic deadlocks.
- **FP:** The contract sends a "Non-bounceable" message, or the destination address is proven to never reject that specific operation.

**49. Account Destruction via Race Conditions**

- **D:** In withdrawal logic, if the balance reaches zero, the contract might be set to self-destruct (Mode 32). In an asynchronous environment, an attacker can send multiple concurrent withdrawal messages, causing the contract to destroy itself before all funds are settled, allowing unowned funds to be claimed after redeployment.
- **FP:** The contract is designed as a "one-time-use" logic, or there are strict "cooling-off" periods and permission checks before destruction.

**50. Execution of Untrusted Third-Party Code**

- **D:** Using the `EXECUTE` directive to run external code units. Since "Out-of-gas" (OOG) exceptions cannot be handled by a CATCH block, an attacker can commit the contract state and then raise an OOG, forcibly manipulating the contract's final behavior.
- **FP:** Only running whitelisted code that has been passed via governance or is pre-stored in the contract.

**51. Variable/Function Name Collision**

- **D:** FunC allows variable names to contain characters like `,`, `++`, or `-`. Without proper Linting, a developer might write `var++` expecting an increment, but the compiler treats it as a single variable name, causing logic to fail silently.
- **FP:** The project utilizes strict automated Linting and has a comprehensive naming convention document.

**52. Manual Throw of System Exit Codes (0 or 1)**

- **D:** Manually executing `throw(0)` or `throw(1)`. In TVM, `0` and `1` represent "Successful Execution." If a security check fails but throws `0` or `1`, the system considers the validation passed and proceeds with illegal operations.
- **FP:** The code is located within a legitimate exit branch intended to end execution early.

---

---

**53. Synchronous Data Pull Expectation**

- **D:** Developers coming from Ethereum often try to "pull" data from other contracts synchronously. Because TON is asynchronous, data must be requested via a message exchange. Expecting an immediate result results in using stale or empty data.
- **FP:** The developer correctly implements the Carry-value pattern or the Request-Response asynchronous callback pattern.

**54. Improper `recv_internal` and `recv_external` Entry Point Usage**

- **D:** Misunderstanding the predefined `method_id`. `recv_internal` (id 0) handles in-blockchain messages, while `recv_external` (id -1) handles messages from the outside world. If logic intended for internal authentication is placed in `recv_external`, or if `recv_external` lacks `accept_message()` and replay protection, the contract can be exploited via unauthorized external calls or gas-guzzling replay attacks.
- **FP:** A contract that intentionally omits `recv_external` to only allow internal interactions, or a contract that uses custom `method_id` for specific off-chain tools.

**55. Failure to Route or Filter Bounced Messages**

- **D:** TON addresses are deterministic and can be uninitialized. Sending a message to an uninitialized account triggers a "bounce." If the contract does not check the bounced flag at the start of `recv_internal`, it may process the bounced message as a new, valid command, leading to unexpected state changes or "phantom" operations.
- **FP:** The contract performs "fire-and-forget" actions where the failure of the message has no impact on the contract's state or the user's balance.

**56. TON Address Representation and Workchain Mismatch**

- **D:** TON addresses have Raw and User-friendly (Bounceable/Non-bounceable) formats. Failing to validate the workchain (e.g., ensuring it is 0:) using `force_chain()` can lead to assets being sent to addresses on non-existent or unsupported workchains, causing permanent loss of funds.
- **FP:** Multi-chain protocols specifically designed to handle cross-workchain messaging.

**57. Use of Non-Bounceable Flags in Value Transfers**

- **D:** Using the `0x10` flag (Non-bounceable) instead of `0x18` (Bounceable) for asset transfers. If the destination account is uninitialized or the call fails, the funds will not bounce back to the sender, resulting in the permanent loss of the transferred TON/Tokens.
- **FP:** Sending small "dust" amounts for notifications where the cost of handling a bounce exceeds the value of the transfer.

**58. Missing Replay Protection for External Messages**

- **D:** `recv_external` messages do not automatically provide replay protection. Without a `seqno` (sequence number) check or a high-load process identifier, an attacker can capture a signed external message and replay it multiple times to drain the contract or execute administrative functions repeatedly.
- **FP:** Operations that are idempotent by nature (e.g., a "heartbeat" or "update price" where the result is the same regardless of how many times it is called).

**59. Message Cascade Race Conditions**

- **D:** A message flow can span multiple blocks. If a contract checks a condition (e.g., "User has 100 Tokens") in the first block but doesn't lock the state, an attacker can initiate a parallel flow to spend those tokens before the third or fourth stage of the original flow completes.
- **FP:** Contracts using a strictly sequential, single-message architecture that does not trigger multi-stage cascades.

**60. Improper State Synchronization (Carry-Value Violation)**

- **D:** Attempting to "query" a balance from another contract instead of using the Carry-Value pattern. In TON, by the time the response returns, the balance may have changed. Contracts must carry the value/state within the message (e.g., Jetton `internal_transfer`) rather than relying on stale lookups.
- **FP:** Non-critical data lookups where real-time accuracy is not required for financial integrity.

**61. Failure to Return Gas Excesses**

- **D:** Not returning excess gas to the `msg.sender` (using `op::excesses`). Over time, this causes the contract to accumulate "dust" funds that belong to users. More critically, if the contract runs out of balance to pay storage fees because it didn't manage incoming gas correctly, the account can become frozen.
- **FP:** Contracts with a built-in "sweep" function that allows an admin to collect accumulated gas for protocol revenue.

**62. Ignored Return Values of Dictionary/System Operations**

- **D:** Failing to check the success boolean of operations like `udict_delete?`. If the deletion fails but the code proceeds as if it succeeded, it creates a logical inconsistency that can lead to double-spending or unauthorized access.
- **FP:** Operations where failure is expected and handled by a subsequent `throw` or alternative logic path.

---

---

**63. Fake Jetton Token Injection**

- **D:** A vault contract receiving an `op::internal_transfer` fails to verify if the `sender_address` is the authentic `wallet_address` for that Jetton. Attackers can deploy a fake Jetton contract and send transfer messages to drain real assets from the vault.
- **FP:** The contract dynamically calculates and verifies the sender's Wallet address using the official Minter's algorithm before processing the transfer.

**64. Unconditional `ACCEPT` of External Messages**

- **D:** External messages do not bring TON for gas; the contract pays from its own balance. If the `ACCEPT` instruction (which sets the `gas_limit` to max) is executed without rigorous preliminary checks (e.g., signature or `seqno` verification), an attacker can flood the contract with external messages to drain its entire TON balance through gas fees.
- **FP:** The contract has a strictly limited `SETGASLIMIT` for specific unauthenticated functions, or the contract is intended to be a public utility with a finite, disposable balance.

**65. Uncalculated Gas Consumption (Unhandled OOG)**

- **D:** TVM "Out of Gas" errors cannot be caught or handled by `CATCH` blocks. If complex logic (like large dictionary iterations) is executed without pre-calculating gas or using `gas_consumed`, the transaction will fail mid-execution, potentially leaving the contract in an inconsistent state or preventing users from executing critical "rescue" functions.
- **FP:** The contract uses `CHECKGAS` to verify sufficient gas exists before starting a loop, or the state is only updated via `COMMIT` at the very end of a successful execution.

**66. Use of Predictable On-chain Randomness**

- **D:** TON's built-in `random()` functions depend on the block's logical time. An attacker can brute-force the logical time within the same block to predict the random outcome. This is especially critical in gambling or NFT minting where "luck" determines value.
- **FP:** The contract uses a "Commit-and-Disclose" scheme (where participants submit hashes first and reveal values later), or the randomness is only used in internal messages where `cur_lt` is harder for an external attacker to manipulate.

**67. Signature Malleability/Front-running in External Messages**

- **D:** Because pending messages in the TON mempool are public, an attacker can observe a signed external message and resubmit it with modified non-signed parameters (like the recipient address or gas fees) if those parameters weren't part of the signed data.
- **FP:** All critical parameters (amount, recipient, expiration, and seqno) are packed into a cell and signed as a single unit, making any modification by a third party result in an invalid signature.

**68. Lack of Replay Protection for Signed Data**

- **D:** Even with a valid signature, if the contract does not track a `seqno` or a unique Nonce, an attacker can capture the signed message and "replay" it multiple times (e.g., executing the same "Withdraw 100 TON" command 10 times).
- **FP:** The signed message includes a timestamp/expiration that has already passed, or the underlying logic is naturally one-time-use (e.g., initializing a contract that can only be initialized once).

**69. Missing Sender Validation on Protected Operations**

- **D:** Operation handlers or state-modifying functions fail to verify the `sender_address`. In TON, roles must be explicitly checked against stored state (e.g., `owner_address`). For contract-triggered operations, the sender must be validated against its StateInit hash to ensure the message originated from the expected contract.
- **FP:** The function is a "public" or "unauthenticated" getter/setter where any user is permitted to trigger the logic (e.g., a public mint or a permissionless arbitrage trigger).

**70. Improper Pre/Post Initialization Logic**

- **D:** A contract performs actions (like acquiring a Jetton wallet address) that change its state during initialization but fails to distinguish which actions are allowed before vs. after this phase. An attacker might trigger "post-init" logic before the contract is ready, leading to null-address interactions or stuck funds.
- **FP:** The contract uses a strict init? boolean flag in its storage to gate all sensitive logic, ensuring no overlap between phases.

**71. Incomplete Message Parsing (`end_parse` omission)**

- **D:** The contract parses an input message using `begin_parse()` but fails to call `end_parse()` at the end of the chain. This allows an attacker to append "excess data" to a legitimate message, which might be ignored by the logic but could lead to unexpected behavior in downstream cell processing or gas manipulation.
- **FP:** The contract explicitly handles "extra" data for forward-compatibility or uses a flexible message schema where remaining bits are intentionally passed to another internal function.

**72. Chain Execution Failure due to Insufficient Gas**

- **D:** An inter-contract operation fails to provide enough gas to complete the entire transaction chain. If the compute phase finishes but the action phase fails (e.g., due to `out_of_gas` during `send_raw_message`), the compute phase results are not automatically rolled back unless `SEND_MODE_BOUNCE_ON_ACTION_FAIL` is used correctly.
- **FP:** The contract implements a try/catch or a bounced message handler that triggers a manual rollback/refund of state if the downstream chain fails.

---

---

**73. Improper `SEND_MODE` causing Unintended Rollbacks**

- **D:** Using `SEND_MODE_BOUNCE_ON_ACTION_FAIL` for "optional" messages (like returning excess gas). If the gas return fails (e.g., zero balance), the entire compute phase—including valid state changes—will be rolled back.
- **FP:** Using `SEND_MODE_IGNORE_ERRORS` for non-critical messages, ensuring that the failure of an optional notification does not revert the core transaction logic.`

**74. Permanent Jetton Loss due to Malformed `transfer_notification`**

- **D:** When a contract receives a `jetton::transfer_notification`, it must parse the `forward_payload`. If the parsing fails (e.g., empty payload, unexpected Jetton type, or incorrect `timeStarted` state), and the contract simply throws an exception, the transaction reverts. However, because the Jettons have already reached the contract's wallet, and there is no try-catch recovery or automated "return to sender" mechanism, the Jettons become permanently locked.
- **FP:** The contract is a simple vault with a manual "rescue" function that allows an admin to withdraw trapped tokens.

**75. Usage of the Same `op` for Different Message Types**

- **D:** Using the same `op` (e.g., `op::burn_notification`) for different message formats (e.g., one between Minter and Master, another between Wallet and Minter). This creates ambiguity for blockchain explorers and indexers and can lead to parsing errors if the contract logic expects one Cell structure but receives another.
- **FP:** The message structures are identical across different paths, and the security requirements for both logic branches are the same.

**76. Wrong Serialization of `TakeWalletAddress` Message**

- **D:** The [TEP-89](https://github.com/ton-blockchain/TEPs/blob/master/text/0089-jetton-wallet-discovery.md#scheme) standard requires the owner_address in a take_wallet_address response to be a Reference (a cell pointer, Maybe ^MsgAddress). If the contract includes the address directly within the same cell as a slice, it deviates from the standard, rendering proper parsing by standard-compliant wallets and indexers impossible.
- **FP:** The contract does not claim TEP-89 support and is used exclusively within a private, closed-loop ecosystem.

**77. `is_bounced()` Returning `int` Instead of `bool`**

- **D:** The function `is_bounced` returns `msg_flags & 1` (resulting in `0` or `1`). In FunC, the standard TRUE representation is `-1`. If the result of `is_bounced()` is used in a bitwise logical expression with other integers, it can lead to accidental logic errors because `1` does not behave like `-1` in bitwise operations.
- **FP:** The developer strictly uses if (`is_bounced()`) (which checks for non-zero) and never combines the result in complex bitwise logic.

**78. Mutable Metadata in Jetton Smart Contract**

- **D:** The Jetton Minter contains functionality (often `op == 4`) that allows the `admin_address` to change the `content` (metadata like name, decimals, or description) after deployment. This allows malicious actors or compromised admins to mislead users by changing token properties.
- **FP:** The token is explicitly documented as having dynamic metadata (e.g., for a game), and the admin is a multi-sig or a DAO.

**79. `jetton-minter::op::mint` Allows Sending Invalid Messages**

- **D:** The `mint` function allows an admin to trigger an `internal_transfer`. If the `master_msg` provided by the admin is not validated, it can lead to: 1. Incorrect `o`p usage; 2. Ignored `forward_ton_amount` causing storage failures; 3. Failure to use `CARRY_REMAINING_GAS`, leading to lost funds.
- **FP:** The admin is a highly trusted, cold-wallet-held address, and the minting process is considered an internal administrative risk.

**80. Bounced `op::internal_transfer` from Minter is Not Processed**

- **D:** When a Minter sends an `internal_transfer` to a Wallet to mint tokens, it increases the `total_supply`. If that message is bounced (e.g., the wallet address was invalid), the Minter must handle the bounce and decrease the `total_supply`. If not, the `total_supply` will be higher than the actual circulating supply.
- **FP:** The minting destination is a pre-calculated, deterministic address that is guaranteed to exist and accept the transfer.

**81. Wrong Authorization in `TokenBurnNotification` Handler**

- **D:** The handler for `TokenBurnNotification` should verify that the `sender` is the legitimate Jetton Wallet of the user. If the code checks the wallet of the `response_destination` instead of the `msg.sender`, an attacker could potentially spoof burn notifications to manipulate the `total_supply`.
- **FP:** The `response_destination` is hard-coded or strictly validated in previous steps to match the actual sender.

**82. Empty `response_destination` is Forbidden/Unwrapped**

- **D:** The [TEP-74](https://github.com/ton-blockchain/TEPs/blob/master/text/0074-jettons-standard.md) standard allows `response_destination` to be `addr_none` (no excesses sent). If the contract uses !! (force unwrap) or fails to handle addr_none, standard-compliant calls from users will revert, making an optional parameter effectively mandatory.
- **FP:** The business logic requires a valid return address for gas accounting or auditing purposes.

---

---

**83. Wrong Mode for Sending Cashback (Gas Theft)**

- **D:** When returning excess gas (cashback) to a user, using mode: `SendPayGasSeparately` (Mode 1) while also sending the `msg_value` causes the contract to pay for the transfer out of its own balance instead of the incoming gas. This allows an attacker to drain the contract's storage fees by repeatedly triggering "excesses" messages.
- **FP:** The contract has a massive TON balance and the excess amount is significantly higher than the fee, or the transaction is gated by a high forward_ton_amount.

**84. Rebasing / Elastic Supply Token Accounting**

- **D:** Contract holds rebasing tokens (stETH, AMPL, aTokens) and caches `balanceOf(this)`. After rebase, cached value diverges from actual balance.
- **FP:** Rebasing tokens blocked at code level (revert/whitelist). Accounting reads `balanceOf` live. Wrapper tokens (wstETH) used.
