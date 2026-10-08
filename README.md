# TryHackMe SOC Simulator: Malware, C2 Traffic, and Domain Blocking

A hands-on write-up of the **Simulated Attack Threats and Detection Engineering** scenario in TryHackMe's SOC Simulator. I played a SOC analyst defending a simulated network against an attacker who adapted after every block.

## Overview

| | |
|---|---|
| **Platform** | TryHackMe SOC Simulator (PicoSecure console) |
| **Role** | SOC analyst (detection and response) |
| **Focus** | Malware analysis, IOC management, network containment, DNS filtering |
| **Outcome** | All three attack stages blocked; no accounts compromised |

## Attack Timeline and Response

| Part | What the attacker did | What I did | Control used |
|---|---|---|---|
| 1 | Delivered a malicious executable (`sample1.exe`) | Analyzed it in the sandbox and blocked its hash | Hash blocklist |
| 2 | Sent a suspicious HTTP GET request to a remote server, then kept looking for other ways in | Blocked the IP in both directions | Firewall rules |
| 3 | Sent an HTTP request carrying an execution command to a domain server | Blocked the domain | DNS Filter blocklist |

---

## Part 1: Malware Analysis and Hash Blocking

### Summary
I analyzed a suspicious executable in the lab's built-in malware sandbox, extracted its file hashes, and added the malicious hash to the blocklist. This stopped the file from executing, and no accounts were compromised.

### Environment
- **Modules used:** Malware Sandbox, Manage Hashes (IOC Management)
- **Sample analyzed:** `sample1.exe`

### Analysis
I selected `sample1.exe` in the sandbox and submitted it for analysis.

| Field | Value |
|---|---|
| File name | `sample1.exe` |
| Size | 202.50 KB |
| Type | PE32+ executable (GUI), x86-64, MS Windows |
| Sandbox tag | Trojan.Metasploit.A |

**Behavior analysis**
- **Malicious:** Metasploit was detected (PID 2492)
- **Suspicious:** Connects to an unusual port
- **Informational:** Reads the machine GUID from the registry, checks LSA protection, reads the computer name, checks supported languages

**Reasoning:** A Metasploit detection plus a connection to an unusual port is consistent with a payload trying to call back to an attacker. The registry, LSA, and system information checks are typical reconnaissance. I treated the file as malicious.

### Indicators of compromise
- **MD5:** `cbda8ae000aa9cbe7c8b982bae006c2a`
- **SHA1:** `83d2791ca93e58688598485aa62597c0ebbf7610`
- **SHA256:** `9c550591a25c6228cb7d74d970d133d75c961ffed2ef7180144859cc09efca8c`

### Containment
In **Manage Hashes**, I selected SHA256, pasted the hash value, and submitted it. The hash was added to the Hash Blocklist, which updates the EDR detection signatures.

### Result
The platform confirmed `sample1.exe` was prevented from executing. No accounts were compromised.

![Malware Sandbox menu](images/01-sandbox-menu.png)
![Sandbox analysis results](images/02-sandbox-results.png)
![Hash blocklist](images/03-hash-blocklist.png)

---

## Part 2: Suspicious Network Activity and IP Blocking

### Summary
After the malicious file was blocked, new network activity appeared, including a suspicious HTTP GET request. I blocked the remote IP in both directions on the firewall. The attacker did not stop and began looking for other ways in, which showed that a single block is containment, not resolution.

### Environment
- **Modules used:** Firewall Manager, Network traffic viewer

### Identifying the suspicious activity
- **Request:** `GET http://154.35.10.113:4444/uvLk8YI32`
- **Remote server:** `154.35.10.113:4444`
- **Why it was suspicious:** [TODO: add 1-2 reasons, e.g. unfamiliar external IP, random-looking URL path, timing right after the malware was blocked]

Port 4444 is the default listener port for Metasploit, which fits the Metasploit detection in Part 1. I treated the IP as attacker infrastructure.

### Containment
In **Firewall Manager**, I created two rules for the IP:
- **Inbound rule:** Block traffic from the IP into the network
- **Outbound rule:** Block traffic from the network to the IP

Blocking both directions stops internal systems from reaching the server and stops the server from reaching in.

### The attacker adapts
The attacker kept going after the IP was blocked and began trying other ways in. [TODO: one or two sentences on what you saw next, which leads into Part 3.]

### Result
No accounts were exposed or exploited.

![Network traffic showing the GET request](images/04-network-traffic.png)
![Firewall rules](images/05-firewall-rules.png)

---

## Part 3: Command Execution Request and Domain Blocking

### Summary
After further analysis, I found an HTTP request carrying an execution command sent to a domain server. This pointed to command-and-control (C2) behavior, so I added the domain to the DNS Filter blocklist.

### Environment
- **Modules used:** [TODO: traffic or log view you used], DNS Filter

### Identifying the malicious request
- **Domain:** [TODO: domain name]
- **Request:** [TODO: method, URL, and the command or parameter you saw]
- **Source host:** [TODO: internal host or IP, if shown]
- **Why it was malicious:** [TODO: e.g. unfamiliar domain, request carried a command, followed the earlier malware and C2 traffic]

### Containment
In **DNS Filter**, I added the domain to the blocklist. Systems in the network can no longer resolve it, so they can't reach the attacker's server.

Blocking at the domain level lasts longer than blocking an IP. Attackers can change IP addresses cheaply, but they have to register a new domain to get back in.

### Result
[TODO: confirmation message shown after the block, and whether any accounts were compromised.]

![DNS Filter blocklist](images/06-dns-filter.png)

---

## Indicators of Compromise Summary

| Type | Indicator | Blocked with |
|---|---|---|
| File (SHA256) | `9c550591a25c6228cb7d74d970d133d75c961ffed2ef7180144859cc09efca8c` | Hash blocklist |
| File (MD5) | `cbda8ae000aa9cbe7c8b982bae006c2a` | (identified in sandbox report) |
| IP:port | `154.35.10.113:4444` | Firewall (inbound and outbound) |
| Domain | [TODO: domain] | DNS Filter |

## Key Takeaways

- **Layered defenses matter.** Each stage was stopped by a different control: file hash, firewall, and DNS. The attacker had to get around all of them.
- **Hash blocking is precise but fragile.** It stops that exact file, but changing a single bit of the file produces a new hash. The scenario's own follow-up message made this point, which is why network and DNS controls were needed next.
- **Attackers adapt.** Blocking one indicator is a stopgap. Response has to continue with monitoring and new blocks as the attacker changes tactics.
- **Domain blocks outlast IP blocks.** IPs are cheap to replace, domains are not.
- [TODO: add one takeaway in your own words, such as something that surprised you or a skill you want to improve.]

## Skills Demonstrated

Malware sandbox analysis, IOC extraction and management, hash-based detection, firewall rule creation, DNS filtering, C2 traffic recognition, incident documentation

## Note

This write-up covers a training simulation. It describes my analysis process and decisions and does not include task flags or answer keys.
