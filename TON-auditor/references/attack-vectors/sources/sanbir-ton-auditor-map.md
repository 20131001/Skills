# Sanbir TON Auditor — Comparative Vector Map

Upstream: `https://github.com/sanbir/ton-auditor-skills/tree/main/ton-auditor`

Reviewed commit: `0fa20bef5c472f005d2bc13289229a7b4b3ce312`

The upstream baseline defines V1-V120 in four language-agnostic files. This skill keeps the useful root causes but separates shared TON/TVM, TEP conformance, and language-specific proof obligations. The table records where every upstream vector lands locally; a local range means the upstream item is decomposed into multiple stronger checks.

Runtime enforcement of Sanbir's specialist-agent instructions and protocol checklists is provided separately by `../../audit-checklists/sanbir-hacking-agents.md`.

| Upstream | Local vector(s) |
|---|---|
| V1-V2 | TC14, TC18, TC58, TP7-TP10 |
| V3-V4 | TC11, TC15, TC34 |
| V5-V6 | TC3, TC31, TC53, TC56 |
| V7 | FC6, TL3/TL6, TA3 |
| V8 | TC9, TP1 |
| V9 | TC50, FC9 |
| V10-V11 | TC20, TC21, TC29, TC45, TC56 |
| V12-V13 | TC18, TC32, TC35, TC72 |
| V14-V15 | TC15, TC29, TC34, TC74 |
| V16 | TC23, TC62 |
| V17-V18 | TC20, TC45, TC56 |
| V19 | TC1, TC38 |
| V20 | TC19, TC76, TA21 |
| V21-V22 | TC38-TC40, TC44, TC60; FC3, TL1, TA1 |
| V23-V24 | TC16, TC28, TC74; TA13-TA14 |
| V25 | FC1 |
| V26 | TC6, TC75 |
| V27 | TC16, TC74 |
| V28 | TC28, TC30, TC41-TC43; FC7, TL4/TL7, TA4/TA7 |
| V29 | TC21, TC26, TC54, TC77 |
| V30 | TP1-TP26, TC41-TC52 |
| V31-V35 | TC7, TC12, TC26, TC53, TC57, TC77 |
| V36-V38 | TC35, TC61, TC63 |
| V39 | TC4, TC29 |
| V40 | TC64, TA19 |
| V41-V44 | TC16, TC74 |
| V45-V47 | TC9, TC46, TC58, TC76; TA16-TA17 |
| V48 | TC11, TC33, TC34 |
| V49-V50 | TC12, TC57 |
| V51 | TC20, TC29, TC45 |
| V52 | TC77 |
| V53 | TC75, FC12 |
| V54 | FC11, TL10 |
| V55-V56 | TC18, TC62, TA9, TA15 |
| V57 | TC41-TC43, TA4, TA6-TA7 |
| V58 | TC16, TC74, TA13-TA14 |
| V59 | FC10 |
| V60 | TC20, TC29, TC45 |
| V61-V64 | TC39, TC51, TC60 |
| V65 | TC67 |
| V66 | TC39, TC50, TC60 |
| V67-V68 | TC59, TC70 |
| V69 | TC13, TC37, TC60 |
| V70 | TC26, TC37, TC44, TC77, TP9 |
| V71 | TC12, TC57, TC77 |
| V72 | TC14, TC26, TC44, TC58, TP7-TP10 |
| V73 | TC37, TC58, TC76, TP2-TP3 |
| V74 | TC24, TC35, TC72 |
| V75 | TC41, TC50; FC7, TL4, TA7 |
| V76 | TC7, TC12, TC53 |
| V77 | TC16, TC55, TC74 |
| V78 | TC77; FC5/FC7, TL2/TL4, TA2/TA4 |
| V79 | TC26, TC77 |
| V80 | TC60, TC68 |
| V81 | TC28, TC60 |
| V82 | TC64, TA19 |
| V83 | TC26, TC37, TP9 |
| V84 | TC41, TC50; FC7, TL4, TA7 |
| V85 | TC20, TC38, TC45 |
| V86 | TC21, TC26, TC54 |
| V87 | TC39, TC59, TC60, TP5, TP22-TP23 |
| V88 | TC73 |
| V89 | TC7, TC16, TC74 |
| V90 | TC61, TC63; FC11, TL10 |
| V91-V94 | TC65, TC73 |
| V95 | TC66 |
| V96 | TC67 |
| V97-V99 | TC64, TC68 |
| V100-V103 | TC69 |
| V104-V106 | TC70 |
| V107-V108 | TC33, TC71 |
| V109-V110 | TC72 |
| V111-V112 | TC18, TC24, TC58, TP2-TP3 |
| V113 | TC61, TC73 |
| V114 | TC9, TC20, TC45, TP1 |
| V115 | TC14, TC58, TC73, TP10 |
| V116 | TC32, TC72 |
| V117 | TC25, TC29, TC53, TC56 |
| V118 | TC76, TA21 |
| V119 | TC6, TC75, FC10, TA20 |
| V120 | TC61, TC75; FC12, TA20 |

## Improvements over the upstream organization

- Tolk is first-class instead of being absent from discovery and vector classification.
- TEP vectors are pinned to specific standards and schemas rather than a single generic “non-compliance” item.
- Language footguns are not sent to unrelated languages.
- Vector agents are supplemented by deterministic semantic coverage probes for lifecycle, parsers, accounting, storage/gas, and economic protocols.
- Oracle, vault, staking, lending, DEX, governance, bridge, dependency, toolchain, upgrade, and off-chain provenance families are now explicit coverage obligations rather than relying on free-form agent intuition.
