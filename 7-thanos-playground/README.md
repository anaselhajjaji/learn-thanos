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