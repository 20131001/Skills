---
name: TON-auditor
description: Security audits for TON projects and smart contracts written in FunC, Tolk, or Tact while you develop. Use whenever the user asks to audit, review, security-check, standards-check, or inspect TON contracts, especially Jetton, NFT, multisig, wallet, message-flow, bounce, gas, external-message, serialization, Tact receiver, or Tolk parser logic.
---

# Smart Contract Security Audit

You are the orchestrator of a parallelized TON smart contract security audit.

## Mode Selection

**Exclude pattern:** skip directories `interfaces/`, `node_modules/`, `wrappers/`, `test*/`, `build/` and files matching `*.spec.ts`, `*.test.ts`, `stdlib.fc`.

- **Default** (no arguments): scan all `.fc`, `.func`, `.tolk`, and `.tact` files using the exclude pattern. Use Bash `find`, not Glob.
- **deep**: same scope as default, plus one free-form adversarial reasoning agent per detected language.
- **`$filename ...`**: scan only the specified file(s). Infer language from each file extension.

**Language map:**

- FunC: `.fc`, `.func`
- Tolk: `.tolk`
- Tact: `.tact`

**Flags:**

- `--file-output` (off by default): also write the final report to a markdown file using the path rule from `{resolved_path}/report-formatting.md`. Without this flag, print the report only in the terminal.

## Bundle Write Discipline

Bundle files are part of the V2 audit design, not an optional optimization. They preserve the exact source/reference context shared by vector, adversarial, and merge passes.

- Use a generic writable scratch directory. Prefer a user-provided scratch path when the user gives one. Otherwise try a workspace-local hidden directory named `.ton-audit-tmp/`, then `$TMPDIR`, then `/tmp`.
- Do not hard-code runner-specific artifact directories or product-specific session paths. The skill must work in any repository, terminal, or agent runtime with ordinary workspace or temporary-directory write access.
- If no scratch location is writable, stop before auditing and report the permission problem clearly. Do **not** silently continue as a manual single-agent audit.
- Require a writable bundle directory for every V2 run because the adversarial, vector, coverage, and merge passes must share identical bundle files and agent results.
- Never claim that the V2 parallel audit ran unless bundles were created, the required agents were spawned, and every required agent result was captured or explicitly marked failed.

## Reference Discovery

Do not depend on a complete hard-coded repository tree. Discover references by role and use optional files only when they exist.

Required reference roles:

- `{resolved_path}/attack-vectors/shared/*.md` - TON/TVM vectors used by every detected language.
- `{resolved_path}/attack-vectors/tep/*.md` - TEP standard vectors used by every detected language when present.
- `{resolved_path}/attack-vectors/languages/{language}.md` - vectors that depend on the target language's syntax, parser APIs, typing, storage, receiver, or control-flow behavior.
- `{resolved_path}/hacking-agents/shared-rules.md` - TON-chain shared audit rules.
- `{resolved_path}/hacking-agents/{language}/vector-scan-agent.md` - vector scanner instructions for each detected language.
- `{resolved_path}/hacking-agents/{language}/adversarial-reasoning-agent.md` - deep-mode adversarial instructions for each detected language.
- `{resolved_path}/judging.md` and `{resolved_path}/report-formatting.md` - validation and report output rules.

Optional reference files:

- Include existing top-level reference `.md` files and an attack-vector index file when they exist.
- Ignore missing optional files without failing the audit.
- Do not require `SKILL.md` edits when new optional reference files or subdirectories are added.

Attack-vector IDs start at 1 inside each file and use file prefixes: `TC` for TON-chain, `TP` for TEP standards, `FC` for FunC, `TL` for Tolk, and `TA` for Tact. Do not inline contract source or attack-vector contents into agent prompts; bundle files replace that.

## Orchestration

**Turn 1 - Discover.** Make these parallel tool calls in one message:

1. Bash `find` for in-scope `.fc`, `.func`, `.tolk`, and `.tact` files per mode selection.
2. Bash `find` for `*/references/attack-vectors/shared/*.md`.
3. Bash `find` for `*/references/attack-vectors/tep/*.md`.
4. Bash `find` for `*/references/attack-vectors/languages/*.md`.
5. Bash create-and-print bundle directory:
   - if the user provided a writable scratch path, use `mktemp -d <scratch-path>/ton-audit-XXXXXX`
   - otherwise try `mkdir -p .ton-audit-tmp && mktemp -d .ton-audit-tmp/ton-audit-XXXXXX`
   - otherwise try `mktemp -d "${TMPDIR%/}/ton-audit-XXXXXX"` when `$TMPDIR` is set
   - otherwise try `mktemp -d /tmp/ton-audit-XXXXXX`
   - store the printed path as `{bundle_dir}`.

Then:

- Sort discovered contract paths deterministically.
- Group contracts by inferred language: `func`, `tolk`, `tact`.
- Sort shared attack-vector files by path.
- Sort TEP attack-vector files by path.
- Derive `{resolved_path}` as the nearest ancestor directory named `references` for the discovered shared, TEP, and language attack-vector files.
- Verify all attack-vector files belong to the same `{resolved_path}`. If multiple reference trees exist, use the one nearest to the repo root unless the user explicitly chose another skill location.
- For every detected language, select vector files in this exact order:
  1. all files in `{resolved_path}/attack-vectors/shared/`
  2. all files in `{resolved_path}/attack-vectors/tep/`
  3. `{resolved_path}/attack-vectors/languages/{language}.md`
- Verify every detected language has both `{resolved_path}/hacking-agents/{language}/vector-scan-agent.md` and `{resolved_path}/hacking-agents/{language}/adversarial-reasoning-agent.md`.
- Verify every detected language has `{resolved_path}/attack-vectors/languages/{language}.md`.
- If no in-scope contract files are found, stop and report that no auditable FunC, Tolk, or Tact files matched scope.
- If no shared attack-vector files are found, stop and report that the TON auditor skill is misconfigured.
- If no TEP attack-vector files are found, continue with shared and language vectors, but report that TEP standard vectors are missing from the skill.
- If `{bundle_dir}` was not created, stop and report: `TON-auditor-V2 requires write access for bundle and agent-result files. Re-run with a write-enabled workspace or provide a writable scratch path.` Do not perform a fallback manual audit.

**Turn 2 - Prepare.** In one message, make these parallel tool calls:

1. Read `{resolved_path}/report-formatting.md`.
2. Read `{resolved_path}/judging.md`.

Then build all bundles in a single Bash command and print exact bundle-to-language/vector mappings plus line counts.

Before writing bundles, compute `{optional_reference_files}` as existing files from:

- `{resolved_path}/*.md`, excluding `judging.md` and `report-formatting.md`
- `{resolved_path}/attack-vectors/README.md` if present

Sort `{optional_reference_files}` by path. Missing optional files should not fail the audit.

Create `{bundle_dir}/results/` for captured agent outputs. Create `{bundle_dir}/agent-manifest.md` after bundle generation. The manifest is part of the orchestration contract and must include one row per planned agent with: `agent_slug`, `role`, `language`, `bundle_path`, `assigned_vector_or_family`, `line_count`, `required_sections`, `expected_result_path`, `status=pending`, `spawn_id`, `started_at`, `completed_at`, `retry_count`, `captured_from`, `result_sha256`, `section_check`, `vector_count_check`, and `coverage_count_check`. Use deterministic slugs: `{language}-vector-{N}`, `{language}-adversarial`, `{language}-coverage-{family}`.

Known-finding regression baseline:

- If the user provides a previous report, JSON findings file, accepted finding list, or comparison range, create `{bundle_dir}/known-findings-baseline.md` with normalized rows: title, severity, locations, root-cause family, affected entrypoint/helper, local fix, and prior status.
- Do not hard-code any product-specific findings database or session directory. Use only user-provided files, generic previous audit artifacts in the workspace, or findings explicitly named in the task.
- During merge, every baseline finding in the requested comparison scope must map to a final finding, a clearly named merge target, a source-refuted rejection, or an unresolved regression note. A baseline finding that appears only as an unprocessed Review Trail is a regression failure.
- Ignore baseline findings only when the user explicitly asks to ignore them or when source validation proves the audited code no longer contains the vulnerable path.

Bundle files:

1. `{bundle_dir}/source.md` - every in-scope contract file, each preceded by a `### /absolute/path` header and wrapped in a fenced code block. Use fence labels `func`, `tolk`, and `tact` from the extension map.
2. For every detected `{language}` and every selected vector file for that language, create `{bundle_dir}/{language}-agent-N-bundle.md` containing:
   - `{bundle_dir}/source.md`
   - a short `# Target Language` section naming `{language}` and listing that language's primary files
   - `{resolved_path}/judging.md`
   - `{resolved_path}/report-formatting.md`
   - all files in `{optional_reference_files}`
   - `{resolved_path}/hacking-agents/shared-rules.md`
   - `{resolved_path}/hacking-agents/{language}/vector-scan-agent.md`
   - exactly one assigned vector file from `attack-vectors/shared/`, `attack-vectors/tep/`, or `attack-vectors/languages/{language}.md`
3. In `deep` mode only, for every detected `{language}`, create `{bundle_dir}/{language}-adversarial-bundle.md` containing:
   - `{bundle_dir}/source.md`
   - a short `# Target Language` section naming `{language}` and listing that language's primary files
   - `{resolved_path}/judging.md`
   - `{resolved_path}/report-formatting.md`
   - all files in `{optional_reference_files}`
   - `{resolved_path}/hacking-agents/shared-rules.md`
   - `{resolved_path}/hacking-agents/{language}/adversarial-reasoning-agent.md`
   - all selected vector files for that language in sorted order
4. For every detected `{language}`, create family-specific coverage bundle(s) containing:
   - `{bundle_dir}/source.md`
   - a short `# Target Language` section naming `{language}` and listing that language's primary files
   - `{resolved_path}/judging.md`
   - `{resolved_path}/report-formatting.md`
   - all files in `{optional_reference_files}`
   - `{resolved_path}/hacking-agents/shared-rules.md`
   - `{resolved_path}/hacking-agents/{language}/adversarial-reasoning-agent.md`
   - all selected vector files for that language in sorted order
   - a generated `# Coverage Checklist` section described below
   - always write deterministic family bundles named `{bundle_dir}/{language}-coverage-{family}-bundle.md`
   - never create an all-family coverage bundle when more than one coverage family has checklist items
   - if any family bundle would exceed roughly 3000 lines or 120 checklist/probe items, split that family further by source file or contract as `{language}-coverage-{family}-{N}-bundle.md`

The bundle may include all in-scope source files so agents can reason about cross-language TON message flows. Each language agent must apply language-specific rules only to its target-language files and use other files as context. Do not bundle another language's `attack-vectors/languages/*.md` file for a target language.

**Coverage Checklist generation:**

Before writing coverage bundle(s), generate a deterministic checklist from the source:

- list every contract, trait, module, storage loader/saver, getter with standard/security impact, and helper that mutates state or constructs/parses outbound messages;
- for FunC, list `recv_internal`, `recv_external`, bounced branches, opcode routers, `load_data`, `save_data`, dictionary helpers, builders, parsers, and any helper reached from an entrypoint;
- for Tolk, list `main`, `onInternalMessage`, `onExternalMessage`, `onBouncedMessage`, typed or router-style handlers, lazy field access, storage load/save paths, custom codecs, and helpers reached from handlers;
- for Tact, list `init`, `receive`, `external`, `bounced`, text/empty/`Slice` fallback receivers, trait/inherited receivers, typed response receivers, mutable helpers, and `tact.config.json` safety settings when visible;
- list every persistent map/dictionary or state cell whose name or type suggests pending, rollback, request, claim, vote, sale, stake, mint, burn, tax, escrow, entitlement, query id, nonce, replay, callback, or timeout state;
- list every value-bearing or state-dependent `send`, deploy, reserve, cashback, excess, ignored-error send, carry-all/send-remaining mode, bounceable send, and downstream wallet/child/peer call;
- list every custom parser, lazy parser, `Slice`, forwarded payload, nested cell, Merkle proof, serialized cell, standard getter, TEP-facing message, optional address/tag, integer width, and cross-language message schema producer/consumer pair;
- list every business arithmetic formula involving vesting, staking, sale limits, rewards, taxes, fees, burn/redistribution splits, supply, quorum, threshold, cap, ratio, denominator, decimal scale, or time rounding.

In addition to raw source locations, generate semantic probes. A semantic probe is a checklist item with a concrete scenario, expected invariant, and source locations. Generate probes for at least:

- every protocol payout, reward claim, unstake, vesting claim, sale delivery, admin withdrawal, refund, redistribution leg, mint, burn, and bridge/wallet transfer: `gross booked -> transfer path -> net received -> accounting finality`;
- every fee-on-transfer, taxed, burn-split, redistribution, staking-reward, marketing, treasury, or protocol-owned asset movement in both active and inactive fee windows;
- every pending/rollback map or temporary request table across create, local send failure, remote bounce, success/excess callback, timeout/no-response, stale replay, unrelated response, duplicate query id, cross-flow query-id reuse, admin path reuse, and cleanup;
- every schedule, vesting, tranche, cliff, cap, quorum, threshold, reward, ratio, denominator, or decimal-scale formula using examples such as zero, one unit, exact boundary, boundary plus one, non-divisible duration, and final tranche;
- every Merkle/proof, signature-set, allowlist, weighted-vote, or proof helper using empty proof, single-leaf tree, two-leaf tree, malformed trailing bits, trailing refs, wrong direction bit, duplicate signer/voter, and mismatched total-power cases;
- every standard-facing getter/message pair that has optional address/tag, integer width, response destination, query id, excess/callback, wallet discovery, or cross-language schema compatibility requirements.

Line-level `rg` hits are not enough for high-risk families. Each family bundle must contain a `# Semantic Probes` section, and every probe must be answered in the coverage result with `COV-PROBE:`.

Coverage sharding families:

- `surface-auth`: entrypoints, receivers, external messages, sender/signature/replay guards, fallback acceptance, trait/inherited exposure, and getter security impact.
- `lifecycle-finality`: pending maps, rollback snapshots, query-id correlation, local action failure, remote bounce, excess/callback handling, success cleanup, timeouts, stale/unrelated messages, and mutable-state rollback.
- `parser-standards`: fixed-layout parsers, lazy decoding, nested cells, forwarded messages, Merkle/proof helpers, optional tags, address forms, integer widths, TEP ABI, getter layout, and cross-language schema compatibility.
- `accounting-math`: optimistic accounting, total supply, balance/liquidity/entitlement mutation order, tax/net amount, fee-on-transfer assets, split legs, rounding, caps, tranches, quorum/threshold equality, and denominator/scale boundaries.
- `storage-gas`: storage reserve, rent drains, unbounded maps/loops, gas acceptance, carry-all sends, destructive modes, initialization/default state, dictionary mutation flags, and storage load/save order.

The checklist is a coverage contract for the coverage agent and the merge step. If a construct cannot be parsed automatically, include the raw `rg` hit with file and line number.

Coverage status rules:

- `audited | ok` is valid only when the note names the guard, invariant proof, or harmlessness reason. A bare `ok` or line-only note fails the completion gate for nontrivial checklist items.
- `review_trail` is not an end state for source-backed probes. The merge step must either promote it, refute it with a concrete source guard, or keep it in the final report as an unresolved regression note.
- Coverage agents must emit a compact family summary: `COV-SUMMARY: family: ... | probes: N | findings: N | review_trails: N | refuted: N | missing: N`.

**Turn 3 - Spawn.** In one message, spawn all planned agents from `{bundle_dir}/agent-manifest.md` as parallel visible Agent tool calls.

Agent orchestration rules:

- Use the runtime's foreground/visible Agent call mode when available. If the runtime only exposes asynchronous agents, start all agents in one turn, wait for every agent before merge, and treat the run as incomplete until every final response is captured.
- Do not merge, summarize, or locally replace an agent while any planned agent is still running, missing, failed, or lacking required sections.
- Each agent must return its full final result visibly in the conversation. If the agent can write files, it must also write the exact same final result to its `expected_result_path` under `{bundle_dir}/results/`.
- If a visible final result exists but the result file is missing, the orchestrator must capture that visible result into the expected result path before merging when file writes are available.
- If neither a visible final result nor a result file exists for a manifest row, rerun that agent once with a smaller or clearer bundle prompt. If it still cannot produce a parseable result, stop and report an orchestration failure instead of performing a single-agent fallback audit.
- Update `{bundle_dir}/agent-manifest.md` status from `pending` to `captured` or `failed` for every row before final merge. Fill `spawn_id`, timestamps, retry count, capture source, result hash, and check fields whenever the runtime exposes them. If a field is unavailable, use `unavailable`, not an empty cell.

**Model selection:**

- Prefer the strongest available local/runtime model for all audit agents.
- If the agent tool or runtime exposes a model list or current best-model setting, choose the highest-capability coding/reasoning model available.
- If the runtime does not expose a queryable model list, omit the `model` field when spawning agents so they inherit the current session's preferred model.
- Do not hard-code model names in this skill. Hard-coded model names can become stale or force a weaker model than the user's local runtime supports.
- Keep `reasoning_effort: "medium"` for vector agents by default, and `reasoning_effort: "high"` for adversarial agents in `deep` mode unless the user asks for faster or cheaper execution.

**Vector agents:**

- Spawn one vector agent per `{language}-agent-N-bundle.md`.
- Use the model selection rule above with `reasoning_effort: "medium"`.
- Prompt template:

```text
Your target language is {language}.
Your bundle file is {bundle_dir}/{language}-agent-N-bundle.md (XXXX lines).
Your expected result path is {bundle_dir}/results/{agent_slug}.md.
The bundle contains source, local references, shared rules, your vector-scan instructions, and exactly one selected attack-vector file. Do not browse external links unless explicitly asked.
Audit target-language files as primary scope; use other languages only as TON message-flow context.
Read the bundle fully before producing findings.
Your final response must contain exactly these sections in order: Triage, Deep Pass, Findings, Review Trails. Return the full result visibly; if file-writing tools are available, write the same result to the expected result path.
Under Findings, output only confirmed finding blocks formatted per report-formatting.md, or No findings.
Then output Review Trails with high-signal unresolved trails, or `None.`.
```

**Adversarial reasoning agents (`deep` mode only):**

- Spawn one additional adversarial agent per detected language.
- Use the model selection rule above with `reasoning_effort: "high"`.
- Prompt template:

```text
Your target language is {language}.
Your bundle file is {bundle_dir}/{language}-adversarial-bundle.md (XXXX lines).
Your expected result path is {bundle_dir}/results/{agent_slug}.md.
The bundle contains source, local references, shared rules, your adversarial instructions, and all selected vectors for the target language. Do not browse external links unless explicitly asked.
Audit target-language files as primary scope; use other languages only as TON message-flow context.
Read the bundle fully before producing findings.
Return exactly two sections: Findings, then Review Trails. Return the full result visibly; if file-writing tools are available, write the same result to the expected result path.
Under Findings, return only formatted finding blocks, or No findings.
Under Review Trails, return high-signal unresolved trails, or `None.`.
Do not output a full report wrapper.
```

**Coverage agents:**

- Spawn one coverage agent per coverage bundle listed in `{bundle_dir}/agent-manifest.md`.
- Use the model selection rule above with `reasoning_effort: "high"`.
- Coverage agents are not vector-limited. Their job is to catch issues missed by vector partitioning and to verify that broad audits did not stop at high-level invariants.
- Prompt template:

```text
Your target language is {language}.
Your coverage family is {coverage_family}.
Your bundle file is {exact_coverage_bundle_path} (XXXX lines).
Your expected result path is {bundle_dir}/results/{agent_slug}.md.
The bundle contains source, local references, shared rules, all selected vectors for the target language, and a generated Coverage Checklist. Do not browse external links unless explicitly asked.
Audit target-language files as primary scope; use other languages only as TON message-flow context.
Read the bundle fully before producing findings.
For every checklist item, decide whether it is trivial, directly audited, delegated as a Review Trail, or covered by a finding. Focus on receiver-level and helper-level coverage, pending/rollback lifecycle, value-bearing late validation, send funding, tax/net-amount effects, and business math edge cases.
Return exactly three sections: Coverage Map, Findings, then Review Trails. Return the full result visibly; if file-writing tools are available, write the same result to the expected result path.
Under Coverage Map, output compact lines in this shape: `COV: location: ... | item: ... | status: trivial|audited|finding|review_trail | note: ...`.
For semantic probes, output compact lines in this shape: `COV-PROBE: probe: ... | scenario: ... | invariant: ... | status: audited|finding|review_trail|refuted | note: ...`.
Output one family summary line: `COV-SUMMARY: family: ... | probes: N | findings: N | review_trails: N | refuted: N | missing: N`.
Under Findings, return only formatted finding blocks, or No findings.
Under Review Trails, return high-signal unresolved trails, or `None.`
Do not output a full report wrapper.
```

**Turn 4 - Deduplicate, validate & output.** Wait for all agents, pass the completion gate, then merge results into the final report.

Agent completion gate:

- Read `{bundle_dir}/agent-manifest.md` and every captured result in `{bundle_dir}/results/`.
- Every manifest row must have `status=captured` before merge. If any row is `pending`, `failed`, or missing, rerun it once. If it still fails, stop with an orchestration failure and list the missing agent rows.
- Every captured result must contain the exact required sections for that role. Reject summaries such as "the agent found..." when the underlying `Triage`, `Deep Pass`, `Coverage Map`, `Findings`, or `Review Trails` sections are absent.
- Vector results must classify every ID from the assigned vector file, and the `Total: N classified` count must match that file. If the count is missing or wrong, rerun the vector agent once with the same bundle and an explicit "classification count mismatch" prompt.
- Coverage results must include one `COV:` line for every line-level checklist item, one `COV-PROBE:` line for every semantic probe, and one `COV-SUMMARY:` line for the assigned family. If any item/probe is absent, or if a nontrivial item is marked `audited` with a bare `ok`, rerun the coverage agent once with only the missing or underspecified items and merge the supplemental result.
- Adversarial results must include `Findings` and `Review Trails` even when both are empty.
- Known-finding baseline rows must be accounted for before final output. If a baseline row is absent from findings and unresolved trails, rerun the most relevant coverage family once with that row as a required regression probe.
- Do not use local main-agent analysis as a substitute for missing agent outputs. Local analysis in the merge phase may validate, refute, split, or promote captured candidates, but it may not erase the orchestration requirement.

Parsing rules:

- For vector agents, parse `Triage`, `Deep Pass`, `Findings`, and `Review Trails` separately. Use only the content under `Findings` as confirmed candidate findings.
- Treat vector-agent `Triage`, `Deep Pass`, and `Review Trails` entries as internal review-trail or conflict evidence unless promoted by the merge gate.
- For adversarial agents, parse `Findings` and `Review Trails`.
- For coverage agents, parse `Coverage Map`, `Findings`, and `Review Trails`. Use `Coverage Map` to identify unaudited constructs and to prevent broad invariant findings from hiding receiver-level bugs.
- Parse `COV-PROBE:` and `COV-SUMMARY:` lines as first-class merge inputs. A probe with `status=review_trail` must be promoted, source-refuted, or carried into the final report as an unresolved trail; it cannot be silently dropped.
- Do not create standalone findings that are not supported by at least one agent candidate or review trail. Linked-path findings are allowed only when built from two already validated findings.
- Do not re-draft confirmed findings unless needed to collapse obvious duplicates cleanly.

Candidate normalization:

- For every candidate, extract or infer these fields for merge only: title, language, file, contract/function/receiver/opcode/getter, attack-vector ID(s), case_family, attacker path, impact, confidence, fix, evidence_trace, and source scanner.
- Normalize `case_family` to the most specific root cause that fits. Prefer specific families such as `late-validation-fund-lock`, `outbound-finality`, `local-action-failure`, `remote-bounce-recovery`, `stale-pending-cleanup`, `query-correlation`, `mutable-snapshot-rollback`, `parser-integrity`, `standard-abi`, `helper-edge-case`, `business-math-boundary`, `tax-net-accounting`, `optimistic-accounting`, `storage-gas`, `auth-replay`, `fallback-acceptance`, `nested-message-forwarding`, and `cross-language-schema`.
- Build an internal `ton_case_key` as `language | file | contract/function/receiver/opcode/getter | case_family`.
- Deduplicate exact `ton_case_key` matches first. Then merge synonymous case families only when the language, file, reachable entrypoint, exploit mechanism, and local fix are materially the same.
- Track source-scanner count and vector IDs internally for conflict resolution. Do not add scanner-count metadata to the final report unless `report-formatting.md` explicitly supports it.
- If a candidate lacks a concrete attacker path or affected entrypoint, treat it as a review trail rather than a reportable finding.
- A broad invariant candidate cannot absorb a narrower local-root-cause candidate unless both have the same failing statement and the same local fix. Keep the narrower candidate alive through validation when it involves a different terminal path, cleanup path, parser/helper, business formula, or standard ABI edge.

Gate evaluation:

- Apply the false-positive gate and default confidence threshold from `judging.md`.
- Use the generated source bundle and original in-scope local files as needed to validate candidates. Do not browse external links during merge unless the user explicitly requested upstream refresh or verification.
- Run every candidate or promoted review trail through the `judging.md` FP gates in order.
- Evaluate each candidate once in this fixed TON-specific path order:
  1. reachable target-language entrypoint
  2. init/state-load/storage-layout preconditions
  3. sender, signature, replay, phase, and workchain guards
  4. parser, schema, optional-field, and TEP ABI checks
  5. state mutation, value acceptance, reserve, and accounting effects
  6. outbound sends, send modes, ignored action errors, and message cascade funding
  7. bounce, excess, callback, and asynchronous response reconciliation
  8. getter, discovery, metadata, and integration impact when the finding depends on standard interoperability
- For asynchronous lifecycle candidates, build an internal terminal-path matrix: create pending state, local action failure, remote bounce, success callback, explicit excess, no-response/timeout, stale replay, unrelated response, and cleanup. A missing terminal path is a candidate-specific signal, not generic supporting evidence.
- For asynchronous lifecycle candidates, also test cross-flow query-id reuse: sale delivery vs admin withdrawal, staking unstake vs reward claim vs emergency payout, vesting claim vs beneficiary mutation/removal, mint/burn/tax operations sharing standard `Transfer` bounce handlers, and stale pending entries hit by unrelated later bounces.
- For parser, proof, and helper candidates, test exact edge shapes internally: empty proof, single-leaf proof, two-leaf proof, trailing bits, trailing refs, optional `addr_none`, unexpected opcode, nested-cell schema mismatch, lazy field first accessed after mutation, cross-language producer/consumer field disagreement, and getter/response tuple mismatch.
- For business arithmetic candidates, test boundary values internally: zero, one unit, exact cap, cap plus one, non-divisible duration, final tranche, equality at quorum/threshold, denominator zero/one, decimal-scale mismatch, fee/tax active and inactive windows, split-leg failure, and gross-booked vs net-received recipient amounts for every protocol payout.
- For tax, fee-on-transfer, burn-split, redistribution, or net-amount candidates, enumerate every protocol payout path that uses the asset transfer function. For each path decide: amount booked by application state, amount debited from sender wallet, amount delivered to recipient wallet, tax/burn/split amounts, and whether state is updated by gross or net. Underpayment caused by a taxed payout path is distinct from failed tax redistribution.
- For each relevant step, decide internally: `BLOCKS`, `ALLOWS`, `IRRELEVANT`, or `UNCERTAIN`.
- `BLOCKS` with a concrete guard refutes the candidate. `UNCERTAIN` keeps the candidate alive for one source-backed validation pass, but unresolved uncertainty cannot by itself produce a finding; apply the partial-path deduction or drop the candidate if the FP gate cannot be satisfied.
- Do not use deployer-intent reasoning. Evaluate what the code allows, not how the deployer, admin, wallet, marketplace, or relayer is expected to behave off-chain.

Review-trail promotion and rejection:

- A review trail is any vector-agent `Surviving` or `Borderline` item, `Deep Pass` item, `Review Trails` item, or agent note that points to a specific root cause but was not emitted as a confirmed finding.
- A coverage-map item with `status=review_trail` or a nontrivial checklist item that is absent from all confirmed findings must be treated as a review trail candidate and validated or explicitly refuted.
- Promote a review trail only if source validation traces a complete caller -> reachable entrypoint -> vulnerable state/trust edge -> impact path, or if two or more independent scanners identify the same non-refuted issue and the source validation satisfies every FP gate.
- Promote source-backed semantic probes without requiring external business documentation when the code itself defines the invariant. Examples: a stored `vestingEnd`/`vestingDuration` plus full claimability before that end is a reportable time-lock break; a proposal system that accepts a root shape but cannot verify the empty proof for a single-leaf tree is a reportable governance liveness/integrity break unless creation rejects that shape; a protocol payout that records gross delivery while a taxed wallet path delivers net is a reportable accounting break.
- Lack of off-chain Merkle format documentation, tokenomics prose, or product requirements is not a blocker when the on-chain code names and enforces the relevant object, such as `vestingEnd`, `duration`, `snapshotRoot`, `totalVotingPower`, `rewardPool`, `totalRewardsPaid`, `pendingSales`, `pendingClaims`, or `pendingUnstakes`.
- A known-finding baseline row that reappears as a source-backed review trail must be promoted or explicitly source-refuted. Do not leave it as an unresolved trail merely because the previous run used different wording or severity.
- Multiple scanners do not override a concrete source refutation. If a real guard blocks the path, drop the review trail even when several scanners mentioned it.
- If multiple scanners flag the same plausible issue but source validation remains uncertain, keep it as an unresolved review trail only when it is not source-refuted.

Merge rules:

- Deduplicate by root cause, not title wording, and include language in the root-cause key when the same TON issue appears through different language-specific entrypoints.
- If two findings describe the same root cause, keep the candidate with higher confidence, clearer attacker path, and more actionable fix.
- Do not merge merely because two findings share `SendIgnoreErrors`, a pending map, a query id, a Jetton wallet, or the same receiver. Preserve them when the vulnerable state edge, failure mode, affected asset, or local fix differs.
- Preserve post-receipt validation/late-rejection fund-lock findings separately from later outbound delivery finality findings when one fix is an inbound refund/reject path and the other fix is payout confirmation or compensation.
- Preserve local action-phase send failure, remote bounce recovery failure, stale pending-entry cleanup, and query-id/correlation confusion as separate findings when each requires a different code-local fix.
- Preserve rollback based on mutable current state separately from missing pending cleanup when the fix requires storing immutable pending data or validating a different correlation field.
- Preserve tax/net-amount accounting findings separately from generic payout finality findings when the exploit depends on a transfer tax, fee window, burn split, redistribution leg, or net-received amount.
- Preserve taxed protocol payout underpayment separately from tax redistribution delivery failure. The first fix is to use net-aware or untaxed payout accounting; the second fix is to reconcile failed split legs.
- Preserve business-math and time-rounding findings separately from message-flow findings even when they affect the same entitlement or payout flow.
- Preserve helper edge cases such as empty/single-leaf Merkle proofs, optional-address encoding, integer-width mismatch, and getter tuple order separately from the higher-level governance, wallet, or discovery finding when the helper itself needs a local fix.
- Preserve cross-flow stale pending collisions separately from generic pending-map storage growth when a later unrelated operation can hit an old record and execute the old rollback path.
- Preserve concrete alias findings when they have a different exploit mechanism or different code-local fix from another finding in the same function or message flow.
- Preserve fixed-layout parser-integrity findings as distinct root causes unless the exploit mechanism and local fix are truly identical to another finding.
- Preserve TEP standard-conformance findings separately from generalized shared-vector findings when the code-local fix depends on a specific TEP opcode, getter, reply, funding rule, optional field, or serialization format. If the root cause is identical, keep the clearer TEP-specific finding.
- Preserve Tact fallback-receiver acceptance, Tolk lazy-parser failures, FunC missing full consumption, and unvalidated caller-supplied nested-message forwarding as distinct root causes when their fixes differ.
- Preserve unvalidated nested-message forwarding separately from optimistic authoritative accounting or supply desync.
- If two findings truly compound, preserve both and mention the interaction once in the stronger finding when possible.
- Check for linked exploit paths after deduplication. If finding A's output, committed state, or trusted message feeds finding B's precondition and the combined impact is strictly worse than either alone, add `Linked path: A + B` with confidence equal to `min(A, B)`. Most audits should have zero to two linked-path findings. Do not add one when the same interaction can be captured clearly in an existing finding.

Fix verification:

- For each finding with confidence `>= 80`, trace the attack again with the proposed fix applied.
- Verify the fix does not introduce a new authorization bypass, replay path, trapped-value path, storage-rent drain, unbounded gas loop, bounce/excess loss, TEP ABI break, or cross-language message-schema mismatch.
- If the same patch pattern is needed in repeated locations, list all affected locations in the finding or choose a centralized fix that covers all of them.
- Do not fabricate a diff for an unsafe or unproven fix. Prefer a precise remediation note under `Fix` when a safe exact diff cannot be derived from the bundled source.

Final report rules:

- Sort findings by confidence, highest first.
- Re-number sequentially after deduplication.
- Do not cap the number of findings. If the confirmed set is large, output all confirmed findings and keep concise descriptions rather than dropping lower-ranked confirmed issues.
- Insert the **Below Confidence Threshold** separator row using the threshold from `judging.md`, unless explicitly overridden.
- Use `report-formatting.md` for the report wrapper, scope table, findings list, and final layout.
- If the user requests a custom machine-readable output format, still complete the full orchestration, agent capture, coverage map, validation, and deduplication process first. Then adapt only the final presentation to the requested format; do not skip internal artifacts or drop review trails before merge validation.
- Include `Review Trails` in the final report only when unresolved, non-refuted trails remain after merge validation. If there are trails but no confirmed findings, use the full report wrapper with `No findings.` under `Findings`.
- Include the audited language next to each file or finding location when the report covers more than one language.
- If no confirmed findings and no reportable review trails remain after merge, respond with `No findings.` unless the user explicitly asked for a full empty report.
- If `--file-output` is set, write the report to the path defined by `report-formatting.md` and print that path. Otherwise, print the report only in the terminal.
