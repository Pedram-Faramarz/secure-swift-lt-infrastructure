# 🏦 Secure SWIFT Logical Terminal Infrastructure

> A multi-stage Linux security project simulating a bank's internal SWIFT terminal environment, built with a strong emphasis on operating system security, system isolation, auditability, and reliability.

**Course:** Security of Operating System  
**University:** Kazakh-British Technical University (KBTU)  
**School:** School of Information Technology and Engineering  
**Author:** Faramarz Pedram  
**Supervisor:** Valyayev Denis  
**Year:** 2026
---

## 📋 Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [SIS 1 — Project Introduction & Machine Provisioning](#sis-1--project-introduction--machine-provisioning)
- [SIS 2 — File System & User Configuration](#sis-2--file-system--user-configuration)
- [SIS 3 — Package Installation & Firewall](#sis-3--package-installation--firewall)
- [SIS 4 — Filesystems, Backup & Redundancy](#sis-4--filesystems-backup--redundancy)
- [SIS 5 — Logging & Monitoring](#sis-5--logging--monitoring)
- [SIS 6 — MAC & Linux Containers](#sis-6--mac--linux-containers)
- [Wazuh — Centralized Monitoring Deployment](#wazuh--centralized-monitoring-deployment)
- [Security Model Summary](#security-model-summary)
- [Tech Stack](#tech-stack)
- [Found Something? Report It](#found-something-report-it)

---

## Project Overview

A new bank is preparing its internal infrastructure before connecting to the international SWIFT system. Before any external connectivity is allowed, the bank must deploy a secure internal SWIFT terminal environment that stores and processes highly confidential financial files.

This project implements the SWIFT terminal as a **logical file-transfer and file-storage system** operating on a local Linux file system. It does **not** implement real SWIFT protocols — instead, it focuses on designing a secure OS environment following best practices across every layer of the stack.

The project is structured as a series of Individual Studies (SIS), each building on the previous stage.

---

## Architecture

The infrastructure consists of **two Fedora Linux virtual machines**:

```
┌─────────────────────────────────────────────────────────────────┐
│                        swift-terminal                           │
│                   (Main SWIFT Terminal Server)                  │
│                                                                 │
│  ┌──────────┐  ┌───────────┐  ┌────────────┐  ┌────────────┐  │
│  │  Nginx   │  │  FastAPI  │  │ PostgreSQL │  │ /swift-lt/ │  │
│  │ (HTTPS)  │→ │ Backend   │→ │  Metadata  │  │  LT1–LT5   │  │
│  └──────────┘  └───────────┘  └────────────┘  └────────────┘  │
│                                                    (LUKS enc.)  │
│  firewalld │ SELinux │ rsyslog-client │ Podman │ mdadm RAID 1  │
└────────────────────────────┬────────────────────────────────────┘
                             │ logs / backups / SSH
┌────────────────────────────▼────────────────────────────────────┐
│                        swift-support                            │
│                  (Supporting Services Server)                   │
│                                                                 │
│  ┌──────────┐  ┌───────────┐  ┌────────────┐  ┌────────────┐  │
│  │ rsyslog  │  │  Grafana  │  │  Node Exp. │  │  /backup/  │  │
│  │ (server) │  │Dashboard  │  │  Metrics   │  │   swift/   │  │
│  └──────────┘  └───────────┘  └────────────┘  └────────────┘  │
│  firewalld │ SELinux │ Wazuh Agent │ rsync/tar backup          │
└─────────────────────────────────────────────────────────────────┘
```

**Data flow:**
- Users access the system only via HTTPS through Nginx
- Nginx serves the frontend and proxies API calls to the FastAPI backend
- Backend writes metadata to PostgreSQL and stores confidential files in LT1–LT5
- All service logs are forwarded to `swift-support` via rsyslog
- Backups of LT directories and DB dumps are transferred to `swift-support` daily
- Monitoring and alerting are handled on `swift-support`

---

## SIS 1 — Project Introduction & Machine Provisioning

**Goal:** Define the project, design the high-level architecture, and provision the virtual machines.

### What was done:
- Selected and described the project scenario (bank SWIFT terminal)
- Designed the two-machine architecture with clearly separated responsibilities
- Justified the choice of **Fedora Linux** (SELinux by default, RHEL-compatible, enterprise-grade)
- Provisioned two VMs using **VirtualBox** with Fedora installed on each

### Defense-in-Depth Layers Defined:

| Layer | Security Measures |
|---|---|
| Virtualization | VM isolation, snapshots, no host OS access |
| Operating System | SELinux enforcing, LUKS encryption, sudo-only privilege escalation |
| Network | firewalld, ports 443 and 22 only, DB local-only |
| Application | Non-root service users, role-based access control |
| Data | Metadata in DB only, sensitive files in encrypted LT directories |
| Logging | journald + rsyslog forwarding to separate server |
| Backup | Encrypted archives on isolated backup server |
| Admin Access | SSH key-based, jump-host model |

---

## SIS 2 — File System & User Configuration

**Goal:** Prepare the file system structure, define roles, manage users and service accounts.

### File System Layout:

| Machine | Path | Purpose | Permissions |
|---|---|---|---|
| swift-terminal | `/swift-lt/LT1–LT5` | SWIFT file storage (most sensitive) | `swiftapi:swift-ops`, 750 |
| swift-terminal | `/var/lib/pgsql/data/` | PostgreSQL metadata DB | `postgres` user only |
| swift-terminal | `/opt/swift-backend/` | Backend application code | `swiftapi` user |
| swift-terminal | `/var/log/` | Local system/service logs | root / rsyslog |
| swift-support | `/backup/swift/` | Backup repository | `backup` user, 700 |
| swift-support | `/var/log/remote/` | Centralized log storage | rsyslog |

### Role-Based Access Control:

| Role | Group | Description | Sudo |
|---|---|---|---|
| admin1 | swift-admin | System administrator | `ALL` |
| sec1 | swift-sec | Security officer | `journalctl`, `firewall-cmd` only |
| operator1 | swift-ops | SWIFT file operator (UI only) | None |
| audit1 | swift-audit | Read-only log access | None |
| swiftapi | — | Backend service account (no login) | None |
| postgres | — | DB service account (no login) | None |
| nginx | — | Web server account (no login) | None |
| backup | — | Backup service account (no login) | None |

### Key commands:
```bash
# Create SWIFT storage directory
sudo mkdir -p /swift-lt/{LT1,LT2,LT3,LT4,LT5}
sudo chown -R swiftapi:swift-ops /swift-lt
sudo chmod -R 750 /swift-lt

# Create service account (no login shell)
sudo useradd -r -s /sbin/nologin swiftapi
```

---

## SIS 3 — Package Installation & Firewall

**Goal:** Install and configure all required packages and services on both machines, and apply firewall rules.

### swift-terminal packages:

| Component | Package | Service | Purpose |
|---|---|---|---|
| Firewall | firewalld | firewalld | Allow only SSH + HTTPS |
| Web proxy | nginx | nginx | HTTPS entry point, serve UI, proxy `/api` |
| Backend runtime | python3, python3-venv | swift-backend (custom) | FastAPI application |
| Database | postgresql-server | postgresql | Metadata storage |
| Log forwarder | rsyslog | rsyslog | Forward logs to swift-support |

### swift-support packages:

| Component | Package | Service | Purpose |
|---|---|---|---|
| Firewall | firewalld | firewalld | Allow SSH + syslog only |
| Log receiver | rsyslog | rsyslog | Receive logs from swift-terminal |
| Backups | rsync, tar | cron | Store backups under `/backup/swift` |
| Monitoring UI | grafana | grafana-server | Dashboard (installed via added repo) |
| Host metrics | node_exporter | node_exporter (custom) | Metrics exporter (installed from archive) |

### Firewall rules applied:
```bash
# swift-terminal
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-service=https

# swift-support
sudo firewall-cmd --permanent --add-port=3000/tcp   # Grafana
sudo firewall-cmd --permanent --add-port=9100/tcp   # Node Exporter
```

### Smoke tests performed:
```bash
curl http://127.0.0.1:8000/health        # Backend API
curl http://127.0.0.1/api/health         # Through Nginx proxy
curl http://localhost:3000               # Grafana
curl http://127.0.0.1:9100/metrics       # Node Exporter
```

---

## SIS 4 — Filesystems, Backup & Redundancy

**Goal:** Define storage policy, implement RAID 1 redundancy, configure automated backups, and verify recovery.

### Storage Policy Principles:
- Separation of data between main server and backup server
- Least privilege — no direct user access to sensitive files
- Encrypted storage for critical SWIFT data (LUKS)
- Centralized backup on a dedicated isolated machine

### Fail-Safe Strategy:

| Risk | Mitigation | Layer |
|---|---|---|
| Single disk failure | RAID 1 mirror on `/swift-lt` | Hardware/RAID |
| Data deletion or corruption | Daily encrypted backups to swift-support | Backup |
| Ransomware / unauthorized modification | Separate backup server, immutable logs | Isolation |
| System crash | ext4 journaling + journald | OS |
| Loss of audit trail | Centralized rsyslog on swift-support | Logging |

### RAID 1 Implementation:
```bash
# Create RAID 1 array with two virtual disks
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc

# Format and mount
sudo mkfs.ext4 /dev/md0
sudo mount /dev/md0 /swift-lt

# Save configuration
sudo mdadm --detail --scan >> /etc/mdadm.conf
```

### Backup Script (`/usr/local/bin/swift-backup.sh`):
- Archives `/swift-lt/` with `tar`
- Dumps PostgreSQL `swiftdb` database
- Transfers both to `/backup/swift/` on `swift-support` via `rsync` over SSH
- Deletes archives older than 7 days automatically
- Scheduled via **cron** at 02:00 daily

### Recovery Test:
A full recovery test was performed — test files were created, backed up, deleted to simulate data loss, then successfully restored from the backup archive with correct ownership and permissions.

---

## SIS 5 — Logging & Monitoring

**Goal:** Configure centralized logging and system monitoring across both machines.

### Logging Architecture:
- **swift-terminal**: journald collects local service logs → rsyslog forwards to swift-support
- **swift-support**: rsyslog server receives and stores logs in `/var/log/remote/`
- Logs on swift-support are isolated from the main server — a compromise of swift-terminal does not affect log integrity

### Monitoring Stack:
- **Grafana** on swift-support: dashboard for system metrics visualization
- **Node Exporter** on swift-support: exposes host metrics (CPU, memory, disk) on port 9100

---

## SIS 6 — MAC & Linux Containers

**Goal:** Implement Mandatory Access Control with SELinux and containerize the backend using Podman.

### SELinux Configuration:

```bash
# Map users to confined SELinux roles
sudo semanage login -a -s sysadm_u admin1     # Full admin role
sudo semanage login -a -s staff_u sec1         # Restricted admin
sudo semanage login -a -s user_u audit1        # Read-only
sudo semanage login -a -s user_u operator1     # Web-only

# Apply security labels to SWIFT directories
sudo semanage fcontext -a -t var_t "/swift-lt(/.*)?"
sudo restorecon -Rv /swift-lt
```

### Why Podman over Docker:

| CRE | Rootless | Daemonless | SELinux Integration |
|---|---|---|---|
| Docker | Partial | ❌ (dockerd) | Manual |
| **Podman** | ✅ Native | ✅ | ✅ Automatic |
| containerd | Partial | ❌ | Manual |

### Container Security Settings:

| Requirement | Implementation |
|---|---|
| Non-root user | `USER swiftapi` in Containerfile |
| Localhost-only port | `--publish 127.0.0.1:8001:8000` |
| Read-only config mount | `--volume .env:/app/.env:ro,Z` |
| Persistent SWIFT storage | `--volume /swift-lt:/swift-lt:Z` |
| Read-only root filesystem | `read_only: true` in podman-compose.yml |
| Declarative config | `podman-compose.yml` |

### Multi-stage Containerfile:
- **Stage 1 (builder):** installs Python dependencies
- **Stage 2 (final):** copies only runtime + app code into minimal Python image
- No build tools, no unnecessary packages, no setuid binaries in final image

---

## Wazuh — Centralized Monitoring Deployment

**Goal:** Deploy a centralized SIEM/monitoring platform to aggregate logs and metrics from all machines in a single unified dashboard.

### Why Wazuh?

This stage goes beyond the Grafana/Node Exporter setup from SIS 5 by adding a full **Security Information and Event Management (SIEM)** layer. While Grafana shows system performance metrics, Wazuh focuses on **security events** — login attempts, file integrity, policy violations, and anomalies.

### Components Deployed:

| Component | Role |
|---|---|
| **Wazuh Manager** | Central brain — collects, correlates, and analyzes data from all agents |
| **OpenSearch** | Search and indexing engine — stores all log and event data |
| **Wazuh Dashboard** | Web UI — real-time alerts, agent status, security events, metrics |

All three components were installed on a dedicated virtual machine and verified to be running before agents were connected.

### Agent Deployment:

Wazuh agents were installed and configured on both machines:

**swift-terminal agent:**
- Forwards system logs, authentication events, and file integrity alerts
- Monitors `/swift-lt/` for unauthorized access or modification
- Reports service status changes (nginx, postgresql, swift-backend)

**swift-support agent:**
- Forwards centralized log reception events
- Monitors backup directory `/backup/swift/` for integrity
- Reports Grafana and rsyslog service health

### Configuration:
Each agent was configured with the Wazuh Manager's IP address to establish a secure connection. After starting the agent services, both machines registered successfully and appeared as **active agents** in the Wazuh Dashboard.

### What the Dashboard Provides:
- **Real-time security alerts** — failed logins, privilege escalation attempts, policy violations
- **Agent health overview** — which machines are online and reporting
- **Log search** — query and filter all forwarded logs from both machines
- **Performance metrics** — CPU, memory, and disk usage across the infrastructure
- **File Integrity Monitoring (FIM)** — detects unauthorized changes to critical files

### How This Fits the Architecture:

```
swift-terminal ──(Wazuh Agent)──┐
                                ├──→ Wazuh Manager + OpenSearch ──→ Wazuh Dashboard
swift-support  ──(Wazuh Agent)──┘
```

This completes the observability stack alongside rsyslog (log forwarding) and Grafana (metrics), giving the infrastructure **three layers of visibility**: raw logs, performance metrics, and security event analysis.

---

## Security Model Summary

```
┌─────────────────────────────────────────────────────┐
│              Defense-in-Depth Layers                │
├──────────────────────┬──────────────────────────────┤
│ Layer                │ Controls                     │
├──────────────────────┼──────────────────────────────┤
│ Virtualization       │ VM isolation, snapshots      │
│ OS                   │ SELinux, LUKS, sudo only     │
│ Network              │ firewalld, ports 22 + 443    │
│ Application          │ Non-root users, RBAC         │
│ Container            │ Podman rootless, read-only   │
│ Data                 │ Encrypted LT dirs, no direct │
│                      │ user access                  │
│ Logging              │ rsyslog → swift-support      │
│ Monitoring           │ Grafana + Wazuh Dashboard    │
│ Backup               │ RAID 1 + daily rsync backups │
│ Admin Access         │ SSH keys, jump-host model    │
└──────────────────────┴──────────────────────────────┘
```

---

## Tech Stack

**Operating System:** Fedora Linux  
**Web Server:** Nginx  
**Backend:** Python 3, FastAPI, Uvicorn  
**Database:** PostgreSQL  
**Container Runtime:** Podman  
**MAC:** SELinux (enforcing mode)  
**Disk Encryption:** LUKS  
**RAID:** mdadm (RAID 1)  
**Backup:** tar, rsync, cron  
**Logging:** journald, rsyslog  
**Monitoring:** Grafana, Node Exporter, Wazuh, OpenSearch  
**Firewall:** firewalld  
**Provisioning:** VirtualBox VMs  

---

---

## Found Something? Report It

This is an educational security project, and feedback from the community is genuinely welcome — whether it's a misconfiguration, a security gap, a better approach, or even a typo.

### 🐛 Found a security issue or misconfiguration?

If you spot something in the configurations, scripts, or architecture that could be improved or is potentially insecure, please open a **GitHub Issue** and use the label `security` or `feedback`.

**What to include in your report:**
- Which SIS stage or section it relates to
- What you found (misconfiguration, weak permission, logic flaw, etc.)
- Why it matters or what the risk is
- Optionally: what you'd suggest instead

### 💡 Have a suggestion or improvement?

Feel free to open an issue with the label `enhancement`. This could be:
- A better tool or approach for any of the stages
- A missing security control worth adding
- A cleaner way to implement something already done

### 📬 Prefer to reach out directly?

You can contact me through GitHub by opening an issue or via the contact information on my profile. I'm happy to discuss anything related to the project.

> **Note:** This project does not implement real SWIFT protocols or handle real financial data. It is a simulated infrastructure for educational purposes only. There are no live systems to exploit.

---

*This project was developed as part of the Security of Operating System course at the School of Information Technology and Engineering, Almaty, 2026.*
