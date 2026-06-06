# Upgrading a self-hosted chizzle

A chizzle release can change the SurrealDB life-graph schema, the
`config.toml` shape, the SQLite pairing/session DBs, or the on-disk
persona layout. `docker compose pull && docker compose up -d` is the
happy path, but it is not always enough — and even when it is, you want
a rollback ready before you find out.

This document is the runbook. Every release tag gets a section under
[Per-release notes](#per-release-notes), even if it's "no migrations" —
the release workflow [enforces this](#release-time-enforcement).

> If you are deploying for the first time, you don't need this file —
> see [`README.md`](README.md). This is the *day-two* document.

---

## 1. What persists where

All persistent state for the compose topology lives in named Docker
volumes. Nothing on the host filesystem; the deployment is portable
across machines as long as the volumes come with it.

| Volume | Owner | Holds |
|---|---|---|
| `chizzle-surreal` | `surrealdb` | The life-graph (RocksDB store). The persona's memory. |
| `chizzle-personas` | `chizzle-server`, `events` | Persona dirs + the persona framework files. Edits made by the persona survive here. |
| `chizzle-shared` | `chizzle-server`, `events` | Built feature bundles + the cross-persona build tooling. |
| `chizzle-data` | `chizzle-server` (RW); `graph-mcp`, `events` (RO) | `config.toml` (wizard-written), the SQLite pairing + web-session DBs. |
| `chizzle-claude-home` | `chizzle-server`, `events` | Claude CLI per-conversation jsonl files. Loss breaks `--resume`. |

The **single bundled image** (`ghcr.io/keeganstothert/chizzle` — the
Fly.io / `docker run` target) collapses all of the above into one
`/data` mount. Snapshot that one mount and you have everything.

---

## 2. Before any upgrade

Take a snapshot of the volumes and record the version you are coming
from. This is the rollback floor: every step below assumes you can
restore to it.

### Compose topology

```bash
# Stop the stack so the snapshot is consistent.
docker compose down

# Record the version you are running. `latest` is not a version —
# resolve it to the digest you actually pulled.
docker compose config | grep image:
docker image inspect ghcr.io/keeganstothert/chizzle-server:${CHIZZLE_VERSION:-latest} \
  --format '{{index .RepoDigests 0}}'

# Snapshot every chizzle volume to a tarball.
mkdir -p ~/chizzle-snapshots
for vol in chizzle-surreal chizzle-personas chizzle-shared \
           chizzle-data chizzle-claude-home; do
  docker run --rm \
    -v "${vol}:/src:ro" \
    -v ~/chizzle-snapshots:/out \
    alpine tar -czf "/out/${vol}-$(date +%Y%m%d-%H%M%S).tgz" -C /src .
done
ls -lh ~/chizzle-snapshots
```

Volumes with `chizzle-` prefixes here assume the default compose
project name. If you ran `docker compose -p <name>` the volumes are
namespaced — `docker volume ls` shows the real names.

### Single bundled image / Fly.io

```bash
# Bundled image — one /data volume holds everything.
docker stop chizzle
docker run --rm \
  -v chizzle-data:/src:ro \
  -v ~/chizzle-snapshots:/out \
  alpine tar -czf "/out/chizzle-data-$(date +%Y%m%d-%H%M%S).tgz" -C /src .

# Fly.io — snapshot the Fly volume.
fly volumes list
fly volumes snapshots create <volume-id>
```

Then read the [release notes for the target version](#per-release-notes)
and decide whether the upgrade is a pull-and-up or needs the manual
steps the release calls out.

---

## 3. Standard upgrade — no migration steps

Most patch/minor releases are pure image bumps. **The supported path is
[`./update.sh`](update.sh)** — it does the §2 snapshot for you, pulls,
restarts, polls `/health`, and prints a one-command rollback pointed at
the snapshot it just took.

```bash
# From the same dir as compose.yml.
curl -O https://raw.githubusercontent.com/keeganstothert/chizzle/main/deploy/update.sh
chmod +x update.sh

# Pull whatever CHIZZLE_VERSION pins (or "latest" if unset):
./update.sh

# Or pin explicitly to a version this same call:
./update.sh --target vX.Y.Z

# Dry-run first if you want to see every command before anything runs:
./update.sh --dry-run
```

`update.sh` runs the same `tar -czf` snapshot from §2 — same volume
names, same archive format — so you can mix-and-match: snapshot by hand,
then `./update.sh --skip-snapshot`, or roll back by hand against an
`update.sh`-produced snapshot using the §4 commands.

If a release requires more than this — a `surrealdb fix` pass, a
config-shape edit, a one-shot script — its [per-release section](#per-release-notes)
spells it out. Read that section *before* you run `./update.sh`.

### Doing it by hand

For an operator who'd rather see the steps, this is what `update.sh`
runs in its default flow:

```bash
# Pin to the new version. `latest` works but defeats reproducibility.
echo 'CHIZZLE_VERSION=X.Y.Z' >> .env   # or edit the existing line

# Re-fetch compose.yml — the release workflow may have bumped pinned
# sub-images (e.g. surrealdb) that the bundled .env override can't reach.
base=https://raw.githubusercontent.com/keeganstothert/chizzle/main/deploy
curl -O $base/compose.yml

# Take the §2 snapshot here.

docker compose pull
docker compose up -d
docker compose logs -f
```

Watch the logs for one minute. Pairing still works? The persona
responds? Then keep the snapshot for a day in case something subtle
shows up; otherwise you're done.

### Bundled-image deployments

`./update.sh --mode bundled` works the same way for the single
`ghcr.io/keeganstothert/chizzle` image: it snapshots `chizzle-data`,
pulls the new tag, and re-creates the `chizzle` container with the
exact same `-v`/`-p`/`-e` flags the previous one was started with
(read back via `docker inspect`). It refuses to run if there isn't an
existing `chizzle` container — first-time installs still need the
initial `docker run` from [README §6](README.md#the-bundled-single-image).

---

## 4. Rolling back to the prior version

If the snapshot was taken by `./update.sh` (§3):

```bash
./update.sh --rollback
```

That restores the most recent snapshot under `~/chizzle-snapshots/` and
re-pins `.env` to the `previous_version` recorded in the snapshot's
`metadata.json`. The §3 caveats still apply (SurrealDB minor downgrades
remain unsafe — see end of this section).

### By hand

You took the snapshots in §2, so this is mechanical.

```bash
# Bring the stack down.
docker compose down

# Restore each volume from its snapshot. This wipes the current
# contents and replaces them with the snapshotted state.
for vol in chizzle-surreal chizzle-personas chizzle-shared \
           chizzle-data chizzle-claude-home; do
  snap=$(ls -t ~/chizzle-snapshots/${vol}-*.tgz | head -n1)
  docker volume rm "${vol}"
  docker volume create "${vol}"
  docker run --rm \
    -v "${vol}:/dst" \
    -v "${snap}:/snap.tgz:ro" \
    alpine sh -c 'cd /dst && tar -xzf /snap.tgz'
done

# Pin back to the prior version.
sed -i.bak 's/^CHIZZLE_VERSION=.*/CHIZZLE_VERSION=PRIOR.X.Y.Z/' .env

docker compose up -d
```

Rolling back across a SurrealDB minor bump (e.g. 3.0 → 3.1 → 3.0) is
**not safe** — RocksDB schema changes are one-way. If the failed
upgrade also bumped `surrealdb/surrealdb:vX.Y.Z` in compose.yml, the
rollback requires restoring the `chizzle-surreal` snapshot *before*
the upgrade ran (which is what §2 captured) and pinning the prior
SurrealDB tag in compose.yml.

---

## 5. Secrets rotation

A leaked secret — `WAKEUP_SHARED_SECRET` exposed in a config backup, a
shell-history grep, a container-layer inspection — does not need a full
wizard re-config. Rotation is a single in-place rewrite of `config.toml`
followed by a restart.

### `WAKEUP_SHARED_SECRET`

The bearer token shared between `events` and `chizzle-server`'s
`/send-as-bot` + `/push-pill` endpoints. Two equivalent paths:

**From the host shell** (recommended — no admin login needed):

```bash
docker compose exec chizzle-server chizzle-server secret rotate wakeup
# Bundled image:
docker exec chizzle chizzle-server secret rotate wakeup
```

The command prints the new secret once. Capture it before scrolling.

**From the wizard's admin API** (useful when the host shell isn't
reachable — VPN-only deployments, a fly app you only have `flyctl ssh`
into):

```bash
# 1. Log in to get an admin session token (10-min idle timeout).
TOKEN=$(curl -fsS https://your-chizzle/setup/api/login \
  -H 'Content-Type: application/json' \
  -d '{"password":"…"}' | jq -r .token)

# 2. Rotate. The new secret comes back once.
curl -fsS -X POST https://your-chizzle/setup/api/wakeup-secret/rotate \
  -H "Authorization: Bearer $TOKEN" | jq -r .wakeup_shared_secret
```

The server schedules a restart 1.5 seconds after the response flushes
(same pattern as `apply`). Expect a ~2-second window of `/send-as-bot`
failures — the ticket chose synchronous-with-restart over a dual-accepts
grace path for simplicity.

After either path:

1. **Restart `chizzle-server`** if you used the CLI (the HTTP path does
   this for you). `docker compose restart chizzle-server`.
2. **Update the events deployment's secret store** if it reads
   `WAKEUP_SHARED_SECRET` from its own env (a separate compose service,
   a sibling Fly app, a Kubernetes secret). `events` resolves with
   env-first precedence, so a stale env var beats a rotated
   `config.toml`. The bundled single-image deployment doesn't have a
   separate events env — the rewrite is enough.
3. **Verify**: tail the `chizzle-server` logs and confirm the next
   `/send-as-bot` call from `events` returns `204`, not `401`.

### Admin password

There is no dedicated rotation route for the admin password — it is one
field in the wizard's reconfigure flow. Open `/setup` in a browser, log
in with the current password, run through "Apply" with a new value.
Hand-edited deployments edit `[setup].admin_password_hash` directly
(use the `argon2` CLI or a one-shot `chizzle-server`-internal call —
the format is the standard Argon2 PHC string).

### AI credentials

Same as the admin password: re-run the wizard, or edit `[env]` in
`config.toml` and restart. AI credentials are env-promoted; a stale
process-env value (set outside `config.toml`) wins over the file, so
clear the env first if it was set there.

---

## 6. Per-release notes

Each release tag gets a section here, in reverse chronological order
(newest first). The release workflow refuses to publish a tag whose
section is missing.

A release-note section must answer:

- **Schema migrations** — anything the operator runs once before / after
  the upgrade? (`surrealdb fix`, a SQL script, a config rewrite.)
- **Config shape** — does an old `config.toml` still parse? If not,
  which fields moved/renamed/were removed.
- **Persona layout** — did files inside `personas/<name>/` move?
- **Breaking compose changes** — new required env, removed services,
  renamed volumes.
- **Rollback** — anything beyond §4 needed to roll back from this tag.

If the answer to all of the above is "no," say so explicitly — that
*is* the note.

### Template for a new release

Copy this into your release PR and fill it in:

```markdown
### vX.Y.Z — YYYY-MM-DD

**Migrations:** none. _(or describe the manual step)_
**Config:** unchanged. _(or list field changes)_
**Persona layout:** unchanged.
**Compose:** unchanged. _(or list added/removed env / services)_
**Rollback:** standard §4 procedure.

_(Optional narrative paragraph for anything that doesn't fit above.)_
```

---

<!-- BEGIN RELEASE NOTES -->

### v0.4.7 — 2026-06-06

The **Telegram surface is removed**. Now that proactive push reaches a
*closed* iOS app end-to-end via the shared relay (v0.4.6), Telegram was no
longer load-bearing for any delivery, and maintaining a second full chat
surface meant every turn-handling, ACL, session, and outbox change had to
be written and tested twice. The native app is the product; this release
retires the bridge. Gone: the ~720-line inbound long-poll dispatcher, the
outbound proactive Telegram fallback rung, the `telegram` cargo feature,
and the `teloxide` dependency (Cargo.lock shrinks ~640 lines). The
proactive dispatch ladder is now web → push relay, with no Telegram tail.

**Migrations:** none.
**Config:** the per-persona Telegram fields (`bot_token`, `allowed_user_ids`,
`allowed_chat_ids`) and the `[server]` `telegram_bot_api_url` / `edit_throttle_ms`
keys are no longer read. **You do not need to edit `config.toml`** — parsing
stays tolerant of these now-unknown keys, so an existing file still loads.
The proactive recipient field `events_chat_id` is **renamed to
`primary_chat_id`** (env `<NAME>__PRIMARY_CHAT_ID`); a serde alias and a
legacy `<NAME>__EVENTS_CHAT_ID` env fallback are kept for **this one
release**, so a running self-host keeps firing proactive events across the
upgrade. Migrate your config/env to the new name before the next release.
**Persona layout:** unchanged.
**Compose:** the chizzle-server Telegram env (`ASSISTANT__BOT_TOKEN`,
`TELEGRAM_BOT_API_URL`, `ASSISTANT__ALLOWED_USER_IDS`,
`ASSISTANT__EVENTS_CHAT_ID`) is removed; the events proactive-recipient
passthrough is renamed to `ASSISTANT__PRIMARY_CHAT_ID`. If you ran a
self-hosted `telegram-bot-api` service alongside the stack, remove it and
its `.env` Telegram block — re-fetch `compose.yml`/`.env.example` (README §3.1).
**Rollback:** standard §4 procedure. Rolling back to a Telegram-capable
binary still works — the config keys it expects were never deleted from
your file.

### v0.4.6 — 2026-06-06

Turnkey proactive push — a closed iOS app now wakes for a persona's
proactive message with **zero config**. Previously a self-hosted server
could only reach a *foregrounded* app or fall back to Telegram; APNs was
never turned on because there was no chizzle-operated relay and no way for
an arbitrary self-hoster to authorize against a shared one. This release
ships both: the chizzle-operated push-relay is real multi-tenant infra,
and the server **auto-enrolls** against it on first boot and persists the
enrollment token to config — the operator sets nothing. A relay token only
ever wakes device tokens the backend already holds via pairing, so open
enrollment can't wake strangers' phones; resource abuse is bounded by
per-backend, per-device, global, and per-IP caps. APNs `Unregistered`
device tokens are pruned automatically. Also hardens proactive events: a
re-fire circuit breaker plus surfaced DB-write errors stop the wakeup
flood where a disk-full graph DB silently dropped the delivery mark and an
event re-fired ~100× (2026-06-05).

**Migrations:** none.
**Config:** proactive push is now **default-on**. On first boot the server
self-enrolls against the shared relay (`DEFAULT_PUSH_RELAY_URL`) and writes
a `[push_relay]` block (backend id + token) to `config.toml` — additive,
backward-compatible, reused on restart. The old "`PUSH_RELAY_URL` set +
secret unset = hard error" is gone. Opt out with `PUSH_RELAY_DISABLE=1`
(or `[push_relay] disabled = true`). Self-hosters operating their *own*
relay set `PUSH_RELAY_URL` to it; see `deploy/fly/relay/README.md`.
**Persona layout:** unchanged.
**Compose:** unchanged.
**Rollback:** standard §4 procedure. The `[push_relay]` config block is
inert to older binaries; no down-migration needed.

### v0.4.5 — 2026-06-04

In-app update check now works on private-source deployments. The server's
"is there a newer release?" probe used to hit the source repo's
`releases/latest` anonymously; once the source repo went private that
404'd and the update banner never appeared. Release *metadata* (tags +
Releases + this runbook) is now published to a separate **public** repo
(`keeganstothert/chizzle-releases`); the probe targets it
(env-overridable via `CHIZZLE_RELEASES_REPO`, defaulting to the public
repo) so it stays anonymous and token-free. Container images are
unchanged — still signed on GHCR, pulled with `docker login`. Also fixes
`deploy/update.sh` writing a `v`-prefixed `CHIZZLE_VERSION` that didn't
match the bare GHCR image tags. Server + tooling only.

**Migrations:** none.
**Config:** unchanged. New optional `CHIZZLE_RELEASES_REPO` env (defaults
to `keeganstothert/chizzle-releases`); leave unset for the standard
deployment.
**Persona layout:** unchanged.
**Compose:** unchanged.
**Rollback:** standard §4 procedure.

### v0.4.4 — 2026-06-04

Persona turn capabilities. Every chat turn and cold-open now carries a
runtime-injected system-prompt fragment (`OUTBOX_SURFACES_FRAGMENT`)
teaching the persona the three on-stage outbox surfaces — the feature
card (`app-card.json`), the topic thread + thread card (`thread.json`),
and smart-bar pills (`pill.json`) — with their exact wire shapes. Before
this, those mechanics lived only in `FRAMEWORK.md`, which a normal turn
never loads, so the persona knew it *should* ship a card but not *how*
(and would hand the user a raw URL) and could wrongly claim threads
"aren't built." Cold-open/event spawns especially need this: they
disallow `Read`, so they can't consult `FRAMEWORK.md` at turn time. Also
corrects a stale line in `personas/CLAUDE.md` that described topic
threads as an unbuilt plan. Server/events only; the fragment ships in the
binary.

**Migrations:** none.
**Config:** unchanged.
**Persona layout:** unchanged — but the seeded framework docs
(`personas/CLAUDE.md`, `FRAMEWORK.md`) changed, and the `chizzle-personas`
volume is seeded only once. Existing deployments keep the old docs; the
behavioral fix (the fragment) ships in the binary regardless. To pick up
the doc edits on an existing volume, sync them in:
`docker compose cp chizzle-server:/opt/chizzle/seed/personas/CLAUDE.md /tmp/ ...`
or edit in place — see the operator note in the release PR. Fresh deploys
seed the new docs automatically.
**Compose:** unchanged.
**Rollback:** standard §4 procedure.

### v0.4.3 — 2026-06-02

Setup-wizard feature: the admin panel now has a **Pair a device** screen
that lists every configured persona and mints a fresh pairing code (QR +
plain code) on demand — the UI replacement for SSHing in to run
`chizzle-server pair create`. It's handled in-process by the running
server, so it reuses the already-open (single-writer) pairing store with
no restart and no lock contention. Two new admin-gated routes back it:
`GET /setup/api/personas` and `POST /setup/api/personas/:id/pair`. The
`pair create` CLI still works for first-boot provisioning but cannot run
against a live server (the store lock); the docs now say so. Wizard +
server only.

**Migrations:** none.
**Config:** unchanged.
**Persona layout:** unchanged.
**Compose:** unchanged.
**Rollback:** standard §4 procedure.

### v0.4.2 — 2026-05-31

Bugfix: scheduled proactive Event delivery was broken on 0.4.0/0.4.1.
The `graph-mcp` `due_events` query ordered by `fields.fire_at` without
selecting it, which SurrealDB v3 rejects (`Missing order idiom`), so
every `events` poll failed and no scheduled wakeups or proactive
messages fired. The query now aliases `fields.fire_at` into its
projection. Server/graph-mcp only; no schema, config, or persona-layout
changes. Upgrade with the standard `docker compose pull && up -d`.

**Migrations:** none.
**Config:** unchanged.
**Persona layout:** unchanged.
**Compose:** unchanged.
**Rollback:** standard §4 procedure.

### v0.4.1 — 2026-05-30

Setup-wizard fix: the "Setup complete" screen now shows one persona
pairing card at a time (prev/next arrows, swipe, dot indicators) and the
pairing QR code scales to its container instead of overflowing and
clipping. Wizard-only; no server, schema, or config changes.

**Migrations:** none.
**Config:** unchanged.
**Persona layout:** unchanged.
**Compose:** unchanged.
**Rollback:** standard §4 procedure.

### v0.4.0 — 2026-05-30

**Migrations:** none for your data. `graph-mcp` now provisions persona
records (name/secret/scopes) lazily and idempotently on first request
(`DEFINE … OVERWRITE`), so existing scoped data attaches automatically
as long as persona ids still slug to the same scope strings. No export/
import step.
**Config:** new **top-level `graph_mcp_url`** (env `GRAPH_MCP_URL`),
shared by every persona, replacing the per-persona
`<PERSONA>__GRAPH_MCP_URL`. `config.toml` may now carry **several
`[[persona]]` blocks** — one shared credential set in `[env]`, **no
per-persona AI keys**. The per-persona env/manifest forms still resolve
first, so single-persona deployments are unchanged.
**Persona layout:** **now multi-persona.** The setup wizard creates up
to five personas (name + prompt each); ids are derived server-side by
slugifying the display name. Hand-edited `config.toml` adds a
`[[persona]]` block per persona.
**Compose:** **re-fetch `compose.yml` required.** The per-persona
`graph-mcp-<id>` services collapse into a **single `graph-mcp`** that
serves every persona — the persona id rides each request as an
`X-Chizzle-Persona` header. `chizzle-server` and `events` now target
`GRAPH_MCP_URL: http://graph-mcp:8000/mcp` (replacing
`<PERSONA>__GRAPH_MCP_URL`); `PERSONA` stays as the no-header fallback.
`compose.multi-persona.yml` is **removed**. Optional seam auth: set
`WAKEUP_SHARED_SECRET` and graph-mcp rejects (`401`) any request without
a matching `X-Chizzle-Graph-Secret`; leave it unset for the prior
unauthenticated behaviour (logged as a boot `warn`). The Fly single-VM
path likewise switched to `GRAPH_MCP_URL` so its one graph-mcp serves
all personas.
**Rollback:** standard §4 procedure. Rolling back to a per-persona-
container compose also requires restoring `compose.multi-persona.yml`
from the prior tag.

The setup wizard configured exactly one persona, hard-wired to id
`assistant`, because each persona was bound to its own `graph-mcp`
*container* fixed at boot by the `PERSONA` env — the wizard can't create
containers. v0.4.0 makes `graph-mcp` **persona-aware**: one container
opens a per-persona scoped SurrealDB session on demand, so a single
deployment serves any number of personas and the wizard creates them
transparently. The life-graph's record-level scope isolation (the real
security boundary) is unchanged; persona identity now travels on each
request instead of being implied by which container is addressed.

### v0.3.4 — 2026-05-28

**Migrations:** none.
**Config:** env var rename on `graph-mcp`: `LOTTIE_CATALOG_PATH` →
`LOTTIE_CORPUS_PATH`. The Dockerfile sets the new var automatically;
operators only need to act if they override it explicitly in their
own compose/env.
**Persona layout:** unchanged. Already-seeded personas pick up the new
stage-Lottie behaviour without a re-seed — the system-prompt fragment
is injected at runtime.
**Compose:** unchanged.
**Rollback:** standard §4 procedure.

Stage Lotties (the animated facial expression above the chat) are now
tag-driven: the persona calls a single `set_lottie(tag, chat_id, mode?)`
MCP verb with one of 96 expressive tags (`thinking`, `searching`,
`celebrate`, …) and the server picks a matching animation at random.
Replaces the prior two-step flow (`find_lottie` search + `lottie.json`
outbox write) — both retired. The image bakes a new
`/opt/chizzle/lottie-corpus.json` artifact in place of the old
`lottie-catalog.json`.

### v0.3.3 — 2026-05-27

**Migrations:** none.
**Config:** unchanged. `GITHUB_TOKEN` is now forwarded from `.env` to
`chizzle-server` by default (v0.3.2 added the code path but missed the
compose forwarding line — operators on v0.3.2 had to inline it
themselves).
**Persona layout:** unchanged.
**Compose:** **re-fetch `compose.yml` recommended** if you want
`GITHUB_TOKEN` to reach `chizzle-server` without a manual edit. v0.3.3
adds one line to both `compose.yml` and `compose.multi-persona.yml`:
`GITHUB_TOKEN: ${GITHUB_TOKEN:-}` under the `chizzle-server` service's
`environment:` block. The corresponding `.env.example` comment lives
under "GitHub Releases probe auth (optional)". Operators on public
repos can ignore this — anonymous probe works.
**Rollback:** standard §4 procedure.

One-line follow-up to v0.3.2: the released `deploy/compose.yml` didn't
forward the `GITHUB_TOKEN` env var into the `chizzle-server` container,
so v0.3.2's probe-auth code path was unreachable from compose-deployed
self-hosts without a manual edit. v0.3.3 closes that gap.
Single-bundled-image / Fly.io deployments aren't affected — their
process env is the host env.

### v0.3.2 — 2026-05-27

**Migrations:** none.
**Config:** unchanged. Optional `GITHUB_TOKEN` env on `chizzle-server` —
if set, the GitHub Releases probe sends it as a `Bearer` header
(needed only for private-repo deployments or to lift the 60/hr
anonymous rate limit; classic and fine-grained PATs both work).
**Persona layout:** unchanged.
**Compose:** new optional `CHIZZLE_SERVER_CONTAINER` env on
`chizzle-updater` (defaults to `chizzle-server`; the sidecar's
`/status` endpoint inspects this container for the running image
digest). The default matches v0.3.1 behaviour — only operators
running an alternate-name stack need to set it. Re-fetching
`compose.yml` is optional; the pin is purely informational.
**Rollback:** standard §4 procedure.

Bug-fix patch on top of v0.3.1 — restores `/api/updates/apply` for
deployments where it was reachable in name but broken in practice:

- **`chizzle-server` IPC flush fix.** v0.3.1's hand-rolled
  HTTP-over-Unix-socket client called `stream.shutdown()` after
  writing the request; on the kernel/tokio combination shipping in
  the v0.3.1 image this closed *both* directions of the socket
  before the sidecar could read the request, so every
  `/api/updates/apply` call surfaced as `502 updater transport:
  missing CRLFCRLF in response`. Replaced with `flush()`; both ends
  use `Connection: close` + explicit `Content-Length`, so the server
  dispatches on length and doesn't need a write-half EOF.
- **`chizzle-server` GitHub Releases probe — optional `GITHUB_TOKEN`
  auth.** The probe called `api.github.com` anonymously, returning
  `latest: null` (and hiding the update banner) for any deployment
  pointed at a private repo. The probe now sends `Authorization:
  Bearer $GITHUB_TOKEN` when the env var is set.
- **`chizzle-updater` `CHIZZLE_SERVER_CONTAINER` override.** The
  hard-coded `chizzle-server` container name in v0.3.1's `/status`
  handler is now an env-driven default; alternate-name stacks
  (isolated verification deploys, etc.) can override it without
  patching the image.
- **`update.sh` cross-project preflight.** A stopped or `Created`
  container from a *different* compose project that squats on one of
  this stack's `container_name`s (`chizzle-server`,
  `chizzle-surrealdb`, etc.) made `docker compose up -d` fail
  mid-flight with an opaque `Conflict. The container name '/X' is
  already in use` error. `update.sh` now surfaces the offender +
  remediation up-front, before the snapshot + pull have already run.
- **`ui-apple` settings — bearer copy-row.** Adds a "Bearer / Tap to
  copy" row under PAIRING for operator smoke ops (mint a token in
  Settings, paste into curl). Native-app affordance; no server
  contract change.

### v0.3.1 — 2026-05-27

**Migrations:** none.
**Config:** unchanged.
**Persona layout:** unchanged.
**Compose:** unchanged. `./update.sh` re-pulls the `:0.3` / `:latest`
tags; hand-managed deployments can bump `CHIZZLE_VERSION` to `0.3.1`
in `.env` (or stay on `0.3` / `latest` and `docker compose pull`).
**Rollback:** standard §4 procedure.

Patch on top of v0.3.0 to clear `CVE-2026-33671` (picomatch < 4.0.4)
on the main `chizzle` image. The CVE was reachable through the bundled
copy at `/usr/lib/node_modules/npm/node_modules/picomatch` — npm's
own vendored dep, not anything in our `bun.lock` (which already
resolved picomatch to 4.0.4). The fix upgrades global npm to current
upstream during image build via `npm install -g npm@latest`, which
replaces npm's bundled picomatch. The `chizzle-updater` image's
residual advisories are unchanged from v0.3.0 (SPLIT_UPDATER D4).

### v0.3.0 — 2026-05-26

**Migrations:** none.
**Config:** unchanged.
**Persona layout:** unchanged.
**Compose:** **breaking — re-fetch `compose.yml` mandatory.** v0.3.0
re-splits `chizzle-updater` back out into its own image at
`ghcr.io/keeganstothert/chizzle-updater:0.3.0` (the namespace that
existed pre-v0.2.1) and drops the `entrypoint:` override on the
sidecar service — the updater image's default entrypoint is the
binary. The main `ghcr.io/keeganstothert/chizzle` image no longer
carries the docker CLI or docker-compose plugin (~110 MB smaller),
which closes the v0.2.1 CVE bucket from those bundled binaries on
every non-updater container. **Operator must re-fetch `compose.yml`
by hand** (see §3.1 — `./update.sh` does not fetch compose.yml on its
own) before running `docker compose up -d` or `./update.sh`.
**Rollback:** standard §4 procedure. Volume contents are unchanged
across the split.

The split is intentional: the privileged sidecar that mounts
`/var/run/docker.sock` is now the *only* container whose attack
surface includes the docker stack — privilege boundary and CVE
boundary line up. Both images are signed under the same tag
(`v0.3.0` / `0.3` / `latest`) and ship from a single
[`deploy/Dockerfile`](Dockerfile) via different `--target` values, so
one `chz release publish` still produces both. See
[`plans/completed/SPLIT_UPDATER.md`](https://github.com/keeganstothert/chizzle/blob/main/plans/completed/SPLIT_UPDATER.md)
for the full rationale.

**Residual advisories on `chizzle-updater:0.3.0`.** The updater image
bundles `docker-ce-cli` + `docker-compose-plugin` at their latest
upstream pinned versions; at release time those packages carry two
HIGH advisories that have no fixed upstream release:

- `CVE-2026-46680` — containerd `runAsNonRoot` evasion path inside
  the compose plugin's embedded gobinary.
- `CVE-2026-34040` — docker authz plugin bypass in the same
  gobinary.

Both are upstream-tracked and reachable only by code paths the
updater itself exercises (the docker socket). The
[main `chizzle` image is the trivy-gated artifact for this release](https://github.com/keeganstothert/chizzle/blob/main/plans/completed/SPLIT_UPDATER.md#resolved-decisions-phase-0)
(per SPLIT_UPDATER D4); operators who want zero-CVE sidecars can run
`docker compose stop chizzle-updater` and remove the service from
`compose.yml` — `/api/updates/apply` returns 503 cleanly and the
host-side `./update.sh` path keeps working.

### v0.2.1 — 2026-05-26

**Migrations:** none.
**Config:** unchanged.
**Persona layout:** unchanged.
**Compose:** **breaking — re-fetch `compose.yml` mandatory.** v0.2.1
collapses the five-image fanout (`chizzle-server`, `chizzle-graph-mcp`,
`chizzle-push-relay`, `chizzle-updater`, plus the fly-bundled
`chizzle`) into a single `ghcr.io/keeganstothert/chizzle` image. Every
chizzle service in compose now points at that one image and picks a
binary via a per-service `entrypoint:` override; the four retired image
names keep their v0.1.x tags on GHCR but **will not** gain a `:latest`
pointing at v0.2.1 — `docker pull
ghcr.io/keeganstothert/chizzle-server:latest` (or any of the four
retired names) returns 404 after this release. **Operator must
re-fetch `compose.yml` by hand** (see §3.1 — `./update.sh` does not
fetch compose.yml on its own) before running `docker compose up -d`
or `./update.sh`.
**Rollback:** standard §4 procedure. The pre-upgrade snapshot still
restores cleanly under the v0.1.x compose shape — volume contents are
unchanged across the rename.

First release through the consolidated path: one Dockerfile
([`deploy/Dockerfile`](Dockerfile)), one `cargo build --release` per
release, one signed image. fly.io and the compose stack now pull the
same image; fly overrides the entrypoint to the baked supervisor at
`/usr/local/bin/chizzle-fly-start`. No application behaviour changes
since v0.2.0 (which was an internal-testing cut and was never published
through the 5-image flow — `:latest` jumps straight from v0.1.1 to
v0.2.1).

### v0.2.0 — 2026-05-26

**Migrations:** none.
**Config:** unchanged.
**Persona layout:** unchanged.
**Compose:** unchanged.
**Rollback:** standard §4 procedure.

Internal-testing cut of the new `chz release` pipeline (CHZ_RELEASE
Phase 6). First release produced from gamebox rather than
`.github/workflows/release.yml`; first release signed against the
long-lived cosign key at `deploy/cosign.pub` rather than keyless OIDC.
Multi-arch is temporarily amd64-only — operators on arm64 build from
source until the GHA-runs-`chz` follow-up lands. No application
behaviour changes since v0.1.1.

### v0.1.1 — 2026-05-26

**Migrations:** none.
**Config:** unchanged.
**Persona layout:** unchanged.
**Compose:** **new service** — `chizzle-updater` sidecar plus two named
volumes (`updater-sock`, `updater-state`). Operators running a
compose.yml from before this release must pull the updated
`deploy/compose.yml` (or replace just the `chizzle-updater` service
block and the new `volumes:` entries) before `update.sh` will bring up
the full stack. The sidecar is optional — `docker compose stop
chizzle-updater` + removing the service makes `/api/updates/apply`
return 503 cleanly, falling back to the host-side `update.sh` path.
**Rollback:** standard §4 procedure.

First release with the end-to-end "Updates" surface: the wizard's
AdminHome shows an "Update available" card (notify + copy
`./update.sh`), and the iOS app gains a chat-surface banner plus a
Settings → Updates view with an in-app **UPDATE NOW** flow. The
chizzle-updater sidecar is the only container that holds the Docker
socket; chizzle-server brokers the apply request over a Unix socket
shared between the two containers, so the server process never inherits
root-on-host privileges.

### v0.1.0 — 2026-05-25

**Migrations:** none.
**Config:** unchanged.
**Persona layout:** unchanged.
**Compose:** unchanged.
**Rollback:** standard §4 procedure.

First tagged chizzle release. From here forward the supported upgrade
flow is [`deploy/update.sh`](./update.sh) — see §3. Self-hosters running
older `git pull && docker compose up --build` setups should migrate to
the GHCR-pull path documented in [README.md §5](./README.md) before the
next release.

<!-- END RELEASE NOTES -->

---

## 7. Release-time enforcement

The release runbook lives at [`deploy/README.md` §6](README.md). The
gate that requires a `### vX.Y.Z` heading in this file is now enforced
by [`chz release version`](../chz/) — it refuses to bump
`[workspace.package].version`, tag, or push if the new version doesn't
have a section here. There is no GitHub Action; the gate runs locally
on the operator's dev machine *before* the tag exists, so a missing
section is caught before any image is built.

So: **add an entry above `<!-- END RELEASE NOTES -->` before running
`chz release version --bump {patch,minor,major}`.** Even if the answer
is "no migrations." The empty-template form above is the minimum.

This is what makes the runbook trustworthy. A self-hoster reading §6
sees every version that has shipped, not just the ones whose authors
remembered to write a note.
