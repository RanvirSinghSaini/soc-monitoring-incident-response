# Module 01 — Safety and Planning

## What you will learn

- Why authorization, isolation, repeatability, and rollback come before tools.
- How to record versions and resources so another learner can reproduce a lab.
- How to define success before generating security events.

## Outcome

A written lab worksheet with authorized scope, VM names, addresses, resources,
software sources, evidence rules, and snapshot names.

## 1. Define the authorization boundary

Write this statement in your private worksheet and fill in the date:

```text
I will perform this project only on virtual machines and network segments that
I own or am explicitly authorized to test. I will not target public, employer,
school, customer, or third-party systems. Authorized on: YYYY-MM-DD.
```

The lab boundary is the VMware host-only subnet `192.168.56.0/24`. A temporary
NAT adapter may be attached for operating-system and vendor updates, but it must
be disconnected before detection tests. Bridged networking is out of scope.

## 2. Create the private build worksheet

Copy the environment inventory from the
[analyst workbook](../reference/ANALYST-WORKBOOK.md) into a private working
file. Do not store passwords, product keys, tokens, or personal information in
that file or in Git.

Record at least:

- Physical host CPU, memory, storage, and hypervisor version.
- VM operating-system edition and build.
- Splunk Enterprise and Universal Forwarder versions.
- Sysmon version and the source of its XML configuration.
- VM names, MAC addresses, isolated IP addresses, and allocated resources.
- Time zone used by every VM. UTC is preferred for analyst records.
- Date and purpose of each snapshot.

## 3. Verify downloaded software before use

Download VMware, operating-system media, Splunk, the Universal Forwarder, and
Sysmon only from their official publishers. On Windows, calculate a SHA-256
hash for each installer:

```powershell
Get-FileHash "C:\LabInstallers\installer-name.exe" -Algorithm SHA256
```

### Hash command explanation

- `Get-FileHash` calculates a cryptographic digest without executing the file.
- The quoted path identifies the exact installer being checked.
- `-Algorithm SHA256` selects SHA-256 rather than an older digest.

Compare the hash with a publisher-provided value when one exists. A hash proves
that two files are identical; it does not independently prove that a file is
safe. Also inspect the digital signature:

```powershell
Get-AuthenticodeSignature "C:\LabInstallers\installer-name.exe" |
    Format-List Status,StatusMessage,SignerCertificate
```

Expected result: `Status` is `Valid` for a signed Windows executable from the
expected publisher. Some Linux packages use repository signatures instead.

## 4. Define the completion gates

The lab is complete only when:

1. The network is isolated and documented.
2. Splunk receives the required event channels.
3. Ingestion delay is measured, not guessed.
4. Every detection has three positive tests and at least one negative test.
5. Every alert is classified using raw events and context.
6. Five response playbooks are exercised.
7. Ten required evidence labels are sanitized and present.
8. A fresh public clone contains all documentation and no secrets.

## 5. Prepare local evidence storage

On the Windows endpoint, open PowerShell as Administrator and create a private
working directory:

```powershell
New-Item -Path "C:\LabEvidence" -ItemType Directory -Force
```

### Directory command explanation

- `New-Item` creates a filesystem object.
- `-Path` sets its location.
- `-ItemType Directory` creates a folder rather than a file.
- `-Force` makes the command repeatable when the folder already exists.

Raw `.evtx`, `.pcap`, and large logs remain outside Git. The repository accepts
only reviewed screenshots, small sanitized examples, queries, configurations,
and notes.

## 6. Create the first snapshot

After installing each clean operating system, shut it down and create the
snapshot `00-clean-os`. Do not include active installer ISOs unless required.

## Evidence and notes checkpoint

Do not publish passwords, keys, serial numbers, personal desktop content, or
full license screens.

Record:

- Authorization date and scope.
- Hash/signature results for downloaded software.
- Version table.
- Snapshot names.
- Any substitutions from the documented architecture.

## Pass criteria

- [ ] Authorization boundary is written.
- [ ] Bridged networking is prohibited in the plan.
- [ ] Software sources and versions are recorded.
- [ ] Private and public evidence locations are separated.
- [ ] `00-clean-os` exists for all three VMs.

## Rollback

No test behavior has occurred. If the plan is wrong, delete only the newly
created lab VMs after confirming their exact names, then redesign the worksheet.
Do not delete unrelated virtual machines.

## Official references

- [Microsoft `Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [Microsoft `Get-AuthenticodeSignature`](https://learn.microsoft.com/powershell/module/microsoft.powershell.security/get-authenticodesignature)
- [GitHub guidance for sensitive data](https://docs.github.com/code-security/secret-scanning/introduction/about-secret-scanning)
