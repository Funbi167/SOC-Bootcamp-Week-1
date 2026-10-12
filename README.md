# SOC Analyst Bootcamp, Week 1: Build, Baseline & Investigate a Small IT Environment

**Team:** Sentinel | **Author:** Oluwafunbi Bader | **Date:** October 12, 2026

> All testing in this project was done inside my own authorized lab. No real malware was used.

**How to read this report.** Each section starts with **In plain words** (a short, non-technical summary), followed by **Technical detail** (commands, values, evidence). Each screenshot has its own caption explaining what it shows. Anything not completed is marked **Pending** with a reason.

## Contents
1. [Executive summary](#01-executive-summary)
2. [Environment overview and network diagram](#02-environment-overview-and-network-diagram)
3. [Asset and network inventory](#03-asset-and-network-inventory)
4. [Windows baseline](#04-windows-baseline)
5. [Linux administration](#05-linux-administration)
6. [Evidence and logs](#06-evidence-and-logs)
7. [Wireshark analysis](#07-wireshark-analysis)
8. [Controlled security activity](#08-controlled-security-activity)
9. [Before vs after](#09-before-vs-after)
10. [Mini investigation](#10-mini-investigation)
11. [Recommendations](#11-recommendations)
12. [Conclusion](#12-conclusion)
13. [Problems faced and solutions](#problems-faced-and-solutions)

---

## 01 Executive summary

**In plain words.** I built a small practice company network with three computers: an employee workstation, a company server and a security-testing machine. I recorded what each looks like when nothing unusual is happening, performed a few controlled actions on my own lab, and then found the traces those actions left behind in the logs and in network traffic. This is the foundation for next week, when these logs will be collected in one central place (a SIEM).

**Technical detail.**
- Windows endpoint: physical Windows 11 laptop (host).
- Linux server: Ubuntu running in WSL2 on the same laptop.
- Security tester: Kali Linux VM in VirtualBox, Bridged Adapter.
- Activities: user and group management, file permission changes, successful and failed logins, sudo and service actions, and a controlled nmap port scan from Kali against the Windows laptop.
- Key finding: every activity left evidence, mostly in `/var/log/auth.log`, `journalctl`, Windows Event Viewer and Wireshark. A file permission change has no log event of its own, only the sudo command that ran it.

---

## 02 Environment overview and network diagram

**In plain words.** Think of three roles in an office: an employee's PC, the company server and a security consultant's laptop. The lab has one of each, and they can talk to each other.

| Role | System | How it runs |
|---|---|---|
| Windows endpoint (employee workstation) | Windows 11 Home (build 22631) | Physical laptop (host machine) |
| Linux server (organization server) | Ubuntu | WSL2 on the same laptop |
| Security testing machine | Kali Linux | VirtualBox VM, Bridged Adapter |

![Network diagram of the Week 1 lab](evidence/fig2_1_network_diagram.png)

*Figure 2.1: Network diagram showing the Windows endpoint, Linux server and Kali tester with IPs, roles and connections. The Linux server runs inside the Windows laptop (WSL2) and reaches the network through Windows.*

**Deviation from the brief (stated openly).** The brief asks for three VMs. The Windows ISO could not be downloaded and Ubuntu was not available as a VM, so I used the host Windows installation and Ubuntu on WSL2. A dedicated Ubuntu Server VM is planned before next week's SIEM work.

---

## 03 Asset and network inventory

**In plain words.** This is the list of every machine and its "address" on the network, like an address book for the lab.

| Field | Windows endpoint | Linux server (WSL2) | Kali tester |
|---|---|---|---|
| Hostname | DESKTOP-A67S0F4 | DESKTOP-A67S0F4 (shared with Windows) | nyx |
| OS | Windows 11 Home, build 22631 | Ubuntu on WSL2 (version not recorded) | Kali Linux (version not recorded) |
| IP address | 10.60.174.211/24 (Wi-Fi) | 192.168.122.184/20 | 10.60.174.79/24 |
| MAC address | 38-68-93-46-51-09 | 00:15:5d:95:f4:8f | Not recorded (bridged over Wi-Fi, so frames carry the host MAC) |
| Role | Employee workstation | Organization server | Security testing |
| Network adapter | Physical Wi-Fi (Intel Wi-Fi 6 AX201) | WSL2 virtual adapter (eth0) | VirtualBox Bridged Adapter (eth0) |
| Default gateway | 10.60.174.106 | 192.168.112.1 | Not recorded |
| DNS | 10.60.174.106 | Not recorded | 10.60.174.106 (seen in Wireshark) |

![Windows ipconfig /all output](evidence/fig3_0a_windows_ipconfig.png)

*Figure 3.0a: `ipconfig /all` on Windows. It lists a Proton VPN tunnel (10.2.0.2) and two VirtualBox Host-Only adapters. The Wi-Fi adapter appears in the next figure.*

![Windows Wi-Fi adapter configuration](evidence/fig3_0a2_windows_wifi.png)

*Figure 3.0a (continued): The Wi-Fi adapter has IPv4 10.60.174.211/24. The gateway, DHCP server and DNS are all 10.60.174.106. The vEthernet (WSL) adapter is 192.168.112.1, the same address WSL uses as its default gateway, so WSL reaches the network through Windows.*

![WSL hostname, ip a, ip route and os-release](evidence/fig3_0b_wsl_ip.png)

*Figure 3.0b: WSL2 `eth0` is 192.168.122.184/20 with gateway 192.168.112.1. The OS line reads "Docker Desktop", so this shell was the docker-desktop WSL distribution, not Ubuntu (see Problem 7).*

### Communication tests

**In plain words.** I proved the machines can reach each other by sending a small "are you there?" message (a ping) and getting an answer.

**Technical detail.** Windows blocks ping by default, so I added a firewall rule first:

```
netsh advfirewall firewall add rule name="Allow ICMP" protocol=icmpv4:8,any dir=in action=allow
```

- **Kali (10.60.174.79) to Windows (10.60.174.211):** 4 replies, 0% packet loss, TTL 128 (Figure 3.3).
- **Windows (10.60.174.211) to Kali (10.60.174.79):** 4 replies, 0% loss, TTL 64 (Figure 3.2).
- **DNS:** shown in the Wireshark capture (Figure 7.2): Kali's queries for http.kali.org and kali.download were answered by 10.60.174.106.
- **Service connection:** SSH is **Pending**. The Ubuntu server runs inside WSL2, which Kali cannot reach without extra network setup. A service-level connection to the Windows endpoint was tested with the nmap scan instead (Figure 8.2).

![Kali pings itself](evidence/fig3_1_kali_ping.png)

*Figure 3.1: Kali pings 10.60.174.79, which Wireshark later showed is Kali's own address. All 203 packets were answered, but this alone does not prove Kali reaches Windows. The real test is Figure 3.3.*

![Windows pings Kali](evidence/fig3_2_windows_ping_kali.png)

*Figure 3.2: Windows pings Kali (10.60.174.79): 4 replies, 0% loss, TTL 64 (typical of Linux).*

![Kali pings the Windows Wi-Fi address](evidence/fig3_3_kali_ping_windows.png)

*Figure 3.3: Kali pings Windows (10.60.174.211): 4 replies, 0% loss, TTL 128 (typical of Windows). With Figure 3.2, this proves two-way communication.*

---

## 04 Windows baseline

**In plain words.** A baseline is a snapshot of what "normal" looks like. If something odd happens later, I can compare it with this snapshot.

| Item | Value |
|---|---|
| Hostname | DESKTOP-A67S0F4 |
| Windows version | Microsoft Windows 11 Home, version 10.0.22631, build 22631 |
| IP configuration | See Section 03 |
| Firewall status | Domain, Private and Public profiles all ON (Figure 4.5) |

![Windows hostname, whoami and systeminfo](evidence/fig4_1_host_info.png)

*Figure 4.1: Identity of the Windows endpoint. At the top, `findstr`, `systeminfo` and `ipconfig` were first typed inside Ubuntu and failed with "command not found". After `exit`, they worked in PowerShell.*

### Process analysis

| Process | Expected? | Evidence |
|---|---|---|
| `explorer.exe` | Yes | PID 17780, running from `C:\WINDOWS\Explorer.EXE`, the correct location |
| `svchost.exe` | Yes | Dozens of instances (for example PIDs 840, 1152, 1320). Readable paths are under `C:\WINDOWS\system32` |
| `lsass.exe` | Yes | One instance, PID 980. Path blank because PowerShell was not run as Administrator |

*Pending: my own-words descriptions of what each process does, why it matters and what would make it suspicious still need to be added.*

![Top 15 processes by CPU time](evidence/fig4_2_processes.png)

*Figure 4.2: The 15 busiest processes by CPU time. A chat notification pop-up was blacked out for privacy.*

![Process paths for explorer, svchost and lsass](evidence/fig4_2b_process_paths.png)

*Figure 4.2b: `Get-Process explorer, svchost, lsass | Select Name, Id, Path`. Blank paths mean PowerShell was not elevated. Re-run as Administrator to confirm them.*

### Services, ports and connections

![Listening TCP ports](evidence/fig4_3_listening_ports.png)

*Figure 4.3: `netstat -ano | findstr LISTENING`. Standard Windows ports (135, 445, 139) and application ports. PID 4 is the Windows System process. `127.0.0.1:22` listens on the loopback only.*

![Established TCP connections](evidence/fig4_3b_established.png)

*Figure 4.3b: `netstat -ano | findstr ESTABLISHED`. Outbound connections use source 10.2.0.2 (the Proton VPN tunnel), mostly to port 443. The VPN was active during the baseline.*

![Running Windows services](evidence/fig4_3c_services.png)

*Figure 4.3c: The first 20 running services.*

![Allow ICMP firewall rule](evidence/fig4_4_icmp_rule.png)

*Figure 4.4: The `Allow ICMP` inbound rule: Enabled, Direction In, ICMPv4 type 8, Action Allow. This lets Windows answer pings.*

![Firewall state for all profiles](evidence/fig4_5_firewall_state.png)

*Figure 4.5: `netsh advfirewall show allprofiles state`. All three profiles are ON.*

---

## 05 Linux administration

**In plain words.** I acted as the server's administrator: I created accounts, grouped them, and controlled who may open a shared folder.

### 5.1 Users and group

Created users `alice` and `bob` and group `project`. Added alice, removed her, and added her back, checking with `getent group project` each time.

![Users, group and alice added](evidence/fig5_1_users_group.png)

*Figure 5.1: `adduser` for alice and bob, `groupadd project`, `usermod -aG project alice`. `getent` returns `project:x:1003:alice`.*

![Alice removed from group and added back](evidence/fig5_2_group_remove_readd.png)

*Figure 5.2: `gpasswd -d alice project` empties the group, then `usermod -aG` adds her back.*

### 5.2 File permissions

`/srv/teamfiles` is owned by `alice`, group `project`, mode **770**.

| Who | Permission | Why |
|---|---|---|
| alice (owner and group member) | read, write, enter | owner and group both allow it |
| bob | none | not owner and not in `project`, so he is "others", which has no permissions in 770 |
| my own WSL user | none (denied without sudo) | same reason as bob |

![Folder and file ownership and permissions](evidence/fig5_4_permissions.png)

*Figure 5.4: `drwxrwx--- alice project` (770) and `-rw-rw---- alice project` (660). Listing as my own user fails with "Permission denied" and works with `sudo`. An earlier `chown` failed because the path was typed `/srvteamfiles` (missing slash).*

![Alice can read the file and bob is denied](evidence/fig5_5_alice_bob.png)

*Figure 5.5: `sudo -u alice cat` prints `secret`. `sudo -u bob cat` returns "Permission denied".*

**Permission change:** Pending. The 770 to 750 change was not captured.

### 5.3 Logins and admin actions

- Normal login with `su - alice` (the matching log line was not captured).
- Failed logins: wrong passwords for alice at 19:16 and for bob at 19:17 (Figure 6.2).
- Admin actions: `sudo apt update`, and `sudo systemctl stop ssh` then `start ssh` at 19:27 (stop) and 19:28 (start), see Figure 6.3.

![SSH service journal](evidence/fig5_7_ssh_journal.png)

*Figure 5.7: `journalctl -u ssh` shows the SSH service starting at 19:28:35. No separate "Stopping" line is visible, so the sudo log (Figure 6.3) is the evidence for the stop.*

---

## 06 Evidence and logs

**In plain words.** Every action leaves footprints. This section shows where I found them and what they tell an investigator.

| Activity | Evidence | Where | What it tells us |
|---|---|---|---|
| Group change | `usermod` / `gpasswd` entries | `/var/log/auth.log` | Alice added, removed and re-added |
| Failed login | `authentication failure`, `FAILED SU` | `/var/log/auth.log` | Wrong password, with user and time |
| Admin action | `COMMAND=/usr/bin/systemctl stop ssh` | `/var/log/auth.log` | Which privileged command ran, by whom, when |
| Permission change | `ls -ld` before/after | Command output | No dedicated log event, so it is easy to miss |
| Windows logons | 4624 / 4625 | Event Viewer, Security | Successful and failed logons |

*Not captured: the `useradd` entries and the successful `su - alice` login line.*

![Group changes in auth.log](evidence/fig6_1_group_log.png)

*Figure 6.1: `groupadd` at 18:57:18, `usermod` at 18:57:38, `gpasswd -d` (removed) at 18:59:12, `usermod` again at 18:59:47. The "session opened" lines are sudo sessions for root, not logins as alice.*

![Failed login entries](evidence/fig6_2_failed_logins.png)

*Figure 6.2: Failed `su` attempts for alice (19:16:04, 19:16:17) and bob (19:17:28). Times carry a +01:00 offset.*

![Sudo commands in auth.log](evidence/fig6_3_sudo_commands.png)

*Figure 6.3: Every sudo command is logged with who ran it, the terminal and the command: `systemctl stop ssh` at 19:27:18 and `start ssh` at 19:28:34.*

![Windows Event Viewer security log with 4624 and 4625 events](evidence/fig6_4_event_viewer.png)

*Figure 6.4: Security log filtered to 4624 and 4625 (295 of 34,619 events). One 4625 (failed logon) appears at 6:08:40 PM. Its cause was not recorded.*

---

## 07 Wireshark analysis

**In plain words.** Wireshark records the conversations between machines on the network, like a call log showing who talked to whom.

Capture taken on the Wi-Fi adapter (Intel Wi-Fi 6 AX201).

| Traffic | Filter | Source and destination | Protocol / ports | Cause |
|---|---|---|---|---|
| ICMP | `icmp` | 10.60.174.79 (Kali) to 10.2.0.2 (Windows VPN address) | Echo and timestamp requests, no replies | nmap host discovery (Figure 8.1) |
| DNS | `dns` | 10.60.174.79 (Kali) to 10.60.174.106 (DNS server) | UDP 53 | Kali software repository lookups |
| HTTPS | `tls` | 10.60.174.211 and 35.186.224.33 / 72.144.120.211 | TCP 443 | Visible with the VPN off (Figure 7.5), hidden with it on (Figures 7.3, 7.4) |

![Wireshark ICMP filter](evidence/fig7_1_icmp.png)

*Figure 7.1: Four ICMP packets from Kali to 10.2.0.2 with no replies and changing TTLs (37, 55, 38, 53). This matches nmap's host-discovery probes from the first scan (Figure 8.1). The source MAC is the Windows Wi-Fi card because VirtualBox bridging over Wi-Fi uses the host's MAC.*

![Wireshark DNS filter](evidence/fig7_2_dns.png)

*Figure 7.2: 26 DNS packets. Kali (10.60.174.79) asks 10.60.174.106 on UDP port 53 for http.kali.org and kali.download, and gets answers.*

![Wireshark TLS filter with no results](evidence/fig7_3_tls_empty.png)

*Figure 7.3: The `tls` filter returns 0 of 21,439 packets even though pages were opened. Figure 7.4 explains why.*

![Wireshark showing WireGuard traffic](evidence/fig7_4_wireguard.png)

*Figure 7.4: The capture is dominated by WireGuard "Transport Data" packets. Proton VPN encrypts the web traffic inside its tunnel, so HTTPS is not visible on the Wi-Fi interface. (The filter text in the bar had not been applied, so this is the unfiltered capture.)*

![Wireshark with the VPN off showing TLS, DNS and ARP](evidence/fig7_5_tls_visible_vpn_off.png)

*Figure 7.5: With the VPN off, TLSv1.2 application data is visible over TCP 443. Frames 44 to 48 show a DNS lookup. Frames 49 and 50 are ARP: the gateway asks "Who has 10.60.174.211?" and Windows answers with its MAC. The filter text had not been applied, so this is the unfiltered capture.*

SSH traffic was not captured because SSH to the WSL2 server was not set up.

---

## 08 Controlled security activity

**In plain words.** To learn what suspicious activity looks like, I deliberately performed a harmless test on my own lab only.

| Question | Answer |
|---|---|
| What was done | `nmap -sS` port scan of 1,000 TCP ports |
| From which machine | Kali (10.60.174.79) |
| Against which machine | Windows endpoint (10.60.174.211), my own lab |
| When | 10/11/2026 at 20:20 on Kali's clock (-0400), which is 01:20 in Windows' +01:00 time zone |
| Why | To see what a scan looks like in network traffic and logs |
| Evidence generated | Nmap output (Figure 8.2): host up, all 1,000 ports filtered (no response). Wireshark capture of the scan: Pending |

![First nmap attempt against the VPN address](evidence/fig8_1_nmap_first_attempt.png)

*Figure 8.1: `-nmap -sS 10.2.0.2` failed with "command not found" (a stray leading dash). `sudo nmap -sS 10.2.0.2` then reported "Host seems down", because 10.2.0.2 is the VPN tunnel address and is not reachable over Wi-Fi.*

![Nmap SYN scan of the Windows laptop](evidence/fig8_2_nmap_scan.png)

*Figure 8.2: `sudo nmap -sS 10.60.174.211`. The host is up, but all 1,000 TCP ports are "filtered (no-response)": the Windows Firewall silently dropped every SYN probe. The scan took 22 seconds. Figure 4.3 lists ports such as 135 and 445 as listening, yet none were reachable from Kali: listening is not the same as reachable.*

*Figure 8.3 (Pending): a Wireshark capture of the scan was not obtained, because the captures taken did not include the scan traffic.*

---

## 09 Before vs after

| Observation | Normal baseline | Controlled activity | Difference | Security relevance |
|---|---|---|---|---|
| Failed logins for alice | 0 | Wrong passwords for alice (19:16) and bob (19:17) | At least 2 `authentication failure` entries | Resembles password guessing |
| Traffic from Kali | Only ping | Port scan | 1,000 TCP ports probed in 22 seconds, all filtered (Figure 8.2) | Typical scan pattern |
| Folder permissions | 770 | 750 (Pending) | Group would lose write access | Silent change with no dedicated log event |

*Observed facts are in the middle columns and conclusions in the last. Unusual does not automatically mean malicious.*

---

## 10 Mini investigation

**Topic:** Failed logins (wrong passwords) for alice and bob on the Linux server

1. **Observation:** auth.log recorded two failed `su` attempts for alice (19:16:04 and 19:16:17) and one for bob (19:17:28) on 10/11/2026.
2. **Evidence:** `authentication failure` and `FAILED SU` lines in `/var/log/auth.log` (Figure 6.2).
3. **Interpretation:** this pattern can indicate password guessing (repeated wrong passwords against one account within seconds).
4. **Alternative explanation:** a user mistyped a password, or an admin ran a legitimate scanner.
5. **Conclusion:** this was my own controlled test, so it is not malicious. The pattern is the same one a real attack would leave.
6. **Additional evidence for a real investigation:** who owns the source IP, whether other hosts were targeted, timing between attempts, whether a successful login followed, firewall logs, and whether the activity was authorized.

---

## 11 Recommendations

- Send Linux auth logs and Windows Security events to a central SIEM (next week's work).
- Alert on repeated failed logins from one source and on port-scan patterns.
- Alert on sudo use and on changes to groups and accounts.
- Use SSH keys instead of passwords and restrict who can reach SSH.
- Keep permissions at least privilege and monitor permission changes, since they have no dedicated log event.
- Keep time zones consistent across machines so logs can be correlated.

---

## 12 Conclusion

I built a working lab, documented how its systems communicate, established a baseline, performed controlled activities and located the evidence in logs and network traffic. The main lesson is that visibility matters: some actions are logged clearly, others (like permission changes) are easy to miss, and a VPN can hide traffic from a capture. Next week the logs will be centralized to close those gaps.

---

## Problems faced and solutions

| # | Problem | Solution | Lesson |
|---|---|---|---|
| 1 | VirtualBox network settings were not where expected | Used Settings > Network > Adapter 1 > Bridged Adapter for Kali | Menus move between versions |
| 2 | "Advanced" adapter option not visible | Expanded it, or skipped it (only needed for capturing others' traffic) | Not every setting is required |
| 3 | Windows ISO would not download | Used the host Windows laptop as the endpoint and documented it | Document limitations honestly |
| 4 | Ubuntu was not installed as a VM | Used Ubuntu on WSL2. A dedicated VM is planned | Plan the SIEM server early |
| 5 | `CommandNotFoundException` for Linux commands | Linux commands go in Ubuntu, not Windows PowerShell | Check the prompt first |
| 6 | `sudo` / `wsl` "not found" | Opened Ubuntu with `wsl -d Ubuntu`. `wsl` commands run only in PowerShell | Each command belongs to one world |
| 7 | A wrong Linux (bare shell, "Docker Desktop") opened | Listed installs with `wsl -l -v` and opened Ubuntu explicitly | Verify which distribution is open |
| 8 | Windows did not answer ping | Added an ICMP allow firewall rule | Firewalls block by default |
| 9 | Ping from Kali never stopped | Ctrl+C, or use `ping -c 4` | Linux ping runs until stopped |
| 10 | "Permission denied" listing `/srv/teamfiles` | Expected under mode 770. Used `sudo ls` | A denial is evidence the permission works |
| 11 | Screenshots would not capture | Win+Shift+S, Snipping Tool, or pasted text output | Text output is valid evidence if explained |
| 12 | Typed `suexit` as one word | `su - alice` and `exit` on separate lines | One command per line |
| 13 | `chown` failed (missing slash in the path) | Retyped `/srv/teamfiles` | Read the error: it names the failing path |
| 14 | `ipconfig` showed VPN and VirtualBox adapters | Used the adapter with a default gateway and labelled the others | Extra virtual adapters are normal |
| 15 | Windows commands typed in Ubuntu | `exit`, then re-ran in PowerShell | The shell decides which commands exist |
| 16 | Chat pop-ups inside screenshots | Blacked them out. Use Do Not Disturb | Evidence should not contain personal data |
| 17 | Kali pinged its own address, not Windows | Re-ran the ping to 10.60.174.211 (Figure 3.3) | Check the target IP against the inventory |
| 18 | Wireshark `tls` filter showed 0 packets | Proton VPN encrypts traffic in a tunnel. Turn it off or capture on the VPN adapter | A VPN hides traffic from the physical adapter |
| 19 | Wireshark filter bar turned red / filter not applied | Replaced the `<kali-ip>` placeholder and pressed Enter | Replace placeholders. A green bar is valid |
| 20 | Nmap "command not found" and first scan hit the VPN address | Removed the dash, used sudo, scanned 10.60.174.211 | Scan the address reachable on the shared network |
| 21 | Kali timestamps use -0400 while Windows and Ubuntu use +01:00 | Converted the times. Better: set the same time zone | Consistent time zones matter in a SIEM |

---

*Educational project for the CCF SOC Accelerator Bootcamp. The screenshots contain the lab hostname, a Windows username, MAC addresses and private IP addresses. Keep the repository private, or blur those details, before making it public.*
