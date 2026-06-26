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

**If inputs provided:** extract from conversation/notes, jump to Phase B after translation table.

### If no inputs provided — output the intake form

Read and display `questionnaires/questionnaire-client-en.md` (EN) or `questionnaires/questionnaire-client-fr.md` (FR). Wait for the customer to fill it in, then apply the **translation table** below.

> File not found? Ask directly: peak concurrent users, target query SLA, approximate data volume.

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

Ask only if mentioned: workload profile (BI / ad-hoc / ETL) + hot data volume (e.g. last 3 months of 5 years).
Cache hit rate by profile: BI 80–95% ✅ strong ROI | Ad-hoc 20–40% ⚠️ moderate | ETL < 10% ❌ do not recommend.

### Sensitivity table (if ≥ 1 metric uncertain)

| Scenario | Concurrency | Volume/query | SLA | T-shirt | Prod workers |
|---|---|---|---|---|---|
| Optimistic | ≤10 | ≤2 GB | >15s | S | 5 |
| Median | 11–50 | 2–10 GB | 5–15s | M | 9 |
| Conservative | >50 | >10 GB | <5s | L | 15 |

---

## Phase B — Parallel sizing computation

Spawn in a **single message** (parallel). After all return, merge.

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

| Role | Min (hard floor) | Recommended |
|---|---|---|
| Coordinator | 16 vCPU / 64 GB | 32 vCPU / 128 GB |
| Worker (standard) | 16 vCPU / 64 GB | 32 vCPU / 128 GB |
| Worker (NVMe) | 16 vCPU / 64 GB + NVMe | 32 vCPU / 128 GB + NVMe |

All workers must be identical spec. Warp Speed → all workers NVMe (no mixing). `Workers = ceil(target_vCPUs / vCPU_per_node)` — default 32 vCPU/node.

### SEP cluster config

| Component | Recommendation |
|---|---|
| Coordinator | ≥ 32 vCPU / 128 GB RAM — always size generously |
| Workers | `workers_retained` — homogeneous, same spec |
| Max workers | 32 / coordinator — beyond → DSR, flag immediately |

### Autoscaling

| Workload | Recommendation |
|---|---|
| BI dashboards | ✅ Recommended — predictable peak/off-peak |
| Ad-hoc | ⚠️ Optional — unpredictable bursts |
| ETL | ❌ Not recommended — prefer batch scheduling |
| Mixed | ✅ For BI component; exclude ETL windows |

`min_workers` = off-peak floor (POC); `max_workers` = workers_retained (tested ceiling).

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

Always clarify DR mode before sizing (licensing impact can double cost — involve AE):

| DR mode | Steady-state workers | Failover workers | Licensing |
|---|---|---|---|
| **Active/Active** | = N (100%) | = N | Doubles total |
| **Active/Passive Warm** | max(2, ceil(N × 0.40)) | = N (scale-out on event) | Steady-state |
| **Active/Passive Cold** | 1 coordinator | = N (fresh deploy) | Minimal — high RTO |

### Node type

| Scenario | Node type | Constraint |
|---|---|---|
| Standard | ≥ 16 vCPU / 64 GB, recommended 32/128 | All workers identical |
| Warp Speed | Same vCPU/RAM + local NVMe | **All workers NVMe** — no mixing |

> ⚠️ Warp Speed = **Elite tier**. Quantify: `(N NVMe + Elite)` vs `(N+k standard + Standard)`. Involve AE. NVMe reference: see Phase B NVMe defaults table.

---

## Phase D — Output

```
STARBURST SIZING — <account>  |  <YYYY-MM-DD>

── WORKLOAD INPUTS ───────────────────────────────────────
Workload type       : <BI dashboards / Ad-hoc / ETL / Mixed>
Expected bottleneck : <Coordinator / Worker memory / Worker compute>
Concurrency         : <n> users  [confirmed / assumption]
Volume/query        : <n> GB     [confirmed / assumption]
Target SLA          : <n> s      [confirmed / assumption]
Data sources        : <Lake / JDBC / Mixed>
Deployment          : <Galaxy / SEP K8s / SEP VM>
DR required         : <yes — mode: Active/Active | Warm | Cold / no>
Expected growth     : <n>% / 12 months

── T-SHIRT SCORING ───────────────────────────────────────
Concurrency  : <n> users  → <S/M/L> (<pts> pt)
Volume       : <n> GB     → <S/M/L> (<pts> pt)
SLA          : <n> s      → <S/M/L> (<pts> pt)
             ──────────────────────────────────
Total        : <n> pts  → T-Shirt <S/M/L/XL>  →  <n> vCPUs  →  <n> workers

── NETWORK CHECK ─────────────────────────────────────────
<detailed calculation OR "N/A — uncertain inputs">
Dominant constraint : <T-shirt / network>  →  <n> workers

── PRODUCTION RECOMMENDATION  (growth +<n>%) ────────────
Node spec    : <vCPU> vCPU / <RAM> GB RAM  (min 16 vCPU / 64 GB)
Coordinator  : 1 node — <vCPU> vCPU / <RAM> GB  (recommended ≥ 32 / 128)
Workers      : <n> nodes — <vCPU> vCPU / <RAM> GB each  [homogeneous]
Total vCPUs  : <n> (workers) + <n> (coordinator)
Autoscaling  : <yes min=<n>/max=<n> / no>
  query.max-memory          = <value>
  query.max-memory-per-node = <value>

── MULTI-ENVIRONMENT ─────────────────────────────────────
Prod          : <n> workers  →  <n> vCPUs
DR (steady)   : <n> workers  →  <n> vCPUs  (<mode>)
DR (failover) : <n> workers  →  <n> vCPUs  (envelope max)
UAT           : <n> workers  →  <n> vCPUs  (~30% prod)
DEV           : <n> workers  →  <n> vCPUs  (~10% prod)
TOTAL licensed: <n> vCPUs  |  TOTAL envelope: <n> vCPUs

── WARP SPEED / CACHING  (omit if not active) ────────────
Workload profile : <BI / ad-hoc>  |  Cache hit rate : ~<N>%
Raw vol/query    : <N> GB  →  Effective: <N> GB after cache
Hot data         : <N> GB  |  Cache/worker: <N> GB (NVMe × 0.85)
Workers (cache)  : ceil(<hot>/<cache_pw>) = <N>
Workers retained : max(T-shirt=<N>, cache=<N>) = <N>
Node type        : NVMe-backed — <vCPU>/<RAM>/<NVMe>  [ALL workers NVMe]
⚠️ Elite tier — involve AE (trade-off: license + node cost)

── POV STARTING CONFIG ───────────────────────────────────
Start   →  1 coordinator + 8 workers — same spec (min 16 vCPU / 64 GB), homogeneous
Iterate →  scale coordinator OR workers (never both) if not sustained
Target  →  <n> final workers

── PRE-POV CHECKLIST ─────────────────────────────────────
□ ANALYZE on all test tables  (missing stats = cluster appears undersized)
□ QPS limit raised above 40   (SEP default — must do before any load test)
□ Avg file size: 64–512 MB    (small-file problem = degraded perf)
□ 5–10 representative queries (peak + common); latencies P50/P95/P99 confirmed
□ All nodes homogeneous — spec ≥ 16 vCPU / 64 GB RAM

── FLAGS (show only applicable) ──────────────────────────
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

Spawn in a **single message** (parallel). Do not proceed to Phase E until all return and verdict is presented.

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
PEER REVIEW — <client>  T-shirt: <X>  Workers: <N>
V1 WORKLOAD COHERENCE  ✅/❌/⚠️  <results>
V2 T-SHIRT SCORING     ✅/❌/⚠️  <results>
V3 NETWORK CHECK       ✅/❌/⚠️  <results>
V4 WARP SPEED CACHE    ✅/❌/⚠️  <results>  [SKIPPED if inactive]
V5 DR SIZING           ✅/❌/⚠️  <results>  [SKIPPED if no DR]
V6 FLAGS               ✅/❌/⚠️  <results>
VERDICT: ❌ Blocking: <N>  ⚠️ Warnings: <N>  ✅ Ready: <yes/no>
```

**HITL:** Present verdict. Blocking → do not proceed, fix and re-run. Warnings only / clean → confirm to save.

---

## Phase E — Save (optional)

Propose and confirm: `account/<Account>/<YYYY-MM-DD> - Cluster Sizing Recommendation.md`

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
