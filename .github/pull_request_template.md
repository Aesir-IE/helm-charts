## Summary

<!-- What changed and why? One or two sentences. -->

## Chart version

- [ ] Bumped `version` in `charts/<chart>/Chart.yaml` (required for any chart change)
- [ ] Updated `appVersion` if the default agent image tag changed
- [ ] README / `values.yaml` comments updated for new or changed values

## Test plan

- [ ] `helm lint charts/*`
- [ ] `ct lint --config ct.yaml`
- [ ] Customer-safe wording (no internal-only names or secrets)
- [ ] Integration verify when chart values, probes, or deployment behavior changes: `cd ../earth-102 && task up && task verify` ([reference harness](https://github.com/Aesir-IE/earth-102))
