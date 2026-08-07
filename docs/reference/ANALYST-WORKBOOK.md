# Analyst Workbook

Copy these templates into a private working document during the build. Publish
only sanitized sections that support the project's evidence claims.

## 1. Environment inventory

```text
Project owner:
Authorization date:
Authorized scope:
Physical host CPU/RAM/storage:
Hypervisor and version:
Time zone / UTC policy:

VM name:
Role:
OS edition/version/build:
vCPU/RAM/disk:
Isolated IP/MAC:
Temporary NAT present?:
Snapshot names:

Splunk Enterprise version/build:
Universal Forwarder version/build:
Sysmon version:
Sysmon configuration source/revision:
Install package hashes/signatures:
```

## 2. Change record

```text
Change ID: CHG-SOC-LAB-YYYYMMDD-001
UTC start/end:
Operator:
Affected VM/component:
Reason:
Pre-change state:
Exact setting/file changed:
Command or UI path:
Expected result:
Actual result:
Validation performed:
Snapshot/rollback point:
Rollback performed?:
Evidence reference:
Lessons learned:
```

## 3. Telemetry source record

```text
Source ID:
Provider/channel:
Forwarder input stanza:
Destination index:
Sourcetype observed:
Host observed:
Expected event IDs:
Local validation command:
Splunk validation SPL:
First/newest event:
Median/p95 delay:
Required fields present?:
Known gaps:
Status:
```

## 4. Baseline worksheet

```text
Baseline ID:
UTC time window:
Endpoint state (idle/admin/normal apps):
OS/tool/config versions:
Count by index/channel:
Events per minute range:
Median/p95 ingestion delay:
Top normal processes:
Top expected destinations:
Missing fields/channels:
Noise observations:
Tuning decisions:
Evidence reference:
```

## 5. Detection specification

```text
Detection ID and name:
Owner/version/date:
Security question:
Behavior:
Required data source:
Required fields:
SPL and time window:
Threshold and justification:
ATT&CK technique/sub-technique:
Severity and reason:
Expected false positives:
Triage fields:
Positive test IDs:
Negative test IDs:
Known blind spots:
Saved-search/alert settings:
Evidence labels:
Status:
```

## 6. Detection test record

```text
Test-case ID:
Detection ID/version:
Tester:
UTC start/end:
Snapshot confirmed?:
NAT/bridged adapters disconnected?:
Pre-test telemetry health:
Exact authorized action:
Expected local event:
Expected indexed event:
Expected rule result:
Actual result/event references:
Search job/time range:
Negative test and result:
Cleanup action/result:
Pass/fail and reason:
Tuning required?:
Evidence reference:
```

## 7. Alert triage case

```text
Case ID:
Alert/detection ID:
Alert UTC time:
Analyst start/end:
Affected host/user:
Source/destination:
Raw event references:
Telemetry health:
Correlated events/timeline:
Expected or authorized context:

Severity (1-5) and reason:
Confidence (1-5) and reason:
Asset criticality (1-5) and reason:
Exposure (1-5) and reason:
Calculated priority score:

Classification:
Selected playbook:
Containment decision/reason:
Eradication/cleanup:
Recovery validation:
Detection tuning decision:
Lessons learned:
Evidence labels:
Closure statement:
```

## 8. Investigation timeline

| UTC time | Host | User | Source | Event ID | Activity | Evidence | Analyst interpretation |
|---|---|---|---|---:|---|---|---|
| `YYYY-MM-DD HH:MM:SS` | | | | | | | |

Separate observed fact from interpretation. For example, “PowerShell process
created” is an observation; “attacker executed PowerShell” is an interpretation
that requires more evidence.

## 9. Playbook exercise record

```text
Exercise ID:
Playbook ID/version:
Case ID:
Participants:
UTC start/end:
Preparation confirmed:
Detection and analysis steps:
Containment decision:
Eradication action:
Recovery validation:
Escalation decision:
Evidence preserved:
Steps that were unclear:
Metrics:
Lessons learned:
Playbook changes proposed:
```

## 10. Screenshot/evidence record

```text
Evidence label:
Related detection/playbook/case:
UTC capture time:
Source host/tool:
Claim proved:
Query/command/time range visible?:
Sanitization performed:
Information intentionally excluded and why:
Reviewer:
Repository path:
Retained raw source and SHA-256 (private):
```

## 11. Reproduction report

```text
Repository URL:
Commit SHA:
Clone date:
Tester environment:
Address/resource substitutions:
Modules completed:
Acceptance tests passed:
Acceptance tests failed:
Detection/test-case results:
Documentation gaps:
Sanitized evidence links:
Overall reproducibility result:
```

## 12. Daily learning notes

```text
Date/module:
Goal for this session:
Concept learned:
Command/SPL learned:
What I expected:
What actually happened:
Error and root cause:
How I validated the fix:
Screenshot captured:
Question to research:
Next action:
```
