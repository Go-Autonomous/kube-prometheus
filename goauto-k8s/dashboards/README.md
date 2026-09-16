# Grafana dashboards and alert rules

Exported copies of the three dashboards used for cluster health, plus the
prod alert rules.

These are **not provisioned**. Grafana still reads them from its SQLite DB on
the `grafana-0` PVC; this directory is a versioned backup and review surface.
Editing a file here changes nothing until it is imported.

```
production/
  incident-monitoring.json
  service-level-overview.json
  namespaces.json
  alert-rules.json
sandbox/
  incident-monitoring.json
  service-level-overview.json
  namespaces.json
```

## Why this exists

The dashboards previously existed **only** in Grafana's PVC. If that volume
were lost they would be gone, with no record of what they contained. They are
also hand-edited in the UI, so there was no way to review a change or see when
one was made.

## The environments are NOT interchangeable

Do not copy a file from one directory to the other. They have diverged
deliberately, in both directions.

| | production | sandbox |
|---|---|---|
| datasource | **Thanos** (`c4b495c4-…`) | Prometheus (`P1809F7CD0C75ACF3`) |
| Incident Monitoring | `project='prod-goauto-1'` | `project='sandbox-goauto-3'` |

**The datasource difference is load-bearing.** Prod runs sharded Prometheus
(`shards: 2`) and no single replica holds a complete view — for example
`kube_deployment_status_replicas_ready` exists only on shard-1. Prod queries
must go through Thanos, which aggregates both. Pointing a prod dashboard at
the Prometheus service makes panels blank at random depending on which replica
answers.

*Incident Monitoring* is further environment-specific: different `$org` and
`$intent` template SQL, different external Postgres/Thanos endpoints, and
differently named panels.

## Re-exporting

Prod, via a short-lived Grafana service account token (Editor role is enough):

```bash
kubectl -n monitoring port-forward svc/grafana 3000:3000 &
curl -sH "Authorization: Bearer $TOKEN" \
  http://localhost:3000/api/dashboards/uid/<uid> | jq .dashboard > <name>.json

curl -sH "Authorization: Bearer $TOKEN" \
  http://localhost:3000/api/v1/provisioning/alert-rules | jq . > alert-rules.json
```

Sandbox Grafana uses Google OAuth with no static admin, so it is simpler to
read its SQLite DB directly:

```bash
kubectl -n monitoring cp monitoring/grafana-0:/var/lib/grafana/grafana.db ./grafana.db
sqlite3 grafana.db "select data from dashboard where uid='<uid>' and is_folder=0"
```

Strip `id` and `version` from the exported JSON — they are instance-specific
and produce noisy diffs.

## Restoring

Import via **Dashboards → Import**, or:

```bash
jq '{dashboard: ., overwrite: true}' <name>.json \
  | curl -sX POST -H "Authorization: Bearer $TOKEN" \
      -H "Content-Type: application/json" -d @- \
      http://localhost:3000/api/dashboards/db
```

The `uid` is preserved in each file, so an import updates the existing
dashboard in place rather than creating a duplicate. Grafana keeps the previous
version, so a bad import can be rolled back from the dashboard's **Versions**
tab.

## alert-rules.json

Prod only — sandbox has no equivalent rules. Exported from the provisioning
API, so it can be re-applied with `PUT /api/v1/provisioning/alert-rules/<uid>`.

Two rules are currently paused (`Customer Perceived Latency`, and the
per-organisation processing-time rule). Note the latter also filters on
`handler="/v1/extract-mail-request"`, a path that no longer exists — the
current one is `/v2/…` — so it would match nothing even if unpaused.

Restoring a rule requires the `X-Disable-Provenance: true` header, otherwise
Grafana marks it read-only in the UI.
