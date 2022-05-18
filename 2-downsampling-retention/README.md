# Steps

## Step 1: Generate artificial metrics for 1 year

We will use our handy thanosbench project to do so! Let's generate Prometheus data (in form of TSDB blocks) with just 5 series (gauges) that spans from a year ago until now (-6h)!

Execute the following command (should take few seconds):

```bash
mkdir -p prom-eu1 && docker run -i quay.io/thanos/thanosbench:v0.2.0-rc.1 block plan -p continuous-365d-tiny --labels 'cluster="eu1"' --max-time=6h | docker run -v $(pwd)/prom-eu1:/prom-eu1 -i quay.io/thanos/thanosbench:v0.2.0-rc.1 block gen --output.dir prom-eu1
```

## Step 2: Starts prometheus instance

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prometheus0_eu1.yml:/etc/prometheus/prometheus.yml \
    -v $(pwd)/prom-eu1:/prometheus \
    -u root \
    --name prometheus-0-eu1 \
    quay.io/prometheus/prometheus:v2.20.0 \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.retention.time=1000d \
    --storage.tsdb.path=/prometheus \
    --storage.tsdb.max-block-duration=2h \
    --storage.tsdb.min-block-duration=2h \
    --web.listen-address=:9090 \
    --web.enable-lifecycle \
    --web.enable-admin-api
```

We shoyld be able to query for 1 year of data: <http://127.0.0.1:9090/graph?g0.range_input=1y&g0.expr=continuous_app_metric0&g0.tab=0>

## Step 3: adds Sidecar and querier

- Add sidecar

```bash
docker run -d --net=host --rm \
    --name prometheus-0-eu1-sidecar \
    -u root \
    quay.io/thanos/thanos:v0.26.0 \
    sidecar \
    --http-address 0.0.0.0:19090 \
    --grpc-address 0.0.0.0:19190 \
    --prometheus.url http://127.0.0.1:9090
```

- Add querier

```bash
docker run -d --net=host --rm \
    --name querier \
    quay.io/thanos/thanos:v0.26.0 \
    query \
    --http-address 0.0.0.0:9091 \
    --query.replica-label replica \
    --store 127.0.0.1:19190
```

- Verify by browsing querier UI <http://127.0.0.1:9091/stores> and also query for 1y <http://127.0.0.1:9091/graph?g0.expr=continuous_app_metric0&g0.tab=0&g0.stacked=0&g0.range_input=1y&g0.max_source_resolution=0s&g0.deduplicate=1&g0.partial_response=0&g0.store_matches=%5B%5D>

