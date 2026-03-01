# NetDiscoverIT - Phase 2: Network Observability

**Version:** 1.0  
**Date:** 2026-03-01

---

## Overview

Phase 2 extends NetDiscoverIT beyond configuration discovery to full network observability. This includes traffic flow analysis, vulnerability assessment, and integration with existing open-source tools.

---

## NetFlow/sFlow/IPFIX Integration

### Problem

Configuration-only discovery shows what *should* be there. Traffic flow analysis shows what *actually* is.

### Solution

Add flow collection to the Local Agent:

```
┌─────────────────────────────────────────┐
│         LOCAL AGENT (Enhanced)           │
├─────────────────────────────────────────┤
│  Existing:                              │
│  ├─ SSH config pull                     │
│  ├─ SNMP polling                        │
│  └─ LLM normalization                   │
│                                          │
│  NEW:                                    │
│  ├─ sFlow receiver (port 9993)          │
│  ├─ NetFlow receiver (port 9994)        │
│  ├─ IPFIX receiver (port 9995)          │
│  ├─ Flow parser                          │
│  └─ Traffic vector generator             │
└─────────────────────────────────────────┘
```

### Supported Protocols

| Protocol | Standard | Vendors |
|----------|----------|---------|
| sFlow | v5 | Arista, HP, Juniper |
| NetFlow | v5/v9 | Cisco, Juniper |
| IPFIX | RFC 7011 | Modern devices |

### Data Collected

```json
{
  "flow": {
    "src_ip": "10.1.1.100",
    "dst_ip": "10.2.2.200",
    "src_port": 443,
    "dst_port": 8080,
    "protocol": "tcp",
    "bytes": 1048576,
    "packets": 1524,
    "tos": 0,
    "flags": "ACK PSH",
    "start_time": "2026-03-01T12:00:00Z",
    "end_time": "2026-03-01T12:00:10Z"
  }
}
```

### ML Enhancement

Traffic patterns improve classification:

| Config Shows | Flow Reveals | ML Deduces |
|--------------|-------------|------------|
| F5 with VIPs | HTTPS → 8443 | WAF role |
| Core router | BGP keepalives | Edge/peering |
| Access switch | 95% to single IP | Uplink to core |
| Server VLAN | External IPs | DMZ/Internet-facing |

### Implementation

- Use `goflow`, `ntopng`, or `pmacct` as flow collector
- Embed in Local Agent Docker image
- Parse flows → generate traffic vectors
- Upload to cloud for analysis

### Timeline

- **Week 1-2**: Flow receiver implementation
- **Week 3-4**: Traffic vector generation
- **Week 5-6**: ML model training on flow data
- **Week 7-8**: Integration and testing

---

## Open-Source Tool Integrations

### Network Scanning & Discovery

| Tool | Use Case | Integration |
|------|----------|-------------|
| **Nmap** | Port scanning, OS detection | Agent runs pre-discovery |
| **Masscan** | Fast Internet-scale scans | Large network sweeps |
| **RANCID/Oxidized** | Config backup/history | Import historical configs |

### Infrastructure Management

| Tool | Use Case | Integration |
|------|----------|-------------|
| **NetBox** | IPAM/DCIM, network docs | Export to NetBox API |
| **Ansible** | Config management | Agent runs playbooks |
| **Terraform** | Infrastructure as code | Generate TF from discovery |

### Monitoring & Observability

| Tool | Use Case | Integration |
|------|----------|-------------|
| **Prometheus** | Metrics collection | Export device metrics |
| **Grafana** | Visualization | NetDiscoverIT dashboard |
| **Zabbix/LibreNMS** | Network monitoring | Device import |
| **ntopng** | Flow collection | (Or build our own) |

### Security

| Tool | Use Case | Integration |
|------|----------|-------------|
| **OpenVAS** | Vulnerability scanning | Run post-discovery |
| **Zeek** | Network analysis | Import flow data |
| **Suricata** | IDS/IPS | Threat detection |

### AI/ML

| Tool | Use Case | Integration |
|------|----------|-------------|
| **Ollama** | Local LLM (already in plan) | Config → JSON |
| **LangChain** | LLM orchestration | Recommendation generation |
| **PyTorch** | ML training | Role classification |
| **scikit-learn** | ML models | Anomaly detection |

---

## Integration Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         EXTERNAL TOOLS LAYER                                  │
│                                                                              │
│   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐    │
│   │  Nmap   │   │ NetBox  │   │Ansible  │   │  Open   │   │  Grafana │   │
│   │         │   │         │   │         │   │  VAS    │   │         │   │
│   └────┬────┘   └────┬────┘   └────┬────┘   └────┬────┘   └────┬────┘    │
│        │             │             │             │             │          │
│        └─────────────┴─────────────┴─────────────┴─────────────┘          │
│                                      │                                      │
└──────────────────────────────────────┼──────────────────────────────────────┘
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         NETDISCVERIT PLATFORM                                 │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                         LOCAL AGENT                                    │  │
│   │                                                                       │  │
│   │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  │  │
│   │  │ Scanner │  │ Config  │  │  Flow   │  │ Vuln    │  │  API    │  │  │
│   │  │ Module  │  │ Module  │  │ Module  │  │ Module  │  │ Module  │  │  │
│   │  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  │  │
│   │       └────────────┴────────────┴────────────┴────────────┘        │  │
│   │                               │                                       │  │
│   │                        ┌──────┴──────┐                                │  │
│   │                        │   Unified   │                                │  │
│   │                        │   Vector DB  │                                │  │
│   │                        └─────────────┘                                │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Priority: NetBox Integration

NetBox is the highest-value integration. It's an open-source IPAM/DCIM that already does:

- Device inventory
- Rack diagrams
- Cable tracking
- IP address management
- VLAN management

### Integration Path

```
NetDiscoverIT                    NetBox
     │                              │
     │  1. Discovery complete       │
     │─────────────────────────────▶│
     │                              │
     │  2. "Create/update devices"  │
     │◀─────────────────────────────│
     │                              │
     │  3. Sync: devices,           │
     │        interfaces,          │
     │        IPs, VLANs            │
     │─────────────────────────────▶│
```

### Value

- Customers who already use NetBox get instant documentation
- NetDiscoverIT becomes the "discovery engine" for NetBox
- Two products working together

---

## Value Comparison

| Integration | Complexity | Value | Priority |
|------------|------------|-------|----------|
| NetBox | Medium | High | P0 |
| NetFlow/sFlow | Medium | High | P0 |
| Nmap | Low | Medium | P1 |
| Ansible | Medium | Medium | P1 |
| Prometheus | Low | Medium | P1 |
| OpenVAS | High | Medium | P2 |
| Grafana | Low | Medium | P2 |
| Zeek | High | Low | P2 |

---

## Roadmap Summary

### Phase 2 Features

1. **NetFlow/sFlow Collection** — Traffic flow analysis
2. **NetBox Integration** — Sync discovered data to DCIM
3. **Nmap Pre-Scan** — Find devices before deep discovery
4. **Basic Vulnerability Check** — Quick CVEs based on OS version
5. **Prometheus Export** — Pull device metrics

---

## Related Documents

- [MVP](./MVP.md) — Phase 1 scope
- [Architecture Overview](../ARCHITECTURE/overview.md) — System design

---

*Phase 2 transforms NetDiscoverIT from a discovery tool to a full network observability platform.*
