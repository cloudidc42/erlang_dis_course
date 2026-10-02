# Part 41: Release Management และ Hot Code Reloading

## สารบัญ
1. [Releases คืออะไร](#releases-คืออะไร)
2. [Build และ Deploy Release](#build-และ-deploy-release)
3. [Hot Code Reloading](#hot-code-reloading)
4. [Release Upgrade (relup)](#release-upgrade-relup)
5. [Docker Deployment](#docker-deployment)
6. [ตัวอย่างจริง: Zero-Downtime Deployment](#ตัวอย่างจริง-zero-downtime-deployment)

---

## Releases คืออะไร

```erlang
%% Release = self-contained bundle ที่รัน Erlang app ได้
%% มีทุกอย่าง: ERTS (Erlang runtime), modules, config, scripts
%% ไม่ต้องติดตั้ง Erlang ที่ target machine

%% โครงสร้าง release:
%% my_app_release/
%% ├── bin/
%% │   ├── my_app           (start/stop script)
%% │   └── my_app-1.0.0     (versioned script)
%% ├── erts-13.2/           (Erlang runtime)
%% ├── lib/
%% │   ├── my_app-1.0.0/    (our app)
%% │   ├── kernel-9.1/
%% │   ├── stdlib-5.0/
%% │   └── ...
%% └── releases/
%%     ├── 1.0.0/
%%     │   ├── sys.config
%%     │   ├── vm.args
%%     │   └── my_app.rel   (release descriptor)
%%     └── RELEASES         (installed releases)

%% rebar.config
{relx, [
    {release, {my_app, "1.0.0"}, [
        my_app,
        sasl,
        runtime_tools  %% for recon, debugging in prod
    ]},
    {sys_config, "./config/sys.config"},
    {vm_args, "./config/vm.args"},
    {extended_start_script, true}
]}.

%% config/vm.args
%% -name myapp@hostname
%% -setcookie my_cookie
%% +K true
%% +A 0
%% +P 10000000
%% -env ERL_MAX_ETS_TABLES 100000
```

---

## Build และ Deploy Release

```bash
# Build
$ rebar3 release              # dev build (symlinks)
$ rebar3 as prod release      # prod build (copies + stripped)
$ rebar3 as prod tar          # create .tar.gz for deployment

# Start/Stop commands (extended_start_script)
$ ./_build/prod/rel/my_app/bin/my_app start     # background daemon
$ ./_build/prod/rel/my_app/bin/my_app stop      # graceful stop
$ ./_build/prod/rel/my_app/bin/my_app restart   # restart
$ ./_build/prod/rel/my_app/bin/my_app console   # foreground with shell
$ ./_build/prod/rel/my_app/bin/my_app attach    # attach to running node
$ ./_build/prod/rel/my_app/bin/my_app remote_console  # remote shell

# Check status
$ ./_build/prod/rel/my_app/bin/my_app status
Node my_app@hostname is started. Status: started

# Deploy to server
$ scp _build/prod/rel/my_app/my_app-1.0.0.tar.gz server:/opt/releases/
$ ssh server
$ cd /opt/my_app
$ tar xzf /opt/releases/my_app-1.0.0.tar.gz
$ ./bin/my_app start
```

```erlang
%% sys.config สำหรับ production
[
    {my_app, [
        {http_port, 8080},
        {db_host, {system, "DB_HOST", "localhost"}},  %% from env var
        {db_password, {system, "DB_PASSWORD"}},
        {pool_size, 20},
        {log_level, info}
    ]},
    {kernel, [
        {logger, [
            {handler, default, logger_std_h, #{
                level => info,
                formatter => {logger_formatter, #{
                    template => [time, " ", level, " ", pid, " ", msg, "\n"]
                }},
                config => #{
                    type => {file, "/var/log/my_app/app.log"},
                    max_no_bytes => 52428800,  %% 50 MB
                    max_no_files => 10
                }
            }}
        ]}
    ]}
].
```

---

## Hot Code Reloading

```erlang
%% Hot code reload: เปลี่ยน code โดยไม่ต้อง stop server
%% Erlang สนับสนุน 2 versions ของ module พร้อมกัน

%% Compile และ load new code
compile:file("src/my_module.erl").
code:load_file(my_module).

%% หรือ purge old + load new
code:purge(my_module).   %% ลบ old code ถ้าไม่มี process ใช้
code:load_file(my_module).

%% soft purge: ล้มเหลวถ้ายังมี process ใช้ old code
code:soft_purge(my_module).  %% returns true/false

%% check ว่า process ยังใช้ old code อยู่ไหม
code:which(my_module).   %% path to current code
erlang:check_process_code(Pid, my_module).  %% true ถ้ายังใช้ old

%% รับ notification เมื่อ module loaded
code:add_path("/new/path").

%% Automatic reload ใน development
%% rebar3 auto plugin หรือ sync:go()

%% code:ensure_loaded
{module, Mod} = code:ensure_loaded(my_module).

%% Hot reload ด้วย release handler (production)
release_handler:install_release(NewVsn).

%% gen_server code_change/3 callback
code_change(OldVsn, OldState, Extra) ->
    %% Transform old state to new state format
    NewState = migrate_state(OldVsn, OldState, Extra),
    {ok, NewState}.

migrate_state("1.0.0", {old_state, V1, V2}, _) ->
    #{v1 => V1, v2 => V2};   %% migrate record to map
migrate_state(_, State, _) ->
    State.
```

---

## Release Upgrade (relup)

```erlang
%% relup: hot upgrade production release
%% ต้อง: appup file + relup file

%% ขั้นตอน:
%% 1. Bump version ใน .app.src: "1.0.0" -> "1.1.0"
%% 2. สร้าง appup file
%% 3. Build new release
%% 4. Generate relup
%% 5. Deploy and install

%% ไฟล์: src/my_app.appup
{
    "1.1.0",
    [
        {"1.0.0", [
            {load_module, my_module},         %% reload module
            {update, my_server, soft_purge, soft_purge, [], []} %% update gen_server
        ]}
    ],
    [
        {"1.0.0", [
            {load_module, my_module}
        ]}
    ]
}.

%% appup commands:
%% {load_module, Mod}                 - reload module
%% {update, Mod, Change, PrePurge, PostPurge, Mods}
%%   Change = soft_purge | {advanced, Extra}
%% {add_module, Mod}                  - add new module
%% {delete_module, Mod}               - remove module
%% {add_application, App}             - add app
%% {remove_application, App}          - remove app
%% {restart_application, App}         - restart app

%% Generate relup file
%% $ rebar3 relup -n my_app -v "1.1.0" -u "1.0.0"

%% Deploy upgrade
%% 1. Copy new release to server
%% $ tar xzf my_app-1.1.0.tar.gz -C releases/

%% 2. Unpack
%% $ ./bin/my_app eval "release_handler:unpack_release(\"my_app-1.1.0\")"

%% 3. Install (hot swap)
%% $ ./bin/my_app eval "release_handler:install_release(\"1.1.0\")"

%% 4. Make permanent
%% $ ./bin/my_app eval "release_handler:make_permanent(\"1.1.0\")"

%% 5. If issues, rollback
%% $ ./bin/my_app eval "release_handler:install_release(\"1.0.0\")"
```

---

## Docker Deployment

```dockerfile
# Multi-stage build สำหรับ Erlang release

# Stage 1: Build
FROM erlang:26-alpine AS builder

WORKDIR /app

# Copy deps first (for caching)
COPY rebar.config rebar.lock ./
RUN rebar3 get-deps

# Copy source and build
COPY . .
RUN rebar3 as prod release

# Stage 2: Runtime (minimal image)
FROM alpine:3.18

# Install runtime deps
RUN apk add --no-cache libstdc++ libgcc ncurses-libs

WORKDIR /app

# Copy release from builder
COPY --from=builder /app/_build/prod/rel/my_app ./

# Create log directory
RUN mkdir -p /var/log/my_app

# Non-root user
RUN addgroup -S erlang && adduser -S erlang -G erlang
USER erlang

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:9090/health || exit 1

EXPOSE 8080 9090

CMD ["bin/my_app", "foreground"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
      - "9090:9090"
    environment:
      - DB_HOST=postgres
      - DB_PASSWORD_FILE=/run/secrets/db_password
      - COOKIE=my_secret_cookie
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first  %% rolling update
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9090/health"]
      interval: 30s
      timeout: 10s
      retries: 3
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myapp
```

---

## ตัวอย่างจริง: Zero-Downtime Deployment

```bash
#!/bin/bash
# zero_downtime_deploy.sh

APP_NAME="my_app"
NEW_VERSION="$1"
RELEASE_DIR="/opt/$APP_NAME"

echo "Deploying $APP_NAME v$NEW_VERSION"

# 1. Upload new release
scp "${APP_NAME}-${NEW_VERSION}.tar.gz" server:/tmp/

# 2. On server: unpack
ssh server "
    mkdir -p $RELEASE_DIR/releases/$NEW_VERSION
    tar xzf /tmp/${APP_NAME}-${NEW_VERSION}.tar.gz \
        -C $RELEASE_DIR/releases/$NEW_VERSION
"

# 3. Hot upgrade (if running)
ssh server "$RELEASE_DIR/bin/$APP_NAME eval \"
    case release_handler:unpack_release(\\\"$APP_NAME-$NEW_VERSION\\\") of
        {ok, _} ->
            case release_handler:install_release(\\\"$NEW_VERSION\\\") of
                {ok, _, _} ->
                    release_handler:make_permanent(\\\"$NEW_VERSION\\\"),
                    io:format(\\\"Upgrade successful!~n\\\");
                {error, R} ->
                    io:format(\\\"Install failed: ~p~n\\\", [R])
            end;
        {error, R} ->
            io:format(\\\"Unpack failed: ~p~n\\\", [R])
    end
\""

echo "Deployment complete"
```

```erlang
%% Graceful shutdown: finish current requests before stopping

%% ใน application:stop/1
stop(State) ->
    %% Signal: stop accepting new requests
    my_app_api:set_accepting(false),
    %% Wait for in-flight requests (max 30s)
    drain_connections(30000),
    ok.

drain_connections(Timeout) ->
    Start = erlang:monotonic_time(millisecond),
    drain_loop(Start, Timeout).

drain_loop(Start, Timeout) ->
    Active = prometheus_gauge:value(active_requests),
    Elapsed = erlang:monotonic_time(millisecond) - Start,
    case Active =:= 0 orelse Elapsed >= Timeout of
        true ->
            io:format("Drained: ~p active, ~p ms~n", [Active, Elapsed]);
        false ->
            timer:sleep(100),
            drain_loop(Start, Timeout)
    end.
```

---

## สรุป Part 41

| Command | Effect |
|---------|--------|
| `rebar3 release` | Build release |
| `rebar3 as prod tar` | Build deployable tarball |
| `bin/app start` | Start daemon |
| `bin/app console` | Start with shell |
| `bin/app attach` | Attach to running node |
| `release_handler:install_release/1` | Hot upgrade |
| `release_handler:make_permanent/1` | Commit upgrade |
| `code:load_file/1` | Reload single module |

---

*[← Part 40: Observability](part_40_observability.md) | [Part 42: Security →](part_42_security.md)*
