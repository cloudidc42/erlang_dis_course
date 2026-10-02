# Part 29: Rebar3 — Build Tool

## สารบัญ
1. [Rebar3 คืออะไร](#rebar3-คืออะไร)
2. [Project Structure](#project-structure)
3. [rebar.config](#rebarconfig)
4. [Dependencies](#dependencies)
5. [Commands ที่ใช้บ่อย](#commands-ที่ใช้บ่อย)
6. [Releases](#releases)
7. [Profiles](#profiles)
8. [Custom Plugins](#custom-plugins)

---

## Rebar3 คืออะไร

```bash
# Rebar3 = build tool มาตรฐานสำหรับ Erlang
# ทำ: compile, test, release, dependency management, plugins

# ติดตั้ง
$ wget https://s3.amazonaws.com/rebar3/rebar3
$ chmod +x rebar3
$ mv rebar3 /usr/local/bin/

# หรือ build จาก source
$ git clone https://github.com/erlang/rebar3.git
$ cd rebar3 && ./bootstrap
$ cp rebar3 /usr/local/bin/

# สร้าง project ใหม่
$ rebar3 new app my_app      # OTP application
$ rebar3 new lib my_lib      # Library (no supervision tree)
$ rebar3 new release my_rel  # Release project
$ rebar3 new escript my_cli  # Standalone escript

# Project structure (app):
# my_app/
# ├── rebar.config
# ├── src/
# │   ├── my_app.app.src
# │   ├── my_app_app.erl   (application callback)
# │   └── my_app_sup.erl   (root supervisor)
# ├── include/             (header files .hrl)
# ├── test/                (test files)
# └── _build/              (build output, gitignore this)
```

---

## Project Structure

```erlang
%% Umbrella project (หลาย apps ใน repo เดียว)
%% my_project/
%% ├── rebar.config         (root config)
%% ├── apps/
%% │   ├── core/            (app 1)
%% │   │   ├── src/
%% │   │   └── test/
%% │   ├── api/             (app 2, depends on core)
%% │   │   ├── src/
%% │   │   └── test/
%% │   └── worker/          (app 3)
%% └── config/
%%     ├── sys.config
%%     └── vm.args

%% rebar.config สำหรับ umbrella:
{project_plugins, [rebar3_hex]}.

{profiles, [
    {test, [
        {deps, [{meck, "0.9.2"}]}
    ]}
]}.

%% rebar3 automatically finds apps in apps/ directory
```

---

## rebar.config

```erlang
%% ไฟล์: rebar.config

%% Erlang compiler options
{erl_opts, [
    debug_info,         %% required for many tools
    {parse_transform, lager_transform},  %% if using lager
    warn_unused_vars,
    warn_shadow_vars,
    warn_unused_import,
    {i, "include"}      %% include path
]}.

%% Dependencies
{deps, [
    {jsx, "3.1.0"},
    {hackney, "1.18.1"},
    {epgsql, "4.7.1"},
    {lager, {git, "https://github.com/erlang-lager/lager.git", {branch, "master"}}},
    {meck, "0.9.2", {pkg, meck}}  %% hex.pm package
]}.

%% Shell configuration (rebar3 shell)
{shell, [
    {config, "config/sys.config"},
    {apps, [my_app]}
]}.

%% EUnit options
{eunit_opts, [verbose, {report, {eunit_surefire, [{dir, "."}]}}]}.

%% Common Test options
{ct_opts, [
    {sys_config, "config/sys.config"},
    {logdir, "logs"}
]}.

%% Cover (code coverage)
{cover_enabled, true}.
{cover_opts, [verbose]}.

%% xref: cross reference analysis
{xref_checks, [
    undefined_function_calls,
    undefined_functions,
    locals_not_used,
    deprecated_function_calls
]}.

%% dialyzer: static type analysis
{dialyzer, [
    {warnings, [
        underspecs,
        unknown
    ]},
    {plt_apps, top_level_deps},
    {plt_extra_apps, [crypto]}
]}.

%% edoc: documentation
{edoc_opts, [
    {doclet, edoc_doclet},
    {new, true},
    {source_path, ["src"]}
]}.
```

---

## Dependencies

```erlang
%% Hex.pm packages (recommended)
{deps, [
    jsx,                    %% latest version
    {jsx, "3.1.0"},        %% exact version
    {jsx, "~> 3.1"},       %% ~> means >= 3.1 and < 4.0
    {jsx, ">= 3.0.0"}      %% >= 3.0.0
]}.

%% Git dependencies
{deps, [
    {my_lib, {git, "https://github.com/user/my_lib.git", {branch, "main"}}},
    {my_lib, {git, "https://github.com/user/my_lib.git", {tag, "v1.2.3"}}},
    {my_lib, {git, "https://github.com/user/my_lib.git", {ref, "abc1234"}}}
]}.

%% Path dependency (local development)
{deps, [
    {my_lib, {path, "../my_lib"}}
]}.

%% ล็อก dependency versions
$ rebar3 lock
%% สร้าง rebar.lock file - commit this file!

%% Update dependencies
$ rebar3 upgrade jsx           # update specific dep
$ rebar3 upgrade               # update all deps

%% Check outdated
$ rebar3 hex outdated

%% Fetch deps
$ rebar3 get-deps

%% Show dep tree
$ rebar3 tree
%% my_app─0.1.0 (project app)
%%    ├─ hackney─1.18.1 (hex package)
%%    │   ├─ certifi─2.9.0
%%    │   ├─ idna─6.1.1
%%    │   └─ metrics─1.0.3
%%    └─ jsx─3.1.0
```

---

## Commands ที่ใช้บ่อย

```bash
# Build
$ rebar3 compile              # compile project
$ rebar3 clean                # clean build artifacts

# Development
$ rebar3 shell                # start Erlang shell with app loaded
$ rebar3 shell --apps my_app  # start with specific apps
$ rebar3 repl                 # interactive REPL (if plugin installed)

# Testing
$ rebar3 eunit                # run EUnit tests
$ rebar3 ct                   # run Common Test
$ rebar3 cover                # generate coverage report
$ rebar3 eunit --module my_module_tests  # specific module

# Analysis
$ rebar3 dialyzer             # run type checking
$ rebar3 xref                 # cross-reference analysis
$ rebar3 lint                 # lint (if elvis plugin)

# Documentation
$ rebar3 edoc                 # generate docs

# Release
$ rebar3 release              # build release
$ rebar3 tar                  # build tarball for deployment

# Hex.pm (package manager)
$ rebar3 hex publish          # publish package
$ rebar3 hex search jsx       # search packages

# Escriptize (standalone executable)
$ rebar3 escriptize

# Update rebar3 itself
$ rebar3 local upgrade
```

---

## Releases

```erlang
%% rebar.config สำหรับ release
{relx, [
    %% Release definition
    {release, {my_app, "1.0.0"}, [
        my_app,
        sasl   %% System Application Support Libraries
    ]},

    %% dev mode: symlinks (faster iteration)
    {mode, dev},

    %% prod mode: copies (for deployment)
    %% {mode, prod},

    %% Config files
    {sys_config, "./config/sys.config"},
    {vm_args, "./config/vm.args"},

    %% Extended start script (more commands)
    {extended_start_script, true},

    %% Include ERTS (Erlang Runtime System) in release
    %% {include_erts, true},  %% default true in prod mode

    %% Overlay: copy extra files into release
    {overlay, [
        {mkdir, "log"},
        {copy, "priv/templates", "templates"},
        {template, "config/sys.config.tmpl", "releases/{{release_version}}/sys.config"}
    ]}
]}.

%% Build release
%% $ rebar3 release
%% Release built at _build/default/rel/my_app/

%% Start/stop commands
%% $ _build/default/rel/my_app/bin/my_app start
%% $ _build/default/rel/my_app/bin/my_app status
%% $ _build/default/rel/my_app/bin/my_app stop
%% $ _build/default/rel/my_app/bin/my_app restart
%% $ _build/default/rel/my_app/bin/my_app console  %% attached console
%% $ _build/default/rel/my_app/bin/my_app remote_console  %% connect to running node

%% Hot code upgrade
%% 1. Bump version in .app.src
%% 2. rebar3 release
%% 3. Create upgrade tarball:
%%    $ rebar3 relup -n my_app -v "1.0.1" -u "1.0.0"
%%    $ rebar3 tar
%% 4. Release handler upgrade
%%    release_handler:unpack_release("my_app_1.0.1")
%%    release_handler:install_release("1.0.1")
%%    release_handler:make_permanent("1.0.1")
```

---

## Profiles

```erlang
%% Profiles: ตั้งค่าต่างกันสำหรับ env ต่างๆ

{profiles, [
    %% Dev profile (default when rebar3 compile)
    {dev, [
        {erl_opts, [debug_info, {d, 'DEV'}]},
        {relx, [{mode, dev}]}
    ]},

    %% Test profile (used by rebar3 eunit/ct)
    {test, [
        {deps, [
            {meck, "0.9.2"},    %% mocking
            {proper, "1.4.0"},  %% property-based testing
            {eunit_formatters, "0.5.0"}
        ]},
        {erl_opts, [debug_info, {d, 'TEST'}]}
    ]},

    %% Production profile
    {prod, [
        {erl_opts, [
            no_debug_info,
            {d, 'PROD'},
            inline_list_funcs,
            deterministic
        ]},
        {relx, [
            {mode, prod},
            {include_erts, true},
            {include_src, false}
        ]}
    ]}
]}.

%% ใช้ profile:
%% $ rebar3 as prod release
%% $ rebar3 as test eunit
%% $ rebar3 as prod tar
```

---

## Custom Plugins

```erlang
%% rebar.config
{plugins, [
    rebar3_auto,      %% auto-compile on file change
    rebar3_hex,       %% publish to hex.pm
    rebar3_elvis,     %% Erlang lint
    covertool,        %% coverage in CI format
    rebar3_ex_doc     %% ExDoc-style documentation
]}.

%% Project plugins (รัน ณ build time)
{project_plugins, [
    rebar3_proper     %% proper integration
]}.

%% ตัวอย่าง elvis config (lint)
{elvis, [
    #{dirs => ["src", "test"],
      filter => "*.erl",
      ruleset => erl_files,
      rules => [
          {elvis_style, line_length, #{limit => 100}},
          {elvis_style, no_tabs, #{}},
          {elvis_style, no_trailing_whitespace, #{}}
      ]}
]}.

%% rebar3_auto: auto-reload on file change
%% $ rebar3 auto
%% -> watches src/, recompiles + reloads when .erl changes

%% ตัวอย่าง: สร้าง plugin เองง่ายๆ
-module(rebar3_greet).

-export([init/1]).

-spec init(rebar_state:t()) -> {ok, rebar_state:t()}.
init(State) ->
    Provider = providers:create([
        {name, greet},
        {module, rebar3_greet},
        {bare, true},
        {deps, [{default, compile}]},
        {example, "rebar3 greet"},
        {opts, []},
        {short_desc, "Greet the developer"},
        {desc, "Says hello!"}
    ]),
    {ok, rebar_state:add_provider(State, Provider)}.

-spec do(rebar_state:t()) -> {ok, rebar_state:t()} | {error, string()}.
do(State) ->
    io:format("Hello from rebar3!~n"),
    {ok, State}.

-spec format_error(any()) -> iolist().
format_error(Reason) ->
    io_lib:format("~p", [Reason]).
```

---

## สรุป Part 29: Rebar3 Commands

| Command | Action |
|---------|--------|
| `rebar3 compile` | Compile |
| `rebar3 shell` | Dev shell |
| `rebar3 eunit` | Unit tests |
| `rebar3 ct` | Integration tests |
| `rebar3 dialyzer` | Type check |
| `rebar3 release` | Build release |
| `rebar3 as prod release` | Prod release |

| File | Purpose |
|------|---------|
| `rebar.config` | Build configuration |
| `rebar.lock` | Locked dependency versions |
| `src/*.app.src` | Application metadata |
| `config/sys.config` | Runtime config |
| `config/vm.args` | VM arguments |

---

*[← Part 28: Application Behaviour](part_28_application.md) | [Part 30: Testing →](part_30_testing.md)*
