# OpenBao bootstrap

Synced into `openbao` by the `openbao` Application (third source). Only
`*.yaml`/`*.yml`/`*.json` files here are applied.

- `bootstrap-job.yaml`: a Sync hook (wave 1) that initialises OpenBao on
  first sync, unseals all three pods that one time, and on every sync
  (re-)applies its configuration: kv-v2 at `secret/`, Kubernetes auth, and
  the `external-secrets` and `openbao-bootstrap` policies and roles.
- `cluster-secret-store.yaml`: `ClusterSecretStore/openbao` (wave 2), which
  External Secrets Operator reads through with the `external-secrets` role.

## First build

The first sync writes the unseal keys (5 shares, threshold 3) and root token
to Secret `openbao/openbao-init`. Move them off the cluster straight away,
from vm-infra on the runner host:

```bash
just bao-keys-backup   # saves /var/lib/terraform/openbao-init.json, deletes the Secret
```

Keep a second copy off that host. Losing the keys means losing every secret.

## Unsealing

Seal is Shamir: any OpenBao pod that restarts (node reboot, upgrade, eviction)
comes back sealed. From vm-infra:

```bash
just bao-status
just bao-unseal
```

Existing Kubernetes Secrets synced by ExternalSecrets stay in place while
OpenBao is sealed. Only refreshes and new secrets wait.

## Writing secrets

Paths follow `secret/<namespace>/<name>`, matching the ExternalSecret that
reads them. With the root token from the backup, or any token allowed to
write there:

```bash
kubectl -n openbao exec -it openbao-0 -- sh -c \
  'BAO_TOKEN=<token> bao kv put secret/monitoring/grafana-admin \
     admin-user=admin admin-password=<choose one>'
```

## Rebuilds

A rebuilt cluster gets a fresh, empty OpenBao with new keys. Re-`put` the
secrets, or restore a Raft snapshot taken beforehand
(`bao operator raft snapshot save` / `restore`).
