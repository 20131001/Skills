# TON Auditor V2

A high-recall, evidence-preserving security audit skill for **TON smart contracts written in Tolk, FunC, or Tact**.

Invoke this repository-root skill through `TON-auditor-V2/SKILL.md`. If a separately registered skill is available, check the loaded path before an audit.

Attribution: this fork keeps the packaging and audit workflow lineage from [pashov/skills](https://github.com/pashov/skills), adapted for TON contracts.

Built for:

- **TON developers** who want fast feedback before shipping Tolk, FunC, or Tact changes
- **Security researchers** who need a first pass over handlers, sends, and bounce flows
- **Auditors** who want broad vector coverage before deeper manual review

It is not a substitute for a full audit. It is the fast pass you should run before you trust a contract system.

## Usage

Point the auditing agent at `TON-auditor-V2/SKILL.md`, then supply a
scope such as:

```bash
--deep
contracts/vault.fc contracts/router.tact --deep
--deep --file-output
```

The skill defaults to the current ChatGPT/Codex model. Use `--model <model>` only when you want all audit agents to use a specific model supported by the runtime.

## Coverage

- **176 attack vectors** tuned for TON-specific security review
- **Durable candidate ledger** that preserves findings, leads, rejections, and merges
- **Independent specialist and contract-local passes** for broader discovery
- **Source-derived async lifecycle probes** for callback, bounce, retry, and cleanup behavior
- **Negative-space review** that revisits uncovered handlers and protocol edges

## What It Looks For

- unsafe external and internal message handling
- bounced-message gaps and supply/accounting drift after failed sends
- `accept_message()` misuse and gas draining vectors
- send-mode mistakes that break later logic or strand value
- Jetton sender / wallet validation bugs
- replay / seqno mistakes
- Tolk lazy loading, typed serialization, `BounceMode`, raw-message, and commit hazards
- FunC storage packing/parsing hazards and Tact footguns
- upgrade and code/data replacement mistakes

## Tips

- **Audit recv handlers, bounce handlers, and send sites first.** Those functions usually contain the real TON trust boundaries.
- **Use `--deep` for Jetton, vault, bridge, escrow, and governance systems.** Async inter-contract flows are where the highest-impact TON bugs usually hide.
