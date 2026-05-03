I want to deploy an instance of home-assistant ( https://www.home-assistant.io/installation/linux/ ). The online docs suggest this docker compose service:

```
services:
  homeassistant:
    container_name: homeassistant
    image: "ghcr.io/home-assistant/home-assistant:stable"
    volumes:
      - /PATH_TO_YOUR_CONFIG:/config
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro
    restart: unless-stopped
    privileged: true
    network_mode: host
    environment:
      TZ: Europe/Amsterdam
```

This repository uses argocd to deploy services to a k3s kubernetes cluseter homelab. A previous thread suggested this Helm chart ( https://community.home-assistant.io/t/home-assistant-helm-chart/665003 ):

``` bash
ingress:
  # Enable ingress for home assistant
  enabled: true
  className: "traefik"
  annotations:
    kubernetes.io/ingress.class: "traefik"
    kubernetes.io/tls-acme: "true"
    traefik.ingress.kubernetes.io/router.entrypoints: "websecure"
    traefik.ingress.kubernetes.io/router.tls: "true"
  hosts:
    - host: your.host.name
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls:
    - hosts:
      - your.host.name
      secretName: your-tls-secret
```

But we use Envoy Gateway, not traefik.

I also found this helm chart: https://github.com/pajikos/home-assistant-helm-chart .

How can we deploy home-assistant using argocd to our k3s cluster?
