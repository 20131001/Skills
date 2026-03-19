# Finding Validation

Every candidate finding must pass the false-positive gate before it can be scored or reported.

Score **certainty**, not severity. A catastrophic exploit with weak evidence should score lower than a modest exploit with a fully proven path.

## FP Gate

Every finding must pass all three checks. If any check fails, drop the finding immediately. Do not score it. Do not report it.

1. **Concrete attack path exists.**
   You can trace a real path from caller -> reachable entrypoint -> vulnerable state change -> loss, lock, griefing, or broken invariant.
   Evaluate what the code actually allows, not what the deployer, admin, or integrator might choose to do off-chain.

2. **The attacker can reach the entrypoint.**
   Confirm the attacker can actually trigger the path after accounting for sender checks, access control, replay protection, init-phase restrictions, workchain checks, and any other gating logic.

3. **No existing guard already stops the exploit.**
   Check `throw_unless`, sender validation, `if`-revert branches, bounce filters, `seqno` checks, init flags, state locks, and equivalent guard logic.
   If a real guard blocks the exploit under the claimed path, drop the finding.

## Confidence Score

Confidence measures how certain you are that the finding is real and exploitable.

- Start every surviving finding at **100**.
- Apply **all** deductions that fit.
- Confidence is independent of severity.
- Report the final score as `[N]`.

Examples: `[95]`, `[75]`, `[60]`.

### Deductions

- Privileged caller required (owner, admin, multisig, governance) -> **-25**
- Attack path is partial (the idea is plausible, but you cannot fully write caller -> call -> state change -> outcome) -> **-20**
- Impact is self-contained (only the attacker's own funds or position are affected, with no broader user or protocol spillover) -> **-15**

## Threshold

Default confidence threshold: **75**

- Findings at or above the threshold get a normal finding entry with a `Fix` section.
- Findings below the threshold may still be listed, but they do **not** get a `Fix` section.

## Reporting Rules

- Report the **root cause**, not every downstream symptom.
- Do not report the same root cause twice under slightly different titles.
- If two findings compound, keep both only when they are meaningfully distinct; otherwise keep the stronger root-cause version.
- If you cannot explain the exploit path in concrete terms, drop the finding.

## Do Not Report

- Anything a linter, compiler, or experienced reviewer would dismiss: style issues, naming, formatting, NatSpec, redundant comments, or minor code hygiene.
- Gas micro-optimizations or efficiency notes with no concrete exploit path.
- Missing logs, events, or observability-only concerns.
- Pure centralization or trust observations with no distinct exploit path.
  Example: "owner could rug" is not a vulnerability unless the code exposes a specific unsafe mechanism beyond the intended trust model.
- By-design privileged actions such as owner/admin ability to set parameters, fees, pause state, or upgrade configuration.
- Theoretical issues that require implausible assumptions.
  Examples: compromised compiler, malicious validator majority, corrupted toolchain, or attacker control over unrealistic external preconditions.
- Issues that depend on integrations behaving incorrectly when the audited contract itself enforces the relevant safety property.
- Duplicate findings that restate the same bug from another angle without a distinct exploit path.
