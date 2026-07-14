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

## Collector deployment modes

The agent supports two OpAMP collector patterns:

| Pattern | Collector | `opamp.injectExtension` | Platform type |
| ------- | --------- | ----------------------- | ------------- |
| **Supervisor** (recommended for K8s) | `opampsupervisor` + `otelcol-contrib` | `false` (default) | `aesir` |
| **Vanilla** (direct otelcol) | `otelcol-contrib` with bootstrap `opamp` extension | `true` | `vanilla` |

When `opamp.injectExtension` is `true`, the agent injects the `opamp` extension into
delivered configs so direct collectors reconnect after applying remote config. Leave it
`false` when using the OpAMP supervisor — injecting the extension breaks supervisor
health confirmation.

For direct `otelcol-contrib` deployments that cannot accept OpAMP remote config
in-process, enable `opamp.k8sApply`. The agent writes merged config to the named
ConfigMap and restarts the collector Deployment. The chart sets `AESIR_AGENT_NAMESPACE`
to the Helm release namespace, so `configMap` and `deployment` must live in that
namespace (install the agent into the same namespace as the collector).

See the [earth-102 vanilla overlay](https://github.com/Aesir-IE/earth-102/blob/main/helm/aesir-agent.vanilla.values.yaml)
for a worked example:

```yaml
opamp:
  injectExtension: true
  k8sApply:
    enabled: true
    configMap: otel-collector-config
    configMapKey: config.yaml
    deployment: otel-collector
```

Collector type is registered automatically on first OpAMP connection. Override with the
OpAMP identifying attribute `aesir.collector.type`, or set `aesir.defaultCollectorType`.

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
| `aesir.defaultCollectorType` | `""` | Default collector type when OpAMP inference is unavailable. |
| `aesir.stateDir` | `/tmp/.aesir` | Writable state directory (mounted via emptyDir at `/tmp`). |
| `opamp.port` | `4320` | OpAMP WebSocket server port. |
| `opamp.advertiseAddr` | `""` | OpAMP URL injected into collector configs; defaults to in-cluster Service DNS. |
| `opamp.injectExtension` | `false` | Inject `opamp` extension into delivered configs (direct/vanilla collectors). |
| `opamp.k8sApply.enabled` | `false` | Write collector config to a ConfigMap and restart the target Deployment (vanilla mode). Requires `opamp.injectExtension: true`, plus `configMap` and `deployment`. |
| `opamp.k8sApply.configMap` | `""` | ConfigMap name to update with merged collector YAML. Must exist in the Helm release namespace. |
| `opamp.k8sApply.configMapKey` | `config.yaml` | Key within the ConfigMap. |
| `opamp.k8sApply.deployment` | `""` | Deployment to restart after a config update (rollout via pod template annotation). Must exist in the Helm release namespace. |
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

## Health probes

The agent container exposes TCP health checks on the OpAMP port (`opamp.port`,
default `4320`). Both probes use `tcpSocket` so the kubelet can detect a hung
process that still accepts connections.

| Probe | Purpose | `initialDelaySeconds` | `periodSeconds` | `failureThreshold` | `timeoutSeconds` |
| ----- | ------- | --------------------- | --------------- | ------------------ | ---------------- |
| `readinessProbe` | Remove the pod from Service endpoints until OpAMP is listening. | 5 | 10 | 3 (default) | 1 (default) |
| `livenessProbe` | Restart the container when OpAMP stops accepting TCP. | 5 | 10 | 3 | 1 |

With these defaults, kubelet restarts the agent after roughly 30 seconds of
consecutive liveness failures (3 × `periodSeconds`).

## RBAC

When `rbac.create` is true the chart creates a ClusterRole granting `get`/`list`
on `pods`, which the OpAMP server uses to match collector pods against pipeline
selector labels.

When `opamp.k8sApply.enabled` is true the ClusterRole is expanded with:

| Resource | Verbs | Purpose |
| -------- | ----- | ------- |
| `configmaps` | `get`, `update` | Write merged collector YAML to the target ConfigMap |
| `deployments` (`apps`) | `get`, `update` | Restart the collector Deployment after a config change |

These rules are cluster-scoped (ClusterRole), but the agent only updates the
ConfigMap and Deployment named in `opamp.k8sApply` within the Helm release
namespace. Install the agent and collector into the same namespace, or ensure
the ServiceAccount can reach the target resources in that namespace.
