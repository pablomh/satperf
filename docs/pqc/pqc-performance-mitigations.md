# poolPQC Performance Mitigations (Satellite)

> **Unified doc:** [satellite-pqc-performance-impact.md](./satellite-pqc-performance-impact.md) §10.

Operational and engineering mitigations merged from program analysis and TLS research. Maps to Jira features under [SAT-35690](https://redhat.atlassian.net/browse/SAT-35690).

---

## 1. Tune TCP initcwnd (highest impact)

**Problem:** ML-DSA cert chains (~12 KB) overflow default initcwnd → extra RTT per handshake ([pqc-tls-handshake-research.md](./pqc-tls-handshake-research.md)).

**Action:**

- Set `initcwnd` to **30–40** on Satellite and Capsule (sysctl / tuned profile).
- Document in installer or `satellite-installer` post-install tuning for PQC deployments.
- Validate on registration storm and CDN sync connect phases.

**Applicable tickets:** [SAT-36257](https://redhat.atlassian.net/browse/SAT-36257), [SAT-38907](https://redhat.atlassian.net/browse/SAT-38907)

**Proposed JIRA:** Installer/sysctl profile for PQC deployments (child of SAT-36257).

---

## 2. Maximize TLS connection reuse

initcwnd tuning (§1) reduces the cost **per** full handshake. Connection reuse reduces **how many** full handshakes happen. Both are required for registration-at-scale under PQC.

**Registration path (reference):**

```text
Client                    Satellite/Capsule              Internal (loopback)
  │                              │                              │
  ├─ GET /register ─────────────►│  (script)                    │
  ├─ POST /rhsm/consumers ──────►│ ────────────────────────────►│ Candlepin (create)
  ├─ GET /rhsm/status  (×12–24) ─►│ ────────────────────────────►│ (was: every call = new conn)
  ├─ GET .../compliance (×12) ───►│  (#11731: 0 CP calls)        │
  ├─ POST /register ─────────────►│                              │
  └─ PUT /rhsm/consumers/:id ─────►│ ────────────────────────────►│ (facts)
```


| Leg                       | Who opens TCP+TLS             | Reuse mitigations in this section                     |
| ------------------------- | ----------------------------- | ----------------------------------------------------- |
| **A** Client → Apache     | `subscription-manager` / curl | Keep-alive, TLS resumption, HTTP/2 (client-dependent) |
| **B** Capsule → Satellite | smart-proxy (forward)         | #935 script cache; future RHSM HTTP pool (gap)        |
| **C** Puma → Candlepin    | Katello RestClient/Faraday    | #11726 / #11754 (§4 — not HTTP keep-alive to client)  |


---

### 2.1 HTTP keep-alive (Apache ↔ client)

**What it is:** After the first request on a connection, Apache leaves the TCP (and TLS) session open. Further HTTP/1.1 requests on the **same connection** skip a new TCP setup and **skip a new full TLS handshake**.

**PQC effect:**


| Without keep-alive                                                                                 | With keep-alive                                                   |
| -------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| Each HTTP request may pay **full** ML-DSA handshake (+ initcwnd RTT on WAN)                        | **One** full handshake per connection, then only application data |
| `GET /rhsm/status` × 24 ≈ **24** handshakes worth of PQC tax (if client opened new conn each time) | Same host: **1** handshake, 24 requests on one TLS session        |


**Where it matters for Satellite:** Client → Satellite/Capsule for `/rhsm/`*, `/register`, and API paths during a **single** `subscription-manager register` run. This is the main way to shrink “many handshakes per host” on the **external** leg.

**What it does *not* fix:**

- **First** connection to Satellite for each host still does one full handshake (initcwnd §1 still applies once).
- **Different hosts** = different connections — 10,000 hosts still means ≥10,000 first handshakes.
- **Puma → Candlepin** — Apache keep-alive does not apply; use Katello pooling (§4).
- **Capsule → Satellite** — separate connection; see §2.5 and smart-proxy work.

**Client behavior:** `subscription-manager` and the registration script use HTTP clients (often curl-backed). Modern clients typically reuse connections **within a single process run** if the server advertises keep-alive (`Connection: keep-alive`). Validate under PQC with packet capture or `ss`/OpenSSL logging on a lab host — do not assume without measurement.

**Server side (Apache on Satellite/Capsule):**

- Foreman/Satellite Apache vhosts should allow keep-alive (default in RHEL httpd is usually on).
- Tune so connections stay alive long enough for the full registration sequence (~30–120 s), but not so long that burst registration exhausts worker/memory limits under PQC (larger TLS state per connection — see [foremanctl#495](https://github.com/theforeman/foremanctl/pull/495)).

**Illustrative directives** (names vary by `mod_ssl` / MPM version — confirm on target Satellite):

```apache
# Conceptual — verify against actual 06-*/foreman-Apache config
KeepAlive On
MaxKeepAliveRequests 100
KeepAliveTimeout 15
```

Under registration storm, balance `KeepAliveTimeout` with admission control: idle PQC TLS sessions consume more RAM than classical sessions.

**Validation (satperf):** Compare handshake count per registration UUID in traces — with keep-alive, multiple `/rhsm/`* entries should share one TLS session (same five-tuple, or single connection in packet capture).

**Tickets:** [SAT-36257](https://redhat.atlassian.net/browse/SAT-36257), [SAT-38907](https://redhat.atlassian.net/browse/SAT-38907), [SAT-43394](https://redhat.atlassian.net/browse/SAT-43394).

---

### 2.2 TLS session tickets / session resumption

**What it is:** After a successful full handshake, server and client can store session state. A **later** TCP connection sends a session ticket (TLS 1.3) or session ID (TLS 1.2); the server resumes with an **abbreviated** handshake — much less data on the wire, typically **no** full certificate chain re-flight, so **initcwnd overflow usually does not apply** to the resumed handshake.

**PQC effect:**


| Scenario                                                           | Benefit                                                       |
| ------------------------------------------------------------------ | ------------------------------------------------------------- |
| Same host **re-registers** or opens a second connection soon after | Abbreviated handshake; avoids repeating ~12 KB cert flight    |
| Burst of hosts each connecting **once**                            | **Little benefit** — first connection is still full handshake |
| Admin/UI/Hammer users hitting Satellite repeatedly                 | High benefit — same browser or client reuses sessions         |


**Where it matters:** Secondary to keep-alive for **first-time** registration storms. Important for **operational** traffic (UI, API automation) and **re-registration** (image rebuild, repair).

**Server side:** OpenSSL/mod_ssl on RHEL generally supports TLS 1.3 session tickets when TLS 1.3 is enabled. Ensure ticket rotation and key material are acceptable for your security model (tickets are encrypted but worth reviewing with InfoSec for PQC deployments).

**What it does *not* fix:**

- Per-host **first** registration in a storm (primary SAT-35690 scale case).
- Cross-host reuse (tickets are per-server, not shared across clients).
- Internal Puma→Candlepin path (separate TLS stack).

**Validation:** `openssl s_client -connect satellite:443 -tls1_3` twice with `-sess_out` / `-sess_in`; second connection should show **Reused** or abbreviated handshake in debug output.

**Tickets:** SAT-36257 (document in PQC hardening guide).

---

### 2.3 HTTP/2 (one handshake, many streams)

**What it is:** HTTP/2 multiplexes many requests as **streams** on **one** TCP+TLS connection. One full handshake sets up many parallel `/rhsm/`* (or API) requests without opening new connections.

**PQC effect:** Same economic model as HTTP/1.1 keep-alive, often **stronger** — clients cannot accidentally open parallel connections as easily; single connection carries all streams. Still **one** initcwnd-priced full handshake per connection.

**Satellite reality check:**


| Component                              | HTTP/2 expectation                                                                                |
| -------------------------------------- | ------------------------------------------------------------------------------------------------- |
| Apache (Satellite/Capsule)             | May offer h2 where `mod_http2` and certs are configured — confirm per vhost                       |
| `subscription-manager` / RHSM consumer | Historically **HTTP/1.1**-oriented; **do not assume h2** for registration without version testing |
| Smart-proxy (WEBrick)                  | HTTP/1.1 for Foreman forwarding — **not** HTTP/2 to Satellite today                               |
| Pulp / CDN clients                     | Separate; valuable for sync (SLO-4/5)                                                             |


**Recommendation:** Treat HTTP/2 as **high value where the client negotiates h2** (browsers, some automation). For **registration storm**, prioritize **HTTP/1.1 keep-alive** + initcwnd until CCT/subman explicitly supports and tests h2 to Satellite under ML-DSA certs.

**Validation:** `curl -v --http2 https://satellite.example.com/pub/...` or ALPN probe in `openssl s_client -alpn h2`.

**Tickets:** SAT-36257, [SAT-43401](https://redhat.atlassian.net/browse/SAT-43401) (Pulp/CDN paths may benefit more than sub-man).

---

### 2.4 Registration PR stack — internal and cross-network reuse

These are **not** Apache keep-alive settings; they are **application-level** reductions of connection count. See [pqc-registration-pr-stack-impact.md](./pqc-registration-pr-stack-impact.md).

#### Katello persistent pooling ([#11726](https://github.com/Katello/katello/pull/11726) / [#11754](https://github.com/Katello/katello/pull/11754))


|                             |                                                                                                                                      |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| **Path**                    | Puma → Candlepin (localhost TLS, port 23443)                                                                                         |
| **Problem without pooling** | Every Katello API call opens new TCP+TLS; ~346 connections per registration × N hosts → **692K** ActiveOpens (2k-host classical A/B) |
| **With pooling**            | ~1–2 connections per Puma worker thread, reused for all CP calls in that thread → **2.7K** ActiveOpens                               |
| **PQC cost avoided**        | Per-handshake **CPU/memory** (parse ~12 KB ML-DSA chain, ML-KEM) — **not** initcwnd (loopback RTT negligible)                        |
| **Complements**             | #11731, #11696, #11694 (fewer calls **and** fewer handshakes per remaining call)                                                     |


#### Katello call reduction (#11731, #11696, #11694)


| PR                                                                        | Effect on connections                                    |
| ------------------------------------------------------------------------- | -------------------------------------------------------- |
| [#11731](https://github.com/Katello/katello/pull/11731) static compliance | **Zero** Puma→CP HTTP for compliance polls (was ~13/reg) |
| [#11696](https://github.com/Katello/katello/pull/11696) status cache      | ~24 status calls → **1** CP round-trip per registration  |
| [#11694](https://github.com/Katello/katello/pull/11694)                   | Drops 2–3 redundant GETs after consumer create           |


Even with TLS reuse, fewer calls reduce Tomcat load and time holding CP threads during ML-DSA cert work.

---

### 2.5 Capsule script cache — [smart-proxy#935](https://github.com/theforeman/smart-proxy/pull/935)

**What it is:** For a given activation key, the registration bash script from `GET /register` is **identical** for every host. Without cache, each concurrent registration through a Capsule triggers smart-proxy → Satellite `GET /register` — a **new cross-network TLS connection** and full template render.

**Flow:**

```text
Without #935 (200 hosts, same activation key):
  Host₁ ──TLS──► Capsule ──TLS──► Satellite  GET /register
  Host₂ ──TLS──► Capsule ──TLS──► Satellite  GET /register
  … 200 times Capsule→Satellite handshakes (plus 200 Client→Capsule)

With #935 (cache warm):
  Host₁ ──TLS──► Capsule ──TLS──► Satellite  GET /register  (cache miss → fill)
  Host₂..N ──TLS──► Capsule  (cache hit — no Satellite round-trip)
```

**PQC effect:**


|                          |                                                                                                             |
| ------------------------ | ----------------------------------------------------------------------------------------------------------- |
| **initcwnd**             | Applies on **Capsule → Satellite** WAN/LAN RTT — each avoided handshake skips +1 RTT and ~12 KB cert flight |
| **Measured (classical)** | `GET /register` P50 ~770 ms → ~298 ms on capsule path when cache warm                                       |
| **Under PQC**            | Cross-network leg is **VERY HIGH** priority — one of few mitigations that hits initcwnd on Capsule topology |


**What it does *not* fix:**

- Client → Capsule still **one handshake per host** for that host’s traffic.
- Forwarded `/rhsm/`* from Capsule to Satellite still opens **new** connections per request today (**gap:** persistent HTTP client in smart-proxy — same pattern as #11754).

**Tickets:** [SAT-43996](https://redhat.atlassian.net/browse/SAT-43996).

---

### 2.6 Summary — which mitigation for which problem


| Problem                                   | Best mitigations                                                                                            |
| ----------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| **+1 RTT initcwnd** on client → Satellite | §1 initcwnd 30–40; §2.1 keep-alive (one handshake per host per run)                                         |
| **Many handshakes per host** on `/rhsm/`* | §2.1 keep-alive; §2.3 HTTP/2 if client supports                                                             |
| **10k hosts × first connection**          | §1 initcwnd; cannot eliminate — admission control [#495](https://github.com/theforeman/foremanctl/pull/495) |
| **Re-registration / UI**                  | §2.2 TLS session resumption                                                                                 |
| **Capsule → Satellite script storm**      | §2.5 #935                                                                                                   |
| **Puma → Candlepin handshake storm**      | §4 #11726/#11754 + call reduction PRs                                                                       |
| **Capsule → Satellite `/rhsm` forward**   | **Gap** — future persistent smart-proxy client                                                              |


**Applicable:** [SAT-43401](https://redhat.atlassian.net/browse/SAT-43401), [SAT-36257](https://redhat.atlassian.net/browse/SAT-36257), [SAT-43394](https://redhat.atlassian.net/browse/SAT-43394).

---

## 3. Foreman ↔ Candlepin: architecture ([SAT-43398](https://redhat.atlassian.net/browse/SAT-43398) → [SAT-38910](https://redhat.atlassian.net/browse/SAT-38910))

**Spike [SAT-43398](https://redhat.atlassian.net/browse/SAT-43398)** must complete before SAT-38910 implementation — evaluates unix socket vs tomcat-native vs container networking. See [pqc-registration-pr-stack-impact.md](./pqc-registration-pr-stack-impact.md) § JIRA.

**Candlepin TLS 1.3:** [SAT-19326](https://redhat.atlassian.net/browse/SAT-19326) (related closed spike [SAT-42162](https://redhat.atlassian.net/browse/SAT-42162)) — required for PQC KEX on the CP TLS listener if the TLS path is retained.

## 3b. Foreman ↔ Candlepin: prefer unix socket ([SAT-38910](https://redhat.atlassian.net/browse/SAT-38910))

**Context:** [PRODSECRM-160](https://redhat.atlassian.net/browse/PRODSECRM-160) — Java PQC in Tomcat delayed until Java 29.

**Options:**


| Option                      | Perf note                                                                                                         |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Unix socket                 | **Eliminates TLS** on highest-traffic internal path — potential **improvement** vs today                          |
| tomcat-native + APR         | Already in CP dev ([CANDLEPIN-1152](https://redhat.atlassian.net/browse/CANDLEPIN-1152)); measure handshake + CPU |
| Container networking bypass | Architecture-dependent                                                                                            |


**Recommendation:** Score each option on p95 Foreman→Candlepin latency under registration load.

---

## 4. Registration at scale — PR stack + connection amortization

**Problem:** 10k hosts = 10k **client** handshakes; without pooling, each host also triggers **hundreds** of Puma→Candlepin TLS handshakes.

**Full analysis:** [pqc-registration-pr-stack-impact.md](./pqc-registration-pr-stack-impact.md) — every Katello/Foreman/smart-proxy/foremanctl registration PR, PQC tier, gaps, and satperf matrix.

**Highest-impact items (summary):**


| Layer                | PRs                                                                                                                                                                                           | PQC tier                        |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------- |
| Puma→CP pooling      | [katello#11726](https://github.com/Katello/katello/pull/11726) **or** [#11754](https://github.com/Katello/katello/pull/11754) (ship **one**)                                                  | **EXTREME**                     |
| CP call elimination  | [#11731](https://github.com/Katello/katello/pull/11731), [#11696](https://github.com/Katello/katello/pull/11696), [#11694](https://github.com/Katello/katello/pull/11694)                     | **EXTREME / HIGH**              |
| Capsule script cache | [smart-proxy#935](https://github.com/theforeman/smart-proxy/pull/935)                                                                                                                         | **VERY HIGH** (capsule path)    |
| DB headroom          | [foreman#10979](https://github.com/theforeman/foreman/pull/10979), [#10980](https://github.com/theforeman/foreman/pull/10980), [katello#11701](https://github.com/Katello/katello/pull/11701) | **CRITICAL (indirect)**         |
| Burst control        | [foremanctl#495](https://github.com/theforeman/foremanctl/pull/495)                                                                                                                           | **HIGH** (re-tune ×5 under PQC) |


**Still not covered by this stack:** client→Satellite initcwnd (§1), CDN `cdn.rb`, smart-proxy RHSM forwarding pool, CP pool/async (Candlepin).

**Applicable JIRAs:** [SAT-43394](https://redhat.atlassian.net/browse/SAT-43394), SAT-35690; link registration SAT tickets (SAT-43940, SAT-44043, SAT-44963, etc.) in test reports.

---

## 5. Certificate chain optimization

**Actions:**

- Keep chains **short** (leaf + CA, avoid unnecessary intermediates).
- Target total auth data **<9 KB** where feasible.
- Evaluate **TLS certificate compression** (RFC 8879) if stack supports it.
- Plan RSA retirement to end **dual-cert** transitional overhead.

**Applicable:** SAT-36257, SAT-43393

---

## 6. Performance test suite (satperf)

**Gap:** No dedicated SAT-35690 card for PQC benchmarking.

**Action:** Execute [pqc-phased-perf-test-plan.md](./pqc-phased-perf-test-plan.md):

- Baseline vs `DEFAULT:PQ` vs full ML-DSA
- **Good** and **lossy** network profiles (Capsule scenarios)
- Report against [pqc-performance-slos.md](./pqc-performance-slos.md)

**Proposed JIRA:** “PQC performance benchmark suite” under SAT-35690.

---

## 7. Gradual crypto-policy rollout

**Phasing:**

1. **DEFAULT:PQ** — hybrid; ML-KEM KEX (modest overhead) without mandating ML-DSA certs everywhere.
2. Full PQC policy — ML-DSA certs when clients and Satellite ready.

Aligns with [SAT-36255](https://redhat.atlassian.net/browse/SAT-36255) Phase 1 / Phase 2.

---

## SSH / REX ([SAT-43331](https://redhat.atlassian.net/browse/SAT-43331))


| Item                                                       | Note                                                                                                           |
| ---------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| ML-KEM KEX                                                 | Generally **faster** than classical ([RHELBU-3378](https://redhat.atlassian.net/browse/RHELBU-3378))           |
| ML-DSA host keys                                           | Larger — modest session setup cost                                                                             |
| [SAT-45177](https://redhat.atlassian.net/browse/SAT-45177) | System OpenSSH vs net-ssh — may **improve** perf but **PR #10626** regression risk on image-based provisioning |


---

## Ruby gem replacements


| Ticket    | Change                  | Perf expectation                        |
| --------- | ----------------------- | --------------------------------------- |
| SAT-45175 | sshkey → OpenSSH/libssh | Possible improvement                    |
| SAT-45177 | net-ssh → system SSH    | Possible improvement; test provisioning |
| SAT-45196 | net-scp → SFTP          | Architecture change                     |


---

## Container crypto-policies ([SAT-38913](https://redhat.atlassian.net/browse/SAT-38913), [SAT-44657](https://redhat.atlassian.net/browse/SAT-44657))

- Runtime policy prep container: **one-time** startup cost at deploy/restart.
- Build-time per-policy images: faster start, less agility.

---

## Program gaps to close


| Gap                                                                          | Recommendation                                                                                        |
| ---------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| No perf test card                                                            | Create under SAT-35690 → satperf plan                                                                 |
| SAT-38970 still New                                                          | Prioritize; use matrix not 70 stories                                                                 |
| No initcwnd installer story                                                  | Add under SAT-36257                                                                                   |
| [CANDLEPIN-1162](https://redhat.atlassian.net/browse/CANDLEPIN-1162) Backlog | Prioritize for SAT-43401                                                                              |
| Disconnected internal TLS                                                    | Confirm SAT-36257 covers Capsule-only PQC (SAT-43401 excludes disconnected **CDN**)                   |
| Registration stack not linked to SAT-35690                                   | Document dependency — [pqc-registration-pr-stack-impact.md](./pqc-registration-pr-stack-impact.md)    |
| Smart-proxy RHSM forward pooling                                             | Extend Katello pooling pattern to capsule→Satellite `/rhsm/`* proxy                                   |
| CDN `cdn.rb` still RestClient                                                | Extend Faraday persistent pattern ([#11754](https://github.com/Katello/katello/pull/11754) follow-on) |


