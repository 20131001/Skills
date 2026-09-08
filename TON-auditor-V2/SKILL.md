---
name: ton-auditor-v2
description: High-assurance TON security audit for Tolk, FunC, or Tact repositories when reproducibility, explicit coverage, multi-pass validation, and machine-verifiable audit artifacts matter.
---

# TON Smart Contract Security Audit

Orchestrate an evidence-preserving audit pipeline with independent roles, source-derived coverage, strict result capture, candidate reconciliation, and a completion gate.

## Evaluation isolation

Historical findings, benchmark outputs, labeled vulnerability sets, and product-specific findings databases are sealed evaluation artifacts. Do not discover or read them automatically, include them in bundles, or derive rules, probes, severities, titles, locations, or remediations from them. If comparison is explicitly requested, freeze and hash the independent audit first, then run a separate evaluator. Never feed evaluator content back into the same audit.

## Mode Selection

**Exclude pattern:** classify tests, generated artifacts, dependencies, caches, toolchain internals, and previous audit artifacts out of primary scope. Common indicators include `tests/`, `test/`, `build/`, `dist/`, `node_modules/`, `wrappers/`, `generated/`, `vendor/`, `.acton/`, `.aiacc/`, and `.ton-audit-tmp/`.

- **Default** (no arguments): scan all `.tolk`, `.fc`, `.func`, and `.tact` files using the exclude pattern. Use Bash `find` (not Glob).
- **`$filename ...`**: scan the specified file(s) only.

Explicitly named files override default exclusions. Record excluded paths and reasons.

**Flags:**

- `--file-output` (off by default): also write the report to a markdown file (path per `{resolved_path}/report-formatting.md`). Never write a report file unless explicitly passed.
- `--deep`: also run the TON protocol analysis pass (Agent 9). Use for thorough reviews. Slower and more costly.
- `--model <model>` (optional): run every spawned audit agent with the requested model. Without this flag, inherit the current ChatGPT/Codex model; do not hard-code a provider-specific model. Accept `--model=<model>` as equivalent. If the runtime cannot select models, continue with its current model and state that the override was unavailable.
- `--reasoning-effort <value>` (optional): run every spawned audit, protocol, and retry agent with the exact requested reasoning effort. If omitted, default to `medium` for every role. Accept `--reasoning-effort=<value>` as equivalent; if repeated, the last value wins. If an explicit value cannot be honored, stop and report the limitation rather than silently substituting another value.

## Orchestration

**Turn 1 — Discover.** Print the banner, then perform these independent operations in parallel where the runtime supports it:

a. Use a shell `find` command for in-scope `.tolk`, `.fc`, `.func`, and `.tact` files per mode selection.
b. Locate `references/attack-vectors/attack-vectors-1.md` relative to this skill and treat its `references/` parent as `{resolved_path}`.
c. Create a temporary directory with `mktemp -d /tmp/audit-XXXXXX` and store it as `{bundle_dir}`.

**Turn 2 — Prepare.** In one message, make parallel tool calls: (a) Read `{resolved_path}/report-formatting.md`, (b) Read `{resolved_path}/judging.md`.

Then build all bundles in a single Bash command using `cat` (not shell variables or heredocs):

1. `{bundle_dir}/source.md` — ALL in-scope `.tolk`, `.fc`, `.func`, and `.tact` files, each with a `### path` header and fenced code block.
2. Agent bundles = `source.md` + agent-specific files:

| Bundle               | Appended files (relative to `{resolved_path}`)                                                                                                                |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `agent-1-bundle.md`  | `attack-vectors/attack-vectors-1.md` + `attack-vectors/attack-vectors-2.md` + `attack-vectors/attack-vectors-3.md` + `attack-vectors/attack-vectors-4.md` + `attack-vectors/attack-vectors-5.md` + `attack-vectors/attack-vectors-6.md` + `attack-vectors/attack-vectors-7.md` + `hacking-agents/vector-scan-agent.md` + `hacking-agents/audit-checklist-agent.md` + `hacking-agents/shared-rules.md` |
| `agent-2-bundle.md`  | `hacking-agents/math-precision-agent.md` + `hacking-agents/shared-rules.md`                                                                                   |
| `agent-3-bundle.md`  | `hacking-agents/access-control-agent.md` + `hacking-agents/shared-rules.md`                                                                                   |
| `agent-4-bundle.md`  | `hacking-agents/economic-security-agent.md` + `hacking-agents/shared-rules.md`                                                                                |
| `agent-5-bundle.md`  | `hacking-agents/execution-trace-agent.md` + `hacking-agents/shared-rules.md`                                                                                  |
| `agent-6-bundle.md`  | `hacking-agents/invariant-agent.md` + `hacking-agents/shared-rules.md`                                                                                        |
| `agent-7-bundle.md`  | `hacking-agents/periphery-agent.md` + `hacking-agents/tact-security-agent.md` + `hacking-agents/shared-rules.md`                                               |
| `agent-8-bundle.md`  | `hacking-agents/first-principles-agent.md` + `hacking-agents/tolk-security-agent.md` + `hacking-agents/shared-rules.md`                                      |

Print line counts for every bundle and `source.md`. Do NOT inline file content into agent prompts.

**Turn 3 — Spawn.** Spawn Agents 1–8 with the runtime's subagent/delegation tool, maximizing safe parallelism within its concurrency limit. By default, let every agent inherit the current ChatGPT/Codex model. When `--model` is supplied, pass that model to every spawned agent. If no subagent tool is available, execute Agents 1–8 as sequential specialist passes. Prompt template (substitute real values):

Resolve the reasoning configuration once before spawning. Pass the explicit `--reasoning-effort` value to every role and retry; otherwise pass `medium` to every role. Record the resolved value in the run manifest.

```
Your bundle file is {bundle_dir}/agent-N-bundle.md (XXXX lines).
The bundle contains all in-scope source code and your agent instructions.
Read the bundle fully before producing findings.
```

If `--deep` is set, also run **Agent 9** (TON protocol analysis), using the same default or `--model` override as the other agents. Agent 9 receives the in-scope file paths and the instruction: your reference directory is `{resolved_path}`. Read `{resolved_path}/hacking-agents/ton-protocol-agent.md` for your full instructions.

**Turn 4 — Capture, validate, reconcile & output.** Do not perform a single-pass merge. Preserve every role result, coverage answer, probe answer, and candidate disposition before final presentation.

1. **Deduplicate.** Parse every FINDING and LEAD from all agents. Group by `group_key` field (format: `Contract | handler | bug-class`). Exact-match first; then merge synonymous bug_class tags sharing the same contract and handler. Keep the best version per group, number sequentially, annotate `[agents: N]`.

   Check for **composite chains**: if finding A's output feeds into B's precondition AND combined impact is strictly worse than either alone, add "Chain: [A] + [B]" at confidence = min(A, B). Most audits have 0–2.

2. **Gate evaluation.** Run each deduplicated finding through the four gates in `judging.md` (do not skip or reorder). Evaluate each finding exactly once — do not revisit after verdict.

   Validate every relevant code path in fixed order (initialization/deployment → admin/upgrade handlers → internal branches → external branches → bounce/fallback/empty handlers → getters). Use `BLOCKS`, `ALLOWS`, `IRRELEVANT`, or `UNCERTAIN`; `UNCERTAIN` triggers one focused source-backed validation pass and never becomes a confirmed finding automatically.

3. **Lead promotion & rejection guardrails.**
   - Promote a candidate only after a complete source trace from reachable input through the failing trust/state transition to material impact.
   - Scanner agreement strengthens evidence but never overrides a concrete source refutation.
   - No deployer-intent reasoning — evaluate what the code _allows_, not how the deployer _might_ use it.

4. **Fix verification** (confidence ≥ 80 only): trace the attack with fix applied; verify no new DoS, state inconsistency, or broken invariants; list all locations if the pattern repeats. If no safe fix exists, omit it with a note.

5. **Format and print** per `report-formatting.md`. Exclude rejected items. If `--file-output`: also write to file.

## Required V2 artifacts

Every run must create a writable run directory containing `run-metadata.json`, `scope-manifest.json`, `source.md`, `inventory.json`, `probe-manifest.json`, `agent-manifest.md`, `results/`, `candidate-ledger.json`, `merge-log.json`, and `assurance-summary.json`. Record execution mode, source/reference hashes, planned and captured roles, bundle/result hashes, expected sections, retry status, and coverage counts. If these artifacts cannot be persisted, mark the audit incomplete rather than claiming a full V2 run.

## Source-driven inventory and probes

Generate inventory and probes from the current source only. Inventory receivers, getters, storage paths, mutable helpers, persistent collections, message edges, value-bearing sends, bounce/callback/excess paths, parsers/codecs, arithmetic boundaries, storage/gas risks, upgrade boundaries, and available build/test/interface evidence. Assign stable run-local IDs.

For each applicable construct, generate generic probes for local dispatch failure, remote failure, bounce, absent or delayed response, duplicate/replayed response, reordered response, mutable state between request and resolution, malformed/truncated/extended encodings, boundary arithmetic, partial completion, cleanup, and schema producer/consumer agreement. Do not create probes for absent constructs merely to satisfy a quota.

Each inventory item and probe must have exactly one source-backed status: `audited`, `finding`, `review_trail`, `refuted`, or `not_applicable`. A nontrivial item cannot be marked complete with only `ok` or a line number.

## Role equivalence and completion gate

If subagents are unavailable, execute every planned role sequentially. Sequential mode is valid only when it produces the same role boundaries, result files, schemas, inventory answers, probe answers, and completion checks. Do not collapse the plan into an informal single-agent review.

Before merge, every planned role must be captured; every assigned inventory ID and probe ID must have exactly one result; every candidate must satisfy the result schema; and summary counts must match parsed records. Retry missing or underspecified records once. If required coverage remains missing, set `audit_status=incomplete` and stop before a comprehensive report.

## Candidate reconciliation

Maintain a ledger entry for every candidate and review trail with language, locations, reachable entrypoint, root-cause family, preconditions, failing transition or invariant, impact path, evidence trace, confidence, source roles, and proposed fix. Allowed terminal dispositions are `reported`, `merged_into:<id>`, `source_refuted`, `review_trail`, and `out_of_scope`. No candidate may disappear because it has one source, low initial confidence, overlaps a broad theme, or does not fit the requested output format.

Deduplicate only when reachability, failure mechanism, violated invariant, terminal path, impact, and local remediation are materially the same. Record every merge decision and reason. Do not merge distinct parser, accounting, lifecycle, or interface causes merely because they share a contract, helper, message, or broad category.

## Post-audit evaluation isolation

Only after the independent audit is frozen may an explicitly requested evaluator compare it with a sealed reference set. Classify misses by stage (`not_discovered`, `lost_during_validation`, `lost_during_deduplication`, or `lost_during_presentation`) without exposing reference content to audit roles. Any subsequent skill improvement must address a general process weakness and be tested on varied or synthetic cases.
