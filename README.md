
A home security operations center (SOC) lab built on Proxmox VE and Wazuh SIEM, with an isolated Raspberry Pi honeypot. The goal is to practice log collection, threat detection, and incident investigation hands-on, and build a foundation for a follow-up cloud security lab in Microsoft Azure.

> **Status:** In progress. 

## Goals

- Build a virtualized lab environment from bare metal
- Segment the lab network so attack traffic stays isolated
- Deploy Wazuh SIEM and collect logs from Windows and Linux endpoints
- Simulate attacks against an internal honeypot and write detections for them
- Document every phase, including what broke and how it was fixed

## Architecture

**Phase 1: On-prem (this repo)**
- Proxmox VE hypervisor hosting the Wazuh server and endpoint VMs
- Raspberry Pi honeypot on an isolated network segment (internal only, not internet-facing)
- Controlled attack simulation from a separate segment, with alerts flowing into Wazuh
- Remote administration over Tailscale with key-only SSH, with no inbound ports exposed

```mermaid
flowchart TB
    Net((Internet)) --- Dorm["Dorm Wi-Fi"]
    Dorm --- Beryl["GL.iNet Beryl router<br/>192.168.8.1 (192.168.8.0/24)"]
    Beryl --- BR0["vmbr0: management bridge<br/>(uplink to Beryl)"]
    BR0 ---|"192.168.8.163"| Host["Proxmox host<br/>Tailscale 100.x.x.x"]
    BR0 ---|"ens18: 192.168.8.185 (DHCP)"| W["wazuh (VM 100)<br/>manager, indexer, dashboard"]
    W ---|"ens19: 10.10.10.10"| BR1["vmbr1: isolated lab bridge<br/>10.10.10.0/24, no uplink, no gateway"]
    BR1 ---|"10.10.10.30"| E1["lab-ubuntu-01 (VM 101)"]
    Laptop["Laptop (Tailscale)"] -.->|"SSH tunnel to :443"| Host
```  

**Phase 2: Cloud (future repo)**
- Internet-facing honeypot in Azure feeding Microsoft Sentinel for real-world attack analysis
_Network diagram coming once the layout is finalized._

## Hardware

| Component | Role |
|---------| -------|
| HP Pavilion Gaming Desktop (GTX 1660 Ti, 16 GB RAM) | Proxmox VE host |
| GL.iNet BERYL travel router | Connects the host to campus wifi |
| Raspberry Pi | Internal honeypot |

---

## Progress
| Phase | Status | Writeup |
|---|---|---|
| 0. Proxmox install and remote access | Done | [Read the writeup](docs/01-proxmox-install-writeup.md) |
| 1. Wazuh server VM | Done | [Read the writeup](docs/02-wazuh-manager.md) |
| 2. Windows and Linux endpoints with agents | In Progress | [Read the writeup](docs/03-lab-endpoint.md) |
| 3. Raspberry Pi honeypot | Planned | not yet |
| 4. Attack simulation and custom detections | Planned | not yet |


## Lessons Learned

- Check release notes and known issues before installing; the newest version isn't always the most stable.
- When two independently verified pieces of media fail the same way, search the exact error text before re-diagnosing.
- Prove a replacement works (key login) from a fresh session before removing the fallback (password login).
- Real environments come with constraints, like campus network rules, and the lab has to be designed around them.
