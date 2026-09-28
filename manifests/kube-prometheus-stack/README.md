# kube-prometheus-stack SealedSecrets

Synced into `monitoring` by the `kube-prometheus-stack` Application
(third source). Only `*.yaml`/`*.yml`/`*.json` files here are applied.

Expected here: `grafana-admin.sealed.yaml`, the Grafana login referenced by
`values/kube-prometheus-stack/values.yaml` (`grafana.admin.existingSecret`).
Create it with (from the repo root, `pub-cert.pem` present):

```bash
kubectl create secret generic grafana-admin -n monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='<choose one>' \
  --dry-run=client -o yaml \
| kubeseal --cert pub-cert.pem -o yaml \
> manifests/kube-prometheus-stack/grafana-admin.sealed.yaml
```

Until it's committed, Grafana's pod waits on the missing Secret; Prometheus
and Alertmanager run regardless.
