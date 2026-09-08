<div align="center">

# 🛡️ SOC Lab

### A self-hosted Security Operations Center lab for hands-on detection engineering

Wazuh SIEM/XDR • Vulnerable target (DVWA) • Automated response (n8n)

![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Wazuh](https://img.shields.io/badge/Wazuh-4.9.0-3AB6E6?style=for-the-badge)
![n8n](https://img.shields.io/badge/n8n-automation-EA4B71?style=for-the-badge&logo=n8n&logoColor=white)
![License](https://img.shields.io/badge/license-Educational_Use-yellow?style=for-the-badge)

</div>

---

## 📖 Overview

This lab is a compact, reproducible SOC environment for practicing detection engineering, log analysis, and automated incident response — entirely on Docker.

**The scenario:** a deliberately vulnerable web app (DVWA) generates attack traffic → its logs are collected and analyzed by Wazuh → detections are enriched and indexed → high-severity alerts trigger a webhook to n8n, which orchestrates an automated response.

Built for learning, demoing, or CTF-style practice — not for production.

---

## 🏗️ Architecture

```
                         ┌─────────────────────────────────────────┐
                         │         Docker network: soc_lab          │
                         │                                           │
   ┌───────────┐  logs   │   ┌───────────────────────────────────┐ │
   │   DVWA    ├─────────┼──▶│         wazuh.manager              │ │
   │  (target) │         │   │   log collection · rule engine     │ │
   │   :80     │         │   └────────────────┬────────────────────┘ │
   └───────────┘         │                    │ alerts                │
                         │                    ▼                       │
   ┌───────────┐ webhook │   ┌───────────────────────────────────┐ │
   │    n8n    │◀────────┼───│         wazuh.indexer              │ │
   │(response) │         │   │      OpenSearch storage             │ │
   │  :5678    │         │   └────────────────┬────────────────────┘ │
   └───────────┘         │                    │                       │
                         │                    ▼                       │
                         │   ┌───────────────────────────────────┐ │
                         │   │         wazuh.dashboard             │ │
                         │   │        web UI · :443                │ │
                         │   └───────────────────────────────────┘ │
                         └─────────────────────────────────────────┘
```

---

## 🧩 Components

| Service | Purpose | Port(s) | Image |
|---|---|---|---|
| `wazuh.indexer` | Event storage & search (OpenSearch) | `9200` | `wazuh/wazuh-indexer:4.9.0` |
| `wazuh.manager` | Log collection, decoding, rule engine | `1514`, `1515`, `514/udp`, `55000` | `wazuh/wazuh-manager:4.9.0` |
| `wazuh.dashboard` | Web UI for visualization & investigation | `443` | `wazuh/wazuh-dashboard:4.9.0` |
| `dvwa` | Intentionally vulnerable web app (attack target) | `80` | custom (Debian + Apache + PHP + MariaDB) |
| `n8n` | Workflow automation for alert response | `5678` | custom (Node 20 Alpine) |

All services share the Docker bridge network **`soc_lab`** (`172.18.0.0/16`).

---

## 🚀 Getting Started

### Prerequisites

- Docker & Docker Compose
- **6 GB+ RAM recommended** for the host — Wazuh + OpenSearch are memory-hungry
- Shared network created upfront:
  ```bash
  docker network create soc_lab
  ```

### 1 · Start the Wazuh stack

```bash
cd wazuh-docker/single-node
docker compose up -d
```

Check everything came up healthy:

```bash
docker ps
docker logs single-node-wazuh.indexer-1 --tail 30
```

> ⏳ **First boot takes time.** OpenSearch can take 60–90s to fully initialize. `connection refused` errors during this window are expected.

### 2 · Start DVWA

```bash
cd ../../dvwa
docker build -t lab-dvwa:custom .
docker run -d --name lab-dvwa --network soc_lab -p 80:80 \
  -v shared_logs:/var/log/apache2 \
  lab-dvwa:custom
```

Visit `http://<VM-IP>` — default login `admin` / `password`. Set security level to **"low"** under *DVWA Security* for unobstructed testing.

### 3 · Start n8n

```bash
cd ../n8n
docker build -t lab-n8n:custom .
docker run -d --name lab-n8n --network soc_lab -p 5678:5678 \
  -e WEBHOOK_URL=http://<VM-IP>:5678/ \
  -e N8N_SECURE_COOKIE=false \
  -v n8n_data:/home/node/.n8n \
  lab-n8n:custom
```

Visit `http://<VM-IP>:5678`.

> ⚠️ `N8N_SECURE_COOKIE=false` disables the secure-cookie requirement for plain HTTP access. **Lab use only — never in production.**

### 4 · Verify connectivity

```bash
docker network inspect soc_lab
```

All 5 containers should be listed on the same subnet.

---

## 🔗 Wiring DVWA into Wazuh

DVWA's Apache logs reach the Wazuh manager through the shared Docker volume `shared_logs`, read-only-mounted at `/var/ossec/dvwa-logs`.

Add to `config/wazuh_cluster/wazuh_manager.conf`:

```xml
<localfile>
  <log_format>apache</log_format>
  <location>/var/ossec/dvwa-logs/access.log</location>
</localfile>

<localfile>
  <log_format>apache</log_format>
  <location>/var/ossec/dvwa-logs/error.log</location>
</localfile>
```

Apply changes:

```bash
docker compose up -d --force-recreate wazuh.manager
```

---

## 🔑 Default Credentials

| Service | User | Password |
|---|---|---|
| Wazuh Dashboard | `admin` | `SecretPassword` |
| Wazuh API | `wazuh-wui` | `MyS3cr37P450r.*-` |
| DVWA | `admin` | `password` |
| n8n | set on first login | — |

> 🔐 **Rotate these before exposing the lab to any untrusted network.**

---

## 🛠️ Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `connection refused` on port 9200 | OpenSearch still initializing | Wait 60–90s after `docker compose up` |
| `Permission denied` on `wazuh_indexer_ssl_certs/` | Certs are intentionally locked down (400, root-owned) | Expected — Docker (root) can still read them |
| `OutOfMemoryError` at indexer startup | JVM heap (`OPENSEARCH_JAVA_OPTS`) too low for the loaded security plugins | Set `-Xms1g -Xmx1g` minimum |
| `no such host` between containers | Containers not (yet) sharing the same network, or not fully up | Check `docker network inspect soc_lab`, wait, retry |
| n8n blocks access with "secure cookie" warning | Accessing over plain HTTP without TLS | Set `N8N_SECURE_COOKIE=false` (lab only) |
| DVWA container `Exited (137)` | OOM-kill — host out of memory | Increase VM RAM or avoid running everything simultaneously |

---

## 🗺️ Roadmap

- [x] Deploy Wazuh stack (indexer, manager, dashboard)
- [x] Shared network (`soc_lab`) with DVWA and n8n
- [x] Ship DVWA's Apache logs to the Wazuh manager
- [ ] Validate detections (SQLi, XSS, brute-force) in the dashboard
- [ ] Tune Wazuh rules/decoders as needed
- [ ] Wazuh → n8n webhook on critical alerts
- [ ] n8n response workflow (notifications, IP blocking via active response)
- [ ] *(Stretch)* Native Wazuh agent on DVWA for deeper host visibility

---

## ⚠️ Disclaimer

This lab bundles **intentionally vulnerable software** (DVWA). It is meant strictly for educational use inside an isolated environment (private network, dedicated VM, no internet exposure).

---

## 📄 License

This project stitches together components under separate licenses:

- Wazuh Docker — GPLv2
- DVWA — GPLv3
- n8n — Sustainable Use License

Refer to each upstream project's license for details.
