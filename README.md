# k8s-infra

Cluster addons for [proxclus](https://github.com/ykantoni/proxclus),
reconciled by Argo CD. No Terraform, no CI access to the cluster: merge to
`main` and Argo CD syncs it.

vm-infra's Terraform installs Argo CD and creates the root Application
`root-k8s-infra` (project `bootstrap-infra`), which syncs `bootstrap/` of this
repository into the `argocd` namespace. Every file there is one child
Application in project `k8s-infra` (any namespace, any cluster-scoped kind).

## What's here

| Wave | Application             | Namespace         | Source                                   |
| ---- | ----------------------- | ----------------- | ---------------------------------------- |
| -1   | `external-secrets`      | `external-secrets` | `external-secrets` chart (operator + CRDs) |
| 0    | `cilium-lb-ipam`        | `kube-system`     | `charts/cilium-lb-ipam` (pool + L2 policy) |
| 0    | `lb-services`           | `kube-system`/`argocd` | `charts/lb-services` (LoadBalancers for Hubble UI, Argo CD) |
| 1    | `longhorn`              | `longhorn-system` | `longhorn` chart; default StorageClass   |
| 2    | `metrics-server`        | `kube-system`     | `metrics-server` chart                   |
| 2    | `cnpg-operator`         | `cnpg-system`     | `cloudnative-pg` chart (operator + CRDs) |
| 2    | `gpu-operator`          | `gpu-operator`    | NVIDIA `gpu-operator` chart              |
| 2    | `openbao`               | `openbao`         | `openbao` chart (3-node Raft) + `manifests/openbao` |
| 3    | `kube-prometheus-stack` | `monitoring`      | `kube-prometheus-stack` chart + `manifests/kube-prometheus-stack` |

Waves wait on the previous wave being Healthy: vm-infra's `modules/argocd`
restores Argo CD's health check for `Application` resources, without which
waves between child Applications do nothing. Every Application also retries
forever with backoff, so anything that races a CRD simply converges.

Not here, on purpose:

- **Cilium itself** — it's the CNI Argo CD runs on, so vm-infra installs it.
  Only the LB-IPAM pool and L2 announcement policy live here.
- **Argo CD itself** — installed by vm-infra, not self-managed.
- **Applications** — those live in k8s-apps, a separate Argo CD pipeline with
  a narrower AppProject.

## Layout

```
bootstrap/<addon>.yaml         one Argo CD Application per addon
values/<addon>/values.yaml     Helm values for chart-repo addons ($values ref)
charts/<chart>/                local charts (their own values.yaml)
manifests/<addon>/             ExternalSecrets and other raw manifests for that addon
```

Chart-repo addons use multi-source Applications: the chart from its Helm
repository, plus this repository as `ref: values` for the values file, plus
(where needed) `manifests/<addon>` as a plain directory source.

## Adding or changing an addon

1. Add `bootstrap/<addon>.yaml` (copy `longhorn.yaml`: chart, version,
   namespace, wave) and `values/<addon>/values.yaml`.
2. If its chart repository isn't in vm-infra's
   `modules/argocd` `k8s_infra_chart_repos`, add it there too — the
   `k8s-infra` AppProject rejects unlisted sources.
3. Namespaces that need more than RKE2's CIS-default `restricted` Pod
   Security level get it through `managedNamespaceMetadata.labels`.
4. Open a PR; `lint.yml` renders local charts and schema-checks every
   manifest. Merge, and Argo CD syncs.

Upgrading an addon is a `targetRevision` bump in its `bootstrap/` file.

## Secrets

Secret values live in OpenBao (the `openbao` Application: three Raft nodes,
2Gi Longhorn PVC each, Shamir-sealed). External Secrets Operator syncs them
into ordinary Kubernetes Secrets, so the repository only holds
`ExternalSecret` manifests, which carry no values. `lint.yml` fails on any
plain `kind: Secret`.

Store a value under `secret/<namespace>/<name>`, then commit an
ExternalSecret reading it through `ClusterSecretStore/openbao`:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: <name>
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: openbao
  target:
    name: <name>
  dataFrom:
    - extract:
        key: <namespace>/<name>
```

Initialisation, unsealing after pod restarts, key backup and writing values
are covered in `manifests/openbao/README.md`.

Secrets expected today:

| OpenBao path                          | ExternalSecret                                   | Used by |
| ------------------------------------- | ------------------------------------------------ | ------- |
| `secret/monitoring/grafana-admin`     | `manifests/kube-prometheus-stack/grafana-admin.yaml` | Grafana login (`admin-user`, `admin-password`) |

## Notes per addon

- **longhorn** — `preUpgradeChecker.jobEnabled: false`, as Longhorn documents
  for Argo CD installs. Uninstalling needs Longhorn's
  `deleting-confirmation-flag`; vm-infra's `just destroy` sets it.
- **gpu-operator** — driver and toolkit disabled (baked into vm-infra's GPU
  template). It creates the `nvidia` RuntimeClass and, through NFD, labels GPU
  nodes `nvidia.com/gpu.present=true`; GPU pods set
  `runtimeClassName: nvidia` and request `nvidia.com/gpu`.
- **kube-prometheus-stack** — discovers every ServiceMonitor/PodMonitor/
  PrometheusRule cluster-wide (selector `NilUsesHelmValues: false`).
  Prometheus, Grafana are LoadBalancer Services on the LB-IPAM pool.
- **cnpg-operator** — here rather than in k8s-apps because its CRDs are
  cluster-scoped; k8s-apps only creates `Cluster` objects.
- **openbao** — readiness probe accepts sealed/uninitialised pods, so Argo CD
  sees the StatefulSet Healthy and runs the bootstrap Job; clients go through
  the `openbao-active` Service, which only selects the unsealed leader.
  `updateStrategyType` is the chart's `OnDelete`: after a chart upgrade,
  delete the pods one at a time and `just bao-unseal` each.
