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

Output the form below verbatim. Wait for the customer to fill it in, then use the **translation table** to convert answers into sizing parameters before proceeding to Phase B.

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

Apply after receiving the filled form. Each answer maps to one or more internal parameters used in Phase B onwards. Never expose these mappings to the customer.

| Form answer | Internal sizing parameter |
|---|---|
| **1a** — peak users count | `Concurrency` = answered value. If unknown: peak ≈ 20–30% of total users |
| **1b** — business users / BI tools (Tableau, Power BI…) | `Workload` = BI dashboards → coordinator bottleneck |
| **1b** — data engineers / pipelines / dbt | `Workload` = ETL → worker compute bottleneck |
| **1b** — both business + technical | `Workload` = Mixed → flag [ETL+WS] if ETL > 50% |
| **1b** — data analysts / ad-hoc | `Workload` = Ad-hoc → worker memory bottleneck |
| **2a** — under 5 seconds | `SLA` = L → 3 pts T-shirt |
| **2a** — 5 to 30 seconds | `SLA` = M → 2 pts T-shirt |
| **2a** — batch / minutes | `SLA` = S → 1 pt T-shirt |
| **2b** — same dashboards / same filters (Yes) | Warp Speed candidate → ask hot data size (3b) |
| **2b** — No or Both | Warp Speed = low ROI → standard sizing unless 1c = BI |
| **3a** — data volume | `Volume/query` ≈ total × 1–5% if filtered by date/entity; full scan if no filter |
| **3a** — < 2 GB effective | `Volume` = S → 1 pt T-shirt |
| **3a** — 2–10 GB effective | `Volume` = M → 2 pts T-shirt |
| **3a** — > 10 GB effective | `Volume` = L → 3 pts T-shirt |
| **3b** — recent slice only (Yes) | `Hot data` = slice size; Warp Speed cache sizing applies |
| **3b** — No (full history) | `Hot data` = full dataset; Warp Speed ROI likely low |
| **3c** — growth rate | `Growth` = answered value; default 20%/year if unknown |
| **4a** — Cloud | `Deployment` = Galaxy (default) |
| **4a** — On-premises | `Deployment` = SEP K8s/VM |
| **4b** — Data lake only | Standard sizing; no [JDBC] flag |
| **4b** — Databases only | Flag [JDBC] — linear model not applicable; throughput ~5 MB/s |
| **4b** — Both | Flag [MIXED] |
| **4c** — Production only | Single env sizing |
| **4c** — Prod + Staging | Add UAT = ceil(N × 0.30) |
| **4c** — Prod + Staging + Dev | Add UAT + DEV |
| **5a** — Not at all / always-on | `DR` = Active/Active — doubles licensed vCPUs; involve AE |
| **5a** — Up to 15–30 minutes | `DR` = Active/Passive Warm — `max(2, ceil(N × 0.40))` steady-state |
| **5a** — Up to a few hours | `DR` = Active/Passive Cold — 1 coordinator steady-state |
| **5a** — No specific requirement | `DR` = none |
| **5b** — Regulatory obligation | Flag [HA/DR]; escalate to AE before committing DR mode |

### Inference rules for blank answers

| Left blank | Default applied | Flag |
|---|---|---|
| 1a (concurrency) | 10 users | [ASSUMPTIONS] |
| 2a (SLA) | M — 5–15s | [ASSUMPTIONS] |
| 3a (volume) | 5 GB/query | [ASSUMPTIONS] |
| 3c (growth) | 20% / 12 months | — |
| 4a (deployment) | Galaxy (cloud) | — |
| 5a (availability) | No DR | — |

---

### Warp Speed — additional inputs (if Elite tier considered)

Ask these two questions only if Warp Speed / Elite tier is mentioned or likely:

> - **Workload profile**: repetitive BI dashboards, ad-hoc exploration, or ETL/batch?
> - **Hot data volume**: what portion of the data is regularly accessed? *(e.g. last 3 months of a 5-year history)*

**Estimated cache hit rate by profile:**

| Profile | Estimated cache hit rate | Warp Speed useful? |
|---|---|---|
| Repetitive BI dashboards | 80–95% | ✅ Strong ROI |
| Ad-hoc exploration | 20–40% | ⚠️ Moderate ROI |
| ETL / full table scans | < 10% | ❌ No added value |

> If ETL-dominant profile → do not recommend Warp Speed. Standard sizing.

### Sensitivity table (if ≥ 1 metric remains uncertain after proxies)

Do not give a single answer on a fragile assumption. Show the range:

| Scenario | Concurrency | Volume/query | SLA | T-shirt | Prod workers |
|---|---|---|---|---|---|
| Optimistic | ≤10 | ≤2 GB | >15s | S | 5 |
| Median | 11–50 | 2–10 GB | 5–15s | M | 9 |
| Conservative | >50 | >10 GB | <5s | L | 15 |

Adapt the table to known metrics (fix confirmed values, vary the unknowns).

---

## Phase B — Parallel sizing computation

Spawn the three agents below **in a single message** (parallel). Each agent receives only the input packet listed — no other context. After all return, run the merge step.

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

**Agent task:**

> You are a Starburst cluster sizing specialist. Apply the Citi T-shirt method to the input packet. Return JSON only — no prose.
>
> **Scoring grid:**
>
> | Metric | S — 1 pt | M — 2 pts | L — 3 pts |
> |---|---|---|---|
> | Concurrency | ≤ 10 users | 11–50 users | > 50 users |
> | Volume/query | ≤ 2 GB | 2–10 GB | > 10 GB |
> | SLA | > 15s | 5–15s | < 5s |
>
> **T-shirt → worker range (reference: 32 vCPU per node — adjust with `ceil(target_vCPUs / vCPU_per_node)` if customer node spec differs):**
>
> | Score | T-shirt | Worker range |
> |---|---|---|
> | 3 | S | 3–5 |
> | 4–6 | M | 5–9 |
> | 7–8 | L | 9–15 |
> | 9 | XL | > 15 |
>
> **Calibration:** position within range based on LOW/MID/HIGH sub-bands (bottom/middle/top third of each metric's range). All LOW → bottom of range. All HIGH → top of range.
>
> **Workload modifier:**
>
> | Workload | Effect |
> |---|---|
> | BI dashboards | Top of range; coordinator ≥ 32 vCPU / 128 GB mandatory |
> | Ad-hoc | Top of range + 1 worker headroom; prioritize RAM |
> | ETL | Top of range |
> | Mixed BI+ETL | ⚠️ Note: recommend separate clusters if ETL jobs > 4h |
>
> **Growth:** `workers_with_growth = ceil(workers_calibrated × (1 + growth_pct / 100))`
>
> **Memory config:**
>
> | Workload | query.max-memory | query.max-memory-per-node |
> |---|---|---|
> | BI dashboards | 20 GB (default) | 1 GB (default) |
> | Ad-hoc | 50–100 GB | 5–10 GB |
> | ETL | 100–200 GB | 10–20 GB |
>
> **Return:**
> ```json
> {
>   "t_shirt": "M",
>   "score": 6,
>   "score_breakdown": {"concurrency": 2, "volume": 2, "sla": 2},
>   "worker_range": [5, 9],
>   "workers_calibrated": 7,
>   "workers_with_growth": 9,
>   "workload_modifier_note": "BI dashboards — top of range",
>   "memory_max": "20 GB",
>   "memory_per_node": "1 GB",
>   "flags": []
> }
> ```

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

**Agent task:**

> You are a network bandwidth sizing specialist for distributed SQL engines. Calculate the minimum worker count to sustain the target query throughput. Return JSON only.
>
> **Formula:**
> ```
> total_concurrent_GB = volume_effective_GB × concurrency
> throughput_GB_s     = total_concurrent_GB / SLA_s
> throughput_Gbps     = throughput_GB_s × 8
> network_workers     = ceil(throughput_Gbps / 12.5)
> ```
> 12.5 Gbps = standard per-worker bandwidth (EC2/cloud). On-prem: flag for manual verification.
>
> If `inputs_confirmed = false`: skip computation, set `network_workers = null`.
>
> **Return:**
> ```json
> {
>   "network_workers": 4,
>   "throughput_Gbps": 3.2,
>   "total_concurrent_GB": 320,
>   "skip_reason": null,
>   "flags": []
> }
> ```
> or if skipped:
> ```json
> {
>   "network_workers": null,
>   "skip_reason": "uncertain inputs",
>   "flags": ["ASSUMPTIONS"]
> }
> ```

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

**Agent task:**

> You are a Starburst Warp Speed NVMe cache sizing specialist (Elite tier). Calculate the minimum number of NVMe-backed worker nodes required to hold the hot dataset. Return JSON only.
>
> **Formula:**
> ```
> effective_volume_GB = volume_query_GB × (1 - cache_hit_rate)
> cache_per_worker_GB = NVMe_GB_per_worker × 0.85
> cache_workers_min   = ceil(hot_data_GB / cache_per_worker_GB)
> ```
>
> **NVMe defaults if `NVMe_GB_per_worker = null`:**
>
> | Instance | NVMe | Effective cache |
> |---|---|---|
> | AWS i3.8xlarge | 6,553 GB | ~5,570 GB |
> | AWS i4i.8xlarge | 3,840 GB | ~3,264 GB |
> | Azure L32s_v3 | 5,242 GB | ~4,456 GB |
> | On-prem unknown | 2,048 GB (conservative) | ~1,741 GB |
>
> If NVMe unknown → use 2,048 GB and add `ASSUMPTIONS` to flags.
>
> **Return:**
> ```json
> {
>   "effective_volume_GB": 400,
>   "cache_per_worker_GB": 5570,
>   "cache_workers_min": 3,
>   "node_type_recommended": "NVMe-backed node (e.g. AWS i3.8xlarge, Azure L32s_v3, or customer on-prem NVMe spec)",
>   "flags": ["WARP SPEED"]
> }
> ```

---

### Merge — workers retained

After all agents return:

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

Carry forward to Phase C: `workers_retained`, `dominant_constraint`, all agent outputs for Phase D, `all_flags`.

---

## Phase C — Configure

> **Scope: this skill sizes SEP (Starburst Enterprise Platform) on-prem deployments (K8s / OpenShift / VM).** Galaxy sizing follows the same methodology but with cloud-managed node types.

### Node specification constraints

**Mandatory — from Starburst technical documentation:**

| Constraint | Rule |
|---|---|
| **Minimum production node** | 16 vCPU / 64 GB RAM — coordinator and workers |
| **Homogeneous cluster** | All worker nodes must have identical specs (vCPU, RAM, storage) — no mixed configurations |
| **Warp Speed homogeneity** | If NVMe cache active: ALL workers must be NVMe-backed — no mixing standard and NVMe nodes |

**Recommended production node specs (reference):**

| Role | Minimum | Recommended |
|---|---|---|
| Coordinator | 16 vCPU / 64 GB RAM | 32 vCPU / 128 GB RAM |
| Worker (standard) | 16 vCPU / 64 GB RAM | 32 vCPU / 128 GB RAM |
| Worker (Warp Speed / NVMe) | 16 vCPU / 64 GB RAM + NVMe | 32 vCPU / 128 GB RAM + NVMe |

> Use the customer's actual node spec to compute `Workers = ceil(target_vCPUs / vCPU_per_node)`. Default reference: **32 vCPU per node** if not specified.

### SEP cluster config

| Component | Recommendation | Note |
|---|---|---|
| Coordinator | **≥ 32 vCPU / 128 GB RAM** (production) | Minimum 16 vCPU / 64 GB — always size generously |
| Prod workers | `max(T-shirt, network)` workers | Homogeneous — all same spec |
| Max workers | **32 / coordinator** | Beyond → multi-coordinator (DSR) — flag immediately |

> Coordinator metrics (CPU/RAM) are not exposed by default → instrument with JMX or Starburst Insights.

### Autoscaling

| Workload | Autoscaling recommendation |
|---|---|
| BI dashboards | ✅ Strongly recommended — predictable peak/off-peak patterns |
| Ad-hoc | ⚠️ Optional — unpredictable bursts, warmup may impact SLA |
| ETL | ❌ Not recommended — prefer batch scheduling (jobs during off-peak hours) |
| Mixed | ✅ Recommended for the BI component; exclude ETL windows |

Parameters:
- `min_workers` = floor matching off-peak measured during POC
- `max_workers` = workers calculated above (tested ceiling)
- Do not activate without having tested the max in POC — autoscaling won't compensate for under-sizing

### Growth headroom

```
Final prod workers = ceil(workers × (1 + growth% / 100))
```

### Multi-environment

| Env | Workers | vCPUs |
|---|---|---|
| **Production** | N (calculated) | N × vCPU_per_node |
| **DR** | per mode — see below | — |
| **UAT** | ceil(N × 0.30) | — |
| **DEV** | max(2, ceil(N × 0.10)) | — |
| **Total licensed** | sum | sum × vCPU_per_node |

### Disaster Recovery (DR)

Always ask if DR is required — common in FSI/banking and significantly impacts total vCPUs. Clarify the mode before sizing.

| DR mode | Description | DR workers steady-state | DR workers failover | Licensing |
|---|---|---|---|---|
| **Active/Active** | Traffic split permanently between PROD and DR | = N (100% of PROD) | = N | Doubles total — major impact |
| **Active/Passive Warm** | Identical architecture, reduced steady-state | 30–50% of N | = N (scale-out on DR event) | Based on steady-state |
| **Active/Passive Cold** | Minimal standby, rebuilt on DR event | 1 coordinator only | = N (fresh deployment) | Minimal — high RTO |

**Default recommendation: Active/Passive Warm**
- Same version, same config, same node types as PROD
- Reduced workers in steady-state (e.g. 2 workers on a 7-worker PROD) to limit costs
- Automatic scale-out (Kubernetes / script) on DR event or DR test
- Customer does not pay 100% of duplicated compute 24×7

**Warm DR sizing:**
```
DR workers steady-state = max(2, ceil(N × 0.40))
DR workers failover     = N  (identical to PROD — document as "envelope max")
```

> For customers with strong regulatory requirements (banks, insurance): verify whether the contract mandates Active/Active or if Active/Passive Warm is accepted. The licensing cost impact is major — involve the AE.

### Node type — Standard vs Warp Speed

| Scenario | Worker node type | Constraint |
|---|---|---|
| Standard (no Warp Speed) | Compute node: ≥ 16 vCPU / 64 GB RAM, recommended 32 vCPU / 128 GB | All workers identical |
| Warp Speed active | NVMe-backed node: same vCPU/RAM specs + local NVMe storage | **All workers must be NVMe** — no mixing |

> ⚠️ Warp Speed requires **Elite tier**. Always quantify the trade-off before recommending:
> `Cost (N NVMe workers + Elite license)` vs `Cost (N+k standard workers + Standard license)`
> Involve the AE on the licensing aspect.

**NVMe reference — cloud examples (on-prem: use customer's actual spec):**

| Cloud | Example instance | vCPUs | NVMe | Effective cache (×0.85) |
|---|---|---|---|---|
| AWS | i3.8xlarge | 32 | 6,553 GB | ~5,570 GB |
| AWS | i4i.8xlarge | 32 | 3,840 GB | ~3,264 GB |
| Azure | L32s_v3 | 32 | 5,242 GB | ~4,456 GB |
| On-prem | Customer NVMe node | ≥ 16 | Per spec | NVMe × 0.85 |

### SEP on-prem infrastructure overhead (K8s / OpenShift / VM)
Same worker calculation. Add infrastructure services: HMS, Ranger, Backend Service (~4–8 vCPUs each depending on load). Provision separately — do not include in licensed vCPU count.

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
Workload type       : <BI dashboards / Ad-hoc / ETL / Mixed — specify ratio>
Expected bottleneck : <Coordinator (BI) / Worker memory (ad-hoc) / Worker compute (ETL)>
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
Coordinator  : 1 node  — <vCPU> vCPU / <RAM> GB  (recommended ≥ 32 vCPU / 128 GB)
Workers      : <n> nodes — <vCPU> vCPU / <RAM> GB each  [homogeneous]
Total vCPUs  : <n> (workers) + <n> (coordinator)
Autoscaling  : <yes min=<n>/max=<n> / no — flat workload>
Recommended memory config:
  query.max-memory          = <value per workload>
  query.max-memory-per-node = <value per workload>

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MULTI-ENVIRONMENT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Prod          : <n> workers  →  <n> vCPUs
DR (steady)   : <n> workers  →  <n> vCPUs  (<mode: warm ~40% / cold=0 / AA=100%>)
DR (failover) : <n> workers  →  <n> vCPUs  (envelope max, not licensed continuously if warm/cold)
UAT           : <n> workers  →  <n> vCPUs  (~30% prod)
DEV           : <n> workers  →  <n> vCPUs  (~10% prod)
──────────────────────────────────────────────────────
TOTAL licensed: <n> vCPUs  (steady-state base)
TOTAL envelope: <n> vCPUs  (if DR scaled to full capacity)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WARP SPEED / CACHING                    ← omit if not active
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Workload profile    : <BI dashboards / ad-hoc / ETL>
Cache hit rate      : ~<N>%
Raw volume/query    : <N> GB
Effective volume    : <N> GB  (<N>% of raw volume after cache)
Hot data size       : <N> GB
Cache per worker    : <N> GB  (NVMe <N> GB × 0.85)
Workers (cache min) : <N>  = ceil(<hot_data> / <cache_per_worker>)
Workers retained    : max(T-shirt=<N>, cache=<N>) = <N>
Node type           : NVMe-backed — <vCPU> vCPU / <RAM> GB / <NVMe> GB NVMe  [ALL workers NVMe]
⚠️ Elite tier required — involve AE (trade-off license + node cost)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
POV STARTING CONFIG  (GTM Playbook)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Node spec →  customer node spec (min 16 vCPU / 64 GB RAM per node)
Start     →  1 coordinator + 8 workers — same spec, homogeneous
Iterate   →  scale coordinator OR workers (never both) if not sustained
Target    →  <n> final workers  (convergence in 2–3 rounds)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PRE-POV CHECKLIST
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
□ ANALYZE on all test tables        (missing stats = cluster appears undersized)
□ QPS limit raised above 40         (SEP default — must do before any load test)
□ Average file size: 64–512 MB      (small-file problem = degraded performance)
□ 5–10 representative queries       (peak + common)
□ Target latencies confirmed        (P50 / P95 / P99)
□ All nodes homogeneous             (coordinator + workers — same vCPU/RAM spec)
□ Node spec ≥ 16 vCPU / 64 GB RAM   (production minimum per Starburst documentation)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠  FLAGS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Show only those that apply]

[ASSUMPTIONS]  <list assumed inputs and their impact on accuracy>
[JDBC]         Single-thread connector (~5 MB/s) — linear model not applicable;
               maximize pushdowns, consider MVs / Table Scan Redirections
[MIXED]        Test JDBC workloads separately, do not mix into the linear model
[>32 WORKERS]  Multi-coordinator (DSR) required — alert SE manager
[XL]           T-shirt XL: JMeter POC mandatory before any sizing commitment
[COORDINATOR]  Coordinator CPU/RAM metrics not exposed by default → configure JMX or Insights
[HA/DR]        Not included in this sizing — clarify Active/Passive vs Active/Active
[WARP SPEED]   Elite tier required — all workers must be NVMe (no mixing) — quantify trade-off
               (N NVMe workers + Elite) vs (N+k standard workers + Standard) before recommending
[ETL+WS]       Warp Speed has no value on ETL/full-scan workloads — do not recommend
[NODE-SPEC]    Node spec below recommended (< 32 vCPU / 128 GB) — validate with customer
```

---

## Phase D.5 — Parallel verification (HITL)

Spawn the 6 agents below **in a single message** (parallel). Each agent receives only the slice of the Phase D output it needs. **Do not proceed to Phase E until all agents return and the merged verdict is presented to the user.**

### Agent persona (all 6 agents)

> You are a Senior Starburst Solutions Architect with 8+ years of experience sizing and deploying Starburst Enterprise clusters across FSI, insurance, healthcare, and public sector. You review sizing with a critical eye. Return a structured JSON report only — no prose outside the JSON.

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

**Checks:**

| Check | Severity |
|---|---|
| Warp Speed active with ETL > 50% | ❌ |
| BI dashboards but coordinator < 32 vCPU / 128 GB RAM | ❌ |
| Ad-hoc with `query.max-memory` = 20 GB (default) | ⚠️ |
| Mixed BI+ETL without separate-cluster note | ⚠️ |
| Workload type inferred (not confirmed) without [ASSUMPTIONS] flag | ⚠️ |

**Return:**
```json
{
  "check": "WORKLOAD COHERENCE",
  "results": [{"status": "✅|❌|⚠️", "detail": "..."}],
  "blocking": 0, "warnings": 0
}
```

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

**Checks:**

| Check | Severity |
|---|---|
| `workers_retained` < 3 (production minimum) | ❌ |
| `workers_with_growth` outside valid T-shirt range | ❌ |
| Growth headroom not applied (`workers_with_growth` = raw value) | ⚠️ |
| T-shirt XL without [XL] flag | ❌ |
| Workers > 32 without [>32 WORKERS] flag | ❌ |
| ≥ 1 input unconfirmed and sensitivity table absent | ⚠️ |

**Return:** same structure as V1

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

**Checks:**

| Check | Severity |
|---|---|
| Uncertain inputs but N/A not noted | ⚠️ |
| Network is dominant but `workers_retained` = T-shirt value (lower) | ❌ |
| Warp Speed active but raw volume used in check (not effective) | ❌ |

**Return:** same structure as V1

---

### Agent V4 — WARP SPEED CACHE SIZING *(skip if `warp_speed_active = false`)*

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

**Checks:**

| Check | Severity |
|---|---|
| `workers_retained` ≠ max(t_shirt, cache_workers_min) | ❌ |
| Cache per worker missing 0.85 safety margin | ❌ |
| NVMe left as default without [ASSUMPTIONS] flag | ⚠️ |
| Elite tier trade-off not mentioned | ⚠️ |
| AE involvement not flagged | ⚠️ |

**Return:** same structure as V1, or `{"check": "WARP SPEED", "skipped": true}` if inactive

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

**Checks:**

| Check | Severity |
|---|---|
| Active/Active without licensing impact stated | ❌ |
| DR mode unspecified while `dr_required = true` | ❌ |
| Warm DR: `dr_workers_steady` ≠ `max(2, ceil(N × 0.40))` | ⚠️ |
| FSI/insurance client with no regulatory DR mention | ⚠️ |

**Return:** same structure as V1, or `{"check": "DR SIZING", "skipped": true}` if not required

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

**Required flags by condition:**

| Condition | Required flag |
|---|---|
| Any input unconfirmed | `[ASSUMPTIONS]` |
| JDBC/database source present | `[JDBC]` |
| Workers > 32 | `[>32 WORKERS]` |
| T-shirt = XL | `[XL]` |
| Coordinator metrics discussed | `[COORDINATOR]` |
| DR excluded or not sized | `[HA/DR]` |
| Warp Speed active | `[WARP SPEED]` |
| ETL workload + Warp Speed considered | `[ETL+WS]` |

**Return:** same structure as V1

---

### Merge — verdict

After all 6 agents return:

```
total_blocking = sum(Vi.blocking  for all non-skipped Vi)
total_warnings = sum(Vi.warnings  for all non-skipped Vi)
ready_to_save  = (total_blocking == 0)
```

Compile and display the full report:

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
  ⚠️  Warnings: <N> — validate with customer
  ✅ Recommendation validated: <yes / no>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### HITL — before Phase E

> "Sr. SA peer review complete. <N> blocking issue(s) / All checks passed.
> Fix the issues above and re-run, or confirm to save?"

- Blocking issues → **do not proceed**. Fix and re-run.
- Warnings only or clean → proceed to Phase E on confirmation.

---

## Phase E — Save (optional)

If the sizing is for a specific account, propose:

```
account/<Account>/<YYYY-MM-DD> - Cluster Sizing Recommendation.md
```

Confirm: "Sizing saved → `account/<Account>/YYYY-MM-DD - Cluster Sizing Recommendation.md`"

---

## Absolute rules

| Rule | Reason |
|---|---|
| Minimum production node: 16 vCPU / 64 GB RAM | Starburst technical documentation — hard floor, no exception |
| Minimum production workers: 3 | Starburst technical documentation — never output fewer than 3 workers |
| All worker nodes must be homogeneous (identical spec) | Platform constraint — mixed configs not supported |
| Warp Speed: all workers must be NVMe — no mixing | Homogeneity rule applies to storage as well |
| Coordinator always ≥ 32 vCPU / 128 GB RAM for POV/prod | Under-provisioned coordinator = guaranteed bottleneck under concurrency |
| Never scale coordinator + workers in the same POC iteration | Can't attribute improvement without isolation |
| Never apply the linear model on JDBC-only sources | Throughput capped at ~5 MB/s (single-thread) |
| Always show the network check or explain why N/A | T-shirt and network can give different answers |
| Max 32 workers per coordinator | Platform limit — flag immediately if exceeded |
| QPS limit = 40 SEP default | Always on the checklist, always raise before load test |
| Always clarify DR mode before sizing (Warm / Cold / AA) | Impact on total licensed vCPUs can double the cost — involve AE |
| Sensitivity table if ≥ 1 input uncertain | A single answer on a fragile assumption = false precision |
| Qualify workload profile before scoring with Warp Speed | BI hit rate ≠ ETL hit rate — adjusted T-shirt can be wrong without this |
| Never recommend Warp Speed on ETL-dominant workload | Cache hit rate < 10% — no benefit, Elite surcharge not justified |
| Always quantify NVMe vs standard trade-off before concluding | AE must validate Elite license impact before commitment |
| Use customer's actual node vCPU count to compute worker count | `ceil(target_vCPUs / vCPU_per_node)` — default 32 vCPU/node if unspecified |
