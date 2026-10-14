# Cybersecurity Generalist

Hi, I'm Harold. I'm building a career in cybersecurity with a focus on **Log analysis, security monitoring, endpoint security, access maangement, and cloud security**. This repo collects the hands-on projects I complete as I train, each with a write-up of my analysis, my decisions, and what I learned.

## Certifications and Training
- Google Cyber Security Certificate
- TryHackMe Pre Security
- CompTIA Security+ (in progress)

## Projects

| Project | Summary | Skills |
|---|---|---|
| [TryHackMe SOC Simulator: Simulated Attack Threats and Detection Engineering](tryhackme-soc-simulation/) | Defended a simulated network through six attack stages, moving up the Pyramid of Pain from blocking a file hash to detecting automated data exfiltration | Malware sandbox analysis, IOC management, firewall and DNS filtering, Sigma rule writing, Sysmon log analysis, MITRE ATT&CK mapping |

More projects coming as I complete them.

## Featured: TryHackMe SOC Simulator

I played a SOC analyst facing an attacker who adapted after every block. Each stage required a stronger kind of detection:

| Stage | Attacker action | My response |
|---|---|---|
| 1 | Delivered malware | Analyzed it in a sandbox and blocked its file hash |
| 2 | Suspicious request to a remote server | Blocked the IP on the firewall |
| 3 | Command execution request to a domain | Blocked the domain with DNS filtering |
| 4 | Disabled host real-time protection via the registry | Wrote a Sigma rule for the registry change |
| 5 | Beaconed to an external server every 30 minutes | Wrote a Sigma rule for the beacon pattern |
| 6 | Automated data exfiltration | Wrote a Sigma rule for the staging file |

**Key lesson:** blocking low-level indicators like hashes, IPs, and domains only slows an attacker down. Detecting behavior forces them to change how they operate.

[Read the full write-up](tryhackme-soc-simulation/)

## Repo Structure

```
Cybersec_projects/
├── README.md
└── tryhackme-soc-simulation/
    ├── README.md
    └── images/
```

## About
Each project folder has its own README covering the scenario, my analysis, the actions I took, and takeaways. Write-ups describe my process and do not include task flags or answer keys.
