# Metadata Schema Design

**Version:** 1.0  
**Created:** 2026-03-01  
**Project:** NetDiscoverIT - Self-Documenting Network Platform

---

## 1. Overview

This document defines the vendor-agnostic metadata schema used by NetDiscoverIT to normalize network device configurations. The schema serves as the foundation for ML-based analysis, role classification, and intelligent recommendations while ensuring sensitive data never leaves the customer's network.

---

## 2. Goals

1. **Vendor Agnostic** — Support Cisco, Juniper, Arista, F5, Palo Alto, etc.
2. **Privacy First** — No credentials, passwords, or raw configs leave the local network
3. **ML-Ready** — Structured data optimized for vector search and classification
4. **Self-Documenting** — Enables automated network documentation and change tracking

---

## 3. Data Flow

```
┌────────────────┐     ┌─────────────┐     ┌────────────┐     ┌─────────────┐
│  Raw Config   │────▶│ Sanitizer   │────▶│  Metadata  │────▶│  Vector DB  │
│  (CLI Output) │     │ (PII Strip) │     │  Schema    │     │  (Embeddings│
└────────────────┘     └─────────────┘     └────────────┘     └─────────────┘
                                │                                    │
                                ▼                                    ▼
                         ┌─────────────┐                      ┌─────────────┐
                         │  Discard    │                      │  ML/LLM     │
                         │  PII/Creds  │                      │  Analysis   │
                         └─────────────┘                      └─────────────┘
```

---

## 4. Sanitization Rules

Before conversion to metadata, all configs are sanitized:

| Pattern | Example | Replacement |
|---------|---------|-------------|
| `password *` | `username admin password Secret123` | `username admin password ***` |
| `community *` | `snmp-server community public` | `snmp-server community ***` |
| `secret *` | `aaa secret 5 $1$xyz` | `aaa secret ***` |
| `key *` | `ip ssh key dsa 1024` | `ip ssh key ***` |
| `username *` | `username admin` | `username [USER]` |
| `hostname *` | `hostname CORE-RTR-01` | `hostname [HOST]` |
| `interface * description *` | `description Uplink to core` | `description [DESC]` (if sensitive) |

**Implementation Note:** Sanitization happens locally, before any network transmission.

---

## 5. Core Schema

### 5.1 Device Object

```json
{
  "device_id": "uuid-v4",
  "scan_timestamp": "2026-03-01T12:00:00Z",
  "collection_method": "ssh | snmp | api",
  
  "identification": {
    "hostname": "string",
    "management_ip": "10.0.0.1",
    "serial_number": "string (hashed)",
    "mac_address": "string (hashed)"
  },
  
  "hardware": {
    "vendor": "cisco | juniper | arista | f5 | palo_alto | hp | dell | ubiquiti",
    "model": "Catalyst 9300 | MX204 | 7050X3 | ... ",
    "part_number": "string",
    "chassis_slots": 8,
    "power_supplies": 2,
    "fan_trays": 3
  },
  
  "software": {
    "os_type": "ios | ios-xe | junos | eos | nx-os | pan-os | ... ",
    "os_version": "17.9.1",
    "os_build": "string",
    "bootflash_total_mb": 8192000,
    "bootflash_used_mb": 2048000
  },
  
  "runtime": {
    "uptime_seconds": 2592000,
    "cpu_percent": 12,
    "memory_total_mb": 4096,
    "memory_used_mb": 1843,
    "temperature_celsius": 42
  },
  
  "role": {
    "detected": "core | distribution | access | edge | dmz | wireless | security",
    "confidence": 0.92,
    "ml_model_version": "v1.2.0"
  },
  
  "classification": {
    "device_type": "switch | router | firewall | load_balancer | wap | ip_phone | camera | iot",
    "detected_apps": ["web_server", "database", "api_gateway"],
    "ports_in_use": [443, 22, 161],
    "service_profile": "WAF | LTM | APM | GTM | ..."  // F5-specific detection
  },
  
  "health": {
    "eol_risk": "low | medium | high | critical",
    "eol_date": "2027-12-31",
    "eos_date": "2029-12-31",
    "firmware_outdated": true,
    "recommended_firmware": "17.12.1",
    "config_aging_days": 180,
    "last_config_change": "2025-09-01T14:30:00Z"
  },
  
  "interfaces": [...],
  "vlans": [...],
  "routing": {...},
  "security": {...}
}
```

### 5.2 Interface Object

```json
{
  "name": "GigabitEthernet1/0/1",
  "type": "physical | vlan | loopback | tunnel | port_channel | svi",
  "status": "up | down | administratively_down",
  "ip_address": "10.1.1.1/24",
  "ip_address_hashed": "a1b2c3d4...",  // optional
  
  "physical": {
    "speed_mbps": 1000,
    "duplex": "full | half",
    "medium": "copper | fiber",
    "transceiver_type": "SR | LR | DAC",
    "auto_negotiate": true,
    "mtu": 9000
  },
  
  "layer2": {
    "vlan": 10,
    "voice_vlan": 20,
    "switchport_mode": "access | trunk | dynamic",
    "trunk_vlans": "10,20,30",
    "native_vlan": 1,
    "spanning_tree": "portfast | guard | ...",
    "bpduguard": true,
    "dhcp_snooping_trusted": true
  },
  
  "layer3": {
    "vrf": "CORP | GUEST",
    "ospf_area": "0",
    "bgp_neighbor": "10.1.1.2",
    "acl_inbound": "ACL-IN",
    "acl_outbound": "ACL-OUT"
  },
  
  "usage": {
    "tx_bytes": 1234567890,
    "rx_bytes": 9876543210,
    "tx_errors": 0,
    "rx_errors": 0,
    "utilization_percent": 45.2,
    "peak_utilization_percent": 78.3
  },
  
  "connected_to": {
    "device_hashed": "abc123...",
    "interface": "Eth1/1",
    "cable_length_m": 5,
    "discovery_method": "cdp | lldp | manual"
  },
  
  "description": "Uplink to Core Switch",
  "is_monitoring_enabled": true,
  "is_management": false
}
```

### 5.3 VLAN Object

```json
{
  "vlan_id": 10,
  "name": "CORPORATE",
  "description": "Main corporate network",
  "status": "active | suspended",
  "type": "default | data | voice | mgmt | guest",
  
  "ipv4_subnets": [
    {
      "network": "10.10.10.0/24",
      "gateway": "10.10.10.1",
      "dhcp_relay": "10.10.10.53"
    }
  ],
  
  "ipv6_subnets": [...],
  
  "ports": ["Gi1/0/1", "Gi1/0/2", "Gi1/0/5"],
  "svi_ip": "10.10.10.1/24",
  "is_management_vlan": false,
  "is_native_vlan": false
}
```

### 5.4 Routing Object

```json
{
  "protocols": ["ospf", "bgp"],
  
  "ospf": {
    "enabled": true,
    "process_id": 1,
    "router_id": "10.0.0.1",
    "area": "0",
    "network_type": "broadcast",
    "passive_interfaces": ["Loopback0"],
    "neighbors": [
      {
        "neighbor_id": "10.0.0.2",
        "priority": 1,
        "state": "FULL",
        "dead_timer": 40
      }
    ]
  },
  
  "bgp": {
    "enabled": true,
    "asn": 65001,
    "router_id": "10.0.0.1",
    "neighbors": [
      {
        "peer_asn": 65002,
        "remote_address": "10.1.1.2",
        "state": "Established",
        "prefixes_received": 150,
        "prefixes_sent": 75
      }
    ]
  },
  
  "static_routes": [
    {
      "destination": "0.0.0.0/0",
      "next_hop": "10.1.1.1",
      "interface": "GigabitEthernet0/0",
      "distance": 1
    }
  ],
  
  "routing_table": [
    {
      "prefix": "10.10.10.0/24",
      "next_hop": "10.0.0.2",
      "protocol": "ospf",
      "metric": 20
    }
  ]
}
```

### 5.5 Security Object

```json
{
  "zones": [
    {
      "name": "TRUST",
      "interfaces": ["Gi1/0/1", "Gi1/0/2"],
      "policy": "permit-outbound"
    },
    {
      "name": "UNTRUST", 
      "interfaces": ["Gi0/0"],
      "policy": "deny-inbound"
    }
  ],
  
  "acls": [
    {
      "name": "ACL-IN",
      "type": "extended",
      "entries": [
        {
          "action": "permit",
          "protocol": "tcp",
          "source": "any",
          "destination": "any",
          "destination_port": "443"
        }
      ],
      "hit_count": 123456
    }
  ],
  
  "nat": {
    "inside_interfaces": ["Gi1/0/1"],
    "outside_interfaces": ["Gi0/0"],
    "rules": [
      {
        "original_ip": "10.0.0.50",
        "translated_ip": "203.0.113.50",
        "type": "static | dynamic",
        "port": 443
      }
    ]
  },
  
  "services": {
    "ssh_enabled": true,
    "ssh_version": 2,
    "snmp_enabled": true,
    "snmp_version": "v2c",
    "http_server_enabled": false,
    "https_server_enabled": true,
    "dns_servers": ["8.8.8.8", "1.1.1.1"],
    "ntp_servers": ["time.google.com"]
  }
}
```

### 5.6 ML Classification Object

```json
{
  "device_role": {
    "prediction": "core",
    "confidence": 0.94,
    "features_used": ["uplink_count", "routing_protocols", "vlan_count"],
    "model_version": "v1.2.0"
  },
  
  "application_profile": {
    "prediction": ["web_server", "api_gateway"],
    "confidence": 0.87,
    "ports_detected": [443, 8080, 8443],
    "config_patterns_detected": ["load_balancer_pool", "ssl_profile"]
  },
  
  "anomaly_score": {
    "value": 0.12,
    "threshold": 0.7,
    "flags": ["high_cpu_7d_avg", "config_not_changed_180d"]
  },
  
  "eol_prediction": {
    "risk": "medium",
    "current_support_end": "2027-06-30",
    "predicted_eol": "2028-Q2",
    "confidence": 0.78
  }
}
```

---

## 6. Vector Embedding Strategy

Each device generates multiple vectors for different query types:

| Vector Type | Source Data | Use Case |
|-------------|-------------|----------|
| `device_role` | Hardware + interface count + routing | "Find all core switches" |
| `topology` | Interface connections + VLANs | Network mapping |
| `config_pattern` | Sanitized config tokens | Anomaly detection |
| `security_posture` | ACLs + zones + services | Security assessment |
| `health` | CPU + memory + uptime + firmware | Maintenance planning |

---

## 7. API Response Format

### 7.1 Device List

```json
{
  "devices": [
    {
      "device_id": "uuid",
      "hostname": "CORE-RTR-01",
      "management_ip": "10.0.0.1",
      "vendor": "cisco",
      "device_type": "router",
      "role": "core",
      "health": {
        "eol_risk": "low",
        "firmware_outdated": false
      }
    }
  ],
  "total": 42,
  "page": 1,
  "per_page": 20
}
```

### 7.2 Network Map

```json
{
  "nodes": [
    {
      "id": "uuid",
      "hostname": "CORE-SW-01",
      "role": "core",
      "vendor": "cisco"
    }
  ],
  "links": [
    {
      "source": "uuid-1",
      "target": "uuid-2",
      "interface_a": "Gi1/0/1",
      "interface_b": "Eth1/1",
      "type": "physical"
    }
  ]
}
```

### 7.3 Recommendations

```json
{
  "recommendations": [
    {
      "device_id": "uuid",
      "severity": "high",
      "category": "security",
      "title": "Firmware Outdated",
      "description": "Device running IOS 17.9.1 has known vulnerabilities. Recommended: 17.12.1",
      "action": "update_firmware",
      "auto_applicable": false,
      "config_impact": {
        "requires_downtime": true,
        "rollback_available": true
      }
    }
  ]
}
```

---

## 8. Implementation Notes

### 8.1 Local Processing (Customer Network)

- Sanitization runs locally before any transmission
- Metadata schema is versioned for compatibility
- Customer can audit all collected data

### 8.2 Cloud Processing

- Receives only metadata + vectors (never raw configs)
- ML models run on structured data
- Results cached and returned to customer

### 8.3 Schema Versioning

```json
{
  "schema_version": "1.0.0",
  "netdiscoverit_version": "2.0.0",
  "compatibility": ["1.0.x", "1.1.x"]
}
```

---

## 9. Related Documentation

- [Security Design](./security-design.md) - PII handling and encryption
- [Discovery Pipeline](./discovery-pipeline.md) - Data collection process
- [ML Classification Model](./ml-classification.md) - Role detection algorithms
- [Architecture Overview](../architecture-diagrams/system-overview.md) - System context

---

## 10. Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-03-01 | NetDiscoverIT Team | Initial schema design |
