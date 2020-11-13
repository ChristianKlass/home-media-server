# Home Media Server
This is a home media server project for me to learn how Docker and Docker-Compose works. This is really a journey of my self-learning adventure, and I may make a lot of changes to things as I learn more about how the things work.

I intend to make it able to automatically download Movies and TV series using softwares such as Sonarr and Radarr. Of course, I'm not actually going to use it, because we all know downloading movies is illegal. But I think it's still an interesting project and I hope I learn a lot from it.

## How To Use
1. Make sure you've installed the prerequisites (From the Before You Start section):
    - Installed Docker and Docker-Compose
    - Set the environment variables
2. In the same directory as the `docker-compose.yml`, run the command:
```
docker-compose up -d
```

## Using Docker Volumes
**Note:** I know that I don't use the Docker volumes in this build, and you totally can if you want. I just want it to be easy to move my stuff around if/when I need to. In my experience, Docker volumes are clunky to move around. I can't just copy a volume from one computer to another easily. If this is possible, please let me know, it would be nice.

To use Docker volumes, change the `docker-compose.yml` file to the following. I'll just use Loki as an example (because it's small), do the same for the rest of the services:
```
version: '3.6'

volumes:
  loki-config: {}

services:
  loki:
  image: grafana/loki:latest
  container_name: loki
  hostname: loki
  restart: always
  command:
    - '--config.file=/etc/loki/local-config.yml'
  env_file: 
    - './env/common.env'
  expose:
    - '3100'
  volumes:
    - 'loki-config:/etc/loki:ro'
```

So as you can see, it's as simple as adding a new volume (via the `volumes` key), and changing the service definition to use the new volume (via the `- 'loki-config:/etc/loki:ro'`).

## Before You Start
Here are some prerequistes you need to install before you can use this Home Media Server.

### Environment Variables
You need to add the following to your environment variables. I don't know the best way to do it, but I use the `/etc/environment` file. If the file doesn't exist, create it:
```
TZ=Asia/Singapore # Change this to your timezone in this format
ROOT_URL=some.address # The address you'd like to access your server from.

PUID=1000
PGID=1001
USERDIR=/path/to/keep/stuff             # I use this to store the media/download/config stuff

```

To get the PUID and PGID, you can run `id` in your terminal and it will give you some output like:
```
uid=1000(mark) gid=1000(mark) groups=1000(mark),10(wheel),1001(docker)
```
The important ones you're looking for are `uid` (PUID) and the `gid` for docker (PGID).

Also, I've stopped using the environment variables directly in the `docker-compose.yml` file. I'm using `.env` files now. I just think it's neater. For example, I have a `common.env` file which I want all the services to use:
```
# common.env
PUID=${PUID}
PGID=${PGID}
TZ=${TZ}
```

Then I use it in the `docker-compose.yml` with the `env_file` key like so (let's use Loki as an example again because it's short):
```
# docker-compose.yml
loki:
  image: grafana/loki:latest
  container_name: loki
  hostname: loki
  restart: always
  command:
    - '--config.file=/etc/loki/local-config.yml'
  env_file: 
    - './env/common.env'
  expose:
    - '3100'
  volumes:
    - './config/loki:/etc/loki:ro'
```

So you can use this for any of the environment variables you need, so you don't have to put everything in one file, like `prometheus.env`, `grafana.env`, etc.

### Docker
You'll also need to install Docker and Docker-Compose. You can do so by following the [official instructions](https://docs.docker.com/engine/install/) from their website. 

Basically, I'm using Fedora, I just need to do add the repository with the below commands.

If you're using some other distro, follow the instructions they've provided on their instructions:
```
sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
```

Then do the install with the package manager:
```
sudo dnf install docker-ce docker-ce-cli containerd.io
```

You can also use the [convenience script](https://docs.docker.com/engine/install/fedora/#install-using-the-convenience-script) they've provided, but I've personally never used it. 

Remember to follow the [post installation steps](https://docs.docker.com/engine/install/linux-postinstall/). It makes life a lot easier.

### Docker Compose
Installing [Docker Compose](https://docs.docker.com/compose/install/) is much easier. Obviously, you'll need to install Docker first, otherwise it won't do anything.

Just download the following file to the `/usr/local/bin/` folder and make it executable:
```
sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose
```

Then you'll be able to run the `docker-compose` commands.

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
Grafana is a dashboarding platform for visualizing metrics. I just use it to check on the status of the server, basically. I use pre-built dashboards because I don't know how to make 'em, just yet.

The dashboards I'm using are:
  * [Traefik 2 Dashboard](https://grafana.com/grafana/dashboards/11462) - 11462
  * [Node Exporter Full](https://grafana.com/grafana/dashboards/1860) - 1860

> In case you're wondering, the number at the back is the ID of the Dashboard's ID. You can just use this number to import the dashboard in Grafana's GUI.

If we're going to serve Grafana on a subpath using Traefik, we need to add the following environment variables into the `./env/grafana.env` file:
```
GF_SERVER_ROOT_URL=%(protocol)s://%(domain)s:%(http_port)s/grafana
GF_SERVER_SERVE_FROM_SUB_PATH=true
GF_SECURITY_ALLOW_EMBEDDING=true # this is needed to use Grafana with Muximux.
```

If you need to install plugins for Grafana, you can place the folder in the following location:
```
volumes:
  - './config/grafana/plugins:/var/lib/grafana/plugins:rw'
```

## Media server
**Note:** For most of these apps, you'll need to first expose their port using the `docker-compose.yml` so that you can access the web UI. You can do this with the `ports` key:
```
sonarr:
  image: linuxserver/sonarr:preview
  container_name: sonarr
  hostname: sonarr
  expose:
    - '8989'
  ports:
    - '8989:8989'
  ...
```
The `expose` and `ports` keys are different.

I don't think the `expose` does anything, it's more like a reminder that these ports are exposed **within** the docker network. This means that you can do access Sonarr within the network with http://sonarr:8989. It seems to work without it as well. I just use it as a way to document which ports are open for a particular app.

Anyway, the info about this can be found in the [Docker-Compose documentation](https://docs.docker.com/compose/compose-file/#expose).

To try this, you can go into a container which can `ping`. I like to use the Grafana container, because :shrug:
```
docker exec -it grafana bash

# you'll get a response similar to this:
bash-5.0$ ping sonarr
PING sonarr (172.28.0.9): 56 data bytes
ping: permission denied (are you root?)
```
The ping won't actually work, but you can see that they'll resolve the `sonarr` name to the an IP address.

The `ports` key maps the internal container port to the host's port so that you can access it. This can also be found in the [Docker-Compose documentation](https://docs.docker.com/compose/compose-file/#ports) as well.

Ports are mapped in a `host-port:container-port` configuration. It works like this:
```
ports:
  - '8989:8989' #this maps the host's port 8989 to the container's port 8989.
```
This example is not particularly helpful, but it's what was done. :3 So to access the container, you can do `http://<host.ip.address>:8989`.

For a more helpful example, let's pretend we want to map host port `1234` to Sonarr's `8989` port, so that we can access Sonarr with `http://<host.ip.address>:1234`:
```
ports:
  - '1234:8989' #it's always <host-port>:<container-port>
```

After you expose them, you need to fumble around in their settings page to find something like `Base URL`. Change the base URL to the address you provided in the `PathPrefix` label for Traefik. It's the second label:
```
sonarr:
  image: linuxserver/sonarr:preview
  ...
  labels:
    - 'traefik.enable=true'
    - 'traefik.http.routers.sonarr.rule=Host(`${ROOT_URL}`) && PathPrefix(`/sonarr`)'
    - 'traefik.http.services.sonarr.loadbalancer.server.port=8989'
```

After you do this, you can remove the `ports` key in the `docker-compose.yml` file, and you can access them with `http://<root-url>/sonarr`.

### Jellyfin
Jellyfin is the thing that lets you watch your media. It will collect, play, and stream the media. Currently, I will only use it for (legally obtained) movies and TV shows, but I think it can do audiobooks, pictures, and some other stuff, although I haven't tested it yet.

There are also Android and iOS apps that you can download and connect to this server so you can watch/listen/whatever on your mobile devices.

**Note:** For some reason the Android Jellyfin app doesn't work with the `http://${ROOT_URL}/jellyfin` so for now I've just exposed the port so I can access with `http://<ip-address>:8096`.

Maybe one day I'll figure out how to get this to work reliably. It works now, but it doesn't connect consistently. Maybe it's my DNS setting and has absolutely nothing to do with this config. Nevertheless, doing this helps it connect stably, so.. Anyway, to do this, just add the `ports:` config:
```
jellyfin:
  ...
  ports:
    - '8096:8096'
  ...
```

Read more on their [website](https://jellyfin.org/) and their [Github](https://github.com/jellyfin/jellyfin)

### Muximux
This is basically the frontend for the whole HMS. However, you need to make sure your Traefik is working properly, because when it asks for the links to the pages to display, it won't work with the Docker internal names. I think it's because it actually loads an iFrame or something like that. Whatever the case, if someone finds a way to make it work, let me know. :D

Otherwise, it's actually a really good piece of software, it's fast and looks good.

Read more about it on their [Github](https://github.com/mescon/Muximux)

### Jackett
**From their Github:** Jackett works as a proxy server: it translates queries from apps (Sonarr, Radarr, SickRage, CouchPotato, Mylar, Lidarr, DuckieTV, qBittorrent, Nefarious etc.) into tracker-site-specific http queries, parses the html response, then sends results back to the requesting software.

It's a single point of indexing and translates it to a format that Sonarr and Radarr can understand (I think).

Read more about it on their [Github](https://github.com/Jackett/Jackett).

### Sonarr
**From their website:** Radarr is a movie collection manager for Usenet and BitTorrent users. It can monitor multiple RSS feeds for new movies and will interface with clients and indexers to grab, sort, and rename them. It can also be configured to automatically upgrade the quality of existing files in the library when a better quality format becomes available.

Remember to add your media and download folders:
```
volumes:
  - '${USERDIR}/Downloads/completed:/downloads'
  - '${USERDIR}/media/tv-shows:/tv-shows'
```

Read more about it on their [Github](https://hub.docker.com/r/linuxserver/radarr) and on their [official website](https://radarr.video/).

### Radarr
**From their Github:** Sonarr is a PVR for Usenet and BitTorrent users. It can monitor multiple RSS feeds for new episodes of your favorite shows and will grab, sort and rename them. It can also be configured to automatically upgrade the quality of files already downloaded when a better quality format becomes available.

Remember to add your media and download folders:
```
volumes:
  - '${USERDIR}/Downloads/completed:/downloads'
  - '${USERDIR}/media/movies:/movies'
```

Read more about it on their [Github](https://github.com/Sonarr/Sonarr) and on their [official website](https://sonarr.tv/).

## Storage
### Victoria Metrics
We need a place to store the long term metrics that are collected by Prometheus, which only keeps 2 weeks or so (I think) of metrics.

Apparently, Victoria Metrics is a drop in replacement for Prometheus' storage, which means that we can direct Prometheus to write to a Victoria Metrics instance, then in Grafana, just add the Victoria Metrics instance as a Prometheus datasource.

To direct Prometheus to write to Victoria Metrics, we can use the `remote_write` feature in Prometheus. You can do this via the `prometheus.yml` file (you may have noticed this in the example provided above):
```
remote_write:
 - url: "http://victoria-metrics:8428/api/v1/write"
```

Read more about it on their [Github](https://victoriametrics.github.io/) and their [official website](https://victoriametrics.com/).

### Postgres
I've converted Grafana to use Postgres as its database rather than just using the built in one. This is not really useful now, but I think if there's many instances of Grafana, they can use a shared database so that their data (users, dashboards[I think?], and settings[I think also?]) are consistent between the various instances.

I've added the following env variables to use with Grafana:
```
GF_DATABASE_TYPE=postgres
GF_DATABASE_HOST=grafana-postgres:5432
GF_DATABASE_USER=postgres-grafana
GF_DATABASE_NAME=postgres-grafana
GF_DATABASE_PASSWORD=postgres      # these are just placeholders, make sure you change it. don't use the default password.
GF_DATABASE_SSL_MODE=disable
```