# Steps

## Step 1: Deploy prometheus and thanos stack

- create volume `mkdir -p prometheus_data`

- start prometheus

```bash
for i in $(seq 0 2); do
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prometheus"${i}".yml:/etc/prometheus/prometheus.yml \
    -v $(pwd)/prometheus_data:/prometheus"${i}" \
    -u root \
    --name prometheus"${i}" \
    quay.io/prometheus/prometheus:v2.22.2 \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/prometheus \
    --web.listen-address=:909"${i}" \
    --web.enable-lifecycle \
    --web.enable-admin-api; done
```

- inject thanos sidecars

```bash
for i in $(seq 0 2); do
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prometheus"${i}".yml:/etc/prometheus/prometheus.yml \
    --name prometheus-sidecar"${i}" \
    -u root \
    quay.io/thanos/thanos:v0.26.0 \
    sidecar \
    --http-address=0.0.0.0:1909"${i}" \
    --grpc-address=0.0.0.0:1919"${i}" \
    --reloader.config-file=/etc/prometheus/prometheus.yml \
    --prometheus.url=http://127.0.0.1:909"${i}"; done
```

- inject thanos querier

```bash
docker run -d --net=host --rm \
    --name querier \
    quay.io/thanos/thanos:v0.26.0 \
    query \
    --http-address 0.0.0.0:10912 \
    --grpc-address 0.0.0.0:10901 \
    --query.replica-label replica \
    --store 127.0.0.1:19190 \
    --store 127.0.0.1:19191 \
    --store 127.0.0.1:19192
```

- verify in querier UI <http://127.0.0.1:10912/stores>

## Step 2: Deploy nginx to simulate latency

- start nginx

```bash
docker run -d --net=host --rm \
    -v $(pwd)/nginx/nginx.conf:/etc/nginx/conf.d/default.conf \
    --name nginx \
    yannrobert/docker-nginx
```
