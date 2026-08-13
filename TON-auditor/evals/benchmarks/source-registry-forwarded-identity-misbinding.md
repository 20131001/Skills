# Source Registry Forwarded Identity Misbinding

## Purpose

Regression case for source/verifier registry audits where a relay authenticates one verifier identity in an outer message but forwards a nested publication payload that claims another verifier identity.

## Minimal Shape

- `verifier-registry` accepts public verifier registration.
- `update_verifier` can store `quorum = 0` or another weak signer threshold.
- `forward_message` authenticates or loads outer `verifier_id = A`, checks freshness and source address, then forwards caller-supplied `payload_to_forward`.
- `sources-registry` accepts `deploy_source_item` only from `verifier-registry`, but reads inner `verifier_id = B`, code hash, and source content from the forwarded payload without checking `B == A`.

## Expected Findings

- High-confidence threshold/quorum finding when `quorum = 0` or equivalent signer-set validation allows unsigned forwarding.
- Separate high-confidence `forwarded-identity-misbinding` finding when outer `A` can differ from inner `B` and the downstream registry stores or publishes under `B`.

## Must Not Happen

- Do not merge the forwarded identity/provenance issue into the quorum finding.
- Do not treat `sender == verifier-registry` in the downstream registry as proof that the inner `verifier_id`, code hash, or source content is bound to the outer authenticated verifier.
- Do not leave this as a Review Trail if the source shows a complete caller -> forwarder -> registry update/deploy path.
