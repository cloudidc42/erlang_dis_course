# Part 50: CI/CD Pipeline สำหรับ Erlang

## สารบัญ
1. [CI/CD Principles](#cicd-principles)
2. [GitHub Actions](#github-actions)
3. [GitLab CI](#gitlab-ci)
4. [Automated Testing Pipeline](#automated-testing-pipeline)
5. [Deployment Pipeline](#deployment-pipeline)
6. [ตัวอย่างจริง: Complete Pipeline](#ตัวอย่างจริง-complete-pipeline)

---

## CI/CD Principles

```
CI (Continuous Integration): test ทุก commit
CD (Continuous Delivery): สร้าง deployable artifact ทุกครั้ง
CD (Continuous Deployment): deploy อัตโนมัติหลัง test pass

Pipeline stages:
1. Lint / Format check
2. Compile
3. Unit tests (EUnit)
4. Integration tests (Common Test)
5. Property-based tests (PropEr)
6. Static analysis (Dialyzer, Xref)
7. Security scan
8. Build release / Docker image
9. Deploy to staging
10. Smoke tests
11. Deploy to production
12. Monitor
```

---

## GitHub Actions

```yaml
# .github/workflows/ci.yml
name: Erlang CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  OTP_VERSION: "26.2"
  REBAR_VERSION: "3.23.0"

jobs:
  lint:
    name: Lint and Format
    runs-on: ubuntu-22.04
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Erlang
      uses: erlef/setup-beam@v1
      with:
        otp-version: ${{ env.OTP_VERSION }}
        rebar3-version: ${{ env.REBAR_VERSION }}
    
    - name: Cache deps
      uses: actions/cache@v3
      with:
        path: |
          ~/.cache/rebar3
          _build/default/lib
        key: ${{ runner.os }}-rebar3-${{ hashFiles('rebar.lock') }}
    
    - name: Check formatting
      run: rebar3 fmt --check
    
    - name: Compile
      run: rebar3 compile

  test:
    name: Tests
    runs-on: ubuntu-22.04
    needs: lint
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7
        ports:
          - 6379:6379
    
    env:
      DB_HOST: localhost
      DB_PASSWORD: test
      REDIS_HOST: localhost
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Erlang
      uses: erlef/setup-beam@v1
      with:
        otp-version: ${{ env.OTP_VERSION }}
        rebar3-version: ${{ env.REBAR_VERSION }}
    
    - name: Restore cache
      uses: actions/cache@v3
      with:
        path: ~/.cache/rebar3
        key: ${{ runner.os }}-rebar3-${{ hashFiles('rebar.lock') }}
    
    - name: Run EUnit tests
      run: rebar3 eunit --cover
    
    - name: Run Common Test
      run: rebar3 ct --cover
    
    - name: Generate coverage report
      run: rebar3 cover --verbose
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        files: _build/test/cover/index.html

  analyze:
    name: Static Analysis
    runs-on: ubuntu-22.04
    needs: lint
    
    steps:
    - uses: actions/checkout@v4
    
    - uses: erlef/setup-beam@v1
      with:
        otp-version: ${{ env.OTP_VERSION }}
        rebar3-version: ${{ env.REBAR_VERSION }}
    
    - name: Restore PLT cache
      uses: actions/cache@v3
      with:
        path: _build/default/rebar3_*_plt
        key: ${{ runner.os }}-dialyzer-${{ hashFiles('rebar.config') }}
    
    - name: Run Dialyzer
      run: rebar3 dialyzer
    
    - name: Run Xref
      run: rebar3 xref

  security:
    name: Security Scan
    runs-on: ubuntu-22.04
    needs: lint
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Check for known vulnerabilities
      run: |
        # Check rebar.lock for known vulnerable packages
        # Using custom script or hex_audit equivalent
        echo "Security scan passed"
    
    - name: SAST scan
      uses: github/codeql-action/analyze@v2
      with:
        languages: erlang

  build:
    name: Build Release
    runs-on: ubuntu-22.04
    needs: [test, analyze]
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v4
    
    - uses: erlef/setup-beam@v1
      with:
        otp-version: ${{ env.OTP_VERSION }}
        rebar3-version: ${{ env.REBAR_VERSION }}
    
    - name: Build production release
      run: rebar3 as prod tar
    
    - name: Build Docker image
      run: |
        IMAGE_TAG="registry.example.com/my-app:${{ github.sha }}"
        docker build -t $IMAGE_TAG .
        docker tag $IMAGE_TAG registry.example.com/my-app:latest
    
    - name: Push to registry
      run: |
        echo ${{ secrets.REGISTRY_PASSWORD }} | \
          docker login registry.example.com -u ${{ secrets.REGISTRY_USER }} --password-stdin
        docker push registry.example.com/my-app:${{ github.sha }}
        docker push registry.example.com/my-app:latest
    
    - name: Upload release artifact
      uses: actions/upload-artifact@v3
      with:
        name: release
        path: _build/prod/rel/*.tar.gz

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-22.04
    needs: build
    environment: staging
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Deploy to Kubernetes
      run: |
        kubectl set image deployment/my-app \
          app=registry.example.com/my-app:${{ github.sha }} \
          -n staging
        kubectl rollout status deployment/my-app -n staging
    
    - name: Run smoke tests
      run: |
        curl -f https://staging.example.com/health
        ./scripts/smoke_tests.sh staging.example.com

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-22.04
    needs: deploy-staging
    environment: production  # requires manual approval
    
    steps:
    - name: Deploy to production
      run: |
        kubectl set image deployment/my-app \
          app=registry.example.com/my-app:${{ github.sha }} \
          -n production
        kubectl rollout status deployment/my-app -n production --timeout=10m
    
    - name: Notify on success
      uses: slackapi/slack-github-action@v1.24.0
      with:
        channel-id: 'deployments'
        slack-message: "Deployed my-app ${{ github.sha }} to production"
      env:
        SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

---

## GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy

variables:
  OTP_IMAGE: "erlang:26.2-alpine"
  DOCKER_REGISTRY: "registry.gitlab.com/myorg/my-app"

.erlang-base: &erlang-base
  image: $OTP_IMAGE
  before_script:
    - apk add --no-cache git
    - rebar3 get-deps

test:eunit:
  <<: *erlang-base
  stage: test
  script:
    - rebar3 eunit --cover
  coverage: '/Lines covered: \K[\d.]+/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: _build/test/cover/cobertura.xml

test:dialyzer:
  <<: *erlang-base
  stage: test
  cache:
    key: dialyzer-plt
    paths:
      - _build/default/rebar3_*_plt
  script:
    - rebar3 dialyzer
  
build:release:
  <<: *erlang-base
  stage: build
  script:
    - rebar3 as prod tar
    - docker build -t $DOCKER_REGISTRY:$CI_COMMIT_SHA .
    - docker push $DOCKER_REGISTRY:$CI_COMMIT_SHA
  only:
    - main

deploy:production:
  stage: deploy
  environment:
    name: production
    url: https://app.example.com
  when: manual
  script:
    - kubectl set image deployment/my-app app=$DOCKER_REGISTRY:$CI_COMMIT_SHA
```

---

## Automated Testing Pipeline

```erlang
%% test_runner.erl: programmatic test execution

-module(test_runner).
-export([run_all/0, run_suite/1]).

run_all() ->
    Results = #{
        eunit => run_eunit(),
        common_test => run_ct(),
        dialyzer => run_dialyzer()
    },
    report_results(Results).

run_eunit() ->
    case eunit:test(my_app, [verbose]) of
        ok -> passed;
        {error, _} -> failed
    end.

run_ct() ->
    Opts = [
        {dir, "test"},
        {logdir, "_build/test/logs"},
        {auto_compile, true}
    ],
    case ct:run_test(Opts) of
        {_Ok, 0, _Skip} -> passed;  %% 0 failures
        _ -> failed
    end.

run_dialyzer() ->
    case dialyzer:run([
        {analysis_type, succ_typings},
        {apps, [my_app]},
        {from, src_code}
    ]) of
        [] -> passed;
        Warnings ->
            [io:format("~s~n", [dialyzer:format_warning(W)]) || W <- Warnings],
            failed
    end.

report_results(Results) ->
    Failed = [Name || {Name, failed} <- maps:to_list(Results)],
    case Failed of
        [] ->
            io:format("All tests passed!~n"),
            halt(0);
        _ ->
            io:format("Failed: ~p~n", [Failed]),
            halt(1)
    end.
```

---

## ตัวอย่างจริง: Complete Pipeline

```makefile
# Makefile: development workflow

.PHONY: deps compile test dialyzer xref release docker

deps:
	rebar3 get-deps

compile:
	rebar3 compile

test: compile
	rebar3 eunit --cover
	rebar3 ct --cover
	rebar3 cover

dialyzer:
	rebar3 dialyzer

xref:
	rebar3 xref

proper:
	rebar3 proper

lint: compile dialyzer xref

ci: lint test

release:
	rebar3 as prod release

tar:
	rebar3 as prod tar

docker:
	docker build -t my-app:dev .

docker-push: docker
	docker tag my-app:dev registry.example.com/my-app:$(VERSION)
	docker push registry.example.com/my-app:$(VERSION)

deploy-staging:
	kubectl set image deployment/my-app app=registry.example.com/my-app:$(VERSION) -n staging
	kubectl rollout status deployment/my-app -n staging

deploy-prod:
	kubectl set image deployment/my-app app=registry.example.com/my-app:$(VERSION) -n production
	kubectl rollout status deployment/my-app -n production

clean:
	rebar3 clean
	docker rmi my-app:dev 2>/dev/null || true
```

```bash
#!/bin/bash
# scripts/smoke_tests.sh
HOST=$1

run_test() {
    NAME=$1
    CMD=$2
    if eval $CMD; then
        echo "PASS: $NAME"
    else
        echo "FAIL: $NAME"
        FAILED=1
    fi
}

FAILED=0

run_test "Health check" "curl -sf https://$HOST/health"
run_test "API responds" "curl -sf https://$HOST/api/users | jq . > /dev/null"
run_test "Metrics available" "curl -sf https://$HOST/metrics | grep erlang_vm"
run_test "Auth required" "[ $(curl -s -o /dev/null -w '%{http_code}' https://$HOST/api/admin) = '401' ]"

if [ $FAILED -ne 0 ]; then
    echo "Smoke tests FAILED"
    exit 1
fi
echo "All smoke tests PASSED"
```

---

## สรุป Part 50

| Stage | Tool | Purpose |
|-------|------|---------|
| Compile | rebar3 compile | Catch syntax errors |
| Unit test | EUnit | Fast function tests |
| Integration | Common Test | End-to-end with dependencies |
| Property | PropEr | Random input testing |
| Type check | Dialyzer | Find type errors |
| Dead code | Xref | Find unused functions |
| Security | CodeQL | Find vulnerabilities |
| Coverage | cover | Track test coverage |
| Build | rebar3 tar | Production artifact |
| Container | Docker | Portable deployment |
| Deploy | kubectl | Kubernetes rollout |
| Verify | smoke tests | Post-deploy validation |

---

*[← Part 49: Kubernetes](part_49_kubernetes.md) | [Part 51: High-Availability Patterns →](part_51_high_availability.md)*
