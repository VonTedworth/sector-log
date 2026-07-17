# Sector Log — HeliCrew Manager → LogTen Pro

Single-page web app. Upload both HCM exports (Excel + LogTen CSV) from the same date range, review the derived sectors, tap Send — LogTen opens and asks to import. Deploy `index.html` to GitHub Pages alongside the flight card app. Works offline after first load except for the two CDN libraries (PapaParse, SheetJS).

## Why both files

| Data | Excel | CSV |
|---|---|---|
| Zulu times | ✓ | — (local only) |
| Commander name | ✓ | — |
| Grid refs + lat/long | ✓ | partial |
| Unique flight key (unix epoch) | — | ✓ (`Flight #`) |
| Off/On (airborne) times | — | ✓ |

Rows are joined on **date + local Out + local In** (verified unique across 230 real sectors; 229/230 matched, the exception had blank times in one file and is flagged `UNMATCHED`).

## Derivations and assumptions

- **AirMaestro placeholder rows** (`Aircraft ID = AIRMAESTRO`, pre-HCM FTL imports) are dropped.
- **Zulu conversion**: offset taken per-sector from the Excel's own (L)−(Z) columns; midnight rollovers handled by monotonic ordering Out ≤ Off ≤ On ≤ In.
- **Capacity**: every sector defaults to **SIC**. PICUS is a crew-room decision, so it is a one-tap toggle per sector (SIC ⇄ P1u/s). Crew names are unaffected: PIC/P1 crew = commander from the Excel (first non-SELF name), SIC/P2 crew = Edward Worsley, matching existing LogTen convention.
- **Takeoff/landing counts come only from the crew-entered `1x …` remark lines.** The CSV's Day T/O and Day Ldg columns are hardcoded 1/1 in every row (including night sectors) and are ignored. An absent line means the other pilot handled it — zero is logged. All counts editable per sector.
- **Remark line → field mapping**:
  - `1x Takeoffs/Landings Day|Night` → day/night T/O & Ldg (night counts on an NVIS sector also populate NVG T/O & Ldg)
  - `1x Elevated Helipads Take offs|Landings|TOLO` → Platform T/O / Platform Ldg
  - `1x NVIS Flight Currency / Sectors [(n mins)]` → NVIS Sectors +1; NVG time = stated minutes, else the sector's night time
  - `1x Instrument Approaches` → IFR Sectors
  - `Sole Reference to Instruments (n mins)` → Actual Inst
  - Everything else → remarks, de-duplicated (HCM's `RTB Redhill RTB Redhill` collapse) and prefixed `GAMA - ` so smart groups keep working.
- **Places**: known names mapped via the editable alias table (Redhill → EGKR, HCM hospital spellings → LogTen names). OS grid refs → From/To `KSS HEMS SITE` with the grid written into remarks. Blank/unrecognisable places (e.g. the `VW…` rows) are flagged `CHECK FROM/TO`.
- **Standing values per sector**: Multi-Pilot, XC, Turbine, Twin Turbine = total time. HEMS time = total time unless the sector matches training keywords (OPC/LPC/TRAINING/RECURRENT/LOFT/CX FLIGHT/ATPL) — toggleable.
- **Registrations** G-KSST/G-KSSC/G-LNAC/G-MGPS → AW169. Anything else (CP-818 sim rows) is excluded by default and flagged — log sim details manually.
- **Deduplication**: `flight_key = HCM-<epoch>`. LogTen updates rather than duplicates on re-send; "Mark batch imported" records keys on-device so re-uploaded exports only surface new sectors. Locked flights in LogTen are never touched by the API.

## Custom field calibration (one-time, required)

LogTen's API addresses custom fields by slot number, not display name. The slot numbers in **Field map** are guesses until calibrated:

1. Field map → **Send calibration flight**. A dummy flight (01/01/2028, ZZZZ–ZZZZ) arrives with custom time slot *n* = 0:0*n*, ops/takeoff/landing slot *n* = *n*.
2. Open it in LogTen and note which of your named fields (Turbine, HEMS, NVIS Sectors, Platform T/O …) shows which value.
3. Set the slots in Field map, save, **Mark calibrated**, then **Delete calibration flight**.

## Not yet built

Own-logbook file output (append every imported sector to a plain JSON/CSV in iCloud) and the CAA-format print page — next phases, same data model.
