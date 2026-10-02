# Part 75: IoT with Erlang — Embedded Systems and Device Management

## สารบัญ
1. [MQTT Protocol](#mqtt)
2. [Device Registry](#registry)
3. [Telemetry Pipeline](#telemetry)
4. [OTA Updates](#ota)
5. [Edge Computing](#edge)
6. [ตัวอย่างจริง: Smart Home Hub](#smart-home)

---

## MQTT Protocol

```
MQTT = lightweight publish/subscribe protocol for IoT
- Low overhead (2-byte header minimum)
- QoS levels: 0 (at most once), 1 (at least once), 2 (exactly once)
- Last Will message: notify when device disconnects
- Retain messages: new subscribers get last value
- Perfect for constrained devices (MCUs, sensors)

Erlang library: emqtt (from EMQ Technologies)
{deps, [{emqtt, "1.9.0"}]}
```

```erlang
%% mqtt_client.erl — connecting to MQTT broker

-module(mqtt_client).
-export([start/0, subscribe/2, publish/3]).

start() ->
    {ok, Client} = emqtt:start_link(#{
        host => "mqtt.myiothub.com",
        port => 8883,          %% MQTTS (MQTT over TLS)
        username => <<"device_id">>,
        password => <<"device_secret">>,
        ssl => true,
        ssl_opts => [
            {verify, verify_peer},
            {cacertfile, "/etc/ssl/certs/ca-bundle.crt"}
        ],
        will_topic => <<"devices/my_device/status">>,
        will_payload => <<"offline">>,
        will_retain => true,
        will_qos => 1,
        keepalive => 60,       %% ping every 60 seconds
        clientid => <<"erlang_hub_001">>
    }),
    
    {ok, _} = emqtt:connect(Client),
    {ok, Client}.

subscribe(Client, TopicPattern) ->
    emqtt:subscribe(Client, TopicPattern, 1).

publish(Client, Topic, Payload) ->
    emqtt:publish(Client, Topic, Payload, [{qos, 1}, {retain, false}]).

%% Handle incoming messages
handle_message({publish, #{topic := Topic, payload := Payload}}) ->
    TopicParts = binary:split(Topic, <<"/">>, [global]),
    route_message(TopicParts, Payload).

route_message([<<"devices">>, DeviceId, <<"telemetry">>], Payload) ->
    Data = jsx:decode(Payload, [return_maps]),
    telemetry_pipeline:ingest(DeviceId, Data);

route_message([<<"devices">>, DeviceId, <<"status">>], Status) ->
    device_registry:update_status(DeviceId, Status);

route_message([<<"devices">>, DeviceId, <<"command_ack">>], Payload) ->
    Data = jsx:decode(Payload, [return_maps]),
    command_tracker:ack(DeviceId, maps:get(<<"command_id">>, Data));

route_message(Topic, _Payload) ->
    logger:debug("Unhandled MQTT topic", #{topic => Topic}).
```

---

## Device Registry

```erlang
%% device_registry.erl — track device state and metadata

-module(device_registry).
-behaviour(gen_server).
-export([start_link/0, register/2, update_status/2, get_device/1,
         list_devices/0, list_by_type/1, update_telemetry/2]).
-export([init/1, handle_call/3, handle_cast/2]).

%% ETS-backed for fast reads
init([]) ->
    ets:new(devices, [named_table, public, {read_concurrency, true}]),
    {ok, #{}}.

register(DeviceId, Metadata) ->
    gen_server:call(?MODULE, {register, DeviceId, Metadata}).

update_status(DeviceId, Status) ->
    gen_server:cast(?MODULE, {update_status, DeviceId, Status}).

update_telemetry(DeviceId, Telemetry) ->
    %% Direct ETS write for high throughput
    case ets:lookup(devices, DeviceId) of
        [{DeviceId, Device}] ->
            Updated = Device#{
                last_telemetry => Telemetry,
                last_seen => erlang:system_time(second)
            },
            ets:insert(devices, {DeviceId, Updated});
        [] -> ok  %% Device not registered
    end.

get_device(DeviceId) ->
    case ets:lookup(devices, DeviceId) of
        [{DeviceId, Device}] -> {ok, Device};
        [] -> not_found
    end.

list_devices() ->
    [Device || {_, Device} <- ets:tab2list(devices)].

list_by_type(Type) ->
    ets:select(devices, [
        {{'_', #{type => Type, '_' => '_'}}, [], ['$_']}
    ]).

handle_call({register, DeviceId, Meta}, _From, State) ->
    Device = Meta#{
        id => DeviceId,
        status => <<"offline">>,
        registered_at => erlang:system_time(second),
        last_seen => erlang:system_time(second)
    },
    ets:insert(devices, {DeviceId, Device}),
    {reply, ok, State}.

handle_cast({update_status, DeviceId, Status}, State) ->
    case ets:lookup(devices, DeviceId) of
        [{DeviceId, Device}] ->
            Updated = Device#{
                status => Status,
                last_seen => erlang:system_time(second)
            },
            ets:insert(devices, {DeviceId, Updated});
        [] ->
            %% Auto-register unknown device
            logger:warning("Unknown device connected", #{device_id => DeviceId}),
            ets:insert(devices, {DeviceId, #{id => DeviceId, status => Status}})
    end,
    {noreply, State}.

%% Device health monitoring
-spec find_stale_devices(integer()) -> [map()].
find_stale_devices(MaxAgeSec) ->
    Now = erlang:system_time(second),
    ets:select(devices, [
        {{'_', #{last_seen => '$1', status => <<"online">>, '_' => '_'}},
         [{'<', '$1', Now - MaxAgeSec}],
         ['$_']}
    ]).

check_device_health() ->
    %% Find devices not seen in 5 minutes
    StaleDevices = find_stale_devices(300),
    [begin
        DeviceId = maps:get(id, D),
        logger:warning("Device appears offline", #{device_id => DeviceId}),
        update_status(DeviceId, <<"offline">>),
        alert_manager:device_offline(DeviceId)
    end || {_, D} <- StaleDevices].
```

---

## Telemetry Pipeline

```erlang
%% telemetry_pipeline.erl — process sensor data streams

-module(telemetry_pipeline).
-behaviour(gen_stage).
-export([ingest/2, start_link/0]).

%% Stage 1: Producer — receives telemetry from MQTT
-module(telemetry_producer).
-behaviour(gen_stage).
-export([start_link/0, ingest/2]).
-export([init/1, handle_cast/2, handle_demand/2]).

start_link() ->
    gen_stage:start_link({local, ?MODULE}, ?MODULE, [], []).

ingest(DeviceId, Data) ->
    gen_stage:cast(?MODULE, {telemetry, DeviceId, Data}).

init([]) ->
    {producer, #{buffer => [], demand => 0}}.

handle_cast({telemetry, DeviceId, Data}, #{buffer := Buffer, demand := 0} = State) ->
    {noreply, [], State#{buffer := [{DeviceId, Data, erlang:system_time(millisecond)} | Buffer]}};

handle_cast({telemetry, DeviceId, Data}, #{demand := Demand} = State) when Demand > 0 ->
    Event = {DeviceId, Data, erlang:system_time(millisecond)},
    {noreply, [Event], State#{demand := Demand - 1}}.

handle_demand(NewDemand, #{buffer := [], demand := Demand} = State) ->
    {noreply, [], State#{demand := Demand + NewDemand}};

handle_demand(NewDemand, #{buffer := Buffer} = State) ->
    {ToSend, Remaining} = lists:split(min(NewDemand, length(Buffer)), lists:reverse(Buffer)),
    {noreply, ToSend, State#{buffer := lists:reverse(Remaining), demand := 0}}.

%% Stage 2: Transformer — validate, enrich, filter
-module(telemetry_transformer).
-behaviour(gen_stage).
-export([start_link/0]).
-export([init/1, handle_events/3]).

start_link() ->
    gen_stage:start_link(?MODULE, [], []).

init([]) ->
    {producer_consumer, #{}, [{subscribe_to, [telemetry_producer]}]}.

handle_events(Events, _From, State) ->
    Transformed = lists:filtermap(fun({DeviceId, Data, Ts}) ->
        case validate_telemetry(Data) of
            {ok, ValidData} ->
                Enriched = enrich_telemetry(DeviceId, ValidData, Ts),
                {true, Enriched};
            {error, Reason} ->
                logger:warning("Invalid telemetry", #{device => DeviceId, reason => Reason}),
                false
        end
    end, Events),
    {noreply, Transformed, State}.

validate_telemetry(#{<<"temperature">> := T} = Data) when is_number(T), T > -50, T < 100 ->
    {ok, Data};
validate_telemetry(#{<<"humidity">> := H} = Data) when is_number(H), H >= 0, H =< 100 ->
    {ok, Data};
validate_telemetry(Data) ->
    case maps:size(Data) > 0 of
        true -> {ok, Data};
        false -> {error, empty_payload}
    end.

enrich_telemetry(DeviceId, Data, Timestamp) ->
    DeviceInfo = case device_registry:get_device(DeviceId) of
        {ok, Device} -> #{location => maps:get(location, Device, undefined),
                          type => maps:get(type, Device, undefined)};
        _ -> #{}
    end,
    maps:merge(Data, DeviceInfo#{device_id => DeviceId, timestamp => Timestamp}).

%% Stage 3: Consumer — store and alert
-module(telemetry_consumer).
-behaviour(gen_stage).
-export([start_link/0]).
-export([init/1, handle_events/3]).

start_link() ->
    gen_stage:start_link(?MODULE, [], []).

init([]) ->
    {consumer, #{}, [{subscribe_to, [{telemetry_transformer, [{max_demand, 100}]}]}]}.

handle_events(Events, _From, State) ->
    %% Batch insert to time-series DB
    TimescaleDB = get_db_conn(),
    batch_insert(TimescaleDB, Events),
    
    %% Check thresholds
    [check_thresholds(Event) || Event <- Events],
    
    %% Update device cache
    [device_registry:update_telemetry(maps:get(device_id, E), E) || E <- Events],
    
    {noreply, [], State}.

batch_insert(Conn, Events) ->
    Values = [
        {maps:get(device_id, E), maps:get(timestamp, E),
         maps:get(<<"temperature">>, E, null),
         maps:get(<<"humidity">>, E, null)}
        || E <- Events
    ],
    %% INSERT INTO telemetry (device_id, timestamp, temperature, humidity) VALUES ...
    epgsql:equery(Conn,
        "INSERT INTO telemetry (device_id, ts, temperature, humidity) "
        "SELECT * FROM unnest($1::text[], $2::bigint[], $3::float[], $4::float[])",
        [lists:unzip4(Values)]).

check_thresholds(#{device_id := DeviceId, <<"temperature">> := Temp}) when Temp > 80 ->
    alert_manager:critical(#{
        type => high_temperature,
        device => DeviceId,
        value => Temp,
        threshold => 80
    });
check_thresholds(_) -> ok.
```

---

## OTA Updates

```erlang
%% ota_manager.erl — Over-the-Air firmware update management

-module(ota_manager).
-export([deploy_update/2, get_update_status/1, rollback/1]).

-record(update_job, {
    job_id,
    firmware_version,
    target_devices,
    status,       %% pending | in_progress | completed | failed
    results = #{} %% DeviceId => {status, timestamp}
}).

deploy_update(FirmwareVersion, TargetDevices) ->
    JobId = generate_job_id(),
    
    %% Validate firmware exists
    case firmware_store:get(FirmwareVersion) of
        {ok, Firmware} ->
            Job = #update_job{
                job_id = JobId,
                firmware_version = FirmwareVersion,
                target_devices = TargetDevices,
                status = in_progress
            },
            
            %% Store job
            ets:insert(update_jobs, {JobId, Job}),
            
            %% Deploy in batches (avoid overwhelming devices)
            BatchSize = 10,
            Batches = batch_list(TargetDevices, BatchSize),
            
            spawn(fun() ->
                deploy_batches(JobId, Batches, Firmware, 0)
            end),
            
            {ok, JobId};
        not_found ->
            {error, firmware_not_found}
    end.

deploy_batches(_, [], _, _) -> ok;
deploy_batches(JobId, [Batch | Rest], Firmware, Offset) ->
    logger:info("Deploying update batch", #{job_id => JobId, offset => Offset, count => length(Batch)}),
    
    %% Send update command to each device in batch
    [send_update_command(DeviceId, Firmware) || DeviceId <- Batch],
    
    %% Wait between batches
    timer:sleep(5000),
    
    deploy_batches(JobId, Rest, Firmware, Offset + length(Batch)).

send_update_command(DeviceId, #{version := Version, url := Url, checksum := Checksum}) ->
    Command = jiffy:encode(#{
        type => <<"ota_update">>,
        version => Version,
        url => Url,
        checksum => Checksum,
        command_id => generate_command_id()
    }),
    mqtt_client:publish(global_client(), <<"devices/", DeviceId/binary, "/commands">>, Command).

%% Device sends back result
handle_update_result(DeviceId, #{<<"command_id">> := CmdId, <<"status">> := Status}) ->
    logger:info("OTA update result", #{device_id => DeviceId, status => Status}),
    
    case Status of
        <<"success">> ->
            device_registry:update_firmware_version(DeviceId, get_target_version(CmdId));
        <<"failed">> ->
            logger:error("OTA update failed", #{device_id => DeviceId}),
            alert_manager:ota_failure(DeviceId)
    end.

batch_list(List, N) ->
    batch_list(List, N, []).
batch_list([], _, Acc) ->
    lists:reverse(Acc);
batch_list(List, N, Acc) ->
    {Batch, Rest} = lists:split(min(N, length(List)), List),
    batch_list(Rest, N, [Batch | Acc]).

generate_job_id() ->
    base64:encode(crypto:strong_rand_bytes(9)).
generate_command_id() ->
    base64:encode(crypto:strong_rand_bytes(9)).
```

---

## ตัวอย่างจริง: Smart Home Hub

```erlang
%% smart_home_hub.erl — central coordinator for home automation

-module(smart_home_hub).
-behaviour(gen_server).
-export([start_link/0, register_device/2, trigger_rule/2, add_automation/2]).

%% Device types: temperature_sensor, humidity_sensor, light, thermostat, lock, camera

-record(hub_state, {
    devices = #{},       %% DeviceId => {type, state, capabilities}
    automations = [],    %% [{trigger, condition, actions}]
    scenes = #{}         %% SceneName => [{DeviceId, Action}]
}).

init([]) ->
    %% Subscribe to all device topics
    mqtt_client:subscribe(global_client(), <<"home/+/telemetry">>),
    mqtt_client:subscribe(global_client(), <<"home/+/status">>),
    
    {ok, #hub_state{}}.

%% Process incoming MQTT messages
handle_cast({mqtt_message, [<<"home">>, DeviceId, <<"telemetry">>], Payload}, State) ->
    Data = jiffy:decode(Payload, [return_maps]),
    
    %% Update device state
    NewDevices = maps:update_with(DeviceId, fun(D) ->
        D#{state => maps:merge(maps:get(state, D, #{}), Data)}
    end, #{state => Data}, State#hub_state.devices),
    
    %% Evaluate automation rules
    NewState = State#hub_state{devices = NewDevices},
    evaluate_automations(NewState, DeviceId, Data),
    
    {noreply, NewState}.

%% Add automation rule
add_automation(Trigger, #{condition := Condition, actions := Actions}) ->
    Rule = #{trigger => Trigger, condition => Condition, actions => Actions},
    gen_server:cast(?MODULE, {add_automation, Rule}).

evaluate_automations(#hub_state{automations = Rules, devices = Devices}, TriggerDevice, Data) ->
    MatchingRules = [R || R <- Rules,
                     maps:get(trigger, R) =:= TriggerDevice],
    
    [execute_if_condition_met(Rule, Devices, Data) || Rule <- MatchingRules].

execute_if_condition_met(#{condition := Condition, actions := Actions}, Devices, Data) ->
    case eval_condition(Condition, Devices, Data) of
        true ->
            [execute_action(Action, Devices) || Action <- Actions];
        false ->
            ok
    end.

eval_condition({temp_above, Threshold}, _, #{<<"temperature">> := T}) ->
    T > Threshold;
eval_condition({temp_below, Threshold}, _, #{<<"temperature">> := T}) ->
    T < Threshold;
eval_condition({time_between, Start, End}, _, _) ->
    Now = calendar:local_time(),
    {_, {H, M, _}} = Now,
    CurrentMin = H * 60 + M,
    CurrentMin >= Start andalso CurrentMin =< End;
eval_condition(_, _, _) -> false.

execute_action({set_light, DeviceId, #{brightness := B, color := C}}, _Devices) ->
    Command = jiffy:encode(#{command => <<"set">>, brightness => B, color => C}),
    mqtt_client:publish(global_client(), <<"home/", DeviceId/binary, "/command">>, Command);

execute_action({set_thermostat, DeviceId, Temp}, _Devices) ->
    Command = jiffy:encode(#{command => <<"set_temperature">>, temperature => Temp}),
    mqtt_client:publish(global_client(), <<"home/", DeviceId/binary, "/command">>, Command);

execute_action({notify, Message}, _Devices) ->
    notification_service:send(Message).

%% Example automation: if temperature > 28°C and time is 14:00-18:00, turn on AC
example_automation() ->
    add_automation(<<"temp_sensor_living_room">>, #{
        condition => {temp_above, 28},
        actions => [
            {set_thermostat, <<"ac_living_room">>, 24},
            {notify, <<"AC turned on automatically">>}
        ]
    }).
```

---

## สรุป Part 75

| IoT Pattern | Erlang Approach |
|-------------|----------------|
| MQTT client | emqtt library |
| Device registry | ETS + gen_server |
| Telemetry | GenStage pipeline |
| OTA updates | Batch command + ACK tracking |
| Automation | Rule engine + pattern matching |
| Edge hub | Local gen_server coordinator |

**ข้อดีของ Erlang สำหรับ IoT**: Process per device = natural isolation, message passing = event-driven, fault tolerance = self-healing

---

*[← Part 74: Game Server](part_74_game_server.md) | [Part 76: Real-Time Bidding →](part_76_rtb.md)*
