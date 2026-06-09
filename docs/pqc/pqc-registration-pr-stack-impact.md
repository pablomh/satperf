# PQC Impact: Registration Performance PR Stack

> **Unified doc:** [satellite-pqc-performance-impact.md](./satellite-pqc-performance-impact.md) §6 — this file is the detailed expansion.

Maps the **Foreman/Katello registration performance program** (April–May 2026) to **Post-Quantum Cryptography (PQC)** costs under [SAT-35690](https://redhat.atlassian.net/browse/SAT-35690).

**Sources:**

- Registration program: `satperf/.claude/worktrees/observability/docs/registration-performance-improvements.md`
- Plan snapshot: `~/.claude/plans/gleaming-leaping-manatee.md` (older PR numbers — see [lineage](#pr-lineage-older-vs-current))
- PQC path model: [pqc-initcwnd-impact-by-path.md](./pqc-initcwnd-impact-by-path.md)

---

## PQC cost model (two drivers)

| Driver | Mechanism | Dominant paths |
|--------|-----------|----------------|
| **Data size** | ML-DSA server cert chain ~**12 KB** vs ~3.5 KB classical → TCP **initcwnd** overflow → **+1 RTT** per **new** handshake when RTT is non-trivial | Client→Satellite/Capsule; Capsule→Satellite |
| **CPU + memory** | Larger chain parse, ML-KEM, bigger buffers; ML-DSA verify ~comparable to ECDSA; **signing** in CP is faster than RSA but **payloads** are larger | All TLS paths; especially **high handshake count** paths |

**Not the main regression driver:** ML-DSA **signing CPU** during consumer creation ([candlepin-pqc-performance-impact.md](./candlepin-pqc-performance-impact.md)).

---

## Connection paths (reference)

| Path | Network | initcwnd overflow? | PQC cost driver |
|------|---------|-------------------|-----------------|
| Client → Satellite/Capsule | WAN (1–100+ ms RTT) | **Yes** | Data size |
| Capsule → Satellite (RHSM proxy, script fetch) | LAN/WAN (1–50 ms) | **Yes** | Data size |
| Puma → Candlepin | Localhost (~0.05 ms) | **No** | CPU + memory per handshake |
| Puma → Pulp | Localhost | **No** | CPU |

---

## Mitigation layers (stacking model)

Test and deploy in layers. Each layer addresses a different PQC failure mode.

```text
Layer A  Client → Apache           initcwnd, keep-alive, foremanctl#495 admission control
Layer B  Capsule → Satellite       smart-proxy#935; future RHSM connection pool (gap)
Layer C  Puma → Candlepin          katello#11726 OR #11754 (one only)
Layer D  CP call count             katello#11731, #11696, #11694
Layer E  Satellite CPU/DB          foreman#10979/#10980, katello#11701
Layer F  CP create path            pool 20→100+, duplicate compliance, async (future)
```

**After Layers C–E:** remaining PQC exposure concentrates on **Layer A** (one external handshake per host) and **Layer B** on capsule topologies.

---

## PR inventory (current)

| PR | JIRA (examples) | Component | Status (May 2026) | Classical role |
|----|-----------------|-----------|-------------------|----------------|
| [katello#11731](https://github.com/Katello/katello/pull/11731) | — | Katello | Open | Static SCA compliance — **0** CP calls (was ~13/reg) |
| [katello#11696](https://github.com/Katello/katello/pull/11696) | SAT-43940 | Katello | Open | Status cache — ~24 CP calls → 1 |
| [katello#11694](https://github.com/Katello/katello/pull/11694) | SAT-43942 | Katello | Open | Drop 2–3 redundant post-create GETs |
| [katello#11701](https://github.com/Katello/katello/pull/11701) | SAT-44043 | Katello | Open | Surgical 3-fact import at register |
| [katello#11726](https://github.com/Katello/katello/pull/11726) | #39291 | Katello | Open | `net-http-persistent` + RestClient pool |
| [katello#11754](https://github.com/Katello/katello/pull/11754) | — | Katello | Open | Faraday 2.x persistent CP (PQC called out in PR) |
| [foreman#10979](https://github.com/theforeman/foreman/pull/10979) | SAT-45132 | Foreman | Open | Bulk fact `insert_all` |
| [foreman#10980](https://github.com/theforeman/foreman/pull/10980) | SAT-45133 | Foreman | Open | `additive:` fact import mode |
| [foreman#10948](https://github.com/theforeman/foreman/pull/10948) | SAT-44625 | Foreman | Open | TCP reset false-failure recovery |
| [foreman#10969](https://github.com/theforeman/foreman/pull/10969) | SAT-44793 | Foreman | Open | TopbarSweeper thread-safety |
| [smart-proxy#935](https://github.com/theforeman/smart-proxy/pull/935) | SAT-43996 | Smart Proxy | Open | Registration script cache |
| [smart-proxy#936](https://github.com/theforeman/smart-proxy/pull/936) | SAT-44913 | Smart Proxy | Open | Configurable `foreman_request_timeout` |
| [foremanctl#483](https://github.com/theforeman/foremanctl/pull/483) | SAT-44878 | foremanctl | **Merged** | Apache MPM event.conf |
| [foremanctl#495](https://github.com/theforeman/foremanctl/pull/495) | SAT-44963 | foremanctl | Draft | Registration admission control |

**Superseded / closed (do not double-count):**

| Old | Current | Note |
|-----|---------|------|
| [katello#11692](https://github.com/Katello/katello/pull/11692) compliance cache | **#11731** static response | #11731 is strictly better for PQC (0 CP calls) |
| [foreman#10942](https://github.com/theforeman/foreman/pull/10942) bulk inserts | **#10979 + #10980** | Split PRs |

**Connection pooling — ship one:**

| PR | Approach | PQC note |
|----|----------|----------|
| [#11726](https://github.com/Katello/katello/pull/11726) | RestClient + `Net::HTTP::Persistent` | Smaller change |
| [#11754](https://github.com/Katello/katello/pull/11754) | Faraday 2.x + `net_http_persistent` | Explicit PQC readiness; mTLS prep |

Classical A/B (Satellite 6.16, 2k hosts, [#11754](https://github.com/Katello/katello/pull/11754) PR body): TCP ActiveOpens **692K → 2.7K** (~99.6%). Order-of-magnitude PQC internal TLS CPU bound: **692K × ~0.65 ms ≈ 450 s** vs **2.7K × ~0.65 ms ≈ 1.75 s** (illustration, not yet measured under `DEFAULT:PQ`).

---

## Per-PR PQC impact

### Phase 1 — Reduce redundant Candlepin traffic

#### katello#11731 — Static compliance (13 → 0 CP calls/reg)

| PQC dimension | Impact |
|---------------|--------|
| Puma→CP handshakes | **EXTREME** without pooling — eliminates highest-frequency internal call entirely |
| Tomcat / CP load | **EXTREME** — zero threads for compliance under SCA |
| initcwnd | N/A on loopback; no client path change |

**Tier: EXTREME** (strictly better than #11692 cache for PQC).

#### katello#11696 — Status cache (~24 → 1 CP call/reg)

| PQC dimension | Impact |
|---------------|--------|
| Handshakes | **HIGH** without #11726/#11754 |
| CP capacity | **HIGH** — frees threads when CP is busy with ML-DSA issuance |
| TTL (180s) | Load-bearing under PQC — shorter TTL → more refresh storms during bursts |

**Tier: HIGH**

#### katello#11694 — Eliminate redundant GETs (2–3 → 0)

Occurs during `POST /rhsm/consumers` flow. Avoids re-fetching **larger** ML-DSA identity payloads (~2 KB vs ~300 B RSA).

**Tier: MODERATE**

---

### Phase 2 — Reduce DB pressure

#### foreman#10979 + #10980 — Bulk inserts + additive mode

No direct TLS interaction. AR P50 **3,413 ms → 566 ms** (−83%) buys **~2.8 s headroom** per request — absorbs PQC-added latency on the same request without hitting Apache 60s timeouts.

**Tier: CRITICAL (indirect)**

#### katello#11701 — Surgical fact import (193 → 3 at step 2)

Shortens critical path while CP generates ML-DSA certs. Deferred facts hit `PUT /rhsm/consumers/:id` ~33s later when burst pressure is lower.

**Tier: MODERATE–HIGH**

---

### Phase 3 — Capsule-side

#### smart-proxy#935 — Registration script cache

Eliminates **Capsule → Satellite** `GET /register` storm. This is a **real RTT** path where initcwnd overflow applies.

Example (200 hosts, 20 ms RTT, warm cache): **200 cross-network PQC handshakes → 1** after warm-up.

**Tier: VERY HIGH** (capsule/LB topologies); **low** for direct-to-Satellite.

#### smart-proxy#936 — Configurable timeout

PQC stretches handshakes and CP work → more requests approach 60s default.

**Tier: MODERATE** (reliability)

---

### Phase 4 — Reliability

#### foreman#10948 — TCP reset recovery

Under load, CP may succeed while client sees failure → **retry loops** → **multiplied PQC handshakes**. Mechanism: longer responses + queue depth + congestion (not cert size alone).

**Tier: HIGH**

#### foreman#10969 — TopbarSweeper thread-safety

Faster requests (after Phase 2) widen race window; may surface at **lower** concurrency than pre-patch. Reliability, not crypto-specific.

**Tier: MODERATE**

---

### Phase 5 — Connection pooling (Layer C)

#### katello#11726 or #11754 (one)

| Metric | Without pooling | With pooling |
|--------|-----------------|--------------|
| TCP connections (2k hosts, classical A/B) | ~692,000 | ~2,700 |
| ML-DSA chain bytes on wire (order-of-mag.) | ~8.3 GB | ~32 MB |
| TLS CPU (illustrative @ 0.65 ms/handshake) | ~450 s | ~1.75 s |

Loopback: **no initcwnd** penalty; **CPU/memory** dominates.

**Tier: EXTREME**

---

### Phase 6 — Infrastructure

#### foremanctl#483 — MPM event (merged)

Explicit `MaxRequestWorkers` tuning. PQC holds Apache workers slightly longer per TLS handshake.

**Tier: MODERATE**

#### foremanctl#495 — Registration admission control

Formula: `max = puma_workers × threads × 5`. Prevents hundreds of completed PQC handshakes queuing with **larger session state**. Under PQC, validate whether **×5 → ×3** is safer (memory per queued TLS session).

**Tier: HIGH**

---

## Candlepin follow-on (Layer F)

From registration program investigation (`gleaming-leaping-manatee.md` §7). Becomes **primary bottleneck** after Katello stack merges.

| Item | PQC interaction | Tier |
|------|-----------------|------|
| Connection pool 20 → 100+ | Longer per-request hold when ML-DSA issuance + larger payloads | **Very high** |
| Long `@Transactional` on create | Holds connection longer under PQC | **High** |
| Duplicate compliance in `ConsumerResource` | CPU (JS rules), not TLS — still worth fixing | **Moderate** |
| Async compliance via JobManager | Shortens critical path | **Very high** |
| Pessimistic pool locking | Serializes; worse when each op is slower | **Risk amplifier** |

Final compliance aggregation for multi-pool registrations remains **required** — do not remove.

---

## JIRA: Foreman ↔ Candlepin transport (architecture)

These tickets are **not** replaced by the registration PR stack. They decide **whether** Puma→Candlepin uses PQC TLS at all under `DEFAULT:PQ`, and therefore how much Layer C pooling matters.

### [SAT-43398](https://redhat.atlassian.net/browse/SAT-43398) — Investigate Foreman–Candlepin communication (PQC)

| Field | Value |
|-------|--------|
| Type | Task (**New**) |
| Relationship | **Informs** epic [SAT-38910](https://redhat.atlassian.net/browse/SAT-38910) |
| Parent program | [SAT-35690](https://redhat.atlassian.net/browse/SAT-35690) / [SAT-36257](https://redhat.atlassian.net/browse/SAT-36257) |

**Goal:** Choose how Foreman talks to Candlepin when host crypto-policy is PQC on RHEL 10.2+ — Tomcat 10 does not do TLS 1.3 + PQC until Java 29 ([PRODSECRM-160](https://redhat.atlassian.net/browse/PRODSECRM-160)).

**Paths under evaluation:**

| Path | PQC perf effect on registration |
|------|--------------------------------|
| **A1** Container networking isolation | Security boundary; perf TBD in spike |
| **A2** Unix socket (bypass TLS) | **Eliminates** Puma→CP TLS handshake tax — Layer C pooling still useful for HTTP reuse, not ML-DSA parse per handshake |
| **B** FFM / OpenSSL in Tomcat (tomcat-native) | **Keeps** TLS; makes [#11726](https://github.com/Katello/katello/pull/11726)/[#11754](https://github.com/Katello/katello/pull/11754) **mandatory** at reg@scale |

**Spike success criteria (from Jira):** document feasibility, **performance implications**, security, complexity, upgrade path — then recommend path for SAT-38910.

**satperf input for SAT-43398:** Provide registration storm data **with** full Katello stack (R1/R3) comparing p95 Foreman→CP latency and ActiveOpens for each architecture option; reference [candlepin-pqc-performance-impact.md](./candlepin-pqc-performance-impact.md) CP-SLOs.

**Status in prior docs:** Listed in [pqc-integration-gaps.md](./pqc-integration-gaps.md) and hierarchy CSV; **not** a duplicate of SAT-38910 — it is the **decision spike** SAT-38910 depends on.

---

### [SAT-19326](https://redhat.atlassian.net/browse/SAT-19326) — TLS 1.3 support for Satellite 6 (Candlepin)

| Field | Value |
|-------|--------|
| Type | Feature Request (**New**) |
| Scope | TLS 1.3 on Candlepin listener (port 23443); predates PQC program |
| Related | [SAT-42162](https://redhat.atlassian.net/browse/SAT-42162) spike **Closed** — “Investigate TLS 1.3 support” |
| Blocks / links | [SAT-38910](https://redhat.atlassian.net/browse/SAT-38910), [SAT-38907](https://redhat.atlassian.net/browse/SAT-38907), [SAT-38913](https://redhat.atlassian.net/browse/SAT-38913), [SAT-35691](https://redhat.atlassian.net/browse/SAT-35691) |

**PQC relevance:** ML-KEM and modern PQC cipher suites require **TLS 1.3**. Without TLS 1.3 on the Candlepin endpoint, enabling `DEFAULT:PQ` on the host may leave Foreman→Candlepin on **TLS 1.2 only** — the registration PR stack optimizes **connection count**, but cannot add PQC negotiation that the listener does not offer.

**Interaction with registration perf PRs:**

| If SAT-19326 / CP TLS 1.3 | Effect on Layer C (#11726/#11754) |
|---------------------------|-----------------------------------|
| Not done | Pooling still cuts **connection churn**; handshakes remain classical TLS 1.2 |
| Done + tomcat-native / Java PQC path | Pooling avoids **692K-class** PQC handshake storms |
| SAT-38910 chooses unix socket (A2) | SAT-19326 less critical for **internal** path; still relevant for **direct** CP TLS clients |

**Recommendation:** Track SAT-19326 under [SAT-36257](https://redhat.atlassian.net/browse/SAT-36257) infrastructure; gate PQC registration **functional** tests on TLS 1.3 probe to Candlepin (`openssl s_client -tls1_3`).

**Gap:** Was **missing** from initial PQC doc set until this update.

---

### How SAT-43398 + SAT-19326 relate to the PR stack

```text
SAT-19326 (TLS 1.3 on CP)     ──► enables PQC KEX on CP listener (if TLS path kept)
SAT-43398 (spike)           ──► picks A1 / A2 / B for SAT-38910
SAT-38910 (epic)            ──► implements chosen path
Registration PR stack       ──► optimizes volume + reuse on whatever transport wins
```

**Order for perf testing:**

1. Classical reg stack (R1) — valid regardless of SAT-38910.
2. SAT-43398 recommendation available — document expected Layer C behavior.
3. PQC policy (R3) — only interpret FAIL if CP TLS 1.3 + SAT-38910 path are in place or explicitly waived.

---

## PQC gaps (not covered by current PRs)

| Gap | Why it matters | Recommendation |
|-----|----------------|----------------|
| **Client → Satellite TLS** | One handshake/host; initcwnd + RTT; **no PR in this stack fixes it** | initcwnd 30–40; Apache keep-alive / session tickets ([pqc-performance-mitigations.md](./pqc-performance-mitigations.md)) |
| **Smart-proxy → Satellite RHSM forward** | Each forwarded `/rhsm/*` may be new cross-network TLS | Persistent HTTP client in smart-proxy (follow #11726/#11754 pattern) |
| **CDN / `cdn.rb`** | RestClient per request ([#11754](https://github.com/Katello/katello/pull/11754) notes) | Extend Faraday persistent pattern to CDN layer |
| **Foreman ↔ CP transport** | Java/Tomcat PQC gap; architecture undecided | [SAT-43398](https://redhat.atlassian.net/browse/SAT-43398) spike → [SAT-38910](https://redhat.atlassian.net/browse/SAT-38910) |
| **Candlepin TLS 1.3** | PQC KEX needs TLS 1.3 on :23443 | [SAT-19326](https://redhat.atlassian.net/browse/SAT-19326), [SAT-42162](https://redhat.atlassian.net/browse/SAT-42162) (closed spike) |
| **PQC perf baseline** | No SAT-35690 card for `DEFAULT:PQ` registration storm | satperf Phase 2 matrix ([pqc-phased-perf-test-plan.md](./pqc-phased-perf-test-plan.md)) |
| **Admission control tuning** | ×5 may be high for larger PQC session state | Re-test under `DEFAULT:PQ` |

---

## Summary: PQC amplification tier

| PR / measure | Classical impact | PQC tier | Primary PQC mechanism |
|--------------|------------------|----------|------------------------|
| #11754 or #11726 | 99.6% fewer TCP opens | **EXTREME** | Internal handshake CPU/memory |
| #11731 | 13 CP calls → 0 | **EXTREME** | Zero CP + zero internal TLS for compliance |
| smart-proxy#935 | Script cache ~100% hit | **VERY HIGH** | initcwnd on Capsule→Satellite |
| #11696 | ~24 CP calls → 1 | **HIGH** | Fewer internal round-trips + CP threads |
| foremanctl#495 | Smooths bursts | **HIGH** | PQC session memory amplification |
| #10948 | False failure recovery | **HIGH** | Retry storm → more handshakes |
| #10979/#10980 + #11701 | −83% AR time | **CRITICAL (indirect)** | Timeout headroom |
| #11694 | 2–3 fewer GETs | **MODERATE** | Larger cert payloads |
| foremanctl#483 | MPM tuning | **MODERATE** | Worker occupancy |
| smart-proxy#936 | Configurable timeout | **MODERATE** | Longer PQC requests |
| #10969 | Thread-safety | **MODERATE** | Throughput side-effect |
| CP pool / async | CP saturation | **Very high** | Post-merge bottleneck |

---

## satperf test matrix (PQC)

### apply_prs bundle (example)

```bash
export PARAM_apply_prs_satellite='{method: diff, targets: [
  {org: theforeman, repo: foreman, base_dir: /usr/share/foreman, prs: [10979, 10980, 10948, 10969]},
  {org: Katello, repo: katello, base_dir: /usr/share/gems/gems/katello-*, prs: [11731, 11696, 11694, 11701, 11726]}
]}'
```

Smart-proxy on capsules: [#935](https://github.com/theforeman/smart-proxy/pull/935), [#936](https://github.com/theforeman/smart-proxy/pull/936) (manual or capsule `apply_prs` when available).

### Recommended comparison grid

| Run | Crypto policy | Patch stack | Topology |
|-----|---------------|-------------|----------|
| R0 | `DEFAULT` | None | Direct + capsule |
| R1 | `DEFAULT` | Full stack (Layers C–E + reliability) | Direct + capsule |
| R2 | `DEFAULT:PQ` | None | Direct + capsule |
| R3 | `DEFAULT:PQ` | Full stack | Direct + capsule |
| R4 | `DEFAULT:PQ` | Full stack + foremanctl#495 | Direct (foremanctl) |

**Metrics:** SLO-1/2/3 ([pqc-performance-slos.md](./pqc-performance-slos.md)); `POST /rhsm/consumers` P50/P95; registration success rate; TCP ActiveOpens / `ss -s`; optional `registration_metrics.py` from observability worktree.

**Exit hypothesis:** R3 should approach R1 pass rates; gap R3 vs R1 isolates **Layer A** PQC tax.

---

## PR lineage (older vs current)

| gleaming-leaping-manatee.md | Current (observability doc) |
|-----------------------------|-----------------------------|
| katello#11692 compliance cache | **katello#11731** static compliance |
| foreman#10942 bulk inserts | **foreman#10979**, **foreman#10980** |
| (not listed) | katello#11726, #11754, foreman#10969, foremanctl#483/495 |

---

## Program narrative for SAT-35690

The registration performance program is **one of the most effective PQC mitigations** in the Satellite stack for **registration at scale**:

1. **Layers C + D** remove hundreds of internal TLS handshakes and CP calls per host.
2. **Layer E** buys timeout headroom when PQC stretches each operation.
3. **Layer A** remains the **SAT-35690 acceptance risk** after merge — plan initcwnd, client keep-alive, and admission control validation under `DEFAULT:PQ`.

**Does not replace:** [SAT-43398](https://redhat.atlassian.net/browse/SAT-43398) / [SAT-38910](https://redhat.atlassian.net/browse/SAT-38910) (Foreman↔CP transport), [SAT-19326](https://redhat.atlassian.net/browse/SAT-19326) (Candlepin TLS 1.3), CP pool sizing, CDN persistent connections, or dedicated PQC benchmark ownership under SAT-35690.

---

## Related documents

| Document | Purpose |
|----------|---------|
| [pqc-performance-mitigations.md](./pqc-performance-mitigations.md) | initcwnd, reuse, gaps index |
| [pqc-initcwnd-impact-by-path.md](./pqc-initcwnd-impact-by-path.md) | Path-by-path initcwnd |
| [pqc-phased-perf-test-plan.md](./pqc-phased-perf-test-plan.md) | Phase 2 registration + PQC matrix |
| [candlepin-pqc-performance-impact.md](./candlepin-pqc-performance-impact.md) | CP signing vs handshake |
| [pqc-analysis-reconciliation.md](./pqc-analysis-reconciliation.md) | Program index |
| [registration-performance-improvements.md](../../.claude/worktrees/observability/docs/registration-performance-improvements.md) | Classical measured results |
