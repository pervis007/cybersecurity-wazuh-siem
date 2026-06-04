# 🔍 Wazuh SIEM — Detection & Monitoring Lab

A working **Wazuh SIEM** deployment built inside my isolated [cybersecurity homelab ("Namek")](https://github.com/pervis007/cybersecurity-homelab). An **Ubuntu Server (Goku)** runs the full Wazuh stack as the central manager, with agents deployed to a **Kali Linux** attacker box and a **Windows 11** target — giving me end-to-end visibility to detect and investigate attacks as they happen.

> Built and maintained by **Pervis Harden** — Cybersecurity Analyst · Ethical Hacker · Orlando, FL
> 🔗 Portfolio: [pervis007.github.io](https://pervis007.github.io) · ✉️ p.harden32@gmail.com

---

## 🖥️ The Server (Goku — Ubuntu Server)

The Ubuntu Server host (**Goku**) is the brain of the SIEM. It runs the complete Wazuh **all-in-one** stack — the **manager** (analysis + rule engine), the **indexer** (data storage/search), and the **dashboard** (web UI). Agents on the other machines report into it, and all detection, correlation, and alerting happens here.

Confirming the install on the server:

```console
goku@ubuntu-srv01:~$ sudo /var/ossec/bin/wazuh-control info
WAZUH_VERSION="v4.12.0"
WAZUH_REVISION="rc1"
WAZUH_TYPE="server"
```

```console
goku@ubuntu-srv01:~$ sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard
● wazuh-manager.service    - Wazuh manager     │ active (running)
● wazuh-indexer.service    - Wazuh indexer     │ active (running)
● wazuh-dashboard.service  - Wazuh dashboard   │ active (running)
```

- **`WAZUH_TYPE="server"`** confirms this node is the manager (not an agent).
- The dashboard is served over HTTPS at `https://<goku-ip>` and is the single pane of glass for the whole lab.

---

## 🛰️ Architecture

```
                         ┌──────────────────────────────┐
                         │      GOKU — Ubuntu Server     │
                         │   Wazuh Manager + Indexer +   │
                         │          Dashboard            │
                         │        (SIEM server)          │
                         └───────────────▲──────────────┘
                      agent telemetry     │     (port 1514/1515)
                  ┌──────────────────────┴───────────────────────┐
                  │                                               │
        ┌─────────┴──────────┐                       ┌────────────┴─────────┐
        │  FRIEZA — Kali     │                       │  VEGETA — Windows 11 │
        │  Wazuh Agent       │                       │  Wazuh Agent         │
        │  10.10.10.219      │                       │  10.10.10.20         │
        └────────────────────┘                       └──────────────────────┘
```

| Host | Role | Wazuh role | OS |
|---|---|---|---|
| **Goku** | Server / Services | **Manager + Indexer + Dashboard** | Ubuntu Server |
| **Frieza** | Attacker / Pentester | Agent (`kali`) | Kali GNU/Linux 2026.1 |
| **Vegeta** | Target / User | Agent (`windows11`) | Windows 11 Home |

> Manager and agents all run **Wazuh v4.12.0**, communicating over the isolated `Namek` network (`10.10.10.0/24`).

---

## 📦 Agent Deployment

A Wazuh **agent** is a lightweight service installed on each monitored endpoint. It collects logs, file-integrity data, and security events and ships them to the manager on **Goku**. Two agents were enrolled — one Linux, one Windows.

### 🔴 Kali Linux (Frieza) — Linux agent

On Debian-based systems like Kali, the agent installs from the Wazuh APT repository. The `WAZUH_MANAGER` variable points the agent at Goku's IP so it knows where to register:

```console
root@frieza:~# WAZUH_MANAGER="<goku-ip>" WAZUH_AGENT_NAME="kali" \
    apt install wazuh-agent

root@frieza:~# systemctl daemon-reload
root@frieza:~# systemctl enable --now wazuh-agent
root@frieza:~# systemctl status wazuh-agent
● wazuh-agent.service - Wazuh agent
     Active: active (running)
```

The agent registers with the manager automatically and appears in the dashboard within seconds.

### 🟡 Windows 11 (Vegeta) — Windows agent

On Windows the agent ships as an MSI. It's installed silently from an elevated PowerShell prompt, again pointing at Goku as the manager, then the service is started:

```powershell
PS C:\> .\wazuh-agent-4.12.0.msi /q `
    WAZUH_MANAGER="<goku-ip>" `
    WAZUH_AGENT_NAME="windows11"

PS C:\> NET START WazuhSvc
The Wazuh Svc service was started successfully.
```

Once the service starts, the Windows host enrolls and begins forwarding event logs (logons, process creation, etc.) to the SIEM.

---

## ✅ Enrolled Agents

Both agents show **active** in the Wazuh dashboard, reporting from the `Namek` network:

![Wazuh enrolled agents dashboard](assets/wazuh-agents.png)

| ID | Name | IP address | Operating system | Version | Status |
|---|---|---|---|---|---|
| 001 | `windows11` | `10.10.10.20` | Microsoft Windows 11 Home (10.0.26200) | v4.12.0 | 🟢 active |
| 002 | `kali` | `10.10.10.219` | Kali GNU/Linux 2026.1 | v4.12.0 | 🟢 active |

- **Agents by status:** 2 active, 0 disconnected, 0 pending
- **Top OS:** windows (1), kali (1) · **Group:** default (2) · **Cluster node:** node01

---

## 🎯 Why This Matters

With agents reporting from both the **attacker** and the **target**, this SIEM closes the purple-team loop: I can launch an attack from Frieza (Kali) against Vegeta (Windows 11) and watch the telemetry land on Goku's dashboard — failed logons, suspicious processes, and MITRE ATT&CK-mapped alerts — then investigate and tune detections.

## 🗺️ Next Steps

- [ ] Deploy **Sysmon** on Windows for deeper process/network telemetry
- [ ] Run a controlled attack (e.g. brute-force RDP/SMB) and capture the resulting alerts
- [ ] Build custom detection rules and a dashboard mapped to ATT&CK techniques
- [ ] Document full attack → detection → response walkthroughs with screenshots

---

*Part of my [cybersecurity homelab](https://github.com/pervis007/cybersecurity-homelab) and [portfolio](https://pervis007.github.io).*
