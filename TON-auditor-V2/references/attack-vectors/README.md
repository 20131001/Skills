# Attack Vector Split

Attack vectors are split by what the auditor needs to reason about:

- `shared/ton-chain.md` contains TON/TVM, message-flow, gas, bounce, send-mode, standards, and cross-contract vectors. Every detected language receives this file.
- `tep/ton-teps.md` contains TON Enhancement Proposal standard-conformance vectors. Every detected language receives this file after shared TON vectors.
- `languages/func.md` contains FunC syntax, parser, storage, and control-flow vectors.
- `languages/tolk.md` contains Tolk typed/lazy parser, storage, and control-flow vectors.
- `languages/tact.md` contains Tact receiver, typed message, fallback, storage, and control-flow vectors.

Some root causes appear in multiple language files with different IDs. That is intentional: the underlying issue is the same, but the concrete proof and fix differ by language.

## Classification

**Shared TON/TVM vectors:** TC1-TC58.

**TEP standard vectors:** TP1-TP26.

**Language-surface vectors:** FC1-FC9, TL1-TL5, TA1-TA17.

- FunC-only in this split: FC1, FC2, FC4, FC9.
- Shared by multiple language files with language-specific wording: FC3/TL1/TA1, FC5/TL2/TA2, FC6/TL3/TA3, FC7/TL4/TA4, FC8/TL5/TA5.
- Tact-only in this split: TA6, TA7, TA8, TA9, TA10, TA11, TA12, TA13, TA14, TA15, TA16, TA17.

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
