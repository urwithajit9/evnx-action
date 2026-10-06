# evnx — .env Security & Validation Action

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-evnx--action-blue?logo=github)](https://github.com/marketplace/actions/evnx-env-security-validation)
[![evnx](https://img.shields.io/badge/evnx-CLI-green)](https://evnx.dev)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

> Validate, scan, diff, and doctor your `.env` files in CI — catching secrets, placeholder values, misconfigs, and drift before they ship.

Built on [evnx](https://evnx.dev) — the Rust CLI for `.env` lifecycle management.

---

## Quick start

```yaml
# .github/workflows/env-check.yml
name: .env security check

on: [pull_request, push]

jobs:
  env-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Scan for secrets
        uses: urwithajit9/evnx-action@v1
        with:
          command: scan
          format: github        # inline PR annotations
```

---

## Inputs

| Input | Description | Default |
|---|---|---|
| `command` | evnx command: `scan` \| `validate` \| `diff` \| `doctor` \| `convert` \| `sync` | `scan` |
| `path` | Directory or file to operate on | `.` |
| `format` | Output format: `github` \| `json` \| `sarif` \| `pretty` \| `github-actions` | `github` |
| `strict` | Exit 1 on warnings, not just errors | `false` |
| `exit_zero` | Report findings but always exit 0 | `false` |
| `env_file` | Path to the `.env` file | `.env` |
| `example_file` | Path to the `.env.example` file | `.env.example` |
| `convert_to` | Target format for the `convert` command | `json` |
| `convert_output` | Output file path for `convert` | `""` |
| `version` | evnx **CLI** version, e.g. `v0.9.0` or `0.9.0`. See *Versioning* below | `latest` |


## Versioning — two numbers, and they are not the same

| | |
|---|---|
| `uses: urwithajit9/evnx-action@v1` | the **action's** version |
| `with: version: v0.9.0` | the **evnx CLI** version it installs |

⚠️ **Pinning one does not pin the other.** `@v1` tracks the latest v1.x of this
action, which installs the latest CLI unless you say otherwise. To pin the tool
your CI actually runs, set `version:`.

```yaml
- uses: urwithajit9/evnx-action@v1     # action: latest v1.x  ← recommended
  with:
    command: scan
    version: v0.9.0                    # CLI: pinned
```

Other refs for the action itself:

```yaml
uses: urwithajit9/evnx-action@v1.1.0   # an exact action release
uses: urwithajit9/evnx-action@main     # unreleased — may break
```

ⓘ This repository previously carried tags mirroring CLI versions
(`v0.3.8` … `v0.9.0`) alongside action tags (`v1`, `v1.0.1`). Two numbering
schemes in one namespace meant `@v0.9.0` looked like a CLI pin and was not.
**The action now publishes `v1` / `v1.x` only.**

ⓘ `v0.5.0` and `v0.5.1` of the CLI have no release assets — failed release runs.
Pinning either fails the install with a 404, by design rather than silently.

### Integrity

Every install verifies the release's published `.sha256` before the binary runs.
A mismatch aborts the step.

## Outputs

| Output | Description |
|---|---|
| `findings` | Number of findings detected |
| `result` | Raw text output from the evnx command |
| `exit_code` | Exit code returned by evnx |

---

## Examples

### Secret scanning with SARIF (Security tab)

```yaml
- name: Scan for secrets → Security tab
  uses: urwithajit9/evnx-action@v1
  with:
    command: scan
    format: sarif

- name: Upload SARIF
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: evnx-results.sarif
  if: always()
```

### Validate — strict mode (block on warnings)

```yaml
- name: Validate .env
  uses: urwithajit9/evnx-action@v1
  with:
    command: validate
    strict: "true"
    format: github-actions     # inline annotations on the PR
```

### Diff — catch .env/.env.example drift

```yaml
- name: Check .env drift
  uses: urwithajit9/evnx-action@v1
  with:
    command: diff
    format: github
```

### Doctor — full health check

```yaml
- name: .env health check
  uses: urwithajit9/evnx-action@v1
  with:
    command: doctor
```

### Convert to Kubernetes secret

```yaml
- name: Convert to k8s Secret
  uses: urwithajit9/evnx-action@v1
  with:
    command: convert
    convert_to: kubernetes
    convert_output: k8s-secret.yaml

- name: Apply secret
  run: kubectl apply -f k8s-secret.yaml
```

### Non-blocking scan (report but don't block PR)

```yaml
- name: Scan (advisory only)
  uses: urwithajit9/evnx-action@v1
  with:
    command: scan
    exit_zero: "true"
    format: github
```

### Use outputs in later steps

```yaml
- name: Scan
  id: scan
  uses: urwithajit9/evnx-action@v1
  with:
    command: scan
    format: json

- name: Comment findings count
  if: steps.scan.outputs.findings != '0'
  run: |
    echo "::warning::evnx found ${{ steps.scan.outputs.findings }} secret(s)"
```

### Full security pipeline

```yaml
name: Full .env security pipeline
on: [pull_request]

jobs:
  env-security:
    runs-on: ubuntu-latest
    permissions:
      security-events: write   # needed for SARIF upload
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Scan for secrets
        uses: urwithajit9/evnx-action@v1
        with:
          command: scan
          format: sarif

      - name: Upload SARIF to Security tab
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: evnx-scan.sarif
        if: always()

      - name: Validate configuration
        uses: urwithajit9/evnx-action@v1
        with:
          command: validate
          strict: "true"

      - name: Check drift
        uses: urwithajit9/evnx-action@v1
        with:
          command: diff
```

---

## What evnx detects

**Secrets** — AWS Access Keys, Stripe live/test keys, GitHub PATs, OpenAI keys, Anthropic keys, RSA/EC private keys, high-entropy strings.

**Misconfiguration** — placeholder values (`YOUR_KEY_HERE`, `CHANGE_ME`), boolean string traps (`DEBUG="False"` is truthy in Python), weak `SECRET_KEY`, `localhost` URLs in production configs, suspicious port patterns.

**Drift** — variables present in `.env` but missing from `.env.example`, and vice versa.

---

## Links

- [evnx.dev](https://evnx.dev) — documentation and guides
- [urwithajit9/evnx](https://github.com/urwithajit9/evnx) — Rust source
- [GitHub Actions integration guide](https://evnx.dev/guides/integrations/github-actions)

## License

MIT — see [LICENSE](LICENSE).