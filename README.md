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
| -1   | `sealed-secrets`        | `kube-system`     | bitnami-labs `sealed-secrets` chart      |
| 0    | `cilium-lb-ipam`        | `kube-system`     | `charts/cilium-lb-ipam` (pool + L2 policy) |
| 0    | `lb-services`           | `kube-system`/`argocd` | `charts/lb-services` (LoadBalancers for Hubble UI, Argo CD) |
| 1    | `longhorn`              | `longhorn-system` | `longhorn` chart; default StorageClass   |
| 2    | `metrics-server`        | `kube-system`     | `metrics-server` chart                   |
| 2    | `cnpg-operator`         | `cnpg-system`     | `cloudnative-pg` chart (operator + CRDs) |
| 2    | `gpu-operator`          | `gpu-operator`    | NVIDIA `gpu-operator` chart              |
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
manifests/<addon>/             SealedSecrets and other raw manifests for that addon
pub-cert.pem                   Sealed Secrets public cert, for offline sealing
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

The repository is public, so secrets are committed only as `SealedSecret`s,
encrypted to the in-cluster Sealed Secrets controller. `lint.yml` fails on any
plain `kind: Secret`.

```bash
kubectl create secret generic <name> -n <namespace> \
  --from-literal=<key>=<value> --dry-run=client -o yaml \
| kubeseal --cert pub-cert.pem -o yaml > manifests/<addon>/<name>.sealed.yaml
```

`pub-cert.pem` comes from `just seal-cert` in vm-infra once the controller is
running; commit it here. The controller's private key is restored by
vm-infra on every rebuild (see vm-infra's `modules/argocd/README.md`), so
sealed files stay valid across cluster rebuilds.

Secrets expected today:

| File                                                      | Used by                         |
| --------------------------------------------------------- | ------------------------------- |
| `manifests/kube-prometheus-stack/grafana-admin.sealed.yaml` | Grafana login (`admin-user`, `admin-password`) |

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
