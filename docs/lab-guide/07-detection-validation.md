# Module 07 — Detection Validation

## What you will learn

- How to turn detection intent into a repeatable positive and negative test.
- Why an alert is not proof of compromise.
- How to map a rule to MITRE ATT&CK without overstating coverage.

## Safety gate

Before every test:

- Confirm the test target is `WIN-ENDPOINT-01`.
- Disconnect bridged and NAT adapters.
- Create snapshot `04-before-detection-tests`.
- Record the detection ID, tester, time, expected event IDs, and cleanup.
- Keep a Splunk search open for the expected data.

> [!CAUTION]
> These drills intentionally create security-relevant events. They are written
> for the disposable, isolated lab only. Do not run them on work, school,
> customer, shared, or production systems.

## Standard test cycle

Use this process for every detection:

1. Confirm telemetry health.
2. Run one controlled positive test.
3. Find the local raw event.
4. Find the indexed event.
5. Run the detection search.
6. Record whether the alert fired and why.
7. Run one benign negative test.
8. Clean up the test artifact.
9. Repeat until three positive tests pass.
10. Save the final SPL and evidence.

## Test labels

Use IDs such as `TC-DET001-P01` for a positive test and `TC-DET001-N01` for a
negative test. Never describe a test only as “worked”; record expected and
actual results.

## DET-001 — Repeated failed logons

### Detection intent

Identify repeated authentication failures against one or more lab accounts.
MITRE ATT&CK mapping: Password Guessing, `T1110.001`.

### DET-001 positive test

Create a synthetic, non-privileged account through Windows settings or Local
Users and Groups. From an interactive prompt, attempt five incorrect passwords
for that account within five minutes. Do not lock a real administrator account.

Search:

```spl
index=windows EventCode=4625
| eval user=coalesce(TargetUserName,Account_Name,user)
| eval source_address=coalesce(IpAddress,Source_Network_Address,src)
| bin _time span=5m
| stats count values(user) AS targeted_users
    by _time host source_address
| where count >= 5
| sort - count
```

- `bin` places events into five-minute buckets.
- `stats` aggregates failures by time, host, and source.
- `where` keeps only threshold crossings.

### DET-001 negative test

Perform one accidental-style failure followed by a successful logon. The event
should exist, but the five-event threshold should not trigger.

### DET-001 cleanup

Remove only the synthetic account after capturing evidence.

## DET-002 — User or privileged-group change

### DET-002 positive test

PowerShell as Administrator:

```powershell
$LabPassword = Read-Host "Temporary lab-only password" -AsSecureString
New-LocalUser -Name "soclab_demo" -Password $LabPassword -Description "SOC lab DET-002"
Add-LocalGroupMember -Group "Administrators" -Member "soclab_demo"
Remove-LocalGroupMember -Group "Administrators" -Member "soclab_demo"
Remove-LocalUser -Name "soclab_demo"
Remove-Variable LabPassword
```

- `Read-Host -AsSecureString` avoids placing the password in plain text.
- `New-LocalUser` creates the synthetic account.
- `Add-LocalGroupMember` should generate the privileged-group change.
- The removal commands return the endpoint to its prior state.

Expected event families include account creation and local group membership
changes. Confirm actual IDs and fields on your Windows build.

### DET-002 negative test

Create a normal local account without adding it to Administrators. The account
creation rule may alert, but the privileged-group condition must not.

## DET-003 — PowerShell logging and encoded lab marker

### DET-003 positive telemetry test

```powershell
$Marker = "Write-Output 'SOC-LAB DET-003 harmless marker'"
$Encoded = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($Marker))
powershell.exe -NoProfile -EncodedCommand $Encoded
Remove-Variable Marker,Encoded
```

This encodes a harmless `Write-Output` command to exercise logging of an
encoded-command pattern. It does not download, execute, or modify an external
payload.

Search for event `4104`, the marker, and the process command line. Treat an
encoded command as a review signal, not automatic proof of malicious intent.

### DET-003 negative test

```powershell
Write-Output "SOC-LAB routine administration"
```

The normal command should be logged but should not meet the encoded-command
condition.

## DET-004 — Unusual parent-child process chain

### DET-004 positive test

```powershell
cmd.exe /c powershell.exe -NoProfile -Command "Write-Output 'SOC-LAB DET-004'"
```

This creates the harmless chain `cmd.exe → powershell.exe`. Validate Windows
`4688` and/or Sysmon `1`, including parent image and command line.

### DET-004 negative test

Launch PowerShell normally from the Start menu and run the same `Write-Output`.
The child process may be identical, but the parent context differs.

## DET-005 — Rare endpoint connection to the simulator

### DET-005 prepare the simulator

On `KALI-SIM-01`:

```bash
mkdir -p ~/soc-lab-http
printf 'SOC-LAB DET-005 benign content\n' > ~/soc-lab-http/lab.txt
cd ~/soc-lab-http
python3 -m http.server 8080 --bind 192.168.56.30
```

- `mkdir -p` creates the test folder and is repeatable.
- `printf` creates a known harmless text file.
- `python3 -m http.server` starts a simple lab-only HTTP server.
- `--bind` restricts it to the isolated interface.

On Windows:

```powershell
Invoke-WebRequest "http://192.168.56.30:8080/lab.txt" -OutFile "C:\LabEvidence\det005-lab.txt"
Get-Content "C:\LabEvidence\det005-lab.txt"
```

Expected Sysmon event: network connection event ID `3` if enabled in the XML.

### DET-005 negative test

Repeat a documented connection to an already common lab destination. A rarity
rule should treat the known destination differently after tuning.

### DET-005 cleanup

Stop the Python server with `Ctrl+C` and remove only the test text file.

## DET-006 — Service creation

### DET-006 positive test

After snapshot confirmation, open an elevated Command Prompt:

```cmd
sc.exe create SOC-LAB-DET006 binPath= "C:\Windows\System32\cmd.exe /c exit 0" start= demand DisplayName= "SOC Lab DET-006"
sc.exe query SOC-LAB-DET006
sc.exe delete SOC-LAB-DET006
```

`sc.exe create` registers a demand-start test service. Spaces after each `=`
are required by `sc.exe` syntax. The service is not started. `query` records
its state; `delete` removes the registration. Validate System event `7045`.

### DET-006 negative test

Query an existing approved service without creating or changing it. A service
inventory action should not meet a service-creation condition.

## DET-007 — Scheduled task creation

### DET-007 positive test

Choose a time later than the current lab time and delete the task immediately
after the event is collected:

```cmd
schtasks.exe /create /sc once /st 23:59 /tn "SOC-LAB-DET007" /tr "cmd.exe /c echo SOC-LAB-DET007 > C:\LabEvidence\det007.txt" /f
schtasks.exe /query /tn "SOC-LAB-DET007" /v /fo list
schtasks.exe /delete /tn "SOC-LAB-DET007" /f
```

- `/create` registers the task.
- `/sc once` schedules one run.
- `/st` specifies local start time.
- `/tn` gives an unmistakable lab name.
- `/tr` defines a harmless action.
- `/f` suppresses confirmation.

Validate event `4698` when the required audit policy is available. If the
current time is near or past `23:59`, use the next safe future time and record
it. Do not leave the task registered.

### DET-007 negative test

Query an existing task without creating or changing one.

## DET-008 — Security log cleared

> [!WARNING]
> This test changes local evidence. Run it last, only on the disposable Windows
> VM, after exporting the log and confirming the snapshot. Never run it on a
> real endpoint or before preserving needed evidence.

Export first:

```powershell
wevtutil.exe epl Security "C:\LabEvidence\Security-before-DET008.evtx"
Get-FileHash "C:\LabEvidence\Security-before-DET008.evtx" -Algorithm SHA256
```

After recording authorization, clear only the lab Security log:

```powershell
wevtutil.exe cl Security
```

- `epl` exports a channel to an EVTX file.
- `cl` clears the named local channel.
- Windows should create event `1102` indicating the audit log was cleared.

Immediately validate event `1102` locally and in Splunk, then revert the VM to
`04-before-detection-tests`. A negative test is a normal log rotation or no
clear event during the comparison window; do not repeatedly clear the log.

## Detection evidence record

For every rule, complete:

```text
Detection ID:
Test-case ID:
Tester and UTC time:
Pre-test telemetry health:
Exact action:
Expected local event:
Expected SPL result:
Actual result:
Negative test:
False-positive observation:
Cleanup completed:
Screenshot/evidence label:
Pass/fail and reason:
```

## Evidence checkpoint

Create `EVID-05-detection-result.png` only after one rule has three positive
passes and a documented negative test. Show the detection ID, search window,
query, representative fields, and result count.

## Pass criteria

- [ ] Every test is tied to a named detection and test-case ID.
- [ ] Telemetry health is checked before test behavior.
- [ ] Each rule has three positive passes.
- [ ] Each rule has at least one meaningful negative test.
- [ ] Cleanup is confirmed.
- [ ] ATT&CK mapping describes detected behavior, not total prevention.
- [ ] `DET-008` is last and followed by snapshot recovery.

## Official references

- [MITRE ATT&CK Password Guessing](https://attack.mitre.org/techniques/T1110/001/)
- [MITRE ATT&CK PowerShell](https://attack.mitre.org/techniques/T1059/001/)
- [MITRE ATT&CK Clear Windows Event Logs](https://attack.mitre.org/techniques/T1070/001/)
