# Boundary Coverage Contract

The matrix tracks **source instances**, not checklist categories. Freeze the
source inventory before discovery, then derive exactly one matrix row for
each pair of inventory item and applicable case. Keep a distinct inventory
item for each pending record, value-bearing send, serializer, stored address
flow, and branch with an independently failing terminal state.

## Case keys and applicability

| Key | Apply to | Minimum behavioral answer |
| --- | --- | --- |
| `cell_growth` | Growing inline cells, maps, variable-width fields, and serialization writes | Maximum bits and refs, reachable state, write result, and any earlier committed payment |
| `message_value` | Each externally triggered notification or callback branch that causes sends, refunds, or material storage/computation | Minimum incoming TON, every downstream action and fee budget including carry modes, repeated low-value input, exhaustion state |
| `pending_liveness` | Each pending record and each independently retryable or reservable operation | Creation, failed eligibility, missing/rejected reply, bounce, overlapping retries before acknowledgement, abandoned caller, cleanup authority and cost |
| `role_overlap` | A branch where one address can satisfy multiple authorization roles | Exact combined identity, later restrictions, and reachable outcome |
| `post_commit_action` | A state/supply update followed by a send or acknowledgement | Action-mode failure, committed state, recipient/initiator observation, and reconciliation path |
| `address_round_trip` | A stored identity later used as recipient, expected sender, or authority | Constraints at intake and later use, including workchain, code, initialization and recoverability |
| `snapshot_version` | An immutable snapshot, registry entry, or recorded aggregate later checked against mutable state | Changed total, unchanged total with changed account distribution, response reordering, false rejection and stale acceptance |
| `callback_origin` | Each settlement callback keyed by a public or reused identifier | Whether an unrelated operation can make the same trusted sender emit the same opcode and ID; origin, category, amount, recipient and rollback record |

Applicability is per source construct. A failed refund may need both
`message_value` and `pending_liveness`; a second refund record needs its own
rows. Do not mark a case `not_applicable` merely because no finding was found.

## JSON shape

`inventory.json` is an array of records such as:

```json
{
  "id": "INV-014",
  "kind": "pending_record",
  "construct": "refund request pending entry",
  "location": "contracts/Example.tolk:120",
  "boundary_cases": ["pending_liveness", "message_value"]
}
```

If `boundary_cases` is empty, provide `boundary_exclusion_reason` with a
source location and explanation. `probe-manifest.json` is an array of records
with unique `id` and `inventory_ids`. Each applicable pair must appear once
in `boundary-closure-matrix.json`:

```json
{
  "id": "BND-014-01",
  "inventory_id": "INV-014",
  "case": "pending_liveness",
  "probe_ids": ["PROBE-014"],
  "location": "contracts/Example.tolk:120",
  "disposition": "unresolved",
  "trace": "",
  "terminal_state": "",
  "conclusion": "",
  "candidate_ids": [],
  "follow_up_id": "FU-014",
  "missing_evidence": "callback failure ordering"
}
```

Allowed dispositions are `answered`, `not_applicable`, and `unresolved`.
`answered` needs a concrete `trace`, `terminal_state`, `conclusion`, and at
least one linked probe. A source-backed safe path can be `answered` without a
finding. `not_applicable` needs a `reason` citing the blocking source path.
`unresolved` needs `follow_up_id` and `missing_evidence`. Keep the evidence
for every row, including refutations and review trails.

`coverage-map.json` must have one record with key
`BOUNDARY:<inventory_id>:<case>` for each matrix row. Its `disposition` must
match the matrix row. It also retains the separate inventory, probe, and
lifecycle keys required by the main skill. Every coverage record uses
`key`, `disposition`, and a behavioral `conclusion`; an unresolved record
instead has `follow_up_id` and `missing_evidence`. Reconcile the expected
`(inventory_id, case)` pairs, matrix rows, and coverage keys before reporting.
The reviewer also verifies that source enumeration and behavioral conclusions
are complete.
