# SAT-38970 Restructure Proposal

## Problem

[SAT-38970](https://redhat.atlassian.net/browse/SAT-38970) (*SPIKE: Investigate PQC KEX in Sat↔Host TLS*) spawned **~70 sub-tasks** (SAT-38971 through SAT-39046), each named `PQC KEX TLS: <Component>`. As of export:

- **69** sub-tasks: **New**
- **1** sub-task closed: [SAT-39004](https://redhat.atlassian.net/browse/SAT-39004) (Remote Execution)

Treating each sub-task as independent delivery work creates:

- Sprint planning overhead (70 items)
- False sense of progress (closing verification ≠ shipping feature)
- Duplication with feature epics under [SAT-36257](https://redhat.atlassian.net/browse/SAT-36257)

The spike value is a **compatibility and performance test matrix**, not 70 engineering stories.

---

## Recommended structure

### Keep (1 item)

| Key | Type | Purpose |
|-----|------|---------|
| SAT-38970 | Spike → **Test Plan** (or Epic) | Owns the matrix definition, exit criteria, and sign-off |

Update SAT-38970 description to state: *"Sub-tasks are verification rows, not implementation work unless tagged `code-change-required`."*

### Create (2 epics)

| Proposed epic | Replaces | Scope |
|---------------|----------|--------|
| **SAT-NEW: PQC TLS compatibility matrix (QE)** | SAT-38971–39046 (verification rows) | Functional: each component negotiates ML-KEM when policy requires; document pass/fail |
| **SAT-NEW: PQC TLS performance baselines (PerfScale)** | Perf slice of matrix | SLO-10 + hot paths: Registration, API, Pulp, Candlepin, Capsule sync |

Link both epics to SAT-36257 and SAT-35690.

### Cancel or bulk-close sub-tasks

For each SAT-3897x sub-task:

1. **If no code change required:** Close as *"Covered by QE matrix row"* and link to QE epic test case ID.
2. **If code change required:** Keep open but **re-parent** to the correct feature epic (e.g. SAT-38910 for Candlepin, SAT-43394 for Registration).

---

## Matrix template (replaces 70 tickets)

Store in Confluence or `satperf/docs/pqc/pqc-tls-component-matrix.csv`:

| Row ID | Component | Owner team | TLS client stack | PQC KEX test | ML-DSA cert test | Code change? | Feature epic | satperf probe |
|--------|-----------|------------|------------------|--------------|------------------|--------------|--------------|---------------|
| M-01 | Registration | Katello | OpenSSL (curl) | | | TBD | SAT-43394 | concurrent_execution |
| M-02 | Candlepin API | Candlepin | Java/tomcat-native | | | Maybe | SAT-38910 | manifest_import |
| M-03 | Pulp API | Pulp | Python/OpenSSL | | | TBD | SAT-36256 | repo_sync |
| ... | ... | ... | ... | ... | ... | ... | ... | ... |

Populate all 70 rows from existing sub-task summaries (one-time script from `pqc-issue-hierarchy.csv`).

---

## Classification of existing sub-tasks

### Likely verification-only (close → matrix row)

Most UI/plugin areas with no custom TLS stack:

- Dashboard, Branding, Navigation, Search, Audit Log, Organizations, Parameters, Notifications, Logging, Settings, Hooks, Localization, Reporting, Tasks, Users/Roles, etc.

### Likely need feature epic linkage (keep, re-parent)

| Sub-task range | Component | Target epic |
|----------------|-----------|-------------|
| SAT-38977, 38976 | Candlepin, Subscription Management | SAT-38910, SAT-43394 |
| SAT-38981 | Registration | SAT-43394 |
| SAT-38988, 38993, 38989 | Repositories, Pulp, Capsule Content | SAT-36256, SAT-43401 |
| SAT-38996 | Ansible Remote Execution | SAT-43331 |
| SAT-39004 | Remote Execution | SAT-43331 (already closed) |
| SAT-38999 | HTTP Proxy | SAT-43394 (pulp-certguard / mTLS) |
| SAT-39012 | Foreman Proxy | SAT-38907 |

### Spike outcomes already elsewhere

- [SAT-43398](https://redhat.atlassian.net/browse/SAT-43398) — Foreman-Candlepin communication investigation (duplicate theme with SAT-38910)
- Merge 43398 findings into SAT-38910 and close 43398 as duplicate

---

## JIRA actions (for PM/EM)

1. Comment on SAT-38970 with link to this proposal.
2. Create two epics (QE matrix + Perf baselines).
3. Bulk transition SAT-38971–39046: Cancel with reason *"Superseded by matrix epic"* OR link as "tests" issue type if available.
4. Add label `pqc-matrix-row` to any sub-task kept for tracking.
5. Add SAT-35690 acceptance criterion: *"SAT-38970 matrix ≥95% rows pass on RC build"*.

---

## Success criteria for restructure

- [ ] Open SAT-3897x count drops from ~69 to &lt;10 (code-change only)
- [ ] Single CSV/matrix is the source of truth for TLS PQC coverage
- [ ] PerfScale owns SLO-10 measurement, QE owns functional matrix
- [ ] No duplicate Foreman-Candlepin spikes (38910 vs 43398)
