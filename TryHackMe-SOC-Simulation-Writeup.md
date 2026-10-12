# TryHackMe SOC Simulator: Simulated Attack Threats and Detection Engineering

A six-part write-up of a TryHackMe SOC Simulator scenario in which I defended a simulated network against an attacker who adapted after every block. Each part moved one level up the **Pyramid of Pain**, from blocking a single file hash to detecting the attacker's techniques.

## Overview

| | |
|---|---|
| **Platform** | TryHackMe SOC Simulator (PicoSecure console) |
| **Role** | SOC analyst (detection and response) |
| **Tools used** | Malware Sandbox, Manage Hashes, Firewall Manager, DNS Filter, Sigma Rule Builder |
| **Frameworks** | MITRE ATT&CK, Pyramid of Pain |
| **Outcome** | All six attack stages blocked or detected; the simulation confirmed the threat was contained |

## Attack Timeline and Response

| Part | Attacker action | My response | Control used |
|---|---|---|---|
| 1 | Delivered malware (`sample1.exe`) | Analyzed it in the sandbox and blocked its hash | Hash blocklist |
| 2 | Suspicious HTTP GET request to a remote server | Blocked the IP in both directions | Firewall rules |
| 3 | HTTP request carrying an execution command to a domain server | Blocked the domain | DNS Filter blocklist |
| 4 | Modified the host registry to disable real-time protection (`sample4.exe`) | Wrote a detection rule for the registry change | Sigma rule (registry modifications) |
| 5 | Beaconed to an external server every 30 minutes | Wrote a detection rule for the beacon pattern | Sigma rule (network connections) |
| 6 | Automated data exfiltration to a staging file | Wrote a detection rule for the staging file | Sigma rule (file creation and modification) |

## The Simulation Through the Pyramid of Pain

The Pyramid of Pain ranks indicators by how much it hurts the attacker when defenders block them. The scenario climbed it one level at a time.

| Part | Level | What I blocked or detected | Difficulty for the attacker to change |
|---|---|---|---|
| 1 | File hashes | `sample1.exe` | Trivial: change one bit of the file |
| 2 | IP addresses | `154.35.10.113:4444` | Easy: use a new server |
| 3 | Domain names | [TODO: domain] | Simple: register a new domain |
| 4 | Host artifacts | Registry change disabling Defender real-time protection | Annoying: must change how the malware operates |
| 5 | Tools | C2 beaconing to `51.102.10.19` | Challenging: must replace or rework the tool |
| 6 | TTPs | Automated exfiltration (T1020) | Tough: must change their entire approach |

---

## Part 1: Malware Analysis and Hash Blocking

**Summary:** I analyzed a suspicious executable in the lab's malware sandbox, extracted its hashes, and added the malicious hash to the blocklist. The file was prevented from executing and no accounts were compromised.

**Analysis of `sample1.exe`**

| Field | Value |
|---|---|
| Size | 202.50 KB |
| Type | PE32+ executable (GUI), x86-64, MS Windows |
| Sandbox tag | Trojan.Metasploit.A |

- **Malicious:** Metasploit was detected (PID 2492)
- **Suspicious:** Connects to an unusual port
- **Informational:** Reads the machine GUID from the registry, checks LSA protection, reads the computer name, checks supported languages

**Reasoning:** A Metasploit detection plus a connection to an unusual port is consistent with a payload calling back to an attacker. The registry and system information checks are typical reconnaissance.

**Indicators of compromise**
- MD5: `cbda8ae000aa9cbe7c8b982bae006c2a`
- SHA1: `83d2791ca93e58688598485aa62597c0ebbf7610`
- SHA256: `9c550591a25c6228cb7d74d970d133d75c961ffed2ef7180144859cc09efca8c`

**Containment:** In **Manage Hashes**, I selected SHA256, entered the hash, and submitted it, which added it to the Hash Blocklist and updated the EDR detection signatures.

**Result:** The platform confirmed `sample1.exe` was prevented from executing.

![Malware Sandbox menu](images/01-sandbox-menu.png)
![Sandbox analysis results](images/02-sandbox-results.png)
![Hash blocklist](images/03-hash-blocklist.png)

---

## Part 2: Suspicious Network Activity and IP Blocking

**Summary:** After the malware was blocked, new network activity appeared, including a suspicious HTTP GET request. I blocked the remote IP in both directions on the firewall. The attacker did not stop and began looking for other ways in.

**Identifying the activity**
- **Request:** `GET http://154.35.10.113:4444/uvLk8YI32`
- **Remote server:** `154.35.10.113:4444`
- **Why it was suspicious:** [TODO: 1-2 reasons, e.g. unfamiliar external IP, random-looking URL path, timing right after the malware was blocked]

Port 4444 is the default listener port for Metasploit, which fits the Metasploit detection in Part 1.

**Containment:** In **Firewall Manager**, I created two rules:
- **Inbound:** Block traffic from the IP into the network
- **Outbound:** Block traffic from the network to the IP

**Result:** No accounts were exposed or exploited.

![Network traffic showing the GET request](images/04-network-traffic.png)
![Firewall rules](images/05-firewall-rules.png)

---

## Part 3: Command Execution Request and Domain Blocking

**Summary:** Further analysis showed an HTTP request carrying an execution command sent to a domain server, which pointed to command-and-control (C2) behavior. I added the domain to the DNS Filter blocklist.

**Identifying the request**
- **Domain:** [TODO: domain name]
- **Request:** [TODO: method, URL, and the command or parameter you saw]
- **Why it was malicious:** [TODO: e.g. unfamiliar domain, request carried a command, followed the earlier malware and C2 traffic]

**Containment:** In **DNS Filter**, I added the domain to the blocklist so systems on the network can no longer resolve it. Domain blocks last longer than IP blocks, because attackers can change IPs cheaply but must register a new domain to return.

**Result:** [TODO: confirmation message shown after the block]

![DNS Filter blocklist](images/06-dns-filter.png)

---

## Part 4: Host Tampering and Sigma Rule Detection

**Summary:** The attacker shifted to the host itself. I scanned `sample4.exe` in the sandbox and found it modifying the registry to disable real-time protection. I built a Sigma rule to detect that registry change.

**Analysis**

| Field | Value |
|---|---|
| Registry key | `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows Defender\Real-Time Protection` |
| Value name | `DisableRealtimeMonitoring` |
| Value | [TODO: value from the scan results] |

**Reasoning:** `DisableRealtimeMonitoring` turns off Windows Defender's real-time scanning. Malware sets it so later payloads run undetected, meaning the attacker was trying to blind the host's defenses.

**Detection rule:** In **Sigma Rule Builder**, I selected **Sysmon Event Logs**, then **Registry Modifications**, entered the registry key, name, and value from the scan results, and set the technique type to **Defense Evasion**. The rule catches this change no matter which file makes it.

**MITRE ATT&CK:** Defense Evasion (tactic); Impair Defenses: Disable or Modify Tools (T1562.001)

**Result:** [TODO: confirmation message after the rule was saved]

![Sandbox results for sample4.exe](images/07-sample4-results.png)
![Sigma rule configuration](images/08-sigma-rule.png)

---

## Part 5: C2 Beaconing Detection

**Summary:** The malware was calling out to an external server on a fixed timer, waiting for instructions. I identified the beaconing pattern in the connection logs and built a Sigma rule on network connections to detect and block it.

**Identifying the beaconing**

| Field | Value |
|---|---|
| Internal host | `10.10.15.12` |
| Remote IP | `51.102.10.19` |
| Remote port | 443 |
| Interval | Every 30 minutes (at :00 and :30 past each hour) |
| Byte size | [TODO: size from the Size column] |

**Reasoning:** Normal user activity is irregular, while this traffic was perfectly timed. This steady check-in is beaconing: the malware connects out, reports in, and waits for commands. Because the connection starts inside the network and goes outward, it can slip past defenses that mainly watch inbound traffic.

**Detection rule:** In **Sigma Rule Builder**, I selected **Network Connections** and set conditions on remote IP (`51.102.10.19`), remote port [TODO], byte size [TODO], and frequency [TODO]. Combining conditions matches the pattern of the beacon rather than just a destination, which reduces false positives.

**MITRE ATT&CK:** Command and Control (TA0011)

**Result:** [TODO: confirmation message after the rule was saved]

![Network log showing 30-minute beaconing](images/09-beaconing-log.png)
![Sigma network connection rule](images/10-sigma-network-rule.png)

---

## Part 6: Automated Data Exfiltration

**Summary:** In the final stage, the attacker tried to steal data. I found evidence of automated exfiltration and built a Sigma rule on file creation and modification to block it. The simulation confirmed the threat was contained.

**Analysis:** `commands.log` showed commands being run automatically, with their output written to `exfiltr8.log`. The repeated, automated pattern is consistent with **Automated Exfiltration (T1020)**: data is collected and staged in a file for transfer out of the network.

**Detection rule:** In **Sigma Rule Builder**, I selected **File Creation and Modification** and set:
- **File path:** `%TEMP%` [TODO: confirm the path you entered]
- **File name:** `exfiltr8.log`
- **MITRE ATT&CK:** Exfiltration (TA0010)

**Result:** The simulation confirmed the threat was contained.

![commands.log and exfiltr8.log](images/11-exfil-logs.png)
![Sigma file rule](images/12-sigma-file-rule.png)

---

## Indicators of Compromise Summary

| Type | Indicator | Response |
|---|---|---|
| File (SHA256) | `9c550591a25c6228cb7d74d970d133d75c961ffed2ef7180144859cc09efca8c` | Hash blocklist |
| File (MD5) | `cbda8ae000aa9cbe7c8b982bae006c2a` | Identified in sandbox report |
| IP:port | `154.35.10.113:4444` | Firewall (inbound and outbound) |
| Domain | [TODO: domain] | DNS Filter blocklist |
| Registry | `...\Windows Defender\Real-Time Protection` / `DisableRealtimeMonitoring` | Sigma rule |
| Beacon | `51.102.10.19:443`, every 30 minutes | Sigma rule |
| File | `exfiltr8.log` in `%TEMP%` | Sigma rule |

## MITRE ATT&CK Mapping

| Part | Tactic | Technique |
|---|---|---|
| 3 | Command and Control | Command execution request over HTTP |
| 4 | Defense Evasion | Impair Defenses: Disable or Modify Tools (T1562.001) |
| 5 | Command and Control (TA0011) | Beaconing |
| 6 | Exfiltration (TA0010) | Automated Exfiltration (T1020) |

## Key Takeaways

- **Blocking low-level indicators is a stopgap.** Each time I blocked a hash, IP, or domain, the attacker simply moved on. They had to change their approach only once I detected behavior.
- **Layered controls matter.** The attack was stopped by hash blocking, firewall rules, DNS filtering, and several detection rules, so the attacker had to defeat each in turn.
- **Hash blocking is precise but fragile.** Changing a single bit of a file produces a new hash.
- **Patterns beat single events.** Beaconing is hard to spot in one log line but obvious as a pattern of timing, size, and destination.
- **Exfiltration is the real goal.** Everything earlier positions the attacker to steal data, so detecting it is the last line of defense.
- [TODO: add one takeaway in your own words, such as something that surprised you or a skill you want to improve]

## Skills Demonstrated

Malware sandbox analysis, IOC extraction and management, hash-based detection, firewall rule creation, DNS filtering, Sigma detection rule writing, Sysmon log analysis, C2 beaconing recognition, exfiltration detection, MITRE ATT&CK mapping, Pyramid of Pain analysis, incident documentation

## Note

This write-up covers a training simulation. It describes my analysis process and decisions and does not include task flags or answer keys.
