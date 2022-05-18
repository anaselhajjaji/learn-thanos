# Steps

## Step 1: Generate artificial metrics for 1 year

We will use our handy thanosbench project to do so! Let's generate Prometheus data (in form of TSDB blocks) with just 5 series (gauges) that spans from a year ago until now (-6h)!

Execute the following command (should take few seconds):

```bash
mkdir -p prom-eu1 && docker run -i quay.io/thanos/thanosbench:v0.2.0-rc.1 block plan -p continuous-365d-tiny --labels 'cluster="eu1"' --max-time=6h | docker run -v $(pwd)/prom-eu1:/prom-eu1 -i quay.io/thanos/thanosbench:v0.2.0-rc.1 block gen --output.dir prom-eu1
```
