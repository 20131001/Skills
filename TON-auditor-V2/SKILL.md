---
name: ton-auditor
description: Security audit of TON/Tolk/FunC/Tact smart contracts while you develop. Trigger on "audit", "check this contract", or "review for security". Supports full-repository and file-scoped reviews, deep TON protocol analysis, report file output, and an optional model override for audit agents.
---

# TON Smart Contract Security Audit

Orchestrate a parallelized TON smart contract security audit using subagents when available. If subagents are unavailable, run the same specialist passes sequentially in the current conversation.

## Mode Selection

**Exclude pattern:** skip directories `tests/`, `test/`, `build/`, `node_modules/`, `wrappers/`, `scripts/` and files matching `*_test.fc`, `*_test.func`, `*_test.tact`, `*_test.tolk`, `test_*.fc`, `test_*.func`, `test_*.tact`, `test_*.tolk`, `*.spec.ts`.

- **Default** (no arguments): scan all `.tolk`, `.fc`, `.func`, and `.tact` files using the exclude pattern. Use Bash `find` (not Glob).
- **`$filename ...`**: scan the specified file(s) only.

**Flags:**

- `--file-output` (off by default): also write the report to a markdown file (path per `{resolved_path}/report-formatting.md`). Never write a report file unless explicitly passed.
- `--deep`: also run the TON protocol analysis pass (Agent 9). Use for thorough reviews. Slower and more costly.
- `--model <model>` (optional): run every spawned audit agent with the requested model. Without this flag, inherit the current ChatGPT/Codex model; do not hard-code a provider-specific model. Accept `--model=<model>` as equivalent. If the runtime cannot select models, continue with its current model and state that the override was unavailable.

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

```
Your bundle file is {bundle_dir}/agent-N-bundle.md (XXXX lines).
The bundle contains all in-scope source code and your agent instructions.
Read the bundle fully before producing findings.
```

If `--deep` is set, also run **Agent 9** (TON protocol analysis), using the same default or `--model` override as the other agents. Agent 9 receives the in-scope file paths and the instruction: your reference directory is `{resolved_path}`. Read `{resolved_path}/hacking-agents/ton-protocol-agent.md` for your full instructions.

**Turn 4 — Deduplicate, validate & output.** Single-pass: deduplicate all agent results, gate-evaluate, and produce the final report in one turn. Do NOT print an intermediate dedup list — go straight to the report.

1. **Deduplicate.** Parse every FINDING and LEAD from all agents. Group by `group_key` field (format: `Contract | handler | bug-class`). Exact-match first; then merge synonymous bug_class tags sharing the same contract and handler. Keep the best version per group, number sequentially, annotate `[agents: N]`.

   Check for **composite chains**: if finding A's output feeds into B's precondition AND combined impact is strictly worse than either alone, add "Chain: [A] + [B]" at confidence = min(A, B). Most audits have 0–2.

2. **Gate evaluation.** Run each deduplicated finding through the four gates in `judging.md` (do not skip or reorder). Evaluate each finding exactly once — do not revisit after verdict.

   **Single-pass protocol:** evaluate every relevant code path ONCE in fixed order (initialization/deployment → admin/upgrade handlers → `recv_internal`/`onInternalMessage` branches → `recv_external`/`onExternalMessage` → bounce/fallback/empty handlers → getters). One-line verdict per path: `BLOCKS`, `ALLOWS`, `IRRELEVANT`, or `UNCERTAIN`. Commit after all paths — do not re-examine. `UNCERTAIN` = `ALLOWS`.

3. **Lead promotion & rejection guardrails.**
   - Promote LEAD → FINDING (confidence 75) if: complete exploit chain traced in source, OR `[agents: 2+]` demoted (not rejected) the same issue.
   - `[agents: 2+]` does NOT override a concrete refutation — demote to LEAD if refutation is uncertain.
   - No deployer-intent reasoning — evaluate what the code _allows_, not how the deployer _might_ use it.

4. **Fix verification** (confidence ≥ 80 only): trace the attack with fix applied; verify no new DoS, state inconsistency, or broken invariants; list all locations if the pattern repeats. If no safe fix exists, omit it with a note.

5. **Format and print** per `report-formatting.md`. Exclude rejected items. If `--file-output`: also write to file.
