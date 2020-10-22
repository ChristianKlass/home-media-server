# Home Media Server
This is a home media server project for me to learn how Docker and Docker-Compose works. This is really a journey of my self-learning adventure, and I may make a lot of changes to things as I learn more about how the things work.

I intend to make it able to automatically download Movies and TV series using softwares such as Sonarr and Radarr. Of course, I'm not actually going to use it, because we all know downloading movies is illegal. But I think it's still an interesting project and I hope I learn a lot from it.

## Traefik
I'm going to be using Traefik as my "Ingress Controller". Everyone comes in through Traefik, and cannot access the various containers directly. I may decide to add some authentication in future (require authentication to access everything), maybe using Keycloak and OAuth.

> Look at me, it almost looks like I know what I'm talking about.
> I don't, really. I just know these words, and have a rough idea of how they work.

## Monitoring
Prometheus and Grafana are gonna be used to do monitoring. They're "industry standard" so I guess they should be very good and ready.

### Prometheus
Prometheus is going to be used to scrape the various services for metrics. I am pretty sure that I won't be using the metrics for anything, but it's good to collect them, and to *know how* to collect them, I guess. I'll probably be using Victoria Metrics to store the long term metrics.

For some reason, Prometheus requires that we use the `--web.external-url=http://${ROOT_URL}/prometheus'`, since we're using Traefik to do the redirection. I guess we could use Traefik to strip the prefix and all that, but honestly, I'm not bothered by this addition.

The way Prometheus works is by a configuration file. This is set via the `'--config.file=/etc/prometheus/prometheus.yml'` flag in `docker-compose.yml`. A (very simple) example of the prometheus.yml file is given below, in case you don't have one:

```
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 10s

remote_write:
  - url: "http://victoria-metrics:8428/api/v1/write"

scrape_configs:
  - job_name: prometheus
    honor_timestamps: true
    metrics_path: '/prometheus/metrics'
    static_configs:
      - targets:
          - 'prometheus:9090'

  - job_name: node-exporter
    honor_timestamps: true
    static_configs:
      - targets:
          - 'node-exporter:9100'

# More information about the Prometheus config file can be found at:
# https://prometheus.io/docs/prometheus/latest/configuration/configuration/
```
Note: we're able to use the names of the services because we're using Docker. So instead of having to type something like `url: "http://192.168.56.101:8428/api/v1/write"` we can just use `http://victoria-metrics:8428/api/v1/write`. I think Docker has some sort of DNS or something that it uses internally.

## Storage
### Victoria Metrics
We need a place to store the long term metrics that are collected by Prometheus, which only keeps 2 weeks or so (I think) of metrics.

Apparently, Victoria Metrics is a drop in replacement for Prometheus' storage, which means that we can direct Prometheus to write to a Victoria Metrics instance, then in Grafana, just add the Victoria Metrics instance as a Prometheus datasource.

Read more about it here: https://victoriametrics.github.io/
