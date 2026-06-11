# Crabbox as the Proof-Capture Substrate for local-mantis Editions

**Author:** Chisel (Product)
**Date:** 2026-06-10
**Repo:** PsiClawOps/crabbox (our fork)
**Companion:** `mantis_architecture_review.md` in PsiClawOps/local-mantis
**Status:** Adaptation plan. No code changed in this pass.

---

## TL;DR

Crabbox already is the substrate `local-mantis` runs on. The proof rig leases a
**local-container** Docker box, builds the system-under-test inside it, and films
a real channel exchange. This document specifies how to make that relationship
**first-class and pluggable**: treat each `local-mantis` channel variant
(Telegram today, **clickclack next**) as a swappable **proof adapter** that rides
on Crabbox's existing Docker lifecycle (`local-container` provider + `job` +
`checkpoint` + `desktop`/`browser` + `artifacts`).

The key insight: Crabbox's lifecycle primitives map almost one-to-one onto the
phases the `local-mantis` scripts hand-roll today. We can replace most of the
bespoke host orchestration with native Crabbox commands, and express each
channel edition as a **Crabbox job + a checkpoint image + a capture script**. The
channel-specific part shrinks to "which client to film and which driver sends the
prompt."

We do **not** need to self-host the Crabbox broker for any of this. Everything
runs **direct** through the `local-container` provider against the local Docker
daemon. The companion research (below, §6) confirms `local-container` (aliases
`docker`, `container`, `local-docker`) is the supported zero-cloud path.

---

## 1. How local-mantis Uses Crabbox Today (and where it hand-rolls)

The `local-mantis` scripts predate a tight Crabbox integration, so they do some
things by hand that Crabbox now does natively:

| local-mantis does by hand | Crabbox-native equivalent |
|---|---|
| `crabbox job run proof-<pr>` leases + builds the box | ✅ already Crabbox (`job`) |
| Custom `preflight-host.sh` (disk, Docker-VM free, poller) | Partial: `crabbox doctor`, but disk-footprint preflight is bespoke and worth keeping |
| `stage-box.sh` unpacks creds via base64-in-payload | `crabbox cp` / `crabbox run --download` / cache volumes |
| In-box `Xvfb + openbox + Telegram` launched by script | `crabbox warmup --desktop --browser` provisions Xvfb/XFCE/x11vnc/ffmpeg/browser |
| `ffmpeg x11grab` + crop + palette GIF by hand | `crabbox artifacts collect` (screenshots, MP4, trimmed GIFs, contact sheet) + `crabbox desktop record/proof` |
| `docker commit` checkpoint by hand | `crabbox checkpoint create --mode native` (docker-commit), `checkpoint fork` |
| `docker exec -i … base64` artifact export (Windows cp bug workaround) | `crabbox cp` / `crabbox run --download remote=local` |
| Driving input (scroll/click) via raw xdotool | `crabbox desktop click/type/paste/key` |

The takeaway: roughly half of what the scripts do by hand is now a native
Crabbox command. The adaptation is partly **deleting bespoke orchestration** in
favor of Crabbox primitives, and partly **formalizing the channel as an adapter**.

---

## 2. Crabbox Lifecycle Mapped to Proof Phases

The proof rig has a fixed lifecycle. Crabbox has primitives for each step:

```
PROOF PHASE                         CRABBOX PRIMITIVE
─────────────────────────────────  ──────────────────────────────────────────
preflight (disk/engine/auth)        crabbox doctor  (+ bespoke footprint gate)
lease a disposable box              crabbox warmup --provider local-container
                                      --desktop --browser --slug proof-<id>
restore a prepared image            crabbox checkpoint fork <chk>   (docker-commit)
sync the SUT source / build PR      crabbox run --id ... (sync) / --fresh-pr
stage credentials / state           crabbox cp  /  cache volumes
launch the observer (client UI)     crabbox desktop launch --browser --url ...
verify a real window exists         crabbox desktop doctor
drive the scenario                  crabbox desktop type/paste/click  (UI driver)
                                      OR in-box API driver via crabbox run
record while driving                crabbox desktop record / artifacts video
still + motion proof                crabbox artifacts collect --all
pull artifacts to host              crabbox cp / run --download / artifacts collect
save the prepared box for reuse     crabbox checkpoint create --mode native
tear down                           crabbox stop <slug>   (lifecycle-cleanup)
```

Every row except the two bespoke ones (footprint preflight, in-box API driver)
is a documented Crabbox command. That is the integration surface.

---

## 3. The Adapter Model

Define a **proof adapter** as the channel-specific triple:

```
adapter = {
  observer:  how to launch + authenticate the client UI we film,
  driver:    how to send the scenario prompt and wait for the reply,
  render:    crop rectangle + GIF trim window + scroll/anchor-to-reply,
}
```

Everything else — the box, the build, the gateway, the recording, the artifact
bundle, the teardown — is **adapter-independent** and owned by Crabbox + a thin
shared harness.

### Adapter registry (proposed)

```
local-mantis/adapters/
  telegram/        # the existing rig, refactored to the adapter contract
    observer.sh    # restore tdata, launch tdesktop, geometry
    driver.py      # telegram-user-driver wrapper (send + anchored wait)
    render.env     # CROP=..., GIF_SS=..., GIF_T=..., anchor=deep-link
  clickclack/      # the new target
    observer.sh    # launch chromium @ clickclack web client, token/profile auth
    driver.(py|ts) # clickclack send/wait (API first, UI optional)
    render.env     # CROP=..., GIF_SS=..., GIF_T=..., anchor=scroll/url-fragment
```

A single `capture.sh` (channel-agnostic) sources the adapter's three files and
runs the fixed pipeline. Adding a channel = adding one `adapters/<name>/`
directory. This is the "different local-mantis editions as adapters" the brief
asks for, expressed concretely.

### Crabbox job per edition

Each edition gets a `.crabbox.yaml` job so the whole capture is one command:

```yaml
# .crabbox.yaml (in the SUT repo or the local-mantis repo)
jobs:
  proof-clickclack:
    provider: local-container          # docker, zero-cloud, direct
    desktop: true                      # Xvfb/XFCE/x11vnc/ffmpeg
    browser: true                      # chromium binary + env
    idleTimeout: 30m
    shell: true
    command: >
      CI=true bash adapters/clickclack/observer.sh &&
      bash capture.sh clickclack "$LABEL" "$SCENARIO"
    stop: always

  proof-telegram:
    provider: local-container
    desktop: true
    browser: false                     # uses tdesktop, not a browser
    idleTimeout: 30m
    shell: true
    command: >
      CI=true bash adapters/telegram/observer.sh &&
      bash capture.sh telegram "$LABEL" "$SCENARIO"
    stop: always
```

`crabbox job run proof-clickclack` then leases the Docker box, provisions the
desktop+browser, runs the adapter's observer, runs the shared capture, and stops
the box. Reuse a warm/checkpointed box with `crabbox job run --id <slug>`.

---

## 4. Checkpoint Strategy (docker-commit)

The single biggest cost saver. Our fork already ships **native local-container
checkpoints** (docker-commit) with image-digest identity and daemon-scope
validation (see CHANGELOG 0.28.0). Use it:

```sh
# build a prepared box once: deps + browser/tdesktop + GUI libs + ffmpeg
crabbox warmup --provider local-container --desktop --browser --slug proof-base
crabbox run --id proof-base --shell 'CI=true ./adapters/clickclack/provision.sh'
# verify a real, non-black window renders before trusting the image
crabbox desktop doctor --id proof-base
crabbox screenshot --id proof-base --output /tmp/precheck.png   # LOOK at it

# commit the prepared box as a reusable checkpoint image
crabbox checkpoint create --id proof-base --mode native --name clickclack-deps

# every subsequent capture forks from the image (seconds, not a full provision)
crabbox checkpoint fork <chk_id> --slug proof-run-1
```

This replaces the hand-rolled `docker commit` + the "no `timeout` wrapper or the
paused container leaks" gotcha — Crabbox's checkpoint path handles the commit,
labels, and daemon-scope validation, and `fork` relocates the workspace and
replays the daemon scope so a changed ambient Docker config can't pick the wrong
engine.

**Caveat from the docs:** native local-container checkpoints are rejected when
the workspace lives in a mounted volume (docker commit omits mounted data). Keep
the SUT source synced into the container (Crabbox default), not bind-mounted, so
the checkpoint captures it.

---

## 5. Capture & Artifacts (replace hand-rolled ffmpeg)

The `local-mantis` scripts hand-roll `ffmpeg x11grab` + crop + palettegen GIF.
Crabbox's `artifacts` + `desktop` commands do this natively and produce a
**publishable bundle**:

```sh
# record while the driver runs (desktop record == artifacts video)
crabbox desktop record --id proof-run-1 run.mp4 --record-duration 40s

# OR collect the full proof bundle in one call
crabbox artifacts collect --id proof-run-1 --all --output artifacts/proof-run-1
# bundle = screenshots + MP4 + trimmed GIF + desktop doctor + webvnc status + metadata
```

Two integration choices:

- **Use Crabbox artifacts wholesale.** Simpler, gives us the contact sheet,
  metadata, and a manifest. The crop-to-conversation-column step (the part that
  makes the GIF tight) may need a post-process, since Crabbox's GIF is
  full-desktop. Add a `--crop` option to `artifacts`/`desktop record`, or
  post-crop with one ffmpeg call in the harness.
- **Keep the hand-rolled crop/GIF, drop everything else.** Use Crabbox for
  lease/build/desktop/teardown, keep the ~10 lines of calibrated crop+palette
  GIF from `mantis-echo.sh` (channel-agnostic, already tuned for <10MB). Lower
  risk, preserves the hard-won GIF sizing.

**Recommendation:** start with the second (keep the proven crop/GIF), and file a
follow-up to add a `--crop WxH+X+Y` flag to `crabbox desktop record` /
`crabbox artifacts video` so the conversation-column crop becomes first-class.
That single flag is the only thing standing between "Crabbox artifacts do
everything" and today's hand-rolled step.

> Note on publishing: the crabbox skill says never push proof images to a product
> repo branch. For local-mantis proof, export artifacts to the host and attach
> them to the PR directly (or use `--storage local`); do not commit GIFs/PNGs
> into the SUT repo.

---

## 6. Running Crabbox on Docker — The Definitive Path

Researched against the Crabbox provider docs and the local-container source. The
question "what's the best way to run crabbox ourselves on docker" has a clean
answer:

**Use the `local-container` provider. Do not self-host the broker.**

### Why local-container (not docker-sandbox, not a self-hosted broker)

- **`local-container`** (aliases `docker`, `container`, `local-docker`) runs
  leases as Linux containers on the local Docker daemon, with **Crabbox-managed
  SSH + rsync sync + desktop/browser/VNC/screenshot + docker-commit
  checkpoints**. This is the provider the proof rig needs, because it gives us a
  filmable desktop and the full sync/artifact workflow. Runs **direct** from the
  CLI; no coordinator involvement.
- **`docker-sandbox`** is a *different* provider that shells out to the
  standalone `sbx` CLI. It is delegated-run only: **no SSH, no desktop, no
  browser, no VNC, no rsync** in v1. Wrong tool for filming a UI. Do not confuse
  the two — `docker`/`container`/`local-docker` resolve to `local-container`;
  `docker-sandbox` has no aliases.
- **Self-hosting the Worker broker** is only for shared-team brokered cloud
  (spend caps, GitHub OAuth, active-lease limits). The proof rig is single-host
  and local; the broker adds nothing and the docs explicitly say local-only use
  is simpler and fully supported without it.

### Minimal setup

```sh
# 0. prerequisite: a running Docker-compatible runtime
docker info

# 1. zero-config one-shot (leases a debian:bookworm container, syncs, runs)
crabbox run --provider local-container -- pnpm test

# 2. warm a reusable desktop+browser box for capture work
crabbox warmup --provider local-container --desktop --browser --slug proof-base
crabbox desktop doctor --provider docker --id proof-base
crabbox screenshot --provider docker --id proof-base --output desktop.png
crabbox webvnc --provider docker --id proof-base          # local noVNC over SSH
crabbox stop --provider docker proof-base
```

### Configuration (repo `.crabbox.yaml` or `~/.config/crabbox/config.yaml`)

```yaml
provider: local-container
localContainer:
  runtime: docker          # detects docker/podman; set explicitly if both exist
  image: debian:bookworm   # or a prebuilt image with deps to skip bootstrap
  user: crabbox
  workRoot: /work/crabbox
  cpus: 0                  # 0 = runtime default
  memory: ""              # e.g. 8g
  network: bridge
  dockerSocket: false      # set true ONLY if in-box commands need host docker
```

### Operational notes specific to our use

1. **Prebuilt image beats `debian:bookworm` bootstrap.** The default image
   bootstraps SSH/Git/rsync/desktop/browser on first start (slow). Bake a
   prebuilt image (or a Crabbox checkpoint) with all deps + the GUI libs the
   proof needs — including `libopengl0`/`libglvnd0`/`libgl1-mesa-dri`, without
   which the filmed app maps no window and captures are black (the single most
   expensive local-mantis gotcha).
2. **`--desktop` provisions the capture stack for free:** Xvfb, XFCE, x11vnc,
   xdotool, screenshot tools, ffmpeg, noVNC, websockify — no systemd. This is
   exactly the L6 capture layer the scripts install by hand.
3. **`--browser` provides a Chromium binary + env** (`BROWSER`, `CHROME_BIN`)
   and pins a per-lease profile. This is the clickclack observer's foundation.
   It does **not** log a profile in; layer the authenticated session on top
   (token auth, or a restored profile via `crabbox cp` / cache volume).
4. **Checkpoints are docker-commit.** `--mode native` required (auto keeps the
   archive default). Don't bind-mount the workspace (commit omits mounted data).
5. **Socket pass-through (`dockerSocket: true`) only if needed.** It mounts the
   host Docker socket into the lease (in-box `docker` works) but breaks the
   host-isolation boundary. The proof rig doesn't need nested Docker, so leave
   it off.
6. **Cleanup is local and label-based.** `crabbox stop <slug>` removes the
   container; `crabbox cleanup --provider docker` sweeps stale non-`keep`
   containers past idle timeout + grace. No coordinator, so no brokered expiry —
   we own teardown. Keep the local-mantis discipline of stopping stale proof
   boxes (the 409/competing-poller incident was a stale box left running).
7. **Disk discipline still applies.** A full host disk corrupts the Docker
   VHDX. Keep a footprint preflight (the one bespoke gate worth preserving)
   before dual-build capture batches.

### Why not the brokered cloud providers for this

For *product* proof we could film on Hetzner/AWS desktop leases (coordinator
WebVNC, bigger boxes). But for the iterate-fast local loop, `local-container` is
zero-cost, zero-credential, and fast. Recommendation: **local-container for
development and routine proof; a brokered cloud desktop only for hero demos** or
when the box needs more capacity than the laptop's Docker VM. Same adapter, same
capture script — only the `provider:` line changes. That portability is a
feature of building on Crabbox instead of raw `docker run`.

---

## 7. Implementation Plan (crabbox-side)

Most work is in `local-mantis` (the adapter + harness). The crabbox-side work is
small and additive:

### Phase A — Validate the local-container capture path (½ day)
- [ ] `crabbox warmup --provider local-container --desktop --browser`, launch
      Chromium, `desktop doctor`, screenshot — confirm a non-black window.
- [ ] Confirm `crabbox desktop record` produces a usable MP4 of the browser.
- **Artifact:** a screenshot + short MP4 of Chromium in a local-container box.

### Phase B — Checkpoint the prepared image (½ day)
- [ ] Provision deps + GUI libs, `crabbox checkpoint create --mode native`,
      `checkpoint fork`, verify the fork renders (non-black).
- **Artifact:** a `clickclack-deps` checkpoint that forks in seconds.

### Phase C — Add a crop flag to capture (1 day, optional but high-value)
- [ ] Add `--crop WxH+X+Y` (and `--gif-trim ss,t`) to
      `crabbox desktop record` / `crabbox artifacts video` so the
      conversation-column crop + tight GIF become first-class instead of a
      post-process. Provider-neutral; lives in `internal/cli`.
- [ ] Tests beside the code (`*_test.go`), table-driven.
- **Artifact:** `crabbox artifacts collect --all --crop ...` emits a cropped
      <10MB GIF directly.

### Phase D — Job templates (½ day)
- [ ] Ship `proof-clickclack` / `proof-telegram` job examples in docs (and a
      `.crabbox.yaml` template in local-mantis).
- **Artifact:** `crabbox job run proof-clickclack` works end to end on Docker.

### Phase E — Docs (½ day)
- [ ] This doc + a short `docs/recipes/local-mantis-proof.md` recipe linking the
      local-container, desktop, checkpoint, and artifacts features into one flow.
- **Note:** per the repo's product-positioning rule, keep crabbox core/docs
      provider-neutral; the OpenClaw/clickclack-specific recipe is fine as an
      explicitly-scoped recipe page, but do not thread channel-specific logic
      into provider code.

**Total crabbox-side:** ~3 days, and Phase C is the only code change; the rest is
validation + docs. The bulk of the build is the adapter/harness in local-mantis.

---

## 8. Boundary: What Lives Where

Respecting this fork's architecture rule (keep core provider-neutral, no
project-specific logic in providers):

| Concern | Owner |
|---|---|
| Docker lease lifecycle, sync, desktop, browser, VNC, checkpoint, artifacts | **Crabbox** (provider-neutral) |
| `--crop` / `--gif-trim` capture options | **Crabbox** (provider-neutral CLI) |
| Channel observer (which client, how to auth) | **local-mantis adapter** |
| Channel driver (send prompt, wait for reply) | **local-mantis adapter** |
| Crop rectangle + GIF window values | **local-mantis adapter** (`render.env`) |
| SUT gateway config (channels.clickclack, streaming) | **local-mantis harness** |
| OpenClaw/clickclack naming | **local-mantis only** (never in crabbox core/docs except a scoped recipe) |

This keeps Crabbox a generic remote-execution tool (its stated positioning) while
giving local-mantis everything it needs to express editions as adapters.

---

## 9. Summary

- Crabbox's `local-container` provider is the correct, supported, zero-cloud way
  to run the proof rig on Docker. No broker self-hosting needed.
- Crabbox's `job` + `checkpoint (docker-commit)` + `desktop`/`browser` +
  `artifacts` primitives already cover ~half of what local-mantis hand-rolls;
  adopt them and delete the bespoke orchestration.
- Model each channel as a **proof adapter** (observer + driver + render) over a
  shared, channel-agnostic capture harness and a Crabbox job. Clickclack is the
  next adapter; Telegram becomes the reference adapter.
- The only crabbox **code** change worth making is a `--crop`/`--gif-trim`
  capture option so the tight conversation-column GIF is first-class. Everything
  else is validation + docs + the local-mantis adapter work.
- Keep crabbox core provider-neutral; keep the channel/product specifics in
  local-mantis.
