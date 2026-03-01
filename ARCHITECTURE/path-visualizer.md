# NetDiscoverIT - Path Visualizer Design

**Version:** 1.0  
**Date:** 2026-03-01

---

## 1. Overview

The Path Visualizer is an interactive network mapping tool that allows engineers to input a source and destination, then see the complete path through the network — including L2/L3 hops, VLANs traversed, security zones, ACLs applied, and potential issues.

**Primary Use Case:**
> "I need to get from server A in VLAN 10 to database B in VLAN 30. What path does traffic take? Will it work?"

---

## 2. User Experience

### 2.1 Interface

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  PATH VISUALIZER                                         [Fullscreen] [?]  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────────┐      ┌──────────────────────┐                  │
│  │ Source IP:           │      │ Destination IP:      │                  │
│  │ [10.1.1.50________] │      │ [10.20.30.10______] │  [Trace Path]    │
│  └──────────────────────┘      └──────────────────────┘                  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                                                                     │  │
│  │         ┌───┐         ┌───┐         ┌───────┐         ┌───┐       │  │
│  │         │ S │────────▶│ R1│────────▶│  FW   │────────▶│ D │       │  │
│  │         │ W │         │   │         │       │         │ B │       │  │
│  │         │ 1 │         └─┬─┘         └───┬───┘         └───┘       │  │
│  │         └───┘           │               │                            │  │
│  │                          │               │                            │  │
│  │                    ┌─────▼─────┐   ┌────▼─────┐                      │  │
│  │                    │    ASW    │   │   DMZ    │                      │  │
│  │                    │  (VLAN20) │   │(VLAN99)  │                      │  │
│  │                    └───────────┘   └───────────┘                      │  │
│  │                                                                     │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ PATH DETAILS                                                        │  │
│  ├─────────────────────────────────────────────────────────────────────┤  │
│  │                                                                     │  │
│  │  Hop # | Device      | Interface   | VLAN   | Zone    | Details   │  │
│  │ ───────┼─────────────┼─────────────┼────────┼─────────┼────────── │  │
│  │  1     | SW-1       | Gi1/0/10    | 10     | TRUST   | Access    │  │
│  │  2     | RTR-1      | Gi0/1       | 10→20  | TRUST   | Core      │  │
│  │  3     | FW-1       | inside      | 20→99  | DMZ     | Security  │  │
│  │  4     | SW-2       | Gi1/0/1     | 99→30  | TRUST   | DB tier   │  │
│  │                                                                     │  │
│  │ ─────────────────────────────────────────────────────────────────── │  │
│  │                                                                     │  │
│  │  METRICS                                                           │  │
│  │  • Total Hops: 4          • Est. Latency: 8ms                    │  │
│  │  • VLANs Crossed: 3        • Security Zones: Trust → DMZ         │  │
│  │                                                                     │  │
│  │ ─────────────────────────────────────────────────────────────────── │  │
│  │                                                                     │  │
│  │  ANALYSIS                                                           │  │
│  │  ✅ Routing: All hops have valid routes                            │  │
│  │  ⚠️  ACL Check: Rule 102 (ACL-OUT) may block TCP/443             │  │
│  │  ✅ NAT: Source NAT applied at Rtr-1                               │  │
│  │  ✅ Firewall: Policy permits HTTPS (port 443)                     │  │
│  │                                                                     │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Controls

| Control | Description |
|---------|-------------|
| Source IP | Enter source IP address |
| Destination IP | Enter destination IP address |
| Trace Path | Button to calculate and display path |
| Protocol | Filter: TCP, UDP, ICMP, Any |
| Port | Destination port (optional) |
| View Mode | L2 Topology / L3 Routing / Security Zones |

### 2.3 Interactive Features

- **Hover** — Show device details
- **Click** — Expand device info panel
- **Drag** — Rearrange nodes
- **Zoom** — Pan and zoom map
- **Layer Toggle** — Show/hide VLANs, ACLs, zones

---

## 3. Technical Design

### 3.1 Data Sources

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DATA PIPELINE                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   │
│  │   L2 Info   │   │   L3 Info   │   │  ACL Info  │   │  Zone Info │   │
│  │             │   │             │   │             │   │             │   │
│  │ - CDP/LLDP  │   │ - Routes    │   │ - ACLs     │   │ - Zones    │   │
│  │ - Spanning  │   │ - OSPF/BGP  │   │ - NAT      │   │ - Policies │   │
│  │ - VLANs      │   │ - NAT       │   │ - Firewall │   │ - Objects  │   │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘   │
│         │                  │                  │                  │          │
│         └──────────────────┼──────────────────┼──────────────────┘          │
│                            ▼                                                  │
│                 ┌─────────────────────┐                                    │
│                 │    PATH CALCULATOR    │                                    │
│                 │                       │                                    │
│                 │  1. Find src subnet  │                                    │
│                 │  2. Find dest subnet │                                    │
│                 │  3. L3 route lookup  │                                    │
│                 │  4. L2 path trace    │                                    │
│                 │  5. ACL eval         │                                    │
│                 │  6. Zone check       │                                    │
│                 └──────────┬────────────┘                                    │
│                            │                                                 │
│                            ▼                                                 │
│                 ┌─────────────────────┐                                    │
│                 │    FRONTEND         │                                    │
│                 │    (React + D3)     │                                    │
│                 └─────────────────────┘                                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Path Calculation Algorithm

```
function tracePath(srcIP, dstIP, protocol, port):
    1. Find src device in database
    2. Find dst device in database
    3. Get src device's connected interface
    4. Get dst device's connected interface
    
    # L3 Path
    5. Look up routing table for dstIP
    6. Get next-hop for each hop
    7. Build hop list
    
    # L2 Path
    8. For each L3 hop, trace L2 via MAC table
    9. Build complete path
    
    # Analysis
    10. Check each hop for ACL matches
    11. Check zone transitions
    12. Check NAT translations
    13. Check utilization
    
    return {
        hops: [...],
        vlanChanges: [...],
        zoneChanges: [...],
        aclHits: [...],
        issues: [...],
        metrics: {...}
    }
```

### 3.3 API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/path/trace` | Calculate path |
| GET | `/api/v1/topology` | Get full topology |
| GET | `/api/v1/topology/l2` | L2 topology only |
| GET | `/api/v1/topology/l3` | L3 topology only |
| GET | `/api/v1/devices/{id}/paths` | All paths through device |

### 3.4 Data Models

#### Device
```json
{
  "device_id": "uuid",
  "hostname": "RTR-1",
  "device_type": "router",
  "role": "core",
  "interfaces": [
    {
      "name": "GigabitEthernet0/1",
      "ip_address": "10.1.1.1/24",
      "vlan": 10,
      "zone": "TRUST",
      "mac_address": "aa:bb:cc:dd:ee:ff",
      "connected_device": "SW-1",
      "connected_interface": "Gi1/0/1"
    }
  ]
}
```

#### Route
```json
{
  "device_id": "uuid",
  "prefix": "10.20.30.0/24",
  "next_hop": "10.1.1.2",
  "interface": "GigabitEthernet0/2",
  "protocol": "ospf",
  "metric": 20
}
```

#### ACL
```json
{
  "acl_id": "uuid",
  "name": "ACL-OUT",
  "type": "extended",
  "entries": [
    {
      "sequence": 100,
      "action": "permit",
      "protocol": "tcp",
      "source": "any",
      "destination": "any",
      "destination_port": "443",
      "hit_count": 12345
    }
  ]
}
```

---

## 4. Frontend Implementation

### 4.1 Technology

- **React 19** — UI framework
- **React Flow** or **D3.js** — Network visualization
- **Cytoscape.js** — Graph analysis

### 4.2 Visualization Components

| Component | Library | Purpose |
|-----------|---------|---------|
| Network Graph | React Flow | Interactive node/edge diagram |
| Path Highlight | React Flow | Highlight src → dst path |
| Layer Toggle | Custom | Show/hide L2/L3/Zones |
| Device Popup | Tippy.js | Hover details |
| Mini Map | React Flow | Overview navigation |

### 4.3 State Management

```
┌─────────────────────────────────────────────────────────────────┐
│                      FRONTEND STATE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  topology: {                                                    │
│    nodes: [                                                    │
│      { id, hostname, type, role, x, y, data: {...} }          │
│    ],                                                           │
│    edges: [                                                     │
│      { source, target, label, vlan, type }                    │
│    ]                                                            │
│  }                                                              │
│                                                                  │
│  path: {                                                        │
│    src: "10.1.1.50",                                           │
│    dst: "10.20.30.10",                                         │
│    hops: [ ... ],                                              │
│    issues: [ ... ],                                            │
│    metrics: { ... }                                            │
│  }                                                              │
│                                                                  │
│  view: {                                                        │
│    mode: "L2" | "L3" | "SECURITY",                            │
│    layers: { showVlan: true, showAcl: false, ... },         │
│    zoom: 1.0,                                                  │
│    selectedNode: null                                           │
│  }                                                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Analytics Engine

### 5.1 Path Analysis

For each hop in the path:

| Check | What It Does |
|-------|--------------|
| **Route Valid** | Does routing table have a path? |
| **ACL Permit/Deny** | Does any ACL block this traffic? |
| **Zone Policy** | Does zone policy allow this? |
| **NAT** | Is NAT applied? What translation? |
| **Utilization** | Is interface congested? |
| **MTU** | Any MTU issues? |
| **VLAN** | VLAN allowed on trunk? |

### 5.2 Issue Detection

```
┌─────────────────────────────────────────────────────────────────┐
│                      ISSUE TYPES                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  🔴 BLOCKER                                                      │
│  • ACL explicitly denies traffic                                │
│  • No route to destination                                      │
│  • VLAN not allowed on trunk                                    │
│                                                                  │
│  🟠 WARNING                                                      │
│  • Interface > 80% utilization                                 │
│  • Missing firewall policy (default deny)                      │
│  • EOL equipment in path                                        │
│  • Config not updated > 180 days                                │
│                                                                  │
│  🟡 INFO                                                         │
│  • NAT applied                                                  │
│  • Multiple zones crossed                                       │
│  • High latency hop (>50ms)                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. What-If Scenarios

### 6.1 Engineer Changes Parameters

| Parameter | Example |
|-----------|---------|
| Source IP | "What if traffic comes from .60 instead?" |
| Destination | "What if destination is .11?" |
| Protocol | "What about UDP instead of TCP?" |
| Port | "What if it's port 80 not 443?" |
| VLAN | "What if source moves to VLAN 20?" |

### 6.2 Network Change Simulation

| Change | Effect |
|--------|--------|
| Add VLAN to trunk | Show new paths available |
| Remove route | Highlight unreachable destination |
| Change ACL | Show what's now blocked/allowed |
| Move device | Update topology, re-trace |

---

## 7. Example Output

### Request
```json
POST /api/v1/path/trace
{
  "source_ip": "10.1.1.50",
  "destination_ip": "10.20.30.10",
  "protocol": "tcp",
  "port": 443
}
```

### Response
```json
{
  "path_found": true,
  "hops": [
    {
      "hop": 1,
      "device": {
        "id": "uuid-1",
        "hostname": "SW-ACCESS-1",
        "role": "access",
        "type": "switch"
      },
      "interface": {
        "name": "GigabitEthernet1/0/10",
        "vlan": 10,
        "zone": "TRUST",
        "ip": "10.1.1.1"
      },
      "egress": {
        "name": "GigabitEthernet1/0/24",
        "vlan": 10
      }
    },
    {
      "hop": 2,
      "device": {
        "id": "uuid-2",
        "hostname": "RTR-CORE-1",
        "role": "core",
        "type": "router"
      },
      "interface": {
        "name": "GigabitEthernet0/1",
        "vlan": 10,
        "zone": "TRUST"
      },
      "egress": {
        "name": "GigabitEthernet0/2",
        "vlan": 20,
        "nat": {
          "applied": true,
          "translated_ip": "203.0.113.10"
        }
      },
      "acl_check": {
        "matched": false,
        "acl": "ACL-OUT",
        "rule": null
      }
    },
    {
      "hop": 3,
      "device": {
        "id": "uuid-3",
        "hostname": "FW-EDGE-1",
        "role": "security",
        "type": "firewall"
      },
      "interface": {
        "name": "inside",
        "vlan": 20,
        "zone": "TRUST"
      },
      "egress": {
        "name": "outside",
        "vlan": 99,
        "zone": "DMZ"
      },
      "acl_check": {
        "matched": true,
        "acl": "FW-POLICY",
        "rule": "permit-https",
        "action": "permit"
      }
    },
    {
      "hop": 4,
      "device": {
        "id": "uuid-4",
        "hostname": "SW-DB-1",
        "role": "access",
        "type": "switch"
      },
      "interface": {
        "name": "GigabitEthernet1/0/1",
        "vlan": 99,
        "zone": "DMZ"
      },
      "egress": {
        "name": "GigabitEthernet1/0/20",
        "vlan": 30,
        "connected_device": "uuid-5"
      }
    }
  ],
  "summary": {
    "total_hops": 4,
    "vlan_changes": [
      { "from": 10, "to": 20, "at": "RTR-CORE-1" },
      { "from": 20, "to": 99, "at": "FW-EDGE-1" },
      { "from": 99, "to": 30, "at": "SW-DB-1" }
    ],
    "zone_changes": [
      { "from": "TRUST", "to": "TRUST", "at": "SW-ACCESS-1 → RTR-CORE-1" },
      { "from": "TRUST", "to": "DMZ", "at": "RTR-CORE-1 → FW-EDGE-1" }
    ],
    "estimated_latency_ms": 8,
    "nat_applied": true
  },
  "analysis": {
    "routing": {
      "status": "ok",
      "message": "All hops have valid routes"
    },
    "acl": {
      "status": "ok",
      "message": "Traffic permitted by ACL rules"
    },
    "zones": {
      "status": "warning",
      "message": "Crosses 2 security zones"
    },
    "utilization": {
      "status": "warning",
      "message": "Hop 2 interface at 78% utilization"
    }
  },
  "issues": [
    {
      "severity": "warning",
      "type": "utilization",
      "device": "RTR-CORE-1",
      "interface": "GigabitEthernet0/2",
      "message": "Interface utilization at 78%"
    },
    {
      "severity": "info",
      "type": "zone_crossing",
      "message": "Traffic crosses 2 zones (TRUST → DMZ)"
    }
  ]
}
```

---

## 8. Performance Considerations

| Optimization | Approach |
|--------------|----------|
| **Topology Cache** | Pre-compute and cache full topology |
| **Incremental Updates** | Only recalculate affected paths |
| **Background Jobs** | Heavy analysis runs async |
| **Web Workers** | Path calc in browser thread |
| **Graph DB** | Neo4j for fast path queries |

---

## 9. Future Enhancements

- **Historical Paths** — Show path at different times
- **Traffic Simulation** — Import NetFlow, show actual traffic
- **Change Impact** — "What if I disable this link?"
- **Multi-Path** — Show all equal-cost paths
- **SLA Calc** — Add latency/jitter requirements

---

## 10. Dependencies

| Feature | Requires |
|---------|----------|
| Full path calc | L2 + L3 + ACL data collected |
| ACL analysis | Config parsing working |
| Zone analysis | Firewall configs parsed |
| Utilization | SNMP polling enabled |
| Historical | Time-series database |

---

## 11. Implementation Order

1. **Topology View** — Basic network map
2. **Static Path** — Hardcoded src/dst
3. **L3 Routing** — Route lookup
4. **L2 Traces** — MAC table lookup
5. **ACL Check** — Config parsing
6. **Zone Analysis** — Firewall integration
7. **What-If** — Parameter changes

---

## 12. Related Documents

- [MVP](./MVP.md) — Phase 1 scope
- [Phase 2 Roadmap](./phase-2.md) — NetFlow, integrations
- [Architecture Overview](../ARCHITECTURE/overview.md) — System design

---

*This is the heart of the user-facing experience — giving engineers instant answers about their network.*
