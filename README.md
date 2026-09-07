# INCIDENT-DETECTION-AUTOMATED-MITIGATION-
 Real-time SSH brute-force detection and automated containment using Wazuh SIEM/SOAR — active response, alert correlation, and post-incident hardening.
# SSH Brute-Force Detection & Automated Containment — Wazuh SIEM/SOAR

Incident response case study documenting end-to-end detection, automated mitigation, and verification of a real-time SSH brute-force attack using **Wazuh SIEM** and its **Active Response (SOAR)** framework.

---

## 1. Executive Summary

During continuous security monitoring with Wazuh SIEM, an automated high-severity alert was raised indicating an active SSH brute-force attack against an internal Ubuntu server (`ubuntu-target`). The threat actor targeted the system using **Hydra** from a remote attack host.

Upon alert triggering, Wazuh's Active Response engine executed an automated containment policy (`host-deny`), dropping and blocking the malicious IP address at the host access-control layer (`/etc/hosts.deny`) and preventing further unauthorized access attempts.

---

## 2. Attack Vector

- **Target:** SSH service (port 22) on `ubuntu-target`
- **Attack type:** Multi-threaded credential brute-force attack
- **Tooling:** Kali Linux running Hydra
  ```
  hydra -l analyst -P rockyou.txt ssh://<target_ip> -t 4
  ```
- **Method:** Dictionary-based password guessing against a known/guessed username, using 4 parallel threads

---

## 3. Timeline of Events

| Time (UTC) | Event |
|---|---|
| 11:16:50 | Wazuh Agent lifecycle reloaded and re-established connection with `wazuh-server` |
| T+x | Attacker initiated multi-threaded SSH brute-force via Hydra |
| T+x | Wazuh Manager correlated multiple failed authentication attempts within a short interval, matching **Rule ID 5763** |
| T+x | Manager dispatched Active Response instruction (`<location>agent</location>`) to `ubuntu-target` |
| T+x | Agent executed the `host-deny` response script, adding attacker IP to `/etc/hosts.deny` |
| T+x | Subsequent connection attempts from the attacker host were denied |

---

## 4. Detection

Wazuh's rule correlation engine flagged the anomalous authentication pattern:

- **Rule ID:** 5763 — repeated SSH authentication failures within a defined time threshold
- **Severity:** High
- **Trigger condition:** Multiple failed logins from a single source IP exceeding the correlation threshold, consistent with brute-force behavior
- **Outcome:** Alert automatically escalated to the Active Response pipeline for automated remediation

---

## 5. Automated Mitigation (Active Response)

Wazuh's SOAR-style Active Response module handled containment without manual intervention:

1. Manager issued an Active Response command scoped to the affected agent (`<location>agent</location>`)
2. Agent invoked the `host-deny` response script
3. Script appended the attacker's IP to `/etc/hosts.deny` (TCP Wrappers-based access control)
4. All further connection attempts from that IP were dropped at the host level before reaching the SSH daemon

---

## 6. Technical Analysis & Configuration Verification

### A. Manager Configuration — `/var/ossec/etc/ossec.conf`
- Verified the `active-response` block is present and correctly scoped to the brute-force rule group
- Confirmed command binding to the `host-deny` script

### B. Agent Configuration — `ubuntu-target`
- **Active Response status:** Confirmed enabled — `<disabled>no</disabled>` in `/var/ossec/etc/ossec.conf`
- **Execution logs:** Reviewed `/var/ossec/logs/active-responses.log`, confirming:
  - Agent lifecycle communication (`restart.sh agent reload`)
  - Script execution timestamped to the rule trigger event

---

## 7. Containment & Verification Strategy

| Check | Method | Result |
|---|---|---|
| ACL entry | `cat /etc/hosts.deny` | `ALL: <attacker_ip>` present |
| Network reachability (from attacker host) | `ping -c 3 <target_ip>` | 100% packet loss |
| Service reachability (from attacker host) | `ssh analyst@<target_ip>` | Connection refused |

These checks confirm the attacker host was fully isolated from the target at the network and service layer following containment.

---

## 8. Post-Incident Hardening Recommendations

1. **Enforce SSH key-based authentication**
   Set `PasswordAuthentication no` in `/etc/ssh/sshd_config` across production nodes to eliminate password-guessing as a viable vector.

2. **Extend Active Response ban duration**
   Increase the timeout from the default `600` seconds (10 min) to `86400` seconds (24 hr) for repeat offenders to reduce re-attack windows.

3. **SSH port obfuscation**
   Move the SSH listener off the default port 22 to reduce exposure to automated/opportunistic scanning.

4. **Credential audit**
   Verify the `analyst` account's credentials were not compromised prior to containment; rotate credentials as a precaution.

---

## Stack

`Wazuh SIEM/XDR` · `Active Response (SOAR)` · `TCP Wrappers (/etc/hosts.deny)` · `Hydra` · `Kali Linux` · `Ubuntu Server`

---

## Disclaimer

This is a controlled lab/test environment exercise for security monitoring and incident-response training purposes. IP addresses have been redacted/generalized.
