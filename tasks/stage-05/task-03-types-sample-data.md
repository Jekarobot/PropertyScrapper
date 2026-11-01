---
id: 05-03
stage: 05
title: Dataset types and committed sample dataset
points: 1
owner: web
status: todo
depends_on: [05-01]
files:
  - web/src/types.ts
  - web/src/test/fixtures/dataset.sample.json
  - web/src/types.test.ts
  - web/tsconfig.json
---

# 05-03 · Dataset types and committed sample dataset

**Points:** 1 · **Stage:** 05 — Web shell · **Owner lane:** web · **Depends on:** 05-01 · **Unblocks:** 05-04, 05-07…05-12, 05-13, 07-01

## Goal

Mirror the published dataset shape from `plan/02-architecture.md` §3.3 in TypeScript and commit the six-unit sample fixture that every later web test reads instead of the real dataset.

## Why this task exists

The web app consumes exactly one file, and its field names are the contract between the pipeline lane and the web lane. If a type is invented here, later components read `price` instead of `priceThb` and nothing catches it until the real dataset lands. The sample fixture is equally load-bearing: unit tests must never fetch, never need the network and never depend on a crawl, so the fixture is authored once, frozen, and used by the loader, filter, grouping, viewport and formatting tests. It deliberately contains a building with `lat: null`, because that path is otherwise untested until production data surprises us.

## Current state (preconditions)

- 05-01 is done: `web/` builds and type-checks, Vitest + jsdom run, the pinned dependencies are installed.
- `web/src/types.ts` and `web/src/test/fixtures/` do not exist; `web/tsconfig.json` is the Vite 5 project config — check whether `resolveJsonModule` is enabled (see Traps).

## Read before you start

- `CLAUDE.md` §3 (fixtures frozen; never re-capture), §5 (no `any`), §6, §7; `plan/02-architecture.md` §3.1 (identities: `Unit` = `CX-<digits>`, `Building` = project code else slug) and §3.3 (exact JSON field names, nullability, sorting).
- `plan/stage-05-web-shell.md` Task 5.2 — the fixture's required composition and counts.
- `plan/90-test-matrix.md` rows W-01 (loader shape), W-06 (null coords), W-12 (ranges recomputed).
- `plan/02-architecture.md` §5.1 — the header shows `6 units` / `3 buildings`, so the fixture totals are fixed.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `web/src/types.ts` | `Dataset`, `Building`, `Unit`, `Stats`, `SearchBlock`; no runtime code |
| create | `web/src/test/fixtures/dataset.sample.json` | the six-unit, three-building sample |
| create | `web/src/types.test.ts` | the seven cases below, including the compile-time assignment |
| modify | `web/tsconfig.json` | only if needed: enable `resolveJsonModule` for the JSON import |

Nothing else may change: no loader, no hooks, no component, no `src/test/fixtures.ts` helper.

## Test first (red)

Write `web/src/types.test.ts` before the fixture and types exist. The sample is imported as a JSON module and assigned to a `Dataset`-typed constant, so a wrong or missing field fails `pnpm -C web typecheck`, not just the test.

1. `test_sample_dataset_matches_the_dataset_type` — the imported fixture assigns to a `Dataset` constant without a cast; `schemaVersion === 1`; `stats.units` equals the sum of `buildings[].units.length`.
2. `test_sample_has_three_buildings_and_six_units_in_dataset_order` — building ids are exactly `p00348`, `premier-thonglor`, `no-location` in that order; six units total.
3. `test_bangkok_sathorn_carries_section_3_3_values` — id `p00348`, name `The Bangkok Sathorn`, district `Sa Thon`, province `Bangkok`, lat `13.7187616`, lng `100.5202356`, non-empty `coordSource`, `unitCount` 3, `priceMin` 45000, `priceMax` 50000, `sizeMin` 58, `sizeMax` 65; unit `CX-84121` has price 45000, size 65, bedrooms 1, bathrooms 1, floor 27, `updatedOn` `2026-10-29`, an `https://connex.in.th/…` URL.
4. `test_premier_thonglor_has_two_units_and_coords` — id `premier-thonglor`, name `Premier Thonglor`, district `Watthana`, province `Bangkok`, `lat`/`lng` non-null and inside the Bangkok box 13.6–13.9 N / 100.4–100.7 E, `unitCount` 2, one unit at 45 sqm.
5. `test_no_location_building_has_null_coords` — id `no-location`, district `Khlong Toei`, province `Bangkok`, `lat` and `lng` both `null`, `coordSource` `null`, `unitCount` 1.
6. `test_units_stay_inside_the_documented_ranges` — every `priceThb` in 25000…50000, every `sizeSqm` in 45…150, every `bedrooms` in 0…3; exactly one unit has `floor: null`; at least one studio (0 bedrooms), one 3-bedroom unit and one 45 sqm unit; unit ids are unique and match `CX-<digits>`.
7. `test_search_and_stats_blocks_mirror_section_3_3` — `search` has exactly the ten §3.3 keys (`project_code`, `listingtype`, `propertytype`, `bedroom`, `minsize`, `maxsize`, `minprice`, `maxprice`, `special`, `sort`) with `listingtype` 2, `propertytype` 1, `bedroom` null, sizes/price bounds 45/150/25000/50000, empty `project_code` and `sort`, null `special`; `stats` has 6 units, 3 buildings, 2 with coords, `partial: false`, `excluded.check_availability` 0, `excluded.not_available` 0; `source` is `connex.in.th` / `rent-condo`; `generatedAt` is the fixed `2026-11-01T21:45:00Z`.

Expected red symptom: `Cannot find module './types'`, then `Module not found: …/dataset.sample.json`, then a `typecheck` error naming the mismatched field. If a case fails with a syntax error, fix the test first.

Run: `pnpm -C web test types` — red until types and the fixture exist; `pnpm -C web typecheck` must also fail on the missing module.

## Implement (green)

- `web/src/types.ts`: exported interfaces only, no runtime values and no validation logic (the loader owns validation in 05-04/05-05). Fields and nullability exactly as §3.3: `Unit` = id, url, priceThb, sizeSqm, bedrooms, bathrooms, floor (`number | null`), updatedOn; `Building` = id, name, district (`string | null`), province, lat/lng (`number | null`), coordSource (`string | null` — the sample's unmapped building has no provenance), unitCount, priceMin, priceMax, sizeMin, sizeMax, units; `Stats` = units, buildings, buildingsWithCoords, pagesScanned, duplicatesDropped, excluded (check_availability, not_available), partial; `SearchBlock` = the ten keys above; `Dataset` = schemaVersion (`number` — JSON modules widen literals, the loader enforces `=== 1`), generatedAt, source (`site`, `kind`), search, stats, buildings. No `any`, no enums, no optional fields.
- `web/src/test/fixtures/dataset.sample.json`: hand-written sample, not a real run. The three buildings above, six units total, prices, sizes and bedrooms within the stated ranges; per-building `unitCount`/`priceMin`/`priceMax`/`sizeMin`/`sizeMax` must equal the values recomputed from that building's units, so 05-10 can prove recomputation by passing a subset. Unit order inside a building is by `priceThb` then `id` (§3.3); building order is `p00348`, `premier-thonglor`, `no-location`. Do not include HTML, descriptions, images or extra keys; names are unique, and only `The Bangkok Sathorn` contains the word `bangkok`, so the F4 substring test in 05-09 is unambiguous. `Premier Thonglor` holds a 45 sqm studio and a 50 sqm two-bed; `no-location` holds the single 3-bed at 150 sqm with `floor: null`; `p00348` is CX-84121 (45,000 / 65 sqm / 1 bed / floor 27), a 50,000 / 58 sqm / 2-bed and a 48,000 / 60 sqm / 2-bed.
- Keep `generatedAt` fixed (`2026-11-01T21:45:00Z`) — later status-bar tests freeze on it.
- Enable `resolveJsonModule` in `web/tsconfig.json` only if the JSON import fails to type-check; the flag is the only permitted config change.

Run: `pnpm -C web test types` — seven green; `pnpm -C web typecheck` clean.

## Refactor / keep it DRY

One type module; no per-domain copies in `state/`, `lib/` or components. One fixture; later tasks import this file and must not clone it. The per-building summary values are computed once by the pipeline and asserted here — do not add a web helper that recomputes them into the fixture (that is `groupByBuilding`'s job at runtime, 05-10).

## Verification (how to test it)

```text
pnpm -C web test types
pnpm -C web test --run
pnpm -C web typecheck
pnpm -C web build
make check
```

Expected: 7 passed for the target, 17 passed for the lane, typecheck 0, build clean, `make check` green. Manual check: `grep -n ': any' web/src/types.ts web/src/types.test.ts` prints nothing.

## Out of scope (do not do)

- No `loadDataset`, fetch, runtime validation or schema library; no hooks, components, App changes or `src/test/fixtures.ts` helper module.
- No real dataset, no `web/public/data/`, no re-capture, no invented extra fields, no images.
- No additional buildings or units beyond the six required; the fixture is a contract, not a playground.

## Definition of done

- [ ] All seven cases exist in `web/src/types.test.ts`, named as written, and pass.
- [ ] The fixture-to-`Dataset` assignment fails typecheck before `types.ts` exists (red evidence); lane tests, typecheck, build and `make check` green; no `any` in either file.
- [ ] Fixture totals stay 3 buildings / 6 units with the stated ids and order; only the listed files are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled; exactly one commit on the trunk (`main`) with the message below.

## Commit

```text
05-03: add dataset types and the committed sample dataset

- points: 1
- red: types.test.ts::test_sample_dataset_matches_the_dataset_type failed (Cannot find module './types')
- add Dataset/Building/Unit/Stats/SearchBlock mirroring architecture section 3.3
- add the six-unit sample fixture with a null-coordinate building
- suite: 17 passed
```

Files in the commit: the three created paths, `web/tsconfig.json` only if the flag was needed, and this task file.

## Traps and review notes

- `resolveJsonModule` is often absent from a Vite 5 tsconfig; without it the JSON import fails typecheck with a misleading module error. Enable it rather than casting.
- Keep `schemaVersion` a plain `number`: JSON modules widen numeric literals, so a literal `1` type would reject the fixture. The loader (05-04) owns the version check.
- The per-building ranges must match the units exactly; a copied real-world number that disagrees makes 05-10's recomputation test meaningless.
- Do not make `coordSource` non-nullable just because §3.3's example shows a string: the `no-location` building has no provenance, and later code must compile against `null`.
- Reviewers: `project_code`, `listingtype`, `minsize`, `maxsize`, `minprice`, `maxprice` keep underscores (§3.3); `excluded` keys are `check_availability`/`not_available`; `partial` is boolean; no field renamed to camelCase.

## Plan samples are untrusted

`plan/02-architecture.md` §3.3 is a sample document, not code. Copy the field names and nullability from it, but verify JSON-module typing and `resolveJsonModule` behaviour against the installed TypeScript (`CLAUDE.md` §6). If the sample's numbers conflict with the required composition, the task's numbers win and the deviation goes under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Deviations | |
