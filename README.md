# Drengr — GitHub Action

Run mobile app tests in CI. Drengr drives Android emulators and iOS simulators with an AI agent — tap, type, swipe, and verify. It parses `drengr-tests.yml`, reports pass/fail as JSON or JUnit, annotates failures inline on the run, uploads the results and the evidence as a build artifact, and on a pull request posts one comment that says what failed and why.

## Requirements

- **A device.** With `device: auto` (the default) Drengr boots a headless emulator or simulator on the runner itself: Android needs KVM on the runner, iOS needs a macOS runner. With `device: existing` it attaches to one something else already booted.
- **An LLM API key.** Drengr uses it to decide what to do on screen. Pass it as `api-key`, or leave it in the job environment as `DRENGR_API_KEY`, `GEMINI_API_KEY`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, etc. Without one the run exits `2`.
- **For the PR comment,** the job needs `permissions: pull-requests: write`. Without it the run still passes or fails correctly; the verdict is on the run's summary tab and in the artifact, and the comment step prints a warning.

## Usage

### iOS simulator

```yaml
name: Mobile Tests
on: [pull_request]

permissions:
  contents: read
  pull-requests: write

jobs:
  test:
    runs-on: macos-15
    steps:
      - uses: actions/checkout@v7
      - uses: SharminSirajudeen/drengr-ci@v1
        with:
          format: junit
          app: build/MyApp.app
          api-key: ${{ secrets.DRENGR_API_KEY }}
```

### Android emulator, booted by the action

```yaml
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

      - uses: SharminSirajudeen/drengr-ci@v1
        with:
          app: app/build/outputs/apk/debug/app-debug.apk
          api-key: ${{ secrets.DRENGR_API_KEY }}
```

### Android emulator from `reactivecircus/android-emulator-runner`

That action shuts its emulator down the moment its own `script:` step ends, so a separate `uses: drengr-ci@v1` step placed after it never sees a live device. Run Drengr **inside** that `script:` instead — it is the same npm package this action wraps. This is the pattern our own smoke workflow runs on a real emulator.

```yaml
      - uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 34
          target: google_apis
          arch: x86_64
          emulator-options: -no-window -no-audio -no-boot-anim -gpu swiftshader_indirect
          disable-animations: true
          script: |
            npm install -g drengr@latest
            drengr test --format junit --output drengr-results.xml --evidence-dir drengr-evidence
        env:
          DRENGR_API_KEY: ${{ secrets.DRENGR_API_KEY }}

      - uses: actions/upload-artifact@v7
        if: always()
        with:
          name: drengr-results
          path: |
            drengr-results.xml
            drengr-evidence
```

In this pattern the evidence and the summary tab still arrive, because the binary writes them. The PR comment does not, because this action's steps are not running; `drengr-evidence/summary.md` in the artifact is the same page.

## What a failure leaves behind

Under `evidence-dir` (default `drengr-evidence`), one folder per task:

| File | Holds |
|------|-------|
| `step_NNN.png` | the screen after each executed action (and `_before` for a fresh observation) |
| `steps.json` | what each action changed: screen, navigation, crash, elements that appeared or vanished |
| `network.json` | every call the task made: URL without its query string, method, status, duration, source. Never headers, never bodies |
| `logs.txt`, `death.txt` | the device log tail and the process verdict, for failed tasks only |

`summary.md` covers the suite: a verdict table, then for each failed task why it failed, how it ended, its last steps, its failed calls and its log tail. It is appended to the run's summary tab and, on a pull request, posted as one comment that is updated in place on every push.

Every failed task is classified from what the run saw: `app` for a crash, an unmet `expect_network` rule or a 5xx; `infrastructure` when there was nothing to launch, observe or ask; `inconclusive` otherwise. The one observation that decided it is quoted. A 4xx is listed but never decides.

## Run only what the pull request touches

```yaml
app: com.example.app
tasks:
  - name: Login
    task: Log in with test credentials (user@example.com / password123)

  - name: Checkout
    task: Complete the checkout flow — add item, go to cart, tap checkout
    timeout: 120s
    paths: ["app/src/main/java/com/example/checkout/*", "app/src/main/java/com/example/payments/*"]
    expect_network:
      - url: "*/api/orders"
        method: POST
        status: 201
```

On a `pull_request` event the action lists the PR's files and passes them to `drengr test --changed-files`. A task with `paths:` runs only when one of its globs matches a changed file; `*` matches any run of characters, `/` included. A task without `paths:` always runs. Skipped tasks are reported as skipped, in JUnit as `<skipped/>`. Set `scope-to-changes: "false"` to run everything.

## Bring your own endpoint

Drengr never calls a model on your behalf and never proxies through us — you bring the key, and the call goes straight from the runner to whoever you point it at. `base-url` takes that further: any OpenAI-compatible endpoint works, so you are not waiting on Drengr to add a provider.

```yaml
- uses: SharminSirajudeen/drengr-ci@v1
  with:
    base-url: https://openrouter.ai/api/v1
    model: qwen/qwen3-vl-235b-a22b-instruct
    api-key: ${{ secrets.OPENROUTER_API_KEY }}
```

The same works for LiteLLM, vLLM, a self-hosted gateway, or anything else speaking that API. Every run prints the provider, the model, and where the choice came from before it starts, so a failure never leaves you guessing which model was actually driving the device.

Pick a model that accepts **image input**. Drengr reads the screen as text first, which is cheap and usually enough, but it escalates to vision when a screen has unlabeled elements — common in Flutter, games, and custom canvas UIs. A text-only model works until it hits one of those.

## Inputs

| Name | Default | Description |
|------|---------|-------------|
| `api-key` | *(env)* | LLM API key. Falls back to whichever provider key is already in the job environment. |
| `provider` | auto | `gemini`, `openai`, `anthropic`, `groq`, `together`, `fireworks`, `ollama` |
| `model` | provider default | Any model the provider serves. The run prints which model it used and why. |
| `base-url` | provider's own | Any OpenAI-compatible endpoint — see "Bring your own endpoint". |
| `device` | `auto` | `auto` boots a headless emulator or simulator on the runner; `existing` attaches to one already booted. |
| `app` | — | Build to install before the suite runs: the `.apk` or `.app` this PR produced. |
| `file` | auto-detect | Path to the suite. Auto-detects `drengr-tests.yml`, `drengr-tests.yaml`, `.drengr/tests.yml`. |
| `format` | `json` | `json`, `junit`, or `human` |
| `output` | by format | Results file (`drengr-results.json` / `.xml` / `.txt`) |
| `evidence-dir` | `drengr-evidence` | Folder for per-task evidence and `summary.md`; uploaded with the results. |
| `scope-to-changes` | `true` | On a pull request, run only tasks whose `paths:` match a changed file. |
| `comment` | `true` | On a pull request, post `summary.md` as one comment, updated in place. |
| `github-token` | `github.token` | Token used to list the PR's files and post the comment. |
| `version` | `latest` | Version of the `drengr` npm package to install. Evidence, classification, `paths:` and the comment need the first release after 0.11.0. |
| `artifact-name` | `drengr-results` | Artifact name for the results and evidence. |
| `artifact-retention` | `7` | Days to retain the artifact. |

## Outputs

| Name | Description |
|------|-------------|
| `passed` | Number of passed tests |
| `failed` | Number of failed tests |
| `total` | Number of tests that ran. Skipped tasks are listed, not counted. |
| `results-file` | Path to the results file |
| `evidence-dir` | Folder holding `summary.md` and one evidence folder per task |
| `artifact-url` | URL of the uploaded results and evidence artifact |

Counts are written by the binary itself, so they are exact in every format — including on a failing run.

## Exit codes

| Code | Meaning |
|------|---------|
| `0` | Every task that ran passed |
| `1` | At least one task failed |
| `2` | Could not start — no suite file, unparseable suite, no API key, unreadable `--changed-files`, or no device |

## Output streams

Results go to the `--output` file (or stdout when it is omitted); progress goes to stderr. `drengr test --format json` therefore pipes straight into `jq` with nothing to strip.

## Running it without this action

```bash
npm install -g drengr
drengr test --format junit --output drengr-results.xml --evidence-dir drengr-evidence
```

`drengr ci` is an alias for `drengr test`. Under Actions both emit `::group::` / `::error::` annotations, write `passed` / `failed` / `total` to `$GITHUB_OUTPUT`, and append `summary.md` to `$GITHUB_STEP_SUMMARY`.

## License

MIT
