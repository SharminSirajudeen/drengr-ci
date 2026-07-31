# Drengr — GitHub Action

Run mobile app tests in CI. Drengr drives Android emulators and iOS simulators with an AI agent — tap, type, swipe, and verify. It parses `drengr-tests.yml`, reports pass/fail as JSON or JUnit, annotates failures inline on the run, and uploads the results file as a build artifact.

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

## Inputs

| Name | Default | Description |
|------|---------|-------------|
| `api-key` | *(env)* | LLM API key. Falls back to whichever provider key is already in the job environment. |
| `provider` | auto | `gemini`, `openai`, `anthropic`, `groq`, `together`, `fireworks`, `ollama` |
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
