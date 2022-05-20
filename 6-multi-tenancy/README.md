# Steps

## Step 1: deploy prometheus instances

- prepare the volumes

```bash
mkdir -p prometheus0_fruit_data prometheus0_veggie_data prometheus1_veggie_data
```

- start team fruit prometheus with sidecar

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prometheus0_fruit.yml:/etc/prometheus/prometheus.yml \
    -v $(pwd)/prometheus0_fruit_data:/prometheus \
    -u root \
    --name prometheus-0-fruit \
    quay.io/prometheus/prometheus:v2.20.0 \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/prometheus \
    --web.listen-address=:9090 \
    --web.enable-lifecycle \
    --web.enable-admin-api
```

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prometheus0_fruit.yml:/etc/prometheus/prometheus.yml \
    --name prometheus-0-sidecar-fruit \
    -u root \
    quay.io/thanos/thanos:v0.26.0 \
    sidecar \
    --http-address 0.0.0.0:19090 \
    --grpc-address 0.0.0.0:19190 \
    --reloader.config-file /etc/prometheus/prometheus.yml \
    --prometheus.url http://127.0.0.1:9090
```

- same for team viggie but with 2-replica prometheus

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prometheus0_veggie.yml:/etc/prometheus/prometheus.yml \
    -v $(pwd)/prometheus0_veggie_data:/prometheus \
    -u root \
    --name prometheus-0-veggie \
    quay.io/prometheus/prometheus:v2.20.0 \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/prometheus \
    --web.listen-address=:9091 \
    --web.enable-lifecycle \
    --web.enable-admin-api
```

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prometheus0_veggie.yml:/etc/prometheus/prometheus.yml \
    --name prometheus-0-sidecar-veggie \
    -u root \
    quay.io/thanos/thanos:v0.26.0 \
    sidecar \
    --http-address 0.0.0.0:19091 \
    --grpc-address 0.0.0.0:19191 \
    --reloader.config-file /etc/prometheus/prometheus.yml \
    --prometheus.url http://127.0.0.1:9091
```

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prometheus1_veggie.yml:/etc/prometheus/prometheus.yml \
    -v $(pwd)/prometheus1_veggie_data:/prometheus \
    -u root \
    --name prometheus-1-veggie \
    quay.io/prometheus/prometheus:v2.20.0 \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/prometheus \
    --web.listen-address=:9092 \
    --web.enable-lifecycle \
    --web.enable-admin-api
```

```bash
docker run -d --net=host --rm \
    -v $(pwd)/prometheus-config/prometheus1_veggie.yml:/etc/prometheus/prometheus.yml \
    --name prometheus-01-sidecar-veggie \
    -u root \
    quay.io/thanos/thanos:v0.26.0 \
    sidecar \
    --http-address 0.0.0.0:19092 \
    --grpc-address 0.0.0.0:19192 \
    --reloader.config-file /etc/prometheus/prometheus.yml \
    --prometheus.url http://127.0.0.1:9092
```

## Step 2: deploy thanos querier

- fruit querier

```bash
docker run -d --net=host --rm \
    --name querier-fruit \
    quay.io/thanos/thanos:v0.26.0 \
    query \
    --http-address 0.0.0.0:29091 \
    --grpc-address 0.0.0.0:29191 \
    --query.replica-label replica \
    --store 127.0.0.1:19190
```

- veggie querier

```bash
docker run -d --net=host --rm \
    --name querier-veggie \
    quay.io/thanos/thanos:v0.26.0 \
    query \
    --http-address 0.0.0.0:29092 \
    --grpc-address 0.0.0.0:29192 \
    --query.replica-label replica \
    --store 127.0.0.1:19191 \
    --store 127.0.0.1:19192
```

This setup can be called "No or Hard Tenancy" where we are setting up separate components (technically disconnected two systems) for each of tenants.

Once started you should be able to reach both Queriers - each exposing either Fruit's or Veggies's data: <http://127.0.0.1:29091> and <http://127.0.0.1:29092>

## Step 3: thanos query multi tenancy

What if we can have one set of Queries instead of separate set for each tenant? Why not reusing exactly same component?

- Let's stop fruit and veggies queriers and run single one spanning all the tenant's Prometheus data:

```bash
docker stop querier-fruit && docker stop querier-veggie
```

- start a new querier

```bash
docker run -d --net=host --rm \
    --name querier-multi \
    quay.io/thanos/thanos:v0.26.0 \
    query \
    --http-address 0.0.0.0:29090 \
    --grpc-address 0.0.0.0:29190 \
    --query.replica-label replica \
    --store 127.0.0.1:19190 \
    --store 127.0.0.1:19191 \
    --store 127.0.0.1:19192
```

Within short time we should be able to see "Tomato" view when we open [Querier UI](http://127.0.0.1:29090)

Undoubtedly, the main problem with this setup is that by default every tenant will see each other data, similar to what you have in Prometheus, if single Prometheus scrapes data from multiple teams.

- [prom-label-proxy](https://github.com/prometheus-community/prom-label-proxy) allows read tenancy for all the resources that Prometheus and Thanos currently exposes, by enforcing certain tenant label to be used in all APIs retrieving the data. This proxy works natively for Prometheus, but since Thanos uses same HTTP APIs on top, it will work for us as well.

```bash
docker run -d --net=host --rm \
    --name prom-label-proxy \
    quay.io/prometheuscommunity/prom-label-proxy:v0.3.0 \
    -label tenant \
    -upstream http://127.0.0.1:29090 \
    -insecure-listen-address 0.0.0.0:39090 \
    -enable-label-apis
```

All requests now have to have extra URL parameter tenant= with the value being tenant to limit scope with.

- Our running proxy does not do any authN or authZ for us - so let's setup some basic flow. To make it simple, let's deploy Caddy server (kind of fancy nginx but written in Go) that will expose two ports: 39091 that will redirect to prom-label-proxy with tenant=team-fruit and 39092 for tenant=team-veggie injection.

```bash
docker run -d --net=host --rm \
    --name caddy \
    -v $PWD/caddy-config/Caddyfile:/etc/caddy/Caddyfile \
    caddy:2.2.1
```

- to verify:

team fruit: `curl -g 'http://127.0.0.1:39091/api/v1/query?query=up'`

team veggie: `curl -g 'http://127.0.0.1:39092/api/v1/query?query=up'`
