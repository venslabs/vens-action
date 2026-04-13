# vens-action

GitHub Action for [vens](https://github.com/venslabs/vens) — prioritize vulnerabilities by real risk, not just CVSS.

Drop it into your pipeline after a Trivy or Grype scan. It re-scores every CVE in your project's context (exposure, data sensitivity, controls) and produces a CycloneDX VEX file plus severity counts you can use to fail the build.

> **v0.1.0 — first release.** Intentionally minimal.

## Quick start

```yaml
- name: Scan image with Trivy
  run: trivy image python:3.11-slim --format json --output report.json

- name: Prioritize with vens
  id: vens
  uses: venslabs/vens-action@v0.1.0
  with:
    version: v0.3.1
    config-file: .vens/config.yaml
    input-report: report.json
    sbom-serial-number: ${{ vars.SBOM_SERIAL }}
    llm-provider: openai
    llm-model: gpt-4o
    llm-api-key: ${{ secrets.OPENAI_API_KEY }}
    fail-on-severity: critical
    enrich: "true"

- uses: actions/upload-artifact@v4
  with:
    name: vens
    path: |
      ${{ steps.vens.outputs.vex-file }}
      ${{ steps.vens.outputs.enriched-report }}
```

## `sbom-serial-number`

This is the **serial number of the SBOM that your scanner report was produced from** — the `urn:uuid:<uuid>` stored in the SBOM's `serialNumber` field. vens uses it as the BOM-Link anchor so the VEX output references the correct SBOM.

## Inputs

| Name | Required | Default | Description |
|---|---|---|---|
| `version` | yes\* | — | vens release tag (e.g. `v0.3.1`). Ignored when `bin-path` is set. |
| `bin-path` | yes\* | — | Path to a pre-installed vens binary. For air-gapped runners and custom builds. |
| `config-file` | yes | — | Path to `config.yaml` describing project context. See [vens config reference](https://github.com/venslabs/vens#configuration). |
| `input-report` | yes | — | Trivy or Grype JSON scan report. |
| `input-format` | no | `auto` | `auto` \| `trivy` \| `grype`. |
| `output-vex` | no | `$RUNNER_TEMP/vens-vex.cdx.json` | Output path for the CycloneDX VEX. |
| `sbom-serial-number` | yes | — | `urn:uuid:<uuid>` — see section above. |
| `sbom-version` | no | `1` | BOM-Link version integer. |
| `llm-provider` | no | `openai` | `openai` \| `anthropic` \| `googleai` \| `ollama` \| `mock`. |
| `llm-model` | no | *(provider default)* | Mapped to `<PROVIDER>_MODEL`. |
| `llm-api-key` | no | — | Masked in logs. Mapped to `<PROVIDER>_API_KEY`. |
| `llm-base-url` | no | — | Custom base URL (OpenAI-compatible endpoint or Ollama host). |
| `llm-batch-size` | no | `10` | CVEs per LLM request. |
| `enrich` | no | `false` | Also run `vens enrich` to emit an enriched Trivy report. |
| `output-enriched-report` | no | `$RUNNER_TEMP/vens-enriched.json` | Output path for the enriched report. |
| `fail-on-severity` | no | — | Fail the step when a CVE is rated at or above this OWASP level: `critical` \| `high` \| `medium` \| `low` \| `info`. |

\* One of `version` or `bin-path` must be set.

## Outputs

| Name | Description |
|---|---|
| `vex-file` | Path to the CycloneDX VEX. |
| `enriched-report` | Path to the enriched Trivy report (empty when `enrich=false`). |
| `sbom-serial-number` | Echoes back the serial passed in. |
| `version` | vens version installed (empty when `bin-path` is used). |
| `count-critical` / `-high` / `-medium` / `-low` / `-info` | CVE counts per OWASP severity bucket. |

## Self-hosted and air-gapped runners

Pre-install the vens binary and pass `bin-path` instead of `version`:

```yaml
- uses: venslabs/vens-action@v0.1.0
  with:
    bin-path: /opt/bin/vens
    config-file: .vens/config.yaml
    input-report: report.json
    sbom-serial-number: ${{ vars.SBOM_SERIAL }}
    llm-provider: ollama
    llm-base-url: http://ollama.corp.example:11434
    llm-model: llama3.1
```

## Notes

- Linux (x64, arm64) and macOS (x64, arm64). Windows fails fast. `jq` must be on the runner (preinstalled on GitHub-hosted).
- For every CVE, the action POSTs the CVE details + your `config-file` fields to the chosen LLM provider. Use `ollama` for self-hosted models, or `mock` in CI where you don't want to spend tokens.
- Pin the action by commit SHA for reproducibility: `uses: venslabs/vens-action@<40-char-sha>`. Dependabot and Renovate both track SHA-pinned actions.

## License

Apache-2.0 — see [LICENSE](LICENSE).
