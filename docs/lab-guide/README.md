# SOC Lab Learning Path

This folder turns the project architecture in the main README into a practical,
beginner-friendly course. Complete the modules in numerical order. Do not skip
a validation gate: later detections are meaningful only when the earlier
telemetry checks pass.

## How to use the guide

For every module:

1. Read **What you will learn** before changing the lab.
2. Take the named VMware snapshot.
3. Run only the commands for the operating system shown.
4. Compare the actual result with **Expected result**.
5. Record your notes in the [analyst workbook](../reference/ANALYST-WORKBOOK.md).
6. Capture the required screenshot only after the pass criteria are met.
7. Stop and troubleshoot when a validation gate fails.

> [!IMPORTANT]
> Commands marked **Administrator** or **root** change system configuration.
> Run them only inside the isolated lab and read the explanation before pressing
> Enter. Replace example values only with values from your own lab worksheet.

## Module map

| Order | Module | Outcome | Evidence checkpoint |
|---:|---|---|---|
| `01` | [Safety and planning](01-safety-and-planning.md) | Authorized scope, inventory, versions, rollback plan | Planning worksheet |
| `02` | [VMware and network](02-vmware-and-network.md) | Three isolated VMs with static addressing | `EVID-01`, `EVID-02` |
| `03` | [Splunk server](03-splunk-server.md) | Healthy Splunk instance, indexes, receiver | Part of `EVID-03` |
| `04` | [Windows telemetry and Sysmon](04-windows-telemetry-and-sysmon.md) | Auditing, PowerShell logging, Sysmon events | Part of `EVID-04` |
| `05` | [Universal Forwarder](05-universal-forwarder.md) | Windows events forwarded on TCP 9997 | `EVID-03` |
| `06` | [Ingestion and baseline](06-ingestion-and-baseline.md) | Searchable telemetry and measured normal activity | `EVID-04` |
| `07` | [Detection validation](07-detection-validation.md) | Repeatable positive and negative tests | `EVID-05` |
| `08` | [Triage, response, and evidence](08-triage-response-and-evidence.md) | Completed case, playbook, timeline, metrics | `EVID-06`–`EVID-10` |
| `09` | [Reproduce and publish](09-reproduce-and-publish.md) | Sanitized repository that another learner can clone | Clean-clone record |

## Supporting references

- [Command reference](../reference/COMMAND-REFERENCE.md) explains what each
  command does, privileges, expected result, and rollback.
- [Learning notes](../reference/LEARNING-NOTES.md) explains the SOC concepts
  behind the work.
- [Analyst workbook](../reference/ANALYST-WORKBOOK.md) provides reusable notes,
  test, case, and evidence templates.
- [Troubleshooting matrix](../reference/TROUBLESHOOTING-MATRIX.md) maps symptoms
  to checks and likely causes.
- [Evidence guide](../evidence/README.md) defines screenshot labels and public
  sanitization rules.

## Progress tracker

- [ ] Module 01 — safety and planning
- [ ] Module 02 — VMware and network
- [ ] Module 03 — Splunk server
- [ ] Module 04 — Windows telemetry and Sysmon
- [ ] Module 05 — Universal Forwarder
- [ ] Module 06 — ingestion and baseline
- [ ] Module 07 — detection validation
- [ ] Module 08 — triage, response, and evidence
- [ ] Module 09 — reproduce and publish

The repository owner should check these boxes only when the associated evidence
exists. A reader reproducing the project should keep their own copy of this
tracker rather than changing the upstream project status.
