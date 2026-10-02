# Part 20: File I/O — การอ่านเขียนไฟล์

## สารบัญ
1. [file Module พื้นฐาน](#file-module-พื้นฐาน)
2. [การอ่านไฟล์](#การอ่านไฟล์)
3. [การเขียนไฟล์](#การเขียนไฟล์)
4. [File Handles (Random Access)](#file-handles-random-access)
5. [Directory Operations](#directory-operations)
6. [Path Operations](#path-operations)
7. [Streaming และ Large Files](#streaming-และ-large-files)
8. [ตัวอย่างจริง: Log File Processor](#ตัวอย่างจริง-log-file-processor)

---

## file Module พื้นฐาน

```erlang
%% file module มาพร้อมกับ Erlang/OTP

%% Error codes ที่พบบ่อย
%% enoent  - file not found
%% eacces  - permission denied
%% eexist  - file already exists
%% eisdir  - is a directory
%% enotdir - not a directory
%% enospc  - no space left
%% enotempty - directory not empty

%% ตรวจสอบ file exists
filelib:is_file("/path/to/file").      %% = true | false
filelib:is_dir("/path/to/dir").        %% = true | false
filelib:is_regular("/path/to/file").   %% = true (not symlink/dir)

%% File info
file:read_file_info("/path/to/file").
%% = {ok, #file_info{size=1234, type=regular, access=read_write, ...}}

%% file_info record fields:
%% size       - bytes
%% type       - regular | directory | symlink | device | other
%% access     - read | write | read_write | none
%% atime      - last access time {{Y,M,D},{H,Min,S}}
%% mtime      - last modification time
%% ctime      - creation/change time
%% mode       - Unix permissions (integer)
%% links      - number of hard links
%% major_device, minor_device - device numbers
%% inode      - inode number
%% uid, gid   - user/group IDs
```

---

## การอ่านไฟล์

```erlang
%% 1. อ่านทั้งไฟล์เป็น binary (ง่ายที่สุด)
{ok, Content} = file:read_file("/path/to/file.txt").
%% Content = <<"line1\nline2\nline3\n">>

%% 2. อ่านทั้งไฟล์เป็น list of terms (Erlang terms)
{ok, [Term1, Term2]} = file:consult("/path/to/data.erl").
%% ไฟล์ต้องมี Erlang terms ที่ valid และต่อท้ายด้วย .

%% ตัวอย่างไฟล์ config.erl:
%% {host, "localhost"}.
%% {port, 8080}.
{ok, Config} = file:consult("config.erl").
%% Config = [{host, "localhost"}, {port, 8080}]

%% 3. อ่านทีละบรรทัด
{ok, Device} = file:open("file.txt", [read, binary]),
read_lines(Device, []).

read_lines(Device, Acc) ->
    case file:read_line(Device) of
        {ok, Line} -> read_lines(Device, [Line | Acc]);
        eof -> {ok, lists:reverse(Acc)};
        Error -> Error
    end.

%% 4. อ่านแบบ binary chunks
{ok, Device} = file:open("bigfile.bin", [read, binary, raw]),
read_chunks(Device, ChunkSize, []).

read_chunks(Device, ChunkSize, Acc) ->
    case file:read(Device, ChunkSize) of
        {ok, Data} -> read_chunks(Device, ChunkSize, [Data|Acc]);
        eof -> {ok, iolist_to_binary(lists:reverse(Acc))};
        Error -> Error
    end.

%% 5. อ่านไฟล์ CSV
read_csv(Filename) ->
    {ok, Content} = file:read_file(Filename),
    Lines = binary:split(Content, [<<"\r\n">>, <<"\n">>], [global, trim_all]),
    [binary:split(Line, <<",">>, [global]) || Line <- Lines].

%% 6. อ่านไฟล์ JSON (ต้องใช้ library เช่น jsx หรือ jsone)
read_json(Filename) ->
    {ok, Content} = file:read_file(Filename),
    jsx:decode(Content, [return_maps]).
```

---

## การเขียนไฟล์

```erlang
%% 1. เขียนทั้งไฟล์ (overwrite)
ok = file:write_file("output.txt", <<"Hello, World!\n">>).
ok = file:write_file("output.txt", "Hello, World!\n").  %% string ก็ได้
ok = file:write_file("output.txt", [<<"Hello">>, <<", ">>, <<"World!\n">>]).  %% iolist

%% 2. เพิ่มท้ายไฟล์ (append)
ok = file:write_file("log.txt", <<"New log entry\n">>, [append]).

%% 3. เขียน Erlang terms
Terms = [{name, alice}, {age, 30}].
ok = file:write_file("data.erl",
    [io_lib:format("~p.~n", [T]) || T <- Terms]).
%% สร้างไฟล์ที่ file:consult/1 อ่านได้

%% 4. เขียน CSV
write_csv(Filename, Headers, Rows) ->
    HeaderLine = iolist_to_binary(lists:join(<<",">>, [atom_to_binary(H) || H <- Headers])),
    DataLines = [
        iolist_to_binary(lists:join(<<",">>, [format_csv_value(V) || V <- Row]))
        || Row <- Rows
    ],
    Content = iolist_to_binary(lists:join(<<"\n">>, [HeaderLine | DataLines])),
    file:write_file(Filename, Content).

format_csv_value(V) when is_binary(V) -> V;
format_csv_value(V) when is_integer(V) -> integer_to_binary(V);
format_csv_value(V) when is_float(V) -> float_to_binary(V, [{decimals, 4}]);
format_csv_value(V) when is_atom(V) -> atom_to_binary(V).

%% 5. Write with io:format
{ok, Device} = file:open("output.txt", [write]),
io:format(Device, "Name: ~s~n", ["Alice"]),
io:format(Device, "Age: ~p~n", [30]),
file:close(Device).

%% 6. Write JSON
write_json(Filename, Data) ->
    file:write_file(Filename, jsx:encode(Data)).
```

---

## File Handles (Random Access)

```erlang
%% ใช้ file:open/2 สำหรับ random access และ complex operations

%% Modes:
%% read        - read only
%% write       - write only (create or truncate)
%% append      - append only
%% read_write  - read and write
%% binary      - return binary instead of list
%% raw         - bypass Erlang I/O server (faster for large files)
%% compressed  - gzip compressed
%% delayed_write - buffer writes
%% read_ahead  - buffer reads

{ok, Device} = file:open("file.txt", [read, write, binary]).

%% Read N bytes
{ok, Data} = file:read(Device, 100).   %% read 100 bytes
%% or
{ok, Line} = file:read_line(Device).   %% read one line

%% Write
ok = file:write(Device, <<"Hello\n">>).

%% Position
{ok, Pos} = file:position(Device, cur).   %% get current position
{ok, _} = file:position(Device, bof).     %% seek to beginning
{ok, _} = file:position(Device, eof).     %% seek to end
{ok, _} = file:position(Device, {bof, 100}).   %% 100 bytes from start
{ok, _} = file:position(Device, {eof, -100}).  %% 100 bytes before end
{ok, _} = file:position(Device, {cur, 50}).    %% 50 bytes from current

%% Truncate at current position
ok = file:truncate(Device).

%% Sync to disk
ok = file:sync(Device).

%% Close
ok = file:close(Device).

%% Copy file
{ok, BytesCopied} = file:copy("source.txt", "dest.txt").
{ok, _} = file:copy("source.txt", "dest.txt", 1024).  %% max bytes

%% ตัวอย่าง: Binary file editor
patch_binary_file(Filename, Offset, NewData) ->
    {ok, Device} = file:open(Filename, [read, write, binary, raw]),
    try
        {ok, _} = file:position(Device, {bof, Offset}),
        ok = file:write(Device, NewData)
    after
        file:close(Device)
    end.
```

---

## Directory Operations

```erlang
%% List directory
{ok, Files} = file:list_dir("/path/to/dir").
%% = ["file1.txt", "file2.erl", "subdir"]

{ok, Files2} = file:list_dir_all("/path/to/dir").
%% อ่านได้แม้มีไฟล์ที่อ่านชื่อ encoding ไม่ได้

%% Create directory
ok = file:make_dir("/path/to/newdir").

%% Create nested directories
ok = filelib:ensure_dir("/path/to/deep/nested/file.txt").
%% สร้าง /path/to/deep/nested/ (ไม่สร้าง file.txt)

ok = filelib:ensure_path("/path/to/deep/nested/").
%% สร้าง path ทั้งหมด

%% Remove directory (must be empty)
ok = file:del_dir("/path/to/emptydir").

%% Remove file
ok = file:delete("/path/to/file.txt").

%% Rename / Move
ok = file:rename("/old/path/file.txt", "/new/path/file.txt").

%% Symbolic links
ok = file:make_symlink("/target/path", "/link/path").
{ok, Target} = file:read_link("/link/path").

%% File permissions (Unix)
ok = file:change_mode("/path/to/file", 8#644).  %% rw-r--r--
ok = file:change_owner("/path/to/file", Uid, Gid).

%% Walk directory recursively
walk_dir(Dir, Fun) ->
    {ok, Files} = file:list_dir(Dir),
    lists:foreach(
        fun(Name) ->
            Path = filename:join(Dir, Name),
            case filelib:is_dir(Path) of
                true  -> walk_dir(Path, Fun);
                false -> Fun(Path)
            end
        end,
        Files
    ).

%% Find files by pattern
filelib:wildcard("/path/**/*.erl").   %% all .erl files recursively
filelib:wildcard("/path/*.txt").      %% .txt in dir
filelib:fold_files("/path", ".*\\.erl$", true, fun(File, Acc) -> [File|Acc] end, []).
```

---

## Path Operations

```erlang
%% filename module

%% Join paths
filename:join(["/home", "user", "file.txt"]).
%% = "/home/user/file.txt"

filename:join("/home/user", "file.txt").
%% = "/home/user/file.txt"

%% Split
filename:split("/home/user/file.txt").
%% = ["/", "home", "user", "file.txt"]

%% Dirname and basename
filename:dirname("/home/user/file.txt").  %% = "/home/user"
filename:basename("/home/user/file.txt"). %% = "file.txt"
filename:basename("/home/user/file.txt", ".txt"). %% = "file" (remove ext)

%% Extension
filename:extension("/home/user/file.txt").  %% = ".txt"
filename:extension("/home/user/noext").     %% = ""

%% Absolute path
filename:absname("relative/path").   %% = "/current/dir/relative/path"
filename:absname("../other").

%% Normalize
filename:nativename("/home/user/file.txt").  %% OS-specific separators

%% Safe path building (prevent directory traversal)
safe_path(Base, UserInput) ->
    Joined = filename:join(Base, UserInput),
    Abs = filename:absname(Joined),
    BaseAbs = filename:absname(Base),
    case string:prefix(Abs, BaseAbs) of
        nomatch -> {error, path_traversal};
        _ -> {ok, Abs}
    end.

safe_path("/var/www", "../../etc/passwd").
%% = {error, path_traversal}

safe_path("/var/www", "public/index.html").
%% = {ok, "/var/www/public/index.html"}

%% Temporary files
Tmpdir = filename:basedir(user_cache, "myapp").
%% = "/home/user/.cache/myapp" (Linux)
%% = "/Users/user/Library/Caches/myapp" (macOS)

filelib:ensure_path(Tmpdir ++ "/"),
TmpFile = filename:join(Tmpdir, "temp_" ++ integer_to_list(erlang:unique_integer([positive]))),
file:write_file(TmpFile, <<"temporary data">>).
```

---

## Streaming และ Large Files

```erlang
%% สำหรับไฟล์ขนาดใหญ่ ควรอ่านทีละส่วน

%% Streaming read
stream_file(Filename, Processor) ->
    {ok, Device} = file:open(Filename, [read, binary, raw, {read_ahead, 65536}]),
    try
        stream_loop(Device, Processor)
    after
        file:close(Device)
    end.

stream_loop(Device, Processor) ->
    case file:read(Device, 65536) of
        {ok, Chunk} ->
            Processor(Chunk),
            stream_loop(Device, Processor);
        eof ->
            ok;
        {error, Reason} ->
            {error, Reason}
    end.

%% ตัวอย่าง: count lines in large file
count_lines(Filename) ->
    Ref = make_ref(),
    put(Ref, 0),
    stream_file(Filename, fun(Chunk) ->
        Count = length([X || <<X>> <= Chunk, X =:= $\n]),
        put(Ref, get(Ref) + Count)
    end),
    erase(Ref).

%% Line-by-line processing
process_lines(Filename, Fun) ->
    {ok, Device} = file:open(Filename, [read, binary, {read_ahead, 65536}]),
    try
        process_line_loop(Device, Fun)
    after
        file:close(Device)
    end.

process_line_loop(Device, Fun) ->
    case file:read_line(Device) of
        {ok, Line} ->
            Fun(string:trim(Line, trailing, "\r\n")),
            process_line_loop(Device, Fun);
        eof ->
            ok;
        Error ->
            Error
    end.

%% Streaming write with buffering
buffered_write(Filename, DataGenerator) ->
    {ok, Device} = file:open(Filename, [write, binary, raw,
                                        {delayed_write, 65536, 2000}]),
    try
        lists:foreach(
            fun(Chunk) ->
                file:write(Device, Chunk)
            end,
            DataGenerator()
        )
    after
        file:close(Device)
    end.

%% Parallel file processing
parallel_process_files(FileList, Fun) ->
    Parent = self(),
    Pids = [spawn_link(fun() ->
                Parent ! {self(), Fun(File)}
            end) || File <- FileList],
    [receive {Pid, Result} -> Result end || Pid <- Pids].
```

---

## ตัวอย่างจริง: Log File Processor

```erlang
%% ไฟล์: log_processor.erl
-module(log_processor).
-export([
    parse_log/1,
    analyze/1,
    filter_by_level/2,
    write_report/2
]).

-type log_entry() :: #{
    timestamp := calendar:datetime(),
    level := debug | info | warning | error,
    module := atom(),
    message := binary()
}.

%% Parse log file into structured data
-spec parse_log(file:filename()) -> {ok, [log_entry()]} | {error, term()}.
parse_log(Filename) ->
    case file:read_file(Filename) of
        {ok, Content} ->
            Lines = binary:split(Content, <<"\n">>, [global, trim_all]),
            Entries = lists:filtermap(fun parse_line/1, Lines),
            {ok, Entries};
        {error, _} = E ->
            E
    end.

%% Parse log line: "2024-01-15 10:30:45 [INFO] mymodule: message here"
parse_line(Line) ->
    Pattern = "^(\\d{4}-\\d{2}-\\d{2} \\d{2}:\\d{2}:\\d{2}) \\[(\\w+)\\] (\\w+): (.+)$",
    case re:run(Line, Pattern, [{capture, all_but_first, binary}]) of
        {match, [Timestamp, Level, Module, Message]} ->
            case parse_timestamp(Timestamp) of
                {ok, DT} ->
                    {true, #{
                        timestamp => DT,
                        level => parse_level(Level),
                        module => binary_to_existing_atom(Module),
                        message => Message
                    }};
                _ ->
                    false
            end;
        _ ->
            false
    end.

parse_timestamp(TS) ->
    case re:run(TS, "(\\d+)-(\\d+)-(\\d+) (\\d+):(\\d+):(\\d+)",
                [{capture, all_but_first, list}]) of
        {match, [Y, Mo, D, H, Mi, S]} ->
            {ok, {{list_to_integer(Y), list_to_integer(Mo), list_to_integer(D)},
                  {list_to_integer(H), list_to_integer(Mi), list_to_integer(S)}}};
        _ ->
            error
    end.

parse_level(<<"DEBUG">>)   -> debug;
parse_level(<<"INFO">>)    -> info;
parse_level(<<"WARNING">>) -> warning;
parse_level(<<"ERROR">>)   -> error;
parse_level(_)             -> info.

%% Analyze log entries
-spec analyze([log_entry()]) -> map().
analyze(Entries) ->
    #{
        total => length(Entries),
        by_level => count_by_level(Entries),
        by_module => count_by_module(Entries),
        errors => [E || #{level := error} = E <- Entries],
        time_range => get_time_range(Entries)
    }.

count_by_level(Entries) ->
    lists:foldl(
        fun(#{level := Level}, Acc) ->
            maps:update_with(Level, fun(V) -> V+1 end, 1, Acc)
        end,
        #{},
        Entries
    ).

count_by_module(Entries) ->
    lists:foldl(
        fun(#{module := Mod}, Acc) ->
            maps:update_with(Mod, fun(V) -> V+1 end, 1, Acc)
        end,
        #{},
        Entries
    ).

get_time_range([]) -> undefined;
get_time_range(Entries) ->
    Timestamps = [maps:get(timestamp, E) || E <- Entries],
    {lists:min(Timestamps), lists:max(Timestamps)}.

%% Filter entries by log level
-spec filter_by_level([log_entry()], atom()) -> [log_entry()].
filter_by_level(Entries, Level) ->
    Levels = level_order(Level),
    [E || #{level := L} = E <- Entries, lists:member(L, Levels)].

level_order(debug)   -> [debug, info, warning, error];
level_order(info)    -> [info, warning, error];
level_order(warning) -> [warning, error];
level_order(error)   -> [error].

%% Write analysis report
-spec write_report(map(), file:filename()) -> ok | {error, term()}.
write_report(Analysis, Filename) ->
    Content = format_report(Analysis),
    file:write_file(Filename, Content).

format_report(#{total := Total, by_level := ByLevel, errors := Errors} = Analysis) ->
    TimeRange = case maps:get(time_range, Analysis) of
        undefined -> "N/A";
        {Start, End} ->
            io_lib:format("~p to ~p", [Start, End])
    end,
    iolist_to_binary([
        "=== Log Analysis Report ===\n",
        io_lib:format("Total entries: ~p~n", [Total]),
        io_lib:format("Time range: ~s~n", [TimeRange]),
        "\n--- By Level ---\n",
        [io_lib:format("  ~p: ~p~n", [Level, Count])
         || {Level, Count} <- lists:sort(maps:to_list(ByLevel))],
        "\n--- Errors ---\n",
        [io_lib:format("  ~p [~p]: ~s~n",
                        [maps:get(timestamp, E),
                         maps:get(module, E),
                         maps:get(message, E)])
         || E <- lists:sublist(Errors, 10)],
        case length(Errors) > 10 of
            true -> io_lib:format("  ... and ~p more errors~n", [length(Errors) - 10]);
            false -> ""
        end
    ]).
```

```erlang
%% ใช้งาน
{ok, Entries} = log_processor:parse_log("/var/log/myapp.log").
Analysis = log_processor:analyze(Entries).

%% Errors only
Errors = log_processor:filter_by_level(Entries, error).
length(Errors).

%% Write report
log_processor:write_report(Analysis, "/tmp/log_report.txt").
```

---

## สรุป Part 20

| Operation | Function | Notes |
|-----------|----------|-------|
| Read all | `file:read_file/1` | Returns binary |
| Read terms | `file:consult/1` | Erlang terms |
| Write all | `file:write_file/3` | Supports iolist |
| Append | `file:write_file/3, [append]` | |
| Open | `file:open/2` | For complex ops |
| Close | `file:close/1` | Always close! |
| List dir | `file:list_dir/1` | |
| Make dir | `file:make_dir/1` | |
| Delete | `file:delete/1` | |
| Copy | `file:copy/2` | |
| Move | `file:rename/2` | |
| Join path | `filename:join/1` | |
| Wildcard | `filelib:wildcard/1` | Glob patterns |
| Ensure dir | `filelib:ensure_dir/1` | Create parents |

---

*[← Part 19: Exceptions](part_19_exceptions.md) | [Part 21: Processes →](part_21_processes.md)*
