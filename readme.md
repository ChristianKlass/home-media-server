# Home Media Server
This is a home media server project for me to learn how Docker and Docker-Compose works.

I intend to make it able to automatically download Movies and TV series using softwares such as Sonarr and Radarr. Of course, I'm not actually going to use it, because we all know downloading movies is illegal. But I think it's still an interesting project and I hope I learn a lot from it.

## Traefik
I'm going to be using Traefik as my "Ingress Controller". Everyone comes in through Traefik, and cannot access the various containers directly. I may decide to add some authentication in future (require authentication to access everything), maybe using Keycloak and OAuth. 

> Look at me, it almost looks like I know what I'm talking about. 
> I don't, really. I just know these words, and have a rough idea of how they work.

## Monitoring
Prometheus and Grafana are gonna be used to do monitoring. They're "industry standard" so I guess they should be very good and ready.