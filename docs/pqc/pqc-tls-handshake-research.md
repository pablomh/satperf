# PQC TLS Handshake — Research Summary

> **Unified doc:** [satellite-pqc-performance-impact.md](./satellite-pqc-performance-impact.md) §3–4.

Quantitative basis for Satellite PQC performance planning.

**Dominant finding:** Handshake **size** (cert chain exceeding TCP initial congestion window) drives latency more than ML-DSA **signing CPU** (which is actually faster than RSA for signing operations).

---

## Comparative metrics (classical vs PQC)

| Metric | Classical (ECDSA/X25519) | PQC (ML-DSA-65 / ML-KEM-768) | Overhead / note |
|--------|------------------------|------------------------------|-----------------|
| TLS handshake data (cert chain) | ~3.5 KB | ~12 KB | **3–5× larger** |
| Signature size | 64 B (ECDSA) | 3,309 B (ML-DSA-65) | **~50×** |
| Public key size | 64 B (ECDSA) | 1,952 B (ML-DSA-65) | **~30×** |
| Key exchange extra data | — | ~2 KB (ML-KEM-768) | Modest |
| **Signing speed** | ~5 ms (RSA-2048) | **~0.2 ms (ML-DSA-65)** | **PQC faster** |
| Verification speed | Baseline | Comparable to ECDSA | Negligible |
| Handshake latency (good network) | Baseline | **+~15%** | Usually acceptable |
| Handshake TTFB (constrained network) | Baseline | **+~32%** | Significant |
| Handshake (lossy network) | Baseline | **6–8× slower** | **Critical** for remote Capsules |

---

## TCP initial congestion window (initcwnd)

**Mechanism:** When the TLS ServerHello + certificate chain exceeds what fits in the first TCP congestion window, the client needs an **additional round trip** before the handshake completes.

- Default **initcwnd ≈ 10 × MSS** (~14.6 KB with 1460 B MSS).
- ML-DSA-65 chains at **~12 KB** for auth data alone, plus other TLS messages, often **overflow** this window.
- Cloudflare cites **~9 KB** as a practical “performance turning point” for auth data.

**Satellite impact:**

| Scenario | Effect |
|----------|--------|
| 10,000 host registrations | 10,000+ handshakes; +1 RTT each → e.g. +50ms × 10k = **~500s** cumulative if no reuse |
| CDN sync | Penalty on **connection setup**; bulk transfer after connect largely unaffected (&lt;5% TTLB for &gt;50 KB per Amazon study) |
| Capsule ↔ Satellite | Every new connection pays tax; **keep-alive critical** |
| Smart-proxy ↔ Foreman | Per-call handshake cost |

**Mitigation:** Increase **initcwnd to 30–40** on Satellite and Capsules — see [pqc-performance-mitigations.md](./pqc-performance-mitigations.md).

---

## Satellite connection surfaces (SAT-36257, SAT-38907, SAT-43401)

High-volume TLS paths:

- Client registration (thousands of hosts)
- Content sync from CDN
- Capsule ↔ Satellite sync
- Web UI, Hammer, API integrations
- Smart-proxy ↔ Foreman

Map each to [pqc-tls-component-matrix.csv](./pqc-tls-component-matrix.csv) / SAT-38970 investigation.

---

## Implications for SLOs

| SLO area | Emphasis |
|----------|----------|
| SLO-10 (handshake p95) | Measure **with** and **without** initcwnd tuning |
| SLO-1/2 (registration) | Test on **good** and **lossy** network profiles |
| CDN sync (SLO-4/5) | Separate **connect** vs **transfer** phases |
| Capsule remote sites | Mandatory lossy-network scenario in [pqc-phased-perf-test-plan.md](./pqc-phased-perf-test-plan.md) Phase 5 |

---

## What this does NOT penalize

- **Steady-state bulk download** after TLS established (CDN, large RPM transfers)
- **ML-DSA signing** in Candlepin during registration (signing is faster than RSA)
- **RPM ML-DSA verify** on client (comparable to ECDSA per program assumptions — still validate at scale)

---

## References

- Amazon: PQC TLS impact on time-to-last-byte
- Cloudflare: PQC certificate chain overhead
- NIST PQC TLS guidance
- Layered / realistic-network PQC TLS benchmarks
- OpenSSL 3.5 PQC lab (RHEL 9.6 context)
