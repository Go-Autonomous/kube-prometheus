# Go-Autonomous Deploy

## Overlay
`kubectl apply -k goauto-k8s/environments/production` applies **everything** in the kustomization: Grafana, Prometheus, Milvus PrometheusRules, KServe monitors, Istio monitors, ingress, etc. The secrets (Alertmanager, Grafana contact points) are **not** in the kustomization—they use `$VAR` placeholders and are applied separately with `envsubst`.

---

## Diff
```bash
set -a && source .env && set +a
envsubst < goauto-k8s/environments/production/alertmanager-secret.yaml | kubectl diff -f -
envsubst < goauto-k8s/environments/production/grafana-contactpoints-secret.yaml | kubectl diff -f -
kubectl diff -k goauto-k8s/environments/production
```

## Dry-run
```bash
set -a && source .env && set +a
envsubst < goauto-k8s/environments/production/alertmanager-secret.yaml | kubectl apply -f - --dry-run=client
envsubst < goauto-k8s/environments/production/grafana-contactpoints-secret.yaml | kubectl apply -f - --dry-run=client
kubectl apply -k goauto-k8s/environments/production --dry-run=client
```

## Apply

### Secrets
Copy `.env_template` → `.env`, fill webhooks: https://api.slack.com/apps/A04SCJNFJF8/incoming-webhooks
```bash
set -a && source .env && set +a
envsubst < goauto-k8s/environments/production/alertmanager-secret.yaml | kubectl apply -f -
envsubst < goauto-k8s/environments/production/grafana-contactpoints-secret.yaml | kubectl apply -f -
```

### Overlay
```bash
kubectl apply -k goauto-k8s/environments/production
```

### Thanos (separate manifests)
Thanos lives under `goauto-k8s/environments/production/thanos/manifests/` (not in the main kustomization). Apply with:

```bash
kubectl apply -f goauto-k8s/environments/production/thanos/manifests/
```

Includes `thanos-store-podDisruptionBudget.yaml` (`maxUnavailable: 1`) for GKE upgrade safety.

#### Object storage auth (Workload Identity)
Thanos (compactor, store) and Prometheus authenticate to the GCS bucket via
Workload Identity — no static key. The KSAs `thanos-compactor`, `thanos-store`
(thanos manifests) and `prometheus-k8s` (`prometheus-serviceAccount.yaml` in the
overlay) are annotated with
`iam.gke.io/gcp-service-account: sa-thanos-storage@saga-prod-goauto-10.iam.gserviceaccount.com`.
The GSA, its bucket IAM, and the WI bindings are managed in
`terragrunt-infrastructure` (`modules/iam`). The `thanos-objstore-config` secret
only contains `type`/`bucket` (no `service_account` block).

---

## Verify
```bash
# Prometheus has Milvus rules
kubectl exec -n monitoring prometheus-k8s-0 -c prometheus -- wget -qO- 'http://localhost:9090/api/v1/rules' | jq '.data.groups[] | select(.name=="milvus") | .name'

# Alertmanager has Slack routing
kubectl exec -n monitoring alertmanager-main-0 -c alertmanager -- wget -qO- 'http://localhost:9093/api/v2/status' | jq '.config.original' | grep -E "milvus|Slack-Milvus"
```

---

## Notes
- grafana-config: gitignored, values local/Bitwarden
- No CI/CD; manual apply
- `set -a` exports vars from .env so envsubst (child process) sees them
