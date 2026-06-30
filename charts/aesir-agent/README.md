# aesir-agent

The Aesir OTel fleet agent connects to the [Aesir](https://aesir.ie) platform,
polls for collector configuration, and manages your OpenTelemetry Collectors over
[OpAMP](https://opentelemetry.io/docs/specs/opamp/). In Kubernetes the agent runs
an OpAMP server that collectors dial into to receive their managed pipeline config.

## Installing

```bash
helm repo add aesir https://aesir-ie.github.io/helm-charts
helm repo update

helm install aesir-agent aesir/aesir-agent \
  --namespace aesir --create-namespace \
  --set aesir.token=svn_xxx \
  --set aesir.environmentId=env_xxx
```

`aesir.token` and `aesir.environmentId` are required.

## How collectors connect

Collectors connect to the agent's OpAMP WebSocket server. By default the chart
advertises the in-cluster Service DNS:

```
ws://<release>-aesir-agent.<namespace>.svc.cluster.local:4320/v1/opamp
```

To expose OpAMP outside the cluster, set `service.type` (e.g. `LoadBalancer` or
`NodePort`) and override `opamp.advertiseAddr` with the externally reachable URL.

## Values

| Key | Default | Description |
| --- | ------- | ----------- |
| `image.repository` | `ghcr.io/frcodinglimited/agent` | Agent container image. |
| `image.tag` | `""` | Image tag; defaults to the chart `appVersion`. |
| `image.pullPolicy` | `IfNotPresent` | Image pull policy. |
| `imagePullSecrets` | `[]` | Registry pull secrets. |
| `aesir.endpoint` | `https://api.aesir.ie` | Aesir platform API endpoint. |
| `aesir.token` | `""` | **Required.** API token (stored in a Secret). |
| `aesir.environmentId` | `""` | **Required.** Environment short ID. |
| `aesir.pollInterval` | `30s` | Config poll interval. |
| `aesir.instanceId` | `""` | Optional stable instance ID override. |
| `opamp.port` | `4320` | OpAMP WebSocket server port. |
| `opamp.advertiseAddr` | `""` | OpAMP URL injected into collector configs; defaults to in-cluster Service DNS. |
| `metrics.enabled` | `false` | Enable the OTLP metrics receiver (collector self-telemetry). |
| `metrics.port` | `4321` | Metrics receiver port. |
| `metrics.agentHost` | `""` | Host collectors use to reach the agent; defaults to in-cluster Service DNS. |
| `logs.enabled` | `false` | Enable the OTLP logs receiver. |
| `logs.port` | `4322` | Logs receiver port. |
| `capture.enabled` | `true` | Enable the capture receiver for pipeline-editor live preview. |
| `capture.port` | `4323` | Capture receiver port. |
| `service.type` | `ClusterIP` | Service type for the agent's ports. |
| `serviceAccount.create` | `true` | Create a ServiceAccount. |
| `serviceAccount.name` | `""` | Override the ServiceAccount name. |
| `rbac.create` | `true` | Create the ClusterRole/ClusterRoleBinding (pod-label selector matching). |
| `resources` | see values.yaml | Pod resource requests/limits. |
| `nodeSelector` | `{}` | Pod node selector. |
| `tolerations` | `[]` | Pod tolerations. |
| `affinity` | `{}` | Pod affinity rules. |
| `podAnnotations` | `{}` | Extra pod annotations. |
| `podLabels` | `{}` | Extra pod labels. |

## RBAC

When `rbac.create` is true the chart creates a ClusterRole granting `get`/`list`
on `pods`, which the OpAMP server uses to match collector pods against pipeline
selector labels.
