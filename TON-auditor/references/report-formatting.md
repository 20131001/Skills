# Report Formatting

Use this file as the formatting contract for final audit output.

## Inputs

- the audited scope
- the merged confirmed findings
- the confidence threshold from `judging.md`

## Goal

Produce one consistent audit report shape so findings from different agents can be merged without reformatting drift.

## Output Contract

- print one terminal-ready final report, or
- write the same report shape to the findings directory when `--file-output` is explicitly enabled

## Report Path

Save the report to:

`assets/findings/{project-name}-ai-audit-report-{timestamp}.md`

Where:

- `{project-name}` is the repo root basename
- `{timestamp}` is `YYYYMMDD-HHMMSS` at scan time

## Output Template

````
# 🔐 Security Review — <ContractName or repo name>

---

## Scope

|                                  |                                                        |
| -------------------------------- | ------------------------------------------------------ |
| **Mode**                         | ALL / default / deep / filename                        |
| **Files reviewed**               | `file1.fc` · `file2.fc`<br>`file3.fc` · `file4.fc`     | <!-- list every file, 3 per line -->
| **Confidence threshold (1-100)** | N                                                      |

---

## Findings

[95] **1. <Title>**

`ContractName.functionName` · Confidence: 95

**Description**
<The vulnerable code pattern and why it is exploitable, in 1 short sentence>

**Fix**

```diff
- vulnerable line(s)
+ fixed line(s)
```
---

[82] **2. <Title>**

`ContractName.functionName` · Confidence: 82

**Description**
<The vulnerable code pattern and why it is exploitable, in 1 short sentence>

**Fix**

```diff
- vulnerable line(s)
+ fixed line(s)
```
---

< ... all findings >

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

## Rules

- Follow the template above exactly.
- Sort findings by confidence, highest first.
- Findings below the threshold must still appear in the report and in the findings list; they get a `Description` but no `Fix` block.
- Draft findings directly in report format.
- Keep FunC / TON terminology.
