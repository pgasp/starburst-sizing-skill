# starburst-sizing-skill

A Claude Code slash command (`/starburst-sizing`) that sizes a Starburst Enterprise Platform (SEP) cluster from customer workload inputs.

Methodology: **GTM Playbook v1.0 — Citi T-shirt grid — linear bandwidth model**

---

## What it does

Given workload inputs (concurrency, data volume, SLA, deployment type), the skill:

1. Translates business-language answers into sizing parameters
2. Runs T-shirt scoring (Citi method), network bandwidth check, and Warp Speed NVMe cache sizing **in parallel**
3. Outputs a full cluster recommendation: node spec, worker count, multi-environment sizing, DR modes, POV starting config
4. Runs a **parallel peer review** (6 independent verification agents simulating a Sr. SA check) with HITL before saving

---

## Scope

- **Primary target:** SEP on-prem (K8s / OpenShift / VM)
- **Secondary:** Galaxy (same methodology, cloud-managed node types)
- **Not covered:** SEP SaaS, Starburst Galaxy auto-scaling (managed)

---

## Files

```
skill/
  starburst-sizing.md          ← Claude Code slash command
docs/
  architecture.md              ← full architecture documentation
questionnaires/
  questionnaire-client-fr.md   ← customer intake form (French)
  questionnaire-client-en.md   ← customer intake form (English)
```

---

## Installation

Copy `skill/starburst-sizing.md` to your Claude Code commands directory:

```bash
cp skill/starburst-sizing.md ~/.claude/commands/
# or into your project:
cp skill/starburst-sizing.md .claude/commands/
```

Then invoke with `/starburst-sizing` in Claude Code.

---

## Usage

**Option A — direct inputs:**
```
/starburst-sizing  50 concurrent users, 500 GB Iceberg on S3, SLA 5s, OpenShift on-prem, DR Warm
```

**Option B — intake form:**
Run `/starburst-sizing` with no inputs. The skill outputs the customer intake form. Fill it in, paste it back, and the skill converts the answers via its translation table.

**Option C — send the questionnaire to the customer first:**
Use `questionnaires/questionnaire-client-fr.md` or `questionnaire-client-en.md` — business-language forms with no Starburst jargon. Paste the filled answers into `/starburst-sizing`.

---

## Architecture

See [`docs/architecture.md`](docs/architecture.md) for the full flow diagram, agent inventory, data flow, T-shirt grid, and flags reference.

---

## Hard constraints (Starburst documentation)

| Constraint | Value |
|---|---|
| Minimum production node | 16 vCPU / 64 GB RAM |
| Minimum production workers | 3 |
| Homogeneous cluster | All worker nodes must be identical |
| Warp Speed | All workers NVMe — no mixing |

---

## Save path (Phase E)

By default the skill saves to `account/<Account>/<date> - Cluster Sizing Recommendation.md`.
Adapt this path to your workspace structure.

---

## Requirements

- [Claude Code](https://claude.ai/code) CLI
- No external MCP tools required — the skill runs entirely within the conversation context
