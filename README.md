# brtc-action

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
  run: |
    npx --yes zxcvbn-cli "${{ secrets.SERVICE_PASSWORD }}" --json \
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
| `fail-under-time` | _none_      | Fail the step if estimated crack time is shorter (e.g. `1y`, `30d`, `12h`).              |
| `budget`          | _none_      | Attacker budget in USD (e.g. `1000usd`).                                                 |
| `output`          | `json`      | `json` or `sarif`. JSON is required for the action outputs to be populated.              |
| `brtc-version`    | `latest`    | brtc release tag to install (e.g. `v0.2.0`, `latest`, `main`).                           |
| `go-version`      | `1.25`      | Go toolchain version used to install brtc.                                               |

## Outputs

| Name                    | Description                                            |
| ----------------------- | ------------------------------------------------------ |
| `cost-usd`              | Estimated USD cost to crack.                           |
| `time-to-crack-seconds` | Estimated seconds to crack.                            |
| `entropy-bits`          | Estimated entropy in bits.                             |
| `raw-json`              | Full brtc JSON output. Only set when `output: json`.   |

## SARIF output for Code Scanning

```yaml
- name: brtc → SARIF
  uses: kanywst/brtc-action@v1
  with:
    password: ${{ secrets.SERVICE_PASSWORD }}
    output: sarif
  id: brtc
- name: Upload SARIF
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: ${{ steps.brtc.outputs.raw-json }}
```

## Versioning

- `@v1` — current major version, tracks the latest minor/patch on the v1 line.
- `@vX.Y.Z` — pin to a specific release.
- `@main` — bleeding edge, may break.

## License

MIT
