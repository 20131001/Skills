---
name: ton-auditor
description: Security audits for TON projects and smart contracts written in FunC, Tolk, or Tact. Use for repository or file-level reviews involving TVM execution, authorization, asynchronous messages, bounce handling, storage, gas, serialization, Jetton/NFT standards, or cross-contract behavior.
---

# TON Smart Contract Security Audit

Run a source-driven TON audit with explicit coverage and preserved candidate dispositions. The detailed role prompts, bundle rules, validation rules, and report requirements are in [references/audit-workflow.md](references/audit-workflow.md); read that file before auditing.

## Non-negotiable workflow

1. Discover and classify primary project sources. Exclude tests, generated artifacts, dependencies, caches, toolchain internals, and previous audit artifacts by default, while honoring explicitly named files. Record exclusions and reasons.
2. Read `references/judging.md`, `references/report-formatting.md`, and the applicable attack-vector, language, standards, and checklist references.
3. Build a source inventory covering receivers, getters, authorization, storage, state-mutating helpers, message schemas, sends, bounce/callback/excess paths, parsers/codecs, arithmetic, storage/gas, upgrades, and available evidence.
4. Generate source-derived semantic probes for applicable normal, failure, delayed, replayed, reordered, malformed, boundary, partial-completion, cleanup, and schema-compatibility paths.
5. Run the planned vector, coverage, adversarial, and assurance roles in parallel when available, or sequentially with the same role boundaries and completion checks.
6. Capture every role result, validate required sections and coverage counts, retry missing or underspecified output once, and mark the audit incomplete if required coverage remains missing.
7. Validate candidates against source, preserving `confirmed`, `review_trail`, `refuted`, `merged_into`, and `out_of_scope` dispositions. `UNCERTAIN` requires a focused validation pass and is never automatically confirmed or silently dropped.
8. Deduplicate only when reachability, failure mechanism, violated invariant, terminal path, impact, and local remediation are materially the same.
9. Freeze the internal ledger before adapting the final output to JSON, Markdown, or another requested format.

## Evaluation isolation

Historical findings, benchmark outputs, labeled vulnerability sets, and product-specific findings databases are sealed evaluation artifacts, not audit instructions. Do not discover or read them automatically, include them in audit bundles, or derive rules, probes, severities, titles, locations, or remediations from them. If comparison is explicitly requested, freeze the independent audit first and run comparison as a separate post-audit evaluation; never feed evaluator content back into the same audit.

## Invocation

- Default: audit all project-owned `.tolk`, `.fc`, `.func`, and `.tact` files.
- `deep`: add adversarial and protocol-composition passes.
- `$filename ...`: audit only the named files; explicit targets override default exclusions.
- `--file-output`: write the final report using `references/report-formatting.md`.
- `--model <id>` / `--model=<id>`: pass the exact model override to every role and retry.
- `--reasoning-effort <value>` / `--reasoning-effort=<value>`: pass the exact override to every role and retry; when omitted, default to `medium` for vector, adversarial, coverage, and retry roles.

If an explicit model or reasoning override cannot be honored, stop and report the limitation rather than silently substituting another value. If subagents are unavailable, sequential execution is valid only when it produces one result per planned role with the same required sections, coverage records, candidate ledger, and completion gate.

## Required assurance summary

The final run artifacts and report must state:

```text
execution_mode
primary_source_files
excluded_path_classes
inventory_total/completed
semantic_probes_total/completed
roles_planned/completed/failed
candidates_confirmed/review_trail/refuted/merged/out_of_scope
audit_status: complete | incomplete
```

An incomplete run must not be presented as a comprehensive audit.
