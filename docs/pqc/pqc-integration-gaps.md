# PQC Integration Gaps: Candlepin Closed vs Satellite Open

> **Unified doc:** [satellite-pqc-performance-impact.md](./satellite-pqc-performance-impact.md) §9.

## Summary

[CANDLEPIN-1021](https://redhat.atlassian.net/browse/CANDLEPIN-1021) is **Closed** with **34/34** child issues closed. Candlepin can generate **legacy and pure ML-DSA** x.509 certificates (epic AC).

Satellite end-to-end integration is **not validated**. Multiple Satellite epics remain **New** while server-side Candlepin work is marked done.

**Risk:** Compliance and performance sign-off on "PQC-ready Satellite" cannot rely on Candlepin epic closure alone.

---

## Gap matrix

| Integration point | CANDLEPIN-1021 requirement | Candlepin status | Satellite ticket | Satellite status | Gap severity |
|-------------------|---------------------------|------------------|------------------|------------------|--------------|
| ML-DSA consumer/identity certs | Generate for requesting clients | Done (generators, scheme negotiation) | [SAT-43394](https://redhat.atlassian.net/browse/SAT-43394) | **New** | **Critical** |
| ML-DSA entitlement certs | Generate + validate | Done | SAT-43394 | **New** | **Critical** |
| Foreman ↔ Candlepin TLS | N/A (Satellite internal) | tomcat-native path (CANDLEPIN-1152) | [SAT-38910](https://redhat.atlassian.net/browse/SAT-38910) | **New** | **Critical** |
| Foreman ↔ Candlepin investigation | Spike for solution | Partial (1133, 1152 closed) | [SAT-43398](https://redhat.atlassian.net/browse/SAT-43398) | **New** | **Critical** — **informs** SAT-38910 (not duplicate) |
| Candlepin TLS 1.3 | TLS 1.3 on CP listener | [SAT-42162](https://redhat.atlassian.net/browse/SAT-42162) spike closed | [SAT-19326](https://redhat.atlassian.net/browse/SAT-19326) | **New** | **High** — prerequisite for PQC on CP TLS path |
| HTTP proxy mTLS (ML-DSA) | Sat HTTP proxy handles mTLS | **No dedicated SAT ticket** | Implied under SAT-43394 / SAT-36257 | Not explicit | **Critical** |
| pulp-certguard ML-DSA | Verify/parse ML-DSA client certs | **No dedicated SAT ticket** | Implied under SAT-43394 | Not explicit | **Critical** |
| Apache dual certs (RSA+ML-DSA) | N/A | N/A | [SAT-36257](https://redhat.atlassian.net/browse/SAT-36257) | Refinement | **High** |
| Client registration TLS (PQC KEX) | N/A | N/A | SAT-43394, SAT-38970 matrix | New / matrix | **High** |
| Manifest import ML-DSA | Import/export signatures | Done (1080, 1079) | [SAT-38909](https://redhat.atlassian.net/browse/SAT-38909) | **New** | **High** |
| CDN sync mTLS ML-DSA | CDN accepts entitlement certs | External + Pulp | [SAT-38912](https://redhat.atlassian.net/browse/SAT-38912) | **New** | **High** |
| subscription-manager client | Parse ML-DSA certs | CCT team | [CCT-1801](https://redhat.atlassian.net/browse/CCT-1801) | In Progress | **High** (cross-team) |
| Smart proxy PQC | Proxy TLS to clients | N/A | [SAT-38907](https://redhat.atlassian.net/browse/SAT-38907) | **New** | **High** |
| Container crypto-policies | Services respect host policy | N/A | [SAT-38913](https://redhat.atlassian.net/browse/SAT-38913) | **New** | **Medium** |
| Custom PQC certificates | Admin-provided certs | N/A | [SAT-43393](https://redhat.atlassian.net/browse/SAT-43393) | **New** | **Medium** |

---

## Detailed gap analysis

### 1. Foreman ↔ Candlepin — [SAT-43398](https://redhat.atlassian.net/browse/SAT-43398) → [SAT-38910](https://redhat.atlassian.net/browse/SAT-38910)

**Candlepin side:** Dual ML-DSA+RSA Tomcat/native listener validated ([CANDLEPIN-1133](https://redhat.atlassian.net/browse/CANDLEPIN-1133), [CANDLEPIN-1152](https://redhat.atlassian.net/browse/CANDLEPIN-1152)).

**Satellite side:**

| Ticket | Role | Status |
|--------|------|--------|
| [SAT-43398](https://redhat.atlassian.net/browse/SAT-43398) | **Spike** — compare Path A1 (container net), A2 (unix socket), B (FFM/OpenSSL); document **performance implications** | **New** |
| [SAT-38910](https://redhat.atlassian.net/browse/SAT-38910) | **Epic** — implement spike recommendation | **New** (informed by SAT-43398) |

Open constraints (both tickets):

- Tomcat 10 on RHEL 10 lacks TLS 1.3 + PQC until Java 29 ([PRODSECRM-160](https://redhat.atlassian.net/browse/PRODSECRM-160))
- Unix socket (A2) may **eliminate** internal TLS handshake cost; tomcat-native (B) makes Katello connection pooling **critical** under PQC

**Perf impact:** Every Katello→Candlepin call on the registration path (~48+ per host pre-optimisation). Registration PR stack ([pqc-registration-pr-stack-impact.md](./pqc-registration-pr-stack-impact.md)) reduces **call count** and enables **pooling**; SAT-43398/38910 decide **whether those calls use PQC TLS or bypass TLS**.

**Recommendation:**

- Complete **SAT-43398** with perf data from satperf R1/R3 + architecture options
- Add perf acceptance to **SAT-38910**: p95 Foreman→Candlepin latency ≤ 125% baseline under `DEFAULT:PQ`
- Block SAT-35690 GA on SAT-38910 Done (after SAT-43398 recommendation)

### 1b. Candlepin TLS 1.3 — [SAT-19326](https://redhat.atlassian.net/browse/SAT-19326)

**Historical FR:** TLS 1.3 on Candlepin (Satellite 6 / port 23443). Informed by closed spike [SAT-42162](https://redhat.atlassian.net/browse/SAT-42162).

**PQC link:** PQC hybrid KEX (ML-KEM) requires **TLS 1.3**. Without it, `DEFAULT:PQ` on the host does not deliver PQC on the Foreman→Candlepin TLS path even with [#11726](https://github.com/Katello/katello/pull/11726)/[#11754](https://github.com/Katello/katello/pull/11754).

**Jira links:** SAT-19326 blocks or relates to SAT-38910, SAT-38907, SAT-38913, SAT-35691.

**Recommendation:** Fold SAT-19326 closure criteria into SAT-36257; verify TLS 1.3 in Phase 1 probes ([pqc-phased-perf-test-plan.md](./pqc-phased-perf-test-plan.md) P1-1).

---

### 2. Client registration + content ([SAT-43394](https://redhat.atlassian.net/browse/SAT-43394))

**Perf mitigation (in flight):** Katello/Foreman registration performance PRs (SAT-43940, SAT-44043, SAT-45132, etc.) — see [pqc-registration-pr-stack-impact.md](./pqc-registration-pr-stack-impact.md). Treat as **release dependency** for PQC registration SLOs, not optional perf polish.

**Epic AC:** ML-DSA identity + entitlement certs; subman, dnf/yum consume content.

**Missing explicit children for:**

- **Satellite HTTP proxy** (CANDLEPIN-1021 bullet)
- **pulp-certguard** (CANDLEPIN-1021 bullet)

**Recommendation:** Create under SAT-43394:

| Proposed story | Validates |
|----------------|-----------|
| Satellite HTTP proxy: mTLS with ML-DSA consumer and entitlement certs | Every client content request |
| pulp-certguard: verify ML-DSA entitlement certificates | Repository auth path |
| Registration storm E2E with ML-DSA certs | SLO-1, SLO-2 |

Link to [CCT-1851](https://redhat.atlassian.net/browse/CCT-1851) / [CCT-1856](https://redhat.atlassian.net/browse/CCT-1856) for client tooling.

---

### 3. Manifest + CDN ([SAT-38909](https://redhat.atlassian.net/browse/SAT-38909), [SAT-38912](https://redhat.atlassian.net/browse/SAT-38912))

Candlepin manifest import/export with PQC signatures is implemented. Satellite must:

- Import manifests containing ML-DSA material ([SAT-43401](https://redhat.atlassian.net/browse/SAT-43401))
- Sync from Akamai with PQC TLS + mTLS

**External dependency:** Akamai PQC endpoints (see [pqc-dependency-tracker.md](./pqc-dependency-tracker.md)).

---

### 4. pulp-certguard and HTTP proxy (no ticket — **highest silent gap**)

CANDLEPIN-1021 explicitly lists Satellite components. No SAT ticket names them.

**Perf note:** pulp-certguard runs on **every authenticated content pull**; ML-DSA verify cost is multiplicative at scale.

**Recommendation:** Add stories under SAT-43394 immediately; tag `perf_relevance=high` in hierarchy CSV.

---

### 5. Phase 1 vs Phase 2 ([SAT-36255](https://redhat.atlassian.net/browse/SAT-36255) closed)

SAT-36255 defined:

- Phase 1: PQC KEX via PROFILE=SYSTEM
- Phase 2: ML-DSA certificates

Epic is **Closed** but Satellite feature pillars (36257, 43394) are not. Risk: Phase 1 assumed complete without published perf evidence.

**Recommendation:** Run Phase 1 perf baselines (SLO-10 only) before starting Phase 2 GA.

---

## Suggested priority order

1. SAT-38910 — Foreman/Candlepin path (blocks all subscription flows)
2. SAT-43394 + new pulp-certguard/proxy stories
3. SAT-36257 — infrastructure TLS + dual certs
4. SAT-38909 / SAT-38912 — connected CDN/manifest
5. SAT-43331 — REX SSH (depends RHELBU-3378)
6. SAT-36256 — ML-DSA content (partially independent)

---

## Definition of "Satellite PQC complete"

Satellite PQC should not be declared complete until:

- [ ] All rows in this gap matrix are **Done** or **Waived** with sign-off
- [ ] E2E: register host → entitlement → dnf install from CV over ML-DSA mTLS
- [ ] Perf SLOs in [pqc-performance-slos.md](./pqc-performance-slos.md) met
- [ ] CANDLEPIN-1021 referenced as **dependency satisfied**, not **delivery complete**
