# PositiveSecurity TON Audit Guide — Line-to-Vector Map

Upstream: `https://github.com/PositiveSecurity/ton-audit-guide`

Reviewed commit: `55827b9b4201f1b12a4586ad8bf66743feb4dadd`

Source file: `README.md` (448 lines at the reviewed commit).

This map extracts every security-relevant checklist bullet. A comma-separated source-line cell means each listed line independently maps to the same vector set. `Control` marks audit process/evidence requirements rather than inventing a vulnerability. Language is the primary implementation surface; `shared` applies to FunC, Tolk, and Tact.

Runtime enforcement is provided by `../../audit-checklists/positive-security.md`; this file is the vector/provenance explanation, not the completion mechanism.

Official terminology was cross-checked against TON Docs `start-here`, `contracts/techniques/security`, and Tolk `features/message-handling` pages discovered through `llms.txt` on 2026-08-13.

## Scope, architecture, and entrypoints

| Source line(s) | Language | Attack vector(s) / control |
|---|---|---|
| 11-13 | shared | TC61 |
| 14 | Tolk | TC61, TL6-TL9 |
| 15 | FunC | FC1-FC7, FC12, TC63 |
| 16 | Tact | TC61, TC62, TA12, TA15, TA18, TA20 |
| 17-18 | shared | TC18, TC35, TC65, TC73 |
| 19 | shared | Control: tool coverage is not proof |
| 25-31 | shared | TC7, TC12, TC26, TC53, TC57, TC62, TC77; Control: message/value/state/invariant model |
| 32-35 | shared | TC7, TC12, TC20, TC26, TC53, TC65, TC73 |
| 41-43 | shared | TC62 |
| 44-46 | shared | TC22, TC23, TC38, TC62 |
| 47 | shared | TC3 |
| 48 | FunC | TC3, FC6 |
| 49 | Tolk | TC23, TC62, TL5, TL6 |
| 50 | Tact | TC3, TC62, TA9, TA12, TA15 |
| 51 | shared | TC49, TC73 |

## Authorization, replay, external messages, and async state

| Source line(s) | Language | Attack vector(s) / control |
|---|---|---|
| 57-60 | shared | TC18, TC32, TC35, TC62, TC72 |
| 61-62 | shared | TC14, TC18, TC25, TC58 |
| 63 | shared | TC9 |
| 64-67 | shared | TC11, TC17, TC33, TC34 |
| 68 | shared | TC19, TC76 |
| 74-78 | shared | TC11, TC15-TC17, TC34 |
| 79-80 | Tolk | TC34, TC54, TL9 |
| 81-82 | shared | TC1, TC11, TC33 |
| 88-98 | shared | TC7, TC12, TC20-TC22, TC26, TC27, TC53, TC57, TC77 |

## Bounce, sends, action phase, gas, and storage

| Source line(s) | Language | Attack vector(s) / control |
|---|---|---|
| 104-106 | shared | TC3, TC26, TC56, TC57 |
| 107 | shared/Tact | TC31, TA12 |
| 108-110 | Tolk | TC31, TC56, TL3 |
| 111-112 | shared | TC3, TC18, TC53, TC57 |
| 118-120 | shared | TC29, TC30, TC45, TC56; Control: send/reserve inventory |
| 121-124 | shared | TC21, TC26, TC29, TC56 |
| 125-128 | shared | TC4, TC21, TC26, TC29 |
| 129-131 | shared | TC16, TC26, TC57, TC74 |
| 132 | shared | TC73 |
| 138-143 | shared | TC13, TC20, TC29, TC45, TC74 |
| 144-148 | shared | TC16, TC55, TC74 |
| 149-151 | shared | TC13, TC36, TC48, TC53, TC57 |
| 152 | shared | TC61 |
| 158-159 | shared | TC61, TC63 |
| 160 | FunC | FC6, FC7 |
| 161 | Tolk/Tact | TL3, TL4, TL7, TA3, TA4, TA7 |
| 162 | shared | TC28, TC74 |
| 163-165 | shared | TC42, TC55; FunC FC6; Tolk TL3/TL6; Tact TA3/TA8 |
| 166-168 | shared | TC39, TC50, TC77; FunC FC3-FC5; Tolk TL1/TL2/TL8; Tact TA1/TA2/TA7 |
| 169 | shared | TC61, TC63, TC73 |

## Arithmetic, randomness, upgrades, and governance

| Source line(s) | Language | Attack vector(s) / control |
|---|---|---|
| 175-177 | shared | TC38-TC40, TC44, TC60 |
| 178-180 | shared | TC51, TC59, TC60, TC67-TC70 |
| 181-184 | shared | TC28, TC39, TC50, TC60; FunC FC3; Tolk TL1/TL8; Tact TA1/TA7 |
| 185 | shared | TC6, TC75 |
| 191-194 | shared | TC1 |
| 195-197 | shared | TC11, TC33, TC38, TC57 |
| 203 | shared | TC18, TC35, TC63 |
| 204 | Tact | TA18 |
| 205-208 | shared | TC35, TC61, TC63, TC73, TC75 |
| 209-213 | shared | TC5, TC18, TC35, TC72, TC73 |

## Tolk-specific lines

| Source line(s) | Language | Attack vector(s) / control |
|---|---|---|
| 221-224 | Tolk | TC61, TC75, TL7 |
| 228 | Tolk | TC62 |
| 229-233 | Tolk | TC3, TC23, TC62, TL5, TL6 |
| 237-241 | Tolk | TC42, TC54, TL3, TL6, TL9 |
| 245-251 | Tolk | TC9, TC39, TC42, TC43, TL1-TL4, TL8 |
| 255-259 | Tolk | TC25, TC28, TC30, TC41, TL7 |
| 260-261 | Tolk | TC9, TC33, TC46, TC58, TC76, TL7 |
| 265-269 | Tolk | TC3, TC31, TC56, TL3 |
| 273-275 | Tolk | TC34, TC54, TL9 |
| 276 | Tolk | TL10 |

## FunC-specific lines

| Source line(s) | Language | Attack vector(s) / control |
|---|---|---|
| 282 | FunC | FC1 |
| 283 | FunC | FC2 |
| 284 | FunC | FC7 |
| 285 | FunC | FC6 |
| 286 | FunC | FC4 |
| 287 | FunC | TC15, TC34 |
| 288 | FunC | TC21, TC29, TC56 |
| 289 | FunC | TC35, TC63 |
| 290 | FunC | TC5, TC54, FC12 |
| 291 | FunC | TC28, TC42, TC55, FC6, FC7 |

## Tact-specific lines

| Source line(s) | Language | Attack vector(s) / control |
|---|---|---|
| 297 | Tact | TC61 |
| 298 | Tact | TC62, TA9, TA11, TA12, TA15 |
| 299 | Tact | TC31, TA3, TA12 |
| 300 | Tact | TA15 |
| 301 | Tact | TA2, TA6 |
| 302 | Tact | TC75, TA20 |
| 303 | Tact | TA18 |
| 304-305 | Tact | TC29, TC64, TA19 |
| 306 | Tact | TC61, TC75, TA20 |
| 307 | Tact | TC20, TC29, TC45 |

## Protocol-specific lines

| Source line(s) | Language | Attack vector(s) / control |
|---|---|---|
| 315-318 | shared | TC14, TC18, TC58, TP7-TP10 |
| 319-321 | shared | TC3, TC26, TC37, TC44, TC53, TC56, TC57, TP7-TP10 |
| 322-323 | shared | TC41-TC48, TP7-TP10, TP24-TP26 |
| 327-329 | shared | TC14, TC18, TC58, TP2, TP3 |
| 330-332 | shared | TC1, TC12, TC24, TC37, TC76, TP2-TP6 |
| 336 | shared | TC14, TC22, TC36, TC48, TC53, TC57 |
| 337-339 | shared | TC59, TC60, TC70 |
| 340 | shared | TC65, TC66, TC73 |
| 341 | shared | TC33, TC71 |
| 342-344 | shared | TC12, TC22, TC26, TC53, TC57, TC59, TC70, TC71 |

## Testing, red flags, and evidence controls

| Source line(s) | Language | Attack vector(s) / control |
|---|---|---|
| 350-357 | shared | Control: handler/integration/failure/race/replay/gas/migration/fuzz test matrix; exercises TC3, TC11-TC12, TC20-TC21, TC28, TC31, TC53, TC57, TC61, TC63, TC74 |
| 358-359 | shared | Control: tooling is advisory, manual coverage required |
| 360-361 | shared | Control: evidence classification and high-impact scenario matrix |
| 369 | shared | TC15, TC34 |
| 370 | shared | TC4, TC29 |
| 371-372 | shared | TC3, TC26, TC56 |
| 373 | shared | TC23, TC62 |
| 374-375 | shared | TC18, TC25, TC58 |
| 376 | shared | TC14, TC58, TP7-TP10 |
| 377 | shared | TC14, TC58, TP2-TP3 |
| 378 | shared | TC35, TC63, TC72 |
| 379 | shared | TC25, TC42, TC55, TC75; Tolk TL7; Tact TA20; FunC FC12 |
| 380-381 | Tolk/Tact | TL8, TA2-TA3, TA6-TA7 |
| 382 | shared | TC16, TC74 |
| 383 | shared | TC1 |
| 384 | shared | TC21, TC26, TC29 |
| 385 | shared | TC42; FunC FC6; Tolk TL3/TL6; Tact TA3/TA8 |
| 386 | shared | TC18, TC35, TC62, TC76 |
| 387 | shared | TC61 |
| 393-404 | shared | Control: required audit evidence, trust model, accepted-risk record, and coverage provenance |
| 410-412 | shared | TC21, TC26, TC29, TC56 |
| 413 | shared | TC48, TC53, TC57 |
| 414 | shared | TC18 |
| 415 | Tact | TA18 |
| 416 | shared | TC20, TC21, TC29, TC45 |

## Source-only lines

Lines 422-448 are source links, not attack vectors. They are captured in `references/source-links.md`; the upstream guide itself and Sanbir skill are pinned above and in the comparative map.
