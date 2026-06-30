# Aesir Helm Charts

Official Helm charts for the [Aesir](https://aesir.ie) platform and products.

This repository is published as a Helm chart repository via GitHub Pages and is
the front-facing home for installing Aesir components into your own Kubernetes
clusters.

## Usage

Add the repository:

```bash
helm repo add aesir https://aesir-ie.github.io/helm-charts
helm repo update
```

Search for available charts:

```bash
helm search repo aesir
```

## Available charts

| Chart | Description |
| ----- | ----------- |
| [aesir-agent](charts/aesir-agent) | Aesir OTel fleet agent — connects to the Aesir platform and manages OpenTelemetry Collectors over OpAMP. |

More charts for other Aesir products will be added here over time.

## Installing a chart

```bash
helm install my-release aesir/aesir-agent \
  --namespace aesir --create-namespace \
  --set aesir.token=svn_xxx \
  --set aesir.environmentId=env_xxx
```

See each chart's README for the full list of configurable values.

## Contributing

Charts live under [`charts/`](charts). Pull requests are linted and test-installed
with [chart-testing](https://github.com/helm/chart-testing). When a change to a
chart (including a bumped `version` in its `Chart.yaml`) lands on `main`, the
release workflow packages it and publishes it to the Pages-hosted repository.

## License

See [LICENSE](LICENSE).
