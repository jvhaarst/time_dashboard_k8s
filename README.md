# time_dashboard_k8s

Helm chart for **<https://time.vanhaarst.net>**, the public status page of the
GPS-disciplined stratum-1 NTP server `ntp.vanhaarst.net`.

The site itself lives in
[time_dashboard](https://github.com/jvhaarst/time_dashboard) and ships as
`ghcr.io/jvhaarst/time_dashboard`. This repository only deploys it.

## Helm repository

```bash
helm repo add time-dashboard https://jvhaarst.github.io/time_dashboard_k8s
helm repo update
helm upgrade --install time-dashboard time-dashboard/time-dashboard -n websites
```

## Requirements

Three things live outside this chart and the page is unreachable without them:

| | |
|---|---|
| DNS | an `A` and `AAAA` record for `time.vanhaarst.net` |
| Router | ports 80 and 443 forwarded to the Traefik ingress |
| TLS | a `letsencrypt-production` ClusterIssuer able to answer the HTTP-01 challenge on that name |

It also needs to reach the metrics store named in `metrics.url`. That is
`VMSingle/ntp` in the `monitoring` namespace: a second VictoriaMetrics holding only this
NTP server's series, with a one-year retention, rather than the cluster's main 30d
store. A `NetworkPolicy` is not part of this chart; the cluster does not use them.

## How the page gets its numbers

The app queries the store **server-side** and renders finished HTML. The store is a
ClusterIP with no route from outside the cluster, and it stays that way: a public page
backed by a metrics store would otherwise mean exposing arbitrary PromQL to the internet
or running a query proxy in front of it. Rendering server-side also means the page works
with JavaScript off, and that a visitor cannot make the store do work — a rendered page
is reused for `cacheSeconds`.

## Configuration

| Value | Default | Notes |
|---|---|---|
| `replicaCount` | `2` | Stateless; replicas share nothing but the image. |
| `metrics.url` | `http://vmsingle-ntp.monitoring.svc.cluster.local:8429` | 8429 is a vmsingle's real listener; its Service also publishes 8428 as an alias. |
| `metrics.timeout` | `5` | Seconds per query. Must stay **below** `readinessProbe.timeoutSeconds`. |
| `cacheSeconds` | `15` | Shorter than the 30 s scrape interval buys nothing. |
| `server.name` | `ntp.vanhaarst.net` | The hostname the page tells clients to use. |
| `server.location`, `server.hardware` | | Text shown on the page. |
| `server.poolUrl` | empty | Set to state NTP Pool membership and link to the server's pool page. |
| `ingress.host` | `time.vanhaarst.net` | |

## Two choices worth keeping

**Liveness does not query the metrics store.** `/healthz` answers from the process
alone; only `/readyz` touches VictoriaMetrics. A liveness probe that depends on another
service restarts a healthy pod whenever that service is slow. On this cluster that is
not hypothetical: `zeildashboard`'s liveness probe rendered a page calling two external
APIs, one stopped answering on 2026-10-21, and liveness restarted the pod five times in
nine minutes while Apache was serving the page with 200 throughout.

**`readinessProbe.failureThreshold` is 3, not 1.** One slow query should not pull a
replica out of the ingress. Traefik's own chart shipped `failureThreshold: 1` here and a
single slow `/ping` dropped an ingress replica.

## Versioning

The published chart version is `<Chart.yaml version major.minor>.<CI run number>`;
`appVersion` tracks the published image tag and Renovate bumps it.
