# Command Reference

This reference explains the commands used by the SOC lab. Read the matching
module before execution. Examples use lab-only addresses and names.

## How to read command examples

| Marker | Meaning |
|---|---|
| `Administrator` | Run in an elevated Windows terminal |
| `root` | Run with `sudo` on Linux |
| `read-only` | Inspects state without intending to change it |
| `state-changing` | Installs, configures, creates, deletes, or clears something |
| `lab-only` | Must remain in the isolated, authorized environment |

Shells interpret spaces, quotes, pipes, redirection, and variables. Copy only a
command written for the shell you are using.

## Git and repository commands

### `git clone URL`

```bash
git clone https://github.com/RanvirSinghSaini/soc-monitoring-incident-response.git
```

Creates a new local directory, downloads repository history and files, creates
the `origin` remote, and checks out the default branch. It does not modify the
remote repository.

### `git status --short --branch`

Displays the current branch, upstream relationship, and compact file changes.
It is read-only.

### `git remote -v`

Lists the fetch and push URLs assigned to each remote. Use it to confirm that
`origin` points to the intended repository before pushing.

### `git diff` and `git diff --cached`

`git diff` shows unstaged changes. `git diff --cached` shows what is staged for
the next commit. `--stat` provides a file/line summary instead of full content.

### `git add PATH...`

Stages selected paths. It changes the Git index but does not create a commit or
upload anything. Use explicit paths in a mixed workspace.

### `git commit -m "message"`

Creates a local commit from staged content. The message should describe the
actual change without claiming unperformed validation.

### `git push origin main`

Uploads local `main` commits to the `main` branch on `origin`. This changes the
public repository and requires authorization.

### `git grep`

```bash
git grep -n -I -E 'password|secret|token|api[_-]?key'
```

Searches tracked text for review terms. `-n` adds line numbers, `-I` skips
binary content, and `-E` enables an extended regular expression. Matches need
human review; no matches do not prove a repository contains no secret.

## Linux network and system commands

### `hostnamectl` — read-only

Displays hostname, kernel, architecture, and operating-system information.

### `ip -br address` — read-only

Lists network interfaces, state, and addresses in brief form. Use it to find
the real interface name before editing Netplan.

### `ip route` and `ip route show default` — read-only

Display routing decisions. A default route sends destinations not covered by a
more specific route to a gateway. The isolated adapter should not add one.

### `sudo netplan try` — root, temporarily state-changing

Applies a candidate Netplan configuration with a confirmation timeout. If the
operator does not confirm, Netplan attempts rollback. Use the VM console so a
network mistake does not permanently lock you out.

### `ping -c 2 ADDRESS` — read-only network test

Sends two ICMP echo requests. Success proves an ICMP path at that moment;
failure does not prove every TCP/UDP service is unavailable.

### `timedatectl status` — read-only

Displays local/UTC time, time zone, and synchronization state.

### `sudo dpkg -i ./PACKAGE.deb` — root, state-changing

Installs a local Debian package. Verify the package source and exact filename
first. Package removal/rollback depends on the installed product and snapshot.

### Splunk CLI

```bash
sudo /opt/splunk/bin/splunk start --accept-license
sudo /opt/splunk/bin/splunk status
sudo /opt/splunk/bin/splunk restart
sudo /opt/splunk/bin/splunk stop
```

- `start` launches Splunk and accepts the displayed license when instructed.
- `status` reports whether Splunk processes are running.
- `restart` stops and starts Splunk to load configuration.
- `stop` performs a controlled shutdown before maintenance or snapshot revert.

The default path can differ by installation method. Do not include passwords in
CLI examples or shell history.

### `ss -lntp` — read-only

Lists listening TCP sockets (`l`), suppresses name resolution (`n`), selects TCP
(`t`), and includes owning processes (`p`, which can require root).

```bash
sudo ss -lntp | grep -E ':8000|:9997|:8089'
```

The pipe sends output into `grep`; `-E` allows alternatives separated by `|`.

### UFW rules — root, state-changing

```bash
sudo ufw allow from 192.168.56.0/24 to any port 9997 proto tcp
sudo ufw status numbered
```

The first rule permits only the lab subnet to reach local TCP `9997`. The
second lists rules with numbers. Firewall changes can break access; document
the old state and use the VM console.

### Lab HTTP server — lab-only

```bash
mkdir -p ~/soc-lab-http
printf 'SOC-LAB benign content\n' > ~/soc-lab-http/lab.txt
cd ~/soc-lab-http
python3 -m http.server 8080 --bind 192.168.56.30
```

- `mkdir -p` creates the directory if absent.
- `printf ... > file` writes the harmless marker, replacing that exact file.
- `cd` changes the current directory.
- Python serves that directory over lab TCP `8080` bound to the isolated IP.
- `Ctrl+C` stops the server.

## Windows inspection commands

### System identity — read-only

```powershell
hostname
Get-ComputerInfo | Select-Object WindowsProductName,WindowsVersion,OsBuildNumber
Get-Date
Get-TimeZone
```

These record host identity, Windows build, current time, and time zone.
`Select-Object` limits properties for readable evidence.

### Network inspection — read-only

```powershell
Get-NetIPConfiguration
Get-NetAdapter | Format-Table Name,Status,MacAddress,LinkSpeed
Get-NetRoute -DestinationPrefix "0.0.0.0/0"
```

They show IP configuration, adapter identity/state, and IPv4 default routes.

### `Test-Connection` — read-only network test

```powershell
Test-Connection 192.168.56.10 -Count 2
```

Sends two ICMP requests. It is the PowerShell counterpart of a limited ping.

### `Test-NetConnection` — read-only network test

```powershell
Test-NetConnection 192.168.56.10 -Port 9997
```

Attempts a TCP connection and reports `TcpTestSucceeded`. It does not prove
that received application data is valid.

### `Get-NetTCPConnection` — read-only

Filters current TCP sessions by remote address and port. No row can mean the
forwarder is disconnected, between reconnect attempts, or not sending yet.

### Hash and signature — read-only

```powershell
Get-FileHash "C:\path\file.exe" -Algorithm SHA256
Get-AuthenticodeSignature "C:\path\file.exe"
```

The hash identifies exact bytes; the signature checks publisher signing state.
Neither command executes the file.

### File and directory commands

```powershell
New-Item -Path "C:\LabEvidence" -ItemType Directory -Force
Get-Content "C:\path\file.log" -Tail 100
```

`New-Item` creates/reuses the exact directory. `Get-Content -Tail` reads only
the newest lines. `>` redirection, used elsewhere, replaces a target file and
must be reviewed before execution.

## Windows audit and policy commands

### Audit policy backup/get — Administrator

```powershell
auditpol /backup /file:"C:\LabEvidence\audit-policy-before.csv"
auditpol /get /category:*
auditpol /list /subcategory:*
```

These export current policy, display effective settings, and list valid local
subcategory names. They do not clear logs.

### Audit policy set — Administrator, state-changing

```powershell
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
```

Enables success and failure auditing for the named subcategory. Use only names
reported by the local system. More auditing can increase event volume.

### Audit policy restore — Administrator, state-changing

```powershell
auditpol /restore /file:"C:\LabEvidence\audit-policy-before.csv"
```

Restores the exported policy. Confirm the file belongs to this endpoint and
lab session.

### `gpupdate /force` — Administrator, state-changing

Reapplies Group Policy. This can overwrite local settings when a machine is
domain-managed; this lab endpoint should be isolated and documented.

## Windows event commands

### `Get-WinEvent` — read-only

```powershell
Get-WinEvent -LogName "Microsoft-Windows-PowerShell/Operational" -MaxEvents 5
```

Reads recent events from the exact channel. A `FilterHashtable` can filter by
log, event ID, time, or provider before results enter the pipeline.

### `Get-WinEvent -ListLog` — read-only

Displays channel configuration, enabled state, record count, and size.

### `wevtutil epl` — Administrator, creates export

```powershell
wevtutil.exe epl Security "C:\LabEvidence\Security-before.evtx"
```

Exports the Security channel without clearing it. Hash the file and retain it
outside Git when it contains sensitive data.

### `wevtutil cl` — Administrator, destructive to local evidence

```powershell
wevtutil.exe cl Security
```

Clears the named local log. This is reserved for `DET-008`, after export and
snapshot confirmation. Revert the lab snapshot immediately after validation.

## Sysmon commands — Administrator

```powershell
.\Sysmon64.exe -accepteula -i "C:\Sysmon\sysmonconfig.xml"
.\Sysmon64.exe -c
.\Sysmon64.exe -c "C:\Sysmon\sysmonconfig.xml"
Get-Service Sysmon*
```

- `-i` installs Sysmon with the XML configuration.
- `-c` alone displays active configuration.
- `-c XML` changes the active configuration.
- `Get-Service` detects the Sysmon service and state.

Do not install standalone Sysmon alongside a conflicting existing
implementation. Configuration changes affect event volume and must be tested.

## Universal Forwarder commands — Administrator

```powershell
$SplunkUF = "$env:ProgramFiles\SplunkUniversalForwarder\bin\splunk.exe"
& $SplunkUF btool inputs list --debug
& $SplunkUF btool outputs list --debug
Restart-Service SplunkForwarder
Get-Service SplunkForwarder
```

- `$env:ProgramFiles` expands the standard installation root.
- `btool` displays merged effective configuration and source files.
- Restart reloads configuration; Get reports current service state.

## Safe test-generation commands — lab-only

### Synthetic local user

```powershell
$LabPassword = Read-Host "Temporary lab-only password" -AsSecureString
New-LocalUser -Name "soclab_demo" -Password $LabPassword
Add-LocalGroupMember -Group "Administrators" -Member "soclab_demo"
Remove-LocalGroupMember -Group "Administrators" -Member "soclab_demo"
Remove-LocalUser -Name "soclab_demo"
Remove-Variable LabPassword
```

The password prompt prevents plain-text storage. The commands create, add,
remove, and delete only the synthetic lab account. Confirm the exact account
name before cleanup.

### Benign encoded PowerShell marker

```powershell
$Marker = "Write-Output 'SOC-LAB harmless marker'"
$Encoded = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($Marker))
powershell.exe -NoProfile -EncodedCommand $Encoded
```

This encodes and runs only the visible harmless string. Encoding is not
encryption and is not automatically malicious.

### `Start-Process` and `Start-Sleep`

Launch a named process and pause the script for a specified time. They are used
to allow Windows event writing before a validation query.

### `Invoke-WebRequest` — lab-only network request

Downloads the known marker from the Kali lab server to a fixed evidence path.
Use only the isolated address and review `-OutFile` before execution.

### `sc.exe` service test — Administrator, state-changing

Creates, queries, and deletes the specifically named demand-start lab service.
The spaces after `binPath=`, `start=`, and `DisplayName=` are required.

### `schtasks.exe` task test — Administrator, state-changing

Creates, queries, and deletes the specifically named one-time lab task. Choose
a safe future time and remove the task before it runs.

## SPL command reference

| SPL element | Purpose |
|---|---|
| `index=windows` | Restrict events to an index |
| `field IN (a,b)` | Match any listed value |
| `host="name"` | Restrict to one reported host |
| `stats` | Aggregate events into measures and groups |
| `earliest()` / `latest()` | Find minimum/maximum event time in a group |
| `eval` | Create or transform fields |
| `coalesce(a,b,c)` | Return the first non-null value |
| `bin _time span=5m` | Bucket time into five-minute intervals |
| `where` | Filter using an evaluated expression |
| `search` | Filter events/results using search syntax |
| `table` | Display selected fields in order |
| `sort` | Order rows; `sort 0` avoids a default result limit |
| `convert ctime(field)` | Convert epoch seconds into readable time |
| `timechart span=1m` | Aggregate as a one-minute time series |
| `fieldsummary` | Summarize discovered fields and coverage |
| `head 20` | Keep the first twenty ordered results |
| `rest` | Query an authorized Splunk REST endpoint inside SPL |

SPL searches are starting points. Validate field names, time ranges, event
volume, and false positives against the actual lab data.
