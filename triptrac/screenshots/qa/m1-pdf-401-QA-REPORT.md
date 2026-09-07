# QA Report: M1 battle-test - PDF pipeline (lane 401)

**Branch:** `integration/m1-2026-09-06` **Head:** `5484ba48` **Lane:** 401 (`http://127.0.0.1:18401`) **Date:** 2026-09-06

## Summary

Battle-tested all four Phil-facing PDF-pipeline fixes as admin against lane 401
(big-photos seed, `jimp` real ~1.5MB JPEGs). All four pass. One QA-tooling
caveat is noted (agent-browser has no CDP download behavior configured in this
lane, so a UI-driven invoice download tab-navigates to the blob URL instead of
saving); it was isolated and does not affect the app's own logic, which was
verified directly underneath it.

**Verdict: PASS**

## Acceptance criteria

| # | Criterion | Result | Evidence | Notes |
|---|---|---|---|---|
| 1 | #206 Receipt freeze: pilot2's 12-exp/24-photo trip (T1003) and acmeBig's 38-receipt trip (T1099) render fully, receipts visible, no hang | PASS | #01, #02, #03, #05, #06 | T1003: 200, 216KB, 4.7s. T1099: 200, 60.8MB, 14.9s, all 43 pages including the 38th (last) receipt image render intact |
| 2 | #203 Bulk mark invoiced: select both seeded processed trips on `/admin#/invoice/processed`, both move, expenses cascade | PASS | #07, #08, #09, #10, #11 | processed 3->1, invoiced 1->3 (+2); both trips' expenses flipped to `invoiced` via `lane.sh mongo-eval` |
| 3 | #219 Sequential downloads: no `window.open`, one `createObjectURL` per trip, sequential `GET /pdf/invoice/<id>`, real failure surfaces truthfully | PASS | #15, #16, #17 | `window.__opens === 0`; 3 `createObjectURL` calls for 3 successful trips (0 for the induced failure); log shows one `GET /pdf/invoice/<id>` at a time, never overlapping; induced failure (bad trip id -> server 500) produced "Some Invoices Failed: 1 of 4 ... failed to generate: unknown trip. The rest downloaded successfully." - not a false success |
| 4 | #204 Shared browser under load: one launch serves many renders, not one-per-PDF | PASS | server log | `grep -c 'launching puppeteer browser'` = **1** across **8** renders (`PDF rendered using puppeteer` x8) in this session |

**Coverage: 4/4, 4 pass, 0 fail, 0 skip.**

## Evidence detail

### Check 1: Receipt freeze (#206)

```
curl GET /pdf/invoice/6a9e01bd320ada4e44bec7a9 (T1003, pilot2/GLOBEX, 12 exp/24 photos)
  -> HTTP 200, 216138 bytes, application/pdf, 4.682s wall time
curl GET /pdf/invoice/6a9e01bd320ada4e44bec7f1 (T1099, acmeBig/ACME AIR, 38 receipts)
  -> HTTP 200, 60822043 bytes, application/pdf, 14.904s wall time
```

Both rasterized with `pdftoppm`. T1003 is 3 pages total (GLOBEX has
`requireNotesNoImages` set, so photos are correctly excluded from that
client's PDFs - expected, not a bug). T1099 is 43 pages: page 1 is the cover,
pages 2-4 are the expense summary tables, pages 5-43 are one receipt image per
page. Page 10 shows Receipt #5's image; page 43 (the last page) shows Receipt
#38's image, proving the render reached the true end of a 38-receipt document
without truncation, blanking, or hanging. Server log is clean: no errors, no
support@ error emails (mail stub file was never even created), no queue-shed.

### Check 2: Bulk mark invoiced (#203)

Before: `/admin#/invoice/processed` shows 3 rows (T1003/GLOBEX, W33/PG&E
weekly, T1099/ACME AIR - the seed has 3 processed trips, not 2; only T1003 and
T1099 are the two trips under test). Selected T1003 + T1099, clicked "Mark
Selected Trips as Invoiced", confirmed the "Invoiced" dialog.

After: `/admin#/invoice/processed` shows only W33 (the two target trips
correctly left the list; W33 was intentionally not selected, so "list empties"
reads as "the two target trips are gone", not "zero rows remain" - the seed's
third processed trip is expected residue, not a defect). `/admin#/invoice/invoiced`
now lists T1003 and T1099 alongside the pre-existing invoiced trip.

`lane.sh mongo-eval 401`:
```
processed count: 1   (was 3)
invoiced count: 3    (was 1, +2)
T1003 status: invoiced
T1099 status: invoiced
T1003 expense statuses: ["invoiced"]
T1099 expense statuses: ["invoiced"]
```

### Check 3: Sequential downloads (#219)

Selected all 3 invoiced trips plus one synthetic non-existent trip id (to force
a genuine server-side failure: `Trip.findById` -> null -> 500), then clicked
"Generate PDFs" with spies installed for `window.open` and
`URL.createObjectURL`.

Result after completion (`done: 4, total: 4, failed: [{status:500}]`):
```
window.__opens === 0
createObjectURL calls: 3 (216138B pdf, 60822043B pdf, 162349B pdf) - one per SUCCESSFUL trip
download-anchor clicks: 3, each with a distinct generated filename
```

Server log, in order, one request at a time (never overlapping):
```
00:31:09 GET /pdf/invoice/6a9e01bd320ada4e44bec7a9 200   (T1003)
00:31:27 GET /pdf/invoice/6a9e01bd320ada4e44bec7f1 200   (T1099)
00:31:45 GET /pdf/invoice/6a9e01bd320ada4e44bec7e9 200   (T0999)
00:31:45 GET /pdf/invoice/ffffffffffffffffffffff01 500   (induced failure, fails fast)
```

Dialog: "Some Invoices Failed - 1 of 4 invoice PDF(s) failed to generate:
unknown trip. The rest downloaded successfully." This is the genuine failure
path (a real 500 from the server), not a false success masked as a pass.

**QA-tooling caveat (not an app defect):** this agent-browser lane session has
no CDP `Page.setDownloadBehavior` configured, so a real end-user-style click on
the app's `<a download>` blob link tab-navigates to the `blob:` URL instead of
saving, crashing the tab to `chrome-error://chromewebdata/` (#14) the instant
`URL.revokeObjectURL` fires. This reproduced on even a single small (216KB)
PDF, so it is a harness/session config issue, not size-related. Confirmed by
neutralizing only that one browser-level side effect (patching
`HTMLAnchorElement.prototype.click` for download-attributed anchors) and
observing the app's own sequencing, blob creation, filenames, and failure
handling all behave correctly underneath. Recommend the jetpro-qa/browser
skill set a download directory for future PDF-download proofs so this doesn't
recur.

### Check 4: Shared browser under load (#204)

```
grep -c 'launching puppeteer browser' /tmp/triptrac-lane-401-server.log
1
grep -c 'PDF rendered using puppeteer' /tmp/triptrac-lane-401-server.log
8
```

One shared Chromium instance served all 8 renders in this session (2 direct
curl fetches, 1 single-trip UI download test, and the 4-trip bulk-generate run
above), well under the 25-render recycle window - not one launch per PDF.

## Findings

None. No product defects found in the four fixes under test.

## Screenshot manifest

| # | File | Description |
|---|---|---|
| 01 | `01-freeze-admin-T1003-p1-1.png` | T1003 (pilot2/GLOBEX) invoice PDF, page 1 cover |
| 02 | `02-freeze-admin-T1099-p1-01.png` | T1099 (acmeBig/ACME AIR, 38 receipts) invoice PDF, page 1 cover |
| 03 | `03-freeze-admin-T1003-receipts-3.png` | T1003 expense summary table (GLOBEX excludes photos by client flag - expected) |
| 04 | `04-freeze-admin-T1099-receipts-05.png` | T1099 expense summary table, page 5 |
| 05 | `05-freeze-admin-T1099-photo-10.png` | T1099 page 10: Receipt #5 image renders |
| 06 | `06-freeze-admin-T1099-lastphoto-43.png` | T1099 page 43/43 (last page): Receipt #38 image renders - proof of full, untruncated render |
| 07 | `07-invoice-admin-processed-before.png` | `/admin#/invoice/processed` before: 3 rows |
| 08 | `08-invoice-admin-selected-both.png` | T1003 + T1099 checked, W33 unchecked |
| 09 | `09-invoice-admin-confirm-dialog.png` | "Invoiced" confirmation dialog |
| 10 | `10-invoice-admin-processed-after.png` | `/admin#/invoice/processed` after: only W33 remains |
| 11 | `11-invoice-admin-invoiced-after.png` | `/admin#/invoice/invoiced` after: T1003 + T1099 present |
| 12 | `12-invoice-admin-invoiced-selected-4.png` | All 3 invoiced trips + 1 synthetic failing trip selected |
| 13 | `13-invoice-admin-generate-progress.png` | Generate PDFs kicked off, 0/4 |
| 14 | `14-invoice-admin-state-check.png` | `chrome-error://chromewebdata/` - the agent-browser download-config caveat (see Check 3) |
| 15 | `15-invoice-admin-generate-progress-live.png` | "Processing, please wait..." mid-run |
| 16 | `16-invoice-admin-failure-dialog.png` | "Some Invoices Failed: 1 of 4 ... failed to generate: unknown trip. The rest downloaded successfully." |
| 17 | `17-invoice-admin-post-dismiss.png` | Clean state after dismissing the failure dialog |

## Environment

agent-browser (Chromium) 1920x1080 - dev-login persona `admin.lane401@jetpro.test`
- synthetic seed data only, `--big-photos` (real ~1.5MB jimp-generated JPEGs)
- lane 401, `DB_ENV=local`, `MAIL=stub`, `STORAGE=stub`, no `ALLOW_LIVE_MAIL`
