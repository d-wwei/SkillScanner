# Report Templates

Output format templates for scan reports. Load when generating final output.

---

## Text Report Format

```
SECURITY SCAN REPORT
====================
Target: <filename, URL, or input description>
Scan Date: <ISO 8601 timestamp>
Scanner: security-scan v1.0.0
Verdict: <CLEAN | LOW RISK | SUSPICIOUS | HIGH RISK | CRITICAL>
Risk Score: <0-100>

--- FINDINGS (<count>) ---

[#1] <SEVERITY> -- <Category ID>: <Category Name>
  OWASP: <LLM0X>
  Location: Line <N>, chars <M-K>
  Evidence: "<matched text, max 120 chars>"
  Context: <true positive | false positive | needs review>
  Reason: <why this is flagged>

[#2] ...

--- COMPOSITE ANALYSIS ---

<If multiple findings interact, describe the attack chain>

--- SUMMARY ---

Categories hit: <list of Txx codes>
OWASP references: <list of LLM0X codes>
Critical findings: <count>
High findings: <count>
Medium findings: <count>
Low findings: <count>
False positives filtered: <count>

--- REMEDIATION ---

1. [Finding #N]: <what to fix and how>
2. ...

--- RECOMMENDATION ---

<SAFE TO PROCESS | REVIEW REQUIRED | DO NOT PROCESS>
<One-sentence justification>
```

---

## JSON Report Format

For programmatic consumption:

```json
{
  "target": "<scan target>",
  "scan_date": "<ISO 8601>",
  "scanner_version": "1.0.0",
  "verdict": "CLEAN|LOW_RISK|SUSPICIOUS|HIGH_RISK|CRITICAL",
  "risk_score": 0,
  "findings": [
    {
      "id": 1,
      "severity": "CRITICAL|HIGH|MEDIUM|LOW|INFO",
      "category": "T1",
      "category_name": "Direct Injection",
      "owasp": "LLM01",
      "location": {"line": 15, "char_start": 0, "char_end": 35},
      "evidence": "Ignore previous instructions and...",
      "context_judgment": "true_positive|false_positive|needs_review",
      "reason": "Explicit instruction override attempt"
    }
  ],
  "composite_analysis": "<attack chain description if applicable>",
  "summary": {
    "categories_hit": ["T1", "T5"],
    "owasp_refs": ["LLM01"],
    "critical": 1, "high": 0, "medium": 0, "low": 0,
    "false_positives_filtered": 2
  },
  "remediation": [
    {"finding_id": 1, "advice": "Remove the direct injection pattern at line 15"}
  ],
  "recommendation": "DO_NOT_PROCESS"
}
```

---

## Self-Test

Self-test payloads are in `references/self-test.md` (loaded only by the `self-test` sub-command). See that file for 4-tier test suite.
