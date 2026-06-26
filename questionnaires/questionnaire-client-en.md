# Platform Sizing Questionnaire — Client Version (EN)

*Send to the customer before the technical scoping call. Fill in and return by email or share during the meeting.*

---

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  PLATFORM SIZING — QUESTIONNAIRE
  Fill in what you know. Leave blank what you don't.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

SECTION 1 — USERS
─────────────────────────────────────────────────────────
1a. How many people use the platform at the same time
    during peak hours ?
    → __________ concurrent users

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
    [ ] Stable           [ ] ~20% per year
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
        (trading, critical operations, regulatory requirement)
    [ ] Up to 15–30 minutes — automatic failover acceptable
    [ ] Up to a few hours — manual intervention acceptable
    [ ] No specific requirement

5b. Any regulatory or contractual availability obligations?
    [ ] Yes — specify: ____________________
    [ ] No / Unknown

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
