# PQC Analysis Reconciliation

> **Unified doc:** [satellite-pqc-performance-impact.md](./satellite-pqc-performance-impact.md) — primary entry point. This file is the investigation merge index.

This document merges the **satperf investigation** (May 2026) with a **parallel analysis** that includes quantitative TLS research, a fuller card inventory, and operational mitigations.

---

## Hierarchy (aligned)

```
HATSTRAT-310 (Consolidated Portfolio Crypto 2026) [In Progress]
  └── PSASTRAT-109 (2026 - Red Hat Satellite) [Planning]
        ├── SAT-35690 (Support PQC) [To Do] — main Satellite delivery outcome
        ├── SAT-36256 (ML-DSA signed content) [Refinement] — linked under program
        └── CANDLEPIN-1021 (PQC certificates) [Closed] — 34/34 children done
```

**Note:** Our first pass linked PSASTRAT-109 directly to SAT-35690/CANDLEPIN-1021 via Jira `linkedIssues`; **HATSTRAT-310** is the portfolio crypto parent above PSASTRAT-109 and should appear in portfolio reporting.

---

## Coverage comparison

| Topic | satperf docs | Parallel analysis | Action taken |
|-------|--------------|-------------------|--------------|
| Issue hierarchy CSV | 137 rows | Fuller feature→epic tree (7 features, blockers) | See [card inventory](#complete-sat-35690-feature-tree) below; refresh CSV optional |
| Quantitative TLS (cert size, initcwnd, +15–32% handshake) | Qualitative only | Detailed table + Cloudflare ~9KB threshold | [pqc-tls-handshake-research.md](./pqc-tls-handshake-research.md) |
| TCP initcwnd tuning mitigation | Not covered | High-impact recommendation | [pqc-performance-mitigations.md](./pqc-performance-mitigations.md) |
| ML-DSA **signing faster** than RSA | Implied CPU cost | Explicit ~0.2ms vs ~5ms | Updated [candlepin-pqc-performance-impact.md](./candlepin-pqc-performance-impact.md) |
| PRODSECRM-160 Java PQC delay | Missing | Blocks Foreman↔CP until Java 29 | Added to [pqc-dependency-tracker.md](./pqc-dependency-tracker.md) |
| CANDLEPIN-1102, 1162, 1171 | Missing | Open follow-on CP work | Added below |
| SAT-43402, SAT-36399 | Missing | Under SAT-36256 / SAT-36257 | Added to feature tree |
| SAT-43398 (Foreman↔CP spike) | In integration gaps only | Full analysis in registration stack doc | [pqc-registration-pr-stack-impact.md](./pqc-registration-pr-stack-impact.md) § JIRA |
| SAT-19326 (CP TLS 1.3 FR) | **Missing** | Prerequisite for PQC on CP TLS path | Added to integration gaps + registration stack doc |
| SAT-45175/45177/45196 | Partial (45428 area) | SSH gem replacement blockers | Added to dependency tracker |
| SAT-43405 container stack | Mentioned SAT-38913 | Full child epics listed | Cross-ref SAT-43405 |
| Dedicated perf test card | Proposed SLOs + phased plan | Explicit gap “no perf card” | Still recommended — link SAT-35690 to satperf plan |
| initcwnd / installer tuning card | Gap identified | Same gap | Proposed story in mitigations doc |
| Registration perf PR stack × PQC | Not in first pass | Full PR-tier analysis + satperf matrix | [pqc-registration-pr-stack-impact.md](./pqc-registration-pr-stack-impact.md) |

---

## Complete SAT-35690 feature tree

| Feature | Key | Status | Child epics / notes |
|---------|-----|--------|---------------------|
| ML-DSA signed content | [SAT-36256](https://redhat.atlassian.net/browse/SAT-36256) | Refinement | [SAT-43402](https://redhat.atlassian.net/browse/SAT-43402) validate workflows |
| TLS 1.3 + PQC infrastructure | [SAT-36257](https://redhat.atlassian.net/browse/SAT-36257) | Refinement | SAT-35691, [SAT-36399](https://redhat.atlassian.net/browse/SAT-36399), SAT-38907, [SAT-43398](https://redhat.atlassian.net/browse/SAT-43398)→SAT-38910, SAT-38913, SAT-43393, SAT-43394, [SAT-19326](https://redhat.atlassian.net/browse/SAT-19326) |
| HTTPS secure provisioning | [SAT-43229](https://redhat.atlassian.net/browse/SAT-43229) | Refinement | SAT-44797, SAT-44802, SAT-44803 |
| PQC SSH for REX | [SAT-43331](https://redhat.atlassian.net/browse/SAT-43331) | Refinement | SAT-38914 |
| CDN sync with PQC | [SAT-43401](https://redhat.atlassian.net/browse/SAT-43401) | Refinement | SAT-38909, SAT-38912 |
| RHEL 10 containers | [SAT-43405](https://redhat.atlassian.net/browse/SAT-43405) | Refinement | SAT-18384, SAT-36494, SAT-42202, SAT-43406, SAT-43407 |
| Client registration (also under 36257) | [SAT-43394](https://redhat.atlassian.net/browse/SAT-43394) | New | Overlaps 36257 children |

### Blockers / readiness (additional)

| Key | Summary | Status |
|-----|---------|--------|
| [SAT-45175](https://redhat.atlassian.net/browse/SAT-45175) | Replace sshkey with OpenSSH/libssh | New |
| [SAT-45177](https://redhat.atlassian.net/browse/SAT-45177) | Replace net-ssh with system SSH | New |
| [SAT-45196](https://redhat.atlassian.net/browse/SAT-45196) | Replace net-scp with SFTP | New |
| [SAT-45354](https://redhat.atlassian.net/browse/SAT-45354) | openscap > 1.4.0-2 (drop libgcrypt) | New |
| [SAT-45428](https://redhat.atlassian.net/browse/SAT-45428) | JWT/OIDC PQC readiness | New |
| [SAT-45429](https://redhat.atlassian.net/browse/SAT-45429) | Remove rack-openid/ruby-openid | New |
| [SAT-45455](https://redhat.atlassian.net/browse/SAT-45455) | pulp_container PyJWT PQC | New |

### Spikes

| Key | Summary | Status |
|-----|---------|--------|
| [SAT-42208](https://redhat.atlassian.net/browse/SAT-42208) | Crypto package analysis | Closed |
| [SAT-44424](https://redhat.atlassian.net/browse/SAT-44424) | PQC deps for RHEL 10.3 | Closed |
| [SAT-44657](https://redhat.atlassian.net/browse/SAT-44657) | Container crypto-policy enforcement | Closed |
| [SAT-38970](https://redhat.atlassian.net/browse/SAT-38970) | PQC KEX Sat↔Host TLS | New — prioritize per both analyses |

---

## Candlepin: closed vs still open

### CANDLEPIN-1021 (Closed) — implementation done

34 children closed; includes PKI refactor, multi-scheme certs, manifest sigs, tomcat-native/Java 25 dual certs ([CANDLEPIN-1152](https://redhat.atlassian.net/browse/CANDLEPIN-1152)), scheme negotiation.

### Follow-on (not “done” for program)

| Key | Summary | Status | Perf relevance |
|-----|---------|--------|----------------|
| [CANDLEPIN-1171](https://redhat.atlassian.net/browse/CANDLEPIN-1171) | PQC Certificates V2 | **Closed** | Anonymous cert generator scheme-aware |
| [CANDLEPIN-1102](https://redhat.atlassian.net/browse/CANDLEPIN-1102) | PQC Testing and Automation | **In Progress** | CP test automation — not load/perf |
| [CANDLEPIN-1162](https://redhat.atlassian.net/browse/CANDLEPIN-1162) | Ship Hosted PQC CA cert for Satellite | **Backlog** | **Blocks** connected CDN trust ([SAT-43401](https://redhat.atlassian.net/browse/SAT-43401)) |

**Reconciliation:** “CANDLEPIN-1021 closed” ≠ “PQC complete for Satellite.” CP perf validation and hosted CA shipping remain open.

---

## Performance: merged conclusions

### Primary bottleneck (both analyses agree)

**TLS handshake size**, not ML-DSA signing CPU:

- ML-DSA-65 chains ~**12 KB** vs ~3.5 KB classical → often exceeds default **TCP initcwnd** (10 × MSS) → **extra RTT** per new connection.
- Good network: ~**+15%** handshake latency; constrained: **+32%**; lossy: **6–8×** (critical for remote Capsules).
- Bulk transfer after connect: **&lt;5%** impact for payloads &gt;50 KB (CDN sync steady state).

### Where CP fits

| CP factor | Impact |
|-----------|--------|
| ML-DSA **signing** | **Faster** than RSA (~0.2ms vs ~5ms) — not the regression driver |
| ML-DSA **cert size** | **Larger** keys/certs → DB/memory + contributes to TLS chain bloat |
| Foreman↔CP path | [PRODSECRM-160](https://redhat.atlassian.net/browse/PRODSECRM-160) Java PQC delay; [SAT-38910](https://redhat.atlassian.net/browse/SAT-38910) may use unix socket (**perf win** vs TLS) |
| Scheme negotiation | Extra per-request work ([CANDLEPIN-1078](https://redhat.atlassian.net/browse/CANDLEPIN-1078)) — secondary to handshake RTT |

### Scale math (registration)

10,000 hosts × one extra RTT at 50ms RTT ≈ **500s** cumulative handshake penalty (order-of-magnitude; connection reuse reduces this).

---

## Merged risk register

| Risk | Severity | Mitigation | Tickets |
|------|----------|------------|---------|
| Registration storm without Katello stack | **High** | Merge reg perf PRs (#11731, #11696, #11726/#11754, etc.) | See [pqc-registration-pr-stack-impact.md](./pqc-registration-pr-stack-impact.md) |
| TLS handshake at scale (initcwnd overflow) | **High** | initcwnd 30–40, keep-alive, session resumption, HTTP/2 | SAT-36257, SAT-43401 |
| net-ssh replacement regression | **High** | Incremental rollout; image-based prov testing | SAT-45177 (PR #10626 reverted) |
| Java PQC delay → Foreman↔CP | **Medium** | Unix socket / tomcat-native | PRODSECRM-160, SAT-38910 |
| Lossy network / remote Capsule | **Medium** | Hybrid DEFAULT:PQ first; short chains | SAT-36257 |
| No dedicated perf benchmark card | **Medium** | satperf phased plan + SLOs | SAT-35690 |
| Hosted PQC CA not shipped | **Medium** | Prioritize CANDLEPIN-1162 | SAT-43401 |
| Cert storage growth | Low | DB sizing | SAT-43394 |
| Dual-cert transition | Low | Temporary | SAT-36257 |
| JWT/OIDC | Low | Monitor | SAT-45428, SAT-45455 |

---

## Merged improvement recommendations

See [pqc-performance-mitigations.md](./pqc-performance-mitigations.md) for detail. Top 7:

1. **Tune TCP initcwnd** (30–40) on Satellite/Capsule — installer or tuning guide  
2. **Maximize TLS connection reuse** — keep-alive, session tickets, HTTP/2 where applicable  
3. **Unix socket for Foreman↔Candlepin** if Java 29 PQC unavailable — may improve vs today  
4. **Registration perf PR stack** — Katello pooling + static compliance + DB headroom ([pqc-registration-pr-stack-impact.md](./pqc-registration-pr-stack-impact.md))  
5. **Short cert chains** — target auth data &lt;9 KB; consider RFC 8879 cert compression  
6. **Dedicated PQC perf suite** — satperf baselines + good vs lossy network ([pqc-phased-perf-test-plan.md](./pqc-phased-perf-test-plan.md))  
7. **Phased rollout** — `DEFAULT:PQ` (ML-KEM first) before pure ML-DSA everywhere  

### Proposed JIRA additions (both analyses)

| Proposed card | Parent | Rationale |
|---------------|--------|-----------|
| PQC performance benchmark suite | SAT-35690 | No explicit perf card today |
| Registration perf PR stack dependency for PQC SLOs | SAT-35690 / SAT-43394 | See [pqc-registration-pr-stack-impact.md](./pqc-registration-pr-stack-impact.md) |
| Installer: TCP initcwnd tuning for PQC | SAT-36257 | Eliminates dominant handshake penalty |
| Prioritize SAT-38970 / matrix approach | SAT-36257 | Know which paths negotiate PQC before build |
| Elevate CANDLEPIN-1162 | CANDLEPIN / SAT-43401 | CDN trust for connected Satellite |

---

## Document map (use together)

| Read this | For |
|-----------|-----|
| [satellite-pqc-performance-impact.md](./satellite-pqc-performance-impact.md) | **Unified** performance impact (start here) |
| This file | Alignment between analyses |
| [pqc-tls-handshake-research.md](./pqc-tls-handshake-research.md) | Quantitative TLS table |
| [pqc-performance-mitigations.md](./pqc-performance-mitigations.md) | initcwnd, reuse, rollout |
| [candlepin-pqc-performance-impact.md](./candlepin-pqc-performance-impact.md) | CP-specific (signing vs handshake) |
| [pqc-performance-slos.md](./pqc-performance-slos.md) | Measurable targets |
| [pqc-phased-perf-test-plan.md](./pqc-phased-perf-test-plan.md) | Execution phases |
| [pqc-registration-pr-stack-impact.md](./pqc-registration-pr-stack-impact.md) | Registration PRs × PQC tiers + R0–R4 test matrix |

---

## Research sources (from parallel analysis)

- ML-KEM vs ML-DSA performance analysis (internal)
- Amazon study: PQC TLS impact on bulk transfer
- Cloudflare: PQC certificate chain overhead (~9KB turning point)
- NIST / layered PQC TLS analyses
- PQC TLS benchmarks on realistic networks
- OpenSSL 3.5 PQC lab (RHEL 9.6)
