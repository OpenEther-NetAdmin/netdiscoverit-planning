# NetDiscoverIT - MVP Definition

**Version:** 1.0  
**Date:** 2026-03-01

---

## MVP Goal

Build the smallest product that validates the core value proposition:
> "We automatically discover your network and turn configs into living documentation"

---

## Scope: What We're Building

### Core Features (MVP)

| Feature | Description | Priority |
|---------|-------------|----------|
| **Local Agent** | Python package that runs on customer network | P0 |
| **Device Discovery** | SSH/SNMP to find devices | P0 |
| **Config Collection** | Pull running config via SSH | P0 |
| **JSON Normalization** | LLM converts config → JSON | P0 |
| **Sanitization** | Strip PII before transmission | P0 |
| **Vector Generation** | Create embeddings locally | P0 |
| **Cloud API** | Receive vectors, store metadata | P0 |
| **Device List** | Web UI showing discovered devices | P0 |
| **Basic Topology** | Node/link diagram | P1 |

### Not in MVP

- ❌ ML classification (Phase 2)
- ❌ AI recommendations (Phase 2)
- ❌ Auto-change generation (Phase 2)
- ❌ Multi-tenancy (Phase 2)
- ❌ Mobile app (Phase 3)
- ❌ On-prem deployment (Phase 2)

---

## Technical Architecture (MVP)

```
┌─────────────────────┐     ┌─────────────────────┐
│  LOCAL AGENT        │     │  CLOUD              │
│  (Customer Network) │     │  (NetDiscoverIT)    │
├─────────────────────┤     ├─────────────────────┤
│  - Collector        │     │  - API (FastAPI)    │
│  - Normalizer       │────▶│  - PostgreSQL       │
│  - Sanitizer        │     │  - Vector DB       │
│  - Vectorizer       │     │  - React Frontend  │
└─────────────────────┘     └─────────────────────┘
```

---

## User Flow (MVP)

1. **Sign up** → Create account on netdiscoverit.com
2. **Download Agent** → Get Docker command
3. **Run Agent** → `docker run -e API_KEY=xxx netdiscoverit/agent`
4. **Discovery Runs** → Agent polls network
5. **View Results** → See device list in web UI

---

## Data Flow (MVP)

```
1. Agent connects to device (SSH)
2. Agent pulls config
3. Agent sanitizes (local)
4. Agent generates vector (local)
5. Agent sends vector to cloud (HTTPS)
6. Cloud stores in PostgreSQL + Vector DB
7. User views in web UI
```

---

## MVP Tech Stack

| Component | Technology |
|-----------|------------|
| Local Agent | Python 3.11+ |
| Agent LLM | Ollama (Llama 3.2 7B) |
| Cloud API | FastAPI |
| Database | PostgreSQL |
| Vector DB | pgvector (or Pinecone) |
| Frontend | React 19 |
| Auth | JWT |
| Deployment | Docker |

---

## Acceptance Criteria

### Agent
- [ ] Connects via SSH to Cisco/Juniper/Arista
- [ ] Pulls running config
- [ ] Sanitizes passwords/keys
- [ ] Generates device metadata JSON
- [ ] Creates vector embedding
- [ ] Sends to cloud API
- [ ] Runs on Docker

### Cloud API
- [ ] Accepts device metadata POST
- [ ] Stores in PostgreSQL
- [ ] Returns device list GET
- [ ] JWT authentication

### Frontend
- [ ] Login/Signup
- [ ] Dashboard showing device count
- [ ] Device list with basic info
- [ ] Simple topology view

---

## Timeline (MVP)

| Week | Milestone |
|------|-----------|
| 1 | Agent skeleton + SSH connect |
| 2 | Config pull + JSON normalization |
| 3 | Sanitizer + vectorizer |
| 4 | Cloud API + database |
| 5 | Frontend basic UI |
| 6 | Integration testing |
| 7 | Bug fixes |
| 8 | Beta launch |

---

## What's Next (Phase 2)

- ML device classification
- AI recommendations
- Multi-tenancy
- More device vendors
- On-prem deployment option

---

*This defines our boundaries. Stay focused.*
