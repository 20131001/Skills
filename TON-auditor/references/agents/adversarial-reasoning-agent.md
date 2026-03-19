# Report Formatting

## Report Path

Save the final report to `assets/findings/{project-name}-ai-audit-report-{timestamp}.md` where:

- `{project-name}` is the repo root basename
- `{timestamp}` is `YYYYMMDD-HHMMSS` at scan time

## Usage

This file defines two output shapes:

1. **Finding blocks** for scanning agents
2. **Full wrapped report** for the orchestrator

Use the correct shape for your role. Do not mix them.

## Finding Block Format

Scanning agents must output **finding blocks only**.

- Do **not** output the full report wrapper.
- Do **not** output `# Security Review`, `## Scope`, `## Findings`, `Findings List`, or the disclaimer.
- Sort findings by confidence, highest first.
- Use placeholder sequential numbering (`1.`, `2.`, `3.`). The orchestrator will re-number after merge.
- Keep `Description` to one short sentence.
- Include a `Fix` block only when the finding is at or above the active confidence threshold.
- If no findings survive, respond exactly: `No findings.`

Example finding block:

````
[95] **1. Missing Bounce Handling on Outbound Transfer**

`jetton-wallet.recv_internal` · Confidence: 95

**Description**
The contract debits local state before sending a bounceable message and does not restore state when the message bounces, which can permanently desynchronize balances.

**Fix**

```diff
- balance -= amount;
+ pending_balance -= amount;
+ handle_bounce_restore(query_id, amount);
```
````

Example below threshold:

````
[60] **2. Partial `msg_value` Validation in Multi-Stage Flow**

`vault.recv_internal` · Confidence: 60

**Description**
The handler appears to under-check downstream gas requirements, so later actions may fail after earlier state changes are already committed.
````

## Full Report Format

The orchestrator must wrap merged findings in the full report format below.

````
# 🔐 Security Review — <ContractName or repo name>

---

## Scope

|                                  |                                                      |
| -------------------------------- | ---------------------------------------------------- |
| **Mode**                         | ALL / default / deep / filename                      |
| **Files reviewed**               | `file1.fc` · `file2.fc`<br>`file3.fc` · `file4.fc`   | <!-- list every file, 3 per line -->
| **Confidence threshold (1-100)** | 75                                                   |

---

## Findings

[95] **1. <Title>**

`ContractName.functionName` · Confidence: 95

**Description**
<The vulnerable pattern and why it is exploitable, in 1 short sentence>

**Fix**

```diff
- vulnerable line(s)
+ fixed line(s)
```
---

[82] **2. <Title>**

`ContractName.functionName` · Confidence: 82

**Description**
<The vulnerable pattern and why it is exploitable, in 1 short sentence>

**Fix**

```diff
- vulnerable line(s)
+ fixed line(s)
```
---

<... all findings ...>

---

Findings List

| # | Confidence | Title |
|---|---|---|
| 1 | [95] | <title> |
| 2 | [82] | <title> |
| | | **Below Confidence Threshold** |
| 3 | [75] | <title> |
| 4 | [60] | <title> |

---

> ⚠️ This review was performed by an AI assistant. AI analysis can never verify the complete absence of vulnerabilities and no guarantee of security is given. Team security reviews, bug bounty programs, and on-chain monitoring are strongly recommended.
````

## Empty Results

- Default empty-result output: `No findings.`
- Only render a full empty report wrapper if the caller explicitly asks for a wrapped empty report.

If a wrapped empty report is required, use this `Findings` section:

````
## Findings

No confirmed findings.
````

## Rules

- Follow the applicable template exactly.
- Use `.fc` examples and FunC/TON terminology, not Solidity examples.
- Sort findings by confidence, highest first.
- Findings below the threshold get a `Description` only and no `Fix` block.
- Preserve confirmed finding wording during merge unless needed to collapse obvious duplicates.
- The orchestrator may merge and re-number findings, but should not re-draft them from scratch.
