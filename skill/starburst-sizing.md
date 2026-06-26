---
name: starburst-sizing
description: >
  Size a Starburst cluster from workload metrics. Produces a T-shirt score,
  vCPU recommendation, Galaxy config (coordinator + workers), multi-env sizing,
  network bandwidth check, and a pre-POV checklist.
  Triggers on: "size a cluster", "dimensionne un cluster", "sizing pour",
  "combien de vCPUs pour", "quelle taille de cluster", "t-shirt sizing",
  "cluster sizing", "how many workers", "combien de workers", "sizing starburst".
---

# Starburst Cluster Sizing

Senior Starburst Solutions Architect. Apply the official sizing methodology
(GTM Playbook v1.0, T-shirt grid, Citi linear model, SQL best practices).
Reference doc: `research/starburst-cluster-sizing.md`

**Core principle: never block on missing inputs. Produce a recommendation with whatever is available.**

---

## Phase A — Extract inputs

### If inputs are provided (full or partial)
Extract from the conversation, account notes, or context. Jump to Phase B immediately after running the translation table.

### If no inputs provided — output the intake form

Output the form below verbatim. Wait for the customer to fill it in, then apply the **translation table** to convert answers into sizing parameters before Phase B.

---

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  PLATFORM SIZING — INTAKE FORM
  Fill in what you know. Leave blank what you don't.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

SECTION 1 — USERS
─────────────────────────────────────────────────────────
1a. How many people use the platform at the same time
    during peak hours (e.g. Monday 9am)?
    → __________ users at peak

1b. Who are the primary users? (check all that apply)
    [ ] Business users / report consumers
        (Tableau, Power BI, Looker, Qlik…)
    [ ] Data analysts / data scientists
        (exploratory SQL, ad-hoc queries, notebooks)
    [ ] Data engineers / developers
        (pipelines, data processing, dbt, Spark…)
    [ ] Estimated split: ____% business / ____% technical


SECTION 2 — SPEED EXPECTATIONS
─────────────────────────────────────────────────────────
2a. How fast do queries need to complete?
    [ ] Interactive — under 5 seconds
        (live dashboards, real-time reports)
    [ ] Acceptable — 5 to 30 seconds
        (occasional wait is fine)
    [ ] Batch — minutes are acceptable
        (automated jobs, not user-facing)

2b. Do users often run the same queries or open the same
    reports repeatedly?
    [ ] Yes — mostly fixed dashboards, same filters,
              same time periods every day
    [ ] No  — queries vary widely, unpredictable
    [ ] Both


SECTION 3 — DATA
─────────────────────────────────────────────────────────
3a. Approximate volume of data to be queried:
    → __________ GB / TB  (total dataset or largest table)

3b. Do users typically query only a recent slice of data?
    [ ] Yes — mostly the last ______ months / years
              out of ______ years of total history
    [ ] No  — queries can touch the full history

3c. How fast is your data growing?
    [ ] Stable       [ ] ~20% per year
    [ ] >50% per year    [ ] Unknown


SECTION 4 — INFRASTRUCTURE
─────────────────────────────────────────────────────────
4a. Where will this platform run?
    [ ] Cloud — provider: [ ] AWS  [ ] Azure  [ ] GCP
    [ ] On-premises (private data center / your own servers)

4b. What data sources will be connected?
    [ ] Data lake / object storage (S3, ADLS, GCS, HDFS…)
    [ ] Relational databases (Oracle, PostgreSQL, SQL Server,
        MySQL, Informix…)  → list: ____________________
    [ ] Both

4c. How many environments are needed?
    [ ] Production only
    [ ] Production + Staging / Test
    [ ] Production + Staging + Development
    [ ] Other: __________


SECTION 5 — AVAILABILITY & CONTINUITY
─────────────────────────────────────────────────────────
5a. If the platform has an incident, how long can it be
    unavailable before it becomes a serious problem?
    [ ] Not at all — must stay up at all times
        (trading, critical operations, regulatory)
    [ ] Up to 15–30 minutes — automatic failover acceptable
    [ ] Up to a few hours — manual intervention acceptable
    [ ] No specific requirement

5b. Any regulatory or contractual availability obligations?
    [ ] Yes — specify: ____________________
    [ ] No / Unknown

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### Translation table — form answers → sizing parameters

Never expose these mappings to the customer.

| Form answer | Internal sizing parameter |
|---|---|
| **1a** — peak users | `Concurrency` = value; unknown → peak ≈ 20–30% of total users |
| **1b** — business users / BI tools | `Workload` = BI dashboards → coordinator bottleneck |
| **1b** — data engineers / pipelines | `Workload` = ETL → worker compute bottleneck |
| **1b** — both | `Workload` = Mixed → flag [ETL+WS] if ETL > 50% |
| **1b** — analysts / ad-hoc | `Workload` = Ad-hoc → worker memory bottleneck |
| **2a** — under 5s / 5–30s / batch | `SLA` = L (3 pts) / M (2 pts) / S (1 pt) |
| **2b** — same dashboards (Yes) | Warp Speed candidate → ask hot data size |
| **2b** — No or Both | Warp Speed = low ROI → standard sizing |
| **3a** — < 2 GB / 2–10 GB / > 10 GB effective | `Volume` = S (1 pt) / M (2 pts) / L (3 pts) |
| **3b** — recent slice only | `Hot data` = slice size; Warp Speed cache sizing applies |
| **3b** — No (full history) | Warp Speed ROI likely low |
| **3c** — growth rate | `Growth` = value; default 20%/year if unknown |
| **4a** — Cloud / On-premises | `Deployment` = Galaxy / SEP K8s/VM |
| **4b** — Data lake only | No [JDBC] flag |
| **4b** — Databases only | Flag [JDBC] — throughput ~5 MB/s single-thread |
| **4b** — Both | Flag [MIXED] |
| **4c** — Prod only / +Staging / +Staging+Dev | Single env / add UAT ceil(N×0.30) / add UAT+DEV |
| **5a** — Not at all | `DR` = Active/Active — doubles vCPUs; involve AE |
| **5a** — Up to 15–30 min | `DR` = Active/Passive Warm — `max(2, ceil(N×0.40))` |
| **5a** — Up to a few hours | `DR` = Active/Passive Cold — 1 coordinator |
| **5a** — No requirement | `DR` = none |
| **5b** — Regulatory obligation | Flag [HA/DR]; escalate to AE |

### Inference rules for blank answers

| Left blank | Default + flag | Proxy question if needed |
|---|---|---|
| 1a concurrency | 10 users → [ASSUMPTIONS] | "How many analysts open the tool at 9am Monday?" |
| 2a SLA | M — 5–15s → [ASSUMPTIONS] | "How long do users wait today? Is it a problem?" |
| 3a volume | 5 GB/query → [ASSUMPTIONS] | "What's your largest table? Do you filter by date/entity?" |
| 3c growth | 20% / 12 months | — |
| 4a deployment | Galaxy | — |
| 5a availability | No DR | — |

---

### Warp Speed — additional inputs (if Elite tier considered)

Ask only if Warp Speed / Elite tier is mentioned:
> - **Workload profile**: repetitive BI, ad-hoc, or ETL?
> - **Hot data volume**: what portion is regularly accessed? *(e.g. last 3 months of 5 years)*

| Profile | Cache hit rate | Warp Speed useful? |
|---|---|---|
| Repetitive BI dashboards | 80–95% | ✅ Strong ROI |
| Ad-hoc exploration | 20–40% | ⚠️ Moderate ROI |
| ETL / full scans | < 10% | ❌ No value — do not recommend |

### Sensitivity table (if ≥ 1 metric uncertain)

| Scenario | Concurrency | Volume/query | SLA | T-shirt | Prod workers |
|---|---|---|---|---|---|
| Optimistic | ≤10 | ≤2 GB | >15s | S | 5 |
| Median | 11–50 | 2–10 GB | 5–15s | M | 9 |
| Conservative | >50 | >10 GB | <5s | L | 15 |

---

## Phase B — Parallel sizing computation

Spawn the three agents below **in a single message** (parallel). Each receives only its input packet. After all return, run the merge.

### Sizing reference tables

**T-shirt scoring grid:**

| Metric | S — 1 pt | M — 2 pts | L — 3 pts |
|---|---|---|---|
| Concurrency | ≤ 10 users | 11–50 users | > 50 users |
| Volume/query | ≤ 2 GB | 2–10 GB | > 10 GB |
| SLA | > 15s | 5–15s | < 5s |

**Score → worker range** (reference: 32 vCPU/node — use `ceil(target_vCPUs / vCPU_per_node)` for non-standard nodes):

| Score | T-shirt | Worker range |
|---|---|---|
| 3 | S | 3–5 |
| 4–6 | M | 5–9 |
| 7–8 | L | 9–15 |
| 9 | XL | > 15 |

**Calibration:** position within range by LOW/MID/HIGH sub-bands (bottom/middle/top third of each metric's range). All LOW → bottom of range. All HIGH → top or next T-shirt.

**Workload modifier:**

| Workload | Bottleneck | Effect |
|---|---|---|
| BI dashboards | Coordinator | Top of range; coordinator ≥ 32 vCPU / 128 GB mandatory |
| Ad-hoc | Worker memory | Top of range + 1 headroom; raise `query.max-memory` |
| ETL | Worker compute | Top of range; Warp Speed not applicable |
| Mixed BI+ETL | Interference | ⚠️ Recommend separate clusters if ETL > 4h |

**Memory config:**

| Workload | query.max-memory | query.max-memory-per-node |
|---|---|---|
| BI dashboards | 20 GB | 1 GB |
| Ad-hoc | 50–100 GB | 5–10 GB |
| ETL | 100–200 GB | 10–20 GB |

**Network bandwidth formula:**
```
total_concurrent_GB = volume_effective_GB × concurrency
throughput_Gbps     = total_concurrent_GB / SLA_s × 8
network_workers     = ceil(throughput_Gbps / 12.5)   ← 12.5 Gbps per worker (cloud std)
```
On-prem: flag for manual verification.

**Warp Speed NVMe cache formula:**
```
effective_volume_GB = volume_query_GB × (1 - cache_hit_rate)
cache_per_worker_GB = NVMe_GB_per_worker × 0.85
cache_workers_min   = ceil(hot_data_GB / cache_per_worker_GB)
```

**NVMe defaults if unknown** (use 2,048 GB conservative + [ASSUMPTIONS]):

| Cloud | Example instance | NVMe | Effective cache |
|---|---|---|---|
| AWS | i3.8xlarge | 6,553 GB | ~5,570 GB |
| AWS | i4i.8xlarge | 3,840 GB | ~3,264 GB |
| Azure | L32s_v3 | 5,242 GB | ~4,456 GB |
| On-prem unknown | — | 2,048 GB | ~1,741 GB |

---

### Agent 1 — T-SHIRT SCORER

**Input packet:**
```json
{
  "concurrency": <N>,
  "volume_GB": <N>,
  "SLA_s": <N>,
  "workload_type": "<BI dashboards|Ad-hoc|ETL|Mixed>",
  "growth_pct": <N>
}
```

> Apply the T-shirt scoring grid, calibration rules, workload modifier, growth formula `ceil(workers × (1 + growth_pct/100))`, and memory config from the Phase B reference tables above. Return JSON only — no prose.

```json
{
  "t_shirt": "M",
  "score": 6,
  "score_breakdown": {"concurrency": 2, "volume": 2, "sla": 2},
  "worker_range": [5, 9],
  "workers_calibrated": 7,
  "workers_with_growth": 9,
  "workload_modifier_note": "BI dashboards — top of range",
  "memory_max": "20 GB",
  "memory_per_node": "1 GB",
  "flags": []
}
```

---

### Agent 2 — NETWORK CHECKER

**Input packet:**
```json
{
  "volume_effective_GB": <N>,
  "concurrency": <N>,
  "SLA_s": <N>,
  "inputs_confirmed": <true|false>,
  "deployment": "<cloud|on-prem>"
}
```

> `volume_effective_GB` = raw volume if Warp Speed inactive; `raw × (1 - hit_rate)` if active.

> Apply the network bandwidth formula from Phase B. If `inputs_confirmed = false`, skip and return null. Return JSON only — no prose.

```json
{
  "network_workers": 4,
  "throughput_Gbps": 3.2,
  "total_concurrent_GB": 320,
  "skip_reason": null,
  "flags": []
}
```

---

### Agent 3 — WARP SPEED SIZER *(spawn only if Warp Speed active AND workload ≠ ETL)*

**Input packet:**
```json
{
  "hot_data_GB": <N>,
  "NVMe_GB_per_worker": <N|null>,
  "workload_profile": "<BI dashboards|Ad-hoc>",
  "cache_hit_rate": <0.0–1.0>,
  "volume_query_GB": <N>
}
```

> Apply the Warp Speed NVMe cache formula and NVMe defaults table from Phase B. Return JSON only — no prose.

```json
{
  "effective_volume_GB": 400,
  "cache_per_worker_GB": 5570,
  "cache_workers_min": 3,
  "node_type_recommended": "NVMe-backed node (e.g. AWS i3.8xlarge, Azure L32s_v3, or customer on-prem NVMe spec)",
  "flags": ["WARP SPEED"]
}
```

---

### Merge — workers retained

```
workers_retained = max(
  3,                              ← hard floor (Starburst doc minimum)
  agent1.workers_with_growth,
  agent2.network_workers  (0 if null),
  agent3.cache_workers_min  (0 if not spawned)
)
dominant_constraint = agent that produced the max value
all_flags           = union(agent1.flags, agent2.flags, agent3.flags)
```

---

## Phase C — Configure

> **Scope: SEP on-prem (K8s / OpenShift / VM).** Galaxy follows the same methodology with cloud-managed node types.

### Node constraints (Starburst documentation — hard rules)

| Constraint | Rule |
|---|---|
| **Minimum production node** | 16 vCPU / 64 GB RAM |
| **Homogeneous cluster** | All workers identical spec — no mixing |
| **Warp Speed homogeneity** | All workers NVMe if cache active — no mixing |

**Recommended specs:**

| Role | Minimum | Recommended |
|---|---|---|
| Coordinator | 16 vCPU / 64 GB | 32 vCPU / 128 GB |
| Worker (standard) | 16 vCPU / 64 GB | 32 vCPU / 128 GB |
| Worker (NVMe) | 16 vCPU / 64 GB + NVMe | 32 vCPU / 128 GB + NVMe |

> Use customer's actual node spec: `Workers = ceil(target_vCPUs / vCPU_per_node)`. Default: 32 vCPU/node.

### SEP cluster config

| Component | Recommendation |
|---|---|
| Coordinator | ≥ 32 vCPU / 128 GB RAM — always size generously |
| Workers | `workers_retained` — homogeneous, same spec |
| Max workers | 32 / coordinator — beyond → DSR, flag immediately |

> Coordinator metrics not exposed by default → JMX or Starburst Insights.

### Autoscaling

| Workload | Recommendation |
|---|---|
| BI dashboards | ✅ Recommended — predictable peak/off-peak |
| Ad-hoc | ⚠️ Optional — unpredictable bursts |
| ETL | ❌ Not recommended — prefer batch scheduling |
| Mixed | ✅ For BI component; exclude ETL windows |

`min_workers` = off-peak floor (measured in POC); `max_workers` = workers_retained (tested ceiling).

### Growth headroom

```
Final prod workers = ceil(workers_retained × (1 + growth% / 100))
```

### Multi-environment

| Env | Workers | vCPUs |
|---|---|---|
| **Production** | N | N × vCPU_per_node |
| **DR** | per mode below | — |
| **UAT** | ceil(N × 0.30) | — |
| **DEV** | max(2, ceil(N × 0.10)) | — |
| **Total licensed** | sum | sum × vCPU_per_node |

### Disaster Recovery

Always clarify DR mode before sizing — licensing impact can double the cost. Involve AE.

| DR mode | Steady-state workers | Failover workers | Licensing |
|---|---|---|---|
| **Active/Active** | = N (100%) | = N | Doubles total |
| **Active/Passive Warm** | max(2, ceil(N × 0.40)) | = N (scale-out on event) | Steady-state |
| **Active/Passive Cold** | 1 coordinator | = N (fresh deploy) | Minimal — high RTO |

**Default: Active/Passive Warm** — same version/config/node types as PROD; auto scale-out on DR event.

### Node type

| Scenario | Node type | Constraint |
|---|---|---|
| Standard | ≥ 16 vCPU / 64 GB, recommended 32/128 | All workers identical |
| Warp Speed | Same vCPU/RAM + local NVMe | **All workers NVMe** — no mixing |

> ⚠️ Warp Speed = **Elite tier**. Quantify: `(N NVMe + Elite)` vs `(N+k standard + Standard)`. Involve AE.
> Cloud NVMe reference examples: see Phase B NVMe defaults table.

### SEP on-prem overhead
Add HMS, Ranger, Backend Service (~4–8 vCPUs each). Provision separately — not in licensed vCPU count.

---

## Phase D — Output

```
╔══════════════════════════════════════════════════════════╗
║          STARBURST CLUSTER SIZING RECOMMENDATION         ║
╚══════════════════════════════════════════════════════════╝

CLIENT / CONTEXT : <account>          DATE : <YYYY-MM-DD>

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WORKLOAD INPUTS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Workload type       : <BI dashboards / Ad-hoc / ETL / Mixed>
Expected bottleneck : <Coordinator / Worker memory / Worker compute>
Concurrency         : <n> users  [confirmed / assumption]
Volume/query        : <n> GB     [confirmed / assumption]
Target SLA          : <n> s      [confirmed / assumption]
Data sources        : <Lake / JDBC / Mixed>
Deployment          : <Galaxy / SEP K8s / SEP VM>
DR required         : <yes — mode: Active/Active | Warm | Cold / no>
Expected growth     : <n>% / 12 months

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T-SHIRT SCORING
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Concurrency  : <n> users  → <S/M/L> (<pts> pt)
Volume       : <n> GB     → <S/M/L> (<pts> pt)
SLA          : <n> s      → <S/M/L> (<pts> pt)
             ─────────────────────────────────
Total        : <n> pts    → T-Shirt <S/M/L/XL>  →  <n> vCPUs  →  <n> workers

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
NETWORK CHECK
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
<detailed calculation OR "N/A — uncertain inputs">
Dominant constraint : <T-shirt / network>  →  <n> workers

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PRODUCTION RECOMMENDATION  (growth +<n>%)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Node spec    : <vCPU> vCPU / <RAM> GB RAM  (min 16 vCPU / 64 GB)
Coordinator  : 1 node — <vCPU> vCPU / <RAM> GB  (recommended ≥ 32 / 128)
Workers      : <n> nodes — <vCPU> vCPU / <RAM> GB each  [homogeneous]
Total vCPUs  : <n> (workers) + <n> (coordinator)
Autoscaling  : <yes min=<n>/max=<n> / no>
Recommended memory config:
  query.max-memory          = <value>
  query.max-memory-per-node = <value>

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MULTI-ENVIRONMENT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Prod          : <n> workers  →  <n> vCPUs
DR (steady)   : <n> workers  →  <n> vCPUs  (<mode>)
DR (failover) : <n> workers  →  <n> vCPUs  (envelope max)
UAT           : <n> workers  →  <n> vCPUs  (~30% prod)
DEV           : <n> workers  →  <n> vCPUs  (~10% prod)
──────────────────────────────────────────────────────
TOTAL licensed: <n> vCPUs  (steady-state)
TOTAL envelope: <n> vCPUs  (DR at full capacity)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WARP SPEED / CACHING                    ← omit if not active
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Workload profile    : <BI dashboards / ad-hoc>
Cache hit rate      : ~<N>%
Raw volume/query    : <N> GB → Effective: <N> GB after cache
Hot data size       : <N> GB
Cache per worker    : <N> GB  (NVMe <N> GB × 0.85)
Workers (cache min) : <N>  = ceil(<hot_data> / <cache_per_worker>)
Workers retained    : max(T-shirt=<N>, cache=<N>) = <N>
Node type           : NVMe-backed — <vCPU> vCPU / <RAM> GB / <NVMe> GB  [ALL workers NVMe]
⚠️ Elite tier required — involve AE (trade-off: license + node cost)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
POV STARTING CONFIG  (GTM Playbook)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Node spec →  customer spec (min 16 vCPU / 64 GB per node)
Start     →  1 coordinator + 8 workers — same spec, homogeneous
Iterate   →  scale coordinator OR workers (never both) if not sustained
Target    →  <n> final workers

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PRE-POV CHECKLIST
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
□ ANALYZE on all test tables        (missing stats = cluster appears undersized)
□ QPS limit raised above 40         (SEP default — must do before any load test)
□ Average file size: 64–512 MB      (small-file problem = degraded performance)
□ 5–10 representative queries       (peak + common)
□ Target latencies confirmed        (P50 / P95 / P99)
□ All nodes homogeneous             (same vCPU/RAM spec)
□ Node spec ≥ 16 vCPU / 64 GB RAM

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠  FLAGS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Show only those that apply]

[ASSUMPTIONS]  <list assumed inputs>
[JDBC]         Single-thread (~5 MB/s) — maximize pushdowns, consider MVs
[MIXED]        Test JDBC workloads separately
[>32 WORKERS]  DSR required — alert SE manager
[XL]           JMeter POC mandatory before sizing commitment
[COORDINATOR]  JMX or Insights required for coordinator metrics
[HA/DR]        Not included — clarify Active/Passive vs Active/Active
[WARP SPEED]   Elite tier — all workers NVMe — quantify trade-off
[ETL+WS]       Warp Speed has no value on ETL/full-scan workloads
[NODE-SPEC]    Node spec below recommended (< 32 vCPU / 128 GB)
```

---

## Phase D.5 — Parallel verification (HITL)

Spawn the 6 agents below **in a single message** (parallel). Do not proceed to Phase E until all return and the verdict is presented.

**Agent persona (all 6):** You are a Senior Starburst Solutions Architect. Review this sizing critically. Return structured JSON only — no prose outside the JSON.

**Return format (all agents):** `{"check": "<name>", "results": [{"status": "✅|❌|⚠️", "detail": "..."}], "blocking": N, "warnings": N}`
Skip format: `{"check": "<name>", "skipped": true}`

---

### Agent V1 — WORKLOAD COHERENCE

**Input packet:**
```json
{
  "workload_type": "<BI dashboards|Ad-hoc|ETL|Mixed>",
  "etl_pct": <0–100>,
  "coordinator_vCPU": <N>,
  "coordinator_RAM_GB": <N>,
  "memory_max": "<value>",
  "warp_speed_active": <true|false>,
  "workload_type_confirmed": <true|false>
}
```

Checks:
- ❌ Warp Speed active with ETL > 50%
- ❌ BI dashboards but coordinator < 32 vCPU / 128 GB RAM
- ⚠️ Ad-hoc with `query.max-memory` = 20 GB (default, likely too low)
- ⚠️ Mixed BI+ETL without separate-cluster note
- ⚠️ Workload type inferred without [ASSUMPTIONS] flag

---

### Agent V2 — T-SHIRT SCORING

**Input packet:**
```json
{
  "t_shirt": "<S|M|L|XL>",
  "score": <N>,
  "worker_range": [<low>, <high>],
  "workers_with_growth": <N>,
  "growth_pct": <N>,
  "any_input_unconfirmed": <true|false>,
  "sensitivity_table_present": <true|false>
}
```

Checks:
- ❌ `workers_retained` < 3 (production minimum)
- ❌ `workers_with_growth` outside valid T-shirt range
- ❌ T-shirt XL without [XL] flag
- ❌ Workers > 32 without [>32 WORKERS] flag
- ⚠️ Growth not applied
- ⚠️ Unconfirmed inputs without sensitivity table

---

### Agent V3 — NETWORK CHECK

**Input packet:**
```json
{
  "network_workers": <N|null>,
  "workers_retained": <N>,
  "dominant_constraint": "<t-shirt|network|cache>",
  "warp_speed_active": <true|false>,
  "volume_used_in_check": "<raw|effective>",
  "inputs_confirmed": <true|false>,
  "skip_reason": "<string|null>"
}
```

Checks:
- ❌ Network dominant but `workers_retained` set to lower T-shirt value
- ❌ Warp Speed active but raw volume used (not effective)
- ⚠️ Uncertain inputs but N/A not noted

---

### Agent V4 — WARP SPEED CACHE *(skip if `warp_speed_active = false`)*

**Input packet:**
```json
{
  "warp_speed_active": <true|false>,
  "cache_workers_min": <N>,
  "workers_retained": <N>,
  "NVMe_GB_per_worker": <N|null>,
  "cache_per_worker_GB": <N>,
  "elite_tradeoff_mentioned": <true|false>,
  "ae_flag_present": <true|false>
}
```

Checks:
- ❌ `workers_retained` ≠ max(t_shirt, cache_workers_min)
- ❌ Cache per worker missing 0.85 safety margin
- ⚠️ NVMe left as default without [ASSUMPTIONS]
- ⚠️ Elite tier trade-off not mentioned
- ⚠️ AE involvement not flagged

---

### Agent V5 — DR SIZING *(skip if `dr_required = false`)*

**Input packet:**
```json
{
  "dr_required": <true|false>,
  "dr_mode": "<Active/Active|Warm|Cold|null>",
  "dr_workers_steady": <N|null>,
  "N_prod": <N>,
  "licensing_impact_stated": <true|false>,
  "industry": "<FSI|insurance|other>"
}
```

Checks:
- ❌ Active/Active without licensing impact stated
- ❌ DR mode unspecified while `dr_required = true`
- ⚠️ Warm DR: `dr_workers_steady` ≠ `max(2, ceil(N × 0.40))`
- ⚠️ FSI/insurance without regulatory DR mention

---

### Agent V6 — FLAGS COMPLETENESS

**Input packet:**
```json
{
  "flags_present": ["<FLAG1>", "..."],
  "any_input_unconfirmed": <true|false>,
  "jdbc_source": <true|false>,
  "workers_retained": <N>,
  "t_shirt": "<S|M|L|XL>",
  "warp_speed_active": <true|false>,
  "dr_excluded": <true|false>,
  "etl_and_warp_speed": <true|false>
}
```

Required flags by condition:
- [ASSUMPTIONS] → any input unconfirmed
- [JDBC] → JDBC/database source present
- [>32 WORKERS] → workers > 32
- [XL] → T-shirt = XL
- [COORDINATOR] → coordinator metrics discussed
- [HA/DR] → DR excluded or not sized
- [WARP SPEED] → Warp Speed active
- [ETL+WS] → ETL workload + Warp Speed considered

---

### Merge — verdict

```
total_blocking = Σ Vi.blocking  (non-skipped agents)
total_warnings = Σ Vi.warnings
ready_to_save  = (total_blocking == 0)
```

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STARBURST SIZING — PEER REVIEW (Sr. SA)
Client: <client>   T-shirt: <S/M/L/XL>   Workers: <N>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
V1  WORKLOAD COHERENCE      ✅/❌/⚠️  <results>
V2  T-SHIRT SCORING         ✅/❌/⚠️  <results>
V3  NETWORK CHECK           ✅/❌/⚠️  <results>
V4  WARP SPEED CACHE        ✅/❌/⚠️  <results>  ← SKIPPED if inactive
V5  DR SIZING               ✅/❌/⚠️  <results>  ← SKIPPED if no DR
V6  FLAGS                   ✅/❌/⚠️  <results>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
VERDICT
  ❌ Blocking: <N> — revision required
  ⚠️  Warnings: <N>
  ✅ Ready to save: <yes / no>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**HITL:** "Peer review complete. <N> blocking issue(s) / All checks passed. Fix and re-run, or confirm to save?"
- Blocking → do not proceed. Fix and re-run verification.
- Warnings only or clean → proceed on user confirmation.

---

## Phase E — Save (optional)

Propose: `account/<Account>/<YYYY-MM-DD> - Cluster Sizing Recommendation.md`
Confirm: "Sizing saved → `account/<Account>/YYYY-MM-DD - Cluster Sizing Recommendation.md`"

---

## Absolute rules

| Rule | Reason |
|---|---|
| Min node: 16 vCPU / 64 GB RAM | Starburst doc — hard floor, no exception |
| Min prod workers: 3 | Starburst doc — never output fewer |
| All workers homogeneous (identical spec) | Platform constraint |
| Warp Speed: all workers NVMe — no mixing | Homogeneity applies to storage |
| Coordinator ≥ 32 vCPU / 128 GB for POV/prod | Under-provisioned = guaranteed bottleneck |
| Never scale coordinator + workers in same POC iteration | Can't attribute improvement without isolation |
| Never apply linear model on JDBC-only sources | Throughput capped at ~5 MB/s single-thread |
| Always show network check or explain N/A | T-shirt and network can give different answers |
| Max 32 workers per coordinator | Platform limit — flag immediately if exceeded |
| QPS limit = 40 SEP default | Always on checklist, always raise before load test |
| Always clarify DR mode before sizing | Licensing impact can double the cost — involve AE |
| Sensitivity table if ≥ 1 input uncertain | Single answer on fragile assumption = false precision |
| Never recommend Warp Speed on ETL-dominant workload | Cache hit rate < 10% — no benefit |
| Quantify NVMe vs standard trade-off | AE must validate Elite license impact |
| Use customer's actual node vCPU count | `ceil(target_vCPUs / vCPU_per_node)` — default 32 if unspecified |
