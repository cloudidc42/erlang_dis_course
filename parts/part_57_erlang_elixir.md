# Part 57: Erlang และ Elixir — การทำงานร่วมกัน

## สารบัญ
1. [ความสัมพันธ์ Erlang/Elixir](#ความสัมพันธ์)
2. [เรียกใช้ Erlang จาก Elixir](#erlang-from-elixir)
3. [เรียกใช้ Elixir จาก Erlang](#elixir-from-erlang)
4. [แชร์ข้อมูล OTP Cluster](#otp-cluster)
5. [Mix Project กับ Erlang deps](#mix-erlang-deps)
6. [ตัวอย่างจริง: Hybrid Application](#hybrid-application)

---

## ความสัมพันธ์

```
Erlang และ Elixir ทำงานบน BEAM VM เหมือนกัน
- Bytecode เหมือนกัน (BEAM files .beam)
- Modules เรียกใช้ข้ามกันได้โดยตรง
- Processes ส่ง message ข้ามกันได้
- OTP behaviours แชร์กันได้
- Node clustering ทำงานร่วมกันได้

Elixir → Erlang: ใช้ :module_name syntax
Erlang → Elixir: ใช้ 'Elixir.Module' atom
```

---

## Erlang from Elixir

```elixir
# เรียกใช้ Erlang module ใน Elixir

# stdlib
:lists.sort([3, 1, 2])              # → [1, 2, 3]
:maps.get(:key, %{key: "val"})      # → "val"
:binary.split("a,b,c", ",")        # → ["a", "b,c"]
:crypto.strong_rand_bytes(32)       # → <<...>>

# OTP
{:ok, pid} = :gen_server.start_link(MyGenServer, [], [])
:supervisor.start_child(MySup, child_spec)

# File I/O
{:ok, fd} = :file.open("data.txt", [:read, :binary])
:file.read(fd, 4096)

# Network
{:ok, socket} = :gen_tcp.connect('localhost', 8080, [:binary])
:gen_tcp.send(socket, "hello")

# Mnesia
:mnesia.create_schema([node()])
:mnesia.start()
:mnesia.create_table(:users, [attributes: [:id, :name]])
:mnesia.transaction(fn ->
  :mnesia.write({:users, 1, "Alice"})
end)

# erlang BIFs
:erlang.system_info(:process_count)
:erlang.memory(:total)
:erlang.send_after(1000, self(), :tick)
```

---

## Elixir from Erlang

```erlang
%% เรียกใช้ Elixir module ใน Erlang
%% Elixir modules ถูก compile เป็น 'Elixir.ModuleName' atom

%% เรียก Elixir module
Result = 'Elixir.String':split(<<"hello world">>, <<" ">>).
%% → [<<"hello">>, <<"world">>]

Encoded = 'Elixir.Base':encode64(<<"binary data">>).
Decoded = 'Elixir.Base':decode64!(Encoded).

%% Jason (Elixir JSON library) from Erlang
Json = 'Elixir.Jason':encode!(#{name => <<"Alice">>, age => 30}).
Map = 'Elixir.Jason':decode!(Json).

%% Ecto from Erlang (rarely done, but possible)
%% 'Elixir.MyApp.Repo':get('Elixir.MyApp.User', 1)

%% Phoenix PubSub
'Elixir.Phoenix.PubSub':broadcast(
    'Elixir.MyApp.PubSub',
    <<"topic:room:1">>,
    #{event => <<"new_msg">>, data => <<"Hello">>}
).

%% Note: Elixir atoms like :ok, :error map to Erlang ok, error
%% Elixir structs are maps with __struct__ key:
%% %MyStruct{field: val} → #{__struct__ => 'Elixir.MyStruct', field => val}
```

---

## OTP Cluster

```erlang
%% Erlang + Elixir nodes ใน same cluster

%% erlang_app/src/erlang_worker.erl
-module(erlang_worker).
-behaviour(gen_server).
-export([start_link/0, compute/1]).

start_link() ->
    gen_server:start_link({global, erlang_worker}, ?MODULE, [], []).

compute(Data) ->
    gen_server:call({global, erlang_worker}, {compute, Data}).

handle_call({compute, Data}, _From, State) ->
    Result = heavy_computation(Data),
    {reply, Result, State}.

%% Elixir side: lib/elixir_app/erlang_client.ex
%% defmodule ElixirApp.ErlangClient do
%%   def compute(data) do
%%     :global.whereis_name(:erlang_worker)
%%     |> GenServer.call({:compute, data})
%%   end
%% end

%% Start mixed cluster
%% iex --name elixir@host --cookie mycookie -S mix
%% erl -name erlang@host -setcookie mycookie -s erlang_worker start

%% Connect nodes
connect_to_elixir() ->
    ElixirNode = 'myapp@hostname',
    case net_adm:ping(ElixirNode) of
        pong -> logger:info("Connected to Elixir node");
        pang -> logger:error("Cannot reach Elixir node")
    end.
```

---

## Mix Project กับ Erlang deps

```elixir
# mix.exs: mix project ที่ใช้ Erlang dependencies

defmodule MyApp.MixProject do
  use Mix.Project

  def project do
    [
      app: :my_app,
      version: "1.0.0",
      elixir: "~> 1.15",
      deps: deps()
    ]
  end

  defp deps do
    [
      # Elixir deps (normal)
      {:phoenix, "~> 1.7"},
      {:ecto_sql, "~> 3.10"},
      
      # Erlang deps from Hex (no wrapper needed)
      {:cowboy, "~> 2.10"},
      {:jsx, "~> 3.1"},
      {:poolboy, "~> 1.5"},
      
      # Pure Erlang library from GitHub
      {:mochiweb, git: "https://github.com/mochi/mochiweb", branch: "main"},
      
      # Local Erlang app
      {:my_erlang_lib, path: "../my_erlang_lib"}
    ]
  end
end
```

```erlang
%% rebar.config: Erlang project ที่ใช้ Elixir deps (ผ่าน hex)

{deps, [
    %% Pure Erlang
    {cowboy, "2.10.0"},
    {jsx, "3.1.0"},
    
    %% Can use Elixir hex packages compiled to BEAM
    %% แต่ต้องระวัง: Elixir standard library ต้องอยู่ใน path ด้วย
    {jason, "1.4.1"}  %% Elixir Jason - works if Elixir is installed
]}.

%% For pure Erlang projects, better to use Erlang-native alternatives
%% jsx หรือ jiffy แทน Jason
```

---

## ตัวอย่างจริง: Hybrid Application

```erlang
%% Erlang backend + Elixir Phoenix frontend ใน same release

%% erlang_core/src/core_engine.erl
-module(core_engine).
-behaviour(gen_server).
-export([start_link/0, process_event/1, get_stats/0]).

start_link() ->
    gen_server:start_link({global, core_engine}, ?MODULE, [], []).

process_event(Event) ->
    gen_server:cast({global, core_engine}, {event, Event}).

get_stats() ->
    gen_server:call({global, core_engine}, get_stats).

init(_) ->
    State = #{
        events_processed => 0,
        last_event => undefined,
        started_at => erlang:system_time(second)
    },
    {ok, State}.

handle_cast({event, Event}, #{events_processed := N} = State) ->
    %% Heavy computation in Erlang
    Processed = transform_event(Event),
    %% Notify Elixir Phoenix via PubSub
    notify_phoenix(Processed),
    {noreply, State#{
        events_processed := N + 1,
        last_event := Processed
    }};

handle_call(get_stats, _From, State) ->
    {reply, State, State}.

transform_event(Event) ->
    %% Erlang-specific processing (e.g., binary manipulation)
    maps:put(processed_by, <<"erlang_core">>, Event).

notify_phoenix(Event) ->
    PhoenixNode = 'phoenix@localhost',
    case net_adm:ping(PhoenixNode) of
        pong ->
            %% Call Elixir Phoenix PubSub
            'Elixir.Phoenix.PubSub':broadcast(
                'Elixir.MyApp.PubSub',
                <<"events">>,
                Event
            );
        pang ->
            logger:warning("Phoenix node not available")
    end.
```

```elixir
# Phoenix LiveView: receive events from Erlang core
# lib/my_app_web/live/dashboard_live.ex

defmodule MyAppWeb.DashboardLive do
  use Phoenix.LiveView

  def mount(_params, _session, socket) do
    Phoenix.PubSub.subscribe(MyApp.PubSub, "events")
    stats = :core_engine.get_stats()
    {:ok, assign(socket, stats: stats, events: [])}
  end

  def handle_info(event, socket) when is_map(event) do
    updated_events = [event | Enum.take(socket.assigns.events, 49)]
    {:noreply, assign(socket, events: updated_events)}
  end

  def render(assigns) do
    ~H"""
    <div>
      <h1>Real-time Dashboard</h1>
      <p>Events processed: <%= @stats.events_processed %></p>
      <%= for event <- @events do %>
        <div><%= inspect(event) %></div>
      <% end %>
    </div>
    """
  end
end
```

---

## สรุป Part 57

| ข้อควรรู้ | รายละเอียด |
|-----------|------------|
| Module naming | Elixir: `Elixir.ModuleName` ใน Erlang |
| Atom compat | `:ok` (Elixir) = `ok` (Erlang) |
| Map compat | Maps ทำงานเหมือนกัน |
| Struct | Map + `__struct__` key |
| Binary | <<>> เหมือนกัน |
| Cluster | nodes ต่อกันผ่าน cookie |
| PubSub | Phoenix PubSub ข้าม nodes |

---

*[← Part 56: Streaming](part_56_streaming.md) | [Part 58: Actor Model Deep Dive →](part_58_actor_model.md)*
