# Sector Log — HeliCrew Manager → LogTen Pro

Single-page web app. Upload both HCM exports (Excel + LogTen CSV) from the same date range, review the derived sectors, resolve anything flagged, tap Send — LogTen opens and asks to import. Deployed to GitHub Pages at **https://vontedworth.github.io/sector-log/**. Works offline after first load except for the two CDN libraries (PapaParse, SheetJS).

**Current build: 12** (shown top of the page as `build N · date`). Bump the `BUILD` constant in `index.html` on every commit so the running version is verifiable on the device.

The app **only references LogTen places by name and writes grids into remarks** — it never creates or modifies places or aircraft in LogTen, and never sends coordinates.

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
- **Rotor-time correction**: some pilots enter identical rotor and airborne times (Out = Off, On = In). Ed logs rotors-running time, so when they're equal the app subtracts 5 min from the start and adds 1 min to the stop and recomputes total. Flagged `ROTOR +6`; times stay editable.
- **Capacity**: every sector defaults to **SIC**. PICUS is a crew-room decision, so it is a one-tap toggle per sector (SIC ⇄ P1u/s). Crew names are unaffected: PIC/P1 crew = commander from the Excel (first non-SELF name), SIC/P2 crew = Edward Worsley, matching existing LogTen convention.
- **Takeoff/landing counts come only from the crew-entered `1x …` remark lines.** The CSV's Day T/O and Day Ldg columns are hardcoded 1/1 in every row (including night sectors) and are ignored. An absent line means the other pilot handled it — zero is logged. All counts editable per sector.
- **Remark line → field mapping**:
  - `1x Takeoffs/Landings Day|Night` → day/night T/O & Ldg
  - `1x Elevated Helipads Take offs|Landings|TOLO` → Platform T/O / Platform Ldg (see rooftop rule below — the app also derives these from the site)
  - `1x NVIS Flight Currency / Sectors [(n mins)]` → NVIS Sectors +1; NVG time = stated minutes, else the sector's night time
  - `1x Instrument Approaches` → IFR Sectors
  - `Sole Reference to Instruments (n mins)` → Actual Inst
  - Everything else → remarks (see below).
- **NVG takeoffs/landings depend on the site, per endpoint.** At night, **Redhill (EGKR) and HEMS scenes are NVG sites** — a night takeoff/landing there also counts NVG. All hospitals and all other airfields are white light — night only, never NVG. So a night scene→hospital sector logs NVG T/O (scene) but no NVG Ldg (hospital); Redhill→scene at night logs NVG at both ends. NVIS sector count and NVG time are unaffected. NVG T/O and NVG Ldg are editable on review for edge cases.
- **Passengers**: HCM carries no pax data, so each sector has an editable **Pax** field (whole numbers, empty by default). When filled it sends as `flight_paxCount` (LogTen's Total Pax); when empty the field is omitted entirely — zero is never sent. Not persisted beyond the batch. (`flight_paxCount` is the assumed attribute name — unlike the custom-field slots it hasn't been calibrated against LogTen; confirm the value lands in Total Pax on first import.)
- **Platform T/O & Ldg** come from the **elevated-helipad (rooftop) flag on the known-sites list**, not from name keywords: a departure from a rooftop site logs Platform T/O, an arrival at one logs Platform Ldg.
- **Standing values per sector**: Multi-Pilot, XC, Turbine, Twin Turbine = total time. HEMS time = total time unless the sector matches training keywords (OPC/LPC/TRAINING/RECURRENT/LOFT/CX FLIGHT/ATPL) — toggleable.
- **Deduplication**: `flight_key = HCM-<epoch>`. LogTen updates rather than duplicates on re-send; "Mark batch imported" records keys on-device so re-uploaded exports only surface new sectors. Locked flights in LogTen are never touched by the API.

## Places, scenes and grids

- **Known sites** are held in a reference list (`KNOWN_SITES`) carrying each site's LogTen identifier, name variants (for recognition), and two flags: `rooftop` (elevated helipad → drives Platform T/O/Ldg) and `scene` (the generic `KSS HEMS SITE`). No coordinates are stored. Seeded sites include the KSS airfields (EGKR/EGTO/EGKA/EGKB/EGMD) and the regular hospitals. Brighton has two distinct entries: **SE02 Brighton Royal Sussex Hospital** (rooftop, main) and **Royal Sussex Hospital Secondary** / East Brighton Park / "SE02A Royal Sussex - Alternate" (ground-level, no platform landing) — names containing "Alternate", "Secondary/Secondry" or "East Brighton Park" always resolve to the Secondary.
- **Recognition** is case-, punctuation- and operational-suffix tolerant: `Bagshot` and `Bagshot HOS` match the same scene (trailing HOS/RTC/OOH/HLS tokens are stripped), while matching stays otherwise strict (`Ashford` never matches `Ashfield`).
- **Grid references** (From/To or the Route/HLS column) → `KSS HEMS SITE`, with the six-figure grid written into remarks. Case-tolerant (`tq742543` → `TQ742543`).
- **Scene-grid carry-forward**: HCM's Route column only holds the *arrival's* grid, so a scene appearing as a *departure* has none of its own. The app carries it forward — a departure scene inherits the grid of the nearest earlier sector that arrived at the same-named scene (the aircraft takes off from where it landed). Anything still missing a scene grid is flagged `SCENE GRID MISSING` with an editable grid field, and **Send is blocked** until it's filled — a scene never reaches LogTen without its grid.
- **Remarks format**: `GAMA - KSS HEMS SITE - <name> <grid>` (departure endpoint first, then arrival), e.g. `GAMA - KSS HEMS SITE - Orpington TQ462692 - Maidstone TQ742543`. Grid-only scenes (no HCM name) show the grid alone. Hospital/airfield sectors use the shorthands `GAMA - KSS HEMS RTB` (return to Redhill) and `GAMA - KSS HEMS STANDDOWN` (Redhill→Redhill). Free-text remark lines are de-duplicated (HCM's `RTB Redhill RTB Redhill` collapse) and appended after ` | `.
- **Unrecognised places** are surfaced before import and **block Send** until resolved. For each distinct name you choose once: **HEMS Site** (→ `KSS HEMS SITE`), an existing **named site** (picked from all known sites), or **create as new** — new sites ask an "Elevated helipad?" toggle and are persisted on-device as first-class user sites (recognised on future imports, suffix-tolerant, driving Platform counts when flagged). User-added sites are listed, editable and deletable in the **Places** editor; deleting one makes its name flag again next time. The Places editor also keeps the older HCM-name → LogTen-name alias table.

## Aircraft

No registration is assumed to already exist in LogTen. On import, any reg not confirmed on this device is flagged and **blocks Send** until you either **add it as a new aircraft** (type defaults to AW169, remembered) or **exclude it** (e.g. a simulator or foreign reg — `EXCLUDED`, not imported). G-KSST/G-KSSC/G-LNAC are seeded as known; G-MGPS and anything else surface for a decision.

## Custom field calibration (one-time, required)

LogTen's API addresses custom fields by slot number, not display name. The shipped **Field map** defaults are already calibrated for Ed's LogTen (Turbine 4, Twin Turbine 9, HEMS 5, Offshore 6, Mountain 3, Sling 2; 3D op 7, 2D op 17, IFR Sectors op 1, NVIS Sectors op 8; Platform T/O takeoff 1, Platform Ldg landing 2), stored on-device. To recalibrate:

1. Field map → **Send calibration flight**. A dummy flight dated **01/01/2028** (future, so it sorts to the top of the logbook), ZZZZ–ZZZZ, arrives with custom **time** slot *n* = *n*:00 (whole hours, so LogTen's decimal display disambiguates) and op/takeoff/landing slot *n* = *n*.
2. Open it in LogTen and note which of your named fields (Turbine, HEMS, NVIS Sectors, Platform T/O …) shows which value.
3. Set the slots in Field map, save, **Mark calibrated**, then **Delete calibration flight**.

Until calibrated (`Mark calibrated` not yet tapped) an amber banner warns that custom values may land in the wrong fields.

## Not yet built

Own-logbook file output (append every imported sector to a plain JSON/CSV in iCloud) and the CAA-format print page — next phases, same data model.
