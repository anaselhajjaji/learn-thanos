# Introduction

Thanos tutorial inspired from:

Inspired from: <https://www.katacoda.com/thanos>

## Step 1: Start prometheus instances

- Prepare "persistent volumes"

```bash
mkdir -p prometheus0_eu1_data prometheus0_us1_data prometheus1_us1_data
```

- Deploy "EU1"

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prometheus0_eu1.yml:/etc/prometheus/prometheus.yml \
    -v $(pwd)/prometheus0_eu1_data:/prometheus \
    -u root \
    --name prometheus-0-eu1 \
    quay.io/prometheus/prometheus:v2.14.0 \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/prometheus \
    --web.listen-address=:9090 \
    --web.enable-lifecycle \
    --web.enable-admin-api
```

- Deploy "US1"

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prometheus0_us1.yml:/etc/prometheus/prometheus.yml \
    -v $(pwd)/prometheus0_us1_data:/prometheus \
    -u root \
    --name prometheus-0-us1 \
    quay.io/prometheus/prometheus:v2.14.0 \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/prometheus \
    --web.listen-address=:9091 \
    --web.enable-lifecycle \
    --web.enable-admin-api
```

and

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prometheus1_us1.yml:/etc/prometheus/prometheus.yml \
    -v $(pwd)/prometheus1_us1_data:/prometheus \
    -u root \
    --name prometheus-1-us1 \
    quay.io/prometheus/prometheus:v2.14.0 \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/prometheus \
    --web.listen-address=:9092 \
    --web.enable-lifecycle \
    --web.enable-admin-api
```

- Setup verification

<http://localhost:9090>
<http://localhost:9091>
<http://localhost:9092>

Look for `prometheus_tsdb_head_series` metric for example.

## Step 2: adding sidecars

- add sidecar to "EU1" Prometheus

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prometheus0_eu1.yml:/etc/prometheus/prometheus.yml \
    --name prometheus-0-sidecar-eu1 \
    -u root \
    quay.io/thanos/thanos:v0.26.0 \
    sidecar \
    --http-address 0.0.0.0:19090 \
    --grpc-address 0.0.0.0:19190 \
    --reloader.config-file /etc/prometheus/prometheus.yml \
    --prometheus.url http://127.0.0.1:9090
```

- Adding sidecars to each replica of Prometheus in "US1"

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prometheus0_us1.yml:/etc/prometheus/prometheus.yml \
    --name prometheus-0-sidecar-us1 \
    -u root \
    quay.io/thanos/thanos:v0.26.0 \
    sidecar \
    --http-address 0.0.0.0:19091 \
    --grpc-address 0.0.0.0:19191 \
    --reloader.config-file /etc/prometheus/prometheus.yml \
    --prometheus.url http://127.0.0.1:9091
```

and

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prometheus1_us1.yml:/etc/prometheus/prometheus.yml \
    --name prometheus-1-sidecar-us1 \
    -u root \
    quay.io/thanos/thanos:v0.26.0 \
    sidecar \
    --http-address 0.0.0.0:19092 \
    --grpc-address 0.0.0.0:19192 \
    --reloader.config-file /etc/prometheus/prometheus.yml \
    --prometheus.url http://127.0.0.1:9092
```

- Uncomment the 3 lines after L12 on the 3 prometheus config files to include the sidecars, can be verified by browsing to <http://localhost:9090/config> or by searching for `thanos_sidecar_prometheus_up` metric.

## Adding thanos querier

- Deploy thanos querier

```bash
docker run -d --net=host --rm \
    --name querier \
    quay.io/thanos/thanos:v0.26.0 \
    query \
    --http-address 0.0.0.0:29090 \
    --query.replica-label replica \
    --store 127.0.0.1:19190 \
    --store 127.0.0.1:19191 \
    --store 127.0.0.1:19192
```

- Verify by browsing to querier UI <http://localhost:29090/stores> and search for `sum(prometheus_tsdb_head_series)`.
So how Thanos Querier knows how to deduplicate correctly?

If we would look again into Querier configuration we can see that we also set query.replica-label flag. This is exactly the label Querier will try to deduplicate by for HA groups. This means that any metric with exactly the same labels except replica label will be assumed as the metric from the same HA group, and deduplicated accordingly.

If we would open prometheus1_us1.yml config file in the editor or if you go to Prometheus 1 US1 /config. you should see our external labels in external_labels YAML option:

  external_labels:
    cluster: us1
    replica: 1
Now if we compare to prometheus0_us1.yaml:

  external_labels:
    cluster: us1
    replica: 0
We can see that since those two replicas scrape the same targets, any metric will be produced twice. Once by replica=1, cluster=us1 Prometheus and once by replica=0, cluster=us1 Prometheus. If we configure Querier to deduplicate by replica we can transparently handle this High Available pair of Prometheus instances to the user.