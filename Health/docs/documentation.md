A&E Documentation

- [01 - The Problem](#01---the-problem)
- [02 - Data & Scope](#02---data--scope)
- [03 - Data Quality Investigation](#03---data-quality-investigation)
- [04 - Organisation Register](#04---organisation-register)
- [05 - Data Model](#05---data-model)
- [06 - Measure Catalog](#06---measure-catalog)
- [07 - Dashboard Architecture](#07---dashboard-architecture)
- [08 - Key Findings](#08---key-findings)
- [09 - Known Limitations](#09---known-limitations)

# Technical Documentation

NHS England A&E 4-Hour Performance Dashboard — the full process behind the Executive Overview and Trust Drill-Down pages, including every data problem found along the way.

40 monthly source files

214 organisations profiled

3 trust mergers confirmed

6 data anomalies caught


## 01 - The Problem

A Regional Director of Urgent & Emergency Care needs to decide which trusts in their region need support to reach the national 4-hour A&E target. A regional average close to target can hide a wide spread — some trusts comfortably ahead, others badly behind — and without knowing which trusts are pulling the average down, support gets spread evenly instead of where it's needed.

The dashboard ranks trusts by **patients short of target**, not raw percentage — a 5-point miss at a 30,000-attendance trust represents more unmet need than a 10-point miss at a 3,000-attendance trust, and a percentage-only ranking gets that backwards.



## 02 - Data & Scope

Source: [NHS England A&E Attendances and Waiting Times](https://www.england.nhs.uk/statistics/statistical-work-areas/ae-waiting-times-and-activity/) — 40 monthly CSVs, April 2023 to July 2026, no month column in the raw files (derived from each file's name during import).

**Grain:** one row = one organisation, one month. **Scope:** only organisations with nonzero Type 1 (major A&E) attendance in at least one month are included — a filter applied at Org Code level after combining all files, not per-file. This is a stated, data-inferred assumption, not verified against an official ODS provider-type list.

**Target source:** 76% (FY2023/24), 78% (FY2024/25 & FY2025/26, the latter independently verified against primary NHS planning guidance), 82% (FY2026/27 onward) — all sourced from NHS England's own fiscal-year planning documents, not secondary reporting.



## 03 - Data Quality Investigation

Every anomaly below was traced to a specific cause and either fixed or documented as an explained limitation — none were assumed or silently dropped.

#### Hidden national-aggregate row critical

One row carried a national monthly total (1,466,119 Type 1 attendances) disguised as an ordinary organisation row, inflating every descriptive statistic. Found via Excel descriptive stats — a single-trust maximum that size is implausible. Removed; post-removal maximum dropped to a realistic 36,434.

#### Inconsistent footer rows

Every file ends with a "Total" row, but casing/spacing varies across five distinct forms (`TOTAL`, `Total`, `TOTAl`, `"Total "`, `"TOTAL "`) — roughly splitting before/after mid-2024, suggesting an NHS export format change. Fixed with a trim+uppercase filter rather than an exact-match list, so it survives any further casing variant.

#### September 2024 schema drift

One file carries six extra blank trailing columns and a stray header named "a." Verified empty in every row before being safely dropped — combine logic keyed by column name, not position, so the extra columns never shifted anything else.

#### RK9 — false 100% performance, April & May 2023

Zero recorded breaches against a normal \~8,000 monthly attendance volume, two consecutive months. Confirmed as a genuine source submission error via both a Power BI table and an independent raw-CSV check. Documented and excluded from that trust's trend narrative — not corrected in the source.

#### NDA57 — impossible admission value

Shows 1 admission against 0 attendances in April 2023. Traced to Haslemere Minor Injuries Unit, a Type 3 facility with no emergency admission pathway by design. Out of scope (zero Type 1 activity throughout its history) so it does not affect any measure — logged as a documented, uncorrected source anomaly.



## 04 - Organisation Register

Of 214 distinct organisations, 34 had incomplete month coverage. Each was investigated individually by name, not assumed from the gap pattern alone.

| Predecessor | Successor | Date | Verification |
| --- | --- | --- | --- |
| `RAP` North Middlesex | `RAL` Royal Free London | 1 Jan 2025 | Step-jump confirmed in successor's chart |
| `RVJ` North Bristol | `RA7` Bristol NHS FT | 1 Jul 2026 | Step-jump confirmed in successor's chart |
| `RVY` Southport & Ormskirk | `RBN` Mersey & West Lancs | 1 Jul 2023 | Exit window matches dissolution date exactly |

The remaining \~31 discontinuous codes were checked by organisation name and confirmed to be GP out-of-hours services, walk-in centres, and urgent treatment centres — minor-care providers outside the dashboard's Type 1 scope, not mergers or errors.

Three predecessor→successor pairings were initially hypothesised purely from date proximity and disproven once real organisation names were checked — a standing reminder that timing coincidence is not evidence.



## 05 - Data Model

- **Fact table** — one row per in-scope organisation per month, all attendance/breach/admission measures.
- **Org Code dimension** — Org Code only, deliberately with no name attached. Organisation names are not stable (two trusts gained "Teaching" status mid-series), so names stay in the fact table as historical, point-in-time attributes rather than being forced into a one-name-per-code lookup.
- **dim_calendar** — independently generated daily calendar (not derived from which months exist in the data), April 2023–July 2026, with fiscal-year columns throughout (NHS runs April–March). Marked as the model's official Date table.
- Relationships: single-direction, many-to-one, dimension → fact. No ambiguous paths.



## 06 - Measure Catalog

Every measure was validated by hand against the raw source CSVs before being trusted — no measure went into the dashboard unverified.

| Measure | What it does | Validated |
| --- | --- | --- |
| `Type 1 4-Hour Performance %` | Core headline rate | RA7 Apr-23 = 66.25% match |
| `Dynamic Target %` | Applicable NHS target by fiscal year | 4 values, each independently sourced |
| `Variance` | Performance − Target, in points | RA7 Apr-23 = −9.8pp match |
| `Gap in Patients` | Patients short of target, by volume | RRK Jul-26 = 7,314 match |
| `Type 1 Performance YoY` | Change vs. same month last year | RA7 Apr-24 vs Apr-23 = −1.7pp match |
| `Type 1 12-Month Rolling %` | Trailing 12-month rate, totals-weighted | RA7 Mar-24 = 63.3% match |
| `Admission Conversion Rate %` | Share of attendances resulting in admission | RA7 Apr-24 = 33.9% match |
| `Type 1 Status` | Distinguishes no-data from zero-activity months | Logic-tested on known merger cases |

`Gap in Patients` replaced an earlier Variance × Volume approach, rejected for producing a unit with no real meaning. The chosen formula returns an actual patient count and sorts correctly without extra handling for over-performing trusts.



## 07 - Dashboard Architecture

Two pages, each answering one question — deliberately not combined into one dense view, given the audience (a Regional Director, reviewing quarterly) is a strategic decision-maker, not a day-to-day analyst.

#### Page 1 — Executive Overview

"Where do I focus right now?" Four KPI cards, a trust table ranked by Gap in Patients, a month/region filter defaulting to the latest complete month, and a plain-language insight sentence stating the regional picture.

#### Page 2 — Trust Drill-Down

"Is this a new problem or a persistent one?" Reached by drilling into any trust row. Full 40-month trend against the target line, a patient-gap column chart, and a clickable period card showing exact figures for whichever month is selected.



## 08 - Key Findings

1. **Not every low performer has the same story.** One trust has run flat, underperforming for the full 40 months — its target moved, not its performance. Another trust genuinely improved for two years, then declined sharply. Treating both as "below target" would miss that they need different interventions entirely.
2. **Percentage rankings and volume-weighted rankings disagree.** The trust with the single worst percentage miss is rarely the trust with the most patients affected — ranking by raw percentage alone would have sent support to the wrong places.
3. **The best performer by percentage isn't the best performer by scale.** The dashboard highlights the trust handling the largest patient volume while still beating target, not the smallest trust with the highest percentage.
4. **Winter pressure is real but not uniform.** A seasonal dip (roughly November–January) appears in every year of data, but its depth and shape vary meaningfully year to year — flagged directly in the dashboard so a month-to-month comparison isn't misread as a new problem.



## 09 -Known Limitations

- One merger (RAP → RAL) is validated at chart level only, not yet with a row-level raw-CSV spot check like the other two mergers received.
- "How does this month compare to last winter?" isn't answerable at a glance — it requires scrolling the trend chart manually.
- Row-Level Security and mobile layout are explicitly out of scope — single fictional audience, portfolio deployment via GitHub rather than Power BI Service.
- 14 raw columns (secondary attendance/admission categories, long-wait indicators) are present in the model but unused by any current measure — kept rather than removed, to avoid a one-way Power Query change for a possible future extension.

> Built by Japhet Olusegun. Data: NHS England, public and organisation-level only. Full source in the project repository.
