# Module 05 — Splunk Universal Forwarder

## What you will learn

- How a forwarder input differs from its output destination.
- Where Windows forwarder configuration and diagnostic logs are stored.
- How to prove each layer from service to TCP connection to indexed event.

## Data path

```mermaid
flowchart LR
    Logs["Windows event channels"] --> Inputs["inputs.conf\nwhat to collect"]
    Inputs --> UF["Splunk Universal Forwarder"]
    UF --> Outputs["outputs.conf\nwhere to send"]
    Outputs -->|"TCP 9997"| Receiver["Splunk receiver"]
    Receiver --> Indexes["windows + sysmon indexes"]
```

## 1. Preflight the receiver

From `WIN-ENDPOINT-01`:

```powershell
Test-NetConnection 192.168.56.10 -Port 9997
```

- `Test-NetConnection` performs a network diagnostic.
- The IP is the Splunk server's host-only address.
- `-Port 9997` tests a TCP connection, not only ping.

Expected result: `TcpTestSucceeded : True`. If false, stop. Check the Splunk
listener, host firewall, VMware network, and destination address before
installing the forwarder.

## 2. Install the official Windows forwarder

Download the supported Universal Forwarder MSI from Splunk. Record its version
and verify its signature. Use the interactive installer for the first build so
every choice is visible.

Recommended learning-lab choices:

1. Install to the default `C:\Program Files\SplunkUniversalForwarder` path.
2. Create a unique local forwarder administration credential.
3. Set the receiving indexer to `192.168.56.10` and port `9997` if prompted.
4. Do not configure a deployment server in this single-forwarder phase.
5. Complete the install and confirm the `SplunkForwarder` service exists.

The forwarder administrator credential is local to the forwarder management
interface. It is not the same thing as the TCP receiver destination.

## 3. Create `inputs.conf`

Open Notepad as Administrator and create:

```text
C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
```

Use:

```ini
[WinEventLog://Security]
disabled = 0
index = windows
renderXml = true

[WinEventLog://System]
disabled = 0
index = windows
renderXml = true

[WinEventLog://Microsoft-Windows-PowerShell/Operational]
disabled = 0
index = windows
renderXml = true

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = 0
index = sysmon
renderXml = true
```

### Input stanza explanation

- `[WinEventLog://channel]` identifies a Windows Event Log channel.
- `disabled = 0` enables collection.
- `index` selects the destination storage index.
- `renderXml = true` preserves structured XML that can improve field access.

Channel names must match Windows exactly. Confirm them with:

```powershell
Get-WinEvent -ListLog * | Where-Object LogName -Match "PowerShell|Sysmon" |
    Select-Object LogName,IsEnabled,RecordCount
```

## 4. Create `outputs.conf`

Create:

```text
C:\Program Files\SplunkUniversalForwarder\etc\system\local\outputs.conf
```

Use:

```ini
[tcpout]
defaultGroup = soc_lab_indexer

[tcpout:soc_lab_indexer]
server = 192.168.56.10:9997
```

### Output stanza explanation

- `[tcpout]` contains global forwarding settings.
- `defaultGroup` selects the named receiver group used by default.
- `[tcpout:soc_lab_indexer]` defines that group.
- `server` gives the receiver address and port.

The group name is a label; it is not a DNS name. It must not contain spaces or
colons.

## 5. Validate merged configuration

PowerShell as Administrator:

```powershell
$SplunkUF = "$env:ProgramFiles\SplunkUniversalForwarder\bin\splunk.exe"
& $SplunkUF btool inputs list --debug
& $SplunkUF btool outputs list --debug
```

- `$SplunkUF` stores the executable path once.
- `&` invokes the executable stored in a variable.
- `btool` shows the configuration Splunk actually merged from all files.
- `--debug` includes the source file for every setting.

Search the output for the four event-log stanzas, `soc_lab_indexer`, and
`192.168.56.10:9997`. If another file overrides the setting, `--debug` shows
which file won.

## 6. Restart and inspect the service

```powershell
Restart-Service SplunkForwarder
Get-Service SplunkForwarder
```

`Restart-Service` reloads configuration. `Get-Service` should show `Running`.

Review only the newest diagnostic lines:

```powershell
Get-Content "$env:ProgramFiles\SplunkUniversalForwarder\var\log\splunk\splunkd.log" -Tail 100
```

`Get-Content -Tail 100` reads the final 100 lines instead of dumping the entire
log. Look for connection, authentication, parsing, permission, or channel-name
errors. Do not publish the raw log without reviewing it.

## 7. Confirm the TCP session

```powershell
Test-NetConnection 192.168.56.10 -Port 9997
Get-NetTCPConnection -RemoteAddress 192.168.56.10 -RemotePort 9997 -ErrorAction SilentlyContinue
```

The first command proves the port is reachable. The second looks for an active
Windows TCP connection to the receiver. `SilentlyContinue` avoids a red error
when no matching connection currently exists; absence still needs diagnosis.

## 8. Confirm the host in Splunk

In Search & Reporting, use **All time** only for the first connection check:

```spl
index IN (windows, sysmon) host="WIN-ENDPOINT-01"
| stats count latest(_time) AS newest by index sourcetype
| convert ctime(newest)
| sort index sourcetype
```

- The base search limits data to the expected indexes and host.
- `stats` counts events and finds the newest event per index/sourcetype.
- `convert ctime` makes epoch time readable.
- `sort` produces a stable evidence table.

If hostname casing differs, first discover the reported value:

```spl
index IN (windows, sysmon)
| stats count by host
| sort - count
```

## Evidence checkpoint

Create `EVID-03-forwarder-health.png` showing:

- Splunk receiver `9997` enabled.
- `SplunkForwarder` running.
- Successful TCP test.
- Splunk search showing the endpoint and newest event.

Combine evidence only when it remains readable. Otherwise keep detailed images
privately and publish a sanitized composite or representative result.

## Pass criteria

- [ ] TCP `9997` succeeds before and after forwarder installation.
- [ ] `btool` shows the intended stanzas and source files.
- [ ] `SplunkForwarder` is running.
- [ ] `splunkd.log` has no unresolved connection or channel error.
- [ ] Splunk shows recent events from `WIN-ENDPOINT-01`.

## Rollback

Restore the previous `.conf` file versions, then restart the service. If the
forwarder install itself is unusable, revert `03-logging-baseline` and repeat
the module. Do not delete raw logs until the failure is documented.

## Official references

- [Splunk: configure forwarding with `outputs.conf`](https://help.splunk.com/en/data-management/forward-data/universal-forwarder-manual/9.1/forward-data/configure-forwarding-with-outputs.conf)
- [Splunk: install a Windows Universal Forwarder](https://help.splunk.com/en/data-management/forward-data/universal-forwarder-manual/10.4/install-the-universal-forwarder/install-a-windows-universal-forwarder)
