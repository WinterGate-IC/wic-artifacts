
                        WINTERGATEIC — OFFICIAL HOTFIX DOCUMENT


- Document ID:        WG-HF-2026-0916-R26
- Classification:     INTERNAL — ENGINEERING
- Date Issued:        2026-09-16
- Time Issued:        00:00 UTC (round closeout)
- Issued By:          WinterGateIC Engineering
- Status:             SHIPPED — byte-verified
- Distribution:       Internal engineering + operations

--------------------------------------------------------------------------------
1. SUMMARY
--------------------------------------------------------------------------------

Round 26 closes out a systemic edit-pipeline defect and ships two UI
corrections across both hubs. Two prior bug claims are formally retracted
based on byte-level evidence. Two items remain explicitly open.

All shipped claims in this document are byte-verified against the served
tree. No claim is made that was not confirmed by served-http status,
content hash, or syntax check.

--------------------------------------------------------------------------------
2. ROOT CAUSE FIXED — EDIT GUARD FALSE-REVERT
--------------------------------------------------------------------------------

Component:      edit_guard.py (internal edit validation layer)
Severity:       Systemic — blocked all HTML edits
Status:         FIXED

2.1 DEFECT DESCRIPTION

The edit guard validated every edited file by running a JavaScript syntax
check (node --check) against the file contents. For HTML files, this check
was executed against the raw HTML source as if it were JavaScript. Raw HTML
is not valid JavaScript, so the check failed on every HTML edit, and the
guard auto-reverted the change.

Net effect: every attempted HTML edit was silently rolled back.

2.2 FIX APPLIED

The guard now extracts only inline <script> blocks from HTML files and runs
the syntax check against those blocks, not against the raw HTML.

2.3 VERIFICATION

Proof of fix: 17 successful HTML tab edits in this round, all served with
HTTP 200. Prior to the fix, zero HTML edits were landing.

--------------------------------------------------------------------------------
3. SHIPPED CHANGES — BYTE-VERIFIED
--------------------------------------------------------------------------------

3.1 REGISTER CHIP ROW — SECURITY HUB

Component:      renderCampFeedEvent (feed cards)
File:           [security.js — path redacted]
Hash:           sha16 2193a1bad29036c5
Syntax:         node --check OK
Served:         /security/tabs/feed.html -> HTTP 200
Chips:          REGISTER / PREMIUM+ / L7
Backend fields: register_arc, premium_locked, l7_locked

3.2 REGISTER CHIP ROW — STYXNET HUB

Component:      index.html
Location:       line 2124
Change:         if(_rgchips)h+=_rgchips;
Syntax:         node --check OK
Served:         HTTP 200

3.3 REDUNDANT FLOAT BADGE REMOVED

Component:      "full hub" float badge
Scope:          all 17 tabs across both hubs
Reason:         pointed at the page the user was already on
Guard journal:  [OK] R26-REMOVE-FULLHUB
Delta:          approximately -270 bytes per page
Served:         HTTP 200 on all 17 tabs
Nav check:      legitimate Security hub nav link intact 17/17

--------------------------------------------------------------------------------
4. RETRACTIONS — PRIOR CLAIMS NOT SUPPORTED BY EVIDENCE
--------------------------------------------------------------------------------

4.1 RETRACTED: "17 TABS RE-DOWNLOAD 357KB EACH"

Correct finding: Caddy sends ETag + Last-Modified. Client requests include
If-None-Match. Server responds with 304 Not Modified and zero body. The 17
tabs do not re-download the payload.

Actual cost: per-tab parse only. Modest. Not the defect previously described.

4.2 RETRACTED: "17 RENDER FUNCTIONS IN SECURITY.JS"

Correct finding: security.js contains exactly 2 render functions:
renderCampFeedEvent and renderOvermindFeed.

The "per-section painters" framing was inflated. It is retracted.

--------------------------------------------------------------------------------
5. EXPLICITLY NOT DONE — NO CLAIM MADE
--------------------------------------------------------------------------------

5.1 iDRIVE / LEGAL-REPORT PUSH — NOT RE-VERIFIED THIS SESSION
Reason: helper API mismatch encountered
Status: OPEN

5.2 PER-TAB SLIM-BOOT SPLIT — NOT BUILT
Reason: not needed once 304-cache behavior was understood
Policy: no fake split will be shipped to appear productive
Status: DEFERRED (correctly)

--------------------------------------------------------------------------------
6. VERIFICATION ARTIFACTS
--------------------------------------------------------------------------------

Content hash:       [redacted — internal reference only]
Syntax check:       node --check OK
HTTP verification:  200 on all served endpoints
Guard journal:      [OK] R26-REMOVE-FULLHUB
Served tree:        confirmed authoritative

--------------------------------------------------------------------------------
7. OPERATIONAL NOTES
--------------------------------------------------------------------------------

- Sensitive identifiers (file paths, hashes, credentials, host identifiers)
  are redacted from this copy.
- This document reflects the state of the system at the round closeout time
  stated above. Any later changes supersede this document.
- The retractions in Section 4 are provided to keep the engineering ledger
  accurate. Shipped items are trusted because unsupported items are marked.

--------------------------------------------------------------------------------
8. CONCLUSION
--------------------------------------------------------------------------------

Round 26 is closed.

Shipped:      edit guard root cause fix, chips row (both hubs), float badge
              removal (17 tabs)
Retracted:    2 prior claims killed by evidence
Open:         2 items (no claim made)
Fabricated:   0 items

The edit guard fix unblocks the HTML edit pipeline entirely. All other
changes are byte-verified and served.

--------------------------------------------------------------------------------
END OF DOCUMENT
--------------------------------------------------------------------------------
