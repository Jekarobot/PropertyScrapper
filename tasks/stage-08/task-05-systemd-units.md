---
id: 08-05
stage: 08
title: Documented systemd service and timer
points: 0.5
owner: any
status: todo
depends_on: [08-02]
files:
  - scripts/connex-watch.service
  - scripts/connex-watch.timer
  - README.md
---

# 08-05 · Documented systemd service and timer

**Points:** 0.5 · **Stage:** 08 — Hardening and release · **Owner lane:** any (ops) · **Depends on:** 08-02 · **Unblocks:** 08-11

## Goal

Ship the two unit files that would schedule the nightly at 02:30 local time, and document the install commands plus a cron alternative in the README — without installing or enabling anything.

## Why this task exists

The nightly only becomes a product when something runs it at night; until then the owner has to remember. systemd user units are the host's own scheduler, so the repository should carry the exact unit text instead of a wiki page nobody can diff,
and the README must show the commands that turn them on, where the journal lives, and a cron line for hosts without a user session bus. Enabling a unit is an unattended production action, so this task documents and never installs. The unit
content is also a contract with 08-01: `ExecStart` names the script, `WorkingDirectory` names the repository, and the calendar expression is the schedule the runbook quotes.

## Current state (preconditions)

- 08-02 is done: `scripts/nightly.sh` is executable, needs no activated virtualenv, exits non-zero with one stderr line, and holds the lock so an overlapping tick is harmless.
- `README.md` has the seven `##` sections from 00-02 in order (What it is, Quick start, Data freshness, Exit codes, Troubleshooting, Roadmap, Ethics), still holding placeholder text.
- `scripts/` holds `nightly.sh`, `tests/stub-connex-watch.sh` and (after 08-04) `test-nightly.sh`; there are no unit files.
- The repository path is `/home/jeka/DeepseekBench` with a `.venv` at its root; confirm both before writing absolute paths into a unit.

## Read before you start

- `CLAUDE.md` §7 (describe directives, do not paste them), §10 (do not install or enable units, cron jobs or anything unattended), §11.
- `plan/stage-08-hardening-ops.md` Task 8.2 — unit content, install commands, cron alternative, "do not enable or install anything".
- `plan/02-architecture.md` §7 (a warm nightly under 4 minutes) and §8 — the schedule must not be tightened to work around slowness.
- `plan/90-test-matrix.md` "Manual acceptance checklist" — the schedule exists but the owner enables it; `README.md` — where "Data freshness" is.

## Files to touch

| Action | Path | What goes there |
|---|---|---|
| create | `scripts/connex-watch.service` | the oneshot unit that runs the nightly |
| create | `scripts/connex-watch.timer` | the daily 02:30 trigger, persistent across downtime |
| modify | `README.md` | install/journal/cron lines inside "Data freshness" only |

Nothing else: no `Makefile`, no `scripts/*.sh`, no eighth README section, no `~/.config/systemd/**` content in the repository, no dependency.

## Test first (red)

No test file: unit text and documentation are configuration scaffolding, the `CLAUDE.md` §3 exception, and these checks are the evidence. Run them from the repo root.

1. `check-1` `test -f scripts/connex-watch.service` — exits 1 before; after, 0.
2. `check-2` the service contains `[Unit]`, `Description=Connex Watch nightly crawl`, `[Service]`, `Type=oneshot`, `WorkingDirectory=/home/jeka/DeepseekBench` and `ExecStart=/home/jeka/DeepseekBench/scripts/nightly.sh`, and no other behaviour
   directive (no `User=`, `Environment=`, `Restart=`). Exactly those directives are the contract with 08-01.
3. `check-3` the timer contains `[Timer]`, `OnCalendar=*-*-* 02:30:00`, `Persistent=true`, and an `[Install]` section with `WantedBy=timers.target` — without `[Install]`, `enable` silently does nothing, which is the failure this file prevents.
4. `check-4` "Data freshness" contains the three documented operations as inline command text: `systemctl --user enable --now connex-watch.timer`, `journalctl --user -u connex-watch` (optionally with `-f`), and the cron line `30 2 * * * cd
   /home/jeka/DeepseekBench && ./scripts/nightly.sh`.
5. `check-5` nothing is installed — `ls ~/.config/systemd/user/connex-watch.*` fails and `systemctl --user is-enabled connex-watch.timer` does not print `enabled`. Run this last; it proves the task obeyed §10.
6. `check-6` `grep -c '^## ' README.md` still prints `7` — the install text lives inside an existing section, so 00-02's structure check keeps passing.
7. Optional: `systemd-analyze verify scripts/connex-watch.service scripts/connex-watch.timer` exits 0 or reports only the expected "unit outside the search path" note; record its exact output in `Result`.

Expected red: checks 1–4 fail on missing files and missing README text; check 5 already passes and must still pass afterwards.

## Implement (green)

- `scripts/connex-watch.service`: a `[Unit]` block with the one-line description, then `[Service]` with `Type=oneshot`, `WorkingDirectory=/home/jeka/DeepseekBench` and `ExecStart=/home/jeka/DeepseekBench/scripts/nightly.sh`. Nothing else: no
  `Restart=`, because restarting after a blocked crawl would hammer the site; no `User=` on a user unit; no `Environment=`, because the script resolves its own paths.
- `scripts/connex-watch.timer`: a `[Unit]` description, `[Timer]` with `OnCalendar=*-*-* 02:30:00` and `Persistent=true` (a laptop asleep at 02:30 runs the missed job after wake, which the dataset's 48 h freshness rule assumes), and `[Install]`
  with `WantedBy=timers.target`. Deliberately absent: `RandomizedDelaySec=`, `AccuracySec=`.
- `README.md`, inside "Data freshness" only, three to ten lines with inline commands: install and start the timer, where the journal is and how to follow it, and the cron line as the alternative for hosts without a user bus. One sentence says
  the schedule is the owner's to enable and that a second concurrent run exits immediately because of the lock.
- No unit content, install command or cron line anywhere else (not in `Makefile`, not in `scripts/`). No `systemctl enable/start/daemon-reload` or `crontab -e` invocation in this task, not even to "check it works".

Run: checks 1–6 again — all pass, check 5 still proves nothing is enabled.

## Refactor / keep it DRY

The schedule appears exactly twice: `OnCalendar` in the timer and the cron line in the README, both at 02:30, both explained once in "Data freshness". Do not repeat the commands in Quick start, do not add a "Scheduling" section, and do not copy
the unit text into the README — the files are the source and the README points at them.

## Verification (how to test it)

```text
test -f scripts/connex-watch.service && test -f scripts/connex-watch.timer
grep -c '^## ' README.md
ls ~/.config/systemd/user/connex-watch.* ; echo "exit=$?"
systemd-analyze verify scripts/connex-watch.service scripts/connex-watch.timer
make check
```

Expected: both files present; `7`; the `ls` exits non-zero (nothing installed); `make check` green. Manual read: "Data freshness" tells an owner who has never seen the repo how to schedule the job and watch the result.

## Out of scope (do not do)

- Installing, enabling, starting, stopping, disabling or reloading any unit or cron entry; copying the files into `~/.config/systemd/user/`; verifying a live unit. The owner does this.
- A `Restart=`/`OnFailure=` policy, `RandomizedDelaySec`, an environment file, a system-level service, an `ExecStartPre` venv creation, or a second timer.
- Editing any other README section (08-11 fills them), the `Makefile`, or `scripts/*.sh`; adding a dependency or an installer helper.

## Definition of done

- [ ] Checks 1–7 behave as written, including check 5 proving nothing is enabled.
- [ ] The unit text contains exactly the directives named above, with the absolute repository path.
- [ ] `README.md` still has exactly seven `##` sections, and the install text is inside "Data freshness".
- [ ] Only the three listed files changed (`git status --short`); no `~/.config` write, no cron edit.
- [ ] Frontmatter `status: done`; `## Result` filled with the commit hash and the check outputs.
- [ ] Exactly one commit on the trunk with the message below.

## Commit

Message:

```text
08-05: document the systemd service and timer

- points: 0.5
- red: test -f scripts/connex-watch.service exited 1
- add the oneshot service and the 02:30 persistent timer, not installed
- document enable, journalctl and the cron alternative in Data freshness
- suite: 17 nightly cases + 6 harness modes unchanged
```

Files in the commit: `scripts/connex-watch.service`, `scripts/connex-watch.timer`, `README.md`, this task file. One commit, on trunk `main`.

## Traps and review notes

- Copying the units into `~/.config/systemd/user/` "to check them" is the action §10 forbids; a reviewer runs check 5 and looks at `ls -l ~/.config/systemd/user`.
- `02:30` alone is not a calendar expression; the timer would fall back to a default schedule — a silent, wrong behaviour that `systemd-analyze verify` catches.
- A timer without `[Install]` makes `systemctl --user enable` a no-op that still exits 0; the check greps the section instead of trusting `enable`.
- `Persistent=true` on a busy host can start the job late; the script's lock keeps that safe, so no `ExecStartPre` sleep and no second timer.
- Relative paths in `WorkingDirectory`/`ExecStart` resolve against the unit's directory — both must be absolute.
- Reviewers: `grep -rn 'systemctl .*enable' README.md scripts Makefile` finds exactly one occurrence, in the README prose.

## Plan samples are untrusted

The plan's Task 8.2 block is an INI fragment, not a valid unit pair: no `[Install]`, no timer `Description`, no `WantedBy`, so `systemctl --user enable` would do nothing. Follow this task's directive list and what `systemd-analyze verify`
accepts, and record the deviation under `## Decisions`.

## Decisions

## Result

| Field | Value |
|---|---|
| Commit | |
| Red evidence | |
| Tests added / suite total | |
| Runtime or request count | |
| Deviations | |

