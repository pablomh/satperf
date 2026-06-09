# PQC Program Investigation Deliverables

Investigation of parent JIRAs [PSASTRAT-109](https://redhat.atlassian.net/browse/PSASTRAT-109), [SAT-35690](https://redhat.atlassian.net/browse/SAT-35690), and [CANDLEPIN-1021](https://redhat.atlassian.net/browse/CANDLEPIN-1021), with performance implications and backlog improvements.

**Target delivery:** May 2027 (aligned with RHEL 10.4 PQC crypto policy)

---

## Start here

### [satellite-pqc-performance-impact.md](./satellite-pqc-performance-impact.md) — **Unified document**

Single entry point for Satellite PQC performance: cost model, path/workload impact, registration PR stack, SAT-43398/SAT-38910/SAT-19326, Candlepin, gaps, mitigations, SLOs, test plan (R0–R4), dependencies, and recommended actions.

---

## Supporting documents

| File | Purpose |
|------|---------|
| [pqc-tls-handshake-research.md](./pqc-tls-handshake-research.md) | Quantitative TLS / initcwnd research |
| [pqc-initcwnd-impact-by-path.md](./pqc-initcwnd-impact-by-path.md) | Where initcwnd overflow applies |
| [pqc-registration-pr-stack-impact.md](./pqc-registration-pr-stack-impact.md) | Registration perf PRs × PQC (detail) |
| [candlepin-pqc-performance-impact.md](./candlepin-pqc-performance-impact.md) | Candlepin-specific (CP-SLO-1–9) |
| [pqc-performance-slos.md](./pqc-performance-slos.md) | SLO table + satperf playbook mapping |
| [pqc-phased-perf-test-plan.md](./pqc-phased-perf-test-plan.md) | Phases 0–5 execution detail |
| [pqc-performance-mitigations.md](./pqc-performance-mitigations.md) | Mitigation detail + SSH/gems |
| [pqc-integration-gaps.md](./pqc-integration-gaps.md) | Candlepin-closed vs Satellite-open |
| [pqc-dependency-tracker.md](./pqc-dependency-tracker.md) | External dependency RAG |
| [pqc-analysis-reconciliation.md](./pqc-analysis-reconciliation.md) | Investigation merge index |
| [sat-38970-restructure-proposal.md](./sat-38970-restructure-proposal.md) | SAT-38970 matrix proposal |
| [pqc-issue-hierarchy.csv](./pqc-issue-hierarchy.csv) | Jira export (140+ rows) |
| [pqc-issue-hierarchy.md](./pqc-issue-hierarchy.md) | Hierarchy summary stats |
| [pqc-tls-component-matrix.csv](./pqc-tls-component-matrix.csv) | SAT-38970 component matrix |

---

## Quick findings

See [satellite-pqc-performance-impact.md § Executive summary](./satellite-pqc-performance-impact.md#1-executive-summary). In brief:

- **Dominant perf risk:** TLS handshake **size** (initcwnd), not ML-DSA signing CPU.
- **CANDLEPIN-1021 closed** ≠ Satellite PQC perf validated.
- **Registration PR stack** + **connection pooling** = strongest reg@scale mitigations.
- **SAT-43398** → **SAT-38910** + **SAT-19326** = Foreman↔CP architecture prerequisites.
- **Client handshake/host** remains the SAT-35690 acceptance risk after internal fixes.

---

## Parent links

- [PSASTRAT-109](https://redhat.atlassian.net/browse/PSASTRAT-109) — Portfolio outcome (Planning)
- [SAT-35690](https://redhat.atlassian.net/browse/SAT-35690) — Satellite outcome (To Do)
- [CANDLEPIN-1021](https://redhat.atlassian.net/browse/CANDLEPIN-1021) — Candlepin epic (Closed)
