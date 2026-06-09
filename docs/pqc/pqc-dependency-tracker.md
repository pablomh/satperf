# PQC Dependency Tracker (RAG)

Dependencies for [SAT-35690](https://redhat.atlassian.net/browse/SAT-35690) / [PSASTRAT-109](https://redhat.atlassian.net/browse/PSASTRAT-109).  
**Target:** May 2027 GA with RHEL 10.4 PQC crypto policy.

**RAG legend:** 🟢 On track | 🟡 At risk | 🔴 Blocked / unknown

---

## Executive RAG

| Dependency | RAG | Impact if late | Owner (typical) |
|------------|-----|----------------|-----------------|
| RHEL 10.4 PQC crypto policy | 🟡 | Cannot meet SAT-35690 platform AC | RHEL crypto |
| Candlepin ML-DSA server | 🟢 | Server ready; Satellite integration not | Candlepin |
| subscription-manager / client tools | 🟡 | No ML-DSA clients → no E2E | CCT |
| Akamai CDN PQC TLS | 🔴 | Connected sync/manifest broken | CDN/Platform |
| Subscription portal PQC | 🔴 | Manifest workflow broken | Hosted CP / IT |
| Ruby/JWT/OIDC in Satellite | 🟡 | Mixed security posture | RHEL + Satellite |
| OpenSSH ML-KEM (REX) | 🟡 | REX criterion fails | RHEL |
| Browser PQC for Web UI | 🔴 | UI access undefined | SAT-36257 open Q |

---

## Platform: RHEL

| Ticket | Summary | Status | Needed by | RAG | Notes |
|--------|---------|--------|-----------|-----|-------|
| RHEL 10.4 (policy) | PQC crypto policy in product | Planned (May 2027 align) | GA | 🟡 | SAT-35690 hard dependency |
| [RHELBU-3378](https://redhat.atlassian.net/browse/RHELBU-3378) | Hybrid key exchange in SSH | In Progress | SAT-43331 | 🟡 | ML-KEM for REX; crypto-policies prefer PQ in 10.1+ |
| [RHEL-125093](https://redhat.atlassian.net/browse/RHEL-125093) | PQC transition for Ruby | In Progress | Foreman OIDC/JWT | 🟡 | Blocks SAT-45428 |
| [IDM-1536](https://redhat.atlassian.net/browse/IDM-1536) | python-cryptography ML-DSA | In Progress | PKI tooling | 🟡 | Dogtag/PKI adjacent |
| [CRYPTO-13045](https://redhat.atlassian.net/browse/CRYPTO-13045) | GnuPG deprecation / Sequoia | In Progress | RPM signing ecosystem | 🟡 | Affects ML-DSA package trust chain narrative |
| [RHELBU-3071](https://redhat.atlassian.net/browse/RHELBU-3071) | Go ML-DSA in TLS | New | Go-based components | 🟡 | Per parallel program analysis |
| [PRODSECRM-160](https://redhat.atlassian.net/browse/PRODSECRM-160) | Java PQC delay | In Progress | Foreman↔CP TLS | 🟡 | Tomcat PQC until Java 29; drives SAT-38910 options |

---

## Candlepin & subscription

| Ticket | Summary | Status | Needed by | RAG | Notes |
|--------|---------|--------|-----------|-----|-------|
| [CANDLEPIN-1021](https://redhat.atlassian.net/browse/CANDLEPIN-1021) | PQC certificates epic | **Closed** | SAT-43394 | 🟢 | 34/34 children closed |
| [CANDLEPIN-1171](https://redhat.atlassian.net/browse/CANDLEPIN-1171) | PQC Certificates V2 | **Closed** | CP follow-on | 🟢 | Anonymous cert generator scheme-aware |
| [CANDLEPIN-1102](https://redhat.atlassian.net/browse/CANDLEPIN-1102) | PQC Testing and Automation | In Progress | CP QA | 🟡 | Automation — not load/perf |
| [CANDLEPIN-1162](https://redhat.atlassian.net/browse/CANDLEPIN-1162) | Ship Hosted PQC CA for Satellite | Backlog | SAT-43401 | 🔴 | CDN trust for connected Satellite |
| [CANDLEPIN-937](https://redhat.atlassian.net/browse/CANDLEPIN-937) | RHEL 10 & Tomcat 10 support | In Progress | CP on Satellite | 🟡 | Platform alignment |
| [CCT-1801](https://redhat.atlassian.net/browse/CCT-1801) | PQC default across client tools | In Progress | E2E clients | 🟡 | subman opt-in → default; depends Candlepin + Satellite |
| [CCT-1851](https://redhat.atlassian.net/browse/CCT-1851) | subscription-manager PQC support | (search) | Registration | 🟡 | Linked from program search |
| [CCT-1856](https://redhat.atlassian.net/browse/CCT-1856) | subscription-manager PQC by default | (search) | GA default | 🟡 | M2 milestone per CCT-1801 |

**CCT-1801 explicit deps:** Candlepin server changes, Satellite SAT-36257, Akamai, Infosec PQC CA, HCC/console.redhat.com TLS.

---

## Satellite internal (linked to SAT-35690)

| Ticket | Summary | Status | RAG | Blocks |
|--------|---------|--------|-----|--------|
| [SAT-42784](https://redhat.atlassian.net/browse/SAT-42784) | Satellite on RHEL 10 | Refinement | 🟡 | All PQC work |
| [SAT-36257](https://redhat.atlassian.net/browse/SAT-36257) | TLS 1.3 + PQC infra | Refinement | 🟡 | TLS metrics |
| [SAT-43394](https://redhat.atlassian.net/browse/SAT-43394) | Client reg + content PQC certs | New | 🔴 | Core E2E |
| [SAT-43398](https://redhat.atlassian.net/browse/SAT-43398) | Foreman↔CP spike (A1/A2/B) | New | 🔴 | **Blocks** SAT-38910 design |
| [SAT-38910](https://redhat.atlassian.net/browse/SAT-38910) | Foreman ↔ Candlepin | New | 🔴 | All CP API; depends SAT-43398 |
| [SAT-19326](https://redhat.atlassian.net/browse/SAT-19326) | TLS 1.3 for Candlepin | New | 🟡 | PQC KEX on CP TLS path; [SAT-42162](https://redhat.atlassian.net/browse/SAT-42162) closed |
| [SAT-43401](https://redhat.atlassian.net/browse/SAT-43401) | CDN sync PQC | Refinement | 🟡 | Connected sites |
| [SAT-43331](https://redhat.atlassian.net/browse/SAT-43331) | PQC SSH REX | Refinement | 🟡 | REX metric |
| [SAT-45428](https://redhat.atlassian.net/browse/SAT-45428) | JWT/OIDC PQC readiness | New | 🟡 | Auth paths |
| [SAT-45455](https://redhat.atlassian.net/browse/SAT-45455) | PyJWT pulp_container | New | 🟡 | Container registry auth |
| [SAT-45197](https://redhat.atlassian.net/browse/SAT-45197) | pulp_container >= 2.26.9 | New | 🟡 | Container content |
| [SAT-45354](https://redhat.atlassian.net/browse/SAT-45354) | openscap > 1.4.0-2 | New | 🟡 | Compliance scans |
| [SAT-45175](https://redhat.atlassian.net/browse/SAT-45175) | Replace sshkey with OpenSSH/libssh | New | 🟡 | REX/provisioning |
| [SAT-45177](https://redhat.atlassian.net/browse/SAT-45177) | Replace net-ssh with system SSH | New | 🔴 | PR #10626 regression risk |
| [SAT-45196](https://redhat.atlassian.net/browse/SAT-45196) | Replace net-scp with SFTP | New | 🟡 | REX/provisioning |
| [SAT-43402](https://redhat.atlassian.net/browse/SAT-43402) | Validate ML-DSA content workflows | New | 🟡 | Child of SAT-36256 |
| [SAT-36399](https://redhat.atlassian.net/browse/SAT-36399) | foreman-proxy crypto-policies | New | 🟡 | Child of SAT-36257 |
| [SAT-43405](https://redhat.atlassian.net/browse/SAT-43405) | RHEL 10 containers | Refinement | 🟡 | SAT-43406/407, Valkey, PG16, Ruby 3.3 |

---

## External services

| Dependency | Capability required | RAG | Evidence / ticket |
|------------|---------------------|-----|-------------------|
| **Akamai CDN** | PQC TLS + accept ML-DSA entitlement mTLS | 🔴 | SAT-43401, SAT-38912; external |
| **subscription.rhsm.redhat.com** | PQC TLS for manifest workflows | 🔴 | SAT-43401 open questions |
| **console.redhat.com / HCC** | PQC TLS for Insights paths | 🔴 | CCT-1801, SAT-38970 sub-tasks |
| **Red Hat Infosec** | PQC CA issuance | 🔴 | CCT-1801 |
| **CDN Authorizer** | Parse ML-DSA entitlement certs | 🔴 | CANDLEPIN-1021 |

---

## Portfolio / program

| Ticket | Summary | Status | RAG |
|--------|---------|--------|-----|
| [HATSTRAT-310](https://redhat.atlassian.net/browse/HATSTRAT-310) | Consolidated Portfolio Crypto 2026 | In Progress | 🟡 | Parent above PSASTRAT-109 |
| [PSASTRAT-109](https://redhat.atlassian.net/browse/PSASTRAT-109) | 2026 Red Hat Satellite | Planning | 🟡 — no portfolio-level metrics |
| [RHIN-2057](https://redhat.atlassian.net/browse/RHIN-2057) | Insights PQC | In Progress | 🟡 — linked sibling outcome |
| [PLMCORE-15555](https://redhat.atlassian.net/browse/PLMCORE-15555) | PQC implementation risk | In Progress | 🟡 — tracks program difficulties |

---

## Release gate checklist

Gate SAT-35690 GA only when:

| # | Gate | RAG must be |
|---|------|-------------|
| G1 | RHEL 10.4 PQC policy available in supported Satellite stream | 🟢 |
| G2 | SAT-38910 + SAT-43394 Done (incl. proxy + certguard) | 🟢 |
| G3 | CCT-1801 M2 (subman PQC default) available for managed clients | 🟢 |
| G4 | Akamai + portal PQC available in staging **and** prod | 🟢 |
| G5 | Perf SLOs (see pqc-performance-slos.md) met on RC; initcwnd tuning validated | 🟢 |
| G6 | JWT/Ruby path documented or waived (SAT-45428) | 🟢 or waived |
| G7 | CANDLEPIN-1162 Done or CDN trust path waived | 🟢 or waived |

---

## Review cadence

- **Weekly:** Update RAG column in this doc during SAT PQC standup
- **Monthly:** Sync with CCT (client tools), Candlepin, RHEL crypto on 🔴 items
- **Pre-RC:** Full gate review G1–G6

---

## Actions for PSASTRAT-109

1. Mirror this RAG table in portfolio outcome description
2. Add explicit 🔴 external deps (Akamai, portal) to executive status
3. Do not report "Candlepin complete" without Satellite integration 🟢
