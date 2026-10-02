# Part 49: Kubernetes Deployment สำหรับ Erlang

## สารบัญ
1. [Erlang บน Kubernetes](#erlang-บน-kubernetes)
2. [Dockerfile สำหรับ Production](#dockerfile-สำหรับ-production)
3. [Kubernetes Manifests](#kubernetes-manifests)
4. [Distributed Erlang บน Kubernetes](#distributed-erlang-บน-kubernetes)
5. [Helm Charts](#helm-charts)
6. [ตัวอย่างจริง: Production Deployment](#ตัวอย่างจริง-production-deployment)

---

## Erlang บน Kubernetes

```yaml
# ความท้าทายของ Erlang บน Kubernetes:
# 1. Node naming: ต้องรู้ IP/hostname ก่อน start
# 2. Cookie distribution: ต้องใช้ secret ร่วมกัน
# 3. EPMD: port 4369 ต้องเปิด (หรือใช้ libcluster)
# 4. Distribution port: ต้องเปิด range (4370-9100)
# 5. Rolling updates: ต้อง graceful drain

# Solutions:
# - libcluster: automatic node discovery
# - erlang-node-discovery: Kubernetes-native
# - disable distribution (single node deployments)
```

---

## Dockerfile สำหรับ Production

```dockerfile
# Dockerfile: multi-stage production build

# Stage 1: Build environment
FROM erlang:26.2.5-alpine AS builder

# Install build tools
RUN apk add --no-cache \
    build-base \
    git \
    openssl-dev \
    ncurses-dev

WORKDIR /app

# Cache dependencies separately
COPY rebar.config rebar.lock ./
RUN rebar3 get-deps 2>/dev/null || true

# Build application
COPY . .
RUN rebar3 as prod release

# Stage 2: Minimal runtime
FROM alpine:3.19 AS runtime

# Only runtime libraries needed
RUN apk add --no-cache \
    libstdc++ \
    libgcc \
    openssl \
    ncurses-libs \
    ca-certificates \
    tini  # PID 1 signal handling

# Security: non-root user
RUN addgroup -S -g 1000 erlang && \
    adduser -S -u 1000 -G erlang erlang

# Application directories
RUN mkdir -p /app /var/log/app /tmp/app && \
    chown -R erlang:erlang /app /var/log/app /tmp/app

WORKDIR /app

# Copy release from builder
COPY --from=builder --chown=erlang:erlang /app/_build/prod/rel/my_app ./

USER erlang

# Health check
HEALTHCHECK \
    --interval=30s \
    --timeout=10s \
    --start-period=15s \
    --retries=3 \
    CMD wget -qO- http://localhost:9090/health || exit 1

# Ports: HTTP, metrics, distribution
EXPOSE 8080 9090 4369 4370-9100

# Use tini as PID 1 for proper signal handling
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["bin/my_app", "foreground"]
```

```bash
# Build and push
docker build -t registry.example.com/my-app:1.0.0 .
docker push registry.example.com/my-app:1.0.0

# Multi-arch build
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    -t registry.example.com/my-app:1.0.0 \
    --push .
```

---

## Kubernetes Manifests

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-erlang-app
  namespace: production
  labels:
    app: my-erlang-app
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-erlang-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # zero-downtime
  template:
    metadata:
      labels:
        app: my-erlang-app
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: erlang-app
      terminationGracePeriodSeconds: 60
      
      containers:
      - name: app
        image: registry.example.com/my-app:1.0.0
        imagePullPolicy: Always
        
        ports:
        - name: http
          containerPort: 8080
        - name: metrics
          containerPort: 9090
        - name: epmd
          containerPort: 4369
          
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: ERLANG_COOKIE
          valueFrom:
            secretKeyRef:
              name: erlang-secrets
              key: cookie
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: password
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: db_host
              
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
            
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 9090
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
          
        livenessProbe:
          httpGet:
            path: /health/live
            port: 9090
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
          
        lifecycle:
          preStop:
            exec:
              # Graceful shutdown: drain connections
              command: ["/bin/sh", "-c", "bin/my_app rpc 'my_app:drain(30000)'"]

      # Erlang cookie secret
      volumes:
      - name: cookie
        secret:
          secretName: erlang-secrets
```

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-erlang-app
  namespace: production
spec:
  selector:
    app: my-erlang-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: metrics
    port: 9090
    targetPort: 9090
  type: ClusterIP

---
# hpa.yaml — Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-erlang-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-erlang-app
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: erlang_process_count
      target:
        type: AverageValue
        averageValue: "100000"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
```

---

## Distributed Erlang บน Kubernetes

```erlang
%% libcluster: automatic node discovery via Kubernetes API
%% deps: {libcluster, "3.3.3"}

%% sys.config
[{libcluster, [
    {topologies, [
        {k8s_dns, [
            {strategy, Cluster.Strategy.Kubernetes.DNS},
            {config, [
                %% Service name for DNS SRV lookup
                {service, "my-app-headless"},
                {application_name, "my_app"}
            ]}
        ]}
    ]}
]}].

%% Kubernetes headless service for DNS discovery
%%   apiVersion: v1
%%   kind: Service
%%   metadata:
%%     name: my-app-headless
%%   spec:
%%     clusterIP: None  # headless!
%%     selector:
%%       app: my-erlang-app
%%     ports:
%%     - name: epmd
%%       port: 4369

%% vm.args สำหรับ Kubernetes
%% -name ${POD_NAME}@${POD_IP}
%% -setcookie ${ERLANG_COOKIE}
%% +K true
%% +P 5000000

%% Node name from environment
start_distributed() ->
    PodName = os:getenv("POD_NAME"),
    PodIp = os:getenv("POD_IP"),
    NodeName = list_to_atom(PodName ++ "@" ++ PodIp),
    net_kernel:start([NodeName, longnames]),
    Cookie = list_to_atom(os:getenv("ERLANG_COOKIE")),
    erlang:set_cookie(node(), Cookie).

%% Monitor cluster membership
-module(cluster_monitor).
-behaviour(gen_server).

init(_) ->
    net_kernel:monitor_nodes(true),
    {ok, #{nodes => nodes()}}.

handle_info({nodeup, Node}, State) ->
    logger:info("Node joined", #{node => Node}),
    prometheus_gauge:inc(cluster_nodes),
    {noreply, State#{nodes := [Node | maps:get(nodes, State)]}};

handle_info({nodedown, Node}, State) ->
    logger:warning("Node left", #{node => Node}),
    prometheus_gauge:dec(cluster_nodes),
    Nodes = lists:delete(Node, maps:get(nodes, State)),
    {noreply, State#{nodes := Nodes}}.
```

---

## Helm Charts

```yaml
# charts/my-erlang-app/Chart.yaml
apiVersion: v2
name: my-erlang-app
description: Erlang OTP application
type: application
version: 1.0.0
appVersion: "1.0.0"
```

```yaml
# charts/my-erlang-app/values.yaml
replicaCount: 3

image:
  repository: registry.example.com/my-app
  pullPolicy: Always
  tag: "1.0.0"

service:
  type: ClusterIP
  port: 80
  metricsPort: 9090

resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "1Gi"
    cpu: "1000m"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

erlang:
  cookie: ""  # override in production!
  distribution: true

database:
  host: "postgres-service"
  port: 5432
  name: "myapp"

ingress:
  enabled: true
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
  - host: api.example.com
    paths:
    - path: /
      pathType: Prefix
  tls:
  - secretName: api-tls
    hosts:
    - api.example.com
```

```bash
# Deploy with Helm
helm install my-app ./charts/my-erlang-app \
    --namespace production \
    --set erlang.cookie=$(openssl rand -base64 32) \
    --set image.tag=1.0.0 \
    --values production-values.yaml

# Upgrade
helm upgrade my-app ./charts/my-erlang-app \
    --namespace production \
    --set image.tag=1.1.0

# Rollback
helm rollback my-app 1 --namespace production
```

---

## ตัวอย่างจริง: Production Deployment

```bash
#!/bin/bash
# deploy.sh: production deployment script

APP="my-erlang-app"
NAMESPACE="production"
NEW_TAG=$1
OLD_TAG=$(kubectl get deployment $APP -n $NAMESPACE \
    -o jsonpath='{.spec.template.spec.containers[0].image}' | cut -d: -f2)

echo "Deploying $APP: $OLD_TAG -> $NEW_TAG"

# 1. Build and push new image
docker buildx build --platform linux/amd64,linux/arm64 \
    -t registry.example.com/$APP:$NEW_TAG \
    --push .

# 2. Update deployment (rolling update)
kubectl set image deployment/$APP \
    app=registry.example.com/$APP:$NEW_TAG \
    -n $NAMESPACE

# 3. Wait for rollout
kubectl rollout status deployment/$APP -n $NAMESPACE --timeout=5m

# 4. Verify health
READY=$(kubectl get deployment $APP -n $NAMESPACE \
    -o jsonpath='{.status.readyReplicas}')
DESIRED=$(kubectl get deployment $APP -n $NAMESPACE \
    -o jsonpath='{.spec.replicas}')

if [ "$READY" -eq "$DESIRED" ]; then
    echo "Deployment successful: $READY/$DESIRED pods ready"
else
    echo "Deployment failed: only $READY/$DESIRED pods ready, rolling back"
    kubectl rollout undo deployment/$APP -n $NAMESPACE
    exit 1
fi
```

```erlang
%% Graceful drain on SIGTERM
-module(my_app).
-export([drain/1]).

drain(TimeoutMs) ->
    logger:info("Starting graceful shutdown"),
    %% Stop accepting new connections
    cowboy:stop_listener(http_listener),
    %% Drain existing connections
    drain_connections(TimeoutMs),
    %% Flush events
    event_bus:flush(),
    logger:info("Graceful shutdown complete").

drain_connections(Timeout) ->
    Start = erlang:monotonic_time(millisecond),
    wait_for_drain(Start, Timeout).

wait_for_drain(Start, Timeout) ->
    Active = active_connections(),
    Elapsed = erlang:monotonic_time(millisecond) - Start,
    if
        Active =:= 0 -> ok;
        Elapsed >= Timeout ->
            logger:warning("Drain timeout", #{active => Active});
        true ->
            timer:sleep(200),
            wait_for_drain(Start, Timeout)
    end.

active_connections() ->
    case ets:info(connections, size) of
        undefined -> 0;
        N -> N
    end.
```

---

## สรุป Part 49

| Component | Purpose |
|-----------|---------|
| Multi-stage Dockerfile | Small production images |
| tini | Proper PID 1 signal handling |
| Readiness probe | Remove from LB before ready |
| Liveness probe | Restart crashed pods |
| preStop hook | Graceful connection drain |
| HPA | Auto-scale on CPU/metrics |
| Headless service | Erlang node discovery |
| Helm | Parameterized deployments |
| libcluster | Automatic cluster formation |

---

*[← Part 48: Microservices](part_48_microservices.md) | [Part 50: CI/CD Pipeline →](part_50_cicd.md)*
