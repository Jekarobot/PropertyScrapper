# Stage 08 — Hardening, operations, release

**Goal:** make the thing survive unattended nightly runs and be usable by a human who has never seen
the repo: one command to run everything, clear logs, retention, alarms on empty/partial data, a11y and
responsive polish, and a full regression pass.

**Depends on:** 02, 04, 06, 07. **Est:** 60 min. **Agent:** any (files do not overlap earlier stages).

**Files:**

```text
scripts/nightly.sh
scripts/connex-watch.service
scripts/connex-watch.timer
Makefile                 (publish, nightly, check additions)
README.md                (runbook, exit codes, troubleshooting, roadmap)
web/src/App.tsx          (a11y/touch-ups only)
web/src/components/*     (a11y/touch-ups only)
```

---

## Task 8.1 — Nightly script

`scripts/nightly.sh` (bash, `set -euo pipefail`), responsibilities:

1. Absolute paths from the script location (`cd "$(dirname "$0")/.."`).
2. `flock -n /tmp/connex-watch.lock` — a second run exits 0 with "already running".
3. Activate `.venv`, then:

   ```text
   connex-watch crawl --preset daily --resume --out data/raw >> logs/nightly.log 2>&1   → run dir
   connex-watch build  --raw <run dir> --out data/runs/<date>/dataset.json \
                       --cache data/cache/buildings.json >> logs/nightly.log 2>&1
   connex-watch check  --dataset data/runs/<date>/dataset.json                            → exit 1 stops
   ```

4. Publish atomically: `cp` to `web/public/data/dataset.json.tmp` then `mv`.
5. Retention: keep the last 14 `data/runs/*` and the last 7 `data/raw/*`; delete the rest.
6. Log rotation: if `logs/nightly.log` > 5 MB, `mv` to `.1`.
7. Exit codes propagate: `0` ok, `3` blocked, `4` parse error, `1` data check failed.

**TDD-ish verification** (shell is awkward to unit test; use a dry-run harness):

- `scripts/nightly.sh --dry-run` prints the exact command lines and touches nothing.
- Add `scripts/test-nightly.sh`: runs the script with `CONNEX_WATCH_BIN` pointing at a stub that
  records arguments and returns 0; asserts the four commands are invoked in order and that a failing
  `check` (stub returns 1) stops the publish step. Run it from `make test`.

Commit: `stage-08: nightly orchestration script`.

---

## Task 8.2 — systemd units (documented, not installed)

```ini
# scripts/connex-watch.service
[Unit] Description=Connex Watch nightly crawl
[Service] Type=oneshot
WorkingDirectory=/home/jeka/DeepseekBench
ExecStart=/home/jeka/DeepseekBench/scripts/nightly.sh

# scripts/connex-watch.timer
[Timer] OnCalendar=*-*-* 02:30:00  Persistent=true
```

`README.md` gets the install commands (`systemctl --user enable --now …`, `journalctl --user -u …`),
plus a plain `cron` alternative (`30 2 * * * cd /home/jeka/DeepseekBench && ./scripts/nightly.sh`).
**Do not enable or install anything during the stage** — that is production-ish and needs the owner.

Commit: `stage-08: systemd timer example`.

---

## Task 8.3 — `make publish` and `make check` completion

- `publish`: `web/public/data/dataset.json` from the latest `data/runs/*` (error if none).
- `check`: `ruff check pipeline` + `pytest pipeline -m "not network"` + `pnpm -C web typecheck` +
  `pnpm -C web test` + `connex-watch check --dataset web/public/data/dataset.json` (skipped with a
  clear message when the dataset is absent — a fresh clone must not fail `check`).
- `nightly`: `scripts/nightly.sh`.
- `make help` prints the targets with one-line descriptions (cheap, and it is the entry point the
  owner will actually use).

Commit: `stage-08: make targets for publish and check`.

---

## Task 8.4 — Accessibility and responsive pass

**TDD order** (extend `web/src/App.test.tsx` + a new `a11y.test.tsx`):

1. Landmarks and headings: exactly one `h1` (header), `main`, `aside`, `section[aria-label="Results"]`,
   `section[aria-label="Map"]`.
2. Every interactive element reachable by Tab; focus ring visible (`focus-visible:ring-2` class
   assertion is acceptable); no positive `tabindex`.
3. Results count and selection changes announce via `aria-live="polite"` (assert `role="status"`).
4. Colour contrast: accent `#0f766e` on white ≥ 4.5:1 (compute once in a test with a tiny contrast
   helper in `lib/format.ts` — 10 lines, no dependency).
5. Mobile (375×812 in Playwright): filter bar collapses to a "Filters" button that opens a sheet;
   map/list toggle works; no horizontal scrollbar (`document.documentElement.scrollWidth <= clientWidth`).
6. Reduced motion: `@media (prefers-reduced-motion: reduce)` disables the tooltip fade.

Commit: `stage-08: accessibility and responsive pass`.

---

## Task 8.5 — Runbook in `README.md`

Required sections, each 3–10 lines:

1. **What it is** + screenshot (`docs/screenshot.png`, captured from the real dataset).
2. **Quick start** — `make check`, `make nightly`, `pnpm -C web dev`.
3. **Data freshness** — where `generatedAt` comes from, what "partial" means, how to force a
   `--full` run.
4. **Exit codes** table (0/1/2/3/4) copied from Stage 02 and Stage 03.
5. **Troubleshooting** — the six failures we actually expect:
   - "0 cards on page 1" → site markup changed → re-capture fixture, run parser tests;
   - "blocked (403)" → stop, wait, reduce frequency, do not change UA;
   - "cache corrupt" → `rm data/cache/buildings.json`, rerun (costs ~20 min);
   - "dataset older than 48 h" → the timer did not run, check `journalctl`;
   - "many buildings without coords" → open one detail page manually, check the `center:` literal;
   - "map is grey" → tile provider blocked/slow, switch `VITE_TILE_URL` to OSM.
6. **Roadmap** — the non-goals from `02-architecture.md` §9.
7. **Ethics** — politeness floor, robots.txt, read-only, no personal data (a paragraph).

Commit: `stage-08: runbook and troubleshooting`.

---

## Task 8.6 — Full regression and sign-off

Run the whole matrix from `90-test-matrix.md` and paste the output into the "Result" table:

```bash
make check                                   # offline
pytest pipeline -m network -q                # 1 live smoke test
make nightly                                 # end-to-end with the real site (owner's approval)
pnpm -C web e2e                              # 7 Playwright tests
du -h web/public/data/dataset.json           # size sanity
python -c "import json;d=json.load(open('web/public/data/dataset.json'));print(d['stats'])"
```

Expected shape of the real-data stats (2026-11-01 baseline, will drift):

```text
units ≈ 1100, buildings ≈ 950, buildingsWithCoords ≥ 85 % of buildings, pagesScanned ≈ 73
```

If `buildingsWithCoords` is far below 85 %, investigate before shipping (Stage 04 traps).

Commit: `stage-08: release checklist`.

---

## Acceptance criteria

- [ ] `make check` green from a clean clone (with `web/public/data/dataset.json` absent)
- [ ] `scripts/nightly.sh --dry-run` prints 4 commands
- [ ] `scripts/test-nightly.sh` passes in `make test`
- [ ] `README.md` contains all seven sections, exit codes match the CLI
- [ ] Playwright mobile scenario passes at 375×812
- [ ] real dataset satisfies `check` and the 85 % coordinate threshold
- [ ] `git log --oneline | wc -l` shows one commit per task across all stages

## Result

| Task | Evidence | Status |
|---|---|---|
| 8.1 | `scripts/test-nightly.sh` output | |
| 8.2 | `systemd-analyze verify` (optional) | |
| 8.3 | `make help`, `make check` | |
| 8.4 | `a11y.test.tsx` + mobile e2e | |
| 8.5 | README sections list | |
| 8.6 | full matrix output pasted below | |

```text
<paste make check / e2e / nightly output here>
```

## Traps

- `flock` on `/tmp` is fine for one host; do not use it as a distributed lock.
- Retention deletes data — check the glob twice and dry-run once (`ls -d data/runs/* | head -n -14`).
- A failed `check` must **not** publish; a stale dataset beats a wrong one.
- Do not enable the systemd timer during the stage; that is the owner's call.
- Screenshot for the README must come from a real run (privacy: no login state, no browser chrome).
