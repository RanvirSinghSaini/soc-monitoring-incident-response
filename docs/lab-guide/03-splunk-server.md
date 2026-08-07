# Module 03 — Splunk Server

## What you will learn

- The single-instance Splunk roles used in this learning lab.
- The difference between Splunk Web, the management port, and the receiving
  port.
- How to create dedicated indexes and validate service health.

## Architecture in this module

`SOC-SPLUNK-01` acts as both indexer and search head. This is appropriate for a
resource-constrained learning lab, not a production high-availability design.

| Port | Direction | Purpose |
|---:|---|---|
| `8000/tcp` | Analyst to Splunk | Splunk Web |
| `9997/tcp` | Windows forwarder to Splunk | Forwarded event receiver |
| `8089/tcp` | Local Splunk components | Management API; do not expose publicly |

## 1. Confirm the server identity

On `SOC-SPLUNK-01`:

```bash
hostnamectl
ip -br address
timedatectl status
```

- `hostnamectl` shows the system hostname and operating-system details.
- `ip -br address` confirms the lab address `192.168.56.10/24`.
- `timedatectl status` shows clock synchronization and time zone.

Correct time is essential because analysts correlate events across machines.

## 2. Install Splunk from the official package

Download the current supported Linux package from Splunk while the temporary
NAT adapter is connected. Record the exact filename, version, download date,
and publisher source. Disconnect NAT after all required updates are complete.

For a downloaded Debian package, the installation pattern is:

```bash
sudo dpkg -i ./splunk-package-name.deb
```

- `dpkg -i` installs a local Debian package.
- `./` explicitly refers to a file in the current directory.
- Replace the example filename with the exact official package you downloaded.

Do not copy an old filename from this guide. Splunk versions and package names
change.

## 3. Perform the first start

```bash
sudo /opt/splunk/bin/splunk start --accept-license
```

- `/opt/splunk/bin/splunk` is the default Splunk CLI path for the package.
- `start` launches Splunk.
- `--accept-license` records acceptance of the displayed license terms.

Read the license and create a unique lab administrator credential when
prompted. Never put that password in a screenshot, shell history, repository,
or command example.

Validate:

```bash
sudo /opt/splunk/bin/splunk status
sudo ss -lntp | grep -E ':8000|:8089'
```

- `status` asks Splunk whether its processes are running.
- `ss -lntp` lists listening TCP sockets and owning processes.
- `grep -E` keeps lines matching either port.

Expected result: Splunk is running and listening locally on the configured Web
and management ports.

## 4. Open Splunk Web safely

From the physical host, browse to:

```text
http://192.168.56.10:8000
```

Use the lab administrator account. The address is intentionally private and
must not be exposed with router port-forwarding, a public tunnel, or a public
cloud security-group rule.

## 5. Create project indexes

In Splunk Web:

1. Open **Settings → Indexes**.
2. Select **New Index**.
3. Create `windows`.
4. Create `sysmon`.
5. Create `lab_alerts`.
6. Keep default retention for the first learning pass; document later tuning.

Why separate indexes:

- `windows` holds standard Windows channels.
- `sysmon` isolates higher-volume endpoint telemetry.
- `lab_alerts` is reserved for lab-generated alert/case output.

An index is the searchable storage boundary. A sourcetype describes the event
format. They are related but not interchangeable.

## 6. Enable the forwarder receiver

Preferred browser method:

1. Open **Settings → Forwarding and receiving**.
2. Select **Configure receiving**.
3. Confirm `9997` is not already configured.
4. Select **New Receiving Port**.
5. Enter `9997` and save.

Configuration-file equivalent at
`/opt/splunk/etc/system/local/inputs.conf`:

```ini
[splunktcp://9997]
disabled = 0
```

Avoid placing the Splunk administrator password in a shell command. The Web UI
or reviewed configuration file avoids storing it in shell history.

Restart and validate after a file-based change:

```bash
sudo /opt/splunk/bin/splunk restart
sudo ss -lntp | grep ':9997'
```

The first command performs a controlled Splunk restart. The second verifies a
listening TCP socket on the conventional receiver port.

## 7. Restrict host firewall access

If Ubuntu uses UFW, allow the host-only subnet only:

```bash
sudo ufw allow from 192.168.56.0/24 to any port 8000 proto tcp
sudo ufw allow from 192.168.56.0/24 to any port 9997 proto tcp
sudo ufw status numbered
```

- `allow from` limits the source network.
- `to any port` selects the local destination port.
- `proto tcp` prevents an unnecessarily broad UDP rule.
- `status numbered` lists active rules with identifiers.

Do not enable UFW remotely until you understand the existing access rules. A
mistake can lock out legitimate administration. This lab can also remain behind
the host-only boundary while you learn firewall management separately.

## 8. Run the first health searches

In **Search & Reporting**, use a 24-hour time range:

```spl
| rest /services/server/info
| table serverName version build os_name numberOfCores physicalMemoryMB
```

`rest` reads Splunk's internal service endpoint. `table` shows only selected
fields. This confirms the active instance and records its version.

Then confirm the new indexes exist:

```spl
| rest /services/data/indexes
| search title IN (windows, sysmon, lab_alerts)
| table title disabled totalEventCount currentDBSizeMB
```

The event counts may be zero before the forwarder is installed. That is
expected.

## Evidence checkpoint

Capture the receiver configuration and a separate terminal result showing TCP
`9997` listening. Do not include credentials or session cookies. This becomes
part of `EVID-03-forwarder-health.png` after the Windows forwarder connects.

Record in the workbook:

- Splunk version and build.
- Install package name and publisher source.
- Web URL used inside the lab.
- Index names.
- Receiver port and firewall decision.

## Pass criteria

- [ ] Splunk starts without an error.
- [ ] Splunk Web is reachable only through the lab path.
- [ ] `windows`, `sysmon`, and `lab_alerts` exist.
- [ ] TCP `9997` is listening.
- [ ] Management port `8089` is not publicly exposed.
- [ ] NAT is disconnected before detection work.

## Troubleshooting

| Symptom | Check | Interpretation |
|---|---|---|
| Web page does not open | `ss -lntp \| grep ':8000'` | No listener means Splunk Web is not active or port changed |
| Splunk will not start | `/opt/splunk/var/log/splunk/splunkd.log` | Review the newest error without publishing secrets |
| Receiver test fails | `ss -lntp \| grep ':9997'` | Enable receiver or correct firewall |
| Searches show wrong time | `timedatectl status` | Correct time before collecting evidence |

## Rollback

Stop Splunk with `sudo /opt/splunk/bin/splunk stop` before reverting the VM
snapshot. Revert to `02-network-validated` if the installation or index design
must be repeated.

## Official references

- [Splunk: enable a receiver](https://help.splunk.com/en/data-management/get-data-in/forward-data-with-universal-forwarders/10.2/configure-the-universal-forwarder/enable-a-receiver-for-splunk-enterprise)
- [Splunk: secure the platform](https://help.splunk.com/en/splunk-enterprise/administer/secure-splunk-enterprise)
