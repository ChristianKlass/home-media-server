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

### Grafana
Grafana is.. a dashboarding platform for visualizing metrics. I just use it to check on the status of the server, basically. I use pre-built dashboards because I don't know how to make 'em, just yet.

The dashboards I'm using are:
  * [Traefik 2 Dashboard](https://grafana.com/grafana/dashboards/11462) - 11462
  * [Node Exporter Full](https://grafana.com/grafana/dashboards/1860) - 1860

> In case you're wondering, the number at the back is the ID of the Dashboard's ID. You can just use this number to import the dashboard in Grafana's GUI.

If we're going to serve Grafana on a subpath using Traefik, we need to add the following environment variables into the `docker-compose.yml` file:
```
environment:
  - 'GF_SERVER_ROOT_URL=%(protocol)s://%(domain)s:%(http_port)s/grafana'
  - 'GF_SERVER_SERVE_FROM_SUB_PATH=true'
```

If you need to install plugins for Grafana, you can place the folder in the following location:
```
volumes:
  - './config/grafana/plugins:/var/lib/grafana/plugins:rw'
```
> TODO: Eventually, I want to set up a MYSQL database so that Grafana can use it. But not so soon, I guess :P


## Media server
### Radarr
**From their own Github page:** Sonarr is a PVR for Usenet and BitTorrent users. It can monitor multiple RSS feeds for new episodes of your favorite shows and will grab, sort and rename them. It can also be configured to automatically upgrade the quality of files already downloaded when a better quality format becomes available.

Remember to add your media and download folders:
```
volumes:
  - '${USERDIR}/Downloads/completed:/downloads'
  - '${USERDIR}/media/movies:/movies'
```

Read more about it their [Github](https://github.com/Sonarr/Sonarr) and on their [official website](https://sonarr.tv/).

## Storage
### Victoria Metrics
We need a place to store the long term metrics that are collected by Prometheus, which only keeps 2 weeks or so (I think) of metrics.

Apparently, Victoria Metrics is a drop in replacement for Prometheus' storage, which means that we can direct Prometheus to write to a Victoria Metrics instance, then in Grafana, just add the Victoria Metrics instance as a Prometheus datasource.

Read more about it on their [Github](https://victoriametrics.github.io/) and their [official website](https://victoriametrics.com/).
