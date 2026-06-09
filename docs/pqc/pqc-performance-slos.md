# PQC Performance SLO Proposal

> **Unified doc:** [satellite-pqc-performance-impact.md](./satellite-pqc-performance-impact.md) §11.

Proposed service-level objectives to supplement [SAT-35690](https://redhat.atlassian.net/browse/SAT-35690), which requires **no functional degradation** but does not define quantitative performance targets.

## Measurement environment

| Parameter | Value |
|-----------|--------|
| Satellite OS | RHEL 10.4+ |
| Crypto policy (PQC) | `DEFAULT:PQ` or org-mandated PQC policy (document which) |
| Crypto policy (baseline) | `DEFAULT` (non-PQC) on same hardware |
| Client OS | RHEL 10.2+ for PQC client paths |
| Scale profile | Match existing satperf "standard" host counts unless noted |

Collect metrics via existing satperf stack: collectd → Graphite/Grafana, plus playbook timing logs.

**SLO-1 / SLO-2 prerequisite:** Measure registration with the **registration performance PR stack** applied ([pqc-registration-pr-stack-impact.md](./pqc-registration-pr-stack-impact.md)). Comparing `DEFAULT:PQ` on an unpatched Satellite conflates PQC overhead with known CP/TLS amplification (use matrix runs **R1** vs **R3**).

---

## SLO table

| ID | Workflow | Metric | Target (PQC vs baseline) | SAT-35690 metric | satperf playbook(s) |
|----|----------|--------|--------------------------|------------------|---------------------|
| SLO-1 | Host registration | p95 end-to-end registration time (per host) | ≤ **110%** of baseline | Integration test suite | `playbooks/tests/concurrent_execution.yaml`, `continuous-reg.yaml`, `roles/concurrent_task/tasks/execute_registration.yaml` |
| SLO-2 | Host registration (storm) | Successful registrations / minute at fixed concurrency | ≥ **95%** of baseline throughput | Manage PQC-enabled hosts | `concurrent_execution.yaml` (tune `size` / `total`) |
| SLO-3 | Candlepin cert issuance | p95 identity+entitlement cert generation latency | ≤ **125%** of baseline | Client registration TLS | FAM registration flows; API timing on Candlepin during reg |
| SLO-3b | *(CP detail)* | See **CP-SLO-1–9** in [candlepin-pqc-performance-impact.md](./candlepin-pqc-performance-impact.md) | Manifest, status endpoint, CP JVM, startup | CANDLEPIN-1021 closed work |
| SLO-4 | CDN repository sync | p95 sync task duration (large repo) | ≤ **115%** of baseline | Content sync workflows | `playbooks/tests/FAM/repo_sync.yaml`, `sync-repositories.yaml`, `sync-mixed-repos-one-cvs.yaml` |
| SLO-5 | CDN sync throughput | Effective GB/hour (wall clock) | ≥ **90%** of baseline | ML-DSA signed packages E2E | `sync-repositories.yaml` on RHEL 10 ML-DSA repo when available |
| SLO-6 | Content publish | p95 publish duration (CV with ML-DSA repo) | ≤ **110%** of baseline | Publish/promote workflows | `playbooks/tests/FAM/cv_publish.yaml` |
| SLO-7 | Content promote | p95 promote duration (multi-environment) | ≤ **110%** of baseline | Publish/promote workflows | `playbooks/tests/FAM/cv_version_promote.yaml` |
| SLO-8 | Capsule sync | p95 Capsule sync from Satellite | ≤ **115%** of baseline | Capsule serves ML-DSA content | `playbooks/tests/FAM/capsule_sync.yaml`, `capsules-sync.yaml` |
| SLO-9 | Remote execution | p95 per-host job latency (100-host job) | ≤ **120%** of baseline | PQC-enabled SSH | `playbooks/tests/rex.yaml`, FAM `job_invocation_create.yaml` |
| SLO-10 | TLS handshake | p95 TLS handshake time (external probe) | ≤ **130%** of baseline | TLS negotiates PQC when required | Custom `openssl s_time` / curl timing against Apache, Pulp, Candlepin vhosts |
| SLO-11 | Manifest import | p95 manifest import wall time (ML-DSA manifest) | ≤ **115%** of baseline | Candlepin ML-DSA manifests | `playbooks/tests/FAM/manifest_import.yaml` |
| SLO-12 | API availability | Error rate on Foreman API under load | ≤ baseline + **0.5%** absolute | No degradation in functionality | `api-get-task-duration.yaml`, Hammer load (if used) |
| SLO-13 | CPU saturation | Peak CPU on Satellite during SLO-2/4/9 | ≤ **125%** of baseline peak | Operational stability | collectd CPU metrics during concurrent runs |
| SLO-14 | Memory | Peak RSS of httpd, tomcat/candlepin, pulpcore during SLO-4/6 | ≤ **115%** of baseline | Dual-cert / larger chains | collectd memory plugins |

### Notes on targets

- Percentages are initial proposals; calibrate after first baseline run on RHEL 10.4 PQC lab.
- **Dual certificate** mode (RSA + ML-DSA) is expected to be the worst case for SLO-10; measure explicitly.
- If any SLO exceeds target, file a Satellite perf defect linked to SAT-35690.

---

## Mapping to SAT-35690 success criteria

| SAT-35690 criterion | SLOs |
|---------------------|------|
| Deploy/operate on RHEL 10.4+ with PQC policies, no degradation | SLO-1–14 (aggregate gate) |
| Full integration test suite passes | All SLOs in CI perf lane (subset for PR, full for release) |
| Content sync/publish/promote with ML-DSA packages | SLO-4–8 |
| Remote execution over PQC SSH | SLO-9 |
| TLS across all services negotiates PQC | SLO-10 (+ SAT-38970 matrix for functional coverage) |
| 30-day adoption trend (post-GA) | Out of scope for satperf; ops telemetry |

---

## satperf implementation checklist

1. Add `pqc_crypto_policy` variable to `conf/satperf.yaml` (e.g. `DEFAULT:PQ` vs `DEFAULT`).
2. Add playbook pre-task: set crypto-policy on Satellite + clients when `pqc_crypto_policy` is set.
3. Tag existing playbooks with `pqc_phase` labels in a wrapper playbook `playbooks/tests/pqc/pqc-regression.yaml`.
4. Store baseline vs PQC results under `run-<timestamp>/pqc-baseline/` and `pqc-pq/`.
5. Publish comparison template (Grafana dashboard or spreadsheet) for release gate.

---

## Recommended new JIRA stories (perf-specific)

| Proposed title | Parent | Validates |
|----------------|--------|-----------|
| PQC perf baseline: registration storm (ML-DSA certs) | SAT-35690 or SAT-43394 | SLO-1, SLO-2, SLO-3 — **R0–R4 matrix** in [pqc-registration-pr-stack-impact.md](./pqc-registration-pr-stack-impact.md) |
| Registration perf PR stack as PQC dependency | SAT-35690 | Blocks honest SLO-2 gate; Katello #11731, #11726/#11754, Foreman #10979/80 |
| PQC perf baseline: dual-cert Apache under concurrent HTTPS | SAT-36257 | SLO-10, SLO-13 |
| PQC perf baseline: full-repo CDN sync ML-DSA | SAT-43401 / SAT-36256 | SLO-4, SLO-5 |
| PQC perf baseline: REX 100-host ML-KEM SSH | SAT-43331 | SLO-9 |
| PQC perf baseline: manifest import ML-DSA | SAT-43401 | SLO-11 |

---

## Gate criteria for GA (pre-GA evidence)

Before marking SAT-35690 complete:

- [ ] All SLO-1–12 measured on release candidate build
- [ ] No SLO regresses beyond target for two consecutive weekly runs
- [ ] SAT-38970 matrix executed as **test evidence** (not open story debt)
- [ ] Integration gaps in [pqc-integration-gaps.md](./pqc-integration-gaps.md) closed or waived with documented risk
