# Wazuh SIEM & Vulnerable Endpoint Lab

## Overview

This project is a self-contained SIEM (Security Information and Event Management) lab, built with Docker and Wazuh on WSL2. It walks through deploying a real detection stack, standing up a deliberately weak endpoint next to it, and then attacking that endpoint to watch the SIEM catch it in real time.

The point isn't just "get Wazuh running" — it's understanding the full loop: an endpoint generates activity, an agent ships that activity to a manager, the manager correlates it against detection rules, and the results surface as alerts on a dashboard. Doing this hands-on, including breaking and fixing the environment along the way, mirrors the kind of troubleshooting and log analysis expected in an actual SOC role.

**Stack:**
- **SIEM core** — Wazuh Indexer, Manager, and Dashboard, run as containers via Docker Compose
- **Target endpoint** — an isolated Ubuntu container (`ssh-victim`) running SSH with a deliberately weak account
- **Attack simulation** — repeated failed logins used to trigger and verify brute-force detection

---

## Table of Contents
1. [Prerequisites](#1-prerequisites)
2. [Standing Up the SIEM](#2-standing-up-the-siem)
3. [Building the Victim Endpoint](#3-building-the-victim-endpoint)
4. [Installing the Wazuh Agent](#4-installing-the-wazuh-agent)
5. [Running the Attack Simulation](#5-running-the-attack-simulation)
6. [Reading the Alerts](#6-reading-the-alerts)
7. [Tearing Down / Freeing Up Space](#7-tearing-down--freeing-up-space)
8. [Notes on Environment Issues Hit Along the Way](#8-notes-on-environment-issues-hit-along-the-way)

---

## 1. Prerequisites

- Windows with WSL2 and an Ubuntu distro installed
- Docker Desktop, with **WSL Integration** turned on for that Ubuntu distro (Settings → Resources → WSL Integration)
- Git
- A browser
- Comfortable free disk space and RAM — the Indexer in particular is memory-hungry; low free RAM (under ~1GB) is a common cause of crashes during this lab

---

## 2. Standing Up the SIEM

Clone the official Wazuh Docker repo and move into the single-node deployment folder:

```bash
git clone -b v4.8.0 https://github.com/wazuh/wazuh-docker.git
cd wazuh-docker/single-node
```

Generate the SSL certificates the Wazuh components use to talk to each other securely:

```bash
docker compose -f generate-indexer-certs.yml run --rm generator
```

Then bring the whole stack up in the background:

```bash
docker compose up -d
```

This pulls and starts three containers — Indexer, Manager, Dashboard — so give it a few minutes on first run.

Once it's up, confirm all three are running:

```bash
docker ps
```

Then open the dashboard in a browser at:

```
https://127.0.0.1
```

You'll hit a self-signed certificate warning — click **Advanced → Proceed** (wording varies by browser). Log in with the default credentials:

- **Username:** `admin`
- **Password:** `SecretPassword`

> To resume later without rebuilding, use `docker compose start` instead of `up -d` — this wakes existing containers rather than recreating them.

---

## 3. Building the Victim Endpoint

Spin up a plain Ubuntu container and attach it to the same Docker network as the SIEM, so it can talk to the manager:

```bash
docker run -it --name ssh-victim --network single-node_default ubuntu:latest bash
```

You're now inside the container as root. Install SSH and the tools needed to set it up:

```bash
apt-get update && apt-get install -y openssh-server curl sudo
```

Create a low-privilege account with a deliberately weak password — this is the account you'll "attack" later:

```bash
useradd -m -s /bin/bash victim
echo "victim:password123" | chpasswd
```

Start the SSH service:

```bash
service ssh start
```

---

## 4. Installing the Wazuh Agent

The agent is what actually reports this container's activity back to the SIEM.

1. In the Wazuh dashboard, go to **Agents → Deploy new agent**.
2. Set:
   - **Operating system:** Debian/Ubuntu
   - **Architecture:** x86_64 (amd64)
   - **Wazuh server address:** `wazuh.manager` (the internal Docker network name — not `localhost`)
   - **Agent group:** default
3. Copy the install command the dashboard generates, paste it into the `ssh-victim` container terminal, and run it.
4. Start the agent:

```bash
service wazuh-agent start
```

5. Back in the dashboard's **Agents** list, confirm `ssh-victim` shows as **Active**.

---

## 5. Running the Attack Simulation

Open a **second** terminal to act as the attacker, separate from the victim container.

Get the victim's internal IP:

```bash
hostname -I
```

Then, from the attacker terminal, attempt an SSH login using that IP (no angle brackets — swap in the actual address):

```bash
ssh victim@<victim-ip>
```

Enter the wrong password on purpose, several times in a row, to generate failed-login events.

> If the two terminals can't reach each other over SSH directly (common with Docker's default networking), you can generate the same failed-auth logs locally inside the victim container instead:
> ```bash
> su - victim
> ```
> Wazuh reads `/var/log/auth.log` either way, so the source of the failed attempt doesn't matter for detection.

---

## 6. Reading the Alerts

1. In the dashboard, go to **Threat intelligence → Threat hunting** (this is where "Security events" lives in current Wazuh versions).
2. Set the time filter (top right) to **Last 15 minutes** so you're only looking at fresh data.
3. Filter to your endpoint: `agent.name: ssh-victim`
4. Look for the spike in failed authentication attempts and the corresponding rule trigger — brute-force SSH failures typically map to **rule ID 5710**.
5. Click into an event to see the full detail: timestamp, targeted account, source, and severity.

---

## 7. Tearing Down / Freeing Up Space

Once you're done, you don't need to keep the containers running to keep the project — the whole environment is defined in code, so it's fully reproducible.

**Save your work (config, scripts, docs) to GitHub:**

```bash
git add .
git commit -m "Wazuh SIEM lab: SIEM core + victim endpoint + brute-force detection"
```

**Stop and remove containers, networks, and volumes to reclaim disk space:**

```bash
docker compose down -v
```

**Optional — clear out unused images/build cache system-wide** (only if nothing else is running):

```bash
docker system prune -a --volumes
```

To bring it all back later: clone the repo again (or `cd` back into it) and run `docker compose up -d`. It'll rebuild the exact same environment.

---

## 8. Notes on Environment Issues Hit Along the Way

Documenting these because they're common WSL2/Docker failure modes, not lab-specific bugs:

- **`docker: command not found` inside WSL** — usually means Docker Desktop's WSL integration dropped. Fix: Settings → Resources → WSL Integration → toggle Ubuntu off/on → Apply & restart.
- **Segmentation fault on `docker ps` after heavy image pulls** — typically a low-RAM symptom. The engine can survive; the CLI just crashes momentarily. A `wsl --shutdown` (run from **Windows** Command Prompt, not inside Ubuntu) followed by restarting Docker Desktop usually clears it.
- **Docker Desktop stuck in a start/stop loop** — sign of a corrupted internal WSL disk. Force-quit Docker Desktop via Task Manager, then from Windows Command Prompt run:
  ```
  wsl --unregister docker-desktop
  wsl --unregister docker-desktop-data
  ```
  Reopening Docker Desktop will recreate both automatically (you'll lose any running containers, but not your project files or config).
- **Permission denied on `/var/run/docker.sock`** — either add your user to the `docker` group and restart WSL fully, or as a quick unblock: `sudo chmod 666 /var/run/docker.sock`.

---

## Quick Reference — Full Command Sequence

```bash
# Clone and start the SIEM
git clone -b v4.8.0 https://github.com/wazuh/wazuh-docker.git
cd wazuh-docker/single-node
docker compose -f generate-indexer-certs.yml run --rm generator
docker compose up -d

# Build the victim endpoint (new terminal/session inside the container)
docker run -it --name ssh-victim --network single-node_default ubuntu:latest bash
apt-get update && apt-get install -y openssh-server curl sudo
useradd -m -s /bin/bash victim
echo "victim:password123" | chpasswd
service ssh start
# (install agent via dashboard-generated command, then:)
service wazuh-agent start

# Simulate the attack (separate terminal)
hostname -I            # run inside victim container to get its IP
ssh victim@<victim-ip> # enter wrong password a few times

# Clean up
docker compose down -v
```
