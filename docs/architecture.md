# Starburst Sizing Skill — Architecture Documentation

**Skill:** `/starburst-sizing`
**File:** `.claude/commands/starburst-sizing.md`
**Scope:** SEP on-prem (K8s / OpenShift / VM) — Galaxy as secondary target
**Methodology:** GTM Playbook v1.0 — Citi T-shirt grid — linear bandwidth model

---

## Overview

The skill sizes a Starburst Enterprise Platform cluster from customer inputs. It operates in 6 phases: intake → parallel computation → configuration → output → peer review → save. Customer-facing inputs use pure business language; all translation to Starburst parameters happens internally.

---

## Architecture diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│  PHASE A — INTAKE                                                   │
│                                                                     │
│  Inputs provided?                                                   │
│    YES → translation table → Phase B                               │
│    NO  → output intake form (5 sections, business language)        │
│           → customer fills → translation table → Phase B           │
│                                                                     │
│  Translation table: business answers → sizing parameters           │
│  (concurrency, SLA pts, volume pts, workload type, DR mode,        │
│   deployment, Warp Speed candidate, flags)                         │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ sizing parameters
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  PHASE B — PARALLEL SIZING COMPUTATION                             │
│                                                                     │
│  Spawn 3 agents in a single message (parallel):                    │
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────────┐   │
│  │ Agent 1         │  │ Agent 2          │  │ Agent 3          │   │
│  │ T-SHIRT SCORER  │  │ NETWORK CHECKER  │  │ WARP SPEED SIZER │   │
│  │                 │  │                  │  │ (conditional)    │   │
│  │ In: concurrency │  │ In: vol_eff,     │  │ In: hot_data,    │   │
│  │     volume      │  │     concurrency, │  │     NVMe_GB,     │   │
│  │     SLA         │  │     SLA,         │  │     hit_rate,    │   │
│  │     workload    │  │     confirmed    │  │     profile      │   │
│  │     growth_pct  │  │                  │  │                  │   │
│  │                 │  │ Out: network_    │  │ Out: cache_      │   │
│  │ Out: t_shirt,   │  │      workers,    │  │      workers_    │   │
│  │  worker_range,  │  │      Gbps,       │  │      min,        │   │
│  │  workers_growth │  │      skip_reason │  │      node_type   │   │
│  │  memory_config  │  │                  │  │                  │   │
│  └────────┬────────┘  └────────┬─────────┘  └────────┬─────────┘   │
│           │                   │                      │             │
│           └───────────────────┴──────────────────────┘             │
│                               │ MERGE                              │
│   workers_retained = max(3, agent1, agent2, agent3)                │
│                    ↑                                               │
│             hard floor: 3 workers minimum (Starburst doc)          │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ workers_retained + all agent outputs
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  PHASE C — CONFIGURE                                                │
│                                                                     │
│  Node constraints (Starburst doc):                                  │
│    • Min node: 16 vCPU / 64 GB RAM                                 │
│    • Min workers in production: 3                                   │
│    • All worker nodes homogeneous (identical spec)                  │
│    • Warp Speed: ALL workers NVMe — no mixing                       │
│                                                                     │
│  Config steps (sequential):                                         │
│    1. SEP cluster config (coordinator + workers)                    │
│    2. Autoscaling recommendation (per workload type)                │
│    3. Growth headroom: ceil(workers × (1 + growth%))               │
│    4. Multi-environment (Prod / DR / UAT / DEV)                     │
│    5. DR sizing (Active/Active | Warm | Cold)                       │
│    6. Node type (standard vs NVMe)                                  │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  PHASE D — OUTPUT                                                   │
│                                                                     │
│  Sections:                                                          │
│    • WORKLOAD INPUTS (workload type, bottleneck, metrics)           │
│    • T-SHIRT SCORING (score breakdown, workers)                     │
│    • NETWORK CHECK (Gbps, dominant constraint)                      │
│    • PRODUCTION RECOMMENDATION (node spec, count, memory config)   │
│    • MULTI-ENVIRONMENT table                                        │
│    • WARP SPEED / CACHING (omit if inactive)                       │
│    • POV STARTING CONFIG                                            │
│    • PRE-POV CHECKLIST                                              │
│    • FLAGS (only applicable ones)                                   │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ Phase D output
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  PHASE D.5 — PARALLEL PEER REVIEW (HITL)                           │
│                                                                     │
│  Spawn 6 agents in a single message (parallel):                    │
│                                                                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ ┌────┐ ┌────┐  │
│  │V1        │ │V2        │ │V3        │ │V4      │ │V5  │ │V6  │  │
│  │WORKLOAD  │ │T-SHIRT   │ │NETWORK   │ │WARP    │ │DR  │ │FLAGS│ │
│  │COHERENCE │ │SCORING   │ │CHECK     │ │SPEED   │ │    │ │    │  │
│  │          │ │          │ │          │ │(skip if│ │(skip│ │    │  │
│  │          │ │          │ │          │ │inactive│ │if no│ │    │  │
│  │          │ │          │ │          │ │)       │ │DR) │ │    │  │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └───┬────┘ └─┬──┘ └─┬──┘  │
│       └────────────┴────────────┴───────────┴─────────┴──────┘     │
│                               │ MERGE                              │
│         total_blocking = Σ Vi.blocking                             │
│         total_warnings = Σ Vi.warnings                             │
│         ready = (blocking == 0)                                    │
│                                                                     │
│  HITL: present verdict → user confirms or fixes before Phase E     │
│  Blocking issues → do not proceed                                  │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ user confirmation
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  PHASE E — SAVE (optional)                                          │
│  account/<Account>/YYYY-MM-DD - Cluster Sizing Recommendation.md   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Agent inventory

### Phase B — Sizing agents

| Agent | Role | Inputs | Output | Spawn condition |
|---|---|---|---|---|
| **T-SHIRT SCORER** | Citi scoring, calibration, workload modifier, growth | `concurrency, volume_GB, SLA_s, workload_type, growth_pct` | `t_shirt, score, worker_range, workers_with_growth, memory_config, flags` | Always |
| **NETWORK CHECKER** | Bandwidth calculation → network workers | `volume_effective_GB, concurrency, SLA_s, inputs_confirmed, deployment` | `network_workers, throughput_Gbps, skip_reason, flags` | Always (returns null if inputs uncertain) |
| **WARP SPEED SIZER** | NVMe cache sizing → minimum cache workers | `hot_data_GB, NVMe_GB_per_worker, workload_profile, cache_hit_rate, volume_query_GB` | `cache_workers_min, cache_per_worker_GB, effective_volume_GB, node_type, flags` | Only if Warp Speed active AND workload ≠ ETL |

**Merge formula:**
```
workers_retained = max(3, agent1.workers_with_growth, agent2.network_workers, agent3.cache_workers_min)
```

### Phase D.5 — Verification agents

| Agent | Checks | Key blocking conditions | Skip condition |
|---|---|---|---|
| **V1 WORKLOAD COHERENCE** | Warp Speed + ETL conflict, coordinator sizing, memory config, mixed cluster | Warp Speed + ETL > 50%; coordinator < 32 vCPU for BI | Never |
| **V2 T-SHIRT SCORING** | Worker count vs range, growth applied, XL flag, DSR floor | workers_retained < 3; workers outside T-shirt range; XL without flag | Never |
| **V3 NETWORK CHECK** | N/A noted, dominant constraint applied, effective volume used | Network dominant but workers set to lower T-shirt value | Never |
| **V4 WARP SPEED** | max() correct, 0.85 margin, Elite flag, AE involvement | workers_retained ≠ max(t_shirt, cache_min) | If Warp Speed inactive |
| **V5 DR SIZING** | Mode specified, Active/Active impact stated, Warm formula | DR required but mode unspecified; AA without licensing impact | If DR not required |
| **V6 FLAGS** | All applicable flags present | — (warnings only) | Never |

---

## Data flow

```
Intake form answers
        │
        ▼
Translation table
        │ concurrency, volume_GB, SLA_s, workload_type,
        │ growth_pct, deployment, hot_data_GB, NVMe_GB,
        │ dr_mode, environments
        ▼
Phase B dispatch (3 parallel packets)
        │
        ├── T-SHIRT packet → {concurrency, volume, SLA, workload, growth}
        ├── NETWORK packet → {volume_effective, concurrency, SLA, confirmed}
        └── WARP SPEED packet → {hot_data, NVMe, hit_rate, profile, volume}
                │
                ▼
            JSON results
                │
                ▼
        Merge → workers_retained + all_flags
                │
                ▼
Phase C → coordinator spec, autoscaling, multi-env, DR
                │
                ▼
Phase D → formatted output
                │
                ▼
Phase D.5 dispatch (6 parallel slices of Phase D output)
                │
                ▼
        Merge → blocking count → HITL
                │
                ▼
Phase E → save to account/<Account>/
```

---

## Intake form — section mapping

| Section | Questions | Maps to |
|---|---|---|
| **1 — Users** | Peak concurrent users; user type | `Concurrency`, `Workload type` |
| **2 — Speed** | Query latency target; repetitive queries | `SLA` (T-shirt pts), Warp Speed candidate |
| **3 — Data** | Total volume; recent slice; growth | `Volume` (T-shirt pts), `Hot data`, `Growth %` |
| **4 — Infrastructure** | Cloud/on-prem; data sources; environments | `Deployment`, `[JDBC]/[MIXED]` flags, multi-env |
| **5 — Availability** | Downtime tolerance; regulatory obligations | `DR mode`, `[HA/DR]` flag |

Blank answers → defaults applied (concurrency: 10, SLA: M, volume: 5 GB, growth: 20%) + `[ASSUMPTIONS]` flag.

---

## T-shirt scoring grid (Citi method)

| Metric | S — 1 pt | M — 2 pts | L — 3 pts |
|---|---|---|---|
| Concurrency | ≤ 10 users | 11–50 users | > 50 users |
| Volume/query | ≤ 2 GB | 2–10 GB | > 10 GB |
| SLA | > 15s | 5–15s | < 5s |

| Score | T-shirt | Worker range |
|---|---|---|
| 3 | S | 3–5 |
| 4–6 | M | 5–9 |
| 7–8 | L | 9–15 |
| 9 | XL | > 15 (JMeter POC mandatory) |

Calibration within range: LOW/MID/HIGH sub-bands per metric → positional rule (all LOW = bottom, all HIGH = top).

---

## Workload modifier

| Workload | Bottleneck | Sizing effect |
|---|---|---|
| BI dashboards | Coordinator throughput | Top of T-shirt range; coordinator ≥ 32 vCPU mandatory |
| Ad-hoc | Worker memory | Top of range + 1 worker headroom; raise `query.max-memory` |
| ETL | Worker compute | Top of range; no Warp Speed |
| Mixed BI+ETL | Interference | ⚠️ Recommend separate clusters if ETL > 4h |

---

## Hard constraints (Starburst documentation)

| Constraint | Value | Source |
|---|---|---|
| Minimum production node | 16 vCPU / 64 GB RAM | Starburst technical doc |
| Minimum production workers | 3 | Starburst technical doc |
| Homogeneous cluster | All worker nodes identical spec | Starburst technical doc |
| Warp Speed homogeneity | All workers NVMe — no mixing | Starburst technical doc |
| Max workers per coordinator | 32 | Platform limit → DSR above |
| QPS limit default | 40 | SEP default — always raise before load test |

---

## Flags reference

| Flag | Trigger condition |
|---|---|
| `[ASSUMPTIONS]` | Any input not confirmed by customer |
| `[JDBC]` | Database connector present (throughput ~5 MB/s) |
| `[MIXED]` | Both lake and database sources |
| `[>32 WORKERS]` | Workers > 32 → DSR required |
| `[XL]` | T-shirt XL → JMeter POC mandatory |
| `[COORDINATOR]` | Coordinator metrics need JMX/Insights |
| `[HA/DR]` | DR excluded or not sized |
| `[WARP SPEED]` | Elite tier + all-NVMe workers required |
| `[ETL+WS]` | ETL workload + Warp Speed mentioned |
| `[NODE-SPEC]` | Node spec below 32 vCPU / 128 GB recommended |

---

## Files

| File | Purpose |
|---|---|
| `.claude/commands/starburst-sizing.md` | Skill definition — full methodology and agent prompts |
| `research/starburst-cluster-sizing.md` | Reference doc — GTM Playbook methodology |
| `research/sizing/questionnaire-client-fr.md` | Customer intake form — French version |
| `research/sizing/questionnaire-client-en.md` | Customer intake form — English version |
| `research/sizing/starburst-sizing-skill-architecture.md` | This document |
