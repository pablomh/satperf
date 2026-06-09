# PQC initcwnd Overflow — Impact by Connection Path

> **Unified doc:** [satellite-pqc-performance-impact.md](./satellite-pqc-performance-impact.md) §4.

Answers: *Where does the ~10–12 KB ML-DSA cert chain / TCP initcwnd penalty actually hurt Satellite?*

**Mechanism (one line):** On a **new** TLS handshake, if the server’s first TCP flight (ServerHello + **ML-DSA certificate chain** + related messages) exceeds the initial congestion window (~**10 × MSS ≈ 14.6 KB** with default initcwnd), the client needs an **extra round trip** before the handshake finishes. Cost ≈ **one RTT per new connection**, not per byte transferred afterward.

**It is not uniform** — it only bites when **RTT is meaningful**, the **server presents a large PQC cert chain**, and the connection is **new** (no reuse / no session resumption).

---

## When the penalty applies (all three)

| Condition | Why |
|-----------|-----|
| Non-trivial **RTT** | Extra RTT on loopback (~50 µs) is invisible; on 20–200 ms WAN it dominates |
| **Server** sends ML-DSA (or dual RSA+ML-DSA) **cert chain** in handshake | ~12 KB chain vs ~3.5 KB classical → often overflows initcwnd |
| **Fresh** TCP+TLS connection | No TLS session ticket/resumption, no HTTP keep-alive reuse |

**Not required:** Client mTLS with ML-DSA adds more handshake bytes (registration/content auth) and can worsen the picture, but the cited “~12 KB chain” problem is primarily the **server certificate flight**.

---

## Where it hurts (cross-network)

| Connection path | TLS server | Cert presented by | Typical RTT | Impact | Primary workload |
|-----------------|------------|-------------------|---------------|--------|------------------|
| **Client → Satellite** | Apache (Satellite) | Satellite ML-DSA chain | 1–100+ ms | **High** | Registration, sub-man, API, UI |
| **Client → Capsule** | Apache / smart-proxy | Capsule chain | 1–100+ ms | **High** | Registration proxy, content, remote sites |
| **Satellite → Capsule** (smart-proxy, sync triggers) | Capsule | Capsule | 1–50 ms | **Medium** | Proxy calls; reuse helps |
| **Capsule → Satellite** (callbacks, forwarding) | Apache | Satellite | 1–50 ms | **Medium** | Registration forwarding |
| **Satellite → CDN (Akamai)** | Akamai | Akamai (if PQC) | 10–200 ms | **Medium** | Repo sync **connect** phase; bulk download amortizes |
| **Capsule Pulp → Satellite Pulp** | Satellite Pulp | Satellite | 1–50 ms | **Medium–Low** | Long syncs reuse connections |

### Registration (worst multiplier)

- Each host: typically **one new TLS handshake** to Satellite or Capsule.
- **10,000 hosts × +1 RTT** at 20 ms ≈ **+200 s** cumulative wall-clock (order-of-magnitude; pooling/resumption reduces this).
- **Highest mitigation priority** for initcwnd tuning and connection reuse on the registration endpoint.

### Content serving (dnf/yum) — hurts less per operation

- Handshake cost paid **once per connection** (or per dnf run).
- After connect: large RPM transfers (**>50 KB** each) → bulk time dominates; studies cite **<5%** impact on time-to-last-byte for large objects.
- Still matters for: **first package** on a new connection, metadata-heavy patterns, many short-lived connections.

### CDN / Pulp sync

- Penalty on **each new connection** to CDN or peer Pulp.
- **Keep-alive** and connection reuse in Pulp during large sync → handshake amortized over GB transferred.
- First connection + metadata fetches still pay full tax.

---

## Where it does **not** matter (or barely)

| Connection path | RTT | Why initcwnd overflow is negligible |
|-----------------|-----|-------------------------------------|
| **Puma → Candlepin** (loopback TLS) | ~0.05 ms | Extra RTT ≈ microseconds |
| **Puma → Pulp API** (local) | ~0.05 ms | Same |
| **Container ↔ container** on same host | ~0.05 ms | Same |

**Important distinction:** Foreman↔Candlepin on loopback is **not** an initcwnd problem. Its PQC issue is different: **Java/Tomcat may not negotiate PQC TLS until Java 29** ([PRODSECRM-160](https://redhat.atlassian.net/browse/PRODSECRM-160), [SAT-38910](https://redhat.atlassian.net/browse/SAT-38910)). A **unix socket** (no TLS) avoids both issues for that hop.

---

## Is it “every encrypted communication”?

**No.**

| Protocol / path | initcwnd + large **server** cert chain? | Notes |
|---------------|----------------------------------------|--------|
| Client ↔ Satellite/Capsule **HTTPS** | **Yes** (when new connection + ML-DSA server cert) | Main concern |
| **SSH** (REX) | Different | ML-KEM KEX ~2 KB extra; host keys larger but not the same 12 KB TLS chain story |
| **Foreman ↔ Candlepin** (localhost) | **No** (initcwnd) | Functional PQC/Tomcat issue separate |
| **PostgreSQL**, **Redis/Valkey**, internal gRPC | Depends on TLS use & cert size | Usually local or few connections; profile separately |
| **Manifest download** (browser/admin) | **Yes** if HTTPS to Satellite | Fewer connections than registration |
| **Already-established** connection with keep-alive | **No** (for that session) | Subsequent requests skip full handshake |

---

## Mitigation priority (by path)

| Priority | Path | Mitigation |
|----------|------|------------|
| 1 | Client → Satellite/Capsule **registration** | initcwnd 30–40; TLS session tickets; tune keep-alive on registration URL |
| 2 | Client ↔ Capsule **remote/WAN** | Same + cert chain shortening; hybrid `DEFAULT:PQ` until clients ready |
| 3 | Satellite ↔ Capsule | [smart-proxy#935](https://github.com/theforeman/smart-proxy/pull/935) script cache; RHSM forward pooling (gap — see [pqc-registration-pr-stack-impact.md](./pqc-registration-pr-stack-impact.md)) |
| 4 | Satellite → CDN | Pulp HTTP keep-alive; sync design already reuses connections |
| 5 | Localhost Puma→CP | **Not initcwnd** — [katello#11726](https://github.com/Katello/katello/pull/11726)/[#11754](https://github.com/Katello/katello/pull/11754) pooling; unix socket ([SAT-38910](https://redhat.atlassian.net/browse/SAT-38910)) |

---

## Short answers to common questions

| Question | Answer |
|----------|--------|
| Registration? | **Yes — worst case** (many new handshakes). |
| Content serving? | **Once per connection**, then amortized; **moderate** overall. |
| Puma → Candlepin? | **No** for initcwnd (loopback). **Yes** for “does PQC TLS work at all” (SAT-38910). |
| Capsule → client? | **Yes** — same as client → Satellite. |
| Every encrypted comm? | **No** — only **new TLS handshakes** with **large server cert chains** over **non-zero RTT**. |

---

## Related docs

- [pqc-tls-handshake-research.md](./pqc-tls-handshake-research.md) — sizes and % overhead
- [pqc-performance-mitigations.md](./pqc-performance-mitigations.md) — initcwnd tuning
- [pqc-registration-pr-stack-impact.md](./pqc-registration-pr-stack-impact.md) — registration PRs × PQC by path
- [pqc-phased-perf-test-plan.md](./pqc-phased-perf-test-plan.md) — test good vs lossy RTT
