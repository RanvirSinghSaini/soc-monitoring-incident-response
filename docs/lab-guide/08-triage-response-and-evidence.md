# Module 08 — Triage, Response, and Evidence

## What you will learn

- How to move from an alert to a defensible classification.
- How to calculate an explainable priority score.
- How to build a timeline, execute a playbook, and close a case.

## Alert is not incident

An **event** is a recorded observation. A **detection** is logic that selects or
aggregates events. An **alert** is a notification created from that logic. An
**incident** is an analyst-confirmed situation requiring coordinated response.

The analyst must validate each transition.

## 1. Open a case

Assign a case ID:

```text
SOC-LAB-YYYYMMDD-001
```

Record:

- Alert and analyst-start timestamps in UTC.
- Detection ID and saved-search version.
- Affected host, user, source, and destination.
- Search time range and event references.
- Initial severity and reason.

Use synthetic usernames and lab-only identifiers in public evidence.

## 2. Validate telemetry health

```spl
index IN (windows, sysmon) host="WIN-ENDPOINT-01"
| stats latest(_time) AS newest_event count by index sourcetype
| eval age_seconds=now()-newest_event
| convert ctime(newest_event)
| sort - age_seconds
```

If required telemetry is stale or missing, classify the result as a data-quality
issue until evidence supports another conclusion. Do not fill gaps with guesses.

## 3. Inspect raw events

Start with the event that caused the alert. Record:

- Event time and index time.
- Host and user.
- Event ID and provider.
- Process image, command line, and parent when relevant.
- Source/destination address and port.
- Hash or event record ID.
- The exact field that met the detection condition.

Use a narrow time range around the alert. Expand only when needed.

## 4. Correlate adjacent activity

Example timeline search:

```spl
index IN (windows, sysmon) host="WIN-ENDPOINT-01"
earliest=-10m@m latest=now
| eval activity=case(
    EventCode=4625,"Failed logon",
    EventCode=4624,"Successful logon",
    EventCode=4688 OR EventCode=1,"Process creation",
    EventCode=4104,"PowerShell script block",
    EventCode=3,"Network connection",
    EventCode=7045,"Service creation",
    EventCode=4698,"Scheduled task creation",
    EventCode=1102,"Security log cleared",
    true(),"Other"
  )
| table _time index host user EventCode activity Image ParentImage CommandLine
    SourceIp DestinationIp DestinationPort
| sort 0 _time
```

### SPL explanation

- `earliest` and `latest` define the investigation window.
- `case` assigns a readable label based on the first true condition.
- `true(),"Other"` catches events that do not match an earlier condition.
- `sort 0 _time` keeps all rows in chronological order.

Adjust field names to the actual sourcetypes found in Module 06.

## 5. Classify expected versus suspicious behavior

Ask:

1. Was the activity authorized in the test worksheet?
2. Does the account normally perform this action?
3. Does the parent-child process relationship make sense?
4. Is the destination expected and lab-owned?
5. Do raw events support the alert fields?
6. Is there a successful action following repeated failures?
7. Does adjacent activity raise or lower confidence?

Classification choices:

- **True positive:** rule correctly selected the behavior of interest.
- **Benign true positive:** behavior occurred and the rule worked, but activity
  was authorized or harmless.
- **False positive:** rule logic selected activity outside its intended
  behavior.
- **Data-quality issue:** missing, stale, duplicated, or incorrectly parsed
  data prevents a reliable decision.

A controlled detection drill is commonly a benign true positive.

## 6. Calculate the priority score

Score each factor from 1 to 5:

```text
weighted_score =
  severity × 0.35 +
  confidence × 0.30 +
  asset_criticality × 0.20 +
  exposure × 0.15

priority_score = round((weighted_score / 5) × 100)
```

Document each factor:

- **Severity:** potential effect if the behavior were real.
- **Confidence:** strength and completeness of supporting evidence.
- **Asset criticality:** business importance represented by the lab asset.
- **Exposure:** external reachability and blast radius.

The score supports judgment; it does not replace it.

## 7. Select the playbook

| Playbook | Example triggers | Lab action |
|---|---|---|
| `PB-001` Identity/brute force | `DET-001`, `DET-002` | Validate source and follow-on success; disable only synthetic account |
| `PB-002` PowerShell/process | `DET-003`, `DET-004` | Review parent, command, script block, hash, and connection context |
| `PB-003` Phishing triage | Synthetic email case | Review sender, URL, attachment, identity and endpoint context |
| `PB-004` Possible exfiltration | `DET-005` plus volume/context | Disconnect lab NIC if justified; preserve logs |
| `PB-005` Persistence/privilege | `DET-002`, `DET-006`–`008` | Preserve artifact details, remove test artifact, restore snapshot |

Every playbook must record preparation, detection, analysis, containment,
eradication, recovery, post-incident review, and escalation criteria.

## 8. Execute containment proportionally

For a lab case, containment may be:

- Stop the harmless test process.
- Disable only the synthetic test account.
- Disconnect only the VM's lab adapter.
- Stop the temporary HTTP server.
- Remove the lab service/task after evidence capture.
- Revert the validated snapshot after the log-clear test.

Do not treat “isolate the machine” as an automatic response. Record the
evidence and risk that justified the action.

## 9. Preserve and label evidence

For each public screenshot:

1. Capture the smallest view that proves the claim.
2. Keep the search, time range, host, detection ID, and result visible.
3. Redact credentials, tokens, personal data, unrelated tabs, and license data.
4. Do not crop or alter information that changes the conclusion.
5. Use the exact `EVID-XX` filename from the evidence register.
6. Add a caption explaining what the screenshot proves.

Required case evidence:

- `EVID-06-alert-triage.png`
- `EVID-07-investigation-timeline.png`
- `EVID-08-dashboard-coverage.png`
- `EVID-09-playbook-execution.png`
- `EVID-10-case-closure.png`

## 10. Close the case

Closure requires:

- Final classification and supporting evidence.
- Priority factor values and reasoning.
- Timeline.
- Containment and cleanup result.
- Detection tuning decision.
- Lessons learned.
- Remaining risk or follow-up.
- Evidence labels.

Example concise closure statement:

```text
Benign true positive. DET-003 correctly identified the authorized encoded
PowerShell marker on WIN-ENDPOINT-01. Script-block and Sysmon process events
matched the test worksheet. No external destination or follow-on persistence
was observed. Test variables were removed and the endpoint returned to the
04-before-detection-tests snapshot. Rule retained; lab-marker suppression was
not added because it would weaken the repeatable test.
```

## 11. Calculate project metrics

Track:

- Event-channel availability.
- Median and p95 ingestion delay.
- Positive-test pass rate per rule.
- Negative-test documentation rate.
- Alert-to-triage start time.
- Triage duration.
- False-positive and data-quality counts.
- Playbooks exercised.
- Evidence completeness.

## Pass criteria

- [ ] Case has a unique ID and complete required fields.
- [ ] Telemetry health was checked before classification.
- [ ] Raw events and adjacent activity support the decision.
- [ ] Priority score factors are visible and justified.
- [ ] Playbook actions are proportional and recorded.
- [ ] Test artifacts are cleaned up.
- [ ] Public evidence is sanitized and labeled.
- [ ] Closure includes tuning and lessons learned.

## Common analyst mistakes

- Calling every triggered alert an incident.
- Assigning ATT&CK coverage without a working data source.
- Treating an IP address or encoded command as malicious by itself.
- Ignoring missing telemetry.
- Using screenshots without a query or time range.
- Tuning away the positive test instead of improving context.
