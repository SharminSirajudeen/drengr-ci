# Containerised drengr-ci — design from scratch

Written 2026-09-09. Status: design; nothing below is built yet except where marked MEASURED.
Governing rule, from the owner: no wrapper around a bug. If the binary is wrong in a container,
fix the binary. The container stays dumb.

## The two facts that constrain everything

1. **An iOS simulator cannot run in a Linux container.** It needs macOS + Xcode + CoreSimulator.
   Docker on a macOS runner cannot reach the host's simulators either. The iOS lane is therefore a
   native macOS job, not an image. Anyone who says otherwise is selling something.
2. **An Android emulator in a container needs hardware virtualisation** (`/dev/kvm`) or it runs
   under software emulation at 10–50× slower and boots in minutes. GitHub-hosted `ubuntu-latest`
   (x86_64) exposes KVM; the existing drengr-ci README already does the `udevadm` dance. On Apple
   Silicon, colima `--vm-type vz --nested` (M3+, macOS 15+) provides it; treat local KVM as
   best-effort and GitHub as the reference bench.

So: **one action, two lanes.** Not one container.

## What drengr itself must get right first (binary fixes, not Dockerfile tricks)

### F1 — the orphan watchdog                                   MEASURED
`src/mcp/mod.rs` spawns a 3-second loop that calls `std::process::exit(0)` when the parent
looks dead. Published 0.10.10 contains 2399eea (skip the watchdog only when *we* are PID 1) but
NOT cde2d90 (compare the current ppid to the one we booted with). Under any init — docker's
PID 1, tini, the npm JS wrapper, a CI step shell — the server's ppid is 1 and stable, and
0.10.10 kills itself at 3.0s. strace, 2026-09-09 00:38Z: stdin reader ready at 173ms;
`getppid()=1` → `client process gone (ppid=1)` → `exit_group(0)` at 3.000s.
cde2d90 (16:18 IST, three minutes after the published commit) is the correct fix and is
unreleased. **Ship it.** Every topology workaround (tini, double shell, native-as-PID-1) is
deleted by this design.
Also: the watchdog is an MCP-client-liveness heuristic. It has no business running under
`drengr test`. Gate it on stdio-MCP mode with a real parent; never in the CI path.

### F2 — the glibc floor                                        MEASURED
0.10.10's Linux binary requires GLIBC_2.38/2.39. It fails to load on Debian bookworm (2.36) and
would on `ubuntu-22.04` runners (2.35). 0.10.8 loaded fine on bookworm; the floor moved in the
fix release without anyone deciding it. Decide it: either build the Linux release in a cross
image pinned to glibc 2.31 (Ubuntu 20.04 base — covers every runner in use), or ship
`*-unknown-linux-musl` static. musl is cleaner iff TLS is rustls (not openssl) — check
`cargo tree -i openssl-sys`. Until decided, images below use a glibc ≥ 2.39 base.

### F3 — startup shell-outs (hygiene, not a bug)                MEASURED
At boot the binary spawns `sh -lc 'command -v adb …'`, `sh -lc 'command -v xcrun'` and a
`hostnamectl … machine-id` pipeline. The env vars `DRENGR_ADB_PATH` / `DRENGR_XCRUN_PATH` are
honoured *before* any shell is spawned — set them in the image and no login shell ever runs.

## The artifacts

### One binary, downloaded — no Node, no npm wrapper
Both images take the native binary from the GitHub Release tarball for `$TARGETARCH`. This
removes Node from the runtime, removes the JS-wrapper process layer, and makes the image a
faithful copy of what the release pipeline built.

### `drengr-mcp` — the Glama image (the trivial sub-case)
    FROM debian:trixie-slim
    ARG TARGETARCH VERSION
    ADD https://github.com/SharminSirajudeen/drengr-community/releases/download/v${VERSION}/drengr-v${VERSION}-<triple-for-arch>.tar.gz /tmp/
    RUN tar -xzf /tmp/*.tar.gz -C /usr/local/bin && chmod +x /usr/local/bin/drengr
    ENTRYPOINT ["drengr","mcp"]
Passes Glama's check (start + answer initialize/tools-list, no device, no network) the moment a
release containing cde2d90 exists. Nothing else.

### `drengr-android` — the CI image
Skeleton to clone, not to reinvent: **google/android-emulator-container-scripts** — Google's own
headless-emulator-in-Docker with KVM, adb on 5555, system-image download. Add the drengr binary
and a test phase to its entrypoint.
Contents: OpenJDK 17, cmdline-tools, platform-tools, emulator, exactly one system image
(`system-images;android-34;google_apis;x86_64` on amd64, `…;arm64-v8a` on arm64), an AVD
created at *build* time so boot is fast, the drengr binary, `DRENGR_ADB_PATH` set.
Entrypoint (~40 lines, the only logic in the image):
  1. refuse to start without `/dev/kvm` unless `DRENGR_ALLOW_TCG=1` (say why, exit 2)
  2. `emulator -avd ci -no-window -no-audio -no-boot-anim -gpu swiftshader_indirect -no-snapshot-save &`
  3. `adb wait-for-device`; poll `getprop sys.boot_completed` (timeout 180s → exit 2 with the logcat tail)
  4. `adb install -r /app/*.apk`
  5. `exec drengr test /tests/drengr-tests.yml --format junit --output /out/results.xml`
  6. exit status = drengr's
Inputs, and nothing else: `DRENGR_API_KEY` (or GEMINI_/OPENAI_/ANTHROPIC_… as today),
`/app` (APK), `/tests` (yml), `/out` (results). Run: `--device /dev/kvm`.
Size ≈ 3.5 GB; publish to GHCR, tagged by drengr version and API level.

### iOS lane — a reusable workflow, not an image
`macos-latest`: `xcrun simctl boot "<device>"` → `xcrun simctl install booted <app>` →
`drengr test` → junit. This is what the README already describes; package it as
`uses: SharminSirajudeen/drengr-ci/.github/workflows/ios.yml@v1` with the same inputs.

### The action becomes two lanes
    - uses: SharminSirajudeen/drengr-ci@v1
      with: { platform: android, api-key: ${{ secrets.GEMINI_API_KEY }}, app: app/build/outputs/apk/debug/app-debug.apk }
`platform: android` → the container step with `--device /dev/kvm`. `platform: ios` → the macOS
lane. "The action does not provision a device" becomes "the action brings the device."

## Release gate (so a fix can never again be in HEAD and absent from the binary)
In release.yml, after each Linux tarball is built: build `drengr-mcp` from it, then
    printf '%s\n' '<initialize>' '<initialized>' '<tools/list>' | docker run -i --network none drengr-mcp
and assert the response lists `drengr_look drengr_do drengr_query`. One container, ~10s.
The TEST line in the session anchor is this gate.

## Local parity
    docker run --rm --device /dev/kvm -e DRENGR_API_KEY=… \
      -v $PWD/app.apk:/app/app.apk -v $PWD/drengr-tests.yml:/tests/drengr-tests.yml -v $PWD/out:/out \
      ghcr.io/sharminsirajudeen/drengr-android:0.10.11-api34
On a Mac without nested virt: run the emulator natively and point the container at it with
`ADB_SERVER_SOCKET=tcp:host.docker.internal:5037` — same image, no KVM needed.

## Order of work and honest sizes
1. Box hygiene (owner, sudo): stop colima, kill the headless emulator, restart fseventsd,
   exclude ~/.colima from Spotlight. Nothing below can be verified on a box at 30 GB swap.
2. Release 0.10.11 from HEAD with F1 (already there) and an explicit F2 decision. Hours,
   mostly CI wall time. Owner publishes.
3. `drengr-mcp` Dockerfile → glama.ai submission → badge on awesome-mcp-servers #13765.
   One hour after (2). The badge is the reply; nothing is posted before it exists.
4. `drengr-android` from Google's skeleton + the entrypoint + a smoke workflow in drengr-ci
   that runs it on `ubuntu-latest` with KVM. 1–2 days including iteration on the runner.
5. iOS reusable workflow. Half a day.

## Deleted by this design
Every Dockerfile variant produced on 2026-09-08/09 (fast, tini, t2, t5, strace, direct) and the
idea behind them. Keep only the evidence: the strace, the timeline, F1–F3.
