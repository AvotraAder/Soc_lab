<div align="center">

# 🛡️ SOC Lab

### A self-hosted Security Operations Center lab for hands-on detection engineering

Wazuh SIEM/XDR • Vulnerable target (DVWA) • Attacker box (Kali) • Automated response (n8n)

![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Wazuh](https://img.shields.io/badge/Wazuh-4.9.0-3AB6E6?style=for-the-badge)
![n8n](https://img.shields.io/badge/n8n-automation-EA4B71?style=for-the-badge&logo=n8n&logoColor=white)
![License](https://img.shields.io/badge/license-Educational_Use-yellow?style=for-the-badge)

</div>

---

## 📖 Overview

This lab is a compact, reproducible SOC environment for practicing detection engineering, log analysis, and automated incident response — entirely on Docker.

**The scenario:** an attacker container (Kali) targets a deliberately vulnerable web app (DVWA) → its logs are collected and analyzed by Wazuh → detections are enriched, correlated, and indexed → high-severity alerts trigger a webhook to n8n, which orchestrates an automated response.

Built for learning, demoing, or CTF-style practice — not for production.

---

## 🏗️ Architecture

```
                         ┌─────────────────────────────────────────┐
                         │         Docker network: soc_lab          │
                         │                                           │
   ┌───────────┐         │   ┌───────────────────────────────────┐ │
   │   kali    │  attack │   │              DVWA                   │ │
   │ attacker  ├─────────┼──▶│          (target) :80               │ │
   └───────────┘         │   └────────────────┬────────────────────┘ │
                         │                    │ logs                  │
                         │                    ▼                       │
                         │   ┌───────────────────────────────────┐ │
   ┌───────────┐ webhook │   │         wazuh.manager              │ │
   │    n8n    │◀────────┼───│   log collection · rule engine     │ │
   │(response) │         │   │      correlation rules              │ │
   │  :5678    │         │   └────────────────┬────────────────────┘ │
   └───────────┘         │                    │ alerts (Filebeat)    │
                         │                    ▼                       │
                         │   ┌───────────────────────────────────┐ │
                         │   │         wazuh.indexer              │ │
                         │   │      OpenSearch storage             │ │
                         │   └────────────────┬────────────────────┘ │
                         │                    │                       │
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
| `wazuh.manager` | Log collection, decoding, rule engine, correlation | `1514`, `1515`, `514/udp`, `55000` | `wazuh/wazuh-manager:4.9.0` |
| `wazuh.dashboard` | Web UI for visualization & investigation | `443` | `wazuh/wazuh-dashboard:4.9.0` |
| `dvwa` | Intentionally vulnerable web app (attack target) | `80` | custom (Debian + Apache + PHP + MariaDB) |
| `attacker` | Kali-based box for launching test attacks (nmap, sqlmap) | — | custom (`kalilinux/kali-rolling`) |
| `n8n` | Workflow automation for alert response | `5678` | custom (Node 20 Alpine) |

All services share the Docker bridge network **`soc_lab`** (`172.18.0.0/16`).

> The `wazuh-docker/build-docker-images/` folder also contains everything needed to build the three Wazuh images from official sources instead of pulling them from Docker Hub — useful if you want to understand what's inside each image. The `docker-compose.yml` in this repo defaults to the official Docker Hub images for reliability.

---

## 🚀 Getting Started

### Prerequisites

- Docker & Docker Compose
- **6 GB+ RAM recommended** for the host — Wazuh + OpenSearch are memory-hungry (a 4 GB VM works but benefits from a 2 GB swap file)
- `vm.max_map_count` raised for OpenSearch:
  ```bash
  sudo sysctl -w vm.max_map_count=262144
  echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
  ```
- Shared network and volume created upfront:
  ```bash
  docker network create --subnet=172.18.0.0/16 soc_lab
  docker volume create shared_logs
  ```

### 1 · Generate TLS certificates for the Wazuh stack

Wazuh's security plugin requires TLS certs for the indexer, manager, and dashboard. Certificates are generated once with the official `wazuh-certs-generator` tool. Since the generator validates each node's `ip`/DNS entry, use fixed IPs on the `soc_lab` subnet rather than container names:

```yaml
# certs.yml
nodes:
  indexer:
    - name: wazuh-indexer
      ip: 172.18.0.10
  server:
    - name: wazuh-manager
      ip: 172.18.0.11
  dashboard:
    - name: wazuh-dashboard
      ip: 172.18.0.12
```

```bash
docker build -t wazuh-certs-generator:0.0.1 wazuh-docker/indexer-certs-creator

mkdir -p certs/output
docker run --rm \
  -v $(pwd)/certs.yml:/config/certs.yml \
  -v $(pwd)/certs/output:/certificates \
  wazuh-certs-generator:0.0.1
```

The `single-node/config/wazuh_indexer_ssl_certs/` folder in this repo already ships with a working cert set for local testing — regenerate only if you need different node identities.

### 2 · Start the Wazuh stack

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

### 3 · Start DVWA

```bash
cd ../../dvwa
docker build -t lab-dvwa:custom .
docker run -d --name lab-dvwa --network soc_lab -p 80:80 \
  -v shared_logs:/var/log/apache2 \
  lab-dvwa:custom
```

Visit `http://<VM-IP>` — default login `admin` / `password`. Set security level to **"low"** under *DVWA Security* for unobstructed testing.

### 4 · Start the attacker box

```bash
cd ../kali-attacker
docker build -t lab-attacker:custom .
docker run -dit --name lab-attacker --network soc_lab \
  --memory=300m lab-attacker:custom bash
```

```bash
docker exec -it lab-attacker bash
nmap --script http-enum -p80 dvwa
```

The attacker container resolves `dvwa` by container name over the `soc_lab` network — no need to hardcode IPs.

### 5 · Start n8n

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

### 6 · Verify connectivity

```bash
docker network inspect soc_lab
```

All containers should be listed on the same subnet.

---

## 🔗 Wiring DVWA into Wazuh

DVWA's Apache logs reach the Wazuh manager through the shared Docker volume `shared_logs`, read-only-mounted at `/var/ossec/dvwa-logs`.

Declared in `config/wazuh_cluster/wazuh_manager.conf` (mounted to `/wazuh-config-mount/etc/ossec.conf`):

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

## 🧪 Detection Rules Observed

Traffic generated from the attacker box against DVWA (SQL injection, XSS, command injection, LFI, brute force, sensitive-path scanning, and an `nmap --script http-enum` scan) was decoded and matched by Wazuh's default `apache`/`web` ruleset:

| Rule ID | Level | Description |
|---|---|---|
| `502` | 3 | Wazuh server started |
| `31108` | 0 | Ignored URLs (simple queries) |
| `31101` | 5 | Web server 400 error code |
| *(apache)* | 5 | Apache: Attempt to access forbidden file or directory |
| *(appsec)* | 6 | Suspicious URL access (`.bak`, `.htaccess`, etc.) |

A single `http-enum` scan generated **1,000+** individual level-5 alerts (rule `31101`) in under two minutes — realistic noise, and the motivation for the custom correlation rule below.

Quick way to test any log line against the ruleset without waiting for the full pipeline:

```bash
docker exec -it single-node-wazuh.manager-1 /var/ossec/bin/wazuh-logtest
```

### Custom correlation rule — scan detection

File: `config/wazuh_cluster/local_rules.xml`, mounted to `/var/ossec/etc/rules/local_rules.xml`.

```xml
<group name="local,attack,recon,">

  <rule id="100100" level="10" frequency="20" timeframe="120" ignore="120">
    <if_matched_sid>31101</if_matched_sid>
    <same_source_ip />
    <description>Possible scan/enumeration detected: multiple 400/404 errors from same source IP ($(srcip)) in short time.</description>
    <mitre>
      <id>T1595</id>
    </mitre>
    <group>recon,attack,</group>
  </rule>

</group>
```

If rule `31101` fires **20+ times from the same source IP within 120 seconds**, a single level-10 alert (`100100`) is raised instead, tagged with MITRE ATT&CK **T1595 (Active Scanning)**. This turns a flood of low-signal 404s into one actionable, high-priority alert — the kind of event worth routing to n8n.

Custom rule IDs must stay ≥ `100000` to avoid colliding with the official ruleset. Deploy with:

```bash
docker compose up -d wazuh.manager
```

---

## 🔑 Default Credentials

| Service | User | Password |
|---|---|---|
| Wazuh Dashboard | `admin` | `SecretPassword` |
| Wazuh API | `wazuh-wui` | `MyS3cr37P450r.*-` |
| Wazuh Indexer / OpenSearch | `admin` | `SecretPassword` |
| Dashboard ↔ Indexer (internal) | `kibanaserver` | `kibanaserver` |
| DVWA | `admin` | `password` |
| n8n | set on first login | — |

> 🔐 **Rotate these before exposing the lab to any untrusted network.**

---

## ✅ Verifying the Stack

```bash
docker ps

# Indexer
curl -k -u admin:SecretPassword https://localhost:9200

# Manager API (returns a JWT token on success)
curl -k -X POST -u wazuh-wui:'MyS3cr37P450r.*-' "https://localhost:55000/security/user/authenticate"

# Dashboard
curl -k -I https://localhost:443

# DVWA
curl -I http://localhost:80

# n8n
curl -I http://localhost:5678
```

Confirm DVWA logs are reaching the manager:

```bash
docker exec single-node-wazuh.manager-1 tail -10 /var/ossec/dvwa-logs/access.log
```

Check whether the correlation rule has fired:

```bash
docker exec single-node-wazuh.manager-1 grep "100100" /var/ossec/logs/alerts/alerts.json | tail -5
```

In the dashboard's **Discover** view, useful DQL filters:

```
rule.groups: "attack"
rule.level >= 6
rule.id: "100100"
data.srcip: "<attacker-ip>"
```

---

## ⏯️ Stop / Restart

Stop while keeping all data (indices, certs, volumes):

```bash
cd wazuh-docker/single-node
docker compose down
docker stop lab-dvwa lab-n8n lab-attacker
```

Restart:

```bash
docker start lab-dvwa lab-n8n lab-attacker
cd wazuh-docker/single-node
docker compose up -d
```

Full teardown (⚠️ destroys indexed data):

```bash
docker compose down -v
docker rm -f lab-dvwa lab-n8n lab-attacker
```

---

## 🛠️ Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `connection refused` on port 9200 | OpenSearch still initializing | Wait 60–90s after `docker compose up` |
| `AccessControlException` reading `.pem` certs at indexer startup | Cert mounted at the wrong path (Java's security manager resolves a fixed internal path, not necessarily where you mounted it) | Match the exact cert filenames/paths referenced in `opensearch.yml` — see `config/wazuh_indexer/wazuh.indexer.yml` |
| `No match for argument: wazuh-indexer--` during a manual image build | Missing `--build-arg WAZUH_VERSION=4.9.0 --build-arg WAZUH_TAG_REVISION=1` | Pass both build args explicitly (version **with dots**, not `490`) |
| `Invalid IP or DNS` from `wazuh-certs-generator` | Container names aren't resolvable at cert-generation time (no other containers running yet) | Use fixed IPs in `certs.yml` instead of hostnames |
| `Permission denied` copying `root-ca-manager.pem` / `wazuh-manager*.pem` | Files are generated with restrictive ownership (`400`, non-`root` user) by design | `sudo cp ... && sudo chown $USER:$USER ...` |
| `OutOfMemoryError` at indexer startup | JVM heap (`OPENSEARCH_JAVA_OPTS`) too low for the loaded security plugins | Set `-Xms1g -Xmx1g` minimum |
| `no such host` between containers | Containers not (yet) sharing the same network, or not fully up | Check `docker network inspect soc_lab`, wait, retry |
| n8n blocks access with "secure cookie" warning | Accessing over plain HTTP without TLS | Set `N8N_SECURE_COOKIE=false` (lab only) |
| DVWA container `Exited (137)` | OOM-kill — host out of memory | Increase VM RAM, add swap, or avoid running everything simultaneously |
| Thousands of level-5 alerts flooding Discover after a scan | Expected — default ruleset alerts per request, no built-in aggregation | Use/extend the custom correlation rule (`100100`) described above |

---

## 🗺️ Roadmap

- [x] Deploy Wazuh stack (indexer, manager, dashboard)
- [x] Shared network (`soc_lab`) with DVWA, attacker box, and n8n
- [x] Ship DVWA's Apache logs to the Wazuh manager
- [x] Validate detections (SQLi, XSS, command injection, LFI, brute force, sensitive-path scanning) in the dashboard
- [x] Add a custom correlation rule to collapse scan noise into a single high-severity alert
- [ ] Wazuh → n8n webhook on critical alerts (`<integration>` block in `ossec.conf`)
- [ ] n8n response workflow (notifications, IP blocking via active response)
- [ ] *(Stretch)* Native Wazuh agent on DVWA for deeper host visibility
- [ ] *(Stretch)* Additional correlation rules (repeated brute-force logins, repeated SQLi attempts)

---

## ⚠️ Disclaimer

This lab bundles **intentionally vulnerable software** (DVWA) and an **attack tooling image** (Kali + nmap/sqlmap). It is meant strictly for educational use inside an isolated environment (private network, dedicated VM, no internet exposure).

---

## 📄 License

This project stitches together components under separate licenses:

- Wazuh Docker — GPLv2
- DVWA — GPLv3
- n8n — Sustainable Use License
- Kali Linux base image — see [Kali's licensing terms](https://www.kali.org/docs/policy/kali-linux-open-source-policy/)

Refer to each upstream project's license for details.
