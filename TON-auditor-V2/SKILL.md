---
name: ton-auditor-v2
description: High-recall, evidence-preserving TON security audit for Tolk, FunC, and Tact repositories. Use for repository or file-level audits where asynchronous messages, bounce/retry behavior, pending-state lifecycles, accounting invariants, correlation identifiers, reproducible coverage, and machine-verifiable artifacts matter. Prefer this skill for deep audits without sacrificing source-backed precision.
---

# TON Smart Contract Security Audit — High-Recall Edition

Run an evidence-preserving audit that separates broad discovery from strict validation. The objective is not a finding quota: maximize distinct, source-backed root-cause recall while retaining every candidate and every rejection decision.

## Non-negotiable properties

1. **Durable evidence first.** Create a repository-local run directory before building bundles or launching roles. A temporary directory may cache data, but it must never be the only copy.
2. **Discovery does not reject.** Discovery roles record concrete suspicions and leads; only the centralized validation phase may refute them.
3. **Validate before deduplicating.** Preserve raw candidates until reachability, failing transition, impact, and remediation boundaries are understood.
4. **Source-derived coverage.** Enumerate handlers, state, sends, pending records, correlations, and protocol edges from source rather than relying on a generic checklist alone.
5. **Independent reasoning.** Keep role inputs and outputs separate. Do not feed one discovery role's conclusions to another discovery role.
6. **No silent disappearance.** Every raw candidate receives a stable ID and terminal disposition.

## Historical evidence isolation

Historical findings, prior audit reports, `.aiacc/`, and previous `.ton-audit-tmp/` runs are sealed during independent discovery and validation. Do not derive candidates, probes, locations, severity, or recommendations from them. After the current run's validation and merge decisions are frozen, compare historical records only as a completeness check; a historical claim must still pass the current source-backed validation gates before it can enter the current findings.

## Mode selection

Exclude tests, generated files, dependencies, caches, toolchain internals, and previous audit artifacts from primary source scope unless explicitly named. Common exclusions include `tests/`, `test/`, `build/`, `dist/`, `node_modules/`, `wrappers/`, `generated/`, `vendor/`, `.acton/`, `.aiacc/`, and `.ton-audit-tmp/`.

- **Default:** scan all `.tolk`, `.fc`, `.func`, and `.tact` files with `find`.
- **Named files:** scan only explicitly supplied files; explicit paths override default exclusions.
- **`--deep`:** add contract-local discovery, cross-contract protocol discovery, TON protocol review, and negative-space review.
- **`--file-output`:** also write the formatted report to the path defined by `references/report-formatting.md`.
- **`--model <model>`:** use the requested model for every role when supported.
- **`--reasoning-effort <value>`:** pass the exact value to every role and retry. Default to `medium` when omitted.

Record explicit paths, exclusions, mode, model, reasoning effort, and unsupported overrides in `run-metadata.json`.

## Phase 1 — Create the durable run

Create `.ton-audit-tmp/<UTC timestamp>-<short source hash>/` under the audited repository. If this directory cannot be created and retained, set `audit_status=incomplete` and stop before claiming a comprehensive audit.

Do not substitute a source-only review for a requested `--deep` run when this
gate fails. Its finding list cannot establish coverage;
report the blocked run and the missing artifacts explicitly.

Create these paths immediately:

```text
run-metadata.json
scope-manifest.json
source-index.json
source.md
inventory.json
probe-manifest.json
coverage-map.json
boundary-closure-matrix.json
agent-manifest.md
results/
raw-candidate-ledger.json
candidate-ledger.json
merge-log.json
export-reconciliation.json
historical-comparison.json
negative-space-review.json
assurance-summary.json
```

Store source hashes and the skill/reference hashes before role execution. Update artifacts atomically as work progresses.

After the source-derived inventory has been frozen, tests, wrappers, deployment scripts, configuration, and design documentation may be inspected as secondary evidence. Record them separately in the scope manifest. They may confirm reachability or refute assumptions, but they must not replace source coverage or silently convert a code-permitted configuration into “impossible.”

## Phase 2 — Build a deterministic source inventory

Enumerate source constructs before launching discovery. Assign stable run-local IDs at the following granularity:

- every internal, external, bounced, fallback, empty-body, and getter entrypoint;
- every opcode/message branch and sender/authentication condition;
- every persistent scalar, dictionary, nested record, counter, pool, supply, and aggregate;
- every pending record and its create, update, complete, bounce, retry, cancel, expire, and cleanup sites;
- every value-bearing send, send mode, reserve, and optimistic state mutation;
- every callback, acknowledgement, notification, excess, and bounce edge;
- every `queryId`, nonce, request ID, operation category, and correlation namespace;
- every parser/serializer and producer/consumer schema pair;
- every arithmetic, rounding, bound, cell-size, storage-growth, and gas boundary;
- every deploy/init, ownership, upgrade, code replacement, and emergency path.

For every address or identity persisted from a message, record where it is
later used as a recipient, sender expectation, owner, or authorization
subject. Record an actor that can hold two authorized roles at once as a
separate identity case; a check may behave differently when the roles overlap.

Do not use one inventory item to cover an entire contract when it contains multiple independently failing transitions.

Record `inventory.json` as an array. Each item must have a stable `id`,
`kind`, `construct`, `location`, and `boundary_cases` array using the case keys
in `references/boundary-coverage.md`. Use a separate item for each entrypoint,
pending record, value-bearing send, serializer, and stored identity flow. An
empty `boundary_cases` array needs a source-backed
`boundary_exclusion_reason`; it cannot mean "not checked." Compare the
inventory against the source index before freezing it. A single item named
after a contract or a whole handler cannot cover its distinct branches and
sends.

### Protocol-lifecycle records

For every asynchronous operation, construct a lifecycle record:

```text
entrypoint
  -> authorization and value checks
  -> pending-state creation/reservation
  -> outbound message and correlation tuple
  -> remote success/failure
  -> callback sender/category/query validation
  -> settlement and pending deletion
  -> bounce rollback
  -> retry/cancel/timeout/cleanup
```

The correlation tuple must include every identity necessary to distinguish concurrent operation classes, not only a shared numeric ID.

### Conservation and consistency invariants

Derive explicit invariants, including where applicable:

- contract balance = available funds + reserved liabilities;
- global totals = sum of live local positions/claims/schedules;
- reward pools and accrued debts change exactly once per settlement;
- supply/minted/burned counters match completed, not merely requested, operations;
- a completed or deleted position cannot retain an unrecorded user entitlement;
- every optimistic mutation has a complete inverse or an explicit debt record;
- every pending record has a terminal success, failure, retry, or recovery path.

## Phase 3 — Generate source-specific probes

Generate probes for each applicable inventory or lifecycle item:

- local dispatch failure;
- remote action/compute failure;
- bounced and non-bounced failure;
- absent, delayed, duplicated, replayed, or reordered response;
- concurrent operations sharing an identifier or mutable state;
- unrelated operation producing a structurally valid callback;
- mutable state changing between request and resolution;
- partial settlement and cleanup failure;
- malformed, truncated, extended, or mismatched encoding;
- boundary arithmetic, rounding, underflow/overflow, and zero values;
- insufficient TON, Jettons, reward liquidity, or reserved backing;
- user exit while rewards or liabilities remain unpaid;
- deployment/configuration combinations allowed by code but not covered by the default script.

Do not create artificial probes for absent constructs. Every probe must reference a concrete inventory ID and source location.

Initialize `coverage-map.json` from the frozen inventory, probe manifest, and asynchronous lifecycle records. Give each inventory item, probe, and applicable lifecycle transition one stable coverage key. A coverage record contains its key, source locations, behavioral conclusion, disposition (`answered`, `not_applicable`, or `unresolved`), linked candidate IDs, and a focused follow-up ID when needed. `not_applicable` requires a source-backed reason; a line reference alone is not a behavioral conclusion.

### Boundary closure matrix

Before discovery, create **one row for each `(inventory_id, case)` pair** in
`boundary_cases`. Do not create one row per case for the entire repository or
combine two pending records, sends, serializers, or identity flows in one row.
The row starts `unresolved` until a concrete bound or message sequence and
terminal state are established. A nearby finding does not close another row.
Use the exact JSON schema and case keys in `references/boundary-coverage.md`.

| Case | Required trace |
| --- | --- |
| Cell and storage growth | Calculate worst-case serialized bits and references for each growing inline record. Include variable-length coin encodings and states reached after repeated normal operations. Check writes that follow an earlier cross-contract payment or other irreversible step. |
| Message-value subsidy | For each externally triggered notification or callback **branch**, compare minimum incoming TON with every downstream send, compute/storage cost, send mode, and returned excess. Trace repeated low-value inputs and the state after balance exhaustion. |
| Pending-state liveness | For each pending record and retry/reservation entrypoint, test failed eligibility, rejected callback, absent response, overlapping retries before the first acknowledgement, caller abandonment, and unbounded metadata or unique IDs. Account for forwarding fees and storage rent. |
| Role overlap | Test a principal satisfying multiple roles simultaneously, such as creator and owner. Apply later restrictions to the combined identity. |
| Action failure after commit | For each state or supply update followed by a send, inspect action-phase failure and ignore-error modes. Determine whether an acknowledgement can disappear after commit and whether the initiator can reconcile. |
| Address round trip | Compare workchain, code, initialization, and address constraints when an identity enters storage with constraints applied when it is later used for payout, refund, callback authentication, or recovery. |
| Snapshot and mutable state | For each immutable snapshot or registered total later compared with live state, test changes during the request and changes that restore the same aggregate with a different account distribution. Trace both false rejection and stale acceptance. |
| Callback origin | For each settlement callback, determine whether an unrelated operation can make the expected sender emit the same opcode and ID. Check the operation type, origin, amount, recipient, and rollback record for every affected contract. |

These rows guide discovery; they are not findings. Every candidate still needs
a reachable path and the validation gates below. Keep unresolved rows visible
in the coverage map and final audit status.

Reconcile the matrix against the frozen inventory after each update. For
every inventory item's `boundary_cases`, check that the corresponding pair
appears once, links a concrete probe and coverage key, and has a behavioral
answer or an explicit unresolved follow-up. Verify the inventory itself
against the source index and contract-local entrypoint walk.

## Phase 4 — Independent discovery

Read `references/report-formatting.md`, `references/judging.md`, and the relevant role instructions. Build bundles in the durable run directory. Prompts must reference bundle paths instead of inlining the bundle.

Give each discovery role the relevant unresolved boundary rows from the
source-derived matrix. Do not include another role's candidate or conclusion.

### Round A — Specialist discovery

Run the original specialist roles independently:

| Role | Required instruction resources |
| --- | --- |
| 1 — attack-vector and checklist scan | `attack-vectors/attack-vectors-1.md` through `attack-vectors/attack-vectors-7.md`, `hacking-agents/vector-scan-agent.md`, `hacking-agents/audit-checklist-agent.md`, `hacking-agents/shared-rules.md` |
| 2 — math and precision | `hacking-agents/math-precision-agent.md`, `hacking-agents/shared-rules.md` |
| 3 — authorization and trust boundaries | `hacking-agents/access-control-agent.md`, `hacking-agents/shared-rules.md` |
| 4 — economic security and conservation | `hacking-agents/economic-security-agent.md`, `hacking-agents/shared-rules.md` |
| 5 — execution traces and asynchronous ordering | `hacking-agents/execution-trace-agent.md`, `hacking-agents/shared-rules.md` |
| 6 — invariants and state consistency | `hacking-agents/invariant-agent.md`, `hacking-agents/shared-rules.md` |
| 7 — periphery, interfaces, and language hazards | `hacking-agents/periphery-agent.md`, the applicable Tact/Tolk instruction, `hacking-agents/shared-rules.md` |
| 8 — first principles and language hazards | `hacking-agents/first-principles-agent.md`, the applicable Tolk/Tact instruction, `hacking-agents/shared-rules.md` |

Resolve all paths relative to this skill's `references/` directory. When `--deep` is active, also run the TON protocol role with `hacking-agents/ton-protocol-agent.md` and `hacking-agents/shared-rules.md`.

### Round B — Contract-local discovery (`--deep`)

For each contract or tightly coupled domain, run a pass containing only:

- that contract/domain's source;
- directly imported storage/messages/helpers;
- its inventory IDs and lifecycle records;
- shared output rules.

This pass must walk every entrypoint in source order and report a coverage answer even if it produces no candidate.

### Round C — Cross-contract protocol discovery (`--deep`)

Create bundles per protocol edge rather than per filename, for example Jetton wallet → staking, staking → governance, sale → wallet, or vesting → wallet. Trace both sides of every message and compare:

- sender expectations;
- opcode and schema;
- query/category identity;
- amount and value semantics;
- success acknowledgement;
- bounce and retry ownership;
- reservation and settlement timing.

### Discovery output rule

Discovery roles must emit every concrete suspicion as `FINDING` or `LEAD`. A record qualifies for capture when it includes at least:

- one concrete source location;
- a reachable or plausibly reachable entrypoint;
- a suspected violated invariant or trust transition;
- the missing evidence needed for confirmation.

Discovery roles may lower confidence, but may not silently reject a source-backed suspicion. Statements such as “probably intended,” “deployment likely prevents it,” or “impact not yet proven” are reasons for a `LEAD`, not deletion.

If subagents are unavailable, execute roles sequentially with isolated bundles and separate result files. Do not reuse earlier role conclusions in later discovery prompts. Record that independence was degraded in `run-metadata.json`.

## Phase 5 — Capture raw candidates before judgment

Parse every `FINDING` and `LEAD` from every result into `raw-candidate-ledger.json` before validation or merging. Assign immutable IDs such as `RAW-0001`.

Required fields:

```text
raw_id, source_role, language, locations, entrypoint, suspected_root_cause,
preconditions, suspected_failing_transition, suspected_impact,
missing_evidence, confidence, inventory_ids, probe_ids
```

The raw count must equal the parsed role-result count plus explicitly labeled post-validation historical follow-up candidates, if any. Keep these two provenance groups separate and hash the role results, follow-up records, and ledger records. Any unexplained mismatch makes the audit incomplete.

## Phase 6 — Complete traces and validate

Validate each raw candidate separately before deduplication.

1. **Trace completion:** follow attacker-controlled or operational input through authorization, parsing, state mutation, outbound messages, callback/bounce handling, and terminal impact.
2. **Refutation search:** actively look for source conditions that block reachability, trigger, or impact.
   For a bounce-based claim, prove which outbound message can generate the bounced message and who can influence its destination, body, and failure. For a transfer with independent tax and delivery messages, trace each leg through success, failure, bounce, and rollback before claiming a retained charge or loss.
3. **Four gates:** apply `references/judging.md` in its defined order.
4. **Focused retry:** `UNCERTAIN` receives one independent, source-backed validation pass. If still unresolved, retain it as `review_trail` with exact missing evidence.
5. **Fix-boundary check:** determine whether two candidates require the same local remediation; this informs later deduplication but does not merge them yet.

Only this phase may assign `source_refuted`. A refutation must cite the blocking source path or authoritative protocol rule and explain why alternate message orderings, bounce behavior, or configuration do not bypass it. A test that expects the observed behavior proves the behavior, not its intended safety. If a safety judgment depends on intended behavior and tests, comments, or design claims disagree, record the conflict and retain `review_trail` until intent and impact are resolved; do not infer safety from a passing test alone.

Do not validate a bounced-message forgery claim from a missing sender check alone; establish a reachable, attacker-influenced protocol bounce. Do not validate a failed-transfer tax claim from separate send actions alone; establish the final tax and delivery states and the asserted refund obligation. If either chain cannot be completed, retain `review_trail` with the missing evidence.

Write normalized records to `candidate-ledger.json`. Allowed dispositions before merge are `validated`, `source_refuted`, `review_trail`, and `out_of_scope`.

## Phase 7 — Negative-space review

Before deduplication, inspect what did **not** generate a validated candidate. Record answers in `negative-space-review.json`:

- Which handlers have no candidate and no source-backed refutation?
- Which pending records lack a tested duplicate, reorder, bounce, retry, and cleanup path?
- Which value-bearing sends lack authenticated acknowledgement or complete rollback?
- Which operation classes share a numeric correlation namespace?
- Which optimistic deletions can discard unpaid principal, rewards, refunds, or claims?
- Which aggregates can diverge from restored local records?
- Which external integrations rely on hard-coded wallet code, workchain, deployment script, or tax configuration?
- Which callbacks can be satisfied by an unrelated but structurally valid transfer?
- Which global locks depend on a remote response that may never arrive?
- Which inventory/probe answers are merely line references rather than behavioral conclusions?
- Which boundary closure rows lack a concrete limit, failing sequence, or source-backed safe-path answer?
- Which source index constructs have no inventory item, or have one broad inventory item standing in for several distinct branches, sends, or records?

Every uncovered item triggers one focused discovery pass. New leads return to the raw ledger and follow the same trace and gate process. Do not bypass validation.

Reconcile the coverage map after these passes: compare its keys with the frozen inventory, probes, and lifecycle transitions; reject missing or duplicate keys. An `answered` item must explain the observed success or failure behavior and link any resulting candidate or source-backed safe-path conclusion. Leave unresolved items explicitly marked `unresolved`, with the attempted follow-up and missing evidence. Do not convert lack of a candidate into `not_applicable`.

## Phase 8 — Deduplicate validated candidates

Deduplicate only after validation. Two candidates may merge only when all of these are materially the same:

- reachability and attacker capability;
- failing mechanism;
- violated invariant;
- terminal state and impact;
- fix boundary and local remediation.

Do not merge distinct parser, accounting, lifecycle, authentication, correlation, or interface failures merely because they share a contract, helper, message, or broad impact.

Record every decision in `merge-log.json`, including both candidate IDs and a field-by-field equivalence explanation. Terminal dispositions are `reported`, `merged_into:<id>`, `source_refuted`, `review_trail`, and `out_of_scope`.

Check composite chains after deduplication. Add a chain only when one finding establishes another's precondition and the combined impact is strictly worse.

### Post-validation historical comparison

If historical findings are available, unseal them only after current validation and merge decisions are frozen. Write `historical-comparison.json` with one row per distinct historical root cause and affected contract path: historical ID, prior triage/validation status, current source locations, matching current candidate or finding IDs, and outcome (`covered`, `source_refuted`, `review_trail`, `unexamined`, or `no_longer_applicable`). Require a source-backed explanation for `source_refuted` and `no_longer_applicable`; title similarity and a shared broad mechanism are not enough for `covered`. Treat conflicting prior labels, such as `invalid` paired with `confirmed`, as unresolved until checked against current source. A historical row never changes a frozen current candidate silently: if comparison reveals a credible missing path, record its historical provenance as a separate post-validation raw candidate, validate it through Phase 6, then rerun merge and reconciliation. Keep `unexamined` visible rather than counting it as covered. If no historical material exists, record an empty comparison with `not_applicable` status.

## Phase 9 — Verify and report

For candidates with confidence at least 80, trace the recommended fix through success, bounce, retry, and cleanup paths. Reject fixes that introduce permanent locks, unrecorded liabilities, accounting drift, or authorization regressions.

Before formatting or updating any final findings artifact, write `export-reconciliation.json`. Give every current raw candidate one row with its terminal disposition, merge target if any, and final finding ID if reported. Map every final finding ID back to one or more validated candidate IDs and record its exact contract path and failing transition. Compare IDs and counts against the raw ledger, candidate ledger, and merge log; a validated candidate cannot disappear because a later pass or report generator omitted it. If a final artifact aggregates several runs, include each source run and reconcile the union of validated candidates before assigning final IDs. Resolve duplicate exports by the Phase 8 equivalence test, recording both source IDs and the merge target. Do not publish a reconciled artifact while a validated candidate lacks a final disposition; keep the export incomplete and list the missing IDs.

Format confirmed findings according to `references/report-formatting.md`. Do not inflate the report with unresolved leads, but summarize review trails and refutations in durable artifacts. If `--file-output` is absent, do not create a report file.

The final response must state:

- audit status and run directory;
- source hash and execution mode;
- planned/captured roles;
- raw, validated, reported, merged, refuted, and review-trail counts;
- final-export reconciliation counts, any missing candidate IDs, and historical comparison outcomes when applicable;
- inventory and probe coverage;
- applicable boundary-pair count and unresolved boundary rows;
- unresolved coverage keys and their follow-up outcomes;
- test/build evidence;
- any degraded independence or unavailable tooling.

## Completion gate

Claim `audit_status=complete` only when all conditions hold:

- every planned role has a captured, hashed result;
- every parsed finding and lead appears in the raw ledger;
- every raw candidate has exactly one validation disposition;
- every inventory and probe ID has one source-backed behavioral answer;
- every applicable boundary closure row has a linked source-backed answer;
- every boundary row has exactly one terminal disposition and reconciles with the coverage map;
- `coverage-map.json` has exactly one record per inventory item, probe, and applicable lifecycle transition, with no `unresolved` records;
- every asynchronous lifecycle has success, failure, bounce, retry, and cleanup coverage or an explicit `not_applicable` reason;
- negative-space review is complete and all triggered follow-ups are reconciled;
- every merge is recorded and field-by-field justified;
- every validated candidate and final finding reconciles through `export-reconciliation.json`, including across runs when results are aggregated;
- every available historical finding has a comparison outcome, with `unexamined` paths explicitly disclosed;
- counts in all artifacts reconcile;
- durable artifacts remain under the repository run directory.

Missing coverage, missing results, ephemeral-only artifacts, unreconciled historical paths, or count mismatches require `audit_status=incomplete`. An incomplete run may report individually validated findings, but must label the result as a partial set and list unresolved coverage and candidate IDs; it must not present the list or its absence of other findings as comprehensive.
