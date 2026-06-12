# Adapting Crabbox for local-mantis proof editions

Status: architecture proposal
Date: 2026-06-12
Repos reviewed: `PsiClawOps/crabbox`, `PsiClawOps/local-mantis`
Goal: adapt Crabbox's Docker and lifecycle model so different local-mantis editions can run as proof adapters inside Crabbox

## Executive summary

Crabbox already has the right primitives for a PsiClawOps proof system:

- Disposable execution through providers.
- A local Docker-backed SSH lease provider, `local-container`.
- Desktop and browser capability flags.
- Sync, run logs, artifacts, screenshots, VNC, and cleanup.
- Jobs that compose warmup, optional hydration, run, downloads, and stop policy.
- An `external` provider for privately versioned lifecycle adapters.
- A provider authoring model if we later want a first-class `local-mantis` provider.

The best first architecture is not to make local-mantis a new Crabbox provider. The best first architecture is to run local-mantis editions as Crabbox jobs on `local-container`, backed by prebuilt proof images and a normalized proof artifact manifest.

Once the proof edition contract is stable, Crabbox can add a thin proof integration layer or a dedicated provider. Until then, the provider abstraction is too low-level for local-mantis. local-mantis is not a machine provider. It is a workload and evidence adapter that runs inside a machine.

Recommended path:

1. Use `local-container` plus `desktop: true` and `browser: true` as the default proof runtime.
2. Package each local-mantis edition as a runnable adapter inside the synced repo or a prebuilt proof image.
3. Use Crabbox jobs to select edition, scenario, image, downloads, TTL, and stop policy.
4. Use Docker images or native local-container checkpoints to avoid expensive bootstrap on each proof run.
5. Emit a `proof.json` manifest and make Crabbox downloads/artifacts collect it.
6. Later, add a small `crabbox proof run` command or a `local-mantis` delegated provider only after the edition contract stops moving.

## Research notes: running Crabbox yourself on Docker

The relevant public and local Crabbox material points to this model:

- `local-container` is a local runtime provider that starts a labeled Linux container through Docker-compatible CLIs such as Docker, Podman, OrbStack, Colima, or Lima.
- The local-container provider publishes SSH on loopback, syncs the checkout over SSH, runs commands with the normal Crabbox SSH executor, and removes containers on stop.
- `local-container` supports desktop, browser, screenshot, video, VNC, and WebVNC helpers locally.
- Local-container is the right default for proof capture because it preserves Crabbox SSH sync, artifact downloads, desktop input helpers, and browser helpers in one runtime.
- `provider: docker`, `provider: container`, and `provider: local-docker` are aliases for `local-container`, so the local Docker story is just the local-container story with friendlier names.
- `crabbox doctor` is the right first preflight before a long workflow or after config and token changes.
- `crabbox run` is the core verb: it syncs the dirty checkout, runs the command, streams output back, and exits with the remote exit code. That is the right wrapper for proof editions.
- It can mount the host Docker socket only when `localContainer.dockerSocket: true` is set. This is powerful but weakens isolation because the lease can control the host engine.
- Crabbox direct and local providers do not need the Cloudflare Worker broker. A broker is only required for shared brokered cloud providers such as AWS, Azure, GCP, and Hetzner.
- The clean Docker path for local-mantis is to keep proof execution inside Crabbox jobs on `local-container`, then let the edition adapter own the surface logic and artifact contract.
- Docker Sandbox is a separate delegated-run provider. It is useful when Docker's `sbx` microVM isolation is the product requirement, but it is not the first proof target because it does not expose the same SSH lease, desktop, browser, and download path that local-mantis needs.
- Docker Compose lifecycle hooks can run commands after container start or before stop, but Docker documents that post-start timing is not guaranteed relative to the entrypoint. That means Compose hooks are useful for side tasks, not for readiness-critical proof sequencing.

Key sources:

- Crabbox local-container docs: https://crabbox.sh/providers/local-container.html
- Crabbox Docker Sandbox provider docs: https://crabbox.sh/providers/docker-sandbox.html
- Crabbox provider reference: https://crabbox.sh/providers/index.html
- Crabbox getting started: https://crabbox.sh/getting-started.html
- Crabbox infrastructure docs: https://crabbox.sh/infrastructure.html
- Crabbox provider authoring: https://crabbox.sh/features/provider-authoring.html
- Docker Sandboxes overview: https://docs.docker.com/ai/sandboxes/
- Docker Compose lifecycle hooks: https://docs.docker.com/compose/how-tos/lifecycle/

Search performed on 2026-06-12. The useful material was mostly Crabbox's own local-container and docker-sandbox documentation plus Docker's sandbox and Compose lifecycle references. Generic Docker deployment articles were not specific enough to change the architecture.

## Architecture decision

Do not make local-mantis a Crabbox provider in v1.

Run local-mantis editions as Crabbox jobs on `local-container`, using proof images or native Docker checkpoints to remove bootstrap cost. Crabbox should own the box lifecycle, sync, command execution, downloads, timing, logs, and cleanup. local-mantis should own proof semantics: edition selection, surface driving, observation, validation, and manifest writing.

This keeps the layers clean:

- Crabbox answers: where does the job run, how is it synced, how is it cleaned up, and how do artifacts come home?
- local-mantis answers: what user surface is being proven, how is it driven, what counts as pass/fail, and what proof is emitted?

Only after the edition contract stabilizes should Crabbox grow a thin `proof` command or manifest-aware artifact helper.

## Crabbox architecture points that matter

### Provider types

Crabbox providers fall into three practical families:

1. **SSH lease providers**
   The provider gives Crabbox an SSH target. Crabbox owns sync, command execution, screenshots, desktop helpers, results, and cleanup. `local-container` is in this family.

2. **Delegated run providers**
   The provider owns sync and execution. Crabbox calls the provider and records normalized output. `docker-sandbox`, E2B, Modal, Cloudflare, and similar backends fit here.

3. **External provider**
   Crabbox shells out to a private lifecycle executable or declarative lifecycle config. That external tool owns acquire, resolve, list, release, and connection discovery.

For local-mantis editions, use SSH lease first. We want Crabbox's existing desktop, browser, sync, logs, and download features.

### Local-container behavior

`local-container` is the strongest fit because it already does the following:

- Starts a Docker-compatible container with Crabbox labels.
- Publishes SSH on loopback.
- Creates per-lease SSH keys.
- Installs or verifies SSH, git, rsync, curl, sudo on Debian/Ubuntu images.
- With `--desktop`, installs and starts Xvfb, XFCE, x11vnc, xdotool, screenshot tools, ffmpeg, noVNC, and websockify.
- With `--browser`, installs a package-manager browser and writes browser environment metadata.
- Supports screenshots and video helpers.
- Supports native Docker checkpoints via `docker commit` for faster forks when not using mounted workspaces.
- Cleans up labeled containers through `crabbox cleanup --provider docker`.

That maps directly to proof capture.

### Jobs

Crabbox jobs are the right user-facing control plane for proof editions. A job can specify:

- Provider and target.
- Desktop and browser flags.
- Local-container image.
- TTL and idle timeout.
- Command to run the proof adapter.
- Environment allowlist and env profiles.
- Downloads for proof artifacts.
- Stop policy.

Jobs let local-mantis stay a repo-owned workflow instead of becoming provider code too early.

## Product architecture: local-mantis editions as Crabbox proof adapters

### Definitions

- **Proof edition**: a runnable local-mantis adapter for one surface or mode, such as `telegram-desktop`, `clickclack-web`, or `clickclack-mobile`.
- **Proof scenario**: a JSON file that describes the message, expected visible events, forbidden events, model mode, and capture preset.
- **Proof image**: a Docker image or local-container checkpoint with heavy dependencies installed.
- **Proof manifest**: `proof.json`, the normalized evidence contract consumed by Crabbox, PR tooling, and dashboards.

### Control flow

```text
host repo checkout
  crabbox job run proof-clickclack-basic
    -> local-container lease starts with proof image
    -> Crabbox syncs repo and local-mantis adapter
    -> job command runs edition adapter
    -> adapter starts OpenClaw gateway and Clickclack surface
    -> adapter drives real UI action
    -> adapter records screen and validates behavior
    -> adapter writes /tmp/cap/proof.json and artifacts
    -> Crabbox downloads artifacts to .artifacts/proof/...
    -> Crabbox stops or keeps the lease by job policy
```

### Why not a provider first

A Crabbox provider answers: where do I get a runner and how do I execute commands there?

A local-mantis edition answers: how do I demonstrate behavior on a user surface?

Those are different layers. Making `local-mantis` a provider now would force proof-specific concerns into the runner lifecycle layer. It would also duplicate what jobs already provide.

A provider may make sense later if we want `crabbox run --provider local-mantis-clickclack` to call an external proof service or specialized container fleet. That is not the first milestone.

## Recommended v1: jobs plus proof images

### Repo config shape

Add proof jobs to the local-mantis repo or consuming repos:

```yaml
provider: local-container
target: linux
localContainer:
  runtime: docker
  image: psiclawops/proofrig-clickclack:2026-06-11
  user: crabbox
  workRoot: /work/crabbox
  cpus: 0
  memory: 8g
  network: bridge
  dockerSocket: false

jobs:
  proof-clickclack-basic:
    provider: local-container
    target: linux
    desktop: true
    browser: true
    ttl: 90m
    idleTimeout: 30m
    shell: true
    command: >
      bash proofrig/editions/clickclack-web/run.sh
      proofrig/editions/clickclack-web/scenarios/streaming-basic.json
    downloads:
      - /tmp/cap/proof.json=.artifacts/proof/clickclack-basic/proof.json
      - /tmp/cap/proof-crop.png=.artifacts/proof/clickclack-basic/proof-crop.png
      - /tmp/cap/proof-motion.mp4=.artifacts/proof/clickclack-basic/proof-motion.mp4
      - /tmp/cap/proof-review.gif=.artifacts/proof/clickclack-basic/proof-review.gif
      - /tmp/cap/gateway.log=.artifacts/proof/clickclack-basic/gateway.log
    stop: success
```

### Edition selection

Use a single adapter entrypoint:

```sh
proofrig run --edition clickclack-web --scenario streaming-basic --out /tmp/cap
```

Crabbox does not need to understand the edition. It only needs to run the command and download artifacts.

### Proof image strategy

Use two image layers:

1. **Base proof image**
   Common capture dependencies:
   - OpenSSH, git, rsync, curl, sudo.
   - Node 22, pnpm, Python.
   - Xvfb, ffmpeg, window manager.
   - Chromium and Playwright dependencies.
   - Fonts and image tools.

2. **Edition image or checkpoint**
   Edition-heavy dependencies:
   - Telegram Desktop and tdlib for `telegram-desktop`.
   - Clickclack/browser dependencies for `clickclack-web`.
   - Mobile browser/device emulation dependencies for `clickclack-mobile`.

For fast iteration, use Crabbox native Docker checkpoints where possible:

```sh
crabbox warmup --provider docker --desktop --browser --slug proof-base
# install heavy dependencies inside the box
crabbox checkpoint create --provider docker --id proof-base --mode native proofrig-clickclack-base
crabbox checkpoint fork --provider docker proofrig-clickclack-base --slug proof-run
```

Caveat: local-container native checkpoints use `docker commit` and reject some mounted workspace cases because Docker commit omits mounted data. Keep proof dependencies in the image, not only in mounted workspace volumes.

## Docker Compose role

Docker Compose can help build and run supporting services, but it should not own the proof sequence.

Good Compose uses:

- Build the proof image.
- Run a local Clickclack support stack if needed.
- Provide a disposable database or service dependency.
- Run a local registry or cache.
- Use `pre_stop` to flush optional debug state.

Bad Compose uses:

- Depending on `post_start` to complete readiness before proof begins. Docker documents that post-start timing is not guaranteed relative to the container entrypoint.
- Putting secrets directly in compose files.
- Mounting the Docker socket into proof containers by default.
- Replacing Crabbox lease cleanup with custom container naming and manual cleanup.

Recommended Compose file for image build only:

```yaml
services:
  proofrig-clickclack-image:
    build:
      context: .
      dockerfile: docker/proofrig-clickclack.Dockerfile
    image: psiclawops/proofrig-clickclack:dev
```

Then run proof through Crabbox:

```sh
docker compose build proofrig-clickclack-image
crabbox job run proof-clickclack-basic
```

## Adapter lifecycle inside Crabbox

Each edition should implement this lifecycle:

1. **Preflight**
   Check disk, display tools, browser, ports, credentials, and stale processes.

2. **Stage**
   Copy scenario and optional credentials into `/tmp/cap` with strict permissions.

3. **Start system under test**
   Generate `openclaw.json`, set `OPENCLAW_STATE_DIR`, start gateway, and wait for readiness through logs or HTTP.

4. **Start surface**
   Start Clickclack web or Telegram Desktop and verify the real window/page is visible.

5. **Drive**
   Send a message as a real user action through the surface driver.

6. **Observe**
   Wait for a correlated visible event, not merely any assistant text.

7. **Record**
   Start screen capture before the drive or at a known marker, record through final reply, export full and cropped assets.

8. **Validate**
   Combine surface assertion, gateway log evidence, screenshot nonblank checks, and duplicate/stale guards.

9. **Manifest**
   Emit `proof.json` with artifact hashes, scenario, runtime, result, and `notTested` notes.

10. **Teardown**
    Stop gateway, browser, display, mocks, and helper processes. Leave artifacts intact.

## local-mantis edition matrix

### `telegram-desktop`

Purpose:

- Preserve existing Mantis-equivalent Telegram proof.
- Useful when the behavior specifically involves Telegram transport or historical Mantis compatibility.

Runtime:

- Desktop box.
- Telegram Desktop.
- tdlib user driver.
- Test bot token.
- Optional live Claude CLI creds or deterministic mock.

Crabbox fit:

- `local-container` with desktop.
- Prebuilt image/checkpoint strongly recommended.

### `clickclack-web`

Purpose:

- Primary PsiClawOps proof edition.
- Captures high-quality examples in our own surface.

Runtime:

- Desktop browser.
- Clickclack web app.
- OpenClaw gateway.
- Playwright or browser automation for input and validation.
- ffmpeg for visual proof.

Crabbox fit:

- `local-container` with desktop and browser.
- Could run with no Docker socket.
- Deterministic model server by default.

### `clickclack-mobile`

Purpose:

- Product proof for phone-sized layout and message behavior.

Runtime:

- Chromium mobile viewport, or a real browser/device node later.
- Same OpenClaw gateway.
- Mobile crop presets.

Crabbox fit:

- Same as `clickclack-web`, with viewport and capture presets.

### `clickclack-regression`

Purpose:

- Given a bug or PR, run a named scenario and prove the fix visually.

Runtime:

- Same as web or mobile edition.
- Scenario includes expected and forbidden events.

Crabbox fit:

- Same job pattern, with `--preset-var` or scenario path.

## Crabbox changes worth making later

### 1. First-class proof artifact support

Crabbox already has downloads and artifacts, but proof runs would benefit from a standard convention:

```sh
crabbox run --emit-proof /tmp/proof.md --proof-manifest /tmp/cap/proof.json -- command
```

or job config:

```yaml
proof:
  manifest: /tmp/cap/proof.json
  output: .artifacts/proof
  markdown: .artifacts/proof/proof.md
```

Core can then:

- Validate manifest schema.
- Download listed artifacts.
- Render a Markdown block.
- Surface proof in `crabbox results`.

### 2. `crabbox proof run`

A higher-level command could wrap jobs:

```sh
crabbox proof run clickclack-web streaming-basic
```

This should stay a thin command that expands to a job. It should not embed Clickclack or Telegram logic in Crabbox core.

### 3. External provider for private proof services

If proof capture later moves to a long-running internal service, use `provider: external`:

```yaml
provider: external
target: linux
external:
  command: node
  args:
    - /opt/psiclawops/proof-provider.mjs
  config:
    edition: clickclack-web
    image: psiclawops/proofrig-clickclack:stable
  workRoot: /work/crabbox
```

This keeps proprietary lifecycle logic out of Crabbox while letting Crabbox keep SSH sync and run behavior.

### 4. Dedicated delegated provider, only if needed

A `local-mantis` delegated provider could make sense if editions become remote proof runners that do not expose SSH. It would implement:

- `Warmup`
- `Run`
- `List`
- `Status`
- `Stop`
- artifact collection

Do this only after the proof service owns sync/execution itself. For local Docker proof, `local-container` is better.

## Security and isolation

### Docker socket

Do not enable `localContainer.dockerSocket` by default. It gives the proof lease access to the host container engine. Use it only when the proof itself must launch sibling containers.

If enabled:

- Use a dedicated host or runner.
- Keep secrets out of mounted paths.
- Treat the container as host-adjacent, not strongly isolated.
- Include `dockerSocket: true` in the proof manifest runtime block.

### Secrets

Rules:

- Do not pass tokens in argv.
- Use env profiles and Crabbox `allowEnv`/`envFromProfile` patterns.
- Redact gateway logs before packaging.
- Never include Telegram `tdata`, tdlib state, OAuth credentials, or bot tokens in downloads.
- Keep proof payload archives outside the repo.

### Artifact redaction

Before downloading logs:

- Replace tokens matching bot-token and bearer-token patterns.
- Remove full request headers.
- Remove raw payload archive fields.
- Keep enough correlation data to prove behavior.

## Implementation plan

### Phase 1: local-mantis job contract

- Add a `proof.json` manifest schema to local-mantis.
- Add common artifact filenames.
- Add example Crabbox jobs for existing Telegram editions.
- Verify downloads work for PNG, GIF, MP4, logs, and manifest.

Acceptance:

- `crabbox job run proof-telegram-echo` produces `.artifacts/proof/...` without manual export.

### Phase 2: proof image

- Create `docker/proofrig-base.Dockerfile`.
- Add desktop/browser dependencies.
- Add Node, pnpm, Python, ffmpeg, Xvfb, fonts, and Playwright dependencies.
- Add image build docs.
- Optionally checkpoint with local-container native Docker checkpoints.

Acceptance:

- First proof run does not spend most of its time installing OS dependencies.

### Phase 3: Clickclack web edition

- Add `proofrig/editions/clickclack-web` in local-mantis.
- Create a deterministic streaming model server.
- Launch OpenClaw gateway.
- Launch Clickclack web app or connect to configured URL.
- Drive a real UI send with Playwright.
- Capture and validate visual output.
- Emit proof manifest.

Acceptance:

- `crabbox job run proof-clickclack-basic` emits a clean artifact set.

### Phase 4: Crabbox proof convenience

- Add docs for proof jobs in Crabbox.
- Consider `proof:` job metadata if downloads plus manifest parsing become repetitive.
- Add tests around manifest parsing if implemented.

Acceptance:

- A consuming repo can copy one job template and get proof artifacts with minimal custom shell.

### Phase 5: First-class command or provider

Only after the edition API is stable:

- Add `crabbox proof run`, or
- Add an external-provider-backed proof service, or
- Add a dedicated delegated provider if proof execution no longer fits SSH lease model.

Acceptance:

- The new interface reduces config, does not hide artifacts, and does not fork core lifecycle behavior.

## Concrete Crabbox job examples

### Clickclack basic proof

```yaml
jobs:
  proof-clickclack-basic:
    provider: local-container
    target: linux
    desktop: true
    browser: true
    ttl: 90m
    idleTimeout: 30m
    shell: true
    command: >
      proofrig/bin/proofrig run
      --edition clickclack-web
      --scenario proofrig/editions/clickclack-web/scenarios/streaming-basic.json
      --out /tmp/cap
    downloads:
      - /tmp/cap/proof.json=.artifacts/proof/clickclack-basic/proof.json
      - /tmp/cap/proof-crop.png=.artifacts/proof/clickclack-basic/proof-crop.png
      - /tmp/cap/proof-review.gif=.artifacts/proof/clickclack-basic/proof-review.gif
      - /tmp/cap/proof-motion.mp4=.artifacts/proof/clickclack-basic/proof-motion.mp4
    stop: success
```

### Clickclack progress proof with a live model

```yaml
jobs:
  proof-clickclack-progress-live:
    provider: local-container
    target: linux
    desktop: true
    browser: true
    ttl: 2h
    idleTimeout: 45m
    shell: true
    envFromProfile:
      - /absolute/path/to/proof.env
    allowEnv:
      - ANTHROPIC_API_KEY
      - OPENCLAW_GATEWAY_TOKEN
    command: >
      proofrig/bin/proofrig run
      --edition clickclack-web
      --scenario proofrig/editions/clickclack-web/scenarios/progress-live.json
      --model live
      --out /tmp/cap
    downloads:
      - /tmp/cap/proof.json=.artifacts/proof/clickclack-progress-live/proof.json
      - /tmp/cap/proof-review.gif=.artifacts/proof/clickclack-progress-live/proof-review.gif
      - /tmp/cap/proof-motion.mp4=.artifacts/proof/clickclack-progress-live/proof-motion.mp4
    stop: success
```

### Telegram compatibility proof

```yaml
jobs:
  proof-telegram-echo:
    provider: local-container
    target: linux
    desktop: true
    browser: false
    ttl: 90m
    idleTimeout: 30m
    shell: true
    envFromProfile:
      - /absolute/path/to/crabbox-test.env
    allowEnv:
      - TELEGRAM_BOT_TOKEN
    command: >
      proofrig/bin/proofrig run
      --edition telegram-desktop
      --scenario proofrig/editions/telegram-desktop/scenarios/echo.json
      --out /tmp/cap
    downloads:
      - /tmp/cap/proof.json=.artifacts/proof/telegram-echo/proof.json
      - /tmp/cap/proof-crop.png=.artifacts/proof/telegram-echo/proof-crop.png
      - /tmp/cap/proof-review.gif=.artifacts/proof/telegram-echo/proof-review.gif
    stop: success
```

## Acceptance criteria for the whole adaptation

A successful Crabbox plus local-mantis proof system should meet these criteria:

- One command runs a proof edition in a disposable Docker-backed Crabbox lease.
- The command produces a normalized proof manifest and visual artifacts.
- The proof can target Clickclack without Telegram assets.
- The proof can still target Telegram when needed.
- Debug-first runs fail fast before video capture.
- Artifacts download through Crabbox, not manual container copying.
- Secrets are not included in logs, manifests, or artifacts.
- The proof text clearly states what was and was not tested.
- The runtime can use prebuilt Docker images or checkpoints for fast startup.
- The system does not require a Cloudflare broker for local proof runs.

## Recommendation

Keep Crabbox as the lifecycle and execution substrate. Keep local-mantis as the proof adapter layer. Use jobs and local-container for v1. Add Crabbox proof conveniences only after the proof manifest and edition API are stable.

The first target should be `clickclack-web-basic`. It gives PsiClawOps a more compelling proof artifact than Telegram screenshots, and it tests the surface we actually want to improve.
