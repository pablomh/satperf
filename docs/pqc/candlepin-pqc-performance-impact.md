# Candlepin (CP) PQC Performance Impact

> **Unified doc:** [satellite-pqc-performance-impact.md](./satellite-pqc-performance-impact.md) §8 — this file is the Candlepin deep dive.

Dedicated analysis for [CANDLEPIN-1021](https://redhat.atlassian.net/browse/CANDLEPIN-1021) and Satellite integration. The general investigation treated CP mainly as a **dependency** of SAT-35690; this document covers **Candlepin-internal** performance.

**Status:** CANDLEPIN-1021 epic **Closed** (34/34 children). **No published CP-specific perf baselines** were found in Jira — functional/spec work only.

---

## 1. What changes in Candlepin with PQC

| Area | Implementation (closed tickets) | Perf sensitivity |
|------|--------------------------------|------------------|
| **PKI stack** | Refactored `Signer`, `SignatureValidator`, `X509CertificateBuilder`, `CertificateAuthority` (1068–1072) | Every sign/verify path |
| **Multi-CA / schemes** | Config for multiple CA chains ([CANDLEPIN-1074](https://redhat.atlassian.net/browse/CANDLEPIN-1074)) | Startup validation + runtime chain selection |
| **KeyPairData** | Algorithm field + DB migration ([CANDLEPIN-1073](https://redhat.atlassian.net/browse/CANDLEPIN-1073)) | Upgrade/migration window; row size |
| **Cert generators** | Scheme-aware Identity, SCA, Entitlement, Ueber ([CANDLEPIN-1153–1156](https://redhat.atlassian.net/browse/CANDLEPIN-1153)) | Registration, refresh, manifest |
| **Scheme negotiation** | Status endpoint ([CANDLEPIN-1077](https://redhat.atlassian.net/browse/CANDLEPIN-1077)); consumer reconciliation ([CANDLEPIN-1078](https://redhat.atlassian.net/browse/CANDLEPIN-1078)) | Extra API work per consumer op |
| **Manifest** | Import/export signature create/validate ([CANDLEPIN-1079](https://redhat.atlassian.net/browse/CANDLEPIN-1079), [1080](https://redhat.atlassian.net/browse/CANDLEPIN-1080)) | Large manifest latency |
| **TLS termination** | tomcat-native + APR + dual RSA/ML-DSA ([CANDLEPIN-1152](https://redhat.atlassian.net/browse/CANDLEPIN-1152)); Tomcat 9.0.110+ dual cert spike ([CANDLEPIN-1133](https://redhat.atlassian.net/browse/CANDLEPIN-1133)) | Handshake + memory per connection |
| **Feature flag** | `candlepin.crypto.system.scheme.negotiation.enabled` ([CANDLEPIN-1169](https://redhat.atlassian.net/browse/CANDLEPIN-1169)) | Hosted vs Satellite rollout control |
| **Spec coverage** | Multi-scheme tests ([CANDLEPIN-1170](https://redhat.atlassian.net/browse/CANDLEPIN-1170)) | Functional, not load |

---

## 2. Performance mechanisms (CP-specific)

### 2.1 ML-DSA signing and verification (CPU-bound)

**Operations affected:**

- Consumer registration → identity + entitlement cert issuance
- Certificate regeneration (bulk refresh after maintenance)
- Entitlement refresh / content access token paths
- Manifest export signature creation
- Manifest import signature validation
- Ueber cert operations

**Expected impact (reconciled with TLS research):**

| Operation | Typical impact | Notes |
|-----------|----------------|-------|
| **ML-DSA signing** | **Faster than RSA** | Research: ~0.2 ms (ML-DSA-65) vs ~5 ms (RSA-2048) — **not the primary regression driver** |
| **ML-DSA verification** | ~Comparable to ECDSA | Negligible vs handshake |
| **Certificate size** | **Much larger** | ~30× public key, ~50× signature → DB/memory + **TLS chain bloat** |
| Pure ML-DSA (epic AC) | No hybrid double-sign | Avoids signing twice; size still drives network cost |

**Primary CP-related perf risk for Satellite is TLS handshake size** on paths that terminate TLS at Candlepin or present ML-DSA chains to clients — not CPU time spent signing in Candlepin during registration.

**Scale hotspots on Satellite:**

- Registration storms → Candlepin is on the critical path for **every** new consumer (via Foreman/Katello).
- Large manifests → import validates signatures over full manifest payload ([CANDLEPIN-1080](https://redhat.atlassian.net/browse/CANDLEPIN-1080)).

### 2.2 Scheme negotiation and reconciliation (latency per request)

[CANDLEPIN-1078](https://redhat.atlassian.net/browse/CANDLEPIN-1078) adds validation/reconciliation on:

- Consumer registration
- Consumer update
- Certificate regeneration

**Expected impact:**

- Additional CPU per request (compare client-reported OIDs vs server-supported schemes).
- Possible **extra round-trips** if client and server schemes diverge and reconciliation triggers regen.
- [CANDLEPIN-1077](https://redhat.atlassian.net/browse/CANDLEPIN-1077) status endpoint: clients may poll before/after registration — adds read load.

**Satellite note:** Every `subscription-manager register` hits this path through Katello → Candlepin.

### 2.3 Dual TLS certificates (tomcat-native / APR)

[CANDLEPIN-1152](https://redhat.atlassian.net/browse/CANDLEPIN-1152) moves dev/CI toward:

- **Java 25**
- **tomcat-native** with **APR listener**
- **Dual RSA + ML-DSA** server certificates for mTLS

**Expected impact:**

| Factor | Effect |
|--------|--------|
| Dual certs on connector | Larger handshake, cert chain processing, more memory per connection |
| tomcat-native vs pure Java TLS | Different performance profile; may improve or change native OpenSSL path — **needs measurement on Satellite** |
| Java 25 | JVM heap/GC behavior change vs Java 17 (current Satellite stream) — affects CP service sizing |

[CANDLEPIN-1133](https://redhat.atlassian.net/browse/CANDLEPIN-1133) verified Tomcat 9.0.110+ accepts dual ML-DSA+RSA CAs — **functional only**, no throughput numbers.

**Critical Satellite gap:** [SAT-38910](https://redhat.atlassian.net/browse/SAT-38910) states Tomcat 10 on RHEL 10 may not do TLS 1.3 PQC until Java 29 — Foreman↔Candlepin path may use **alternate termination** (unix socket, sidecar). That architectural choice **dominates** CP-visible latency more than ML-DSA sign cost.

### 2.4 Database and storage

[CANDLEPIN-1073](https://redhat.atlassian.net/browse/CANDLEPIN-1073):

- `KeyPairData` stores algorithm identifier
- In-place migration for existing rows

**Expected impact:**

- Slightly larger rows; negligible unless millions of key records
- **Migration duration** during Satellite upgrade (CP DB) — operational, not steady-state
- Mixed algorithm consumers during rollout → more complex queries if not indexed properly

### 2.5 Artifact size (network + I/O)

[CANDLEPIN-1173](https://redhat.atlassian.net/browse/CANDLEPIN-1173) (signature.json packaging):

- Larger consumer export zips
- Manifest import/export payloads

**Expected impact:**

- Longer transfer times for manifest import (SAT-43401)
- More disk I/O on CP for export/import temp files
- Not necessarily CPU-bound but affects **wall-clock** manifest workflows

### 2.6 Multi-scheme CA configuration

[CANDLEPIN-1074](https://redhat.atlassian.net/browse/CANDLEPIN-1074) — multiple CA chains at startup:

- Config validation at Candlepin start
- Runtime selection of chain by scheme

**Expected impact:**

- Longer **startup time** if many chains/certs loaded
- Slightly higher memory footprint (multiple CA material in memory)

---

## 3. Candlepin-specific SLOs (proposed)

Extend [pqc-performance-slos.md](./pqc-performance-slos.md) with CP-focused IDs:

| ID | CP operation | Metric | Target vs baseline | Jira evidence |
|----|--------------|--------|-------------------|---------------|
| CP-SLO-1 | Consumer register (single) | p95 CP wall time for cert issue | ≤ **125%** | 1078, 1153–1155 |
| CP-SLO-2 | Registration storm | CP requests/sec sustainable | ≥ **90%** baseline | Katello load → CP |
| CP-SLO-3 | Entitlement cert regen | p95 per consumer | ≤ **125%** | 1078 |
| CP-SLO-4 | Manifest import | p95 import (ML-DSA manifest) | ≤ **115%** | 1080 |
| CP-SLO-5 | Manifest export | p95 export + signature create | ≤ **115%** | 1079 |
| CP-SLO-6 | Status endpoint | p95 GET /status (scheme list) | ≤ **110%** | 1077 |
| CP-SLO-7 | mTLS handshake (CP connector) | p95 handshake ms | ≤ **130%** | 1152, 1133 |
| CP-SLO-8 | CP JVM | Peak heap + CPU under CP-SLO-2 | ≤ **125%** peak | Java 25 / tomcat-native |
| CP-SLO-9 | Candlepin startup | Time to ready after restart | ≤ **120%** | 1074 multi-CA |

**Measurement:** Isolate CP via:

- Direct Candlepin API/spec load (CP team's spec tests extended to perf)
- APM on `candlepin` service during satperf `concurrent_execution.yaml`
- PostgreSQL slow query log during registration storm

---

## 4. Hosted Candlepin vs Satellite Candlepin

| Dimension | Hosted | Satellite (embedded) |
|-----------|--------|----------------------|
| Rollout | [CANDLEPIN-1169](https://redhat.atlassian.net/browse/CANDLEPIN-1169) flag default **false** for non-manifest consumers | Satellite consumers are non-manifest → flag must be enabled for PQC |
| Load pattern | Many small consumers globally | Registration storms, manifest import, Capsule-related sync |
| TLS path | Akamai + hosted infra | SAT-38910 Foreman↔CP architecture TBD |
| Perf risk | Gradual opt-in | **Big-bang** when Satellite enables PQC for all registrations |

Satellite CP is **higher risk** for perf regressions because load is bursty and co-located with Foreman/Pulp on same host.

---

## 5. Gaps in current CP / program tracking

| Gap | Risk |
|-----|------|
| No CP perf epic under CANDLEPIN-1021 | Closed epic without perf sign-off |
| CANDLEPIN-1170 is functional spec tests only | No load/soak tests |
| SAT-38910 "performance implications undecided" | Biggest unknown for Satellite CP path |
| SLO-3 in main doc is brief | Needs CP-SLO-1–9 breakdown above |
| No baseline numbers published | Cannot verify "no degradation" today |

---

## 6. Recommended CP perf work items

| Action | Owner | Links to |
|--------|-------|----------|
| Run CP-SLO-1/2 on PQC branch with tomcat-native setup | Candlepin + PerfScale | CANDLEPIN-1152 stack |
| Benchmark manifest import/export ML-DSA vs RSA | Candlepin | 1079, 1080, SAT-43401 |
| Document Foreman→CP latency for each SAT-38910 architecture option | Satellite | SAT-38910 |
| Add CP metrics to Candlepin (Micrometer): cert issue timer, scheme reconciliation counter | Candlepin eng | New story |
| Gate Satellite PQC on CP-SLO-2 pass at 1k registrations/hour (example threshold TBD) | SAT-35690 | Outcome |

---

## 7. Mapping to parent JIRAs

| Parent | CP perf relevance |
|--------|-------------------|
| [CANDLEPIN-1021](https://redhat.atlassian.net/browse/CANDLEPIN-1021) | Implementation complete; **perf validation missing** |
| [SAT-35690](https://redhat.atlassian.net/browse/SAT-35690) | CP on path for registration, manifest, TLS — use CP-SLO-1–9 |
| [PSASTRAT-109](https://redhat.atlassian.net/browse/PSASTRAT-109) | Should not report CP "done" until CP-SLO suite defined |

---

## Related

- [pqc-performance-slos.md](./pqc-performance-slos.md) — SLO-3 (cert issuance), SLO-11 (manifest)
- [pqc-integration-gaps.md](./pqc-integration-gaps.md) — SAT-38910 Foreman↔CP
- [pqc-phased-perf-test-plan.md](./pqc-phased-perf-test-plan.md) — Phase 2 CP-heavy
- [pqc-registration-pr-stack-impact.md](./pqc-registration-pr-stack-impact.md) — Katello reduces CP **call volume**; [SAT-43398](https://redhat.atlassian.net/browse/SAT-43398)/[SAT-38910](https://redhat.atlassian.net/browse/SAT-38910) transport; [SAT-19326](https://redhat.atlassian.net/browse/SAT-19326) TLS 1.3
