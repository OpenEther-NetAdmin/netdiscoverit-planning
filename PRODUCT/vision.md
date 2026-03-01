# NetDiscoverIT - Product Vision

**Version:** 1.0  
**Date:** 2026-03-01  
**Owner:** Chris Cline (Chy)

---

## The Problem

Network documentation is broken. Every network engineer knows this:

- **Config docs are always stale** — Last update was 6 months ago, nobody remembers what changed
- **Spreadsheets lie** — "I'll update the network diagram tomorrow" → never
- **Tribal knowledge** — Only "Dave" knows why that VLAN exists
- **Security gaps** — EOL equipment, forgotten ACLs, shadow IT

**The result:** Networks that are expensive to maintain, risky to change, and impossible to audit.

---

## The Solution

NetDiscoverIT is an AI-powered platform that automatically discovers, documents, and monitors your network — while keeping your data secure.

### Core Capabilities

1. **Auto-Discovery** — Agentless + agent-based scanning
2. **AI Documentation** — Config → JSON → dynamic diagrams
3. **Smart Classification** — ML determines device roles (that F5 is actually a WAF)
4. **Proactive Alerts** — "This switch hits EOL in 6 months"
5. **Change Recommendations** — "Your routing is suboptimal, here's the fix"
6. **Human-in-the-Loop** — You approve every change

---

## Key Differentiators

| Traditional Tools | NetDiscoverIT |
|-------------------|----------------|
| Snapshots | Continuous monitoring |
| Vendor-specific | Vendor-agnostic |
| Cloud-only | Hybrid (local agent) |
| Manual docs | AI-generated |
| No ML | ML classification |
| Configs stay with vendor | Your data stays yours |

---

## Target Market

### Primary
- **Mid-market enterprises** (500-5000 employees)
- **Managed Service Providers (MSPs)** — managing multiple client networks
- **Healthcare** — HIPAA compliance, network visibility
- **Finance** — PCI-DSS, strict audit requirements

### Secondary
- **Government** — FedRAMP, on-prem requirements
- **Large enterprise** — Complex multi-site networks

---

## Revenue Model

| Tier | Price | Features |
|------|-------|----------|
| **Starter** | $99/mo | Up to 50 devices, basic discovery |
| **Professional** | $299/mo | Unlimited devices, ML recommendations |
| **Enterprise** | Custom | On-prem agent, dedicated support, SLA |

---

## Competitive Landscape

### Competitors
- **SolarWinds NCM** — Expensive, old-school, cloud-only
- **Infoblox** — DNS-centric, not full discovery
- **Microsoft Intune** — Device management, not network
- **Augraphy** — New, similar vision, early stage

### Our Moat
1. **Privacy-first architecture** — Local agent, data stays on-prem
2. **ML classification** — Actually understands what devices do
3. **Self-documenting** — Config → docs automatically
4. **AI advisor** — Not just discovery, but recommendations

---

## Success Metrics

- **MVP**: 10 beta customers
- **Year 1**: 50 paying customers
- **Year 2**: 200 customers, Series A ready
- **Retention**: >90% annual renewal

---

## Next Steps

1. ✅ Architecture design
2. ⬜ MVP scope definition
3. ⬜ Build local agent
4. ⬜ Build cloud API
5. ⬜ ML classification model
6. ⬜ Beta launch

---

*This is a living document. Update as we learn.*
