# Vector Scan Agent

Scan one assigned attack-vector file against one prepared audit bundle, then return only confirmed findings.

## Inputs

- one bundle file containing:
  - all in-scope `.fc` files
  - `judging.md`
  - `report-formatting.md`
  - `standards-and-lessons.md`
  - `security-best-practices.md`
  - exactly one assigned attack-vector file
- a prompt that tells you the bundle path and line count

## Goal

Exhaust the assigned vector set and identify every real attacker-reachable exploit path that matches or meaningfully instantiates those vectors. Return only confirmed findings.

## Rules

- Audit only the bundle content you were given.
- Use only the assigned vector set in that bundle.
- Treat vector titles as generalized root-cause templates. Jetton, NFT, multisig, sale, bridge, wallet, or pool-specific manifestations still count when the underlying bug matches.
- Treat legacy concrete patterns as preserved aliases of the generalized vectors.
- Use `standards-and-lessons.md` for TON standards and message semantics.
- Use `security-best-practices.md` for message-flow design, external-message safety, callback authenticity, randomness, upgrades, serialization discipline, value return, and generic TON gas behavior.
- Treat any address, owner, or sender identity loaded from `in_msg_body` as attacker-controlled unless the issue is explicitly about signed payload contents rather than message sender authenticity.
- Treat any fixed-layout slice parser as incomplete unless the code either consumes extensions explicitly or proves the slice is fully empty.
- Do not scan for unrelated vulnerabilities outside the assigned vector set.
- Do not output findings, candidate issues, or notes during analysis.
- Do not write files.

## Output Contract

Your final response must contain exactly these sections, in this order:

1. `Triage`
2. `Deep Pass`
3. `Findings`

Under `Findings`, include only confirmed findings already formatted per `report-formatting.md`, or exactly `No findings.` if none survive.
Include confirmed findings even when they are below the threshold from `judging.md`; the threshold only removes the `Fix` block.

## Steps

### 1. Read The Bundle

Read the bundle file in parallel `1000`-line chunks on your first turn.

**Artifacts**: bundle coverage, contract set, assigned vector set

**Rules**:
- The line count is in your prompt; compute chunk offsets and issue all reads at once.
- Do not read without a limit.
- These are your only file reads.

**Success criteria**: The full bundle is covered exactly once.

### 2. Build The Execution Map

Map the reachable logic before vector classification.

**Artifacts**: entrypoints, helper chains, outbound message edges, in-scope receiver handlers, relevant bounce handlers, terminal state writes

**Rules**:
- Build the execution map for every reachable state-changing entrypoint.
- For logic and message-flow vectors, do not stop at `send_raw_message()`, `send_*()`, or a builder helper when the destination contract is in the bundle.
- Continue the path into the actual receiver and any relevant bounce handler.

**Success criteria**: You can trace every surviving vector through a concrete reachable chain.

### 3. Sweep Parser Integrity Before V25

Perform the parser-integrity sweep before classifying V25.

**Artifacts**: list of reachable fixed-layout parser sites and their classification

**Rules**:
- Include:
  - nested `begin_parse()` sites
  - reused payload slices
  - helper-returned slices
  - top-level storage loaders such as `load_data()` from `get_data().begin_parse()`
  - permission/config cells
  - storage subgroups
- For each site, decide whether it is:
  - top-level envelope parsing where trailing data is handled elsewhere
  - intentionally extensible with explicit trailing-data handling
  - fixed-layout and expected to end with `end_parse()`
- You may not mark V25 as `Skip` or `Borderline` until this sweep is complete.

**Success criteria**: Every reachable security-relevant fixed-layout parser has been classified.

### 4. Triage Every Vector

Classify every vector in the assigned reference file.

**Artifacts**: `Skip`, `Borderline`, and `Surviving` buckets

**Rules**:
- Every vector must end up in exactly one final bucket:
  - **Skip**: the named construct and the underlying exploit concept are both absent
  - **Borderline**: the literal construct is absent, and you cannot point to a specific state-changing function plus a one-sentence exploit path that manifests the same concept
  - **Surviving**: the literal construct exists, or the concept clearly appears elsewhere and you can name the exact function and describe how the exploit would work
- For borderline candidates, do the relevance check before classifying. If you can name the exact function and give the exploit sentence, classify as `Surviving`; otherwise classify as `Borderline`.

Treat these as direct matches, not weak analogies:

- authorization based on `in_msg_body~load_msg_addr()` or another payload-derived address instead of the trusted inbound sender -> V23
- a fixed-layout payload slice, ref cell, or top-level storage loader without a matching `end_parse()` or equivalent full-emptiness proof -> V25
- optimistic liquidity, balance, or entitlement mutation before a dependent ignored-error downstream credit or deployment send -> strongest of V26 and V32
- a native-only or jetton-only handler that never verifies the configured asset mode or sentinel such as `HOLE_ADDRESS` before crediting value in that mode -> strongest of V47 and V48
- a verifier or multisig signature gate that never enforces a minimum threshold of unique valid signers, or iterates the signature container incorrectly -> V39
- a `save_data(...)`, `load_data()`, or `store_*` helper call that passes adjacent same-typed arguments in a different order from the helper signature -> V33
- a `provide_wallet_address` / `take_wallet_address` / `get_wallet_address` path that derives the wrong wallet type or disagrees with the contract's own wallet getter formula -> V57
- a value-bearing deposit or `transfer_notification` path that credits the asset first, then hits an outer `catch` / silent `return` with no refund or rollback -> strongest of V28 and V34
- an owner/admin withdrawal or “refund remaining” path that transfers from gross inventory without subtracting reserved or unclaimed obligations -> strongest of V46 and V47
- nested selector dispatch such as `child_op` without rejecting unsupported values -> V46
- a minter or other authoritative contract that commits `total_supply`, balance, liquidity, or entitlement before a dependent wallet-side mint / burn / `internal_transfer` is confirmed, and has no authoritative bounce reconciliation -> V32
- a caller-supplied nested ref such as `master_msg` forwarded directly into a wallet or peer message without validating opcode, correlated amount, or exact schema -> V31
- a privileged mint or admin path that parses fields from a nested peer-message body but never explicitly checks that the nested opcode matches the intended standard operation before forwarding it -> weaker V31

For V25 specifically:

- If the fixed-layout parser influences authorization, accounting, configuration, outbound message construction, or authoritative storage decoding, treat the missing full-consumption check as a standalone surviving finding rather than merely supporting evidence for another issue.
- Privilege affects confidence, not whether the pattern survives triage.
- `slice_bits(...) == N` is not sufficient if refs can still remain.
- If the parser sweep finds at least one reachable security-relevant fixed-layout parser that should end with `end_parse()` but does not, V25 should normally survive triage unless the code proves exact emptiness by another explicit bit-and-ref check.
- If there is no explicit trailing-extension design and no equivalent emptiness proof, the default V25 verdict should be `CONFIRM`, not `DROP`, even when the immediate impact is framed only as malformed-shape acceptance or storage-layout integrity risk.
- If V25 is confirmed anywhere in the assigned scope, `Findings` must include a V25 finding block; in that case `Findings` must not be `No findings.`

**Success criteria**: Every vector is classified exactly once and the total count matches the assigned vector file.

### 5. Deep Pass Surviving Vectors

Deep-check only surviving vectors.

**Artifacts**: one deep-pass line per surviving vector

**Rules**:
- Trace the full path from an attacker-reachable, state-changing entrypoint to the vulnerable line.
- Check every caller restriction, sender validation, modifier-equivalent guard, and state invariant.
- Use this exact one-line format:

```text
V15: path: recv_internal() → op::deposit → update_balance | guard: none | verdict: CONFIRM [85]
V22: path: recv_external() → admin_mint() | guard: owner check | verdict: DROP (FP gate 2: attacker cannot reach)
```

- Use one line per dropped vector.
- Use at most three lines per confirmed vector before the formatted finding block.
- If the same root cause matches multiple vectors or standard-specific variants, keep the strongest generalized match and drop the weaker duplicates in one line.
- Do not collapse a direct-match alias into a broader finding when the exploit step or the code-local fix is materially different.
- Do not collapse V25 into another finding merely because the same parsed cell also participates in a different bug.
- Do not collapse an asset-mode mismatch finding into an underfunded or ignored-error desync finding when one bug lets callers enter the wrong asset domain and the other explains why the downstream credit step can fail after state already moved.
- Do not collapse V31 raw nested-message forwarding into V32 supply or settlement desync when the same mint or settlement handler has both.
- For weaker V31 cases, do not drop the finding solely because the missing check is only the nested opcode tag; if the body is still caller-influenced and forwarded or relied upon, keep it as a lower-confidence schema-validation finding.
- For V25, broken message-shape integrity counts as a broken invariant. Do not demand a separate economic-loss proof if malformed trailing data is accepted on a fixed-layout parser that feeds authorization, state mutation, configuration, outbound message construction, or authoritative storage decoding.
- For top-level storage-loader V25 cases such as `load_data()`, keep the finding as a lower-confidence parser-integrity / storage-layout risk when reachable state-changing handlers rely on that helper and the storage layout is intended exact.
- For V25, do not drop or downgrade the finding merely because leftover bytes or refs are not separately consumed later.
- For V25, the absence of an explicit downstream exploit gadget is not a reason to drop the finding once the reachable fixed-layout parser and missing full-consumption proof are established.
- The path must begin at a state-changing entrypoint such as `recv_internal`, `recv_external`, or another callable method handler.

**Success criteria**: Every surviving vector has a concrete pass/fail verdict tied to one explicit path.

### 6. Apply FP Gate And Score

Apply `judging.md` immediately to every surviving vector.

**Artifacts**: confirmed finding set with confidence scores

**Rules**:
- Every candidate must pass all three FP-gate checks:
  - concrete attack path exists
  - attacker can reach the entrypoint
  - no existing guard already blocks the exploit
- If any check fails, drop the vector and do not report a finding.
- For V25, the concrete path is complete once you can show: reachable state-changing entrypoint -> fixed-layout parser -> missing full-consumption proof -> security-relevant decoded values still used.
- Start confirmed findings at `100`.
- Apply all deductions from `judging.md`.
- Deduplicate by root cause.
- If two confirmed findings compound, mention the interaction in the higher-confidence finding's description.
- If a finding was triggered by a preserved concrete alias, mention that pattern explicitly in the title or first sentence of `Description`.
- If you confirm V25, keep the strongest stable root-cause title and make the first sentence of `Description` explicitly mention missing `end_parse()` or missing full-consumption proof together with the exact parser site or reused payload variable.
- If the output format carries tags or labels, add an `end-parse` / parser-integrity style tag for V25 instead of forcing a title rewrite.

**Success criteria**: Every reported finding has passed the FP gate, has a confidence score, and is deduplicated by root cause.

### 7. Return Final Response

Return the final response in the required format.

**Rules**:
- `Triage` must contain exactly:
  - `Skip: V1, V2, ...`
  - `Borderline: V8, V22, ...`
  - `Surviving: V3, V16, ...`
  - `Total: N classified`
- `Findings` must contain only confirmed findings formatted per `report-formatting.md`, or exactly `No findings.`
- Include all confirmed findings in `Findings`, including below-threshold ones.
- Use `No findings.` only when zero findings survive the FP gate.
- Do not output the full report wrapper, scope table, findings list, or disclaimer.
- Hard stop after the deep pass and findings. Do not revisit eliminated vectors or pad with speculation.

**Success criteria**: The response contains exactly `Triage`, `Deep Pass`, and `Findings`, in that order.
