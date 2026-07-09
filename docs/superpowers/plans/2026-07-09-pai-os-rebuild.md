# PAI-OS Rebuild Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. Execution is approved to run via ultracode workflows.

**Goal:** Rebuild the Nightshift/PAI-OS system on a freshly wiped Pi ("core") running latest upstream NanoClaw with zero local patches, governed by a guardian harness in a new `aig-pai-os` meta-repo.

**Architecture:** A meta-repo (`aig-pai-os`) holds the harness (knowledge base + guardian subagent + review skill + weekly self-update) and a manifest/sync script for all sibling repos. The Pi is rebuilt from verified backups (bare repos, OneCLI vault) plus a fresh `pai-v1` branch cut from upstream/main. Features return only afterward, one harness-vetted iteration at a time.

**Tech Stack:** git/gh, bash, launchd (Mac), systemd (Pi), Docker + OneCLI, NanoClaw v2 (Node host / Bun container), age encryption, Claude Code headless (`claude -p`).

**Spec:** `docs/superpowers/specs/2026-07-09-pai-os-rebuild-design.md`

## Global Constraints

- Fork: `git@github.com:mashkovtsevlx/aig-ncv0526-nanoclaw.git`; upstream: `https://github.com/nanocoai/nanoclaw.git`; new base branch name: `pai-v1`.
- Base install carries **zero local patches** — Telegram via stock `/add-telegram` only.
- Repos on core (bare): `~/git/aig-ncv0526-{soul,workspace,infra}.git`; GitHub org/user: `mashkovtsevlx`, all mirrors **private**.
- Hosts: Mac = `alex@<this machine>`, core = `alex@core.local` (fallback IPs `192.168.8.194` eth / `192.168.8.195` wlan). Old-Pi SD remains the deep rollback — never touch it.
- OneCLI restore invariants: `onecli_app-data` volume must be restored byte-for-byte; image pinned to digest `ghcr.io/onecli/onecli@sha256:5662d27b5941…` (full digest read from core's `~/.onecli/docker-compose.yml` during backup); `ONECLI_BIND_HOST=172.17.0.1` in `~/.onecli/.env`; postgres role is `onecli`.
- Secrets: age-encrypted only; recipient key read from existing `infra` usage (`age1ngkqu…ulrc`); private key never leaves Alex's devices; no plaintext secrets in git, ever.
- Destructive steps (re-flash, deletions in Phase 6, stopping live day-claw) are **[HUMAN]-gated**: prepare + verify automatically, act only on Alex's confirmation.
- Verification gates are hard: Phase 3 cannot start before the Phase 2 gate passes; Phase 5 cannot start before the BASE gate passes.
- Timezone for all schedules: Asia/Makassar (WITA).

---

## Phase 0+1 — Foundation: meta-repo + guardian harness

### Task 1: Scaffold `aig-pai-os` meta-repo

**Files:**
- Create: `~/projects/aig-pai-os/` (git init), `README.md`, `repos.json`, `sync.sh`, `.gitignore`
- Create: private GitHub repo `mashkovtsevlx/aig-pai-os`

**Interfaces:**
- Produces: `repos.json` schema `{ "repos": [{ "name", "github", "core", "branch" }] }` and `./sync.sh [name…]` — consumed by Tasks 2–5, Phase 6, and all future dev sessions.

- [ ] **Step 1: Init repo and manifest**

```bash
mkdir -p ~/projects/aig-pai-os && cd ~/projects/aig-pai-os && git init -b main
cat > repos.json <<'EOF'
{
  "repos": [
    { "name": "soul",      "github": "git@github.com:mashkovtsevlx/aig-ncv0526-soul.git",      "core": "alex@core.local:git/aig-ncv0526-soul.git",      "branch": "main" },
    { "name": "workspace", "github": "git@github.com:mashkovtsevlx/aig-ncv0526-workspace.git", "core": "alex@core.local:git/aig-ncv0526-workspace.git", "branch": "main" },
    { "name": "infra",     "github": "git@github.com:mashkovtsevlx/aig-ncv0526-infra.git",     "core": "alex@core.local:git/aig-ncv0526-infra.git",     "branch": "main" },
    { "name": "nanoclaw",  "github": "git@github.com:mashkovtsevlx/aig-ncv0526-nanoclaw.git",  "core": "",                                               "branch": "pai-v1" }
  ]
}
EOF
printf 'repos/\n.DS_Store\n' > .gitignore
```

- [ ] **Step 2: Write `sync.sh`**

```bash
cat > sync.sh <<'EOF'
#!/usr/bin/env bash
# Clone-or-pull all PAI-OS sibling repos into ./repos/<name>.
# Prefers core (LAN origin); falls back to the GitHub offsite mirror.
# Usage: ./sync.sh [name…]   (no args = all)
set -euo pipefail
cd "$(dirname "$0")"
command -v jq >/dev/null || { echo "jq required (brew install jq)"; exit 1; }
mkdir -p repos
names=("$@")
while IFS=$'\t' read -r name github core branch; do
  if [ ${#names[@]} -gt 0 ]; then
    case " ${names[*]} " in *" $name "*) ;; *) continue ;; esac
  fi
  dest="repos/$name"
  src="$core"
  [ -z "$src" ] || git ls-remote "$src" >/dev/null 2>&1 || src="$github"
  [ -n "$src" ] || src="$github"
  if [ -d "$dest/.git" ]; then
    echo "== pull $name ($branch)"
    git -C "$dest" fetch --all --prune
    git -C "$dest" checkout "$branch"
    git -C "$dest" pull --ff-only
  else
    echo "== clone $name from $src"
    git clone --branch "$branch" "$src" "$dest"
    [ "$src" = "$github" ] || git -C "$dest" remote add offsite "$github"
  fi
done < <(jq -r '.repos[] | [.name, .github, .core, .branch] | @tsv' repos.json)
echo "sync complete."
EOF
chmod +x sync.sh
```

- [ ] **Step 3: README + first commit**

Write `README.md`: what the meta-repo is, the Mac policy (only `aig-pai-os` + one nanoclaw checkout are permanent; soul/workspace fetched on demand into `repos/`, disposable), how to bootstrap a new Mac (`git clone && ./sync.sh`), pointer to `harness/` and `docs/`.

```bash
git add -A && git commit -m "scaffold: manifest, sync.sh, README"
```

- [ ] **Step 4: Create private GitHub repo and push**

```bash
gh repo create mashkovtsevlx/aig-pai-os --private --source ~/projects/aig-pai-os --push
```

- [ ] **Step 5: Verify** — `./sync.sh soul infra` clones both (workspace repo doesn't exist yet — expected to fall back/skip until Task 6); `gh repo view mashkovtsevlx/aig-pai-os --json visibility` → `PRIVATE`. Note: nanoclaw entry only syncs after Task 8 creates `pai-v1`.

- [ ] **Step 6: Copy spec + this plan into `aig-pai-os/docs/` and commit** (the fork branch holding them is being retired).

### Task 2: Build the harness knowledge base

**Files:**
- Create: `harness/kb/00-overview.md`, `harness/kb/architecture-invariants.md`, `harness/kb/entity-model.md`, `harness/kb/runtime-and-supply-chain.md`, `harness/kb/customization-doctrine.md`, `harness/kb/nightshift-guardrails.md`
- Source material: fresh clone of latest `upstream/main` at `repos/nanoclaw-upstream/` (read-only), `~/Downloads/nightshift-bootstrap.md`, project memory `nightshift-v0.md`

**Interfaces:**
- Produces: KB docs cited by name from the guardian agent (Task 3) and harness-check skill (Task 4). Every normative claim must carry a `file:line` citation into the upstream repo or runbook.

- [ ] **Step 1: Clone reference upstream**

```bash
cd ~/projects/aig-pai-os && git clone --depth 50 https://github.com/nanocoai/nanoclaw.git repos/nanoclaw-upstream
```

- [ ] **Step 2: Write the six KB docs** (execution: fan out one explorer agent per doc; each reads the relevant upstream sources and returns the doc with citations). Required content per doc:
  - `00-overview.md` — what PAI-OS is (day-claw/night-claw/soul), repo map, where the harness fits, how to update the KB.
  - `architecture-invariants.md` — everything-is-a-message (no IPC/file-watchers between host and container); two-DB split with exactly one writer per file; host even / container odd `seq` parity; heartbeat = file touch; `journal_mode=DELETE` is load-bearing cross-mount; central DB owns entities, session DBs own message flow. Cite `docs/db*.md`, `src/session-manager.ts`, `container/agent-runner/src/db/connection.ts`.
  - `entity-model.md` — users → messaging groups → agent groups → sessions; privilege is user-level (owner/admin) not group-level; `cli_scope` tiers; approval routing order (scoped admin → global admin → owner). Cite `docs/isolation-model.md`, `src/modules/permissions/access.ts`, `src/command-gate.ts`.
  - `runtime-and-supply-chain.md` — Node host vs Bun container, no shared modules; `minimumReleaseAge` 3 days, `onlyBuiltDependencies` human-gated, `--frozen-lockfile` in automation; bun named-param `$name` gotcha; container build-cache gotcha; pinned global CLIs in Dockerfile.
  - `customization-doctrine.md` — **skills-first: never patch trunk when a skill/config/mount can do it**; channels/providers come from sibling branches via `/add-*` skills; per-group behavior belongs in `groups/<g>/CLAUDE.local.md` and container_configs, not source; keep fork diff ≈ 0; any source change must state why no skill could do it and be re-checked against upstream on every update.
  - `nightshift-guardrails.md` — Pi coordinates / Mac computes; `main` is the human's, night commits to `night/<date>` branches; SOUL.md + `system/` PROTECTED (PR + DECISIONS.md, never self-merge); one increment per night; documented failure > hidden failure; kill switch (`~/git/STOP`) + timeout + 3-attempt cap live outside the agent; secrets age-encrypted only; sandbox boundary is the container, which is why `--dangerously-skip-permissions` inside it is by design.

- [ ] **Step 3: Verify citations** — for each doc, check every cited path exists in `repos/nanoclaw-upstream` (or the runbook): `grep -oE '[a-zA-Z0-9_./-]+\.(ts|md|sh|json)' harness/kb/*.md | sort -u` and test existence. Fix or drop dead citations.

- [ ] **Step 4: Commit** — `git add harness/kb && git commit -m "harness: knowledge base v1 (from upstream@<sha>)"` recording the upstream SHA in the message.

### Task 3: Guardian subagent

**Files:**
- Create: `.claude/agents/nanoclaw-guardian.md`

**Interfaces:**
- Produces: agent invocable as `nanoclaw-guardian`, consumed by harness-check (Task 4) and all future feature work.

- [ ] **Step 1: Write the agent definition**

```markdown
---
name: nanoclaw-guardian
description: Use BEFORE designing, implementing, or merging any change to nanoclaw, soul, infra, or workspace. Checks the proposal against the PAI-OS harness knowledge base and reports principle violations with citations. Also use when unsure whether something should be a skill, a config change, or a source patch.
tools: Read, Grep, Glob, Bash
---

You are the PAI-OS guardian. Your knowledge base is `harness/kb/*.md` in the aig-pai-os repo (locate it relative to this file). You NEVER implement changes; you review them.

Given a proposed change (diff, design, or description):
1. Read all six KB docs.
2. Classify the change: skill/config-level, per-group customization, infra, or trunk source patch.
3. For each KB principle it touches, verdict: COMPLIES / VIOLATES / NEEDS-DECISION, each with the KB doc + the specific rule quoted.
4. Trunk source patches get extra scrutiny: state whether a skill, container_config, mount, or CLAUDE.local.md entry could achieve the same result. If yes → VIOLATES customization-doctrine.
5. End with a verdict block: APPROVE / APPROVE-WITH-CHANGES (list them) / REJECT (why, and the compliant alternative).

Be strict. A convenient violation now is a broken invariant during the next migration. If the KB itself seems wrong or stale, say so explicitly and recommend a KB update instead of silently deviating.
```

- [ ] **Step 2: Verify** — from `~/projects/aig-pai-os`, dispatch the agent with a known-bad test proposal ("edit src/router.ts in the fork to hardcode a Telegram chat id for routing") and confirm it REJECTs citing `customization-doctrine.md`. Dispatch a known-good one ("add a per-group CLAUDE.local.md instruction") and confirm APPROVE.

- [ ] **Step 3: Commit.**

### Task 4: harness-check skill

**Files:**
- Create: `.claude/skills/harness-check/SKILL.md`

**Interfaces:**
- Consumes: `nanoclaw-guardian` agent (Task 3), KB docs (Task 2).
- Produces: `/harness-check [diff|path|description]` used as the mandatory pre-merge gate in Phase 5.

- [ ] **Step 1: Write the skill**

```markdown
---
name: harness-check
description: Mandatory pre-merge gate for PAI-OS work. Reviews a diff, design doc, or change description against the harness KB via the nanoclaw-guardian agent. Use before implementing a roadmap iteration and again before merging it.
---

# Harness Check

1. Determine the subject: an explicit argument (diff/file/description), else the working tree diff (`git diff` + `git diff --cached`), else the current branch vs main.
2. Dispatch the `nanoclaw-guardian` agent with the subject and wait for its verdict.
3. Render the verdict table to the user verbatim (principle, verdict, citation).
4. If REJECT or APPROVE-WITH-CHANGES: do NOT proceed to merge/implementation; surface the compliant alternative.
5. Record the outcome as one line appended to `harness/checks.log` in aig-pai-os: `<date> <subject-summary> <verdict>`.
```

- [ ] **Step 2: Verify** — run `/harness-check` on the same two test proposals from Task 3 Step 2; confirm verdicts render and `harness/checks.log` gains two lines.

- [ ] **Step 3: Commit.**

### Task 5: Weekly harness self-update

**Files:**
- Create: `harness/self-update.sh`, `harness/SELF_UPDATE.md`, `~/Library/LaunchAgents/com.aig-pai-os.harness-update.plist`

**Interfaces:**
- Consumes: `repos/nanoclaw-upstream` clone, KB docs.
- Produces: weekly commit `harness: weekly KB refresh (upstream@<sha>)` on `aig-pai-os` main, pushed.

- [ ] **Step 1: Write `harness/self-update.sh`**

```bash
#!/usr/bin/env bash
# Weekly harness KB refresh: pull latest upstream nanoclaw, let Claude diff
# reality against the KB, commit doc updates. Log to harness/self-update.log.
set -euo pipefail
cd "$(dirname "$0")/.."
exec >> harness/self-update.log 2>&1
echo "=== self-update $(date -u +%FT%TZ)"
git pull --ff-only
git -C repos/nanoclaw-upstream pull --ff-only || git clone --depth 50 https://github.com/nanocoai/nanoclaw.git repos/nanoclaw-upstream
SHA=$(git -C repos/nanoclaw-upstream rev-parse --short HEAD)
claude -p --permission-mode acceptEdits --add-dir "$PWD" \
  "You maintain the PAI-OS harness KB at harness/kb/. Upstream nanoclaw is checked out at repos/nanoclaw-upstream (now at $SHA). Read each KB doc, verify every cited file/behavior still matches upstream (docs/, CONTRIBUTING.md, key src files). Update stale claims, add newly-introduced invariants, keep citations accurate. Edit only harness/kb/*.md. If nothing changed, change nothing."
if ! git diff --quiet harness/kb; then
  git add harness/kb && git commit -m "harness: weekly KB refresh (upstream@$SHA)" && git push
else
  echo "KB unchanged."
fi
```

`chmod +x harness/self-update.sh`. Write `harness/SELF_UPDATE.md` documenting: schedule (Mon 10:00 WITA — Mac awake, human around), how to run manually, how to read the log, and that the commit is meant to be skimmed by Alex.

- [ ] **Step 2: Install launchd job**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>com.aig-pai-os.harness-update</string>
  <key>ProgramArguments</key><array>
    <string>/bin/bash</string>
    <string>/Users/alex/projects/aig-pai-os/harness/self-update.sh</string>
  </array>
  <key>StartCalendarInterval</key><dict>
    <key>Weekday</key><integer>1</integer>
    <key>Hour</key><integer>10</integer>
    <key>Minute</key><integer>0</integer>
  </dict>
</dict></plist>
```

```bash
launchctl load ~/Library/LaunchAgents/com.aig-pai-os.harness-update.plist
```

- [ ] **Step 3: Verify** — `launchctl kickstart gui/$(id -u)/com.aig-pai-os.harness-update`, tail `harness/self-update.log` until it completes; confirm either a "KB unchanged." line or a pushed refresh commit.

- [ ] **Step 4: Commit** script + doc + a copy of the plist under `harness/launchd/`.

---

## Phase 2 — Pre-wipe safeguard (hard gate before any flash)

### Task 6: Workspace offsite mirror (spec item 5)

**Files:**
- Create: private GitHub repo `mashkovtsevlx/aig-ncv0526-workspace`
- Modify (on core, in infra repo checkout): the offsite push script (`soul-offsite-push.sh` — locate with `grep -rl offsite ~/git/aig-ncv0526-infra` on core)

- [ ] **Step 1:** `gh repo create mashkovtsevlx/aig-ncv0526-workspace --private`
- [ ] **Step 2:** On core: `cd ~/git/aig-ncv0526-workspace.git && git remote add offsite git@github.com:mashkovtsevlx/aig-ncv0526-workspace.git && git push offsite --all && git push offsite --tags`
- [ ] **Step 3:** Add workspace to the offsite push script following the exact pattern soul/infra already use in that script; commit to infra; push infra (origin + offsite).
- [ ] **Step 4: Verify** — `gh api repos/mashkovtsevlx/aig-ncv0526-workspace/branches --jq '.[].name'` lists `main` and all `night/*` branches present on core.

### Task 7: Backup + salvage + doc-drift fixes

**Files:**
- Create (on core, then age-encrypted into infra): `onecli-app-data-<date>.tgz`, `onecli-pg-<date>.sql`, image-digest note
- Create (in infra): `salvage/dm-with-alex/` (CLAUDE.local.md, transcribe.mjs + note about `pnpm install --ignore-workspace`)
- Modify (in soul): `PROCEDURES.md` (offsite timer says 05:00, match reality)

- [ ] **Step 1: Sync check** — on core, for each bare repo: `git push offsite --all` idempotently; confirm soul/infra/workspace GitHub mirrors current (`git ls-remote` SHAs match).
- [ ] **Step 2: OneCLI export** (on core):

```bash
docker run --rm -v onecli_app-data:/d -w /d alpine tar cz . > ~/onecli-app-data-$(date +%F).tgz
docker exec $(docker ps -qf name=postgres) pg_dumpall -U onecli > ~/onecli-pg-$(date +%F).sql
grep -o 'ghcr.io/onecli/onecli@sha256:[a-f0-9]*' ~/.onecli/docker-compose.yml > ~/onecli-image-digest.txt
grep ONECLI_BIND_HOST ~/.onecli/.env >> ~/onecli-image-digest.txt
```

- [ ] **Step 3: Encrypt into infra** — read the age recipient from the existing encrypt procedure in infra/PROCEDURES notes; `age -r <recipient> -o secrets/onecli-backup-<date>.tgz.age ~/onecli-app-data-<date>.tgz` (same for pg dump); ALSO scp plain copies to the Mac at `~/nightshift/migrate-staging/` as belt-and-braces (Mac is Alex's device — allowed to hold key material). Copy `~/nanoclaw/.env` content check against existing `secrets.env.age` — re-encrypt if drifted.
- [ ] **Step 4: Salvage groups/ state** — from core `~/nanoclaw/groups/dm-with-alex/`: `CLAUDE.local.md` (minus the stale "Location tracking" section and the `rpi.local` vault pointer — drop both per memory), `transcribe.mjs`, plus a `README.md` noting the form-data/`--ignore-workspace` gotcha and the Groq vault-key name ("Grok"). Commit under `infra/salvage/dm-with-alex/`.
- [ ] **Step 5: Fix PROCEDURES.md drift** — in soul: offsite push time (04:00 → actual 05:00 per audit; confirm with `systemctl --user list-timers` on core first). Commit + push soul.
- [ ] **Step 6: GATE (verify restorable)** — on the Mac: `tar tzf ~/nightshift/migrate-staging/onecli-app-data-<date>.tgz | grep -E 'secret-encryption-key|ca.pem'` (both present); `head -5` of pg dump shows valid SQL; `git clone` each GitHub mirror into a temp dir and confirm HEAD SHAs match core. Only after all pass, report **"Phase 2 gate: PASS — safe to flash"** to Alex.

---

## Phase 3 — Clean Pi + base NanoClaw

### Task 8: Cut `pai-v1` from latest upstream

**Files:**
- Create: branch `pai-v1` on the fork (from `upstream/main`, zero local commits)

- [ ] **Step 1:** In the Mac fork checkout: `git fetch upstream && git push origin refs/remotes/upstream/main:refs/heads/pai-v1`
- [ ] **Step 2: Verify** — `gh api repos/mashkovtsevlx/aig-ncv0526-nanoclaw/branches/pai-v1 --jq .commit.sha` equals `git rev-parse upstream/main`. `./sync.sh nanoclaw` in aig-pai-os now clones it.

### Task 9: [HUMAN] Flash + first boot

- [ ] **Step 1 [HUMAN]:** Before flashing, shut down core cleanly (`sudo poweroff`) — day-claw goes down here; Telegram is silent until Task 11. Then: Raspberry Pi Imager → Pi OS Lite 64-bit → the NVMe SSD; hostname `core`, SSH public-key only (Mac's `~/.ssh/id_ed25519.pub`), Wi-Fi `GL-X3000-71f`, timezone `Asia/Makassar`. Boot from the blue USB 3.0 port.
- [ ] **Step 2: Post-boot verify** (from Mac; retry ssh — LAN is flaky): `vcgencmd get_throttled` → `0x0`; `findmnt /` shows the USB/SSD device not `mmcblk0`; `sudo apt update && sudo apt full-upgrade -y && sudo apt install -y git smartmontools age jq`; `sudo smartctl -a /dev/sda | head -40` readable. Install Docker per infra bootstrap (next task).

### Task 10: Restore infra: repos, OneCLI, timers

**Files:**
- Create on core: `~/git/aig-ncv0526-{soul,workspace,infra}.git` (bare), `~/soul` checkout, `~/git/aig-ncv0526-infra` checkout, `~/.onecli/` (compose + .env + restored volume + pg), user systemd units

- [ ] **Step 1: Bare repos** — `for r in soul workspace infra; do git clone --mirror git@github.com:mashkovtsevlx/aig-ncv0526-$r.git ~/git/aig-ncv0526-$r.git; done` then in each: `git remote add offsite git@github.com:mashkovtsevlx/aig-ncv0526-$r.git` (mirror clone sets origin=github; rename so origin semantics match old layout — bare repo IS the origin, offsite = github). Working checkouts: `git clone ~/git/aig-ncv0526-soul.git ~/soul`; `git clone ~/git/aig-ncv0526-infra.git ~/git/aig-ncv0526-infra`. (A deploy key/PAT for GitHub pushes from core is itself a secret — restore it from the age backup per infra PROCEDURES.)
- [ ] **Step 2: Run infra `bootstrap.sh`** (read it first; it should install Docker, timers, sensor collector — fill gaps by hand and **commit the fixes back to infra** so the next rebuild is one script).
- [ ] **Step 3: OneCLI restore** — recreate `~/.onecli/docker-compose.yml` pinned to the recorded digest; `.env` with `ONECLI_BIND_HOST=172.17.0.1`; `docker volume create onecli_app-data && cat onecli-app-data-<date>.tgz | docker run --rm -i -v onecli_app-data:/d -w /d alpine tar xz`; start postgres, restore dump as role `onecli`; `docker compose up -d`.
- [ ] **Step 4: Verify OneCLI** — tunnel `ssh -L 10254:172.17.0.1:10254 alex@core.local` → web UI lists agents + secrets; a vault secret decrypts (open one in UI — no "decryption failed"). If Anthropic OAuth is dead anyway, re-auth in the UI now (known 10-minute task, memory documents it).
- [ ] **Step 5: Timers armed but STOPPED for now** — `touch ~/git/STOP` so no night fires mid-rebuild; verify `systemctl --user list-timers` shows nightshift-trigger, briefing-check, soul-offsite (times matching PROCEDURES.md), nanoclaw-sensors.

### Task 11: Fresh NanoClaw install + BASE gate

**Files:**
- Create on core: `~/nanoclaw` (clone of fork `pai-v1`), `.env` from decrypted secrets, systemd user service via `/setup`

- [ ] **Step 1:** `git clone -b pai-v1 git@github.com:mashkovtsevlx/aig-ncv0526-nanoclaw.git ~/nanoclaw` on core.
- [ ] **Step 2:** Run Claude Code on core in `~/nanoclaw`: `/setup` (service, container build, OneCLI wiring — point it at the restored gateway, do NOT re-init the vault), then `/add-telegram` (stock skill; bot token from decrypted secrets), then `/init-first-agent` (owner = `telegram:<Alex's handle>`, agent group `dm-with-alex`).
- [ ] **Step 3: Re-apply salvage** — copy `infra/salvage/dm-with-alex/` files into `groups/dm-with-alex/`; `cd groups/dm-with-alex && pnpm install --ignore-workspace` for transcribe.mjs deps.
- [ ] **Step 4: BASE gate — all must pass, in order:**
  1. Service up, no crash-loop in `logs/nanoclaw.error.log`.
  2. Telegram DM → reply received (routing chain visible in `logs/nanoclaw.log`).
  3. Credential injection: gateway log shows `POST /v1/messages 200 injections_applied=1` (never judge by `/v1/models`).
  4. Container lifecycle clean: spawn, heartbeat file touching, outbound delivery, idle shutdown.
  5. Voice note → Groq transcription reply (exercises salvage + vault).
  6. Sensors: `groups/homeassistant/sensors.json` fresh (<5 min) — wire the homeassistant group only if it was part of base; otherwise defer to Phase 5 and note it.
  7. Reboot core; everything above still true after boot.
- [ ] **Step 5:** Report **"BASE gate: PASS"** with evidence lines to Alex. Tag the moment: `git -C ~/nanoclaw tag base-v1-verified && git push origin base-v1-verified`.

---

## Phase 4 — Identity & trust fixes

### Task 12: Mount the soul (spec item 4)

**Files:**
- Modify: mount allowlist (via `/manage-mounts` on core) + `container_configs` for group `dm-with-alex` (via `ncl groups config update`)
- Modify: `soul/PROCEDURES.md` (+ `groups/dm-with-alex/CLAUDE.local.md` note telling the agent `/soul` exists and what SOUL.md/MEMORY.md are)

- [ ] **Step 1:** Run `/harness-check` on this change first (it touches container config — should APPROVE as config-level).
- [ ] **Step 2:** On core: add `/home/alex/soul` to the mount allowlist (`/manage-mounts`), then `ncl groups config get --id <dm-with-alex-id>` to see the exact `additionalMounts` schema, then update config mapping `/home/alex/soul` → `/soul` **read-write** (day-claw may append to MEMORY.md/inbox; PROTECTED files are protected by merge rules + git, not mount flags — night pushes and human merges remain the write path to main).
- [ ] **Step 3:** `ncl groups restart --id <dm-with-alex-id> --message "You now have /soul mounted — read /soul/SOUL.md and /soul/MEMORY.md."`
- [ ] **Step 4: Verify** — `docker inspect` of the fresh container shows the bind; DM day-claw "what does SOUL.md say?" and get a real answer.
- [ ] **Step 5:** Document in `soul/PROCEDURES.md`: the identity model is now real (chassis mounts the soul), the exact config commands, and the git-pull cadence question (checkout freshness) as a ROADMAP candidate. Commit soul.

### Task 13: SSH forced command (spec item 6)

**Files:**
- Modify: Mac `~/.ssh/authorized_keys` (core's key line)

- [ ] **Step 1:** Identify core's key line: `grep -n "$(ssh alex@core.local cat .ssh/id_ed25519.pub | awk '{print $2}' | cut -c1-24)" ~/.ssh/authorized_keys` (match by key body, not comment).
- [ ] **Step 2:** Prefix that line with `restrict,command="/Users/alex/nightshift/run-night.sh" ` (keep the key intact; `restrict` disables pty/forwarding/X11).
- [ ] **Step 3: Verify both directions** — from core: `ssh alex@<mac> echo pwned` must NOT echo (runs run-night.sh instead — with STOP still in place from Task 10 it should log "STOP present, skipping" and exit; confirm in `~/nightshift/logs/`); and the legitimate Timer-A invocation still works (manually: `ssh alex@<mac> anything` → run-night.sh starts). Check run-night.sh tolerates being invoked with an ignored client command string.
- [ ] **Step 4:** Record the exact authorized_keys line format in `infra/` docs; commit.

---

## Phase 5 — Feature roadmap + iterations

### Task 14: Roadmap planning pass (spec item 7)

**Files:**
- Modify: `soul/ROADMAP.md`

- [ ] **Step 1:** Dispatch a planning agent with: `nightshift-bootstrap.md`, all six KB docs, and the seeded-candidates list below. Output: `ROADMAP.md` rewritten as tiered `- [ ]` checkboxes, one `**NEXT UP:**` pointer (left where Alex last set it or proposed-with-question), every item small enough for one night/one sitting and annotated with which KB doc constrains it.
- Seeded candidates (verify-first, then implement only if needed):
  1. **Delivery-isolation fix** — check `repos/nanoclaw-upstream/src/delivery.ts` for per-session try/catch (upstream reworked delivery: see `ac9535b`, `53e1989`); port 1537f66 only if the hole still exists.
  2. **Task-retention prune** — check upstream `src/modules/scheduling/` for completed-row pruning; port dc49b49 only if absent.
  3. **Telegram deltas** — diff stock adapter vs local `telegram-markdown-sanitize.ts` + `telegram-pairing.ts`; keep only what stock lacks, as a skill/patch reviewed by the guardian.
  4. Mac-sleep fix for the 02:00 window (pmset schedule vs caffeinate window).
  5. Timer-B alert path decision (age key on core vs alternative) — a DECISIONS.md entry, human decides.
  6. Night-branch merge hygiene + NEXT-UP pointer flow.
  7. Soul checkout freshness on core (pull cadence for `~/soul`).
  8. Remove STOP + first scheduled night re-armed (explicitly last).
- [ ] **Step 2:** `/harness-check` the roadmap itself (guardrails: one increment per night, PROTECTED rules referenced).
- [ ] **Step 3:** Commit to soul on a branch `roadmap/2026-07`; Alex merges (ROADMAP NEXT-UP is human-owned).

### Task 15: Execute iterations (repeating template — one per roadmap item)

For each item Alex points NEXT-UP at: `/harness-check` the design → implement (TDD where it's code: failing test → minimal fix → pass; fixes to the fork go on a feature branch off `pai-v1`) → `/harness-check` the diff → PR → Alex merges → deploy (`git pull` + service restart or `/update-skills` on core) → verify live → journal entry. This template is the plan for all of Phase 5; no fixed task list — the roadmap drives it.

---

## Phase 6 — Mac cleanup (last)

### Task 16: Sweep and retire stale checkouts

**Files:**
- Remove (each gated on verification): `~/projects/nightshift/{soul,workspace,infra}`, `~/nightshift/{soul,workspace}` stale clones, `~/nightshift/migrate-staging/` plaintext backups, any other project strays the sweep finds
- Keep: `~/projects/aig-pai-os` (+ on-demand `repos/`), ONE nanoclaw checkout, `~/nightshift/` runtime (run-night.sh, SANDBOX.md, selftest.sh, scripts, logs, night image)

- [ ] **Step 1: Sweep** — `find ~ ~/projects -maxdepth 3 -name .git -type d 2>/dev/null` filtered to project-related paths (match `soul|workspace|infra|nanoclaw|nightshift|ncv0526|pai`); for each hit produce: path, remotes, `git status --porcelain` count, unpushed commits (`git log --branches --not --remotes --oneline`), stashes (`git stash list`).
- [ ] **Step 2: Resolve** — anything with unpushed work or stashes: push it or surface to Alex; NEVER delete unpushed work. The known one-commit-ahead `~/projects/nightshift/soul` gets pushed or diffed-and-dropped per content.
- [ ] **Step 3: Decide the fork checkout** — either retire `~/projects/nanoclaw-rpi-ncv0526` in favor of `aig-pai-os/repos/nanoclaw` (on `pai-v1`) as THE checkout, or keep it and re-point it to `pai-v1`. Recommend the former; the old branch stays on GitHub for history. Ask Alex only if unpushed local state makes it ambiguous.
- [ ] **Step 4 [HUMAN-gated]:** Present the kill list with evidence (per-repo: clean? pushed? mirrored where?); delete only after Alex confirms. `~/nightshift/migrate-staging/` plaintext OneCLI backups are deleted here too (the age-encrypted copies in infra remain).
- [ ] **Step 5:** Update `aig-pai-os/README.md` "Mac layout" section to match final reality; update project memory (`nightshift-v0.md`) with the new topology; commit both.

---

## Self-review notes

- Spec coverage: item 1 → Tasks 2–5; item 2 → Task 1 (+16); item 3 → Tasks 8–11; item 4 → Task 12; item 5 → Task 6; item 6 → Task 13; item 7 → Tasks 14–15; added phase → Task 16. Doc-drift + salvage from spec Phase 2 → Task 7.
- Ordering hazards encoded: STOP file up during rebuild (Task 10) and removed only as the last roadmap item; Phase 2 gate before flash; BASE gate before Phase 5; deletions last and human-gated.
- Environment-dependent details (exact digest, age recipient, offsite script path, additionalMounts schema) are deliberately read-at-execution from the live systems that own them, with the command to read them given inline.
