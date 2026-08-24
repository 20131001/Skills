# Solana Auditor for Codex

A Codex skill for security reviews of Solana Rust programs.

It is intended for:

- program developers reviewing instruction changes before release;
- security researchers checking handlers, PDAs, CPIs, and state transitions;
- auditors who want a structured first pass over account validation and protocol invariants.

It is not a substitute for a full manual audit.

## Usage in Codex

Invoke the skill explicitly or ask for a Solana security review in natural language:

```text
Use $solana-auditor to audit this repository.
Use $solana-auditor --deep to review this DeFi protocol.
Use $solana-auditor to audit programs/vault/src/lib.rs.
Use $solana-auditor --file-output to audit this repository and save the report.
```

## Review architecture

Codex coordinates eight independent specialist passes and, with `--deep`, one additional protocol pass. Collaboration passes are scheduled in batches according to the available concurrency; no particular model is required.

| Pass | Focus |
| --- | --- |
| Vector scan | Solana attack-vector coverage |
| Math and precision | Arithmetic, rounding, truncation, and decimals |
| Access control | Signers, owners, PDA authority, and initialization |
| Economic security | Oracles, token behavior, CPI trust, and value extraction |
| Execution trace | Post-CPI state, serialization, and remaining accounts |
| Invariants | Conservation laws, coupled state, and round trips |
| Periphery | Utility modules, helpers, and serialization code |
| First principles | Assumption extraction and adversarial violation |
| Protocol (`--deep`) | Lending, AMMs, vaults, staking, bridges, and governance |

## Coverage

- Solana-specific attack vectors
- account validation and authority boundaries
- PDA seeds, bumps, initialization, closing, and reinitialization
- CPI trust and stale-account assumptions
- SPL Token and Token-2022 behavior
- arithmetic, fixed-point precision, and unsafe casts
- oracle, fee, slippage, liquidation, and economic logic
- remaining-account injection and instruction-introspection bypasses
- state invariants and denial-of-service paths

## Output behavior

The report is returned in the Codex response by default. Project files are only modified when `--file-output` is explicitly requested.

Attribution: the audit workflow lineage comes from [pashov/skills](https://github.com/pashov/skills), adapted for Solana programs and Codex.
