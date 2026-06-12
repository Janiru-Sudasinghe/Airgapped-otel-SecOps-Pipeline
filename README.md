# Air-Gapped Windows Event Log Forwarding to Google SecOps via Self-Hosted Bindplane

> Collecting Windows Event Logs from a fully offline (air-gapped) Windows host and forwarding them to **Google Security Operations (SecOps / Chronicle)** through a **self-hosted Bindplane** server and gateway collector - with **zero internet access** on the log-source machine.

<p align="center">
  <img alt="Platform" src="https://img.shields.io/badge/Platform-CentOS%20Stream%209%20%7C%20Windows%2010%20Pro-blue">
  <img alt="Bindplane" src="https://img.shields.io/badge/Bindplane-v1.100.0-5A3FFF">
  <img alt="Collector" src="https://img.shields.io/badge/OpenTelemetry%20Collector-v1.100.0-F5A800">
  <img alt="SIEM" src="https://img.shields.io/badge/SIEM-Google%20SecOps%20(Chronicle)-34A853">
  <img alt="Environment" src="https://img.shields.io/badge/Environment-Air--Gapped%20%2F%20Offline-critical">
  <img alt="License" src="https://img.shields.io/badge/License-MIT-green">
</p>

---

## 📌 Table of Contents

1. [Overview](#-overview)
2. [Architecture](#-architecture)
3. [Key Features](#-key-features)
4. [Technology Stack](#-technology-stack)
5. [Prerequisites](#-prerequisites)
6. . [Network & Port Plan](#-network--port-plan)
7. . [Implementation Guide](#-implementation-guide)
   - [Phase 1 — Stage Binaries (Internet-Connected Hosts)](#phase-1--stage-binaries-internet-connected-hosts)
   - [Phase 2 — CentOS Bindplane Server (Offline)](#phase-2--centos-bindplane-server-offline)
   - [Phase 3 — Google SecOps Onboarding](#phase-3--google-secops-onboarding)
   - [Phase 4 — Gateway Collector (Offline CentOS)](#phase-4--gateway-collector-offline-centos)
   - [Phase 5 — Windows 10 Agent (Offline)](#phase-5--windows-10-agent-offline)
   - [Phase 6 — Pipelines (Bindplane GUI)](#phase-6--pipelines-bindplane-gui)
   - [Phase 7 — Verification & Testing](#phase-7--verification--testing)
8. [Configuration Reference](#-configuration-reference)
9. [Troubleshooting](#-troubleshooting)
10. [Security & Secret Handling](#-security--secret-handling)
11. [Screenshots](#-screenshots)
12. [Repository Structure](#-repository-structure)
13. [Lessons Learned](#-lessons-learned)
14. [References](#-references)
15. [Author & License](#-author--license)

---

## 🔎 Overview

This project demonstrates a **production-style, air-gapped log ingestion pipeline**. A Windows 10 host with **no internet connectivity** generates Windows Event Logs (Security, Application, System). Because Google SecOps is a cloud service, the offline host cannot reach it directly - so the design uses Bindplane's **gateway collector pattern**:

- The **offline Windows agent** ships raw Windows Events over the **LAN** (OTLP) to a gateway.
- The **gateway collector** (on the CentOS server, the only host with controlled outbound HTTPS) holds the Google credentials and forwards everything to **Google SecOps**.

This keeps all credentials on a single hardened egress point and leaves the log source completely isolated - the model used in real segmented/OT/DMZ environments.

| | |
|---|---|
| **Goal** | Forward Windows Event Logs from an air-gapped host to Google SecOps |
| **Log source** | Windows 10 Pro (offline) — Security, Application, System channels |
| **Pipeline** | Self-hosted Bindplane (Google Edition) + OpenTelemetry collectors |
| **Destination** | Google SecOps (Chronicle), region `asia-southeast1`, log type `WINEVTLOG` |
| **Egress model** | Gateway collector pattern (only the CentOS server reaches Google) |
| **Result** | ✅ Windows Events successfully ingested and searchable in Google SecOps |

---

## 🏗 Architecture

This maps out the complete lifecycle of the telemetry, from the moment a security event is generated on the airgapped Windows machine to its final destination in the Google SecOps cloud.

<p align="center">
  <br>
  <img src="./Docs/diagram.png" width="1000" alt="Network Topology Diagram">
</p>

---

## ✨ Key Features

- **True air-gap for the log source** — the Windows host has no route to the internet; verified by a failing outbound connectivity test.
- **Gateway egress pattern** — credentials and outbound HTTPS live on exactly one host (the CentOS gateway).
- **100% offline software supply chain** — every binary is staged on internet-connected hosts, checksummed, transferred via media, and installed locally. Bindplane runs in `offline: true` mode so collectors never call out to GitHub/`bdot.bindplane.com`.
- **GUI-managed pipelines** — sources, processors, and destinations are built in the Bindplane console and pushed to collectors via OpAMP (no hand-edited collector YAML).
- **Raw WINEVTLOG ingestion** — Windows Events sent raw so the Google SecOps `WINEVTLOG` parser normalizes them correctly.
- **Reproducible** — all versions pinned to **v1.100.0**.

---

## 🧰 Technology Stack

| Layer | Component | Version |
|---|---|---|
| Hypervisor | VMware Workstation |  |
| Server OS | CentOS Stream 9 |  |
| Agent OS | Windows 10 Pro |  |
| Pipeline mgmt | Bindplane (Google Edition) | v1.100.0 |
| Collector | observIQ Distro for OpenTelemetry Collector (BDOT) | v1.100.0 |
| Database | PostgreSQL (PGDG) | 15 |
| SIEM | Google SecOps (Chronicle) | `asia-southeast1` |
| Transport | OTLP (gRPC) over LAN; gRPC/HTTPS to SecOps |  |

---

## ✅ Prerequisites

- **2 offline VMs:** CentOS Stream 9 (Bindplane server + gateway) and Windows 10 Pro (agent), on the same isolated LAN segment.
- **2 internet-connected staging hosts:** one CentOS, one Windows — used only to download binaries and transfer them via offline media.
- A **Bindplane (Google Edition) license key**.
- A **Google SecOps** tenant (this project uses region `asia-southeast1`).
- Removable media (USB/ISO) for air-gap transfers.

---

## 🌐 Network & Port Plan

| Host | Example IP | Ports (inbound) | Purpose |
|---|---|---|---|
| Bindplane server / gateway (CentOS) | `192.168.98.130` | `3001/tcp` | Bindplane UI/API + OpAMP (WebSocket) |
| | | `4317/tcp` | OTLP gRPC (gateway ingest) |
| | | `4318/tcp` | OTLP HTTP (gateway ingest) |
| | | `5432/tcp` | PostgreSQL (local/LAN) |
| Windows agent | `192.168.98.140` | — | Outbound only to server `:3001` and `:4317` |
| Google SecOps | cloud | `443` (outbound from gateway) | Ingestion API |


---

## 🚀 Implementation Guide

> **Legend:** each step is tagged with the machine it runs on: <br>
>  🟢 `staging-CentOS` <br>
> 🪟 `staging-Windows` <br>
> 🔵 `offline-CentOS-server` <br>
> ⬛ `offline-Windows-agent` <br>
> 🌐 `Bindplane console` <br>
> ☁️ `Google SecOps console`.

### Phase 1 - Stage Binaries (Internet-Connected Hosts)

🟢 **staging-CentOS** - download all Linux artifacts and resolve PostgreSQL dependencies:

```
# Create nested folders and move into the root airgap directory
mkdir -p ~/airgap/{bindplane,postgres,collector,artifacts} && cd ~/airgap  
```
```
# Download BindPlane Enterprise RPM directly into the bindplane folder
wget "https://downloads.bindplane.com/bindplane/1.100.0/bindplane-ee_linux_amd64.rpm" \
  -O ~/airgap/bindplane/bindplane-ee_v1.100.0_linux_amd64.rpm  
```
```
 # Download the BindPlane CLI zip into the bindplane folder
wget "https://storage.googleapis.com/bindplane-op-releases/bindplane/1.100.0/bindplane-ee-linux-amd64.zip" \
  -O ~/airgap/bindplane/bindplane-cli-v1.100.0.zip
```
```
# Download the BindPlane CLI zip into the bindplane folder
wget "https://github.com/observIQ/bindplane-otel-collector/releases/download/v1.100.0/observiq-otel-collector_v1.100.0_linux_amd64.rpm" \
  -P ~/airgap/collector/  # Download the OpenTelemetry collector RPM into the collector folder
```
```
# Download the OpenTelemetry offline artifacts into the collector folder
wget "https://github.com/observIQ/bindplane-otel-collector/releases/download/v1.100.0/observiq-otel-collector-v1.100.0-artifacts.tar.gz" \
  -P ~/airgap/collector/
```
```
# Move into the postgres directory to isolate the database files
cd postgres/
```
```
# Install the Postgres repo locally so dnf can locate the packages
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```
```
# Disable the default OS Postgres module to prevent version conflicts
sudo dnf module disable -y postgresql
```
```
# Install necessary tools (dnf-plugins-core provides the 'dnf download' command)
sudo dnf install -y dnf-plugins-core wget
```
```
# Save the repo setup file itself for the target offline machine
wget -P ~/airgap/postgres/ https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```
```
# Fetch Postgres 15 and resolve ALL dependencies for offline use
dnf download --resolve --alldeps --destdir=postgres postgresql15-server postgresql15 postgresql15-libs postgresql15-contrib
```
```
# Return to root, hash all downloaded files, and output to a text file for integrity verification
cd ~/airgap && find . -type f \( -name '*.rpm' -o -name '*.tar.gz' -o -name '*.zip' \) -exec sha256sum {} \; > SHA256SUMS.txt 
```

🪟 **staging-Windows** - download the agent MSI: (PowerShell)

```
# Force PowerShell to use TLS 1.2 for secure web connections, preventing connection errors with modern web servers
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
```
```
# Download the observIQ OpenTelemetry Collector installer (v1.100.0) and save it locally as an MSI file
Invoke-WebRequest -Uri "https://bdot.bindplane.com/v1.100.0/observiq-otel-collector.msi" -OutFile "observiq-otel-collector-v1.100.0.msi"
```
```
# Calculate the SHA256 cryptographic hash of the downloaded MSI and save that value into a text file for integrity verification
(Get-FileHash .\observiq-otel-collector-v1.100.0.msi -Algorithm SHA256).Hash | Out-File -FilePath .\observiq-v1.100.0-SHA256.txt
```

**Transfer** `~/airgap/` to the CentOS server and the MSI to `C:\offline\` on the Windows agent via offline media, then re-verify checksums on both targets.

---

### Phase 2 - CentOS Bindplane Server (Offline)

🔵 **offline-CentOS-server** - base system, firewall, time:

```
# Set the system timezone to Sri Lanka time (Asia/Colombo)
sudo timedatectl set-timezone Asia/Colombo
```
```
# Display the current system date and time to verify the timezone change was successful
date
```
```
# Set the machine's hostname to 'bindplane.local'
sudo hostnamectl set-hostname bindplane.local
```
```
# Open TCP port 3001 permanently in the firewall (default port for the BindPlane OP user interface and API)
sudo firewall-cmd --permanent --add-port=3001/tcp
```
```
# Open TCP port 4317 permanently (default port for the OpenTelemetry collector receiving OTLP over gRPC)
sudo firewall-cmd --permanent --add-port=4317/tcp
```
```
# Open TCP port 4318 permanently (default port for the OpenTelemetry collector receiving OTLP over HTTP)
sudo firewall-cmd --permanent --add-port=4318/tcp
```
```
# Open TCP port 5432 permanently (default port for PostgreSQL database connections)
sudo firewall-cmd --permanent --add-port=5432/tcp
```
```
# Reload the firewall to apply the newly added port rules, then immediately list all open ports to verify them
sudo firewall-cmd --reload && sudo firewall-cmd --list-ports
```
```
# Enable the Chrony NTP (Network Time Protocol) service to start automatically on boot, and start it right now
sudo systemctl enable --now chronyd
```
```
# Enable the Chrony NTP service to start on boot, start it immediately, and check its synchronization status
sudo systemctl enable --now chronyd && chronyc tracking
```

**Install PostgreSQL 15 (offline):**

```
# Navigate into the directory containing the downloaded PostgreSQL RPM files
cd postgres/
```
```
# Install all PostgreSQL packages locally from the downloaded RPM files without requiring an internet connection
sudo dnf localinstall -y *.rpm
```
```
# Initialize the PostgreSQL database cluster, which creates the base files and system tables required to run the database
sudo /usr/pgsql-15/bin/postgresql-15-setup initdb
```
```
# Enable the PostgreSQL 15 service to start automatically on system boot, and start it immediately
sudo systemctl enable --now postgresql-15
```
```
# Check the current status of the PostgreSQL service to confirm it is actively running without errors
systemctl status postgresql-15
```

**Create the database and user** (use a strong password - do **not** reuse the username):

```
# Run a block of SQL commands as the default 'postgres' superuser to create the BindPlane database and user
sudo -u postgres psql <<'SQL'
CREATE USER bindplane_user WITH PASSWORD 'bindplane_user';
CREATE DATABASE bindplane OWNER bindplane_user;
GRANT ALL PRIVILEGES ON DATABASE bindplane TO bindplane_user;
SQL
```
```
# Connect directly to the newly created 'bindplane' database to grant the user permissions over the public schema
sudo -u postgres psql -d bindplane <<'SQL'
GRANT ALL ON SCHEMA public TO bindplane_user;
ALTER DATABASE bindplane OWNER TO bindplane_user;
SQL
```

**Enable password auth** in `/var/lib/pgsql/15/data/pg_hba.conf` — set the `127.0.0.1/32` and `::1/128` lines to `scram-sha-256` (and add a LAN line if PostgreSQL is accessed remotely):

```conf
host    all          all             127.0.0.1/32        scram-sha-256
host    all          all             ::1/128             scram-sha-256
host    bindplane    bindplane_user  192.168.98.0/24     scram-sha-256
```

If PostgreSQL must listen beyond localhost, in `/var/lib/pgsql/15/data/postgresql.conf` set `listen_addresses = '*'`. Then:

```
# Restart the PostgreSQL 15 service to apply any recent configuration changes and ensure the database is running smoothly
sudo systemctl restart postgresql-15
```
```
# Verify the database connection by connecting via local TCP/IP (127.0.0.1) as 'bindplane_user' to the 'bindplane' database, and execute a simple query to check the version
psql -h 127.0.0.1 -U bindplane_user -d bindplane -c "SELECT version();"
```

**Install & initialize Bindplane:**

```
# Copy the downloaded BindPlane Enterprise RPM file from your offline bundle into the temporary directory for installation
cp ~/airgap/bindplane/bindplane-ee_* /tmp/
```
```
# Install the BindPlane application package locally using the copied RPM file, without requiring an internet connection
sudo dnf localinstall -y /tmp/bindplane-ee_*.rpm
```
```
# Initialize the BindPlane server configuration, setting the environment variable for the data directory and specifying the path for the newly generated configuration file
sudo BINDPLANE_CONFIG_HOME=/var/lib/bindplane /usr/local/bin/bindplane init server --config /etc/bindplane/config.yaml
```

Answer the `init` prompts:

| Prompt | Value |
|---|---|
| License Key | `<BINDPLANE_LICENSE_KEY>` |
| Server Host | `0.0.0.0` |
| Server Port | `3001` |
| Remote URL | `http://192.168.98.130:3001` |
| Authentication Method | `single-user` |
| Username / Password | `admin` / `<CONSOLE_PASSWORD>` |
| Database Host / Port | `192.168.98.130` (or `localhost`) / `5432` |
| Database Name / User / Password | `bindplane` / `bindplane_user` / `<DB_PASSWORD>` |
| SSL Mode | `disable` |

**Enable offline mode** in `/etc/bindplane/config.yaml` (host-add `offline: true`; confirm the OpAMP `secretKey`):

```yaml
apiVersion: bindplane.observiq.com/v1beta1
offline: true
env: production
auth:
  type: system
  username: admin
  password: <CONSOLE_PASSWORD>
  secretKey: <OPAMP_SECRET_KEY>     # UUID used to enroll collectors
```
```
# Enable the BindPlane service to start automatically on system boot, and start it immediately
sudo systemctl enable --now bindplane
```
```
# Check the current operational status of the BindPlane service to verify it started successfully without errors
sudo systemctl status bindplane
```
```
# Retrieve and display the default BindPlane account credentials (username and password) generated during setup so you can log into the web interface
bindplane get account
```

**Host the offline collector artifact** so air-gapped collectors install without internet:

```
# Create a hidden '.bindplane' directory in your home folder to store CLI configurations and cache
mkdir -p ~/.bindplane
```
```
# Extract the BindPlane CLI executable from your offline zip file directly into a system-wide directory so you can run the 'bindplane' command from anywhere
sudo unzip ~/airgap/bindplane/bindplane-cli*.zip -d /usr/local/bin/
```
```
# Create a new CLI profile named 'airgap' that tells the CLI how to connect to your BindPlane server (be sure to replace <CONSOLE_PASSWORD> with the password from the previous step)
bindplane profile set airgap --remote-url http://192.168.98.130:3001 --username admin --password <CONSOLE_PASSWORD>
```
```
# Set the newly created 'airgap' profile as your active profile for all future CLI commands
bindplane profile use airgap
```
```
# Verify that the CLI is correctly installed and can successfully communicate with the BindPlane server by checking the version
bindplane version
```
```
# Upload the offline OpenTelemetry collector artifacts to the BindPlane server, allowing you to deploy and upgrade agents on your network without internet access
bindplane upload agent-upgrade ~/airgap/collector/observiq-otel-collector-v1.100.0-artifacts.tar.gz --version v1.100.0
```

🌐 Browse to `http://192.168.98.130:3001` and log in.

---

### Phase 3 - Google SecOps Onboarding

☁️ **Google SecOps console:**

1. **SIEM Settings → Collection Agents** → download the **Ingestion Authentication File** (service-account JSON). 
2. **SIEM Settings → Profile → Organization Details** → copy the **Customer ID** and note the **GCP Project Number**.
3. Record the ingestion details used by the gateway:

| Field | Value |
|---|---|
| Name | `google-secops-dest` |
| Region | `asia-southeast1` |
| Endpoint (gRPC) | `asia-southeast1-malachiteingestion-pa.googleapis.com` |
| Customer ID | `<SECOPS_CUSTOMER_ID>` |
| GCP Project Number | `<GCP_PROJECT_NUMBER>` |
| Log type / parser | `WINEVTLOG` |
| Auth method | `json` (paste credentials in the destination) |

---

### Phase 4 - Gateway Collector (Offline CentOS)

🔵 **offline-CentOS-server:**

```
# Install or upgrade the observIQ OpenTelemetry Collector using the RPM package from your offline bundle
sudo rpm -U ~/airgap/collector/observiq-otel-collector_v1.100.0_linux_amd64.rpm
```
```
# Open the collector's manager configuration file in the vim text editor so you can point it to your BindPlane OP server
sudo vim /opt/observiq-otel-collector/manager.yaml
```

```yaml
endpoint: ws://192.168.98.130:3001/v1/opamp
secret_key: <OPAMP_SECRET_KEY>
```
```
# Enable the OpenTelemetry collector service to start automatically on system boot, and start it immediately
sudo systemctl enable --now observiq-otel-collector
```
```
# Check the current operational status of the collector service to ensure it is running smoothly and communicating properly
sudo systemctl status observiq-otel-collector
```

🌐 **Bindplane console** - confirm the gateway shows **Connected**, then build the gateway pipeline:

- **Create Configuration:** Give it a name `Gateway-Pipeline`, Select Platform = **Linux** Click Next.
- **Add Source → Select Bindplane Gateway** (defaults - listens on `4317`/`4318`) Click Save.
- **Add Processors → Search and add Batch** Save it, then again Search and add **Google SecOps Standardization** with `logType = WINEVTLOG` Save it.
- **Add Destination on right → Search and add Google SecOps:** Protocol `gRPC`; Endpoint `asia-southeast1-malachiteingestion-pa.googleapis.com`; Auth `json` (paste the credentials JSON); Customer ID `<SECOPS_CUSTOMER_ID>`; Fallback Log Type `WINEVTLOG`; enable **Retry on Failure** Click Save.
- **Click Apply → select the gateway collector → Start Rollout.**

---

### Phase 5 - Windows 10 Agent (Offline)

⬛ **offline-Windows-agent** (elevated PowerShell / CMD):

```
# Rename the Windows machine to 'WIN-AGENT01' and immediately restart the system to apply the new name
Rename-Computer -NewName WIN-AGENT01 -Restart
```
```
# Test local network connectivity to the BindPlane server's UI/API port to ensure this agent can communicate with the management console (This should report success)
Test-NetConnection 192.168.98.130 -Port 3001
```
```
# Test local network connectivity to the BindPlane server's OpenTelemetry gRPC port where the agent will send its metric and log data (This should report success)
Test-NetConnection 192.168.98.130 -Port 4317
```
```
# Verify the air-gap isolation is actively working by confirming the machine cannot reach external public IP addresses (This MUST fail)
Test-NetConnection 8.8.8.8 -Port 443
```

Assuming you have already transferred the observiq-otel-collector-v1.100.0.msi file to C:\offline\, install the agent silently from the locally staged MSI, pointing it at the server's OpAMP endpoint:

```
:: Install the observIQ OpenTelemetry Collector silently from the offline MSI file, configuring it to connect and authenticate with your BindPlane OP management server
msiexec /i "C:\offline\observiq-otel-collector-v1.100.0.msi" /quiet ^
  ENABLEMANAGEMENT=1 ^
  OPAMPENDPOINT=ws://192.168.98.130:3001/v1/opamp ^
  OPAMPSECRETKEY=<OPAMP_SECRET_KEY>
```
```
:: Query the Windows Service Control Manager to check the status of the newly installed collector service (you want to see STATE: 4 RUNNING)
sc query observiq-otel-collector   :: expect STATE: 4 RUNNING
```

🌐 In the Bindplane console → **Agents**, confirm `WIN-AGENT01` is **Connected**.

---

### Phase 6 - Pipelines (Bindplane GUI)

🌐 **Bindplane console** - build the Windows agent pipeline:

- **Create Configuration:** Give it a name `Windows-Server-Logs`, Select Platform = **Windows**.
- **Add Source →Select Windows Events:** Under the Channels section enable **System**, **Application**, **Security**. Under **Advanced**, turn **Raw Logs ON** - *mandatory; SecOps rejects WINEVTLOG without it.* (Optionally set **Start At → beginning** to backfill.) Click Save.
- **Add Destination → Select Bindplane Gateway (OTLP):** Endpoint/Hostname `192.168.98.130`, Port `4317`, Protocol `gRPC`, TLS **insecure/disabled** (internal LAN).
- **Add Processor →Select Batch.** Click Save.
- **Apply → select `WIN-AGENT01` → Start Rollout.**

---

### Phase 7 - Verification & Testing

⬛ **offline-Windows-agent** - generate test events:

```
:: Generate a synthetic "Error" event in the Windows SYSTEM log to verify the OpenTelemetry collector is successfully reading and forwarding System events to BindPlane
eventcreate /T ERROR /ID 999 /L SYSTEM      /SO BindplaneTest /D "Air-gap WINEVTLOG test event"
```
```
:: Generate a second synthetic "Error" event, this time in the APPLICATION log, to ensure application-level events are also being captured and routed correctly
eventcreate /T ERROR /ID 999 /L APPLICATION /SO BindplaneTest /D "Air-gap WINEVTLOG test event"
```

🌐 **Bindplane console** - confirm live throughput: `Windows Events → Bindplane Gateway` (agent pipeline) and `Bindplane Gateway → Google SecOps` (gateway pipeline).

🔵 **offline-CentOS-server** - inspect the gateway export:

```bash
# Continuously monitor the OpenTelemetry collector's live log file in real-time to verify that telemetry data is being successfully authenticated and exported without 401 (Unauthorized) or 403 (Forbidden) errors
sudo tail -f /opt/observiq-otel-collector/log/collector.log
# success = chronicle exports with no 401/403
```

☁️ **Google SecOps console:**
- **Raw Log Search** for `BindplaneTest`.
- **UDM Search:** `metadata.log_type = "WINEVTLOG"` over a recent window.
- **SIEM Settings → Health Hub:** confirm WINEVTLOG **Last Ingested / Last Normalized** are recent. ✅

---

## ⚙️ Configuration Reference

The collector configs are **GUI-generated by Bindplane and pushed via OpAMP** - included only as reference; do not hand-edit them on the hosts.

<details>
<summary><strong>configs/gateway/gateway-config.yaml.example</strong> (CentOS gateway)</summary>

```yaml
# Managed by Bindplane (GUI-generated reference — do not hand-edit)
receivers:
  otlp/source0:
    protocols:
      grpc: { endpoint: 0.0.0.0:4317 }
      http: { endpoint: 0.0.0.0:4318 }
processors:
  batch/winevtlog: {}
  google_secops_standardization/secops:
    logType: WINEVTLOG
exporters:
  chronicle/SecOps:
    compression: gzip
    creds: <REDACTED_INGESTION_AUTH_JSON>
    customer_id: <SECOPS_CUSTOMER_ID>
    endpoint: asia-southeast1-malachiteingestion-pa.googleapis.com
    log_type: WINEVTLOG
    raw_log_field: body
    retry_on_failure: { enabled: true }
service:
  pipelines:
    logs/gateway-to-secops:
      receivers: [otlp/source0]
      processors: [batch/winevtlog, google_secops_standardization/secops]
      exporters: [chronicle/SecOps]
```
</details>

<details>
<summary><strong>configs/windows-agent/windows-agent-config.yaml.example</strong> (Windows agent)</summary>

```yaml
# Managed by Bindplane (GUI-generated reference — do not hand-edit)
receivers:
  windowseventlog/source0__security:    { channel: security,    raw: true, start_at: end }
  windowseventlog/source0__application: { channel: application, raw: true, start_at: end }
  windowseventlog/source0__system:      { channel: system,      raw: true, start_at: end }
processors:
  batch/winevtlog: {}
exporters:
  otlp/bindplane_gateway:
    endpoint: 192.168.98.130:4317
    compression: gzip
    tls: { insecure: true }
    retry_on_failure: { enabled: true }
    sending_queue: { enabled: true }
service:
  pipelines:
    logs/winevtlog:
      receivers: [windowseventlog/source0__security, windowseventlog/source0__application, windowseventlog/source0__system]
      processors: [batch/winevtlog]
      exporters: [otlp/bindplane_gateway]
```
</details>

---

## 🛠 Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Gateway `403 / PermissionDenied` | Wrong region endpoint, Customer ID, or credentials | Verify `asia-southeast1` endpoint + Customer ID; try the multi-region endpoint |
| Gateway `401` | Bad/expired credentials | Re-download the Ingestion Authentication File and re-paste |
| Agent shows **Disconnected** | LAN reachability to `:3001`, wrong scheme | Re-test `Test-NetConnection …:3001`; ensure `ws://` (not `wss://`); check firewalld 3001 |
| Logs flow in Bindplane but **nothing in SecOps** | **Raw Logs not enabled** (most common), clock skew, narrow time range | Confirm `raw: true` in the rolled-out config; sync clocks; widen search window |
| `connection refused` on `:4317` | Gateway not listening / firewall | Confirm gateway Connected, 4317/tcp open, OTLP source rolled out |

---

## 🔐 Security & Secret Handling

> **None of the following may ever be committed:** the Google service-account / Ingestion Authentication File JSON, the Bindplane **license key**, the **OPAMP secret key**, the **Customer ID**, the **GCP project number**, and any **passwords**. If any of these were ever exposed, **revoke and rotate them immediately** (delete the service-account key in Google Cloud Console → IAM → Service Accounts → Keys, and regenerate).

Every committed file uses placeholders (`<DB_PASSWORD>`, `<OPAMP_SECRET_KEY>`, `<SECOPS_CUSTOMER_ID>`, `<GCP_PROJECT_NUMBER>`, `<BINDPLANE_LICENSE_KEY>`, `<REDACTED_INGESTION_AUTH_JSON>`). 

Before any push, run a secret scanner (e.g., `gitleaks detect`) and confirm no real keys remain.

---

## 📸 Screenshots

Place evidence in [`screenshots/`](screenshots/) and reference it here. Recommended capture set:

**On the Windows agent (⬛):**
1. [`Test-NetConnection`](screenshots/Test-NetConnection.png) succeeding on `:3001`/`:4317` and **failing** on the internet (air-gap proof). 
2. [`sc query observiq-otel-collector`](screenshots/sc_query_observiq-otel-collector.png) showing **RUNNING**.
3. [`collector_log`](screenshots/collector_log.png) tail showing OTLP export to `192.168.98.130:4317`.
4. [`Event_Viewer`](screenshots/Event_Viewer.png) showing the test events.

**In the Bindplane console (🌐):** 
1. [`Agents_page`](screenshots/Agents_page.png) - both collectors **Connected**. 
2. [`Windows_pipeline`](screenshots/Windows_pipeline.png) - `Windows Events → Bindplane Gateway` with live throughput. 
3. [`Gateway_pipeline`](screenshots/Gateway_pipeline.png) - `Bindplane Gateway → Google SecOps` with throughput.

**In Google SecOps (☁️):** 
1. [`UDM_Search`](screenshots/UDM_Search.png) `metadata.log_type = "WINEVTLOG"` returning the test events
   
---

## 📂 Repository Structure

```
Airgapped-otel-SecOps-Pipeline/
├── README.md                              # This file
├── docs/
│   └── diagram.png
└── screenshots/
    └── ...                        # Evidence (see Screenshots section)
```

---

## 📚 Lessons Learned

- **Gateways exist for exactly this.** An air-gapped source can't reach a cloud SIEM directly — routing through a single egress gateway is the clean, credential-minimizing answer.
- **Raw Logs is non-negotiable** for `WINEVTLOG`. The single most common "no logs in SecOps" cause.
- **Offline mode + hosted artifacts** (`offline: true` + `bindplane upload agent-upgrade`) is what makes a fully disconnected collector fleet installable and upgradeable.
- **Pin and checksum everything.** Reproducibility in an air-gap depends on controlled, verified binaries.
- **Clock discipline matters** — skewed clocks make ingested logs land outside your search window and look "missing."

---

## 🔗 References

- Google SecOps — *Use Bindplane with Google SecOps* · *Collect Microsoft Windows Event logs* · *Ingestion API*
- Bindplane Docs — *Offline Collector Package Installation* · *Bindplane Gateway* · *Google SecOps (Chronicle) destination* · *Linux/Windows collector installation*
- observIQ — `bindplane-otel-collector` GitHub releases (v1.100.0)
- PostgreSQL — PGDG RHEL 9 repository

---

## 👤 Author & License

**Janiru Sudasinghe**

- GitHub: [@Janiru-Sudasinghe](https://github.com/Janiru-Sudasinghe)
- LinkedIn: [Janiru Sudasinghe](https://www.linkedin.com/in/janiru-sudasinghe/?skipRedirect=true)

Released under the **MIT License** 

> *Disclaimer: This repository documents a lab implementation. All credentials, keys, and identifiers shown are placeholders. Adapt IPs, regions, and versions to your environment.*
