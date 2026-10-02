# Part 56: Streaming Data Processing

## สารบัญ
1. [Streaming Concepts](#streaming-concepts)
2. [GenStage Pattern](#genstage-pattern)
3. [Large File Processing](#large-file-processing)
4. [HTTP Streaming](#http-streaming)
5. [Stream Processing Pipeline](#stream-processing-pipeline)
6. [ตัวอย่างจริง: Real-time Log Processor](#ตัวอย่างจริง-real-time-log-processor)

---

## Streaming Concepts

```erlang
%% Streaming: process data in chunks without loading all into memory
%% Key benefits:
%% - Handle files larger than RAM
%% - Process data as it arrives (real-time)
%% - Back-pressure: don't overwhelm downstream
%% - Constant memory usage

%% Erlang tools for streaming:
%% - file:open with read mode + file:read(File, ChunkSize)
%% - binary:split for parsing
%% - io:get_line for line-by-line
%% - cowboy streaming for HTTP
%% - gen_tcp active modes
```

---

## GenStage Pattern

```erlang
%% GenStage: demand-driven data pipeline
%% (Elixir concept, implementable in Erlang)

%% Three roles:
%% Producer: generates data (responds to demand)
%% Consumer: consumes data (sends demand)
%% Producer-Consumer: both (transforms data)

-module(gen_producer).
-behaviour(gen_server).
-export([start_link/1, demand/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

start_link(Source) ->
    gen_server:start_link(?MODULE, Source, []).

demand(Pid, N) ->
    gen_server:cast(Pid, {demand, N, self()}).

init(Source) ->
    State = #{
        source => Source,
        buffer => [],
        pending_demand => []
    },
    {ok, State}.

handle_cast({demand, N, ConsumerPid}, State) ->
    State2 = add_demand(N, ConsumerPid, State),
    {noreply, try_dispatch(State2)}.

add_demand(N, Pid, #{pending_demand := PD} = State) ->
    State#{pending_demand := [{Pid, N} | PD]}.

try_dispatch(#{pending_demand := [], buffer := _} = State) -> State;
try_dispatch(#{buffer := []} = State) ->
    %% Load more data
    State2 = load_data(State),
    try_dispatch(State2);
try_dispatch(#{pending_demand := [{Pid, N} | Rest], buffer := Buffer} = State) ->
    {Send, NewBuffer} = case length(Buffer) >= N of
        true -> lists:split(N, Buffer);
        false -> {Buffer, []}
    end,
    Pid ! {data, Send},
    try_dispatch(State#{pending_demand := Rest, buffer := NewBuffer}).

load_data(#{source := {file, Fd}} = State) ->
    case file:read(Fd, 8192) of
        {ok, Data} ->
            Lines = binary:split(Data, <<"\n">>, [global]),
            State#{buffer := Lines};
        eof ->
            State#{source := eof, buffer := []}
    end.

%% Consumer
-module(gen_consumer).
-behaviour(gen_server).
-export([start_link/2]).
-export([init/1, handle_info/2]).

-define(BATCH_SIZE, 100).

start_link(Producer, ProcessFun) ->
    gen_server:start_link(?MODULE, {Producer, ProcessFun}, []).

init({Producer, ProcessFun}) ->
    gen_producer:demand(Producer, ?BATCH_SIZE),
    {ok, #{producer => Producer, process => ProcessFun}}.

handle_info({data, Items}, #{producer := Producer, process := Fun} = State) ->
    lists:foreach(Fun, Items),
    %% Request more
    gen_producer:demand(Producer, ?BATCH_SIZE),
    {noreply, State};

handle_info({eof}, State) ->
    logger:info("Stream complete"),
    {stop, normal, State}.
```

---

## Large File Processing

```erlang
%% Process files line-by-line without loading into memory

-module(file_stream).
-export([process_lines/3, count_lines/1, grep/3]).

%% Process each line with a function
process_lines(FilePath, ProcessFun, Opts) ->
    ChunkSize = maps:get(chunk_size, Opts, 65536),
    {ok, Fd} = file:open(FilePath, [read, binary, {read_ahead, ChunkSize}]),
    try
        Stats = process_loop(Fd, ProcessFun, <<>>, #{lines => 0, errors => 0}),
        {ok, Stats}
    after
        file:close(Fd)
    end.

process_loop(Fd, Fun, Buffer, Stats) ->
    case file:read(Fd, 65536) of
        {ok, Data} ->
            Combined = <<Buffer/binary, Data/binary>>,
            {Lines, Remaining} = split_lines(Combined),
            NewStats = process_lines_list(Lines, Fun, Stats),
            process_loop(Fd, Fun, Remaining, NewStats);
        eof ->
            %% Process remaining data (last line without newline)
            case Buffer of
                <<>> -> Stats;
                LastLine ->
                    process_line(LastLine, Fun, Stats)
            end;
        {error, Reason} ->
            {error, Reason}
    end.

split_lines(Binary) ->
    Parts = binary:split(Binary, <<"\n">>, [global]),
    case lists:reverse(Parts) of
        [Last | RevLines] ->
            {lists:reverse(RevLines), Last};
        [] ->
            {[], <<>>}
    end.

process_lines_list([], _Fun, Stats) -> Stats;
process_lines_list([Line | Rest], Fun, Stats) ->
    NewStats = process_line(Line, Fun, Stats),
    process_lines_list(Rest, Fun, NewStats).

process_line(Line, Fun, Stats) ->
    try
        Fun(Line),
        Stats#{lines := maps:get(lines, Stats, 0) + 1}
    catch
        _:_ ->
            Stats#{errors := maps:get(errors, Stats, 0) + 1}
    end.

count_lines(FilePath) ->
    {ok, Stats} = process_lines(FilePath, fun(_) -> ok end, #{}),
    maps:get(lines, Stats).

grep(FilePath, Pattern, OutputFile) ->
    {ok, Out} = file:open(OutputFile, [write, binary]),
    process_lines(FilePath, fun(Line) ->
        case re:run(Line, Pattern) of
            {match, _} -> file:write(Out, <<Line/binary, "\n">>);
            nomatch -> ok
        end
    end, #{}).

%% CSV processing
process_csv(FilePath, ProcessFun) ->
    process_lines(FilePath, fun(Line) ->
        Fields = binary:split(Line, <<",">>, [global]),
        ProcessFun(Fields)
    end, #{}).
```

---

## HTTP Streaming

```erlang
%% HTTP Chunked Transfer Encoding สำหรับ large responses

%% Cowboy streaming response
-module(stream_handler).
-export([init/2]).

init(Req, State) ->
    %% Start streaming response
    Req2 = cowboy_req:stream_reply(200, #{
        <<"content-type">> => <<"application/x-ndjson">>
    }, Req),
    %% Stream data
    stream_data(Req2, get_data_source()),
    {ok, Req2, State}.

stream_data(Req, Source) ->
    case next_chunk(Source) of
        {ok, Chunk, NewSource} ->
            Line = iolist_to_binary([jsx:encode(Chunk), "\n"]),
            cowboy_req:stream_body(Line, nofin, Req),
            stream_data(Req, NewSource);
        eof ->
            cowboy_req:stream_body(<<>>, fin, Req)
    end.

get_data_source() ->
    ets:first(big_table).

next_chunk(eof) -> eof;
next_chunk(Key) ->
    case ets:next(big_table, Key) of
        '$end_of_table' -> eof;
        NextKey ->
            [{_, Value}] = ets:lookup(big_table, NextKey),
            {ok, Value, NextKey}
    end.

%% Server-Sent Events (SSE) streaming
-module(sse_handler).
-export([init/2, info/3]).

init(Req, State) ->
    Req2 = cowboy_req:stream_reply(200, #{
        <<"content-type">> => <<"text/event-stream">>,
        <<"cache-control">> => <<"no-cache">>,
        <<"connection">> => <<"keep-alive">>
    }, Req),
    event_bus:subscribe(updates, self()),
    {cowboy_loop, Req2, State, 60000}.  %% 60s idle timeout

info({event, Data}, Req, State) ->
    Json = jsx:encode(Data),
    SSE = <<"data: ", Json/binary, "\n\n">>,
    cowboy_req:stream_body(SSE, nofin, Req),
    {ok, Req, State};

info(timeout, Req, State) ->
    cowboy_req:stream_body(<<":\n\n">>, nofin, Req),  %% keepalive comment
    {ok, Req, State}.

%% HTTP client: download large file in chunks
download_file(Url, OutputPath) ->
    {ok, _} = httpc:request(get, {Url, []},
        [{timeout, 30000}],
        [{stream, {file, OutputPath}}]).

%% Stream with progress
stream_with_progress(Url, OutputPath, ProgressFun) ->
    {ok, Fd} = file:open(OutputPath, [write, binary]),
    Ref = make_ref(),
    httpc:request(get, {Url, []},
        [{timeout, 300000}],
        [{sync, false}, {stream, self}, {receiver, {self(), Ref}}]),
    stream_loop(Ref, Fd, 0, ProgressFun).

stream_loop(Ref, Fd, Written, ProgressFun) ->
    receive
        {http, {Ref, stream_start, _Headers}} ->
            stream_loop(Ref, Fd, Written, ProgressFun);
        {http, {Ref, stream, Data}} ->
            file:write(Fd, Data),
            NewWritten = Written + byte_size(Data),
            ProgressFun(NewWritten),
            stream_loop(Ref, Fd, NewWritten, ProgressFun);
        {http, {Ref, stream_end, _Headers}} ->
            file:close(Fd),
            {ok, Written}
    end.
```

---

## ตัวอย่างจริง: Real-time Log Processor

```erlang
%% log_processor.erl: real-time log analysis pipeline

-module(log_processor).
-behaviour(gen_server).
-export([start_link/1, get_stats/0]).
-export([init/1, handle_info/2, handle_call/3]).

%% Log format: timestamp level module message
%% 2024-01-15T10:30:00Z INFO cowboy_handler Request completed

start_link(LogFile) ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, LogFile, []).

get_stats() ->
    gen_server:call(?MODULE, get_stats).

init(LogFile) ->
    %% Tail file: follow new entries
    Port = open_port({spawn, "tail -F " ++ LogFile}, [
        binary, {line, 4096}, exit_status
    ]),
    State = #{
        port => Port,
        stats => #{
            total => 0,
            by_level => #{},
            by_module => #{},
            error_rate => 0,
            recent_errors => []
        },
        window_start => erlang:system_time(second),
        window_errors => 0
    },
    {ok, State}.

handle_info({Port, {data, {eol, Line}}}, #{port := Port} = State) ->
    NewState = process_log_line(Line, State),
    {noreply, NewState};

handle_info({Port, {exit_status, _}}, State) ->
    logger:error("Log tail process died, restarting"),
    {stop, log_died, State}.

handle_call(get_stats, _From, State) ->
    {reply, maps:get(stats, State), State}.

process_log_line(Line, State) ->
    case parse_log_line(Line) of
        {ok, #{level := Level, module := Module} = Entry} ->
            update_stats(Entry, State);
        {error, _} ->
            State
    end.

parse_log_line(Line) ->
    Pattern = "^(\\S+) (\\w+) (\\w+) (.+)$",
    case re:run(Line, Pattern, [{capture, all_but_first, binary}]) of
        {match, [Ts, Level, Module, Msg]} ->
            {ok, #{
                timestamp => Ts,
                level => Level,
                module => Module,
                message => Msg
            }};
        _ ->
            {error, parse_failed}
    end.

update_stats(#{level := Level, module := Module, message := Msg}, State) ->
    Stats = maps:get(stats, State),
    
    %% Increment counters
    ByLevel = maps:get(by_level, Stats, #{}),
    ByModule = maps:get(by_module, Stats, #{}),
    
    NewStats = Stats#{
        total := maps:get(total, Stats, 0) + 1,
        by_level := ByLevel#{Level => maps:get(Level, ByLevel, 0) + 1},
        by_module := ByModule#{Module => maps:get(Module, ByModule, 0) + 1}
    },
    
    %% Track error rate in sliding window
    Now = erlang:system_time(second),
    WindowStart = maps:get(window_start, State),
    WindowErrors = maps:get(window_errors, State),
    
    {NewWindowStart, NewWindowErrors} = case Now - WindowStart > 60 of
        true ->
            %% New window
            {Now, 0};
        false ->
            {WindowStart, WindowErrors + case Level of <<"error">> -> 1; _ -> 0 end}
    end,
    
    ErrorRate = NewWindowErrors / max(1, Now - NewWindowStart),
    
    %% Alert on high error rate
    case ErrorRate > 10 of  %% > 10 errors/second
        true -> alert_high_error_rate(ErrorRate);
        false -> ok
    end,
    
    %% Keep recent errors
    RecentErrors = case Level of
        <<"error">> ->
            lists:sublist([#{msg => Msg, time => Now} |
                           maps:get(recent_errors, NewStats, [])], 100);
        _ ->
            maps:get(recent_errors, NewStats, [])
    end,
    
    State#{
        stats := NewStats#{
            error_rate := ErrorRate,
            recent_errors := RecentErrors
        },
        window_start := NewWindowStart,
        window_errors := NewWindowErrors
    }.

alert_high_error_rate(Rate) ->
    logger:critical("High error rate detected", #{rate => Rate}),
    prometheus_gauge:set(error_rate, Rate).
```

---

## สรุป Part 56

| Use Case | Approach |
|----------|----------|
| Large files | `file:read/2` with chunk size |
| Line-by-line | `{line, N}` port option |
| HTTP streaming | `cowboy_req:stream_body/3` |
| SSE | `cowboy_loop` + stream |
| Download | `httpc stream` |
| Back-pressure | GenStage demand |
| Real-time | Port tail + parse |

---

*[← Part 55: Interoperability](part_55_interoperability.md) | [Part 57: Erlang and Elixir →](part_57_erlang_elixir.md)*
