---
name: solana-auditor
description: Audit Solana Rust programs for exploitable security issues using Codex specialist review passes. Use when asked to audit, security-review, or check a Solana program, either repository-wide or for specific Rust files.
---

# Solana Program Security Audit

Coordinate a security review of Solana Rust programs in Codex. Preserve the user's requested scope, keep the review read-only unless report output is explicitly requested, and report only issues supported by concrete code paths.

## Invocation and scope

Accept natural-language requests or explicit `$solana-auditor` invocation.

- With no file arguments, review all in-scope `.rs` files under the current program or repository.
- With file arguments, review only those files, while reading directly related code when necessary to trace a call, account constraint, or state transition.
- `--deep` adds the protocol-specific review pass for DeFi protocols.
- `--file-output` writes the final Markdown report using the path rules in [references/report-formatting.md](references/report-formatting.md). Without this flag, return the report in the final response only.

For repository-wide discovery, use `rg --files -g '*.rs'` and exclude directories `tests/`, `test/`, `migrations/`, `scripts/`, `target/`, and `node_modules/`. Exclude files matching `*_test.rs`, `*_tests.rs`, `test_*.rs`, `tests.rs`, and exclude `mod.rs` unless it contains instruction handlers or security-relevant logic.

The skill root is the directory containing this `SKILL.md`; resolve all linked references relative to it. Do not fetch remote skill metadata or perform update checks.

## Workflow

### 1. Discover and prepare

Print the banner below, then:

1. Identify the repository root and enumerate the exact in-scope files.
2. Read [references/judging.md](references/judging.md) and [references/report-formatting.md](references/report-formatting.md).
3. Identify the framework (Anchor, native Rust, Pinocchio, or mixed) and the main instruction entrypoints.
4. Share a concise commentary update with the scope and review mode.

Do not create source bundles. Codex agents share the workspace and should read the source and references directly from their paths.

### 2. Run specialist passes

When Codex collaboration tools are available, use `spawn_agent` for the independent passes below and `wait_agent` to collect their results. Spawn only as many agents as the available concurrency permits, run remaining passes in batches, and wait for every pass before synthesis. Do not request or pin a particular model.

If collaboration agents are unavailable, perform the same passes sequentially in the main agent.

| Pass | Reference instructions |
| --- | --- |
| Vector scan | All files in `references/attack-vectors/` plus `references/hacking-agents/vector-scan-agent.md` |
| Math and precision | `references/hacking-agents/math-precision-agent.md` |
| Access control | `references/hacking-agents/access-control-agent.md` |
| Economic security | `references/hacking-agents/economic-security-agent.md` |
| Execution trace | `references/hacking-agents/execution-trace-agent.md` |
| Invariants | `references/hacking-agents/invariant-agent.md` |
| Periphery | `references/hacking-agents/periphery-agent.md` |
| First principles | `references/hacking-agents/first-principles-agent.md` |
| Protocol analysis (`--deep` only) | `references/hacking-agents/solana-protocol-agent.md` |

Each pass must also read `references/hacking-agents/shared-rules.md`. Give each agent:

- the exact in-scope file paths;
- the absolute skill-root path;
- its specialist reference path or paths;
- an instruction to read all assigned references and relevant source before returning structured `FINDING` and `LEAD` blocks only;
- an instruction not to edit files or broaden the audit scope.

Use clear task names such as `vector_scan`, `math_precision`, and `access_control`. Collect completed results before reusing an agent for another independent pass.

### 3. Deduplicate and validate

Perform one synthesis pass over all specialist results:

1. Parse every `FINDING` and `LEAD`. Group exact `group_key` matches first, then merge synonymous bug classes for the same program and handler. Retain the strongest evidence-backed item, number items sequentially, and record how many passes independently identified it.
2. Check for composite chains only when one issue's output enables another issue's precondition and the combined impact is strictly worse. Most reviews should have zero to two chains.
3. Evaluate each deduplicated item once through the four gates in [references/judging.md](references/judging.md), in order. Trace relevant paths in a fixed order: initialize, deposit/stake, process/swap, withdraw/unstake, claim, close. Mark each path `BLOCKS`, `ALLOWS`, `IRRELEVANT`, or `UNCERTAIN`; treat `UNCERTAIN` as `ALLOWS` for the final gate decision.
4. Promote or demote leads only under the rules in `judging.md`. Independent convergence does not override a concrete code refutation.
5. For findings with confidence at least 80, verify the proposed fix against the attack path and check for new denial-of-service behavior, CPI failures, or broken invariants. If a safe fix cannot be established, omit the fix and say why.

Do not print an intermediate deduplication list. Exclude rejected items from the report.

### 4. Report

Format the final result exactly as specified in [references/report-formatting.md](references/report-formatting.md), sorted by confidence. Always list every reviewed file and clearly distinguish confirmed findings from leads.

If `--file-output` is present, create the report at the specified path and link it in the final response. Otherwise, do not write or modify project files.

## Banner

Print this before starting discovery:

```text
███████╗ ██████╗ ██╗      █████╗ ███╗   ██╗ █████╗      █████╗ ██╗   ██╗██████╗ ██╗████████╗ ██████╗ ██████╗
██╔════╝██╔═══██╗██║     ██╔══██╗████╗  ██║██╔══██╗    ██╔══██╗██║   ██║██╔══██╗██║╚══██╔══╝██╔═══██╗██╔══██╗
███████╗██║   ██║██║     ███████║██╔██╗ ██║███████║    ███████║██║   ██║██║  ██║██║   ██║   ██║   ██║██████╔╝
╚════██║██║   ██║██║     ██╔══██║██║╚██╗██║██╔══██║    ██╔══██║██║   ██║██║  ██║██║   ██║   ██║   ██║██╔══██╗
███████║╚██████╔╝███████╗██║  ██║██║ ╚████║██║  ██║    ██║  ██║╚██████╔╝██████╔╝██║   ██║   ╚██████╔╝██║  ██║
╚══════╝ ╚═════╝ ╚══════╝╚═╝  ╚═╝╚═╝  ╚═══╝╚═╝  ╚═╝    ╚═╝  ╚═╝ ╚═════╝ ╚═════╝ ╚═╝   ╚═╝    ╚═════╝ ╚═╝  ╚═╝
```
