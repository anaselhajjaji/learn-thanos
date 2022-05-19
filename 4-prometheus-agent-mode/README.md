# Steps

The agent mode is optimized for efficient metric scraping and forwarding (i.e. immediate metric removal once it's securely delivered to a remote location).

## Step 1: setup thanos receive and thanos query environment

### thanos receive

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

### thanos query

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

## Step 2: start prometheus in agent mode

Prometheus was designed as a stateful time-series database, and it adds certain mechanics which are not desired for full forward mode. For example:

- Prometheus builds additional memory structures for easy querying from memory.
- Prometheus does not remove data when it is safely sent via remote write. It waits for at least two hours and only after the TSDB block is persisted to the disk, it may or may not be removed, depending on retention configuration.

This is where Agent mode comes in handy! It is a native Prometheus mode built into the Prometheus binary. If you add the --agent flag when running Prometheus, it will run a dedicated, specially streamlined database, optimized for forwarding purposes, yet able to persist scraped data in the event of a crash, restart or network disconnection.

### Start first prometheus cluser

Create the data folder `mkdir prom-batmobile-data`

```bash
docker run -d --net=host --rm \
-v $(pwd)/prometheus-config/prometheus-batmobile.yaml:/etc/prometheus/prometheus.yaml \
-v $(pwd)/prom-batmobile-data:/prometheus \
-u root \
--name prom-agent-batmobile \
quay.io/prometheus/prometheus:v2.32.0-beta.0 \
--enable-feature=agent \
--config.file=/etc/prometheus/prometheus.yaml \
--storage.agent.path=/prometheus \
--web.listen-address=:9090
```

Verify that prom-agent-batmobile is running by navigating to the [Batmobile Prometheus Agent UI](http://127.0.0.1:9090/targets).

### Start second prometheus cluser

Create the data folder `mkdir prom-batcopter-data`

```bash
docker run -d --net=host --rm \
-v $(pwd)/prometheus-config/prometheus-batcopter.yaml:/etc/prometheus/prometheus.yaml \
-v $(pwd)/prom-batcopter-data:/prometheus \
-u root \
--name prom-agent-batcopter \
quay.io/prometheus/prometheus:v2.32.0-beta.0 \
--enable-feature=agent \
--config.file=/etc/prometheus/prometheus.yaml \
--storage.agent.path=/prometheus \
--web.listen-address=:9091
```

Verify that prom-agent-batcopter is running by navigating to the [Batmobile Prometheus Agent UI](http://127.0.0.1:9091/targets).

## Step3: 