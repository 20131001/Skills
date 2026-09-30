# Finding Validation

Every candidate passes four sequential gates. Reject only when source evidence or an authoritative protocol rule disproves reachability, trigger, or impact. If evidence is incomplete, retain a lead or `review_trail` with the exact missing proof. Record the actor, configuration, and impact separately so a reachable loss is not discarded merely because it is not a profitable unprivileged attack.

## Gate 1 — Refutation

Construct the strongest argument that the finding is wrong. Check protocol message provenance and transaction semantics before treating an absent contract guard as exploitable. Find any source guard, runtime constraint, or specified behavior that kills the claimed transition; cite the blocking evidence and trace it through the exact scenario. For example, a normal contract message cannot set the bounced-message flag or invoke a bounced-only receiver. An attacker must be shown to cause a legitimate outbound message to bounce with the claimed body and sender.

Existing tests establish observed behavior, not intended safety. When a test expectation conflicts with a source comment or design claim, record the conflict and assess the concrete impact; do not use the passing test alone to reject the candidate. If intent remains necessary to decide material harm, retain a `review_trail` with the unresolved question.

- Concrete refutation (specific guard blocks exact claimed step) → **REJECTED** (or **DEMOTE** if code smell remains)
- Speculative refutation ("probably wouldn't happen") → **clears**, continue

## Gate 2 — Reachability

Prove the vulnerable state is reachable under a configuration permitted by the code. Consider the asynchronous message model — state may be reachable through message sequences that are not obvious from a single handler. Distinguish code-enforced constraints from deployment-script defaults.

- Structurally impossible (enforced invariant prevents it) → **REJECTED**
- Requires a privileged action outside normal operation → record that precondition and assess the resulting harm; do not reject solely for privilege
- Achievable through normal usage, standard Jetton interactions, or common message sequences → **clears**, continue

## Gate 3 — Trigger

Identify who can trigger the failing transition and prove the required message or operation is reachable. Check sender validation and admin guards. Include ordinary user actions, authorized maintenance, and permitted configurations when they can cause unintended loss or persistent state corruption.

- Only a trusted role can trigger → record the trust assumption and assess whether normal authorized use harms other users or breaks a protocol invariant; intentional privileged powers alone are not findings
- Costs exceed extraction → reject a claim of profitable extraction only if no independent loss, denial of service, or state corruption remains
- Reachable trigger with source-backed unintended consequences → **clears**, continue, whether or not the triggerer profits

## Gate 4 — Impact

Prove a concrete adverse terminal state and identify who bears it. Material harm can be asset loss, an unpayable entitlement, a persistent lock, or a consequential accounting or governance error; attacker profit is not required. For a failed taxed-transfer claim, prove the recipient transfer failed while the tax remained settled, then establish why retaining that tax violates the specified transfer behavior. If the message sequence or refund obligation is uncertain, retain a `review_trail`.

- Voluntary, expected self-harm with no effect on others or protocol state → **REJECTED**
- Trivial bounded impact with no compounding → **DEMOTE** to a lead or lower-severity finding, with the bound explained
- Source-backed material harm → **CONFIRMED**, with conditional preconditions and actor control stated explicitly

## Confidence

Start at **100**, deduct: partial attack path **-20**, bounded non-compounding impact **-15**, requires specific (but achievable) state **-10**. Confidence ≥ 80 gets description + fix. Below 80 gets description only.

## Safe patterns (do not flag)

- Standard `throw_unless` sender validation checking against stored admin/owner address
- `end_parse()` enforcement after all fields loaded
- Proper bounce handlers that revert the corresponding state change
- `accept_message()` called AFTER signature/seqno validation in `recv_external`
- `raw_reserve` before `send_raw_message` to protect minimum balance
- Standard TEP-74 Jetton wallet address computation via StateInit hash
- Standard TEP-62 NFT ownership transfer with proper authorization
- Consistent protocol-favoring rounding unless compounding or zero-rounding

## Lead promotion

Before finalizing leads, revisit where warranted. Promotion still requires a source-backed trace through all four gates:

- **Cross-contract echo.** Check the same pattern in each affected contract; confirm each instance's reachability and impact separately.
- **Multi-agent convergence.** Use independent agreement to prioritize a focused trace, not as proof by itself.
- **Partial-path completion.** Complete the missing transition or retain the candidate as a lead with the exact uncertainty.

## Leads

High-signal trails for manual investigation. No confidence score, no fix — title, code smells, and what remains unverified.

## Do Not Report

Linter/compiler issues, gas micro-opts, naming, documentation. Intended admin powers without unintended downstream harm. Missing logging (TON has no Solidity-style events). Centralization without a concrete failure path. Implausible preconditions (but Jetton misbehavior, bounce failures, and gas exhaustion ARE plausible for contracts handling arbitrary tokens/messages).
