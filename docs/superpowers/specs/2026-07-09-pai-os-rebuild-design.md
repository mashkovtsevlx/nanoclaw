# PAI-OS Rebuild — Design Spec

**Date:** 2026-07-09
**Status:** Approved by Alex (phases 0–6)
**Home:** This spec is committed to the nanoclaw fork for now; it moves to `aig-pai-os/docs/` in Phase 1 (the fork branch `nanoclaw-rpi-ncv0526` is being retired).

## Goal

Rebuild the Nightshift/PAI-OS system from the ground up on the new Pi ("core"), on the latest upstream NanoClaw, governed by a guardian harness that encodes NanoClaw's principles and prevents us from violating them as we build features on top.

## Context (facts this design relies on)

- Current fork `mashkovtsevlx/aig-ncv0526-nanoclaw`, branch `nanoclaw-rpi-ncv0526`, is **241 commits behind** `nanocoai/nanoclaw` with ~4 local commits: telegram adapter (2497374), delivery-isolation fix (1537f66), task-retention fix (dc49b49), template drop.
- Upstream now installs Telegram via the stock `/add-telegram` skill — the local adapter patch is likely obsolete as a base customization.
- day-claw currently runs live on core (Pi 4 8GB, NVMe boot). Old Pi is retired; its SD is the deep rollback.
- Bare repos on core: `~/git/aig-ncv0526-{soul,workspace,infra}.git`. GitHub private mirrors exist for soul + infra only; **workspace has no offsite**.
- OneCLI migration is known to require three things beyond pg_dump: the `onecli_app-data` volume (secret-encryption-key + gateway CA), the **pinned image digest** (`ghcr.io/onecli/onecli@sha256:5662d27b5941…`), and `ONECLI_BIND_HOST=172.17.0.1`.
- Known gaps from the 2026-07-09 audit: soul not mounted into any day-claw container ("chassis mounts the soul" unimplemented); core's SSH key on the Mac has a full shell (no forced command); soul-offsite timer runs 05:00 while PROCEDURES.md says 04:00; Mac sleeps through the 02:00 night window; Timer B is log-only.

## Decisions (locked with Alex)

1. **Fork strategy:** fresh branch (`pai-v1`) cut from latest `upstream/main`. Base install carries **zero local patches**. The two host fixes and any telegram deltas are re-evaluated and, if still needed, re-applied later as Phase-5 feature iterations under harness review.
2. **Pi wipe scope:** preserve identity + secrets — bare repos, OneCLI vault (volume + pg dump + pinned image), Telegram bot token. Fresh OS, fresh NanoClaw, same soul, same credentials.
3. **Repo layout:** new private meta-repo **`aig-pai-os`** with a repos manifest + `sync.sh`; soul/workspace do not live permanently on the Mac.
4. **Harness home:** the meta-repo. Knowledge base + guardian subagent + review skill + weekly self-update job.

## Phases (strictly ordered)

### Phase 0 — Guardian harness

Form: **knowledge base (md docs) + subagent + review skill**. Not a hook — principle enforcement needs judgment, not a mechanical gate.

- `aig-pai-os/harness/kb/*.md`, built by an explorer agent reading the **latest upstream nanoclaw repo** (docs/, CONTRIBUTING.md, source) plus the Nightshift runbook and project memory. Covers:
  - Architecture invariants: everything-is-a-message, two-DB session split, one-writer-per-file, seq parity, heartbeat-as-file-touch, journal_mode=DELETE.
  - Entity model and privilege model (user-level roles, cli_scope tiers).
  - Runtime split (Node host / Bun container), supply-chain policy (minimumReleaseAge, onlyBuiltDependencies, frozen lockfiles).
  - Customization doctrine: **skills-first — never patch trunk when a skill/config can do it**; keep the fork diff near zero.
  - Nightshift guardrails: Pi coordinates / Mac computes; main is the human's; PROTECTED files (SOUL.md, system/) merge only by human; one increment per night; kill switch outside the agent.
- `.claude/agents/nanoclaw-guardian.md` — subagent answering "does this change violate a principle?", citing KB files.
- `.claude/skills/harness-check/` — skill to review a diff/design against the KB before merge.
- **Weekly self-update:** scheduled headless job on the Mac (agent-scheduler/launchd) — fetch upstream, diff architecture/docs vs KB, commit KB updates to aig-pai-os. Output = a skimmable git commit.

### Phase 1 — Meta-repo `aig-pai-os`

Private GitHub repo `mashkovtsevlx/aig-pai-os`:

```
aig-pai-os/
├── repos.json          # manifest: name, GitHub URL, core bare path, default branch
├── sync.sh             # clone-or-pull all sibling repos into repos/
├── harness/kb/*.md
├── .claude/agents/, .claude/skills/
└── docs/               # runbook, this spec, plans
```

Mac policy: only `aig-pai-os` and the nanoclaw fork checkout are permanent on the Mac. Soul and workspace are fetched on demand via `sync.sh` and disposable locally; core bare repos + GitHub mirrors are the truth.

### Phase 2 — Pre-wipe safeguard (includes item 5: workspace offsite)

All before any flash:

1. Create private `mashkovtsevlx/aig-ncv0526-workspace`; add `offsite` remote to core's bare `workspace.git`; add workspace to `soul-offsite-push.sh`; push and verify.
2. Verify soul + infra offsite mirrors current; push unpushed night branches.
3. Export OneCLI vault: `onecli_app-data` volume tar, `pg_dump`, record pinned image digest. age-encrypt exports into `infra/secrets/`.
4. Salvage `groups/` runtime state worth keeping (dm-with-alex CLAUDE.local.md sections, `transcribe.mjs` + its deps note) into infra or soul.
5. Fix doc drift: soul-offsite timer time vs PROCEDURES.md.
6. Verify backups restorable (tar listable, repos fetchable from GitHub) — hard gate before Phase 3.

### Phase 3 — Clean Pi + base NanoClaw (verification-gated)

1. **[HUMAN]** flash Pi OS Lite 64-bit to NVMe (hostname `core`, SSH pubkey-only, Asia/Makassar).
2. Verify hardware: `get_throttled` = 0x0, SSD boot, smartctl baseline; apt update; install git/smartmontools/age/docker.
3. Restore: bare repos, OneCLI (volume restore, pinned image, `ONECLI_BIND_HOST=172.17.0.1`), systemd timers via infra `bootstrap.sh`.
4. NanoClaw: cut `pai-v1` from latest `upstream/main`, push to fork; clone on core; `/setup`; `/add-telegram` (stock); `/init-first-agent`.
5. **BASE gate:** day-claw replies over Telegram DM; credential injection confirmed on `/v1/messages` (not `/v1/models`); container spawn/heartbeat/delivery clean; sensors collector re-wired; timers armed. Phase 5 does not start until this gate passes.

### Phase 4 — Identity & trust fixes (items 4, 6)

- **Soul mount:** `additionalMounts` entry mapping core's `~/soul` checkout to `/soul` in the owner agent group's container, implementing "the chassis mounts the soul". Record the decision and mechanics in PROCEDURES.md so plan and reality agree.
- **SSH forced command:** core's key in the Mac's `authorized_keys` gets `restrict,command="/Users/alex/nightshift/run-night.sh"` — core can trigger nights, nothing else.

### Phase 5 — Feature roadmap + iterations (item 7)

A planning agent reads `nightshift-bootstrap.md` + the harness KB and produces `ROADMAP.md` iterations — small, one per increment, harness-vetted, human keeps the NEXT-UP pointer. Seeded candidates:

- Re-apply **delivery-isolation** and **task-retention** fixes iff still needed against latest upstream (verify first — upstream delivery changed substantially).
- Telegram sanitize/pairing deltas vs the stock adapter, if the stock one lacks them.
- Mac-sleep fix for the 02:00 night window (pmset schedule / caffeinate).
- Timer-B alert path decision (age key on core vs alternative).
- Night-branch merge hygiene; NEXT-UP pointer flow.

Every iteration passes through `harness-check` before implementation.

### Phase 6 — Mac cleanup (last, after everything verified)

Sweep `~` and `~/projects` for project-related checkouts; for each: confirm fully pushed, then remove or consolidate so only the sanctioned layout remains:

- **Keep:** `~/projects/aig-pai-os` (meta-repo, with on-demand `repos/`), one nanoclaw fork checkout, `~/nightshift/` runtime (run-night.sh, sandbox, image — the night worker itself, not a dev copy).
- **Retire:** `~/projects/nightshift/{soul,workspace,infra}` dev copies, `~/nightshift/{soul,workspace}` stale clones (recreated fresh by run-night.sh or sync.sh as designed), any other strays found in the sweep.
- Nothing is deleted without verifying its content is pushed/reachable from core or GitHub; anything ambiguous is listed for Alex instead of deleted.

## Error handling / rollback

- The wipe proceeds only after Phase 2's verification gate.
- If the fresh install fails, core is restorable from the same backups; the old Pi SD remains the deep rollback for day-claw.
- Phase 6 deletions are verification-gated per item; ambiguity → ask, don't delete.

## Success criteria

- Harness KB exists, self-updates weekly, and both the guardian agent and harness-check skill are invocable from the meta-repo.
- `git clone aig-pai-os && ./sync.sh` reproduces the full dev environment on any Mac.
- Fresh core: BASE gate green on stock upstream + `/add-telegram`, zero local patches.
- Soul visible at `/soul` inside the owner group container; PROCEDURES.md matches reality.
- Core's key on the Mac can only run `run-night.sh`.
- Workspace history survives the Pi dying (offsite verified).
- ROADMAP.md holds harness-vetted iterations; local fixes re-applied through that flow, not hand-merged.
- `~` and `~/projects` contain only the sanctioned checkouts.
