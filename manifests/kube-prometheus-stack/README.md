# kube-prometheus-stack ExternalSecrets

Synced into `monitoring` by the `kube-prometheus-stack` Application
(third source). Only `*.yaml`/`*.yml`/`*.json` files here are applied.

`grafana-admin.yaml` is the Grafana login referenced by
`values/kube-prometheus-stack/values.yaml` (`grafana.admin.existingSecret`).
It syncs OpenBao's `secret/monitoring/grafana-admin` into Secret
`grafana-admin`. Write the value once (see `manifests/openbao/README.md`
for the token):

```bash
kubectl -n openbao exec -it openbao-0 -- sh -c \
  'BAO_TOKEN=<token> bao kv put secret/monitoring/grafana-admin \
     admin-user=admin admin-password=<choose one>'
```

Until the value exists, Grafana's pod waits on the missing Secret.
Prometheus and Alertmanager run regardless.
