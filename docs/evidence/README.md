# Evidence Guide

This folder will contain the sanitized proof used to validate the SOC Monitoring & Incident Response lab.

## Evidence workflow

1. Run only the documented authorized test case.
2. Record the detection ID, test-case ID, timestamp, host, and analyst.
3. Preserve raw logs outside Git when they contain sensitive or high-volume data.
4. Capture the smallest screenshot that proves the result.
5. Sanitize the screenshot without changing the security conclusion.
6. Save it under `docs/evidence/images/` with the required `EVID-XX` filename.
7. Add a caption to the main README explaining what the image proves.
8. Complete a second-person or self-review before publishing.

## Evidence record template

```text
Evidence label:
Detection / playbook ID:
Test-case ID:
Capture timestamp (UTC):
Source host:
Analyst:
Purpose:
Expected result:
Actual result:
Sanitization performed:
Related raw-log location:
SHA-256 of retained source file, if applicable:
Conclusion:
```

## Required image labels

| Label | Filename |
|---|---|
| `EVID-01` | `EVID-01-vm-inventory.png` |
| `EVID-02` | `EVID-02-host-only-network.png` |
| `EVID-03` | `EVID-03-forwarder-health.png` |
| `EVID-04` | `EVID-04-telemetry-ingestion.png` |
| `EVID-05` | `EVID-05-detection-result.png` |
| `EVID-06` | `EVID-06-alert-triage.png` |
| `EVID-07` | `EVID-07-investigation-timeline.png` |
| `EVID-08` | `EVID-08-dashboard-coverage.png` |
| `EVID-09` | `EVID-09-playbook-execution.png` |
| `EVID-10` | `EVID-10-case-closure.png` |

Never store passwords, tokens, private keys, real victim information, restricted datasets, or unreviewed raw logs in this folder.
