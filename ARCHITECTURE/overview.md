# NetDiscoverIT - Architecture Overview

**Version:** 1.0  
**Date:** 2026-03-01

---

## System Context

NetDiscoverIT has two primary deployment modes:

1. **Cloud** — Everything hosted by NetDiscoverIT
2. **Hybrid** — Local agent on customer network, AI/ML in cloud

This document covers the **Hybrid** architecture (recommended for most customers).

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CUSTOMER PREMISES                                    │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                         LOCAL AGENT                                   │   │
│  │                                                                        │   │
│  │  ┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐  │   │
│  │  │ Collector   │──▶│ Normalizer  │──▶│ Sanitizer   │──▶│ Vectorizer │  │   │
│  │  │ (Netmiko)  │   │ (LLM)       │   │ (PII Strip) │   │ (Embedding)│  │   │
│  │  └────────────┘   └────────────┘   └────────────┘   └──────┬─────┘  │   │
│  │                                                           │         │   │
│  │  ┌────────────┐   ┌────────────┐   ┌────────────┐         │         │   │
│  │  │ Scheduler  │   │ Crypto     │   │ Local DB   │         │         │   │
│  │  │ (Cron)     │   │ (TLS 1.3)  │   │ (SQLite)   │         ▼         │   │
│  │  └────────────┘   └────────────┘   └────────────┘   ┌──────────┐   │   │
│  │                                                      │  Queue   │───┼───┤
│  │                                                      │ (Batched)│   │   │
│  │                                                      └──────────┘   │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                        │                                     │
│                                  HTTPS (TLS 1.3)                            │
│                           Encrypted Vectors + Metadata                     │
│                                        │                                     │
└────────────────────────────────────────┼────────────────────────────────────┘
                                         │
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         NETDISCVERIT CLOUD                                  │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                         API GATEWAY                                    │   │
│  │                                                                        │   │
│  │  ┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐  │   │
│  │  │ Auth       │   │ REST API   │   │ WebSocket  │   │ Internal   │  │   │
│  │  │ (JWT)      │   │             │   │ (Real-time)│   │ Service    │  │   │
│  │  └────────────┘   └────────────┘   └────────────┘   └──────┬─────┘  │   │
│  │                                                             │         │   │
│  └─────────────────────────────────────────────────────────────┼─────────┘   │
│                                                              │               │
│                              ┌───────────────────────────────┼───────────┐   │
│                              │                               ▼           │   │
│                              │  ┌────────────┐   ┌────────────┐   ┌────────┐ │   │
│                              │  │  ML Model  │   │  LLM       │   │ Config │ │   │
│                              │  │  Service   │   │  Advisor   │   │ Gen    │ │   │
│                              │  └────────────┘   └────────────┘   └────────┘ │   │
│                              │                                                │   │
│                              ▼                                                │   │
│                    ┌─────────────────┐                                      │   │
│                    │    Vector DB     │                                      │   │
│                    │ (Pinecone/       │◀─────────────────────────────────────┘   │
│                    │  Milvus)         │                                          │
│                    └────────┬────────┘                                          │
│                             │                                                   │
│                    ┌────────┴────────┐                                         │
│                    │   PostgreSQL     │                                         │
│                    │ (Metadata, Users)│                                         │
│                    └─────────────────┘                                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Component Details

### Local Agent

| Component | Responsibility | Technology |
|-----------|---------------|------------|
| Collector | Connect to devices via SSH/SNMP | Netmiko, NAPALM |
| Normalizer | Convert config → JSON | Ollama (LLM) |
| Sanitizer | Remove PII/credentials | Custom regex |
| Vectorizer | Generate embeddings | sentence-transformers |
| Scheduler | Run discovery on schedule | Cron |
| Crypto | Encrypt data in transit | TLS 1.3 |

### Cloud API

| Component | Responsibility | Technology |
|-----------|---------------|------------|
| Auth | JWT token management | Python-Jose |
| REST API | CRUD operations | FastAPI |
| ML Service | Device classification | PyTorch, scikit-learn |
| LLM Advisor | Generate recommendations | Claude API, Ollama |
| Config Gen | Create change configs | Jinja2 templates |

### Database

| Database | Purpose |
|----------|---------|
| PostgreSQL | Users, organizations, metadata, device info |
| Vector DB | Embeddings for similarity search |
| Redis | Cache, session store, rate limiting |

---

## Data Flow

### Discovery Flow

```
1. Scheduler triggers discovery
2. Collector connects to device
3. Collector pulls running config
4. Normalizer converts to JSON
5. Sanitizer removes PII
6. Vectorizer creates embeddings
7. Data queued for upload
8. HTTPS POST to cloud API
9. Cloud stores in PostgreSQL + Vector DB
10. Frontend updates
```

### Recommendation Flow

```
1. User requests recommendations
2. API queries Vector DB for similar devices
3. ML model analyzes device metadata
4. LLM generates recommendations
5. User sees recommendations in UI
6. User approves/rejects
7. If approved, Config Gen creates change
8. Agent applies change (via SSH)
```

---

## Security

### Encryption

| Data State | Protection |
|------------|------------|
| In Transit | TLS 1.3 |
| At Rest | AES-256 (database) |
| In Vault | HashiCorp Vault |

### Privacy

- **Local Agent**: Raw configs never leave customer network
- **Cloud**: Only receives sanitized metadata + vectors
- **PII Stripping**: Passwords, keys, credentials removed locally
- **Audit Logs**: Customer can export all data collected

---

## Deployment Options

### Option 1: Cloud (Default)

```
Customer Network          NetDiscoverIT Cloud
┌─────────────┐          ┌─────────────────┐
│ Local Agent │────HTTPS─▶│ API + ML + DB   │
└─────────────┘          └─────────────────┘
```

### Option 2: On-Prem (Enterprise)

```
Customer Network
┌─────────────────────────────────────────────┐
│ Customer's Cloud/Kubernetes                  │
│ ┌─────────────┐     ┌─────────────────┐    │
│ │ Local Agent │────▶│ API + ML + DB   │    │
│ └─────────────┘     └─────────────────┘    │
│                                             │
│ Everything stays on customer infrastructure │
└─────────────────────────────────────────────┘
```

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/devices` | Register device |
| GET | `/api/v1/devices` | List devices |
| GET | `/api/v1/devices/{id}` | Get device details |
| DELETE | `/api/v1/devices/{id}` | Remove device |
| POST | `/api/v1/discoveries` | Trigger discovery |
| GET | `/api/v1/recommendations` | Get AI recommendations |
| POST | `/api/v1/agent/vectors` | Upload vector batch |

---

## Scaling

| Component | Scaling Strategy |
|-----------|------------------|
| Local Agent | Horizontal (one per network) |
| Cloud API | Kubernetes + HPA |
| PostgreSQL | Read replicas |
| Vector DB | Managed service (Pinecone) |
| ML Models | GPU instances on demand |

---

## Related Documentation

- [Metadata Schema](./metadata-schema.md)
- [Local Agent Architecture](./local-agent-architecture.md)
- [Security Design](./security-design.md)
- [Discovery Pipeline](./discovery-pipeline.md)

---

*Architecture is subject to change as we learn.*
