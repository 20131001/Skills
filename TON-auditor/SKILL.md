---
name: TON-auditor
description: Audit FunC TON contracts and return a deduplicated report of confirmed security findings.
when_to_use: Use when the user wants a TON smart contract security audit, exploit-path review, or standards check for FunC contracts. Examples: "audit this jetton minter", "review these TON contracts", "find security bugs in this wallet", "check TEP-74 compliance", "deep review this bridge". Do not use for non-TON code, ordinary feature work, or pure refactors with no security-review intent.
allowed-tools:
  - Read
  - Task
  - Bash(find:*)
  - Bash(rg:*)
  - Bash(sort:*)
  - Bash(sed:*)
  - Bash(wc:*)
arguments:
  - scope_and_flags
argument-hint: "[deep] [$file1.fc $file2.fc ...] [--file-output]"
context: inline
agent: general-purpose
---

# Smart Contract Security Audit

Orchestrate a parallel TON audit for FunC contracts, then merge only confirmed findings into one final report.

## Inputs

- `$scope_and_flags`: optional mode selector and scope override.
  - `deep`: run the normal vector scan plus the adversarial reasoning agent.
  - `$file1.fc $file2.fc ...`: audit only the named FunC files.
  - `--file-output`: also write the final report using the path rule in `report-formatting.md`.

## Goal

Produce one deduplicated TON audit result with explicit exploit-path evidence and confidence scoring. The end state is either:

- a final report containing only confirmed findings after merge and deduplication, or
- `No findings.` when nothing survives the FP gate.

## Rules

- Use Bash `find`, not Glob, for contract and reference discovery.
- Sort discovered paths deterministically before bundling, spawning, or reporting.
- Treat `judging.md` as the source of truth for the FP gate and confidence threshold.
- Treat `standards-and-lessons.md` as the source of truth for TON standards, interface semantics, and FunC-course-derived expectations.
- Treat `security-best-practices.md` as the source of truth for generic TON-specific security guidance.
- Match project-specific code to generalized attack vectors; preserve old concrete patterns as aliases of the nearest generalized vector.
- Reconstruct the reachable execution map for every audit: entrypoint -> helpers -> outbound messages -> in-scope receivers -> bounce paths -> terminal state writes.
- Perform an explicit parser-integrity sweep on every audit: enumerate reachable fixed-layout slice parsers, including `begin_parse()` sites, reused payload slices, and top-level storage loaders such as `load_data()`.
- Do not inline attack-vector file contents into agent prompts. Agents must read bundle files instead.
- Do not invent findings during merge.
- Never write a report file unless the user explicitly passed `--file-output`.
- Preserve V25 parser-integrity findings as distinct root causes when confirmed, including top-level storage-loader variants such as `load_data()`.
- When reporting V25, keep the existing root-cause title stable whenever possible; surface missing `end_parse()` or missing full-consumption proof in the first sentence of `Description`, and in tag-based formats add an `end-parse` / parser-integrity style tag instead of forcing a title rewrite.
- Do not merge raw nested-message forwarding into broader supply-desync findings, or vice versa, when the exploit step or local fix differs.
- If the environment cannot spawn subagents, cannot create `/tmp` bundles, or is otherwise forced into inline execution, run the same audit manually instead of weakening the workflow.
- In manual fallback mode, first emulate the vector-scan workflow exhaustively across both attack-vector files, then run one adversarial pass for additional findings; do not replace exhaustive coverage with a “strongest findings only” pass.
- In manual fallback mode, perform the same execution-map reconstruction, parser-integrity sweep, vector matching, FP gate, and merge discipline that the scanner agents would have applied.
- In manual fallback mode, each confirmed V25 parser site must either become a standalone finding or an explicit dropped candidate with a concrete reason such as intentional extension design or equivalent emptiness proof.
- If an outer harness requires a different final format such as JSON, preserve the same confirmed finding set and only translate the presentation format.

## Steps

### 1. Discover Scope And Reference Tree

Discover the in-scope FunC contracts and the attack-vector reference files.

**Artifacts**: sorted in-scope contract list, sorted attack-vector files, resolved reference directory

**Rules**:
- Default scope: all `.fc` files, excluding `interfaces/`, `node_modules/`, `wrappers/`, `test*/`, `*.spec.ts`, and `stdlib.fc`.
- `deep` does not change scope; it only adds the adversarial reasoning agent.
- If specific `.fc` files are provided, audit only those files.
- Verify that all discovered `attack-vectors-*.md` files belong to one resolved `references/` tree. If multiple trees exist, use the one nearest to the repo root unless the user explicitly chose another.
- If no in-scope `.fc` files are found, stop and report that no auditable FunC files matched scope.
- If no `attack-vectors-*.md` files are found, stop and report that the skill is misconfigured.

**Success criteria**: There is one deterministic contract scope, one deterministic vector-file list, and one resolved reference directory.

### 2. Load References And Build Bundles

Load the shared reference files and create one bundle per attack-vector file.

**Artifacts**: loaded reference files, bundle paths, bundle-to-vector mapping, bundle line counts

**Rules**:
- Read these reference files:
  - `{resolved_path}/agents/vector-scan-agent.md`
  - `{resolved_path}/report-formatting.md`
  - `{resolved_path}/judging.md`
  - `{resolved_path}/standards-and-lessons.md`
  - `{resolved_path}/security-best-practices.md`
  - in `deep` mode only: `{resolved_path}/agents/adversarial-reasoning-agent.md`
- Build `/tmp/audit-agent-1-bundle.md`, `/tmp/audit-agent-2-bundle.md`, and so on in sorted vector order.
- Each bundle must concatenate content in this exact order:
  1. every in-scope `.fc` file, each preceded by a `### /absolute/path.fc` header and wrapped in a fenced `func` block
  2. `judging.md`
  3. `report-formatting.md`
  4. `standards-and-lessons.md`
  5. `security-best-practices.md`
  6. exactly one assigned attack-vector file
- Create all bundles in a single shell command and print exact bundle-to-vector mappings plus line counts.
- Do not inline any attack-vector contents into the parent prompt.

**Success criteria**: Every vector file has exactly one bundle, every bundle has a known line count, and every bundle contains the required references in the required order.

### 3. Launch Scanners Or Run Manual Fallback

Spawn one vector agent per bundle and, in `deep` mode, one adversarial reasoning agent. If that execution model is unavailable, run the equivalent audit inline.

**Artifacts**: vector agent ids and optional adversarial agent id, or one manual triage set plus a manual parser-site list

**Rules**:
- Spawn one vector agent per discovered bundle.
- Vector agents use `model: "gpt-5.3-codex"` with `reasoning_effort: "medium"`.
- Each vector-agent prompt must contain the full text of `vector-scan-agent.md`, then append:
  - `Your bundle file is /tmp/audit-agent-N-bundle.md (XXXX lines).`
  - `Build the execution map first and use it for every vector: entrypoint -> helpers -> outbound messages -> in-scope receivers -> bounce paths -> terminal state writes.`
  - `Use standards-and-lessons.md and security-best-practices.md inside the bundle whenever the code depends on TON standards, message semantics, gas behavior, upgrade logic, or external-message safety.`
  - `Your final response must contain exactly these sections in order: Triage, Deep Pass, Findings.`
  - `Under Findings, output only confirmed finding blocks formatted per report-formatting.md, or No findings.`
- In `deep` mode, spawn exactly one adversarial reasoning agent with `model: "gpt-5.4"` and `reasoning_effort: "high"`.
- The adversarial prompt must:
  - provide the in-scope `.fc` file paths
  - provide `Your reference directory is {resolved_path}.`
  - instruct the agent to read `{resolved_path}/agents/adversarial-reasoning-agent.md`
  - remind it to read `judging.md`, `report-formatting.md`, `standards-and-lessons.md`, and `security-best-practices.md`
  - remind it to build the execution map before confirming logic findings
  - remind it: `Return only formatted finding blocks, or No findings. Do not output a full report wrapper.`
- If subagents or `/tmp` bundle creation are unavailable, switch to manual fallback:
  - read `judging.md`, `report-formatting.md`, `standards-and-lessons.md`, `security-best-practices.md`, and both `attack-vectors-*.md` directly
  - build the same execution map inline
  - run the explicit parser-integrity sweep before any other deep classification
  - emulate `vector-scan-agent.md` coverage across both attack-vector files before doing any adversarial free-form pass
  - keep a manual triage list for both vector files so coverage does not collapse to only the strongest few findings
  - write down every reachable fixed-layout parser site, especially nested `begin_parse()` cells, reused payload slices, and top-level storage loaders such as `load_data()`
  - for every confirmed V25 parser site, carry one standalone candidate finding into merge unless the exact same parser-integrity root cause on the same site is already represented
  - after exhaustive vector coverage is complete, run one adversarial pass only to add new confirmed findings or strengthen attacker paths; do not let that pass delete already confirmed vector findings
  - if an outer harness requires JSON or another format, keep the confirmed finding set intact and change only formatting, not selection

**Success criteria**: Either every required agent is running with the correct scope, prompt, and model settings, or the inline fallback has the same execution map, parser sweep, and candidate-finding artifacts ready for merge.

### 4. Merge And Validate Findings

Wait for all agents, extract confirmed findings, and merge them by root cause.

**Artifacts**: candidate finding set, deduplicated finding set, final report draft

**Rules**:
- For vector agents, use only the `Findings` section as candidate findings.
- Do not forward `Triage` or `Deep Pass` to the user; use them only for coverage checks, duplicate resolution, and conflict handling.
- For the adversarial agent, treat the entire response as either formatted finding blocks or `No findings.`
- In manual fallback mode, use the inline triage / deep-pass notes the same way you would have used the child-agent `Triage` and `Deep Pass` sections.
- In manual fallback mode, merge the exhaustive vector-coverage findings first, then layer in additional adversarial findings; do not keep only the strongest subset.
- Treat below-threshold confirmed findings as real findings during merge; threshold only changes presentation, not existence.
- Deduplicate by root cause, not by title wording.
- If two findings describe the same root cause, keep the version with:
  1. higher confidence
  2. clearer attacker path
  3. more actionable fix
- Do not merge away a concrete alias finding when it has a different exploit mechanism or different local fix from another finding in the same function or message flow.
- Preserve separate findings when they are meaningfully distinct, including:
  - V25 parser-integrity findings
  - V31 raw nested-message forwarding
  - V32 authoritative supply/accounting desync
- If an agent confirms V25, do not discard it solely because the impact is framed as malformed-shape acceptance or broken parser integrity rather than immediate fund loss.
- Apply the same preservation rule to top-level storage-loader V25 cases such as `load_data()` when multiple reachable handlers rely on the exact storage layout.
- If an agent confirms V25 on a specific parser site, keep it as a standalone finding unless another confirmed finding is the exact same parser-integrity root cause on the same site with no meaningful wording or fix difference.
- Preserve the existing V25 title wording during merge whenever possible; if `end_parse()` is not already visible, add it in the first sentence of `Description` rather than rewriting the whole title.
- If a vector agent's `Deep Pass` explicitly marks V25 as `CONFIRM` with a concrete parser site and reachable path, but the `Findings` section omits the corresponding V25 block, treat that as agent-formatting drift and reconstruct one minimal V25 finding from the confirmed path instead of silently dropping it.
- Do not invent new findings during merge.
- Do not re-draft confirmed findings unless needed to collapse obvious duplicates.

**Success criteria**: The merged finding set is deduplicated, root-cause-correct, and still preserves materially distinct parser, forwarding, and accounting bugs.

### 5. Report Outcome

Produce the final user-facing result.

**Human checkpoint**: Only write a report file if `--file-output` was explicitly passed.

**Artifacts**: final report or `No findings.`

**Rules**:
- Sort findings by confidence, highest first.
- Re-number sequentially after deduplication.
- Insert the **Below Confidence Threshold** separator row using the threshold from `judging.md`, unless explicitly overridden.
- Use `report-formatting.md` for the report wrapper, scope table, and final layout.
- Include confirmed below-threshold findings in the final report; they should appear below the separator and omit `Fix`.
- If no confirmed findings remain after merge, respond with `No findings.` unless the user explicitly asked for a full empty report.
- If `--file-output` is set, write the report to the path defined by `report-formatting.md` and print that path. Otherwise, print the report only in the terminal.

**Success criteria**: The final output is correctly formatted, confidence-sorted, scoped accurately, and consistent with `judging.md`.

## Output

The final response must include:

- what scope was audited
- the confirmed findings after merge, or `No findings.`
- confidence-ordered results using the report-formatting rules
- any uncertainty or below-threshold separation required by `judging.md`

## Final Rules

- Do not skip validation or merge checks.
- Do not claim a finding survived unless it passes the FP gate.
- Do not write a report file without explicit user permission via `--file-output`.
- Prefer the smallest sufficient tool usage and deterministic discovery.
