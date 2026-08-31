# Drengr — GitHub Action

**Write mobile tests in plain English. No selectors, no XPath, no test IDs.**

Drengr drives real Android emulators and iOS simulators with an AI agent — it reads the screen, taps, types, swipes and verifies. It works on Flutter, games and custom canvas UIs, where selector-based tools have nothing to select. Your key, your model, your runner: Drengr never proxies a call on your behalf.

```yaml
- name: Login flow
  task: Log in with test credentials (user@example.com / password123)
```

![Drengr tapping a button drawn on an HTML canvas in an iOS simulator](docs/canvas-demo.gif)

*An HTML `<canvas>` has no accessibility tree, no DOM children and no selectors. Appium and Detox
see one opaque rectangle — there is nothing to query. Drengr finds the button and taps it. Watch
the hit counter.*

## How it compares

| | Appium | Maestro | Detox | **Drengr** |
|---|---|---|---|---|
| Scope | native + hybrid | native + hybrid | **React Native only** | native + hybrid |
| Tests written as | selectors / XPath | YAML flows with selectors | JS matchers, `by.id` etc | **plain English tasks** |
| Finds elements via | accessibility tree | system-exposed properties | app internals (grey box) | **screen contents, plus vision when unlabeled** |
| Works when elements are unlabeled | ✗ | ✗ | ✗ | **✓** |
| Blind coordinate tap available | ✓ | ✓ `tapOn: point: "84%,23%"` | — | ✓ |
| Deterministic | ✓ | ✓ | ✓ | **✗ — agent-driven** |
| Cost per run | free | free | free | LLM tokens |

**Sources.** Maestro's selector set and percentage-point taps:
[docs.maestro.dev](https://docs.maestro.dev). Detox describes itself as *"an open-source end-to-end
(E2E) testing framework for React Native mobile applications"* using *"gray box"* testing with
*"access to the internals of the app under test"*:
[wix.github.io/Detox](https://wix.github.io/Detox/docs/introduction/getting-started).

**The row that matters is row 4.** Every selector-based tool needs something to select. Maestro's
own Flutter guide says it plainly: *"Flutter web renders to `<canvas>` and does not enable the DOM
accessibility/semantics overlay by default. Maestro relies on this overlay to find elements."* The
fix they recommend is to add `semanticLabel` or wrap widgets in `Semantics` — that is, change your
app so the test tool can see it. Same story for a game, a charting canvas, or any custom-drawn
surface.

Drengr reads the screen instead, and escalates to vision when elements are unlabeled, so nothing
has to be added to the app.

**Two honest caveats.** All three tools can tap a raw coordinate you already know — Maestro's
`tapOn: point:` does exactly that. What no selector-based tool can do is work out *where* to tap
on a surface it cannot see. And Drengr is not deterministic: an agent decides from what it sees, so
the same suite can reach the same result by a different route, and it costs tokens. If your app is
a conventional native form with stable accessibility IDs, Maestro is simpler, deterministic and
free. Use it.

## Requirements

- **A booted device** on the runner — an Android emulator or an iOS simulator. The action does not provision one; see the examples below.
- **An LLM API key.** Drengr uses it to decide what to do on screen. Pass it as `api-key`, or leave it in the job environment as `DRENGR_API_KEY`, `GEMINI_API_KEY`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, etc. Without one the run exits `2`.

## Usage

### Android emulator

`reactivecircus/android-emulator-runner` shuts its emulator down the moment its own `script:`
step ends, so a separate `uses: drengr-ci@v1` step placed *after* it never sees a live device.
Run Drengr **inside** that `script:` instead — it's the same npm package this action wraps:

```yaml
name: Mobile Tests
on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - name: Enable KVM
        run: |
          echo 'KERNEL=="kvm", GROUP="kvm", MODE="0666", OPTIONS+="static_node=kvm"' \
            | sudo tee /etc/udev/rules.d/99-kvm4all.rules
          sudo udevadm control --reload-rules
          sudo udevadm trigger --name-match=kvm

      - uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 34
          target: google_apis
          arch: x86_64
          emulator-options: -no-window -no-audio -no-boot-anim -gpu swiftshader_indirect
          disable-animations: true
          script: |
            npm install -g drengr@latest
            drengr test --format junit --output drengr-results.xml
        env:
          DRENGR_API_KEY: ${{ secrets.DRENGR_API_KEY }}

      - uses: dorny/test-reporter@v3
        if: always()
        with:
          name: Drengr Results
          path: drengr-results.xml
          reporter: java-junit
```

Use the `drengr-ci@v1` action itself for Android when the device is already live before your job
starts — a persistent self-hosted runner, a device farm, or any setup that doesn't tear the
emulator down between steps.

### iOS simulator

```yaml
runs-on: macos-15
steps:
  - uses: actions/checkout@v7

  - name: Boot iOS simulator
    run: |
      xcrun simctl boot "iPhone 16"
      xcrun simctl bootstatus "iPhone 16"

  - uses: SharminSirajudeen/drengr-ci@v1
    with:
      format: junit
      api-key: ${{ secrets.DRENGR_API_KEY }}
```

## Bring your own endpoint

Drengr never calls a model on your behalf and never proxies through us — you bring the key, and the
call goes straight from the runner to whoever you point it at. `base-url` takes that further: any
OpenAI-compatible endpoint works, so you are not waiting on Drengr to add a provider.

```yaml
- uses: SharminSirajudeen/drengr-ci@v1
  with:
    base-url: https://openrouter.ai/api/v1
    model: qwen/qwen3-vl-235b-a22b-instruct
    api-key: ${{ secrets.OPENROUTER_API_KEY }}
```

The same works for LiteLLM, vLLM, a self-hosted gateway, or anything else speaking that API. Every
run prints the provider, the model, and where the choice came from before it starts, so a failure
never leaves you guessing which model was actually driving the device.

Pick a model that accepts **image input**. Drengr reads the screen as text first, which is cheap and
usually enough, but it escalates to vision when a screen has unlabeled elements — common in Flutter,
games, and custom canvas UIs. A text-only model works until it hits one of those.

## Inputs

| Name | Default | Description |
|------|---------|-------------|
| `api-key` | *(env)* | LLM API key. Falls back to whichever provider key is already in the job environment. |
| `provider` | auto | `gemini`, `openai`, `anthropic`, `groq`, `together`, `fireworks`, `ollama` |
| `model` | provider default | Any model the provider serves. The run prints which model it used and why. |
| `base-url` | provider's own | Any OpenAI-compatible endpoint — see "Bring your own endpoint" below. |
| `file` | auto-detect | Path to the suite. Auto-detects `drengr-tests.yml`, `drengr-tests.yaml`, `.drengr/tests.yml`. |
| `format` | `json` | `json`, `junit`, or `human` |
| `output` | by format | Results file (`drengr-results.json` / `.xml` / `.txt`) |
| `version` | `latest` | Version of the `drengr` npm package to install |
| `artifact-name` | `drengr-results` | Artifact name for the results file |
| `artifact-retention` | `7` | Days to retain the artifact |

## Outputs

| Name | Description |
|------|-------------|
| `passed` | Number of passed tests |
| `failed` | Number of failed tests |
| `total` | Total test count |
| `results-file` | Path to the results file |

Counts are written by the binary itself, so they are exact in every format — including on a failing run.

## Test file format

Create `drengr-tests.yml` in your repo root:

```yaml
app: com.example.app
tasks:
  - name: Login flow
    task: Log in with test credentials (user@example.com / password123)
    timeout: 60s

  - name: Checkout
    task: Complete the checkout flow — add item, go to cart, tap checkout
    timeout: 120s
```

Each task describes what to do in plain English. Drengr's agent drives the device to complete it and reports pass/fail.

## Exit codes

| Code | Meaning |
|------|---------|
| `0` | Every task passed |
| `1` | At least one task failed |
| `2` | Could not start — no suite file, unparseable suite, no API key, or no device |

## Output streams

Results go to the `--output` file (or stdout when it is omitted); progress goes to stderr. `drengr test --format json` therefore pipes straight into `jq` with nothing to strip.

## Running it without this action

```bash
npm install -g drengr
drengr test --format junit --output drengr-results.xml
```

`drengr ci` is an alias for `drengr test`. Under Actions both emit `::group::` / `::error::` annotations and write `passed` / `failed` / `total` to `$GITHUB_OUTPUT`.

## License

MIT
