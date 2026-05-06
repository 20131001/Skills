# TON Auditor

A parallel TON smart-contract security audit skill for FunC, Tolk, and Tact codebases.

It is built for:

- TON developers who want a security pass before shipping contract changes.
- Auditors who want vector-constrained coverage plus optional free-form adversarial review.
- Teams working with Jettons, NFTs, wallets, multisigs, message cascades, bounce handling, external messages, gas, and serialization-heavy FunC, Tolk, or Tact logic.

This is not a substitute for a formal audit. It is a high-signal review workflow that helps find concrete, attacker-reachable issues quickly.

## Usage

```text
run TON auditor
```

```text
run TON auditor deep
```

```text
run TON auditor contracts/token.fc contracts/router.tolk contracts/minter.tact --file-output
```

## Modes

- Default scans all in-scope `.fc`, `.func`, `.tolk`, and `.tact` files.
- `deep` adds one free-form adversarial reasoning agent per detected language.
- Explicit filenames limit the audit to those files.
- `--file-output` writes the final report under `assets/findings/`.
- The orchestrator builds bundles as `TON shared rules + language-specific hacking agent + shared TON vectors + TEP standard vectors + target-language vectors`.

## Tips

- Target the contracts you are actively changing when you need dense context and fast feedback.
- In mixed-language projects, include every contract in the message cascade so language agents can reason over cross-contract TON flows.
- Use `deep` for release candidates or when message-flow interactions are complex.
- Run more than once on high-value contracts; independent passes can surface different exploit paths.
