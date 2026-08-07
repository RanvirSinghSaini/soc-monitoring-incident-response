# Module 04 — Windows Telemetry and Sysmon

## What you will learn

- How Windows audit policy determines which security events exist.
- Why PowerShell, process-creation, and Sysmon logging are complementary.
- How to validate telemetry locally before blaming the SIEM.

## Important concepts

Windows logging has three separate layers in this project:

1. **Audit policy** decides whether selected Windows security activity is
   recorded.
2. **Event channels** store events locally, such as `Security` or Sysmon's
   `Operational` channel.
3. **The Universal Forwarder** reads selected channels and sends copies to
   Splunk.

If an event does not exist locally, the forwarder cannot create it.

## 1. Confirm endpoint identity and time

Open PowerShell as Administrator:

```powershell
hostname
Get-ComputerInfo | Select-Object WindowsProductName,WindowsVersion,OsBuildNumber
Get-Date
Get-TimeZone
```

- `hostname` returns the endpoint name expected in Splunk.
- `Get-ComputerInfo` records edition, version, and build.
- `Get-Date` and `Get-TimeZone` provide the time context for later correlation.

Rename the VM to `WIN-ENDPOINT-01` before continuing if required, then restart
and update the worksheet.

## 2. Back up the current audit policy

```powershell
auditpol /backup /file:"C:\LabEvidence\audit-policy-before.csv"
auditpol /get /category:*
```

- `/backup` exports the effective policy so it can be restored.
- `/file:` sets the export path.
- `/get /category:*` displays every category and subcategory.

Audit subcategory display names are localized. Confirm names with
`auditpol /list /subcategory:*` before copying commands on a non-English build.

## 3. Enable required audit subcategories

Run in an elevated Command Prompt or PowerShell:

```powershell
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Process Creation" /success:enable /failure:disable
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable
auditpol /set /subcategory:"Security Group Management" /success:enable /failure:enable
auditpol /set /subcategory:"Other Object Access Events" /success:enable /failure:enable
auditpol /set /subcategory:"System Integrity" /success:enable /failure:enable
```

### Command anatomy

- `/set` changes effective audit policy.
- `/subcategory:` targets a narrow behavior rather than an entire broad
  category.
- `/success:enable` records successful operations.
- `/failure:enable` records failed attempts where supported.

Avoid enabling every subcategory at once. Excessive events consume storage and
make investigation harder. Confirm the selected result:

```powershell
auditpol /get /subcategory:"Logon","Process Creation","User Account Management","Security Group Management","Other Object Access Events","System Integrity"
```

## 4. Include command lines in process events

Preferred Group Policy path:

```text
Computer Configuration
└── Administrative Templates
    └── System
        └── Audit Process Creation
            └── Include command line in process creation events = Enabled
```

This adds command-line context to event `4688`. Command lines can contain
secrets, so use only synthetic lab data and sanitize screenshots.

Apply policy and confirm the effective setting:

```powershell
gpupdate /force
auditpol /get /subcategory:"Process Creation"
```

`gpupdate /force` reapplies computer and user policy even if Windows does not
detect a change.

## 5. Enable PowerShell logging

Open `gpedit.msc` and navigate to:

```text
Computer Configuration
└── Administrative Templates
    └── Windows Components
        └── Windows PowerShell
```

Enable:

- **Turn on PowerShell Script Block Logging**.
- **Turn on Module Logging**, and enter `*` only for the isolated learning lab.

Script block invocation logging can generate high event volume; leave it off
for the initial baseline unless a specific test requires it.

Apply and generate a harmless event:

```powershell
gpupdate /force
Write-Output "SOC-LAB PowerShell logging validation"
Get-WinEvent -LogName "Microsoft-Windows-PowerShell/Operational" -MaxEvents 5 |
    Select-Object TimeCreated,Id,ProviderName,Message
```

`Get-WinEvent` reads the named channel. `-MaxEvents 5` limits output. The
pipeline selects the fields useful for validation.

## 6. Install standalone Sysmon

Download Sysmon from Microsoft Sysinternals and use a reviewed configuration
XML. Keep the source URL and configuration revision in the workbook.

First detect an existing installation:

```powershell
Get-Service Sysmon*
```

Do not install a second standalone or built-in Sysmon implementation on top of
an existing one. From the extracted official Sysmon folder:

```powershell
.\Sysmon64.exe -accepteula -i "C:\Sysmon\sysmonconfig.xml"
```

- `-accepteula` records acceptance of the Sysinternals license.
- `-i` installs the service and applies the specified configuration.
- The XML path must refer to a reviewed local file.

Inspect the active configuration:

```powershell
.\Sysmon64.exe -c
Get-Service Sysmon*
```

`-c` prints the current configuration state. `Get-Service` confirms the
service is installed and running.

> [!NOTE]
> Sysmon records telemetry. It does not analyze events, create SIEM alerts, or
> block activity. Splunk searches provide those later functions.

## 7. Validate local Sysmon events

Generate a harmless process event:

```powershell
Start-Process notepad.exe
Start-Sleep -Seconds 2
Get-WinEvent -FilterHashtable @{
    LogName = "Microsoft-Windows-Sysmon/Operational"
    Id = 1
} -MaxEvents 5 | Select-Object TimeCreated,Id,Message
```

- `Start-Process` launches Notepad.
- `Start-Sleep` gives the event service time to write the record.
- `FilterHashtable` asks Windows to filter at the event-log layer.
- Sysmon event ID `1` represents process creation.

Close Notepad after the check.

## 8. Confirm required channels

```powershell
Get-WinEvent -ListLog Security,System,
    "Microsoft-Windows-PowerShell/Operational",
    "Microsoft-Windows-Sysmon/Operational" |
    Select-Object LogName,IsEnabled,RecordCount,FileSize,MaximumSizeInBytes
```

Expected result: all required channels are enabled and the PowerShell/Sysmon
channels contain recent lab events.

## 9. Snapshot the logging baseline

Create `03-logging-baseline` only after local validation passes.

## Evidence checkpoint

Record now, but publish later as part of `EVID-04-telemetry-ingestion.png`:

- Effective audit policy output.
- PowerShell Operational event.
- Sysmon event ID `1` for the benign Notepad launch.
- Sysmon version and configuration source.

Do not publish full command lines if they contain usernames, paths, tokens, or
other sensitive values.

## Pass criteria

- [ ] Original audit policy is backed up.
- [ ] Required audit subcategories show intended success/failure values.
- [ ] Process command-line logging is enabled for synthetic data only.
- [ ] PowerShell Operational events are generated.
- [ ] Sysmon is running with a reviewed configuration.
- [ ] Sysmon event ID `1` is visible locally.
- [ ] `03-logging-baseline` exists.

## Rollback

Restore the backed-up audit policy:

```powershell
auditpol /restore /file:"C:\LabEvidence\audit-policy-before.csv"
```

For a complete rollback, revert the `02-network-validated` snapshot. Do not use
`Sysmon64.exe -u` unless you intend to remove Sysmon; uninstalling changes the
endpoint and should be documented.

## Official references

- [Microsoft `auditpol`](https://learn.microsoft.com/windows-server/administration/windows-commands/auditpol)
- [Microsoft Sysinternals Sysmon](https://learn.microsoft.com/sysinternals/downloads/sysmon)
- [Microsoft PowerShell Script Block Logging policy](https://learn.microsoft.com/windows/client-management/mdm/policy-csp-windowspowershell)
- [Microsoft PowerShell Module Logging policy](https://learn.microsoft.com/windows/client-management/mdm/policy-csp-admx-powershellexecutionpolicy)
