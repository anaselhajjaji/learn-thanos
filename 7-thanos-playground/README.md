# Steps

Playground for a demo <https://www.youtube.com/watch?v=j4TAGO019HU&feature=emb_title&ab_channel=DavidMcKay>

## Step 1: Generate test data

- Plan for 1y of metrics data

```bash
docker run -it --rm quay.io/thanos/thanosbench:v0.2.0-rc.1 block plan -p continuous-365d-tiny --max-time=6h > $(pwd)/block-spec.yaml
```

- Plan and generate data for cluster="eu1", replica="0" Prometheus

```bash
mkdir $(pwd)/prom-eu1-replica0 && docker run -it --rm quay.io/thanos/thanosbench:v0.2.0-rc.1 block plan -p continuous-365d-tiny --labels 'cluster="eu1"' --max-time=6h | \
    docker run -v $(pwd)/prom-eu1-replica0:/out -i quay.io/thanos/thanosbench:v0.2.0-rc.1 block gen --output.dir /out
```

- Plan and generate data for cluster="eu1", replica="1" Prometheus

```bash
mkdir $(pwd)/prom-eu1-replica1 && docker run -it --rm quay.io/thanos/thanosbench:v0.2.0-rc.1 block plan -p continuous-365d-tiny --labels 'cluster="eu1"' --max-time=6h | \
    docker run -v $(pwd)/prom-eu1-replica1:/out -i quay.io/thanos/thanosbench:v0.2.0-rc.1 block gen --output.dir /out
```

- Plan and generate data for cluster="us1", replica="0" Prometheus

```bash
mkdir $(pwd)/prom-us1-replica0 && docker run -it --rm quay.io/thanos/thanosbench:v0.2.0-rc.1 block plan -p continuous-365d-tiny --labels 'cluster="us1"' --max-time=6h | \
    docker run -v $(pwd)/prom-us1-replica0:/out -i quay.io/thanos/thanosbench:v0.2.0-rc.1 block gen --output.dir /out
```

## Step 2: Deploy prometheus

- cluster="eu1", replica="0" Prometheus

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prom-eu1-replica0-config.yaml:/etc/prometheus/prometheus.yml \
    -v $(pwd)/prom-eu1-replica0:/prometheus \
    -u root \
    --name prom-eu1-0 \
    quay.io/prometheus/prometheus:v2.20.0 \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/prometheus \
    --storage.tsdb.retention.time=1000d \
    --storage.tsdb.max-block-duration=2h \
    --storage.tsdb.min-block-duration=2h \
    --web.listen-address=:9091 \
    --web.enable-lifecycle \
    --web.enable-admin-api
```

- cluster="eu1", replica="1" Prometheus

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prom-eu1-replica1-config.yaml:/etc/prometheus/prometheus.yml \
    -v $(pwd)/prom-eu1-replica1:/prometheus \
    -u root \
    --name prom-eu1-1 \
    quay.io/prometheus/prometheus:v2.20.0 \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/prometheus \
    --storage.tsdb.retention.time=1000d \
    --storage.tsdb.max-block-duration=2h \
    --storage.tsdb.min-block-duration=2h \
    --web.listen-address=:9092 \
    --web.enable-lifecycle \
    --web.enable-admin-api
```

- cluster="us1", replica="0" Prometheus

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prom-us1-replica0-config.yaml:/etc/prometheus/prometheus.yml \
    -v $(pwd)/prom-us1-replica0:/prometheus \
    -u root \
    --name prom-us1-0 \
    quay.io/prometheus/prometheus:v2.20.0 \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/prometheus \
    --storage.tsdb.retention.time=1000d \
    --storage.tsdb.max-block-duration=2h \
    --storage.tsdb.min-block-duration=2h \
    --web.listen-address=:9093 \
    --web.enable-lifecycle \
    --web.enable-admin-api
```

## Step 3: deploy sidecars

- cluster="eu1", replica="0" sidecar

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prom-eu1-replica0-config.yaml:/etc/prometheus/prometheus.yml \
    --name prom-eu1-0-sidecar \
    -u root \
    quay.io/thanos/thanos:v0.26.0 \
    sidecar \
    --http-address 0.0.0.0:19091 \
    --grpc-address 0.0.0.0:19191 \
    --reloader.config-file /etc/prometheus/prometheus.yml \
    --prometheus.url "http://127.0.0.1:9091"
```

- cluster="eu1", replica="1" sidecar

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prom-eu1-replica1-config.yaml:/etc/prometheus/prometheus.yml \
    --name prom-eu1-1-sidecar \
    -u root \
    quay.io/thanos/thanos:v0.26.0 \
    sidecar \
    --http-address 0.0.0.0:19092 \
    --grpc-address 0.0.0.0:19192 \
    --reloader.config-file /etc/prometheus/prometheus.yml \
    --prometheus.url "http://127.0.0.1:9092"
```

- cluster="us1", replica="0" sidecar

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prom-us1-replica0-config.yaml:/etc/prometheus/prometheus.yml \
    --name prom-us1-0-sidecar \
    -u root \
    quay.io/thanos/thanos:v0.26.0 \
    sidecar \
    --http-address 0.0.0.0:19093 \
    --grpc-address 0.0.0.0:19193 \
    --reloader.config-file /etc/prometheus/prometheus.yml \
    --prometheus.url "http://127.0.0.1:9093"
```

## Step 4: deploy the querier

