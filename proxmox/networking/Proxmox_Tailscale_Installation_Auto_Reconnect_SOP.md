# Standard Operating Procedure: Tailscale VPN on a Proxmox VE Host

| Document control | Value |
|---|---|
| Document ID | SOP-PVE-TS-001 |
| Revision | 1.2 |
| Last updated | 2026-11-21 |
| Classification | Public - generic procedure; no environment-specific data |
| Applies to | Debian-based Proxmox Virtual Environment (VE) hosts |
| Purpose | Install, authenticate, verify, and persist Tailscale remote-management connectivity |
| Change type | Host networking / remote administrative access |

> **Scope:** This procedure installs Tailscale **directly on the Proxmox VE host**, not inside a virtual machine (VM) or Linux container (LXC). It is intended for secure administrative access to the host over an existing Tailscale network (a *tailnet*). It does **not** configure a subnet router, exit node, site-to-site VPN, or Proxmox cluster transport.
>
> **Execution context:** Commands below assume the **root shell** in the Proxmox web console or a trusted local console. When using a non-root account with appropriate permissions, prepend `sudo` to commands requiring elevated privileges. Do not assume a `sudo` package is installed on every Proxmox host.

## 1. Objective and success criteria

A deployment is complete only when all of the following are true:

1. Tailscale is installed on the intended host.
2. The `tailscaled` background service is **active** and **enabled at boot**.
3. The host has been authenticated to the intended tailnet.
4. The host has a Tailscale IP address, and an authorized remote device can reach the Proxmox web interface at `https://<HOST_TAILSCALE_IP>:8006`.
5. Following an approved, controlled reboot, the host reappears on the tailnet **without a new interactive `tailscale up` or login**.
6. All applicable Proxmox services are healthy, and cluster quorum has recovered, if clustered.

**Important distinction:** `systemctl is-active tailscaled` reports whether the Linux **service** is running; it does **not**, by itself, prove the host is logged into Tailscale or reachable from a remote device.

## 2. Prerequisites and change safeguards

- Administrative access to the Proxmox host, preferably through a local/LAN console that does **not** depend on Tailscale.
- Working outbound internet access and DNS resolution from the Proxmox host.
- An existing Tailscale account/tailnet and permission to register the host.
- A second trusted device running Tailscale, signed into the intended tailnet, for remote validation.
- An approved maintenance window if the host runs VMs/containers or belongs to a cluster.
- A tested recovery path (physical console, remote management controller, or independently reachable LAN console).

**Do not change these in this SOP:** Proxmox LAN addressing, bridges, default gateway, Corosync cluster links, cluster vote counts, subnet advertisements, or exit-node settings. Do **not** forward TCP port `8006` from an internet router to the Proxmox host.

### Cluster-specific warning

Before rebooting a cluster member, run `pvecm status` and assess quorum. In some two-node cluster configurations, both votes are needed: rebooting one node temporarily removes quorum from the other. Existing guests may keep running, but cluster-management operations can be impaired. Review high-availability (HA) behavior, storage activity, migrations, jobs, and active workloads. **Do not reboot solely to test Tailscale if a safe maintenance window or recovery path is unavailable.**

## 3. Check Proxmox and installation prerequisites

Run each command in the root shell:

```bash
pveversion
cat /etc/os-release
command -v curl
command -v tailscale
```

| Command | Explanation |
|---|---|
| `pveversion` | Displays the installed Proxmox VE version, helping document compatibility. |
| `cat /etc/os-release` | `cat` prints the operating system release metadata; `/etc/os-release` is the standard distribution information file. Check that the host is Debian-based. |
| `command -v curl` | The shell builtin `command` with `-v` reports where `curl` is installed; an empty response/nonzero exit status means it was not found. |
| `command -v tailscale` | Detects an existing Tailscale client. If already installed, confirm its state before attempting another installation. |

If `curl` is missing, install the download utility and certificate bundle:

```bash
apt-get update
apt-get install -y curl ca-certificates
```

- `apt-get update`: refreshes package indexes; **does not** upgrade the installed operating system.
- `apt-get install`: installs named packages.
- `-y`: automatically answers *yes* to package-manager confirmation questions; review the proposed package changes and repository trust before using it.
- `curl`: downloads data over HTTP(S).
- `ca-certificates`: provides trusted certificate authorities for TLS verification.

> If Tailscale is already installed, **skip Section 4** and proceed to Section 5. An `inactive` or `not-found` response from systemd must be interpreted alongside whether the package and unit exist.

## 4. Install Tailscale

### Standard installation (official Linux installer)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

**Every component explained:**

| Component | Meaning |
|---|---|
| `curl` | Retrieves the installer from the specified URL. |
| `-f` | *Fail*: returns a failure status for HTTP errors rather than treating an error page as the installer. |
| `-s` | *Silent*: hides the progress meter. |
| `-S` | *Show error*: displays errors even though `-s` is in effect. |
| `-L` | *Location*: follows HTTP redirects. |
| `https://tailscale.com/install.sh` | Official Tailscale Linux installer URL. |
| `\|` | Shell pipeline: forwards the downloaded script to the next command. |
| `sh` | Executes that script using the system shell. In a Proxmox root shell, the installer runs with root privileges. |

This installer detects the supported Linux distribution, configures the appropriate Tailscale package source, and installs its packages. Read the terminal output and confirm that it reports successful installation. **Piping downloaded code directly to a shell is a trust decision**; only use the official source and follow your organization's software-review policy. For a manually reviewed or package-managed installation, follow Tailscale's official distribution-specific instructions linked in Section 16.

Verify the binary:

```bash
tailscale version
```

- `tailscale`: Tailscale command-line client.
- `version`: prints installed client/build version information. This verifies that the command is available, **not** that the host is authenticated.

## 5. Configure Tailscale to start automatically on every boot

```bash
systemctl enable --now tailscaled
```

| Component | Meaning |
|---|---|
| `systemctl` | Controls services managed by Linux `systemd`. |
| `enable` | Registers the service to start automatically during future boots. |
| `--now` | Also **starts it immediately**, rather than waiting for the next boot. |
| `tailscaled` | Tailscale's background daemon/service; this is the systemd unit name, not the `tailscale` CLI. |

Verify both properties independently:

```bash
systemctl is-enabled tailscaled
systemctl is-active tailscaled
systemctl status tailscaled --no-pager
```

| Command / option | Meaning | Successful result |
|---|---|---|
| `systemctl is-enabled tailscaled` | Checks whether the unit starts automatically at boot. | `enabled` |
| `systemctl is-active tailscaled` | Checks whether the daemon is presently running. | `active` |
| `systemctl status tailscaled --no-pager` | Displays service state, recent logs, and error details. `status` is the inspection subcommand; `--no-pager` prints directly to the terminal without opening a pager. | `Active: active (running)` |

> **Persistence:** A successfully installed, enabled daemon starts on reboot. Once the host has authenticated successfully, Tailscale ordinarily retains its login/state locally and reconnects on subsequent boots. **Do not** put `tailscale up` in crontab, `rc.local`, or an extra custom startup service merely to make it reconnect.

## 6. Authenticate the Proxmox host to the tailnet

```bash
tailscale up
```

- `tailscale`: the command-line client that communicates with `tailscaled`.
- `up`: brings the device onto the tailnet; on an unauthenticated machine, it provides an authentication URL.
- No flags are needed for basic host-only access. Avoid adding routing or exit-node flags without a separate design and change approval.

Open the generated authentication URL **privately** in a trusted browser, log into the intended Tailscale account, and authorize the host. Use the **same tailnet** as the remote administrative devices. Do not paste the one-time authentication URL into a public repository or ticket.

To view enrolled devices, open the official [Tailscale Machines page](https://login.tailscale.com/admin/machines).

## 7. Verify VPN identity and reachability

```bash
tailscale status
tailscale ip -4
```

| Command / option | Meaning |
|---|---|
| `tailscale status` | Lists the local tailnet device and visible peers, including connectivity states. A listing by itself does not prove every peer is currently reachable. |
| `tailscale ip -4` | Prints this host's Tailscale **IPv4** address; `ip` requests address information and `-4` restricts the output to IPv4. |

Write down the private address in a **secure asset inventory**, not in a public README. The address will generally be in Tailscale's `100.x.y.z` space; do not use an address copied from documentation as a real host address.

### Optional peer connectivity test

From a **different authorized Tailscale client**, run:

```bash
tailscale ping <HOST_TAILSCALE_NAME_OR_IP>
```

- `ping` is Tailscale's peer-reachability test and can show whether the connection is direct or relayed.
- `<HOST_TAILSCALE_NAME_OR_IP>` is a placeholder: replace it with the actual hostname or Tailscale IP **only in your private shell**. Do not type the angle brackets literally.
- If this test fails, verify account/tailnet membership and tailnet access policies before changing Proxmox networking.

## 8. Check Proxmox's own service status

Verify that the hypervisor's management services remain healthy after installing Tailscale:

```bash
systemctl is-active pveproxy
systemctl is-active pvedaemon
systemctl is-active pvestatd
systemctl is-active pve-cluster
```

| Service | Role | Expected state |
|---|---|---|
| `pveproxy` | HTTPS web interface and API proxy (normally TCP `8006`). | `active` |
| `pvedaemon` | Proxmox VE API daemon. | `active` |
| `pvestatd` | Proxmox node and resource status collection. | `active` |
| `pve-cluster` | Proxmox configuration filesystem/cluster service; also present on standalone installations. | `active` |

The `systemctl is-active` syntax checks the running state of each named systemd unit; it does not modify the service.

**For clustered hosts only**, check Corosync and quorum:

```bash
systemctl is-active corosync
pvecm status
```

- `systemctl is-active corosync`: checks the Corosync cluster communication service.
- `pvecm`: Proxmox VE cluster manager command.
- `status`: displays cluster membership, vote information, and whether `Quorate: Yes` (or equivalent quorum flags) is currently reported.
- On a **standalone** Proxmox host, `pvecm status` can report that no cluster configuration exists; that is not a Tailscale failure.

Do not use `pvecm expected 1` as a routine workaround for a lost quorum during a planned reboot. Resolve quorum or follow an independently approved cluster recovery SOP.

## 9. Validate remote Proxmox access

From a different device with Tailscale installed and signed into the authorized tailnet:

1. Connect the device to Tailscale.
2. Open this URL in a browser: `https://<HOST_TAILSCALE_IP>:8006`.
3. Confirm that the **correct** Proxmox login page opens and authentication succeeds with authorized Proxmox credentials.
4. Verify that the host is manageable without any internet-facing router port forwarding.

The browser might warn about an **untrusted or self-signed HTTPS certificate**. Tailscale transport encryption and the browser's HTTPS certificate-validation trust are separate controls. Verify the certificate fingerprint through a trusted channel or deploy a properly trusted certificate; do not assume every certificate warning is harmless.

If the page does not load, differentiate VPN reachability (`tailscale ping`) from application availability (`pveproxy` status), firewall policy, and tailnet access rules.

## 10. Prepare for an approved reboot test

> **Do not proceed** unless you have a maintenance window and a recovery path independent of the Tailscale connection being tested.

Check registered VM and container workloads:

```bash
qm list
pct list
```

- `qm`: Proxmox QEMU/KVM virtual machine management CLI; `list` enumerates registered VMs and their state on this host.
- `pct`: Proxmox container management CLI; `list` enumerates registered containers and their state on this host.
- Empty listings mean no VMs/containers are registered on that host; they do **not** prove no other services, storage operations, HA resources, or cluster dependencies exist.

If clustered, recheck:

```bash
pvecm status
```

Confirm cluster health and understand the effect of this node going offline. Reboot **one cluster node at a time**, and wait for quorum and critical services to recover before considering another node.

## 11. Reboot and prove automatic reconnection

From a local/LAN administrative session and **only after approval**:

```bash
reboot
```

- `reboot`: requests a controlled operating-system restart; this command takes no flags here. It disconnects your current shell and temporarily makes hosted services unavailable.

When the host returns, **do not manually issue `tailscale up` first**. The test is intended to prove unattended reconnection.

From a remote, authorized Tailscale client, attempt:

```bash
tailscale ping <HOST_TAILSCALE_NAME_OR_IP>
```

Then load `https://<HOST_TAILSCALE_IP>:8006` in the browser.

After regaining a host shell, execute:

```bash
systemctl is-enabled tailscaled
systemctl is-active tailscaled
tailscale status
tailscale ip -4
systemctl is-active pveproxy
```

For a clustered host, also run:

```bash
pvecm status
```

**Pass criteria:** `enabled`; `active`; local Tailscale identity/IP visible; authorized remote peer can reach the host; the Proxmox web interface works; and, if applicable, the cluster has recovered quorum. A boot that requires manual login is **not** a successful unattended-reconnection test.

## 12. Key-expiry policy for unattended servers

A systemd service being enabled does not guarantee indefinite authentication: a Tailscale device's node key can expire. For a trusted, managed, always-on server, administrators may choose to disable key expiry:

1. Open the [Tailscale Admin Console — Machines](https://login.tailscale.com/admin/machines) in a trusted browser.
2. Locate the intended Proxmox host by its device entry; confirm you are editing the **correct** device.
3. Open that device's three-dot (⋯) menu.
4. Select **Disable key expiry**, if permitted by the organization's security policy, and confirm any prompt presented.
5. **Explicit verification (required if this setting is chosen):** Return to the device's entry or detail view and verify its key-expiry indicator reads **Expiry disabled**. UI labels or placement may change over time; check the device details if the field is not immediately visible.
6. If **Expiry disabled** is not shown, **do not record this check as passed**. Refresh the console, confirm the correct device is selected, verify account permissions and policy, and retry only if authorized. Escalate persistent mismatches to the tailnet administrator.

**Acceptance evidence:** Record privately that the target device shows **Expiry disabled**, together with the review date. Do not publish an unredacted console screenshot or device identifier in a public repository.

**Security tradeoff:** Disabling expiry reduces forced reauthentication and helps avoid unattended outages, but it also extends the usable lifetime of a compromised device identity. Maintain restrictive access policies, remove retired devices, protect the host, and promptly revoke credentials after compromise. This setting is **optional**, not a prerequisite for `tailscaled` to start automatically.

## 13. Troubleshooting and safe recovery

| Symptom | Diagnostic command / step | Interpretation / next action |
|---|---|---|
| `tailscale: command not found` | `command -v tailscale` | The binary is missing or not on the command path. Return to installation. |
| `tailscaled` `inactive` | `systemctl status tailscaled --no-pager` | Inspect the reason, then start/enable the service if appropriate. |
| `tailscaled` `not-found` | `command -v tailscale` and installation output | The service unit/package likely has not been installed correctly. |
| Service active, but no tailnet connection | `tailscale status` | Confirm authentication and inspect connectivity messages; `active` alone is insufficient. |
| Host disappeared after reboot | `systemctl is-enabled tailscaled` | Must report `enabled`; check login state and network availability. |
| Another tailnet device cannot reach host | `tailscale ping <HOST_TAILSCALE_NAME_OR_IP>` | Check connectivity, tailnet membership, and access policy. |
| VPN works, but Proxmox UI does not | `systemctl is-active pveproxy` | Check Proxmox service, listening port, applicable firewalls, and authorization. |
| Cluster is not quorate after reboot | `pvecm status` | Investigate Corosync and cluster membership; do not change vote counts casually. |

### Inspect recent daemon logs

```bash
journalctl -u tailscaled -b -n 50 --no-pager
```

| Component | Meaning |
|---|---|
| `journalctl` | Queries systemd's journal logs. |
| `-u tailscaled` | Limits the log to the `tailscaled` service unit. |
| `-b` | Shows messages from the current boot. |
| `-n 50` | Limits the result to the newest 50 log lines. |
| `--no-pager` | Sends output directly to the terminal. |

**Never post raw diagnostic output publicly without reviewing and sanitizing it.** Logs and `tailscale status` may expose private hostnames, Tailscale addresses, public endpoints, user identifiers, or other network metadata.

### Start a stopped service (local/recovery access)

```bash
systemctl start tailscaled
```

- `start`: launches the service now, but **does not** enable it on future boots. Use Section 5 for persistence.

### Repair an expired authorization (local/recovery access only)

```bash
tailscale up --force-reauth
```

- `up`: connects the device to the tailnet.
- `--force-reauth`: deliberately starts a fresh authentication flow.
- **Do not run this over your only Tailscale-based SSH session**; it may sever your only access before authentication is complete.

### Missing TUN kernel module (only if an error specifically indicates it)

```bash
modprobe tun
```

- `modprobe`: loads a Linux kernel module and its dependencies.
- `tun`: the kernel tunnel-device module Tailscale commonly needs on Linux hosts.
- This instruction applies to a **Proxmox host**, not to unprivileged containers; investigate the precise error before modifying modules.

### Controlled rollback (only with independent access)

If a formal change rollback requires stopping Tailscale on the host:

```bash
systemctl disable --now tailscaled
```

- `disable`: removes automatic boot activation for the daemon.
- `--now`: immediately **stops** it as well.
- **Warning:** This will interrupt remote connections using Tailscale. Execute only with a verified independent management path; it does **not** uninstall packages or erase tailnet state.

## 14. Security and publication checklist

- [ ] Use tailnet access policies to limit which devices/users can access management ports such as TCP `8006` and, if applicable, SSH.
- [ ] Require strong Proxmox administrator credentials and multifactor authentication where configured/supported.
- [ ] Keep host, Tailscale, and browser software updated using an approved maintenance process.
- [ ] Keep Corosync and storage/cluster traffic on the approved LAN; treat Tailscale as an administrative-access overlay.
- [ ] Do not add router port forwarding solely for the Proxmox web UI.
- [ ] Do not add `--advertise-routes`, `--advertise-exit-node`, or `--accept-routes` unless a separate network design requires them.
- [ ] Confirm how the host firewall and Tailscale access policies apply to the management interface; tailnet access is not a substitute for Proxmox authentication.
- [ ] Review the key-expiry decision as an explicit security exception/operational policy.
- [ ] **Public repository hygiene:** Do not publish real hostnames, node IDs, cluster names, Tailscale/physical IP addresses, identity-provider details, account names, login links, auth keys, device keys, screenshots of unredacted terminal output, or raw `tailscale status` / journal logs.
- [ ] Replace all `<PLACEHOLDER>` tokens with private values **only in your own environment**, never in the public copy of this SOP.

### Final acceptance record (keep filled-in evidence private)

| Check | Required evidence | Pass / Fail |
|---|---|---|
| Installation | `tailscale version` succeeds | `[ ]` |
| Boot persistence | `systemctl is-enabled tailscaled` = `enabled` | `[ ]` |
| Running daemon | `systemctl is-active tailscaled` = `active` | `[ ]` |
| Authentication | `tailscale status` lists the intended device/tailnet | `[ ]` |
| Assigned address | `tailscale ip -4` returns an address | `[ ]` |
| Remote access | Authorized tailnet device opens Proxmox on port `8006` | `[ ]` |
| Post-reboot reconnection | Reachable without manual `tailscale up` | `[ ]` |
| Key-expiry policy (if disabled) | Console explicitly shows **Expiry disabled** for the intended device; record evidence privately | `[ ]` / `N/A` |
| Proxmox health | Required Proxmox units active | `[ ]` |
| Cluster health (if applicable) | `pvecm status` shows restored quorum | `[ ]` |

## 15. Essential command reference — installation, startup, authentication, and health

> **Purpose:** This is a standalone, task-oriented **quick reference** for the core commands used in this SOP. It is useful during first-time setup, routine verification, and post-reboot checks. For security prerequisites, cluster maintenance rules, and failure recovery, follow the full procedures above. Run the commands in the **Proxmox host's root shell**; replace no values in the commands below.

### 15.1 Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

| Part | What it does |
|---|---|
| `curl` | Downloads data from a web address. |
| `-f` | Fails on HTTP error responses instead of passing an error page to the shell. |
| `-s` | Runs silently, hiding the progress meter. |
| `-S` | Still displays errors when combined with `-s`. |
| `-L` | Follows redirects from the URL. |
| `https://tailscale.com/install.sh` | Official Tailscale Linux installation-script address. |
| `\|` | Pipes the script's downloaded contents to the next command. |
| `sh` | Executes the script with the system shell, with the privileges of the user running the command. |

**Why:** Installs the Tailscale client and background daemon on a compatible Debian-based Proxmox host. **Security:** This directly executes downloaded code; use only after validating the trusted source and following your software-change policy. Review Section 4 for installation details. Verify installation with `tailscale version`.

### 15.2 Enable automatic startup at boot

```bash
systemctl enable --now tailscaled
```

| Part | What it does |
|---|---|
| `systemctl` | Manages services controlled by Linux `systemd`. |
| `enable` | Configures the named service to start during future system boots. |
| `--now` | Also starts the service immediately, without waiting for a reboot. |
| `tailscaled` | The Tailscale **daemon** (background service); this is distinct from the `tailscale` client command. |

**Why:** Makes the daemon start automatically after power-on and reboot. **Important:** A running daemon is not proof that the host is authenticated to the tailnet. Once authenticated, Tailscale normally uses saved local state to reconnect; do not add a duplicate startup script or scheduled `tailscale up` job.

### 15.3 Authenticate and verify the VPN connection

```bash
tailscale up
tailscale status
tailscale ip -4
```

| Command | What it does | Key argument or flag |
|---|---|---|
| `tailscale up` | Connects the host to the tailnet; if not authenticated, starts a browser-based authorization flow. | `up` is a **subcommand**, not a flag. |
| `tailscale status` | Displays the local Tailscale identity and visible peers/status information. | `status` is a **subcommand**; it does not establish that every peer is reachable. |
| `tailscale ip -4` | Prints the host's assigned Tailscale IPv4 address. | `ip` requests Tailscale address information; `-4` selects **IPv4** instead of IPv6. |

**Why:** Confirms that the VPN was authorized and identifies the address to use for remote management. Keep login URLs, peer names, and assigned IP addresses **private**. To prove remote reachability, test from a **separate authorized client** using `tailscale ping <HOST_TAILSCALE_NAME_OR_IP>` and browse to `https://<HOST_TAILSCALE_IP>:8006` (see Sections 7 and 9). Angle-bracketed values are placeholders, not literal hostnames.

### 15.4 Check that automatic startup is persistent

```bash
systemctl is-enabled tailscaled
systemctl is-active tailscaled
```

| Command | What it verifies | Expected result |
|---|---|---|
| `systemctl is-enabled tailscaled` | Whether `tailscaled` is configured to start at boot; `is-enabled` queries the unit's startup state without changing it. | `enabled` |
| `systemctl is-active tailscaled` | Whether the daemon is currently running; `is-active` queries service runtime state without changing it. | `active` |

**Why:** These two checks prove different things: boot persistence and current process state. **Neither alone proves VPN connectivity or post-reboot reconnection**; also run `tailscale status` and test access from another authorized device. For detailed service diagnostics use `systemctl status tailscaled --no-pager` (see Section 5).

### 15.5 Verify Proxmox cluster health (clustered hosts only)

```bash
pvecm status
```

| Part | What it does |
|---|---|
| `pvecm` | Proxmox VE cluster-management command. |
| `status` | Displays cluster membership, votes, and quorum state; it **does not** alter cluster configuration. |

**Why:** On a clustered host, confirm quorum is healthy before and after maintenance. A healthy cluster should report **`Quorate: Yes`** (with vote totals appropriate to that cluster's design). **Do not reboot a cluster member based on this command alone**: also assess running workloads, high-availability behavior, maintenance timing, and independent recovery access (see Sections 2, 8, 10, and 11). On a standalone host, a no-cluster error is normal.

### 15.6 Quick verification sequence after reboot

After an **approved** restart, first verify remote access from a separate authorized Tailscale device **without manually running `tailscale up` on the Proxmox host**. Then run on the host:

```bash
systemctl is-enabled tailscaled
systemctl is-active tailscaled
tailscale status
tailscale ip -4
systemctl is-active pveproxy
```

On a clustered host, additionally run:

```bash
pvecm status
```

**Pass conditions:** The daemon is `enabled` and `active`; the device is authenticated and reachable from the remote client; the Proxmox web interface on TCP `8006` works; `pveproxy` is active; and, for a cluster, quorum has recovered. If a device key might expire, also apply the **optional key-expiry policy and explicit `Expiry disabled` verification** in Section 12. Record actual command output privately, never in the public repository.

## 16. Official references

- [Tailscale — Install on Linux](https://tailscale.com/docs/install/linux)
- [Tailscale — Command-line reference](https://tailscale.com/docs/reference/tailscale-cli)
- [Tailscale — `tailscale up`](https://tailscale.com/docs/reference/tailscale-cli/up)
- [Tailscale — Key expiry](https://tailscale.com/docs/features/access-control/key-expiry)
- [Tailscale — Unattended devices](https://tailscale.com/docs/how-to/run-unattended)
- [Proxmox VE — Administration Guide](https://pve.proxmox.com/pve-docs/pve-admin-guide.html)

## 17. Document revision history

| Revision | Date | Change summary | Approval / ownership |
|---|---|---|---|
| 1.0 | Not recorded | Initial SOP: installation, daemon status, boot persistence, tailnet authentication, remote-access validation, safe reboot testing, and troubleshooting. | Generic template — assign internally |
| 1.1 | 2026-11-21 | Added explicit verification of the Tailscale Admin Console **Expiry disabled** status, acceptance evidence, and this formal revision-history table. | Generic template — assign internally |
| 1.2 | 2026-11-21 | Added Section 15, a dedicated command reference covering installation, boot startup, tailnet authentication, persistence verification, Proxmox quorum checks, and post-reboot validation, with command and flag explanations. | Generic template — assign internally |

**Revision-control rule:** Update the document revision, last-updated date, and this table whenever published instructions change. Keep real reviewer names, ticket numbers, infrastructure identifiers, and signed approvals in private change-management records rather than the public SOP.

---

**Operational note:** This SOP describes a reproducible configuration. It does not guarantee that every outage can be recovered remotely: power loss, loss of ISP service, damaged disks, a revoked key, tailnet policy changes, or host networking failure can still make the machine unreachable. Maintain an independent recovery path.
