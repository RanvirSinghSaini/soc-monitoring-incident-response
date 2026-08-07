# Module 02 — VMware and the Isolated Network

## What you will learn

- The difference between host-only, NAT, and bridged virtual networking.
- Why a detection lab needs stable addresses and an explicit trust boundary.
- How to validate connectivity without exposing test traffic.

## Required virtual machines

| VM | Role | vCPU | RAM | Disk | Isolated address |
|---|---|---:|---:|---:|---|
| `SOC-SPLUNK-01` | Ubuntu and Splunk Enterprise | 4 | 6–8 GB | 80 GB | `192.168.56.10/24` |
| `WIN-ENDPOINT-01` | Windows logs, Sysmon, forwarder | 2–4 | 4–6 GB | 64 GB | `192.168.56.20/24` |
| `KALI-SIM-01` | Safe simulation and connection tests | 2 | 2–4 GB | 40 GB | `192.168.56.30/24` |

These allocations are targets for this project, not vendor minimums.

## 1. Create the VMware network

In VMware Workstation Pro:

1. Open **Edit → Virtual Network Editor** as Administrator.
2. Select **Add Network** and choose an unused network such as `VMnet2`.
3. Select **Host-only**.
4. Set the subnet to `192.168.56.0` and mask to `255.255.255.0`.
5. Disable the VMware DHCP service for this network.
6. Apply the configuration.

### Why these settings matter

- **Host-only** permits communication among the host and attached VMs but does
  not route lab traffic directly to the physical network.
- **Static addresses** make configuration and evidence repeatable.
- **DHCP disabled** prevents an address change from silently breaking the
  forwarder destination.
- **Bridged mode disabled** prevents test traffic from joining the home,
  workplace, or campus broadcast domain.

## 2. Attach the VM adapters

For each powered-off VM:

1. Open **VM Settings → Network Adapter**.
2. Select **Custom: Specific virtual network**.
3. Select `VMnet2`.
4. Enable **Connect at power on**.
5. Confirm there is no bridged adapter.

For updates only, add a second NAT adapter. Label it clearly and disconnect it
before any detection test.

## 3. Configure Ubuntu networking

Use the Ubuntu interface and Netplan file names discovered on your VM; do not
assume the interface is always `ens33`.

```bash
ip -br address
ip route
```

`ip -br address` lists interfaces and addresses in a compact format. `ip route`
shows where the host will send traffic. On the isolated adapter, the project
expects the static address but no default route.

Example Netplan structure:

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.56.10/24
```

Replace `ens33` only with the discovered isolated interface. Validate before
applying:

```bash
sudo netplan try
```

`sudo` runs the command with administrative privileges. `netplan try` applies
the candidate configuration temporarily and rolls it back if it is not
confirmed. This is safer than immediately applying an untested remote-network
change.

After confirming, check:

```bash
ip -br address
ip route
```

Configure `KALI-SIM-01` the same way using `192.168.56.30/24` and the network
manager available in that image.

## 4. Configure the Windows address

Use **Settings → Network & Internet → Advanced network settings → More network
adapter options**. Open the VMware adapter properties and configure IPv4:

```text
Address: 192.168.56.20
Mask:    255.255.255.0
Gateway: blank
DNS:     blank
```

Then inspect the result in PowerShell:

```powershell
Get-NetIPConfiguration
Get-NetAdapter | Format-Table Name,Status,MacAddress,LinkSpeed
```

The first command displays IP, gateway, and DNS configuration. The second
shows adapter identity and link state. Record the adapter name and MAC address.

## 5. Test same-subnet connectivity

From Windows:

```powershell
Test-Connection 192.168.56.10 -Count 2
Test-Connection 192.168.56.30 -Count 2
```

From Ubuntu:

```bash
ping -c 2 192.168.56.20
ping -c 2 192.168.56.30
```

`Test-Connection` and `ping` send ICMP echo requests. `-Count 2` and `-c 2`
limit the test to two requests instead of running continuously.

If ICMP is blocked, test the required application port later; a failed ping
does not prove that TCP is unavailable.

## 6. Prove the isolation boundary

With NAT disconnected on all VMs:

```powershell
Get-NetRoute -DestinationPrefix "0.0.0.0/0"
```

```bash
ip route show default
```

The Windows command lists IPv4 default routes. The Linux command shows only
default routes. The isolated interfaces should not introduce an internet
gateway. The physical host may still have its normal route.

## 7. Snapshot the validated network

Create `02-network-validated` after address and isolation checks pass.

## Evidence checkpoint

Capture:

- `EVID-01-vm-inventory.png`: VM names and allocated CPU, RAM, disk.
- `EVID-02-host-only-network.png`: `VMnet2`, host-only mode, subnet, DHCP off.

Redact serial numbers, unrelated VMs, license data, and personal paths.

## Pass criteria

- [ ] All three VMs are on the same dedicated host-only subnet.
- [ ] Static addresses match the worksheet.
- [ ] No detection-test VM is bridged.
- [ ] Required same-subnet connectivity works.
- [ ] NAT adapters are disconnected before tests.
- [ ] `02-network-validated` exists.

## Common mistakes

- Assigning the same IP to two VMs.
- Leaving a default gateway on the host-only adapter.
- Editing the wrong Netplan interface.
- Treating ping failure as proof that every protocol is blocked.
- Capturing screenshots while personal VM names are visible.

## Rollback

Revert to `00-clean-os` if network packages or system files were changed
incorrectly. Otherwise power off the affected VM and correct only its virtual
adapter or static address.
