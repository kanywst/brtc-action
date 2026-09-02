# brtc-action

[![CI](https://github.com/kanywst/brtc-action/actions/workflows/test.yml/badge.svg)](https://github.com/kanywst/brtc-action/actions/workflows/test.yml)
[![Release](https://img.shields.io/github/v/release/kanywst/brtc-action?sort=semver)](https://github.com/kanywst/brtc-action/releases)
[![Marketplace](https://img.shields.io/badge/marketplace-brtc--action-blue?logo=github)](https://github.com/marketplace/actions/brtc-action)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

GitHub Action wrapper for [brtc](https://github.com/kanywst/brtc) — calculate the offline brute-force cost of a password or secret, and optionally fail the build when the estimated crack time falls below a threshold.

> **brtc is a cost calculator, not a strength meter.** For accurate strength evaluation, run [zxcvbn](https://github.com/dropbox/zxcvbn) first and feed its `guesses` value via the `guesses:` input.

## Quick start

Gate a deployment on a secret being expensive enough to crack:

```yaml
- name: Gate on password strength
  uses: kanywst/brtc-action@v1
  with:
    password: ${{ secrets.SERVICE_PASSWORD }}
    algorithm: bcrypt
    cost: "12"
    hardware: aws-p5.48xlarge
    fail-under-time: 1y
```

If `aws-p5.48xlarge` could crack the secret in less than a year, the step exits non-zero and the workflow fails.

## Pair with zxcvbn (recommended)

Use a real strength estimator for entropy, brtc for the cost translation:

```yaml
- name: Estimate guesses with zxcvbn
  id: zxcvbn
  env:
    # Pass the secret via env, not string interpolation into the script.
    PASSWORD: ${{ secrets.SERVICE_PASSWORD }}
  run: |
    npx --yes zxcvbn-cli "$PASSWORD" --json \
      | jq -r '.guesses' \
      | xargs -I{} echo "guesses={}" >> "$GITHUB_OUTPUT"

- name: Convert to USD cost
  uses: kanywst/brtc-action@v1
  with:
    guesses: ${{ steps.zxcvbn.outputs.guesses }}
    algorithm: bcrypt
    cost: "12"
    fail-under-time: 1y
```

## Gates

Three independent gates decide whether the step fails. Use one, or combine them — the step fails on the first one that trips.

```yaml
- uses: kanywst/brtc-action@v1
  with:
    password: ${{ secrets.SERVICE_PASSWORD }}
    algorithm: bcrypt
    cost: "12"
    fail-under-time: 1y        # too cheap to crack
    fail-under-entropy: 60     # too few bits
    fail-on-breach: true       # already in a public corpus
```

`fail-on-breach` looks the secret up in [Have I Been Pwned](https://haveibeenpwned.com/) using the k-anonymity range API: only the first 5 hex characters of the secret's SHA-1 hash leave the runner, and the suffix is matched locally. It needs network access, and a lookup that **cannot be completed** fails the step too — a gate that was never evaluated must not report a pass. Set it only when your runners can reach `api.pwnedpasswords.com`.

`fail-under-entropy` reads brtc's entropy estimate, which is a naive `R^L` upper bound unless you feed a real guess count through `guesses`. Pair the two for a threshold that reflects actual guessability.

Both gates need brtc **v1.4.0 or later**. With an older `brtc-version` the action stops with a clear error instead of passing the unknown flag through.

## Inputs

| Name              | Default     | Description                                                                              |
| ----------------- | ----------- | ---------------------------------------------------------------------------------------- |
| `password`        | _none_      | Secret to evaluate. Either `password` or `guesses` must be set.                          |
| `guesses`         | _none_      | External guess count (e.g. `zxcvbn` output, `1e10`). Skips brtc's built-in entropy.      |
| `algorithm`       | `bcrypt`    | `md5`, `sha256`, `bcrypt`, or `argon2id`.                                                |
| `cost`            | `10`        | Work factor (bcrypt) or time iterations (argon2id).                                      |
| `memory`          | _none_      | Argon2id memory (e.g. `64m`, `128m`, `1g`).                                              |
| `hardware`        | `rtx-4090`  | Attacker hardware profile — `rtx-5090`, `rtx-4090`, `rx-7900xtx`, `rtx-3060`, `gtx-1080ti`, `mac-m3-max`, `mac-m3`, `cpu-standard`, `aws-p5.48xlarge`, `raspberry-pi-4`. An unrecognized name fails the step. |
| `all-hw`          | `false`     | Compare across every hardware profile. Ignores the `fail-*` gates and `budget`, no `sarif`. |
| `fail-under-time` | _none_      | Fail the step if estimated crack time is shorter (e.g. `1y`, `30d`, `12h`).              |
| `fail-under-entropy` | _none_   | Fail the step if estimated entropy is below this many bits (e.g. `60`, `80`).            |
| `fail-on-breach`  | `false`     | Fail the step if the secret is in Have I Been Pwned. Needs runner network access.        |
| `budget`          | _none_      | Attacker budget in USD (e.g. `1000usd`).                                                 |
| `output`          | `json`      | `json` or `sarif`. `json` populates the outputs; `sarif` populates `sarif-file`.        |
| `brtc-version`    | `v1.4.0`    | brtc release tag to install. Pinned for reproducibility; `latest`/`main` are not.       |
| `go-version`      | `1.25`      | Go toolchain version used to install brtc.                                               |

## Outputs

| Name                    | Description                                            |
| ----------------------- | ------------------------------------------------------ |
| `cost-usd`              | Estimated USD cost to crack.                           |
| `time-to-crack-seconds` | Estimated seconds to crack.                            |
| `entropy-bits`          | Estimated entropy in bits.                             |
| `breach-count`          | Times the secret appears in HIBP. `0` unless `fail-on-breach` is set. |
| `raw-json`              | Full brtc JSON output. Only set when `output: json`.   |
| `sarif-file`            | Path to the written SARIF file. Only set when `output: sarif`. |

## SARIF output for Code Scanning

`output: sarif` writes a SARIF file into the workspace and exposes its path via the `sarif-file` output. `upload-sarif` needs a checked-out repo and the `security-events: write` permission:

```yaml
permissions:
  security-events: write

steps:
  - uses: actions/checkout@v7
  - name: brtc → SARIF
    id: brtc
    uses: kanywst/brtc-action@v1
    with:
      password: ${{ secrets.SERVICE_PASSWORD }}
      output: sarif
  - name: Upload SARIF
    uses: github/codeql-action/upload-sarif@v3
    with:
      sarif_file: ${{ steps.brtc.outputs.sarif-file }}
```

## Notes

- The `password` is piped to brtc over stdin so it never appears in the runner's process list. brtc trims surrounding whitespace from a stdin password, so leading/trailing spaces are not counted.
- `fail-on-breach` is the only input that sends anything off the runner, and it sends 5 hex characters of a SHA-1 hash — never the secret. Every other gate is computed locally.
- `brtc-version` defaults to a pinned release for reproducible runs. Set it to `latest` or `main` only if you accept non-reproducible installs.

## Versioning

- `@v1` — current major version, tracks the latest minor/patch on the v1 line.
- `@vX.Y.Z` — pin to a specific release.
- `@main` — bleeding edge, may break.

The estimates come from whichever brtc the `brtc-version` input installs, so **a new action release can change your numbers** when you track `@v1` without pinning it. The default moved `v1.2.0` → `v1.4.0` in the release that added the entropy and breach gates, which also picked up brtc v1.3.0's 2026 hardware baselines — those shift crack times, USD costs, and therefore where `fail-under-time` draws the line. Pin `brtc-version` if a run has to keep producing the same numbers over time.

The default now moves `v1.4.0` → `v2.0.1`. That release makes an unrecognized `hardware` value fail instead of quietly falling back to `rtx-4090`. If a workflow has been passing a misspelled profile, it has been reporting numbers for the wrong hardware — and, with `fail-under-time` set, possibly passing a gate it should have failed. It now stops with brtc's error listing the valid names. The estimates themselves are unchanged.

## License

MIT
