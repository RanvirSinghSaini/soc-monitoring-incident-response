# Module 06 — Ingestion Validation and Baseline

## What you will learn

- How to separate service health, source coverage, freshness, volume, and field
  quality.
- Why a baseline must exist before thresholds are tuned.
- How event time and index time reveal collection delay.

## 1. Set a controlled search window

Use **Last 15 minutes** for live validation and **Last 60 minutes** for the
first baseline review. Avoid **All time** after initial onboarding because it
can hide stale data and waste resources.

## 2. Inventory indexed sources

```spl
index IN (windows, sysmon)
| stats count earliest(_time) AS first_event latest(_time) AS newest_event
    by index host source sourcetype
| convert ctime(first_event) ctime(newest_event)
| sort index host source
```

### SPL explanation

- `index IN (...)` restricts the search to project indexes.
- `stats count` measures volume.
- `earliest` and `latest` summarize event-time coverage.
- `by` creates one row per index, host, source, and sourcetype.
- `convert ctime` makes timestamps readable.

Expected result: rows for Security, System, PowerShell Operational, and Sysmon
Operational channels from `WIN-ENDPOINT-01`.

## 3. Check event freshness

```spl
index IN (windows, sysmon) host="WIN-ENDPOINT-01"
| stats latest(_time) AS newest_event by index source
| eval age_seconds=now()-newest_event
| convert ctime(newest_event)
| sort - age_seconds
```

- `now()` is the search-head time.
- Subtracting the newest event time calculates its age.
- A small positive value means data is current.

During active tests, the project target is less than 60 seconds. Idle channels
may legitimately have no new events; generate one known benign event before
judging freshness.

## 4. Measure indexing delay

```spl
index IN (windows, sysmon) host="WIN-ENDPOINT-01"
| eval delay_seconds=_indextime-_time
| stats count avg(delay_seconds) AS average
    perc50(delay_seconds) AS median
    perc95(delay_seconds) AS p95
    max(delay_seconds) AS maximum
    by index sourcetype
| eval average=round(average,2), median=round(median,2),
    p95=round(p95,2), maximum=round(maximum,2)
| sort index sourcetype
```

`_time` is the event time Splunk assigns. `_indextime` is when Splunk indexed
the event. Their difference estimates pipeline delay. Negative or extreme
values usually indicate clock, time-zone, parsing, or backlog problems.

## 5. Validate specific event families

### Authentication

```spl
index=windows EventCode IN (4624,4625)
| stats count by EventCode host
```

`4624` represents a successful Windows logon; `4625` represents a failed
logon. Their presence confirms authentication telemetry, not compromise.

### Process creation

```spl
index IN (windows, sysmon) EventCode IN (4688,1)
| stats count by index EventCode host
```

Windows Security `4688` and Sysmon `1` can both describe process creation but
have different fields and context.

### PowerShell

```spl
index=windows EventCode IN (4103,4104)
| stats count by EventCode host
```

Module logging commonly produces `4103`; script block logging commonly
produces `4104`.

## 6. Inspect field quality

```spl
index=sysmon EventCode=1 host="WIN-ENDPOINT-01"
| fieldsummary
| sort - count
```

`fieldsummary` lists discovered fields, value counts, and coverage. Use it to
confirm actual field names before writing detections. Do not assume another
learner's add-ons produce identical fields.

Inspect representative raw events and record the fields used for:

- User
- Process image
- Command line
- Parent process
- Source and destination address
- Source and destination port
- Hash
- Event record identifier

## 7. Collect a normal baseline

For at least 30 minutes:

1. Leave the endpoint idle for 10 minutes.
2. Perform normal lab administration for 10 minutes.
3. Open and close approved applications for 10 minutes.
4. Do not run the detection tests yet.

Search one-minute volume:

```spl
index IN (windows, sysmon) host="WIN-ENDPOINT-01"
| timechart span=1m count by index
```

`timechart` creates a time series. `span=1m` groups events into one-minute
buckets. Record idle and administrative ranges rather than only an average.

Measure common processes:

```spl
index=sysmon EventCode=1 host="WIN-ENDPOINT-01"
| eval process=coalesce(Image,process_name,NewProcessName)
| stats count by process
| sort - count
| head 20
```

`coalesce` uses the first non-null field, accommodating field-name differences.
`head 20` keeps the twenty most frequent rows.

## 8. Record a baseline sheet

Use the workbook to record:

- Exact time window and time zone.
- Endpoint state: idle or normal administration.
- Count by index/channel.
- Median and p95 delay.
- Expected high-volume sources.
- Missing or poorly extracted fields.
- Any configuration change made after observing the baseline.

## Evidence checkpoint

Create `EVID-04-telemetry-ingestion.png` showing the host, required channels,
event counts, newest event, and time range. A second panel may show delay.

The screenshot proves data availability; it does not prove a detection works.

## Pass criteria

- [ ] All required event channels are represented.
- [ ] Known benign events appear locally and in Splunk.
- [ ] Freshness meets the active-test target.
- [ ] Median and p95 delay are recorded.
- [ ] Required fields are confirmed from actual data.
- [ ] At least 30 minutes of normal activity is documented.

## Troubleshooting decision

| Finding | Likely layer |
|---|---|
| Event absent locally | Audit policy, application logging, or Sysmon config |
| Event local but absent in Splunk | Forwarder input, permissions, or channel name |
| All channels stale | Forwarder service, TCP path, receiver, or clock |
| Data present with missing fields | Sourcetype, XML rendering, add-on, or search field names |
| Extreme delay | Time sync, parsing, forwarder backlog, or resource pressure |

## Rollback

Do not tune detections around broken data. Fix collection, document the change,
restart only the affected component, and repeat the baseline window.
