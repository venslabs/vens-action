# Test fixtures

- `config.yaml` — minimal project context (`exposure`, `data_sensitivity`,
  `business_criticality`) required by `vens generate`.

Scan reports are generated on the fly in CI by running real Trivy and Grype
against a pinned image — see `.github/workflows/test-action.yml`. The matrix
covers the two latest releases of each scanner so we catch JSON schema drift
as soon as it ships.

The `gate` job also checks `fail-on-severity` on fixed VEX files: `vex/*.cdx.json` (fictitious `CVE-2099-*` ids), written
by `fake-vens/vens`, a test double passed as `bin-path` whose `generate` copies `$FAKE_VEX` to the output path.
