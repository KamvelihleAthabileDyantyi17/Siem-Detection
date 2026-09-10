# Cybersecurity Operations Home Lab

**Author:** Kamvelihle Athabile Dyantyi  
**Location:** Cape Town, South Africa  
**Objective:** Building a practical, phase-based cybersecurity portfolio focused on SIEM orchestration, endpoint security, and SOC automation.

---

## Project Overview
This repository documents the progressive deployment and configuration of a comprehensive cybersecurity lab environment. The project emphasizes hands-on troubleshooting, infrastructure orchestration, and defensive security monitoring.

## Phase 1: SIEM Orchestration (Wazuh)
**Status:** ✅ Completed / In Review

### Environment Setup
- **Host System:** Windows Virtual Machine
- **Container Engine:** Docker Desktop with WSL2 (Ubuntu) integration.
- **Kernel Tuning:** Configured WSL2 virtual memory map parameters (`sysctl -w vm.max_map_count=262144`) to support OpenSearch/Elasticsearch requirements for the Wazuh Indexer.

### Troubleshooting & Architectural Fixes
- **Filesystem Isolation:** Bypassed NTFS/ext4 cross-filesystem permission restrictions (`chmod` locks) by strictly isolating Git cloning and Docker orchestration within the native Linux environment (`~`).
- **Daemon Access Control:** Resolved unprivileged access blocks to the Docker Engine by modifying Unix domain socket file modes (`chmod 666 /var/run/docker.sock`).
- **WSL State Recovery:** Documented and resolved Docker CLI segmentation faults and `daemon.json` I/O corruption by forcefully unregistering the broken WSL backends (`wsl --unregister docker-desktop`) and executing a clean engine rebuild.

### Deployment Commands
```bash
# Clone the Wazuh repository
git clone -b v4.8.0 https://github.com/wazuh/wazuh-docker.git

# Generate secure communication certificates
docker compose -f generate-indexer-certs.yml run --rm generator

# Orchestrate the SIEM stack (Manager, Indexer, Dashboard)
docker compose up -d
```

---

## Phase 2: Endpoint Vulnerability & Monitoring (Upcoming)
**Status:** ⏳ Pending

### Objectives
- [ ] Spin up a vulnerable Ubuntu endpoint (`ssh-victim`) on the SIEM's Docker network.
- [ ] Configure the OpenSSH server with mock user credentials.
- [ ] Deploy and enroll the Wazuh Agent for centralized log ingestion.
- [ ] *[Placeholder for attack simulation and log analysis]*

---

## Phase 3: Active Directory & Windows Security (Planned)
**Status:** ⏳ Pending
- *Details and documentation will be added here during Phase 3.*

---

## Phase 4: Phishing Analysis & Triage (Planned)
**Status:** ⏳ Pending
- *Details and documentation will be added here during Phase 4.*

---

## Phase 5: Cloud Security & SOC Automation (Planned)
**Status:** ⏳ Pending
- *Details and documentation will be added here during Phase 5.*

---
*Note: This documentation is actively updated as the lab progresses.*
