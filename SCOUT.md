# SCOUT.md — Texas plumbing/HVAC off-market scout (runbook)

Portable: hand this file to any environment that can run Python 3 and reach the hosts below.
Status of last attempt (2026-10-06, Claude Code cloud session): **NOT RUN** — see "Capability check".

## 0. Capability check (do first, stop if any fail)
Required: run Python 3.10+, stream multi-hundred-MB downloads, call HTTPS APIs, write files.
Required hosts (allow-list them): `data.sba.gov`, `data.texas.gov`, `www.askebsa.dol.gov`
(plus whatever host the SBA CSV resolves to, e.g. an S3/CDN domain — read it from the dataset page).

2026-10-06 result in the cloud session: code execution OK; all three hosts returned
`403 CONNECT tunnel failed` from the egress proxy (org network policy). Fix: Environment settings →
Network access → Custom, add the three hosts under Allowed domains (keep default package managers).
Do not work around the policy. Log each unavailable source once in `source-ledger.csv` (status=unavailable).

## 1. Outputs (all in this folder)
- `scout-tracker.csv` — one row per business, permanent IDs `biz-001`…
- `scout-results.html` — readable results page
- `source-ledger.csv` — url, retrieval_date, source_id (dataset id / resource), reporting_period, sha256, rows, status, notes
- `data/raw/` — raw inputs or `download-log.csv` (url, headers, bytes, sha256, timestamp) if raw not kept
- Reruns: load existing tracker, reuse IDs by match key, never duplicate; preserve `outreach_status`, scores, and any extra columns.

## 2. Step A — SBA PPP (≥ $150K file)
1. Find the current "PPP FOIA data $150,000 plus" CSV URL via the SBA dataset page (don't hardcode).
2. Stream to disk (`requests.get(stream=True)`, 1 MB chunks), compute sha256 while writing. Detect encoding
   (try utf-8-sig, fall back cp1252/latin-1); record which was used.
3. Filter `BorrowerState == "TX"`, `NAICSCode == 238220`, `CurrentApprovalAmount >= 150000` (cast numeric).
4. Keep every loan separate; keep `LoanNumber`. Report: source rows read, matching loans, business groups.
5. Business grouping key: normalized legal name (upper, strip punctuation/LLC/INC/CO suffix) + normalized
   address (street+city+zip5). Merge DBA-only variants only with documented DBA evidence (e.g. `BusinessName`
   vs `Franchisor`/DBA field, or Texas SOS assumed-name record) — store the evidence text. Otherwise keep separate.
6. Per business: loan_count, loan_ids, total/first/last approved amount, `JobsReported` (historical; keep separate
   from any headcount), date approved, lender.
7. Payroll proxy: `payroll_proxy = loan/2.5*12` ONLY for verified standard, uncapped 2.5x payroll loans
   (first-draw, 2.5x payroll month rule, not capped at $20.8K/mo, not seasonal 3.5x, not NAICS 72 at 3.5x).
   Else blank + `payroll_proxy_basis="not verified"`. Never infer revenue or purchase price.

## 3. Step B — Texas Active Franchise Taxpayers (data.texas.gov `9cir-efmm`)
Use SODA API (`https://data.texas.gov/resource/9cir-efmm.json`, `$limit`/`$offset` paging, optional app token).
Capture SOS charter date or verified original trade-license date; save the date, its meaning
(charter vs license vs permit), source, match_confidence (high/medium/low).
Match only with name AND address corroboration (or documented DBA). Ambiguous → `review`.
Flag `age_30plus=true` if the date is ≥ 30 years before run date. Record age ≠ owner age ≠ sale intent.

## 4. Step C — DOL Form 5500 / 5500-SF (latest national files)
From the EBSA FOIA datasets page, pick the latest plan year with both `F_5500` and `F_5500_SF` (national, not state).
Sponsors may be out of state: match against the full national file, not just TX.
Require retirement benefit codes (Form 5500 line 8a features: `1*`, `2*`/`3*` pension codes; SF: `type_pension_bnft_code`).
Match sponsor by independently known EIN, else name+address. Keep: plan period start/end, filing date,
active participants (`TOT_ACTIVE_PARTCP_CNT`/SF equivalent), plan ID (PN), evidence, confidence.
Never sum overlapping plans (same sponsor, overlapping periods, or multiple PNs) — list them separately.
Participants ≠ current company headcount. No filing found → `unknown`, business stays eligible.
Verify exact column names against the downloaded layout/data dictionary; they change by year.

## 5. Tracker schema (prompt-1 owned columns)
Identity: `biz_id, legal_name, dba, address, city, zip, match_key`
Loans: `loan_count, loan_numbers, total_approved, loan_dates, lender, jobs_reported_historical, payroll_proxy, payroll_proxy_basis`
Registration: `sos_date, sos_date_meaning, sos_source, sos_match_confidence, years_old, age_30plus`
Pension: `plan_period, filing_date, active_participants, plan_id, pension_codes, pension_match_evidence, pension_confidence`
Workflow: `eligibility_status, eligibility_reason, outreach_status, score, notes`

Eligibility: `ready` = verified 30+ yrs; `review` = uncertain/ambiguous joins; `closed` = clear exclusion
(e.g. out of business evidence, not TX plumbing/HVAC). `outreach_status="ready"` only on newly created IDs;
never overwrite existing values.

## 6. Verification before reporting
- Unique `biz_id`; no duplicate match_keys; loan count in tracker == filtered loan count.
- Full coverage: rows read == rows in file; Texas rows paged to exhaustion (compare to `$select=count(*)`).
- Rerun on same inputs → zero new IDs, zero changed preserved columns.
- Results page: counts by status, unknown counts, unfinished work, then **five real businesses** with clickable
  evidence links (SBA row, SOS/franchise record, DOL filing). Pick only from verified output.
