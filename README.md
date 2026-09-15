# Alert History for kube-prometheus-stack (Zabbix-style)

A Zabbix "Problem history"-style table for Grafana, built entirely on core
Grafana features and the native Prometheus `ALERTS` metric — **no custom
plugin required**. Designed for [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
(Prometheus Operator + Alertmanager), where alert rules are managed as
`PrometheusRule` CRDs rather than Grafana-managed alert rules, so Grafana's
built-in "Alerting → History" page is empty and not useful there.

Also published on the [Grafana dashboard gallery](https://grafana.com/grafana/dashboards/).

## What it shows

**Currently Active Problems (live snapshot)**
An instant query on `ALERTS{alertstate="firing"}`, refreshed on the
dashboard's own refresh interval (default: every minute). Columns: checked-at
timestamp, severity (color-coded), problem name, namespace, pod, container,
instance.

**Problem History (last 7 days, based on ALERTS metric)**
A range query on the same metric, grouped into one row per firing episode
(started / last seen / duration), using only core Grafana transformations:
Labels to fields → Group by → Add field from calculation → Sort by. Columns:
started, problem name, container, instance, namespace, pod, severity
(color-coded), last seen (relative time), duration.

"Pending" alerts (rules that matched but haven't crossed their `for:`
duration yet) are intentionally excluded — only alerts that actually reached
`firing` state are shown.

## Requirements

- A Prometheus data source — the same Prometheus instance kube-prometheus-stack
  deploys (or any Prometheus/Thanos/Mimir endpoint that exposes the `ALERTS`
  metric).
- No exporter, no extra `scrape_config`, no additional recording rules.
- History depth is limited by your Prometheus TSDB retention (commonly
  1–2 weeks on default kube-prometheus-stack installs) — this is a
  Prometheus-side query, not a separate long-term alert log.

## Files in this repo

| File | Purpose |
|---|---|
| `alert-history.json` | Portable dashboard export (`${DS_PROMETHEUS}` templated) for manual **Import** into any Grafana, or for uploading to the grafana.com gallery. |
| `alert-history-configmap.yaml` | Kubernetes `ConfigMap` for GitOps-style provisioning via the kube-prometheus-stack Grafana **sidecar** (`grafana_dashboard: "1"` label). |

## Setup

### Option A — Manual import
1. In Grafana: **Dashboards → New → Import**.
2. Upload `alert-history.json`.
3. When prompted, pick your Prometheus data source.
4. Done — no dashboard variables, no additional configuration needed.

### Option B — GitOps / ConfigMap provisioning (kube-prometheus-stack sidecar)
kube-prometheus-stack's Grafana pod ships with a sidecar
(`grafana-sc-dashboard`) that watches for `ConfigMap`s labeled
`grafana_dashboard: "1"` in the release namespace and loads them
automatically — no Helm upgrade needed.

```bash
kubectl apply -f alert-history-configmap.yaml -n <kube-prometheus-stack-namespace>
```

> **Note:** this file already has its `datasource.uid` resolved to the literal
> value `prometheus`. If your Prometheus data source has a different UID,
> edit the two `"uid": "prometheus"` occurrences in the embedded JSON before
> applying — direct file/ConfigMap provisioning does **not** resolve the
> `${DS_PROMETHEUS}` template variable the way the Grafana Import wizard
> does, so a `${DS_PROMETHEUS}`-style placeholder here would show "No data"
> on every panel.

## Known limitations

- Only reflects alerts Prometheus itself has evaluated and fired — alerts
  silenced or inhibited in Alertmanager still show here as "firing" from
  Prometheus's point of view, since Alertmanager-side suppression happens
  downstream of the `ALERTS` metric.
- "Currently active" and "history" are two independent queries (instant vs.
  range) against the same metric; they are not linked/drill-down panels.

## Contributing

Issues and pull requests are welcome — e.g. additional columns, alternate
groupings, or adapting the transforms for other alerting label schemes.

## License

[MIT](LICENSE)
