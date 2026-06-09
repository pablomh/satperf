# Satellite PQC Performance Impact

**Unified performance analysis for Post-Quantum Cryptography (PQC) on Red Hat Satellite**

| | |
|---|---|
| **Program** | [SAT-35690](https://redhat.atlassian.net/browse/SAT-35690) — Support PQC |
| **Portfolio** | [HATSTRAT-310](https://redhat.atlassian.net/browse/HATSTRAT-310) → [PSASTRAT-109](https://redhat.atlassian.net/browse/PSASTRAT-109) |
| **Target** | May 2027 (RHEL 10.4 `DEFAULT:PQ` alignment) |
| **Audience** | Perf&Scale, Satellite engineering, program management |
| **Last updated** | May 2026 |

**satperf location:** `redhat-performance/satperf/docs/pqc/`

---

## Table of contents

1. [Executive summary](#1-executive-summary)
2. [Program context](#2-program-context)
3. [PQC cost model](#3-pqc-cost-model)
4. [Impact by connection path](#4-impact-by-connection-path)
5. [Impact by workload](#5-impact-by-workload)
6. [Registration performance PR stack](#6-registration-performance-pr-stack)
7. [Foreman ↔ Candlepin architecture](#7-foreman--candlepin-architecture)
8. [Candlepin performance](#8-candlepin-performance)
9. [Integration gaps](#9-integration-gaps)
10. [Mitigations (prioritized)](#10-mitigations-prioritized)
11. [Performance SLOs](#11-performance-slos)
12. [Test plan](#12-test-plan)
13. [Dependencies and risks](#13-dependencies-and-risks)
14. [Recommended actions](#14-recommended-actions)
15. [Appendices and deep-dive documents](#15-appendices-and-deep-dive-documents)

---

## 1. Executive summary

### What hurts performance under PQC

| Rank | Factor | Why |
|------|--------|-----|
| **1** | TLS handshake **size** (~12 KB ML-DSA chains vs ~3.5 KB classical) | TCP **initcwnd overflow** → **+1 RTT** per **new** handshake on paths with real RTT |
| **2** | **Connection churn** (no HTTP/TLS reuse) | Hundreds of internal Puma→Candlepin handshakes per registration × thousands of hosts |
| **3** | **Candlepin saturation** at registration storm | Pool exhaustion, long transactions, redundant API calls — worse when each op is slower |
| **4** | **Burst admission** without throttling | Hundreds of concurrent PQC TLS sessions queue at Apache with larger memory footprint |

### What does **not** drive the main regression

- **ML-DSA signing CPU** in Candlepin (~0.2 ms vs ~5 ms RSA) — signing is **faster**
- **Steady-state bulk download** after TLS is established (CDN, large RPMs) — typically **&lt;5%** TTLB impact
- **initcwnd on loopback** (Puma→Candlepin) — RTT ~50 µs; extra RTT is invisible

### Strategic conclusion

1. **CANDLEPIN-1021 closed ≠ Satellite PQC performance validated** — integration and load testing remain open.
2. The **registration performance PR program** (Katello/Foreman/smart-proxy) is one of the **most effective PQC mitigations** for registration-at-scale — but it does **not** fix the client-facing handshake tax.
3. **[SAT-43398](https://redhat.atlassian.net/browse/SAT-43398)** → **[SAT-38910](https://redhat.atlassian.net/browse/SAT-38910)** and **[SAT-19326](https://redhat.atlassian.net/browse/SAT-19326)** are **architectural** prerequisites for honest PQC measurement on the Foreman↔Candlepin path.
4. **[SAT-35690](https://redhat.atlassian.net/browse/SAT-35690)** needs **quantitative SLOs** (Section 11) and a **satperf benchmark suite** — "no degradation" is not measurable without them.

---

## 2. Program context

### Issue hierarchy

```
HATSTRAT-310 (Portfolio Crypto 2026) [In Progress]
  └── PSASTRAT-109 (2026 - Red Hat Satellite) [Planning]
        ├── SAT-35690 (Support PQC) [To Do] — main delivery outcome
        ├── SAT-36256 (ML-DSA signed content) [Refinement]
        ├── SAT-36257 (TLS 1.3 + PQC infrastructure) [Refinement]
        └── CANDLEPIN-1021 (PQC certificates) [Closed] — 34/34 children
```

### SAT-35690 feature pillars (performance-relevant)

| Feature | Key | Perf focus |
|---------|-----|------------|
| TLS 1.3 + PQC infrastructure | [SAT-36257](https://redhat.atlassian.net/browse/SAT-36257) | Apache, Capsule, initcwnd, dual certs |
| Client registration + content | [SAT-43394](https://redhat.atlassian.net/browse/SAT-43394) | Registration storm, mTLS, pulp-certguard |
| CDN sync | [SAT-43401](https://redhat.atlassian.net/browse/SAT-43401) | Connect phase + Akamai dependency |
| ML-DSA signed content | [SAT-36256](https://redhat.atlassian.net/browse/SAT-36256) | Publish/promote CPU |
| PQC SSH / REX | [SAT-43331](https://redhat.atlassian.net/browse/SAT-43331) | ML-KEM KEX (generally favorable) |
| RHEL 10 containers | [SAT-43405](https://redhat.atlassian.net/browse/SAT-43405) | Startup policy cost |

Full issue export: [pqc-issue-hierarchy.csv](./pqc-issue-hierarchy.csv) (140+ rows).

### Candlepin vs Satellite status

| | Candlepin | Satellite |
|--|-----------|-----------|
| ML-DSA cert generation | **Done** (CANDLEPIN-1021) | Integration **New** (SAT-43394) |
| Perf baselines | **Not published** | **Not started** (proposed SLOs below) |
| Hosted PQC CA | [CANDLEPIN-1162](https://redhat.atlassian.net/browse/CANDLEPIN-1162) **Backlog** | Blocks connected CDN |

---

## 3. PQC cost model

Two independent cost drivers apply to different paths.

| Driver | Mechanism | Dominant paths |
|--------|-----------|----------------|
| **Data size** | ML-DSA server cert chain ~**12 KB** vs ~3.5 KB → initcwnd overflow → **+1 RTT** per new handshake | Client→Satellite/Capsule; Capsule→Satellite |
| **CPU + memory** | Parse larger chains, ML-KEM, bigger buffers per handshake | All TLS; **worst** when handshake **count** is high (Puma→CP without pooling) |

### Quantitative reference (classical vs PQC)

| Metric | Classical | PQC (ML-DSA-65 / ML-KEM-768) |
|--------|-----------|------------------------------|
| TLS cert chain (handshake) | ~3.5 KB | ~12 KB (**3–5×**) |
| Signature size | 64 B (ECDSA) | 3,309 B (**~50×**) |
| Signing speed | ~5 ms (RSA-2048) | **~0.2 ms** (faster) |
| Handshake latency (good net) | Baseline | **+~15%** |
| Handshake latency (constrained) | Baseline | **+~32%** |
| Handshake latency (lossy) | Baseline | **6–8×** (remote Capsules) |

**initcwnd:** Default ~10 × MSS (~14.6 KB). ML-DSA auth data often exceeds this → extra RTT. Mitigation: **initcwnd 30–40**. Cloudflare cites ~9 KB as a practical turning point.

---

## 4. Impact by connection path

| Path | RTT | initcwnd overflow? | Primary PQC cost |
|------|-----|-------------------|------------------|
| **Client → Satellite/Capsule** | 1–100+ ms | **Yes** | Data size (+1 RTT) |
| **Capsule → Satellite** (script, RHSM forward) | 1–50 ms | **Yes** | Data size |
| **Satellite → CDN** | 10–200 ms | **Yes** (connect phase) | Data size; bulk amortizes |
| **Puma → Candlepin** (loopback) | ~0.05 ms | **No** | CPU/memory per handshake |
| **Puma → Pulp** (local) | ~0.05 ms | **No** | CPU |
| **Foreman ↔ Candlepin** (functional) | Loopback | N/A for initcwnd | **Java/Tomcat PQC** until Java 29 — see Section 7 |

### Registration multiplier (worst case)

- **10,000 hosts × +1 RTT** at 20 ms ≈ **+200 s** cumulative (order-of-magnitude; session reuse reduces this).
- One **unavoidable** client handshake per host remains after all internal optimisations.

### What is **not** uniformly affected

| Path / protocol | initcwnd + large server cert? |
|-----------------|--------------------------------|
| Client ↔ Satellite/Capsule HTTPS (new conn) | **Yes** |
| SSH (REX) | Different (ML-KEM KEX, not 12 KB TLS chain) |
| Foreman ↔ Candlepin loopback | **No** (initcwnd); separate transport issue |
| HTTP keep-alive / TLS session resumption | **No** (for that session) |

---

## 5. Impact by workload

### 5.1 Host registration (highest priority)

**Flow:**

```text
Client ──TLS──► Apache ──► Puma ──TLS──► Candlepin
  (1 handshake/host)      (many CP calls/host without optimisations)
```

| Risk | Severity | Mitigation |
|------|----------|------------|
| Client handshake storm | **High** | initcwnd, keep-alive, admission control |
| Internal CP call multiplication | **High** | Katello caches + static compliance (#11731) |
| Internal TLS handshake storm | **Extreme** | [#11726](https://github.com/Katello/katello/pull/11726) or [#11754](https://github.com/Katello/katello/pull/11754) |
| CP `POST /consumers` saturation | **High** | CP pool tuning, async compliance (future) |
| Capsule script fetch storm | **Very high** (capsule) | [smart-proxy#935](https://github.com/theforeman/smart-proxy/pull/935) |

**SLOs:** SLO-1, SLO-2, SLO-3 (Section 11).

### 5.2 Content serving (dnf/yum)

- Handshake paid **once per connection**; bulk RPM transfer dominates.
- **Moderate** overall impact; still test first-package-on-new-connection patterns.
- **Gaps:** HTTP proxy mTLS, pulp-certguard ML-DSA — no dedicated SAT tickets yet.

**SLOs:** Functional + latency under SAT-43394; pulp-certguard CPU (P2-6 in test plan).

### 5.3 CDN / repository sync

- Penalty on **each new connection** to Akamai; keep-alive amortises over GB transferred.
- **External blocker:** Akamai PQC TLS + ML-DSA entitlement acceptance ([CANDLEPIN-1162](https://redhat.atlassian.net/browse/CANDLEPIN-1162)).
- **Gap:** `cdn.rb` still uses RestClient per request — extend persistent HTTP pattern from #11754.

**SLOs:** SLO-4, SLO-5.

### 5.4 Manifest import/export

- ML-DSA signature validation over full manifest payload.
- Larger artifacts ([CANDLEPIN-1173](https://redhat.atlassian.net/browse/CANDLEPIN-1173)).

**SLOs:** SLO-11, CP-SLO-4/5.

### 5.5 Remote execution (REX)

- ML-KEM SSH KEX generally **favorable** vs classical ([RHELBU-3378](https://redhat.atlassian.net/browse/RHELBU-3378)).
- Watch [SAT-45177](https://redhat.atlassian.net/browse/SAT-45177) (net-ssh → system SSH) for regression risk.

**SLOs:** SLO-9.

---

## 6. Registration performance PR stack

The Foreman/Katello registration optimisation program (April–May 2026) is a **major PQC mitigation**. Classical results: pass rate **69% → 98.5%** at 152–1520 concurrent hosts on 16-CPU Satellite.

### Mitigation layers

```text
Layer A  Client → Apache           initcwnd, keep-alive, foremanctl#495
Layer B  Capsule → Satellite       smart-proxy#935; RHSM pool (gap)
Layer C  Puma → Candlepin          katello#11726 OR #11754 (ship one)
Layer D  CP call count             katello#11731, #11696, #11694
Layer E  Satellite CPU/DB          foreman#10979/#10980, katello#11701
Layer F  CP create path            pool 20→100+, async compliance (future)
```

**After C–E:** remaining PQC risk is **Layer A** (client handshake) and **Layer B** (capsule topologies).

### PR summary and PQC tier

| PR / measure | PQC tier | Primary mechanism |
|--------------|----------|-------------------|
| [#11754](https://github.com/Katello/katello/pull/11754) or [#11726](https://github.com/Katello/katello/pull/11726) | **EXTREME** | 692K → 2.7K TCP opens (2k hosts, classical A/B) |
| [#11731](https://github.com/Katello/katello/pull/11731) static compliance | **EXTREME** | 13 CP calls → **0** |
| [smart-proxy#935](https://github.com/theforeman/smart-proxy/pull/935) | **VERY HIGH** | Capsule→Satellite initcwnd path |
| [#11696](https://github.com/Katello/katello/pull/11696) status cache | **HIGH** | ~24 CP calls → 1 |
| [foremanctl#495](https://github.com/theforeman/foremanctl/pull/495) | **HIGH** | Burst admission; re-tune ×5 under PQC |
| [#10948](https://github.com/theforeman/foreman/pull/10948) TCP reset recovery | **HIGH** | Avoid retry handshake storms |
| [#10979](https://github.com/theforeman/foreman/pull/10979)/[#10980](https://github.com/theforeman/foreman/pull/10980) + [#11701](https://github.com/Katello/katello/pull/11701) | **CRITICAL (indirect)** | −83% AR time → timeout headroom |
| [#11694](https://github.com/Katello/katello/pull/11694) | **MODERATE** | Avoid re-fetching larger cert payloads |

**Deep dive:** [pqc-registration-pr-stack-impact.md](./pqc-registration-pr-stack-impact.md)  
**Classical measured results:** `satperf/.claude/worktrees/observability/docs/registration-performance-improvements.md`

### PQC test prerequisite

Do **not** judge PQC registration SLOs on an unpatched Satellite. Use comparison matrix **R1 vs R3** (Section 12).

---

## 7. Foreman ↔ Candlepin architecture

These JIRAs decide **whether** internal registration traffic uses PQC TLS — separate from the PR stack that optimises **volume and reuse**.

### [SAT-43398](https://redhat.atlassian.net/browse/SAT-43398) — Investigation spike (New)

**Informs** [SAT-38910](https://redhat.atlassian.net/browse/SAT-38910). Evaluates:

| Path | PQC perf effect |
|------|-----------------|
| **A1** Container networking | TBD in spike |
| **A2** Unix socket (no TLS) | **Eliminates** Puma→CP TLS handshake tax |
| **B** FFM / OpenSSL in Tomcat | **Keeps** TLS; makes #11726/#11754 **mandatory** |

Success criteria include **performance implications** — feed satperf R1/R3 data into spike recommendation.

### [SAT-38910](https://redhat.atlassian.net/browse/SAT-38910) — Implementation epic (New)

Implements SAT-43398 recommendation. Blocks on [PRODSECRM-160](https://redhat.atlassian.net/browse/PRODSECRM-160) (Java PQC in Tomcat until Java 29).

**Perf acceptance (proposed):** p95 Foreman→Candlepin ≤ **125%** baseline under `DEFAULT:PQ`.

### [SAT-19326](https://redhat.atlassian.net/browse/SAT-19326) — TLS 1.3 on Candlepin (New)

Historical FR for TLS 1.3 on port 23443. Closed spike: [SAT-42162](https://redhat.atlassian.net/browse/SAT-42162).

**PQC link:** ML-KEM requires **TLS 1.3**. Without it, host `DEFAULT:PQ` may not negotiate PQC on Foreman→CP even with connection pooling.

---

## 8. Candlepin performance

### What changed (CANDLEPIN-1021 — closed)

PKI refactor, ML-DSA generators, scheme negotiation, manifest sigs, tomcat-native dual certs ([CANDLEPIN-1152](https://redhat.atlassian.net/browse/CANDLEPIN-1152)).

### Performance mechanisms

| Mechanism | Impact |
|-----------|--------|
| ML-DSA **signing** | **Faster** than RSA — not the regression driver |
| ML-DSA **cert size** | Larger DB rows + TLS chain bloat |
| Scheme negotiation ([CANDLEPIN-1078](https://redhat.atlassian.net/browse/CANDLEPIN-1078)) | Extra CPU per consumer op |
| Dual TLS certs (tomcat-native) | Larger handshakes, more memory per connection |
| Registration storm | **Every** new consumer hits CP — Katello call reduction critical |

### CP bottlenecks (post-Katello-stack)

| Issue | Severity | Fix |
|-------|----------|-----|
| Connection pool max **20** vs 150 Tomcat threads | **Critical** | Increase to 100+ (tuned) |
| Long `@Transactional` on create | **Critical** | Async compliance (medium effort) |
| Pessimistic pool locking | **Severe** | Fine-grained locking |
| Duplicate compliance calculation | **Moderate** | Remove duplicate call in `ConsumerResource` |

### Proposed Candlepin SLOs (CP-SLO-1–9)

| ID | Operation | Target vs baseline |
|----|-----------|-------------------|
| CP-SLO-1 | Single consumer register | p95 ≤ **125%** |
| CP-SLO-2 | Registration storm throughput | ≥ **90%** |
| CP-SLO-4/5 | Manifest import/export | p95 ≤ **115%** |
| CP-SLO-7 | mTLS handshake | p95 ≤ **130%** |

**Deep dive:** [candlepin-pqc-performance-impact.md](./candlepin-pqc-performance-impact.md)

---

## 9. Integration gaps

| Integration point | CP status | Satellite | Severity |
|-------------------|-----------|-------------|----------|
| ML-DSA consumer/entitlement certs | Done | [SAT-43394](https://redhat.atlassian.net/browse/SAT-43394) New | **Critical** |
| Foreman↔CP transport | Partial (1152) | [SAT-43398](https://redhat.atlassian.net/browse/SAT-43398) → [SAT-38910](https://redhat.atlassian.net/browse/SAT-38910) | **Critical** |
| Candlepin TLS 1.3 | [SAT-42162](https://redhat.atlassian.net/browse/SAT-42162) closed | [SAT-19326](https://redhat.atlassian.net/browse/SAT-19326) New | **High** |
| HTTP proxy mTLS ML-DSA | N/A | **No ticket** | **Critical** |
| pulp-certguard ML-DSA | N/A | **No ticket** | **Critical** |
| CDN / Akamai PQC | External | [SAT-43401](https://redhat.atlassian.net/browse/SAT-43401) | **High** (+ [CANDLEPIN-1162](https://redhat.atlassian.net/browse/CANDLEPIN-1162)) |
| subscription-manager | [CCT-1801](https://redhat.atlassian.net/browse/CCT-1801) | Cross-team | **High** |
| Registration perf PR stack | N/A | In review (open PRs) | **Release dependency** for SLO-2 |

**Deep dive:** [pqc-integration-gaps.md](./pqc-integration-gaps.md)

---

## 10. Mitigations (prioritized)

| Priority | Mitigation | Owner / ticket |
|----------|------------|----------------|
| **1** | Tune **initcwnd 30–40** on Satellite/Capsule | SAT-36257; installer story needed |
| **2** | Merge **registration PR stack** (Layers C–E) | Katello/Foreman/smart-proxy PRs |
| **3** | **Connection pooling** Puma→CP (#11726 or #11754) | Katello |
| **4** | Complete **SAT-43398** spike → implement **SAT-38910** | Satellite arch |
| **5** | **foremanctl#495** admission control; validate ×5 → ×3 under PQC | foremanctl |
| **6** | **smart-proxy#935** on all capsule registration paths | Smart Proxy |
| **7** | **CP connection pool** 20 → 100+ | Candlepin |
| **8** | TLS **keep-alive**, session tickets, HTTP/2 where applicable | SAT-43401, SAT-36257 — detail: [pqc-performance-mitigations.md §2](./pqc-performance-mitigations.md#2-maximize-tls-connection-reuse) |
| **9** | **Phased rollout:** `DEFAULT:PQ` (KEX) before full ML-DSA certs | SAT-36255 |
| **10** | Short cert chains; target auth data &lt;9 KB | SAT-36257, SAT-43393 |

**Gaps still open after mitigations:**

- Client→Satellite: **one handshake/host** (initcwnd)
- Smart-proxy RHSM forward: no connection pool yet
- CDN `cdn.rb`: RestClient per request
- Dedicated **PQC perf benchmark** JIRA under SAT-35690

**Deep dive:** [pqc-performance-mitigations.md](./pqc-performance-mitigations.md)

---

## 11. Performance SLOs

Proposed targets for [SAT-35690](https://redhat.atlassian.net/browse/SAT-35690) — calibrate on first RHEL 10.4 PQC lab run.

| ID | Workflow | Metric | Target (PQC vs baseline) |
|----|----------|--------|--------------------------|
| **SLO-1** | Host registration | p95 end-to-end time per host | ≤ **110%** |
| **SLO-2** | Registration storm | Successful reg/min at fixed concurrency | ≥ **95%** |
| **SLO-3** | CP cert issuance | p95 identity+entitlement generation | ≤ **125%** |
| **SLO-4** | CDN repo sync | p95 sync duration (large repo) | ≤ **115%** |
| **SLO-5** | CDN throughput | Effective GB/hour | ≥ **90%** |
| **SLO-6** | Content publish | p95 publish duration | ≤ **110%** |
| **SLO-7** | Content promote | p95 promote duration | ≤ **110%** |
| **SLO-8** | Capsule sync | p95 sync from Satellite | ≤ **115%** |
| **SLO-9** | REX | p95 per-host latency (100-host job) | ≤ **120%** |
| **SLO-10** | TLS handshake | p95 external probe | ≤ **130%** |
| **SLO-11** | Manifest import | p95 wall time (ML-DSA manifest) | ≤ **115%** |
| **SLO-12** | API availability | Error rate under load | ≤ baseline + **0.5%** |
| **SLO-13** | CPU saturation | Peak CPU during SLO-2/4/9 | ≤ **125%** peak |
| **SLO-14** | Memory | Peak RSS httpd/tomcat/pulpcore | ≤ **115%** |

**Prerequisite for SLO-1/2:** Full registration PR stack applied ([Section 6](#6-registration-performance-pr-stack)).

**Deep dive:** [pqc-performance-slos.md](./pqc-performance-slos.md)

---

## 12. Test plan

### Phases (aligned with SAT-36255)

| Phase | Focus | Crypto policy | Key gate |
|-------|--------|---------------|----------|
| **0** | Platform baseline | `DEFAULT` | Install + classical metrics |
| **1** | PQC KEX only | `DEFAULT:PQ` | SLO-10; SAT-38970 matrix |
| **2** | ML-DSA certs + registration | Full PQC | **SLO-1/2/3** on **R3** matrix |
| **3** | CDN + ML-DSA content | Full PQC | SLO-4–8 |
| **4** | REX PQC SSH | Full PQC | SLO-9 |
| **5** | GA regression | Production-like | All SLO-1–12, 2× weekly green |

### Registration comparison matrix (Phase 2)

| Run | Policy | Patch stack | Purpose |
|-----|--------|-------------|---------|
| **R0** | `DEFAULT` | None | Classical baseline |
| **R1** | `DEFAULT` | Full reg stack | Patched classical |
| **R2** | `DEFAULT:PQ` | None | Raw PQC regression (diagnostic) |
| **R3** | `DEFAULT:PQ` | Full reg stack | **Target gate** for SAT-35690 |
| **R4** | `DEFAULT:PQ` | Full + foremanctl#495 | Admission control under PQC |

**Exit gate (Phase 2):** R3 pass rate within **95%** of R1 at same concurrency; SAT-38910 Done; SAT-43398 recommendation documented.

### satperf variables (proposed)

```yaml
pqc_enabled: false
pqc_crypto_policy: DEFAULT:PQ
pqc_phase: 0
```

**Deep dive:** [pqc-phased-perf-test-plan.md](./pqc-phased-perf-test-plan.md)

---

## 13. Dependencies and risks

### Dependency RAG (summary)

| Dependency | RAG | Impact if late |
|------------|-----|----------------|
| RHEL 10.4 PQC crypto policy | 🟡 | Cannot meet SAT-35690 platform AC |
| Candlepin ML-DSA (CANDLEPIN-1021) | 🟢 | Server ready; Satellite integration not |
| [SAT-43398](https://redhat.atlassian.net/browse/SAT-43398) / [SAT-38910](https://redhat.atlassian.net/browse/SAT-38910) | 🔴 | Foreman↔CP broken or unmeasured under PQC |
| Akamai / portal PQC | 🔴 | Connected sync/manifest broken |
| CCT / subscription-manager | 🟡 | No ML-DSA E2E clients |
| Registration perf PR merge | 🟡 | False PQC regression on SLO-2 |

### Risk register

| Risk | Severity | Mitigation |
|------|----------|------------|
| TLS handshake at scale (initcwnd) | **High** | initcwnd 30–40; keep-alive |
| Registration without PR stack | **High** | Merge Layers C–E before PQC SLO gate |
| Foreman↔CP architecture undecided | **High** | SAT-43398 → SAT-38910 |
| net-ssh replacement ([SAT-45177](https://redhat.atlassian.net/browse/SAT-45177)) | **High** | Incremental rollout; image-based prov tests |
| Lossy network / remote Capsule | **Medium** | Mandatory in Phase 5; hybrid `DEFAULT:PQ` first |
| No dedicated perf benchmark card | **Medium** | satperf phased plan + SLOs |
| [CANDLEPIN-1162](https://redhat.atlassian.net/browse/CANDLEPIN-1162) hosted CA | **Medium** | Prioritize for SAT-43401 |

**Deep dive:** [pqc-dependency-tracker.md](./pqc-dependency-tracker.md)

---

## 14. Recommended actions

### Engineering (immediate)

1. Merge registration performance PR stack; treat as **SAT-35690 dependency** for registration SLOs.
2. Ship **one** of [#11726](https://github.com/Katello/katello/pull/11726) / [#11754](https://github.com/Katello/katello/pull/11754).
3. Complete **SAT-43398** with perf data from satperf R1/R3 runs.
4. Run Phase 0 baseline + Phase 2 **R3** matrix on RHEL 10.4 lab.

### JIRA (proposed)

| Proposed card | Parent | Rationale |
|---------------|--------|-----------|
| PQC performance benchmark suite | SAT-35690 | No explicit perf card today |
| Registration PR stack as PQC dependency | SAT-35690 | Honest SLO-2 gate |
| Installer: TCP initcwnd tuning for PQC | SAT-36257 | Dominant client-path mitigation |
| HTTP proxy + pulp-certguard ML-DSA | SAT-43394 | Critical integration gaps |
| Elevate CANDLEPIN-1162 | SAT-43401 | CDN trust |

### Program / reporting

- Do not report **CANDLEPIN-1021 closed** as "PQC performance validated."
- Report **SAT-35690** against SLO table (Section 11), not qualitative "no degradation" alone.
- Use [SAT-38970](https://redhat.atlassian.net/browse/SAT-38970) as **test matrix evidence**, not ~70 delivery stories ([sat-38970-restructure-proposal.md](./sat-38970-restructure-proposal.md)).

---

## 15. Appendices and deep-dive documents

This file is the **unified entry point**. Topic-specific documents remain for maintenance and detail:

| Document | Contents |
|----------|----------|
| [pqc-tls-handshake-research.md](./pqc-tls-handshake-research.md) | Quantitative TLS table, references |
| [pqc-initcwnd-impact-by-path.md](./pqc-initcwnd-impact-by-path.md) | Path-by-path initcwnd FAQ |
| [pqc-registration-pr-stack-impact.md](./pqc-registration-pr-stack-impact.md) | Per-PR PQC tiers, apply_prs, JIRA § |
| [candlepin-pqc-performance-impact.md](./candlepin-pqc-performance-impact.md) | CP-SLO-1–9, mechanisms |
| [pqc-integration-gaps.md](./pqc-integration-gaps.md) | Full gap matrix + recommendations |
| [pqc-performance-slos.md](./pqc-performance-slos.md) | SLO playbook mapping, GA checklist |
| [pqc-phased-perf-test-plan.md](./pqc-phased-perf-test-plan.md) | Phase 0–5 detail, mermaid |
| [pqc-performance-mitigations.md](./pqc-performance-mitigations.md) | Mitigation detail + SSH/gems |
| [pqc-dependency-tracker.md](./pqc-dependency-tracker.md) | Full RAG tables |
| [pqc-analysis-reconciliation.md](./pqc-analysis-reconciliation.md) | Investigation merge index |
| [pqc-issue-hierarchy.csv](./pqc-issue-hierarchy.csv) | Jira export |
| [pqc-tls-component-matrix.csv](./pqc-tls-component-matrix.csv) | SAT-38970 component matrix |
| [sat-38970-restructure-proposal.md](./sat-38970-restructure-proposal.md) | JIRA consolidation proposal |

### Registration program (classical benchmarks)

- `satperf/.claude/worktrees/observability/docs/registration-performance-improvements.md`
- `satperf/.claude/worktrees/observability/docs/registration-performance-report.md`

---

*Document maintained by Perf&Scale as part of the SAT-35690 investigation. For updates, extend this file first, then sync topic-specific children if needed.*
