# Local Agent Architecture

**Version:** 1.0  
**Created:** 2026-03-01  
**Project:** NetDiscoverIT - Privacy-First Network Discovery

---

## 1. Overview

The Local Agent is a lightweight component that runs on the customer's network, collecting device data and generating metadata vectors locally. This ensures sensitive network information never leaves the customer's infrastructure while still leveraging NetDiscoverIT's AI/ML capabilities.

---

## 2. Problem Statement

**Current Architecture (Cloud-Only):**
- Customer sends raw configs to NetDiscoverIT cloud
- Requires high trust — customers hesitant to share network topology
- Compliance issues for HIPAA, PCI, government
- "Black box" perception

**Desired Architecture (Hybrid):**
- Customer controls data collection
- Only processed vectors leave the network
- Transparent, auditable, compliance-friendly

---

## 3. Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CUSTOMER NETWORK (ON-PREM)                           │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      LOCAL AGENT (Docker/VM)                         │   │
│  │                                                                       │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│  │  │  Collector  │─▶│  Normalizer  │─▶│  Sanitizer  │─▶│  Vectorizer │  │   │
│  │  │  (Netmiko/ │  │  (LLM/JSON)  │  │  (PII Strip)│  │  (Embeddings│  │   │
│  │  │   NAPALM)  │  │              │  │             │  │             │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └──────┬──────┘  │   │
│  │                                                              │         │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │         │   │
│  │  │  Scheduler  │  │  Crypto     │  │  Local DB   │         │         │   │
│  │  │  (Cron)     │  │  (TLS 1.3)  │  │  (SQLite)   │         ▼         │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘   ┌───────────┐   │   │
│  │                                                      │   Queue   │   │   │
│  │                                                      │  (Vector  │───┼───┤
│  │                                                      │   Batch)  │   │   │
│  │                                                      └───────────┘   │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                         │                                    │
│                                  HTTPS (TLS 1.3)                           │
│                                  Encrypted Vectors                         │
│                                         │                                    │
└─────────────────────────────────────────┼────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           NETDISCVERIT CLOUD                                │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                        API GATEWAY                                     │   │
│  │                                                                       │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │   │
│  │  │  Auth       │  │  Vector     │  │  ML         │  │  LLM        │ │   │
│  │  │  (JWT/API)  │  │  Ingest     │  │  Classifier │  │  Advisor    │ │   │
│  │  └─────────────┘  └─────────────┘  └──────┬──────┘  └──────┬──────┘ │   │
│  │                                            │                │        │   │
│  └────────────────────────────────────────────┼────────────────┼────────┘   │
│                                                 │                │            │
│                           ┌─────────────────────┼────────────────┐        │
│                           ▼                     ▼                ▼        │
│                    ┌─────────────┐       ┌─────────────┐  ┌───────────┐   │
│                    │  Vector DB  │       │  PostgreSQL │  │  Config  │   │
│                    │  (Pinecone/ │       │  (Metadata) │  │  Generator│   │
│                    │   Milvus)   │       │             │  │           │   │
│                    └─────────────┘       └─────────────┘  └───────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Components

### 4.1 Collector

**Responsibility:** Connect to network devices via SSH, SNMP, or API

**Supported Protocols:**
| Protocol | Use Case | Libraries |
|----------|----------|-----------|
| SSH (Netmiko) | Config retrieval, CLI commands | `netmiko`, `napalm` |
| SNMP | Polling for interface stats, topology | `easysnmp`, `pysnmp` |
| gRPC | Modern device APIs | `grpcio` |
| REST | Cloud-managed devices | `requests` |

**Device Support:**
- Cisco (IOS, IOS-XE, NX-OS)
- Juniper (Junos)
- Arista (EOS)
- F5 (BIG-IP)
- Palo Alto (PAN-OS)
- Ubiquiti (UniFi)

### 4.2 Normalizer (Local LLM)

**Responsibility:** Convert vendor-specific CLI output to JSON

**Approach:**
1. **Lightweight LLM** (Ollama 7B) runs locally
2. Prompt engineering converts config → standardized JSON
3. Fallback: TextFSM templates for common commands

**Local LLM Requirements:**
- Hardware: 8GB RAM, 20GB storage
- Model: Llama 3.2 7B or smaller
- Alternative: Skip local LLM, use templates only (reduced accuracy)

### 4.3 Sanitizer

**Responsibility:** Remove PII before transmission

**Sanitization Rules:**
```python
SCRUB_PATTERNS = {
    "password": "***",
    "community": "***",
    "secret": "***",
    "key": "***",
    "username": "[USER]",
    "hostname": "[HOST]",
    "ip_address": "[IP]",  # optional, for extra privacy
}
```

**What NEVER leaves the network:**
- Raw configs
- Credentials, passwords, community strings
- Management IPs (optional)
- SNMP strings

### 4.4 Vectorizer

**Responsibility:** Generate embeddings for ML queries

**Vector Types:**
| Vector | Dimensions | Purpose |
|--------|------------|---------|
| `device_role` | 768 | Find devices by function |
| `topology` | 768 | Network mapping |
| `security` | 768 | Vulnerability assessment |
| `config` | 768 | Anomaly detection |

### 4.5 Scheduler

**Responsibility:** Periodic discovery runs

**Configurable:**
- Interval: hourly, daily, weekly
- Time of day: off-peak hours
- Full vs delta scans

### 4.6 Crypto

**Responsibility:** Secure transmission

- TLS 1.3 for all cloud communication
- Certificate pinning (optional)
- API key + JWT authentication

### 4.7 Local DB

**Responsibility:** Store collected data locally

**Technology:** SQLite (embedded, no admin required)

**Retention:**
- Configurable: 7, 30, 90 days
- Supports on-prem backup

---

## 5. Deployment Options

### 5.1 Docker (Recommended)

```yaml
# docker-compose.yml
services:
  netdiscover-agent:
    image: netdiscoverit/agent:latest
    volumes:
      - ./config:/app/config
      - ./data:/app/data
    environment:
      - API_KEY=your_api_key
      - API_ENDPOINT=https://api.netdiscoverit.com
      - SCAN_INTERVAL=24h
    restart: unless-stopped
```

### 5.2 VM (On-Prem)

- OVA/VMDK for VMware/Hyper-V
- Minimal OS (Alpine Linux)
- 2 vCPU, 4GB RAM, 20GB storage

### 5.3 Kubernetes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: netdiscover-agent
spec:
  containers:
  - name: agent
    image: netdiscoverit/agent:latest
    env:
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: netdiscover-secrets
          key: api-key
```

---

## 6. Data Transmission

### 6.1 What's Sent to Cloud

```json
{
  "batch_id": "uuid",
  "customer_id": "uuid",
  "timestamp": "2026-03-01T12:00:00Z",
  
  "devices": [
    {
      "device_id": "uuid",
      "metadata": { /* normalized, sanitized */ },
      "vectors": {
        "device_role": [0.1, 0.2, ...],
        "topology": [0.3, 0.4, ...],
        "security": [0.5, 0.6, ...]
      }
    }
  ],
  
  "recommendations_requested": true
}
```

### 6.2 What's NEVER Sent

- ❌ Raw configurations
- ❌ Passwords, secrets, keys
- ❌ SNMP community strings
- ❌ Management IP addresses (unless opted-in)
- ❌ Full interface descriptions (if sensitive)

---

## 7. Security Model

### 7.1 Trust Hierarchy

```
┌─────────────────────────────────────────────┐
│              CUSTOMER OWNS DATA              │
│                                             │
│  1. Customer installs agent                 │
│  2. Agent runs in customer's network        │
│  3. Customer controls scan schedule         │
│  4. Customer can audit all collected data  │
│  5. Customer can disable/delete anytime    │
│                                             │
│  NetDiscoverIt NEVER sees raw data          │
└─────────────────────────────────────────────┘
```

### 7.2 Compliance

| Standard | Support |
|----------|---------|
| HIPAA | ✅ Data stays on-prem |
| PCI-DSS | ✅ No cardholder data transmitted |
| FedRAMP | ✅ On-prem option available |
| SOC 2 | ✅ Audit logs available |
| GDPR | ✅ No EU data leaves EU (on-prem) |

### 7.3 Privacy Modes

| Mode | What Goes to Cloud | Use Case |
|------|-------------------|----------|
| **Full Cloud** | Everything | Dev/test, non-sensitive |
| **Hybrid** (Default) | Vectors only | Production, most customers |
| **On-Prem Only** | Nothing | Highest security, air-gapped |

---

## 8. API Endpoints (Agent → Cloud)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v1/agent/register` | POST | Register agent with customer |
| `/api/v1/agent/heartbeat` | POST | Health check |
| `/api/v1/agent/vectors` | POST | Upload vector batch |
| `/api/v1/agent/recommendations` | GET | Fetch recommendations |
| `/api/v1/agent/config` | GET | Fetch agent config updates |

---

## 9. Installation

### 9.1 Quick Start

```bash
# 1. Get API key from dashboard
# 2. Run agent
docker run -d \
  -e API_KEY=your_key \
  -e CUSTOMER_ID=your_org \
  -v ./config:/app/config \
  netdiscoverit/agent:latest
```

### 9.2 Configuration

```yaml
# config/agent.yaml
agent:
  customer_id: "uuid"
  api_endpoint: "https://api.netdiscoverit.com"
  scan_interval: "24h"
  log_level: "info"

discovery:
  methods:
    - ssh
    - snmp
  ssh:
    timeout: 30
    retry: 3
  snmp:
    timeout: 5
    community: "public"

privacy:
  scrub_ips: true
  scrub_hostnames: false
  scrub_descriptions: false
```

---

## 10. Monitoring

### 10.1 Local Logs

```bash
# View logs
docker logs netdiscover-agent

# Stream logs
docker logs -f netdiscover-agent
```

### 10.2 Cloud Dashboard

- Agent status (online/offline)
- Last successful sync
- Devices discovered
- Vector upload status

---

## 11. Related Documentation

- [Metadata Schema](./metadata-schema.md) - Data format
- [Security Design](./security-design.md) - Encryption details
- [Discovery Pipeline](./discovery-pipeline.md) - Collection process

---

## 12. Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-03-01 | NetDiscoverIT Team | Initial architecture |
