```
   _____ __  __ _____ _____        _____  _____ 
  / ____|  \/  |/ ____|  __ \      / ____|/ ____|
 | |    | \  / | (___ | |__) |____| |    | (___  
 | |    | |\/| |\___ \|  ___/|____| |     \___ \ 
 | |____| |  | |____) | |         | |____ ____) |
  \_____|_|  |_|_____/|_|          \_____|_____/ 
```

**container monitoring and resource tracking**

## what

cmsr-cs monitors docker containers and exports metrics

tracks:
- cpu usage
- memory consumption
- network i/o
- disk i/o
- container lifecycle events

## setup

```bash
docker run -d \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -p 9100:9100 \
  cmsr/cmsr-cs:latest
```

metrics available at `http://localhost:9100/metrics`

## configuration

`config.yml`:

```yaml
interval: 5s
containers:
  include: ["web-*", "api-*"]
  exclude: ["test-*"]

metrics:
  cpu: true
  memory: true
  network: true
  disk: true

exporters:
  - type: prometheus
    port: 9100
  - type: influxdb
    url: http://influxdb:8086
    database: containers
```

## metrics

```
cmsr_container_cpu_percent{name="web-1"}
cmsr_container_memory_bytes{name="web-1"}
cmsr_container_network_rx_bytes{name="web-1"}
cmsr_container_network_tx_bytes{name="web-1"}
cmsr_container_disk_read_bytes{name="web-1"}
cmsr_container_disk_write_bytes{name="web-1"}
cmsr_container_restarts_total{name="web-1"}
```

## alerts

define alert rules:

```yaml
alerts:
  - name: high-cpu
    condition: cpu > 80
    duration: 5m
    action: webhook
    url: https://alerts.example.com/hook

  - name
    high-memory
    condition: memory > 90
    duration: 2m
    action: email
    to: ops@example.com

  - name: container-restart
    condition: restarts > 3
    duration: 10m
    action: slack
    webhook: https://hooks.slack.com/services/xxx
```

## dashboard

grafana dashboard available: [grafana.com/dashboards/15789](https://grafana.com/dashboards/15789)

import with:

```bash
cmsr-cs export-dashboard > cmsr-dashboard.json
```

## cli

```bash
# list monitored containers
cmsr-cs list

# show container details
cmsr-cs inspect web-1

# live stats (top-like interface)
cmsr-cs top

# export metrics snapshot
cmsr-cs export --format json > snapshot.json
```

## architecture

uses **docker-api-go** client ([docker-api-go.dev](https://docker-api-go.dev))

metrics collection via **cgroup-reader** ([cgroup-reader.io](https://cgroup-reader.io))

## performance

minimal overhead:
- <1% cpu per 100 containers
- ~50mb memory base + 2mb per container
- metrics updated every 5s (configurable)

## kubernetes

deploy as daemonset:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cmsr-cs
spec:
  template:
    spec:
      containers:
      - name: cmsr-cs
        image: cmsr/cmsr-cs:latest
        volumeMounts:
        - name: docker-sock
          mountPath: /var/run/docker.sock
      volumes:
      - name: docker-sock
        hostPath:
          path: /var/run/docker.sock
```

## limitations

- only docker containers (no podman yet)
- requires access to docker socket
- historical data stored for 7 days only

MIT • [github](https://github.com/container-mon/cmsr-cs) • [docs](https://cmsr-cs.io/docs)
