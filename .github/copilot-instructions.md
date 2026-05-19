# Copilot Instructions — IEEE 1588 PTP Analyzer Appliance

## Project Overview

This is the **SyncAuto AI** platform: a dedicated appliance for monitoring, analyzing, and troubleshooting IEEE 1588 Precision Time Protocol (PTP) environments. It targets telecom operators (5G/backhaul), industrial automation (SCADA/PLC), financial trading infrastructure (MiFID II compliance), and datacenters.

The system operates in two modes:
- **Offline** — analyzing uploaded PCAP/PCAPNG files
- **Online** — analyzing live mirrored/SPAN/TAP network traffic in near real-time

This repository is currently in the **planning/documentation phase**. The README contains the full PRD. Implementation follows the sprint plan at the end of the README.

---

## Planned Architecture

The system has four primary layers:

```
Web UI Dashboard (React + Grafana/ECharts + WebSockets)
        ↓
Analytics Backend (Python FastAPI — API, AI engine)
  • Packet Parser / Protocol Engine   → Rust or C++
  • Timing Analysis Engine            → Rust
  • Anomaly Detection / AI Engine     → Python
  • Alerting & Reporting Engines      → Python
        ↓
Packet Capture Layer
  • PCAP Import (libpcap)
  • Live NIC Capture (AF_PACKET → DPDK → PF_RING)
        ↓
Storage Layer
  • Metrics:      InfluxDB or VictoriaMetrics
  • Events:       PostgreSQL
  • Raw packets:  PCAP files
  • Logs:         Loki or OpenSearch
```

**POC hardware:** NVIDIA Jetson Orin Nano (GPU acceleration, edge deployment). Production targets x86 + NVIDIA GPU + dual 10G/25G NICs.

---

## Key Domain Concepts

- **PTP message types in scope:** Sync, Follow_Up, Delay_Req, Delay_Resp, Announce, Signaling, Management
- **Clock roles to track:** Grandmaster, Boundary Clock, Transparent Clock
- **Primary anomalies to detect:** offset instability, frequency drift, delay asymmetry, BMCA flapping, missing/duplicate/out-of-order packets, malformed packets
- **Protocol priority:** IEEE 1588v2 (critical) → SyncE (high) → NTP (medium) → RTP timestamps (optional)

---

## Technology Decisions

| Component | Technology |
|---|---|
| High-performance packet parsing | Rust or C++ |
| Stream processing | Rust |
| API server | Python FastAPI |
| AI/ML engine | Python |
| Live capture (initial) | libpcap / AF_PACKET |
| Live capture (scale) | DPDK / PF_RING |
| Frontend | React |
| Charts | Grafana or ECharts |
| Real-time updates | WebSockets |
| Metrics DB | InfluxDB or VictoriaMetrics |
| Events DB | PostgreSQL |
| Base OS | Ubuntu Server |

---

## MVP Scope (Phase 1 + Phase 2)

**Included:** PCAP upload and parsing, live capture, dashboards, alerting, graphs, export reports.

**Explicitly excluded from MVP:** inline transparent mode, distributed cluster mode, AI predictive analytics, multi-appliance federation.

---

## Core Data Models

### Sync Health Score
The primary composite metric surfaced on the dashboard. It is a weighted score derived from four inputs:

| Field | Description |
|---|---|
| `offset_from_master` | Deviation (ns/μs) from the primary clock |
| `pdv` | Packet Delay Variation — jitter impacting sync stability |
| `gnss_snr` | GNSS signal-to-noise ratio and interference indicators |
| `clock_class` | Hierarchical health/quality level of the clock |

### PTP Session Record
Reconstructed from packet analysis. Key fields:
- `grandmaster_id` — Clock identity of the active Grandmaster
- `clock_class` / `clock_accuracy` / `priority1` / `priority2` — BMCA election fields
- `domain_id` — PTP domain number (mismatches indicate Timing Islands)
- `offset_ns` — Measured offset from master
- `mean_path_delay_ns` — Measured one-way delay
- `pdv_ns` — Packet delay variation
- `message_rates` — Per-message-type rates (Sync, Announce, Delay_Req, etc.)

### Anomaly / Event Record
- `anomaly_type` — e.g., `gmFlapping`, `delayAsymmetry`, `pdvSpike`, `timingIsland`, `holdoverDegradation`, `gnssInterference`, `pathAsymmetry`, `protocolViolation`
- `severity` — threshold-relative severity level
- `timestamp`, `duration`
- `affected_clock_ids`
- `root_cause_suggestion` — AI-generated reasoning string

### Remediation Playbook Entry
- `failure_mode` — maps to anomaly type
- `vendor` — Cisco / Nokia / Juniper / Ericsson
- `cli_commands` — ordered list of vendor-specific CLI commands
- `standard_ref` — e.g., `ITU-T G.8275.1`, `IEEE 1588v2`

### Inter-Component Contract (The Hands → The Interface)
All data passed from the Agentic Workflow Engine to the dashboard must be **structured JSON** containing both the Sync Health Score value and the reasoning context ("why" behind the score). Truncation or data loss in this hand-off is a hard verification failure.

---

## Input Data Formats

The Agentic Workflow Engine must parse all of:
- **PCAP / PCAPNG** — deep PTP packet timing analysis
- **CLI config exports** — Cisco, Nokia, Juniper, Ericsson syntax
- **NMEA logs** — GNSS signal data with SNR and interference indicators

---

## Standards Coverage

| Standard | Focus |
|---|---|
| IEEE 1588v2 | Core PTP |
| ITU-T G.8275.1 | Full Timing Support (FTS) telecom profile |
| ITU-T G.8275.2 | Partial Timing Support (PTS) telecom profile |
| ITU-T G.8262 | SyncE / ESMC clock quality messaging |
| IEEE C37.238 | Power profile PTP |
| IEC 61850 | Substation automation phase-accurate PTP |
| MiFID II / FINRA | UTC clock traceability, audit trails |
| SMPTE ST 2110 | IP-video broadcast sync |

---

## Performance Targets

| Metric | Target |
|---|---|
| Live analysis latency | < 1 second |
| Packet loss | < 0.01% |
| UI refresh rate | 1 second |
| PCAP processing throughput | 1 GB/min (initial) |

---

## Security Requirements

- Authentication: local accounts, LDAP (SSO deferred)
- Transport: TLS on all APIs
- Authorization: RBAC
- Audit: user action logging, config change tracking
