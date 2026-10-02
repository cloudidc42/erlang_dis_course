# Part 73: Cloud Deployment — AWS, Docker, Kubernetes

## สารบัญ
1. [Docker Container](#docker)
2. [AWS EC2/ECS Deployment](#aws)
3. [Kubernetes on EKS/GKE](#kubernetes)
4. [Health Checks และ Graceful Shutdown](#health)
5. [Configuration Management](#config)
6. [ตัวอย่างจริง: Production Deployment Pipeline](#pipeline)

---

## Docker Container

```dockerfile
# Dockerfile — multi-stage build

# Stage 1: Build
FROM erlang:26-alpine AS builder

WORKDIR /build

# Install build tools
RUN apk add --no-cache git make gcc g++ musl-dev

# Copy dependency files first (cache layer)
COPY rebar.config rebar.lock ./
RUN rebar3 deps

# Copy source and build
COPY . .
RUN rebar3 as prod release

# Stage 2: Runtime (minimal image)
FROM alpine:3.19

# Install only runtime deps
RUN apk add --no-cache libstdc++ openssl ncurses-libs

WORKDIR /app

# Copy release from builder
COPY --from=builder /build/_build/prod/rel/my_app ./

# Non-root user
RUN addgroup -S erlang && adduser -S erlang -G erlang
RUN chown -R erlang:erlang /app
USER erlang

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD ./bin/my_app rpc "health_check:status()" | grep ok

EXPOSE 8080

ENTRYPOINT ["./bin/my_app"]
CMD ["foreground"]
```

```yaml
# docker-compose.yml for local development

version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - ERLANG_COOKIE=local_dev_cookie
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=myapp
      - DB_USER=postgres
      - DB_PASSWORD=postgres
      - REDIS_HOST=redis
      - NODE_NAME=app@app
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app_net

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./priv/sql/schema.sql:/docker-entrypoint-initdb.d/schema.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - app_net

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
    networks:
      - app_net

volumes:
  pgdata:

networks:
  app_net:
    driver: bridge
```

---

## AWS ECS Deployment

```json
// ecs-task-definition.json
{
  "family": "my-erlang-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "my-erlang-app",
      "image": "123456789.dkr.ecr.ap-southeast-1.amazonaws.com/my-erlang-app:latest",
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {"name": "NODE_NAME", "value": "app@127.0.0.1"}
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:123456789:secret:db-password"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-erlang-app",
          "awslogs-region": "ap-southeast-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      },
      "stopTimeout": 30,
      "ulimits": [
        {
          "name": "nofile",
          "softLimit": 65536,
          "hardLimit": 65536
        }
      ]
    }
  ]
}
```

```bash
# deploy.sh — AWS ECS deployment script

#!/bin/bash
set -e

AWS_REGION="ap-southeast-1"
ECR_REPO="123456789.dkr.ecr.${AWS_REGION}.amazonaws.com/my-erlang-app"
ECS_CLUSTER="production"
ECS_SERVICE="my-erlang-app"

# Build and push Docker image
docker build -t my-erlang-app .
docker tag my-erlang-app:latest ${ECR_REPO}:${GITHUB_SHA}
docker tag my-erlang-app:latest ${ECR_REPO}:latest

aws ecr get-login-password --region ${AWS_REGION} | \
    docker login --username AWS --password-stdin ${ECR_REPO}

docker push ${ECR_REPO}:${GITHUB_SHA}
docker push ${ECR_REPO}:latest

# Update ECS service
aws ecs update-service \
    --region ${AWS_REGION} \
    --cluster ${ECS_CLUSTER} \
    --service ${ECS_SERVICE} \
    --force-new-deployment

# Wait for deployment to complete
aws ecs wait services-stable \
    --region ${AWS_REGION} \
    --cluster ${ECS_CLUSTER} \
    --services ${ECS_SERVICE}

echo "Deployment complete!"
```

---

## Kubernetes Deployment

```yaml
# k8s/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: erlang-app
  namespace: production
  labels:
    app: erlang-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: erlang-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: erlang-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: erlang-app
      
      terminationGracePeriodSeconds: 60
      
      containers:
        - name: erlang-app
          image: myregistry/erlang-app:latest
          imagePullPolicy: Always
          
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 9090
              name: metrics
          
          env:
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: NODE_NAME
              value: "app@$(POD_IP)"
            - name: ERLANG_COOKIE
              valueFrom:
                secretKeyRef:
                  name: erlang-secrets
                  key: erlang-cookie
          
          envFrom:
            - configMapRef:
                name: erlang-app-config
            - secretRef:
                name: erlang-app-secrets
          
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "2Gi"
          
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
          
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
          
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
          
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      
      volumes:
        - name: tmp
          emptyDir: {}

---
apiVersion: v1
kind: Service
metadata:
  name: erlang-app
  namespace: production
spec:
  selector:
    app: erlang-app
  ports:
    - name: http
      port: 80
      targetPort: 8080
  type: ClusterIP

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: erlang-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: erlang-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

---

## Health Checks และ Graceful Shutdown

```erlang
%% health_check.erl — comprehensive health check endpoint

-module(health_handler).
-behaviour(cowboy_handler).
-export([init/2]).

init(Req, State) ->
    Path = cowboy_req:path(Req),
    handle_path(Path, Req, State).

%% Kubernetes liveness probe — is the process alive?
handle_path(<<"/health/live">>, Req, State) ->
    Reply = cowboy_req:reply(200, #{<<"content-type">> => <<"application/json">>},
                             jiffy:encode(#{status => <<"ok">>}), Req),
    {ok, Reply, State};

%% Kubernetes readiness probe — is the app ready for traffic?
handle_path(<<"/health/ready">>, Req, State) ->
    Checks = [
        {db, check_database()},
        {redis, check_redis()},
        {pool, check_pool()}
    ],
    
    case lists:all(fun({_, ok}) -> true; (_) -> false end, Checks) of
        true ->
            Body = jiffy:encode(#{status => <<"ready">>, checks => maps:from_list(Checks)}),
            Reply = cowboy_req:reply(200, #{<<"content-type">> => <<"application/json">>}, Body, Req),
            {ok, Reply, State};
        false ->
            FailedChecks = [{K, V} || {K, V} <- Checks, V =/= ok],
            Body = jiffy:encode(#{status => <<"not_ready">>, failed => maps:from_list(FailedChecks)}),
            Reply = cowboy_req:reply(503, #{<<"content-type">> => <<"application/json">>}, Body, Req),
            {ok, Reply, State}
    end;

%% Detailed health metrics
handle_path(<<"/health">>, Req, State) ->
    Health = #{
        status => <<"ok">>,
        node => node(),
        uptime_seconds => uptime_seconds(),
        memory_mb => erlang:memory(total) div (1024 * 1024),
        processes => erlang:system_info(process_count),
        schedulers => erlang:system_info(schedulers_online)
    },
    Reply = cowboy_req:reply(200, #{<<"content-type">> => <<"application/json">>},
                             jiffy:encode(Health), Req),
    {ok, Reply, State}.

check_database() ->
    case catch epgsql:squery(db_conn(), "SELECT 1") of
        {ok, _, _} -> ok;
        _ -> {error, db_unreachable}
    end.

check_redis() ->
    case eredis:q(redis_conn(), ["PING"]) of
        {ok, <<"PONG">>} -> ok;
        _ -> {error, redis_unreachable}
    end.

check_pool() ->
    case poolboy:status(db_pool) of
        {State, _, _, Workers} when State =:= ready orelse Workers > 0 -> ok;
        _ -> {error, pool_exhausted}
    end.

uptime_seconds() ->
    {TotalMs, _} = erlang:statistics(wall_clock),
    TotalMs div 1000.

%% Graceful shutdown handler
-module(shutdown_handler).
-export([setup/0, shutdown/0]).

setup() ->
    %% Trap signals for graceful shutdown
    process_flag(trap_exit, true),
    
    %% Handle SIGTERM from Kubernetes
    os:set_signal(sigterm, handle),
    os:set_signal(sigint, handle).

handle(sigterm) ->
    logger:info("Received SIGTERM, starting graceful shutdown"),
    shutdown();
handle(sigint) ->
    logger:info("Received SIGINT, starting graceful shutdown"),
    shutdown().

shutdown() ->
    %% 1. Stop accepting new connections
    cowboy:stop_listener(http),
    
    %% 2. Wait for in-flight requests
    wait_for_requests(5000),
    
    %% 3. Flush queues
    flush_message_queues(),
    
    %% 4. Close DB connections
    db_pool:stop(),
    
    %% 5. Stop application
    application:stop(my_app),
    
    %% 6. Halt
    init:stop().

wait_for_requests(TimeoutMs) ->
    case active_request_count() of
        0 -> ok;
        N when TimeoutMs > 0 ->
            logger:info("Waiting for ~p active requests", [N]),
            timer:sleep(500),
            wait_for_requests(TimeoutMs - 500);
        N ->
            logger:warning("Timeout waiting for ~p requests to complete", [N])
    end.
```

---

## Configuration Management

```erlang
%% sys.config — Erlang release configuration

[
    {my_app, [
        {http_port, 8080},
        {db_pool_size, 20},
        %% Environment-specific overrides via env vars
        {db_host, {system_env, "DB_HOST", "localhost"}},
        {db_port, {system_env_int, "DB_PORT", 5432}},
        {db_name, {system_env, "DB_NAME", "myapp"}}
    ]},
    
    {kernel, [
        {logger_level, info},
        {logger, [
            {handler, default, logger_std_h, #{
                formatter => {logger_formatter, #{
                    template => [time, " ", level, " ", msg, "\n"]
                }}
            }}
        ]}
    ]}
].

%% config.erl — helper for reading configuration
-module(config).
-export([get/2, get/3]).

get(Key, Default) ->
    application:get_env(my_app, Key, Default).

get(App, Key, Default) ->
    application:get_env(App, Key, Default).

%% Read from environment variable with fallback
env(EnvVar, Default) ->
    case os:getenv(EnvVar) of
        false -> Default;
        "" -> Default;
        Value -> Value
    end.

env_int(EnvVar, Default) ->
    case os:getenv(EnvVar) of
        false -> Default;
        Value -> list_to_integer(Value)
    end.

%% vm.args — BEAM VM configuration

%% -name app@${POD_IP}          -- node name from environment
%% -setcookie ${ERLANG_COOKIE}  -- cluster cookie
%% +P 2000000                   -- max processes
%% +Q 65536                     -- max ports
%% +K true                      -- kernel poll
%% +A 16                        -- async thread pool size
%% -env ERL_MAX_PORTS 65536
%% -env ERL_FULLSWEEP_AFTER 0   -- aggressive GC for low latency
%% +Muacnl 0                    -- ERTS allocator config
%% +Bi                          -- ignore ^C (handle via signal handler)
```

---

## ตัวอย่างจริง: CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml

name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Erlang
        uses: erlef/setup-beam@v1
        with:
          otp-version: '26'
          rebar3-version: '3.22'
      
      - name: Cache dependencies
        uses: actions/cache@v3
        with:
          path: _build
          key: ${{ runner.os }}-rebar3-${{ hashFiles('rebar.lock') }}
      
      - name: Run tests
        run: |
          rebar3 eunit
          rebar3 ct
          rebar3 dialyzer
      
      - name: Check code coverage
        run: |
          rebar3 cover
          # Fail if coverage < 80%

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-1
      
      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Build and push Docker image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/my-erlang-app:$IMAGE_TAG .
          docker push $ECR_REGISTRY/my-erlang-app:$IMAGE_TAG
          
          # Tag as latest
          docker tag $ECR_REGISTRY/my-erlang-app:$IMAGE_TAG \
                     $ECR_REGISTRY/my-erlang-app:latest
          docker push $ECR_REGISTRY/my-erlang-app:latest
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Update ECS service
        run: |
          aws ecs update-service \
            --cluster production \
            --service my-erlang-app \
            --force-new-deployment \
            --region ap-southeast-1
          
          aws ecs wait services-stable \
            --cluster production \
            --services my-erlang-app \
            --region ap-southeast-1
```

---

## สรุป Part 73

| Topic | Tool/Approach |
|-------|--------------|
| Containerize | Docker multi-stage |
| AWS | ECS Fargate + ECR |
| Kubernetes | Deployment + HPA |
| Health check | `/health/live`, `/health/ready` |
| Graceful stop | `prep_stop/1` + SIGTERM handler |
| Config | sys.config + env vars |
| CI/CD | GitHub Actions |

---

*[← Part 72: Mnesia Advanced](part_72_mnesia_advanced.md) | [Part 74: Game Server Patterns →](part_74_game_server.md)*
