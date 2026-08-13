# Attack Vector Split

Attack vectors are split by what the auditor needs to reason about:

- `shared/ton-chain.md` contains TON/TVM, message-flow, gas, bounce, send-mode, standards, and cross-contract vectors. Every detected language receives this file.
- `tep/ton-teps.md` contains TON Enhancement Proposal standard-conformance vectors. Every detected language receives this file after shared TON vectors.
- `languages/func.md` contains FunC syntax, parser, storage, and control-flow vectors.
- `languages/tolk.md` contains Tolk typed/lazy parser, storage, and control-flow vectors.
- `languages/tact.md` contains Tact receiver, typed message, fallback, storage, and control-flow vectors.

Some root causes appear in multiple language files with different IDs. That is intentional: the underlying issue is the same, but the concrete proof and fix differ by language.

## Classification

**Shared TON/TVM vectors:** TC1-TC77.

**TEP standard vectors:** TP1-TP26.

**Language-surface vectors:** FC1-FC12, TL1-TL10, TA1-TA21.

- FunC-only in this split: FC1, FC2, FC4, FC9-FC12.
- Shared by multiple language files with language-specific wording: FC3/TL1/TA1, FC5/TL2/TA2, FC6/TL3/TA3, FC7/TL4/TA4, FC8/TL5/TA5.
- Tolk-only in this split: TL6-TL10.
- Tact-only in this split: TA6-TA21.

## Source Traceability

- `sources/positive-security-ton-audit-guide.md` maps every security-relevant checklist line in PositiveSecurity's guide to shared and language-specific vector IDs.
- `../../audit-checklists/positive-security.md` contains the same upstream checklist as mandatory runtime coverage rows, including process/evidence controls that should not be misrepresented as attack vectors.
- `../../audit-checklists/sanbir-hacking-agents.md` contains normalized runtime rows from Sanbir's specialist hacking agents and protocol checklists.
- Preserve source line numbers and upstream commit metadata when refreshing the map so reviewers can distinguish newly introduced guidance from local interpretation.
- A checklist line may map to multiple vectors because one review control can expose several independent root causes. Lines that describe audit evidence rather than an exploitable condition are marked as audit controls instead of inventing a vulnerability ID.

## Prefix Rule

Each attack-vector file starts from 1 and uses a file-specific prefix:

- `TC` = TON-chain shared vectors in `shared/ton-chain.md`
- `TP` = TEP-derived standard vectors in `tep/ton-teps.md`
- `FC` = FunC language vectors in `languages/func.md`
- `TL` = Tolk language vectors in `languages/tolk.md`
- `TA` = Tact language vectors in `languages/tact.md`

## Bundling Rule

For a detected target language, build vector bundles from:

1. all files in `shared/`
2. all files in `tep/`
3. that language's file under `languages/`

Do not send any language-only vector file to a different target language. FunC-only vectors stay with FunC, Tolk-only vectors stay with Tolk, and Tact-only vectors stay with Tact.
