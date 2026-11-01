---
id: 05-11
stage: 05
title: Bounding-box test and no-location split
points: 1
owner: web
status: todo
depends_on: [05-03]
files:
  - web/src/lib/viewport.ts
  - web/src/lib/viewport.test.ts
---

# 05-11 · Bounding-box test and no-location split

**Points:** 1 · **Stage:** 05 — Web shell · **Owner lane:** web · **Depends on:** 05-03 · **Unblocks:** 05-15, 07-04

## Goal

Add `inBBox` and `splitByLocation`: inclusive coordinate containment and the split into visible buildings versus the "No location" bucket, with null-coordinate buildings never in the view.

## Why this task exists

The right-hand list shows what is inside the current map view, and two rules make that honest: a building exactly on the viewport edge must be included (the map's own filtering is inclusive, and a building flickering at the edge is a bug report), and a building with `lat: null` can never be "in view" — it belongs to a separate, explicitly labelled bucket (F6) instead of silently disappearing. Thailand has no antimeridian crossing, so this module deliberately does not implement dateline wrapping; that assumption is written down at the top of the file so nobody "fixes" it into a world map.

## Current state (preconditions)

- 05-03 is done: the fixture has three buildings — `p00348` at 13.7187616 / 100.5202356, `premier-thonglor` with non-null Bangkok coordinates, and `no-location` with `lat`/`lng` null.
- `web/src/lib/` contains `filter.ts` and `group.ts`; there is no `viewport.ts`.
- No map, no Leaflet and no `moveend` handling exists yet (stage 07).

## Read before you start

- `CLAUDE.md` §3 (exact coordinates, no network), §5 (pure functions, no `any`), §6, §7.
- `plan/02-architecture.md` §5.2 rule F6 — null coords never in the viewport, always in the "No location (N)" bucket; §5.1 for where the list and bucket live.
- `plan/stage-05-web-shell.md` Task 5.3 — the viewport case table, including the Thailand-only comment.
- `plan/90-test-matrix.md` row W-06 — "null coords never in viewport, always in noLocation".
- `web/src/test/fixtures/dataset.sample.json` — the three buildings' coordinates.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `web/src/lib/viewport.ts` | `BBox`, `inBBox`, `splitByLocation`, the Thailand-only comment |
| create | `web/src/lib/viewport.test.ts` | the six cases below |

Nothing else may change: no map, no hook, no `group.ts` or `filter.ts` edit, no fixture edit.

## Test first (red)

Import the fixture and look buildings up by id. Define the test box once at the top of the file: south 13.6, west 100.4, north 13.9, east 100.7 (it contains central Bangkok, hence `p00348`).

1. `test_inBBox_is_inclusive_on_all_four_edges` — the point (13.7, 100.5) is inside; (13.6, 100.5), (13.9, 100.5), (13.7, 100.4) and (13.7, 100.7) are each also inside, one per edge; the four corners are inside.
2. `test_inBBox_rejects_points_outside_any_edge` — (13.599, 100.5), (13.901, 100.5), (13.7, 100.399) and (13.7, 100.701) are each outside, one per edge.
3. `test_inBBox_rejects_null_coordinates` — `(null, 100.5)`, `(13.7, null)` and `(null, null)` are all false, never an exception.
4. `test_inBBox_does_not_swap_latitude_and_longitude` — `inBBox(13.7, 100.5, box)` is true and `inBBox(100.5, 13.7, box)` is false; latitude is the first argument, longitude the second.
5. `test_splitByLocation_buckets_mapped_and_unmapped_buildings` — with the test box, `inView` is `p00348` then `premier-thonglor` and `noLocation` is `[no-location]`; with a narrow box containing only `p00348`, `inView` is `[p00348]` and `premier-thonglor` appears in neither bucket (an off-screen building is not "unmapped").
6. `test_splitByLocation_preserves_input_order` — pass `[no-location, premier-thonglor, p00348]`: `inView` is `[premier-thonglor, p00348]` and `noLocation` is `[no-location]`, i.e. the input order, never sorted; the input array and the building objects are unchanged.

Expected red symptom: `Cannot find module './viewport'`. A case failing with a type error after the module is created means the case is wrong — fix it before the module.

Run: `pnpm -C web test viewport` — six red for the missing module.

## Implement (green)

- `BBox` interface with exactly `south`, `west`, `north`, `east` (all numbers). No `lat`/`lng` aliases, no optional fields.
- `inBBox(lat: number | null, lng: number | null, b: BBox): boolean` — `false` if either coordinate is `null` (test with `== null`, so a legitimate coordinate `0` is not mistaken for missing); otherwise inclusive on all edges: `lat >= b.south && lat <= b.north && lng >= b.west && lng <= b.east`. Latitude first. No rounding, no tolerance, no projection.
- `splitByLocation(buildings, b)` returns `{ inView, noLocation }`: null-coordinate buildings go to `noLocation`; buildings with coordinates go to `inView` only when `inBBox` is true; a building outside the box goes to neither array. Both arrays preserve the input order, share the building objects (read-only), and are new arrays. No sorting, no dedupe, no default box.
- Add a file-level comment stating the assumption: the bbox is a Thailand-sized rectangle that does not cross the antimeridian, so west ≤ east and no dateline handling exists. One-sentence docstrings on both functions. No `any`, no React import, no Leaflet import.

Run: `pnpm -C web test viewport` — six green.

## Refactor / keep it DRY

One containment rule, inside `inBBox`; `splitByLocation` calls it instead of repeating the four comparisons. Do not add a second "is inside the city" helper or a hard-coded Bangkok bbox in the module — the hook (05-15) owns `bbox: null` meaning the whole city, and the map (07-04) supplies the real bounds.

## Verification (how to test it)

```text
pnpm -C web test viewport
pnpm -C web test --run
pnpm -C web typecheck
pnpm -C web build
make check
```

Expected: 6 passed for the target, 59 for the lane, typecheck 0, build clean, `make check` green. Manual: `sed -n '1,10p' web/src/lib/viewport.ts` shows the Thailand-only/dateline comment.

## Out of scope (do not do)

- No dateline/antimeridian handling, no geodesic distance, no radius, no zoom levels.
- No Leaflet import or map event subscription (07-04); no `bbox: null` handling (05-15).
- No filtering by units, no grouping, no component, no hook, no dependency.

## Definition of done

- [ ] All six cases exist in `web/src/lib/viewport.test.ts`, named as written, and pass.
- [ ] Red run observed and quoted; the fixture's null-coordinate building is in `noLocation` in every relevant case.
- [ ] `pnpm -C web typecheck` clean; lane suite and `make check` green; no `any`.
- [ ] Only the two listed files are modified (`git status --short` checked).
- [ ] Frontmatter `status: done`; `## Result` filled; exactly one commit on the trunk (`main`) with the message below.

## Commit

```text
05-11: test viewport bounds and split unmapped buildings

- points: 1
- red: viewport.test.ts::test_inBBox_is_inclusive_on_all_four_edges failed (Cannot find module './viewport')
- add inclusive inBBox with null coordinates never inside
- split buildings into inView and noLocation, preserving input order
- suite: 59 passed
```

Files in the commit: `web/src/lib/viewport.ts`, `web/src/lib/viewport.test.ts`, and this task file.

## Traps and review notes

- `if (!lat)` treats latitude `0` (the equator) as missing; use an explicit null check. The same bug would silently drop a legitimate building.
- Swapping the `lat`/`lng` arguments still passes a symmetric test box; case 4 is asymmetric on purpose.
- Making the edges exclusive (`<`, `>`) breaks F6's neighbour rule and makes buildings at the map edge flicker.
- Sending off-screen buildings to `noLocation` would invent locations for them; the bucket is only for null coordinates.
- Reviewers: `grep -n 'south\|west\|north\|east' web/src/lib/viewport.ts` shows one inclusive chain; `grep -n 'dateline\|antimeridian' web/src/lib/viewport.ts` shows the documented assumption.

## Plan samples are untrusted

The plan gives the `BBox` field names and the case table, no body, and its composition sketch uses a `WORLD` box that this task does not create. Verify the installed TypeScript accepts `number | null` narrowing on `== null`, and do not paste a snippet (`CLAUDE.md` §6). Record deviations under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Deviations | |
