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
5. Bash `mktemp -d /tmp/ton-audit-XXXXXX` and store it as `{bundle_dir}`.

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

**Turn 2 - Prepare.** In one message, make these parallel tool calls:

1. Read `{resolved_path}/report-formatting.md`.
2. Read `{resolved_path}/judging.md`.

Then build all bundles in a single Bash command and print exact bundle-to-language/vector mappings plus line counts.

Before writing bundles, compute `{optional_reference_files}` as existing files from:

- `{resolved_path}/*.md`, excluding `judging.md` and `report-formatting.md`
- `{resolved_path}/attack-vectors/README.md` if present

Sort `{optional_reference_files}` by path. Missing optional files should not fail the audit.

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

The bundle may include all in-scope source files so agents can reason about cross-language TON message flows. Each language agent must apply language-specific rules only to its target-language files and use other files as context. Do not bundle another language's `attack-vectors/languages/*.md` file for a target language.

**Turn 3 - Spawn.** In one message, spawn all agents as parallel foreground Agent tool calls. Do not use background execution.

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
The bundle contains source, local references, shared rules, your vector-scan instructions, and exactly one selected attack-vector file. Do not browse external links unless explicitly asked.
Audit target-language files as primary scope; use other languages only as TON message-flow context.
Read the bundle fully before producing findings.
Your final response must contain exactly these sections in order: Triage, Deep Pass, Findings, Review Trails.
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
The bundle contains source, local references, shared rules, your adversarial instructions, and all selected vectors for the target language. Do not browse external links unless explicitly asked.
Audit target-language files as primary scope; use other languages only as TON message-flow context.
Read the bundle fully before producing findings.
Return exactly two sections: Findings, then Review Trails.
Under Findings, return only formatted finding blocks, or No findings.
Under Review Trails, return high-signal unresolved trails, or `None.`.
Do not output a full report wrapper.
```

**Turn 4 - Deduplicate, validate & output.** Wait for all agents, then merge results into the final report.

Parsing rules:

- For vector agents, parse `Triage`, `Deep Pass`, `Findings`, and `Review Trails` separately. Use only the content under `Findings` as confirmed candidate findings.
- Treat vector-agent `Triage`, `Deep Pass`, and `Review Trails` entries as internal review-trail or conflict evidence unless promoted by the merge gate.
- For adversarial agents, parse `Findings` and `Review Trails`.
- Do not create standalone findings that are not supported by at least one agent candidate or review trail. Linked-path findings are allowed only when built from two already validated findings.
- Do not re-draft confirmed findings unless needed to collapse obvious duplicates cleanly.

Candidate normalization:

- For every candidate, extract or infer these fields for merge only: title, language, file, contract/function/receiver/opcode/getter, attack-vector ID(s), case_family, attacker path, impact, confidence, fix, evidence_trace, and source scanner.
- Build an internal `ton_case_key` as `language | file | contract/function/receiver/opcode/getter | case_family`.
- Deduplicate exact `ton_case_key` matches first. Then merge synonymous case families only when the language, file, reachable entrypoint, exploit mechanism, and local fix are materially the same.
- Track source-scanner count and vector IDs internally for conflict resolution. Do not add scanner-count metadata to the final report unless `report-formatting.md` explicitly supports it.
- If a candidate lacks a concrete attacker path or affected entrypoint, treat it as a review trail rather than a reportable finding.

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
- For each relevant step, decide internally: `BLOCKS`, `ALLOWS`, `IRRELEVANT`, or `UNCERTAIN`.
- `BLOCKS` with a concrete guard refutes the candidate. `UNCERTAIN` keeps the candidate alive for one source-backed validation pass, but unresolved uncertainty cannot by itself produce a finding; apply the partial-path deduction or drop the candidate if the FP gate cannot be satisfied.
- Do not use deployer-intent reasoning. Evaluate what the code allows, not how the deployer, admin, wallet, marketplace, or relayer is expected to behave off-chain.

Review-trail promotion and rejection:

- A review trail is any vector-agent `Surviving` or `Borderline` item, `Deep Pass` item, `Review Trails` item, or agent note that points to a specific root cause but was not emitted as a confirmed finding.
- Promote a review trail only if source validation traces a complete caller -> reachable entrypoint -> vulnerable state/trust edge -> impact path, or if two or more independent scanners identify the same non-refuted issue and the source validation satisfies every FP gate.
- Multiple scanners do not override a concrete source refutation. If a real guard blocks the path, drop the review trail even when several scanners mentioned it.
- If multiple scanners flag the same plausible issue but source validation remains uncertain, keep it as an unresolved review trail only when it is not source-refuted.

Merge rules:

- Deduplicate by root cause, not title wording, and include language in the root-cause key when the same TON issue appears through different language-specific entrypoints.
- If two findings describe the same root cause, keep the candidate with higher confidence, clearer attacker path, and more actionable fix.
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
- Insert the **Below Confidence Threshold** separator row using the threshold from `judging.md`, unless explicitly overridden.
- Use `report-formatting.md` for the report wrapper, scope table, findings list, and final layout.
- Include `Review Trails` in the final report only when unresolved, non-refuted trails remain after merge validation. If there are trails but no confirmed findings, use the full report wrapper with `No findings.` under `Findings`.
- Include the audited language next to each file or finding location when the report covers more than one language.
- If no confirmed findings and no reportable review trails remain after merge, respond with `No findings.` unless the user explicitly asked for a full empty report.
- If `--file-output` is set, write the report to the path defined by `report-formatting.md` and print that path. Otherwise, print the report only in the terminal.
