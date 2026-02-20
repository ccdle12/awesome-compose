## Compose sample

## Updates for Bitseek Deployment

I added the below to `docker-compose.yml`:

- For `prometheus` we need to add the external url that the dashbaord will resolve to, this needs to match the url and path in nginx
- `grafana` also requires the URL to be at the `GF_SERVER_ROOT_URL`
- I also had to add an docker internal gateway for prometheus
- `nginx-exporter` also was added to read nginx logs and stats
    - It scrapes the logs from an nginx config at the resource path `http://127.0.0.1/nginx_status`
- `cadvisor`: for monitoring docker containers


```
     container_name: prometheus
     command:
       - '--config.file=/etc/prometheus/prometheus.yml'
+      - '--web.external-url=https://chat.bitseek.ai/monitor/'
+      - '--web.route-prefix=/'
     ports:
       - 9090:9090
     restart: unless-stopped
+    extra_hosts:
+      - "host.docker.internal:host-gateway"
     volumes:
       - ./prometheus:/etc/prometheus
       - prom_data:/prometheus
+
   grafana:
     image: grafana/grafana
     container_name: grafana
@@ -19,7 +24,51 @@ services:
     environment:
       - GF_SECURITY_ADMIN_USER=admin
       - GF_SECURITY_ADMIN_PASSWORD=grafana
+      - GF_SERVER_ROOT_URL=https://chat.bitseek.ai/ui/
     volumes:
       - ./grafana:/etc/grafana/provisioning/datasources
+
+
+  nginx-exporter:
+    image: nginx/nginx-prometheus-exporter:latest
+    container_name: nginx-exporter
+    network_mode: "host"
+    restart: unless-stopped
+    ports:
+      - 9113:9113
+    command:
+      - '-nginx.scrape-uri=https://127.0.0.1/nginx_status'
+
+  cadvisor:
+    image: gcr.io/cadvisor/cadvisor:latest
:
```

- To monitor the different services, we will need copy the alerts in `prometheus/docker_alert.yml` but update the name of each container service (not container name)

```
```

- I think for Bitseek, I'll need to change the network and change it to the nodes network



### Prometheus & Grafana

Project structure:
```
.
├── compose.yaml
├── grafana
│   └── datasource.yml
├── prometheus
│   └── prometheus.yml
└── README.md
```

[_compose.yaml_](compose.yaml)
```
services:
  prometheus:
    image: prom/prometheus
    ...
    ports:
      - 9090:9090
  grafana:
    image: grafana/grafana
    ...
    ports:
      - 3000:3000
```
The compose file defines a stack with two services `prometheus` and `grafana`.
When deploying the stack, docker compose maps port the default ports for each service to the equivalent ports on the host in order to inspect easier the web interface of each service.
Make sure the ports 9090 and 3000 on the host are not already in use.

## Deploy with docker compose

```
$ docker compose up -d
Creating network "prometheus-grafana_default" with the default driver
Creating volume "prometheus-grafana_prom_data" with default driver
...
Creating grafana    ... done
Creating prometheus ... done
Attaching to prometheus, grafana

```

## Expected result

Listing containers must show two containers running and the port mapping as below:
```
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
dbdec637814f        prom/prometheus     "/bin/prometheus --c…"   8 minutes ago       Up 8 minutes        0.0.0.0:9090->9090/tcp   prometheus
79f667cb7dc2        grafana/grafana     "/run.sh"                8 minutes ago       Up 8 minutes        0.0.0.0:3000->3000/tcp   grafana
```

Navigate to `http://localhost:3000` in your web browser and use the login credentials specified in the compose file to access Grafana. It is already configured with prometheus as the default datasource.

![page](output.jpg)

Navigate to `http://localhost:9090` in your web browser to access directly the web interface of prometheus.

Stop and remove the containers. Use `-v` to remove the volumes if looking to erase all data.
```
$ docker compose down -v
```
