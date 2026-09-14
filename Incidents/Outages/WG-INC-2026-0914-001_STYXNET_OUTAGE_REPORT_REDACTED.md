# INCIDENT REPORT — PUBLIC / REDACTED VERSION

**Incident ID:** WG-INC-2026-0914-001
**Classification:** Public — Redacted
**Date of Incident:** 2026-09-14
**Date of Report:** 2026-09-14
**Status:** RESOLVED — Root cause removed, permanent hardening applied
**Severity:** Service Outage (Site unreachable via CDN — HTTP 523)
**Affected Service:** Public search endpoint and all routes terminating at the edge node
**Affected Hosts:** Edge node and downstream application node (identifiers redacted)
**Downtime Window:** ~17:26 UTC → 22:28 UTC
**Total Duration:** ~5 hours 2 minutes
**Detected By:** Operator (manual check)
**Resolved By:** Operator (console reboot + permanent hardening)
**Data Loss:** None
**Breach:** None
**Attack:** None (confirmed)

---

## 1. EXECUTIVE SUMMARY

On 2026-09-14, the public search endpoint became unreachable, returning a CDN **523 error** ("origin unreachable"). The edge node had run its root filesystem to 100% capacity, stalling all disk writes for the reverse proxy, system journal, and tunnel service, which in turn stopped the node from forwarding any HTTP traffic to the application node.

The root cause was identified as **unbounded log growth** from an internal tiered-response daemon, which had been writing two identical large log files since 2026-07-09 — one via the daemon's own Python file handler, and a second via the systemd stdout mirror.

The incident was **not** an attack. There was no breach. There was no unauthorized access, no unexpected listener, no crypto miner, no crontab injection, no abnormal CPU. All evidence is documented below.

Service was restored after a console reboot of the edge node and removal of the offending log volume. Permanent hardening was applied to prevent recurrence:

- The daemon's file handler was removed entirely (stream handler only)
- Systemd stdout was nulled on the daemon unit
- The system journal was capped
- A logrotate guard was added with `copytruncate` as a defensive fallback

The service is now fully operational. The root cause has been eliminated and cannot recur in its previous form.

---

## 2. IMPACT

### 2.1 User-Facing Impact

| Impact Area | Details |
|---|---|
| Site availability | Public search endpoint returned CDN 523 for ~5 hours |
| API availability | All endpoints behind the edge node were unreachable |
| Search functionality | Fully unavailable during the outage window |
| Static assets | Unavailable (reverse proxy could not serve) |
| Tunnel traffic | Stopped forwarding at ~17:26 UTC |

### 2.2 Internal Impact

| Impact Area | Details |
|---|---|
| Edge node | Disk 100% full; write stalls across all services |
| System journal | Unable to flush; elevated CPU after boot |
| Reverse proxy | Could not write logs, could not serve traffic |
| Tunnel service | Listener open but not forwarding |
| Application node | Unaffected directly; inaccessible from the internet |

### 2.3 Data Impact

**None.** No data was lost. The application node continued to hold all state. No database corruption occurred. No evidence or logs from the application node were lost.

---

## 3. TIMELINE (UTC)

| Time | Event |
|---|---|
| ~17:26 | Last successful request forwarded from the edge node into the application node |
| 22:04 | Operator reports public endpoint returning CDN 523 |
| ~22:15 | Connect work begins. Edge node remote access unresponsive. Standard web port answered SYN but never served HTTP. DNS records still pointed to the edge node (correct). Origin was down. |
| ~22:20 | Operator reboots the edge node from the provider console |
| 22:28 | Verified public endpoint → HTTP 200 end-to-end (CDN edge → edge node → application node) |
| **22:28** | **INCIDENT RESOLVED** |

Post-incident hardening continued after resolution and was completed the same day.

---

## 4. ROOT CAUSE ANALYSIS

### 4.1 Primary Root Cause

The edge node ran disk at **100% capacity** on its root filesystem.

Two identical **large log files** were written by an internal tiered-response daemon:

1. One log written by the daemon's own Python file handler
2. A second identical log written by the systemd unit stdout mirror

### 4.2 Why the Files Grew

The daemon logs **every INFO-level tier event**:

| Tier | Event Type |
|---|---|
| T1 | Monitor |
| T2 | Rate-limit |
| T3 | Tarpit |
| T5 | ASN-block |

Unbounded growth began **2026-07-09** and continued for **67 days** until the filesystem filled.

### 4.3 Failure Cascade

1. Disk reaches 100% → no more writes can succeed
2. Reverse proxy can no longer write logs or session data → stalls
3. System journal cannot flush to disk → backs up
4. Tunnel service cannot forward → stops
5. Logging daemons spin at elevated CPU trying to flush against a full disk
6. CDN edge cannot reach origin → returns **523**
7. Users see "host server is not reachable"

### 4.4 What the Log Content Actually Was

The log content was the daemon doing its normal job — recording **rejected connection attempts** from ordinary internet background scanning.

The scanners were **blocked as designed**. The daemon was **working correctly**. The problem was that the record of that work filled the disk.

---

## 5. ATTACK DETERMINATION — VERDICT: NOT AN ATTACK

### 5.1 Evidence Reviewed

| Check | Result |
|---|---|
| Auth logs for unexpected accepted sessions | **None** — only benign signature negotiation noise |
| Unexpected listeners on the edge node | **None** — only expected services |
| Crypto miners / abnormal CPU | **None** — single sustained consumer was the logging subsystem flushing after boot against a full disk |
| Scheduled task entries | **Clean** — no injected scheduler entries |
| File integrity | **Clean** — no modified binaries, no unusual processes |

### 5.2 Conclusion

> **Plainly: the network scanner noise was filtered and blocked as designed. The outage was caused by self-inflicted log volume filling the boot disk.**

No breach occurred. No unauthorized access occurred. No data exfiltration occurred.

---

## 6. REMEDIATION & FIXES APPLIED

### 6.1 Immediate Fixes

| Action | Result |
|---|---|
| Truncated both log twins | Freed substantial disk space |
| Verified disk usage | Down to normal levels |
| Verified load | Normal |
| Verified daemon log | Flat, no longer growing |

### 6.2 Permanent Hardening Applied

**On the daemon itself:**

- Patched its logging setup to use a stream handler only
- **No file handler anywhere** — the daemon can no longer write to disk directly

**On the systemd unit:**

- Stdout nulled
- Stderr redirected to the journal
- The daemon's output no longer touches the filesystem

**On the system journal:**

- Hard cap applied to total journal size
- Journal is now bounded

**On logrotate (defensive backstop):**

- Guard added with size rotation, retention, compression, and `copytruncate`
- Protects against any future writer even if one is re-added

### 6.3 Architectural Change

**Before:** Edge node wrote unbounded logs locally.

**After:** Edge node is **zero-persist by design**. All significant logging happens on the application node. The system journal on the edge node keeps only a bounded trace.

This is a strictly better architecture than the pre-incident setup.

---

## 7. POST-INCIDENT VERIFICATION

### 7.1 Health Checks

| Check | Status |
|---|---|
| Public endpoint | **HTTP 200** end-to-end |
| CDN edge → edge node → application node | **Verified** |
| Edge node disk | Normal |
| Application node disk | Normal |
| Edge node load | Normal |
| Daemon log | Flat |
| Logrotate guard | Present, active |

### 7.2 Credentials & Recovery

Infrastructure credentials are stored on the application node in a restricted, permission-locked directory, with a mirrored backup in a secure location. Access details are redacted from this public version.

No credentials were rotated as a result of this incident (no breach occurred).

---

## 8. LESSONS LEARNED

### 8.1 Technical Lessons

1. **A defensive system that logs aggressively can kill itself.** Your own record-keeping is a potential denial-of-service vector against your own infrastructure.
2. **Disk-full is a cascade, not a single failure.** Every service that touches disk will fail, often in confusing ways.
3. **Twin log files are easy to miss.** If both the application and systemd capture the same output, you get double the growth for the same event stream.
4. **Bound every writer.** Journals, application loggers, systemd, and logrotate should all have hard limits.
5. **Edge nodes should be zero-persist.** Keep forwarders stateless; keep records on the application tier.

### 8.2 Operational Lessons

1. **A reboot resolves the symptom, not the cause.** The reboot restored service; the truncation and daemon patch fixed the incident.
2. **Post-incident hardening matters more than the fix.** Removing the file handler, capping the journal, and adding logrotate means this incident cannot recur.
3. **Write the report.** Incident reports are the difference between "it happened again" and "it can't happen again."
4. **Do not blame the scanners.** They were rejected. The system did its job. The failure was on our side.

---

## 9. RECURRENCE RISK ASSESSMENT

| Risk | Likelihood After Fix | Notes |
|---|---|---|
| Daemon fills disk again | **Eliminated** | No file handler; systemd stdout nulled |
| Journal fills disk | **Eliminated** | Hard cap applied |
| Some future writer fills disk | **Mitigated** | Logrotate guard present |
| Silent disk growth undetected | **Mitigated** | Zero-persist design + logrotate monitoring |
| Network scanners cause outage | **Not possible** | Scanners were rejected; they were never the cause |

**Overall recurrence risk for this class of incident: negligible.**

---

## 10. ACTION ITEMS

| # | Action | Status |
|---|---|---|
| 1 | Truncate twin log files | Done |
| 2 | Remove file handler from daemon | Done |
| 3 | Null stdout on unit | Done |
| 4 | Cap system journal | Done |
| 5 | Add logrotate guard | Done |
| 6 | Verify end-to-end HTTP 200 | Done |
| 7 | Document incident | Done (this report) |
| 8 | Confirm zero-persist edge design | Done |
| 9 | Confirm application node healthy | Done |

---

## 11. APPENDIX

### 11.1 Affected Paths

Log file paths, daemon source paths, systemd unit paths, journald config paths, logrotate config paths, and credential store paths are **redacted** from this public version. Full paths are retained in the internal, unredacted copy.

### 11.2 Services on the Edge Node

| Service | Role | Status Post-Incident |
|---|---|---|
| Remote access | Administrative access | Healthy |
| Reverse proxy | HTTP/HTTPS termination | Healthy |
| Tunnel service | Forward to application node | Healthy |
| Tiered-response daemon | Response logger | Patched, stream handler only |
| System journal | System log | Capped |

### 11.3 Confirmed Rejected Activity

The daemon was blocking and logging:

- SSH brute-force scanners
- Port scanners
- Tier 1 (monitor) through Tier 5 (ASN-block) events

**All rejected. All logged. That logging is what filled the disk.**

---

## 12. CONCLUSION

On 2026-09-14, the public search endpoint went down for approximately 5 hours due to a **self-inflicted disk-full condition** on the edge node. The cause was **unbounded log growth** from an internal tiered-response daemon — not an attack, not a breach, not a compromise.

Service was restored via console reboot. Permanent hardening was applied: the daemon's file logging was removed, the system journal was capped, and a logrotate guard was added. The edge node is now **zero-persist by design**, with all significant logging performed on the application node.

**No data was lost. No breach occurred. The root cause cannot recur in its previous form.**

The incident is closed.

---

*Report prepared by the operator. Public distribution — sensitive infrastructure details redacted.*
*For any questions, contact the operations team.*

**END OF REPORT**
