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

## Inputs

| Name              | Default     | Description                                                                              |
| ----------------- | ----------- | ---------------------------------------------------------------------------------------- |
| `password`        | _none_      | Secret to evaluate. Either `password` or `guesses` must be set.                          |
| `guesses`         | _none_      | External guess count (e.g. `zxcvbn` output, `1e10`). Skips brtc's built-in entropy.      |
| `algorithm`       | `bcrypt`    | `md5`, `sha256`, `bcrypt`, or `argon2id`.                                                |
| `cost`            | `10`        | Work factor (bcrypt) or time iterations (argon2id).                                      |
| `memory`          | _none_      | Argon2id memory (e.g. `64m`, `128m`, `1g`).                                              |
| `hardware`        | `rtx-4090`  | Attacker hardware profile.                                                               |
| `all-hw`          | `false`     | Compare across every hardware profile. Ignores `fail-under-time`/`budget`, no `sarif`.  |
| `fail-under-time` | _none_      | Fail the step if estimated crack time is shorter (e.g. `1y`, `30d`, `12h`).              |
| `budget`          | _none_      | Attacker budget in USD (e.g. `1000usd`).                                                 |
| `output`          | `json`      | `json` or `sarif`. `json` populates the outputs; `sarif` populates `sarif-file`.        |
| `brtc-version`    | `v1.2.0`    | brtc release tag to install. Pinned for reproducibility; `latest`/`main` are not.       |
| `go-version`      | `1.25`      | Go toolchain version used to install brtc.                                               |

## Outputs

| Name                    | Description                                            |
| ----------------------- | ------------------------------------------------------ |
| `cost-usd`              | Estimated USD cost to crack.                           |
| `time-to-crack-seconds` | Estimated seconds to crack.                            |
| `entropy-bits`          | Estimated entropy in bits.                             |
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
- `brtc-version` defaults to a pinned release for reproducible runs. Set it to `latest` or `main` only if you accept non-reproducible installs.

## Versioning

- `@v1` — current major version, tracks the latest minor/patch on the v1 line.
- `@vX.Y.Z` — pin to a specific release.
- `@main` — bleeding edge, may break.

## License

MIT
