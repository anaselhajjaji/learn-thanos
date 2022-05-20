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

```bash
docker run -d --net=host --rm \
    --name querier \
    quay.io/thanos/thanos:v0.26.0 \
    query \
    --http-address 0.0.0.0:9090 \
    --grpc-address 0.0.0.0:19190 \
    --query.replica-label replica \
    --store 127.0.0.1:19191 \
    --store 127.0.0.1:19192 \
    --store 127.0.0.1:19193
```

now you can browse to the querier UI <http://127.0.0.1:9090>

## Step 5: long term retention setup

### run minio

```bash
mkdir $(pwd)/minio && \
docker run -d --rm --name minio \
     -v $(pwd)/minio:/data \
     -p 9000:9000 -e "MINIO_ACCESS_KEY=anas" -e "MINIO_SECRET_KEY=anaselhajjaji" \
     minio/minio:RELEASE.2019-01-31T00-31-19Z \
    server /data
```

and create thanos bucket `mkdir $(pwd)/minio/thanos`

We can browse to minio ui to verify: <http://127.0.0.1:9000>

### configure sidecar to upload blocks

- stop sidecars

```bash
docker stop prom-eu1-0-sidecar
docker stop prom-eu1-1-sidecar
docker stop prom-us1-0-sidecar
```

- cluster="eu1", replica="0" sidecar

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prom-eu1-replica0-config.yaml:/etc/prometheus/prometheus.yml \
    -v $(pwd)/minio-config/minio-bucket.yaml:/etc/thanos/minio-bucket.yaml \
    -v $(pwd)/prom-eu1-replica0:/prometheus \
    --name prom-eu1-0-sidecar \
    -u root \
    quay.io/thanos/thanos:v0.26.0 \
    sidecar \
    --tsdb.path /prometheus \
    --objstore.config-file /etc/thanos/minio-bucket.yaml \
    --shipper.upload-compacted \
    --http-address 0.0.0.0:19091 \
    --grpc-address 0.0.0.0:19191 \
    --reloader.config-file /etc/prometheus/prometheus.yml \
    --prometheus.url "http://127.0.0.1:9091"
```

- cluster="eu1", replica="1" sidecar

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prom-eu1-replica1-config.yaml:/etc/prometheus/prometheus.yml \
    -v $(pwd)/minio-config/minio-bucket.yaml:/etc/thanos/minio-bucket.yaml \
    -v $(pwd)/prom-eu1-replica1:/prometheus \
    --name prom-eu1-1-sidecar \
    -u root \
    quay.io/thanos/thanos:v0.26.0 \
    sidecar \
    --tsdb.path /prometheus \
    --objstore.config-file /etc/thanos/minio-bucket.yaml \
    --shipper.upload-compacted \
    --http-address 0.0.0.0:19092 \
    --grpc-address 0.0.0.0:19192 \
    --reloader.config-file /etc/prometheus/prometheus.yml \
    --prometheus.url "http://127.0.0.1:9092"
```

- cluster="us1", replica="0" sidecar

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prom-us1-replica0-config.yaml:/etc/prometheus/prometheus.yml \
    -v $(pwd)/minio-config/minio-bucket.yaml:/etc/thanos/minio-bucket.yaml \
    -v $(pwd)/prom-us1-replica0:/prometheus \
    --name prom-us1-0-sidecar \
    -u root \
    quay.io/thanos/thanos:v0.26.0 \
    sidecar \
    --tsdb.path /prometheus \
    --objstore.config-file /etc/thanos/minio-bucket.yaml \
    --shipper.upload-compacted \
    --http-address 0.0.0.0:19093 \
    --grpc-address 0.0.0.0:19193 \
    --reloader.config-file /etc/prometheus/prometheus.yml \
    --prometheus.url "http://127.0.0.1:9093"
```

We can check whether the data is uploaded into thanos bucket by visiting <http://127.0.0.1:9000> It will take a minute to synchronize all blocks. Note that sidecar by default uploads only "non compacted by Prometheus" blocks.

## Step 6: query long term data

- start store gateway

```bash
docker run -d --net=host --rm \
    -v $(pwd)/minio-config/minio-bucket.yaml:/etc/thanos/minio-bucket.yaml \
    --name store-gateway \
    quay.io/thanos/thanos:v0.26.0 \
    store \
    --objstore.config-file /etc/thanos/minio-bucket.yaml \
    --http-address 0.0.0.0:19094 \
    --grpc-address 0.0.0.0:19194
```

- point query to the store gateway

```bash
docker stop querier && \
docker run -d --net=host --rm \
    --name querier \
    quay.io/thanos/thanos:v0.26.0 \
    query \
    --http-address 0.0.0.0:9090 \
    --grpc-address 0.0.0.0:19190 \
    --query.replica-label replica \
    --store 127.0.0.1:19191 \
    --store 127.0.0.1:19192 \
    --store 127.0.0.1:19193 \
    --store 127.0.0.1:19194
```

visit querier UI to verify <http://127.0.0.1:9090>

## Step 7: retention dedup and downsamling

- start the compactor

```bash
docker run -d --net=host --rm \
    -v $(pwd)/minio-config/minio-bucket.yaml:/etc/thanos/minio-bucket.yaml \
    --name compactor \
    quay.io/thanos/thanos:v0.26.0 \
    compact \
    --wait --wait-interval 30s \
    --consistency-delay 0s \
    --objstore.config-file /etc/thanos/minio-bucket.yaml \
    --http-address 0.0.0.0:19095
```

visit compactor ui to verify <http://127.0.0.1:19095/loaded>

visit querier UI to make queries <http://127.0.0.1:9090>
