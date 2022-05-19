# Steps

## Step 1: deploy prometheus

- start first cluster

create data folder: `mkdir prometheus-batcave-data`

then, start prometheus

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prometheus-batcave.yaml:/etc/prometheus/prometheus.yaml \
    -v $(pwd)/prometheus-batcave-data:/prometheus \
    -u root \
    --name prometheus-batcave \
    quay.io/prometheus/prometheus:v2.27.0 \
    --config.file=/etc/prometheus/prometheus.yaml \
    --storage.tsdb.path=/prometheus \
    --web.listen-address=:9090 \
    --web.enable-lifecycle
```

verify that it's running by browsing <http://127.0.0.1:9090>

- start the second cluster

create data folder: `mkdir prometheus-batcomputer-data`

then, start prometheus

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prometheus-batcomputer.yaml:/etc/prometheus/prometheus.yaml \
    -v $(pwd)/prometheus-batcomputer-data:/prometheus \
    -u root \
    --name prometheus-batcomputer \
    quay.io/prometheus/prometheus:v2.27.0 \
    --config.file=/etc/prometheus/prometheus.yaml \
    --storage.tsdb.path=/prometheus \
    --web.listen-address=:9091 \
    --web.enable-lifecycle
```

verify that it's running by browsing <http://127.0.0.1:9091>