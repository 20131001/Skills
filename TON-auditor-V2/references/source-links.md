# Source Links

Use this file as the external-source map for maintaining this skill. The audit agents should rely on bundled local references and attack-vector files during normal audits; external links are for refresh, verification, or user-requested research.

## TON Standards

- TON Enhancement Proposals: `https://github.com/ton-blockchain/TEPs/tree/master`
- TEP text files: `https://github.com/ton-blockchain/TEPs/tree/master/text`

Use these links to refresh TEP-derived vectors, getter/message ABI notes, metadata serialization requirements, and standard-conformance rules.

## FunC Lessons

- FunC course: `https://github.com/markokhman/func-course/tree/main`

Use this link to refresh FunC lesson-derived guidance such as external-message handling, replay protection, `accept_message()` ordering, carry-value, gas management, and storage management.

## Security Best Practices

- Tact security best practices: `https://docs.tact-lang.org/book/security-best-practices/`
- TON security best practices: `https://docs.ton.org/contract-dev/techniques/security`

Use these links to refresh shared TON security guidance and Tact-specific checks. Keep the local summaries in `security-best-practices.md` and `standards-and-lessons.md` synchronized with high-signal changes rather than asking audit agents to browse these links during every audit.

## Maintenance Rule

- Do not replace local reference files with live links in audit bundles.
- Refresh local files from these links when the user asks to update source material or when attack-vector coverage is being expanded.
- If a link disagrees with a local reference, prefer the link only after confirming the current upstream content and then update the local reference or attack-vector file.
