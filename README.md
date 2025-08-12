# Homelab---Enterprise-101
Homelab project built by following Enterprise 101 guide. Configured Active Directory, Wazuh (SIEM), MailHog for simulated enterprise defense, and simulation attacks for offense, with troubleshooting and documentation.

---

#Topology
| Hostname           | IP         | OS                     | Role |
|--------------------|------------|------------------------|-------------------------------|
| project-1-dc       | 10.0.0.5   | Windows Server 2025    | AD DS / DNS / DHCP / SSO      |
| project-1-win      | 10.0.0.100 | Windows 11 Enterprise  | Windows workstation           |
| project-1-linux    | 10.0.0.8   | Ubuntu 22.04           | Linux workstation             |
| project-1-admin    | 10.0.0.101 | Ubuntu 22.04 Desktop   | Corporate server (apps/files) |
| project-1-sec-work | 10.0.0.103 | Security Onion (opt.)  | Security workstation          |
| project-1-sec-box  | 10.0.0.10  | Ubuntu 22.04           | Wazuh manager + dashboard     |
| attacker           | DHCP       | Kali 2024.2            | Attack box                    |

---

# VirtualBox NAT: `Project-1-nat` (10.0.0.0/24), DHCP scope 10.0.0.100–10.0.0.200

---

# Tools
Defensive: Active Directory, Wazuh, MailHog  
Offensive: Nmap, Hydra, NetExec, Evil-WinRM, xfreerdp, SecLists

---

# Provisioning + Configurations + how I verified

Workstations: 
1. Windows 11 (project-1-win) — [setup guide](https://docs.projectsecurity.io/e101/setupwindows/#connect-windows-11-enterprise-project-x-win-client-to-windows-domain-controller)
- Configured to join CORP.PROJECT1-DC.COM domain.
- Installed & enrolled Wazuh agent in project-1-sec-box (10.0.0.10)
- Enabled audits for logon/logoffs, process creation + cmdline
- Verified: Event Viewer shows 4625/4688; agent green in Wazuh

2. Ubuntu Desktop (project-1-linux) — [setup guide](https://docs.projectsecurity.io/e101/setupubuntudesktop/)
- Installed & enrolled Wazuh agent '10.0.0.10'
- Wazuh ingests SSH failures from /var/log/auth.log
- Verified: SSH failures in `/var/log/auth.log`; FIM alert in Wazuh

3. Security Onion (project-1-sec-work) — [setup guide](https://docs.projectsecurity.io/e101/setupsecurityonion/)
- Deployed as analysis box for triage
- Verified: able to review lab traffic/logs when used

4. Kali (attacker) — [setup guide](https://docs.projectsecurity.io/e101/setupattacker/)
- Offensive Tools: `nmap`, `hydra`, `netexec`, `evil-winrm`, `xfreerdp`, `seclists`
- Verified: generated SSH/WinRM activity → detections in Wazuh

---

Servers:
1. Domain Controller (Windows Server 2025) — [building AD](https://docs.projectsecurity.io/e101/buildingad/)
- Set up as CORP.PROJECT1-DC.COM alongside DNS/DHCP
- Creation of OUs for Users/Computers/Servers
- Installed & enrolled server to Wazuh in project-1-sec-box (10.0.0.10)
- Verified: ADUC shows joined hosts; Event IDs 4624/4625/4688 present

2. Corporate Server (project-1-admin) — [setup](https://docs.projectsecurity.io/e101/setupcorporateserver/)
- Use of static IP 10.0.0.101; baseline hardening with OpenSSH
- Hosted Internal service for files/apps, added FIM to monitor secret.txt path and activities generated for logs
- Enrolled into Wazuh and added FIM watch for sensitive path
- Verified: reachable from clients; events visible in Wazuh

3. Security Server (project-1-sec-box) — [Wazuh setup](https://docs.projectsecurity.io/e101/toolguide/setupwazuh/)
- Installed Wazuh indexer/manager/dashboard on `10.0.0.10`
- Enrolled Windows + Linux agents; added simple detections and FIM on `secret.txt`
- Verified: API responds, rule `5760` fires, events in wazuh-alerts

---

# Creating Vulnerable Environment — [guide](https://docs.projectsecurity.io/e101/configurevulnenv/)
- Enabled RDP/WinRM/SSH and a basic SMB share to create an attack surface
- Configured account lockout and simple password policy to allow spraying attacks
- Installed and enrolled Wazuh agent for each host and added FIM to detect changes on a file named `secret.txt`
- Ports WinRM 5985/5986, RDP 3389, and SSH 22 opened for each box

---

# Attack Simulation — [guide](https://docs.projectsecurity.io/e101/cyberattacksimulation/)
- Recon by using Nmap service/port scan to find SSH/WinRM/RDP that were exposed
- Initial access by Phishing credentials that were captured to MailHog and used them to SSH'd into the corporate server
- Lateral movement by Password spray over WinRM, then Evil-WinRM shell into the Windows client where I pivot to the Domain Controller via RDP
- Exfiltration: Copied a file titled `secret.txt` off the Domain Controller
- Persistence by adding a local admin and a scheduled task that triggers a reverse shell
- Detection by Wazuh that alerted SSH-auth-fails, WinRM logon events, Windows logon/process creation and FIM changes

---

# Detections Implemented (Wazuh)
- SSH authentication failures → (rule 5760 / MITRE T1110.001)
- WinRM logon activity → searchable in `wazuh-alerts`
- File Integrity Monitoring → `secret.txt` 

---

# Results of Attack Simulation
- 3 agents reporting (Win11, Win Server 2025, Ubuntu) to Wazuh  
- SSH auth-fail detections (rule 5760 / MITRE T1110.001)  
- FIM alert triggered on `secret.txt` after modification

---

# Evidence
- SSH auth-fail detected (rule 5760, MITRE T1110.001)
<img width="926" height="643" alt="alert-detail rule-5760" src="https://github.com/user-attachments/assets/76abba69-e9f9-4e54-8222-cf0de270080a" />

- 3 agents active; events flowing to Wazuh.
<img width="1179" height="550" alt="wazuh-overview-3-agents" src="https://github.com/user-attachments/assets/57fe661f-34d6-4c87-9c7a-7f86de5ffd7d" />

- Windows security events indexed in `wazuh-alerts-`.
<img width="1203" height="650" alt="discover-raw-event" src="https://github.com/user-attachments/assets/e95477d1-b763-4226-8866-1fdb4a7f9786" />

---

# Challenges and Troubleshooting:
1. NAT Network Boot Delays and Login Freezes on Linux Servers:
VMs were black screening or became frozen forever on the login screen and/or on NAT
Network. I tried troubleshooting with different adapters by checking enp0s3 and enp0s8,
compared against a clone and swapped the display manager to LightDM which made the server
boot up fast and solved the issue

2. Clone worked, original did not:
The cloned VM worked perfectly fine but the original did not. I did a side-by-side comparison of
configurations and found that display manager/boot services were completely different and
matched them which made the original stable

3. Postfix “connection refused”:
Mail to the Linux user would bounce. I added a virtual_alias_maps and reloaded the Postfix and
tested internally where it delivered the mail

4. Wazuh agent stuck “pending”:
Agent showed up but was never fully registered. I re-enrolled it and checked ports
(1514/1515/55000), I tried NAT vs NAT Network. I ran on NAT until network issues were sorted

6. MailHog would not load (IP conflict):
Two of my VMs were sharing the same IP (misconfiguration). I cleaned up the netplan and gave
everything unique statics, afterwards MailHog’s UI worked

7. Hydra found credentials but SSH was closed:
Using Hydra during the attack simulation and it worked properly but corp-svr SSH was disabled.
I enabled sshd and tried applying the credentials again in my attack box and was logged in

8. Phishing test delivery issue:
The phishing email never hit the inbox. I checked and fixed the mail script, checked logs, and
tested SMTP by hand. The root cause was the NAT/NAT Network. Afterwards Postfix and
network fixes the phishing email hit the inbox.

---

# Lessons learned:
While following the step by step guide for this homelab this was one of the most valuable
experiences where I learned so many things. Setting up and provisioning multiple workstations
and servers where I had to implement Active Directory, email services, Wazuh SIEM, and even
simulating an attack simulation through the environment. Throughout the project there were
plenty of issues I encountered and had to resolve a range of problems that taught me technical
and workflow.

Breaking problems down – Where I fix black screens on the VMs, network freezes, and services
failure issues taught me to not configure and change so many variables at the same time but
constantly configuring one variable at a time and testing after each step helped me break down
and pinpoint where issues were happening

Network Basics – Understanding of NAT and NAT Network, DNS and IP configs were so
important when it came to being able to get each machine to talk
Documenting everything – This was crucial where keeping track of commands and fixes that
saved a lot of time as the same issues did resurface not just one time

Verify – Always test end-to-end, and not just that the service is running
Staying patient and persistent – When I started this project I did not expect it to take as long as
it did. So many complex issues like Postfix mail routing and Wazuh agents visibility took a lot of
time and rechecking of logs and configs.

By the end of the project I stayed persistent and as patient as possible even if one issue took
me hours to troubleshoot and recover. I can confidently set up enterprise tools, workstations,
and servers through VMs like Wazuh, Postfix, Active Directory and even learned how to utilize
defensive and offensive tools during the attack simulation.

---

# References (Guide by Grant Collins)
1. AD build: https://docs.projectsecurity.io/e101/buildingad/

2. Windows client: https://docs.projectsecurity.io/e101/setupwindows/

3. Ubuntu desktop: https://docs.projectsecurity.io/e101/setupubuntudesktop/

4. Security server (Wazuh): https://docs.projectsecurity.io/e101/secserversetupubuntudesktop/

5. Corporate server: https://docs.projectsecurity.io/e101/setupcorporateserver/

6. Attacker (Kali): https://docs.projectsecurity.io/e101/setupattacker/

7. Vulnerable config: https://docs.projectsecurity.io/e101/configurevulnenv/

8. Attack simulation: https://docs.projectsecurity.io/e101/cyberattacksimulation/

9. Tool guides: Wazuh https://docs.projectsecurity.io/e101/toolguide/setupwazuh/ • MailHog https://docs.projectsecurity.io/e101/toolguide/setupmailhog/
