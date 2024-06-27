# Traefik-with-OpenTelemetry-and-Elastic-APM

## Step 1: Deploy Elastic APM Server

If using Elastic Cloud:
1. Log in to Elastic Cloud
2. Create a new deployment with APM enabled
3. Note down the APM Server URL and secret token

If self-managed:
1. Deploy Elasticsearch and Kibana
2. Deploy APM Server using Helm:

```bash
helm repo add elastic https://helm.elastic.co
helm install apm-server elastic/apm-server \
  --set apmConfig.secret_token=<your-secret-token> \
  --set apmConfig.elasticsearch.hosts=["https://<your-elasticsearch-host>:9200"]
```

## Step 2: Deploy OpenTelemetry Collector

1. Create a ConfigMap for the OpenTelemetry Collector:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
data:
  otel-collector-config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
          http:
    
    processors:
      batch:
    
    exporters:
      logging:
        loglevel: debug
      otlp/elastic:
        endpoint: "<your-apm-server-url>:8200"
        headers:
          Authorization: "Bearer <your-apm-secret-token>"
    
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [logging, otlp/elastic]
        metrics:
          receivers: [otlp]
          processors: [batch]
          exporters: [logging, otlp/elastic]
```

2. Deploy the OpenTelemetry Collector:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
spec:
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
      - name: otel-collector
        image: otel/opentelemetry-collector-contrib:latest
        args:
        - "--config=/conf/otel-collector-config.yaml"
        volumeMounts:
        - name: otel-collector-config
          mountPath: /conf
      volumes:
      - name: otel-collector-config
        configMap:
          name: otel-collector-config
---
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
spec:
  selector:
    app: otel-collector
  ports:
  - port: 4317
    targetPort: 4317
    name: grpc-otlp
  - port: 4318
    targetPort: 4318
    name: http-otlp
```

Apply these manifests:

```bash
kubectl apply -f otel-collector.yaml
```

## Step 3: Deploy Traefik

1. Create a values file for Traefik (traefik-values.yaml):

```yaml
additionalArguments:
  - "--tracing.otlp=true"
  - "--tracing.otlp.grpc=true"
  - "--tracing.otlp.endpoint=otel-collector.default.svc.cluster.local:4317"
  - "--tracing.otlp.insecure=true"
```

2. Install Traefik using Helm:

```bash
helm repo add traefik https://helm.traefik.io/traefik
helm install traefik traefik/traefik -f traefik-values.yaml
```

## Step 4: Configure Traefik IngressRoute

Create an IngressRoute for your application:

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: my-app
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`my-app.example.com`)
    kind: Rule
    services:
    - name: my-app-service
      port: 80
```

Apply this manifest:

```bash
kubectl apply -f ingressroute.yaml
```

## Step 5: Verify Traces in Elastic APM

1. Log in to Kibana
2. Navigate to Observability > APM
3. Look for the "traefik" service in the Services overview
4. Click on the "traefik" service to view traces and metrics
