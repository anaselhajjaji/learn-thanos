# Steps

In this tutorial we will more implement realtime global view and this will need thanos receive instead of sidecar because we cannot access the Thanos Sidecar directly - we cannot query metrics data in real-time. Thanos Sidecar only uploads blocks of metrics data that have been written to disk, which happens every 2 hours in Prometheus.
This means that the Global View would be at least 2 hours out of date, and does not satisfy realtime requirement.

Thanos Receive is a component that implements the Prometheus Remote Write API. This means that it will accept metrics data that is sent to it by other Prometheus instances (or any other process that implements Remote Write API).

Prometheus can be configured to Remote Write. This means that Prometheus will send all of its metrics data to a remote endpoint as they are being ingested - useful for our requirements!

In its simplest form, when Thanos Receive receives data from Prometheus, it stores it locally and exposes a Store API server so this data can be queried by Thanos Query.

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

## Step 2: deploy thanos receive and thanos query

- create data folder: `mkdir receive-data`

- Start thanos receive

```bash
docker run -d --rm \
    -v $(pwd)/receive-data:/receive/data \
    --net=host \
    --name receive \
    quay.io/thanos/thanos:v0.21.0 \
    receive \
    --tsdb.path "/receive/data" \
    --grpc-address 127.0.0.1:10907 \
    --http-address 127.0.0.1:10909 \
    --label "receive_replica=\"0\"" \
    --label "receive_cluster=\"wayne-enterprises\"" \
    --remote-write.address 127.0.0.1:10908
```

This starts Thanos Receive that listens on <http://127.0.0.1:10908/api/v1/receive> endpoint for Remote Write and on 127.0.0.1:10907 for Thanos StoreAPI

We can verify that the thanos receive works well by browsing <http://127.0.0.1:10909/metrics>

- start thanos query

```bash
docker run -d --rm \
    --net=host \
    --name query \
    quay.io/thanos/thanos:v0.21.0 \
    query \
    --http-address "0.0.0.0:39090" \
    --store "127.0.0.1:10907"
```

we can verify here <http://127.0.0.1:39090/stores>

- make sure that in prometheus remote write is present in prometheus config so prometheus will write data to thanos receive.

Since we supplied the --web.enable-lifecycle flag in our Prometheus instances, we can dynamically reload the configuration by curl-ing the /-/reload endpoints.

```bash
curl -X POST http://127.0.0.1:9090/-/reload
curl -X POST http://127.0.0.1:9091/-/reload
```

we can verify the prometheus config: <http://127.0.0.1:9090/config> and <http://127.0.0.1:9091/config>

To make sure that we can query data from each of our Prometheus instances from our Thanos Query instance. Navigate to the Thanos Query UI <http://127.0.0.1:39090>, and query for a metric like up - inspect the output and you should see batcave and batcomputer in the cluster label.

At this point, we have:

- Two prometheus instances configured to remote_write.
- Thanos Receive component ingesting data from Prometheus
- Thanos Query component configured to query Thanos Receive's Store API.
