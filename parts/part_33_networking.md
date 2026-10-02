# Part 33: TCP/UDP Networking

## สารบัญ
1. [gen_tcp — TCP Sockets](#gen_tcp--tcp-sockets)
2. [TCP Server Pattern](#tcp-server-pattern)
3. [gen_udp — UDP Sockets](#gen_udp--udp-sockets)
4. [ssl — TLS/SSL](#ssl--tlsssl)
5. [inet — Network Utilities](#inet--network-utilities)
6. [ตัวอย่างจริง: Echo Server](#ตัวอย่างจริง-echo-server)
7. [ตัวอย่างจริง: TCP Chat Server](#ตัวอย่างจริง-tcp-chat-server)

---

## gen_tcp — TCP Sockets

```erlang
%% gen_tcp: Erlang TCP socket module

%% ====== TCP Server ======

%% 1. เปิด listen socket
{ok, ListenSocket} = gen_tcp:listen(Port, Options).

%% Options สำคัญ:
%% binary           - รับ data เป็น binary (แนะนำ)
%% {active, false}  - pull mode: ต้องเรียก gen_tcp:recv
%% {active, true}   - push mode: ส่ง {tcp, S, Data} message
%% {active, N}      - hybrid: ส่ง N messages แล้ว passive
%% {active, once}   - ส่ง 1 message แล้ว passive
%% {packet, raw}    - no framing (default)
%% {packet, line}   - line-based (\n)
%% {packet, 4}      - 4-byte length prefix
%% {reuseaddr, true} - reuse port (prevent "address already in use")
%% {backlog, N}     - listen queue size
%% {keepalive, true} - TCP keepalive
%% {nodelay, true}  - disable Nagle algorithm (low latency)

{ok, ListenSocket} = gen_tcp:listen(8080, [
    binary,
    {active, false},
    {reuseaddr, true},
    {packet, line}
]).

%% 2. รอ connection
{ok, Socket} = gen_tcp:accept(ListenSocket).

%% 3. รับ data
{ok, Data} = gen_tcp:recv(Socket, 0).       %% any size
{ok, Data} = gen_tcp:recv(Socket, 0, 5000). %% timeout 5s

%% 4. ส่ง data
ok = gen_tcp:send(Socket, <<"Hello!\n">>).
ok = gen_tcp:send(Socket, [<<"Part1">>, <<" Part2">>]).  %% iolist

%% 5. ปิด connection
ok = gen_tcp:close(Socket).
ok = gen_tcp:close(ListenSocket).

%% ====== TCP Client ======
{ok, Socket} = gen_tcp:connect("localhost", 8080, [
    binary,
    {active, false},
    {packet, line}
]).

ok = gen_tcp:send(Socket, <<"Hello server\n">>).
{ok, Reply} = gen_tcp:recv(Socket, 0, 5000).
gen_tcp:close(Socket).
```

---

## TCP Server Pattern

```erlang
%% Pattern: Acceptor + per-connection handler

-module(tcp_server).
-export([start/1, start_link/1]).

start(Port) ->
    spawn(fun() -> listen(Port) end).

start_link(Port) ->
    spawn_link(fun() -> listen(Port) end).

listen(Port) ->
    {ok, Sock} = gen_tcp:listen(Port, [
        binary, {active, false}, {reuseaddr, true}, {packet, raw}
    ]),
    accept_loop(Sock).

accept_loop(ListenSock) ->
    {ok, ClientSock} = gen_tcp:accept(ListenSock),
    %% Spawn handler, pass socket
    Pid = spawn(fun() -> handle_client(ClientSock) end),
    %% Transfer socket ownership to handler process
    gen_tcp:controlling_process(ClientSock, Pid),
    accept_loop(ListenSock).

handle_client(Sock) ->
    %% Switch to active mode after ownership transfer
    inet:setopts(Sock, [{active, once}]),
    receive
        {tcp, Sock, Data} ->
            Response = process_request(Data),
            gen_tcp:send(Sock, Response),
            inet:setopts(Sock, [{active, once}]),
            handle_client(Sock);
        {tcp_closed, Sock} ->
            io:format("Client disconnected~n"),
            gen_tcp:close(Sock);
        {tcp_error, Sock, Reason} ->
            io:format("Socket error: ~p~n", [Reason]),
            gen_tcp:close(Sock)
    after 30000 ->
        io:format("Client timeout~n"),
        gen_tcp:close(Sock)
    end.

process_request(Data) ->
    %% Echo back
    Data.
```

---

## gen_udp — UDP Sockets

```erlang
%% gen_udp: ส่ง/รับ UDP datagrams

%% Open UDP socket
{ok, Socket} = gen_udp:open(Port, [binary, {active, false}]).
{ok, Socket} = gen_udp:open(0).  %% OS assigns port

%% Send UDP datagram
ok = gen_udp:send(Socket, "localhost", 9090, <<"hello">>).
ok = gen_udp:send(Socket, {127,0,0,1}, 9090, <<"hello">>).

%% Receive (pull mode)
{ok, {SourceIP, SourcePort, Data}} = gen_udp:recv(Socket, 0).
{ok, {SourceIP, SourcePort, Data}} = gen_udp:recv(Socket, 0, 5000).

%% Active mode: push to mailbox
{ok, Socket} = gen_udp:open(9090, [binary, {active, true}]).
receive
    {udp, Socket, SourceIP, SourcePort, Data} ->
        handle_datagram(SourceIP, SourcePort, Data)
end.

%% Close
gen_udp:close(Socket).

%% Multicast
{ok, Socket} = gen_udp:open(MulticastPort, [
    binary,
    {active, true},
    {reuseaddr, true},
    {add_membership, {MulticastIP, {0,0,0,0}}}  %% join group
]).

%% Broadcast
{ok, Socket} = gen_udp:open(0, [binary, {broadcast, true}]).
gen_udp:send(Socket, {255,255,255,255}, Port, <<"broadcast">>).

%% ตัวอย่าง UDP logger
-module(udp_logger).

start() ->
    {ok, Sock} = gen_udp:open(5140, [binary, {active, true}]),
    loop(Sock).

loop(Sock) ->
    receive
        {udp, Sock, IP, Port, Data} ->
            io:format("[~p:~p] ~s~n", [IP, Port, Data]),
            loop(Sock)
    end.
```

---

## ssl — TLS/SSL

```erlang
%% ssl: TLS/SSL on top of TCP

%% Start ssl application
ssl:start().

%% TLS Server
{ok, ListenSocket} = ssl:listen(8443, [
    {certfile, "server.crt"},
    {keyfile, "server.key"},
    {cacertfile, "ca.crt"},
    {verify, verify_peer},
    {versions, ['tlsv1.2', 'tlsv1.3']},
    {ciphers, ssl:cipher_suites(strong, 'tlsv1.3')},
    binary,
    {active, false},
    {reuseaddr, true}
]).

{ok, TLSSocket} = ssl:transport_accept(ListenSocket).
ok = ssl:handshake(TLSSocket).

{ok, Data} = ssl:recv(TLSSocket, 0).
ssl:send(TLSSocket, <<"Response">>).
ssl:close(TLSSocket).

%% TLS Client
{ok, Socket} = ssl:connect("example.com", 443, [
    {verify, verify_peer},
    {cacertfile, "/etc/ssl/certs/ca-certificates.crt"},
    {server_name_indication, "example.com"},
    {versions, ['tlsv1.2', 'tlsv1.3']},
    binary,
    {active, false}
]).

ssl:send(Socket, <<"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n">>).
{ok, Response} = ssl:recv(Socket, 0, 5000).
ssl:close(Socket).

%% Self-signed certificate for development
%% $ openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
```

---

## inet — Network Utilities

```erlang
%% inet: network utilities

%% DNS lookup
{ok, {A,B,C,D}} = inet:getaddr("example.com", inet).
%% {ok, {93,184,216,34}}

{ok, Addrs} = inet:getaddrs("example.com", inet).
%% {ok, [{93,184,216,34}]}

%% Reverse DNS
{ok, Hostname} = inet:gethostbyaddr({93,184,216,34}).

%% Socket options (setopt/getopt after creating socket)
inet:setopts(Socket, [{active, once}, {nodelay, true}]).
{ok, Opts} = inet:getopts(Socket, [active, packet]).

%% Socket info
{ok, {LocalIP, LocalPort}} = inet:sockname(Socket).
{ok, {PeerIP, PeerPort}} = inet:peername(Socket).
{ok, Stats} = inet:getstat(Socket).
{ok, Stats} = inet:getstat(Socket, [recv_cnt, send_cnt, recv_oct, send_oct]).

%% IP address parsing
{ok, {127,0,0,1}} = inet:parse_address("127.0.0.1").
{ok, {0,0,0,0,0,0,0,1}} = inet:parse_address("::1").  %% IPv6

%% Interface info
{ok, Interfaces} = inet:getifaddrs().
%% [{"lo", [...flags, {addr, {127,0,0,1}}, ...]}, {"eth0", ...}]

%% Hostname
{ok, Hostname} = inet:gethostname().
```

---

## ตัวอย่างจริง: Echo Server

```erlang
%% ไฟล์: echo_server.erl
-module(echo_server).
-behaviour(gen_server).

-export([start_link/1, stop/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
         terminate/2, code_change/3]).

-record(state, {
    listen_sock :: gen_tcp:socket(),
    acceptor :: pid(),
    connections = #{} :: #{pid() => gen_tcp:socket()}
}).

start_link(Port) ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [Port], []).

stop() ->
    gen_server:stop(?MODULE).

init([Port]) ->
    {ok, LSock} = gen_tcp:listen(Port, [
        binary, {active, false}, {reuseaddr, true},
        {packet, line}, {keepalive, true}
    ]),
    io:format("Echo server listening on port ~p~n", [Port]),
    Self = self(),
    Acceptor = spawn_link(fun() -> accept_loop(LSock, Self) end),
    {ok, #state{listen_sock = LSock, acceptor = Acceptor}}.

accept_loop(LSock, Server) ->
    case gen_tcp:accept(LSock) of
        {ok, Sock} ->
            gen_server:cast(Server, {new_connection, Sock}),
            accept_loop(LSock, Server);
        {error, closed} ->
            ok
    end.

handle_cast({new_connection, Sock}, #state{connections = Conns} = State) ->
    Self = self(),
    Handler = spawn_link(fun() -> handle_conn(Sock, Self) end),
    gen_tcp:controlling_process(Sock, Handler),
    Handler ! start,
    {noreply, State#state{connections = Conns#{Handler => Sock}}};

handle_cast({connection_closed, Pid}, #state{connections = Conns} = State) ->
    {noreply, State#state{connections = maps:remove(Pid, Conns)}};

handle_cast(_, State) -> {noreply, State}.

handle_call(stats, _From, #state{connections = C} = State) ->
    {reply, #{active_connections => map_size(C)}, State}.

handle_info({'EXIT', Pid, _Reason}, #state{connections = Conns} = State) ->
    {noreply, State#state{connections = maps:remove(Pid, Conns)}};

handle_info(_, State) -> {noreply, State}.

terminate(_, #state{listen_sock = LSock}) ->
    gen_tcp:close(LSock).

code_change(_, S, _) -> {ok, S}.

handle_conn(Sock, Server) ->
    receive start -> ok end,
    inet:setopts(Sock, [{active, once}]),
    conn_loop(Sock, Server).

conn_loop(Sock, Server) ->
    receive
        {tcp, Sock, Data} ->
            Upper = string:uppercase(Data),
            gen_tcp:send(Sock, Upper),
            inet:setopts(Sock, [{active, once}]),
            conn_loop(Sock, Server);
        {tcp_closed, Sock} ->
            gen_server:cast(Server, {connection_closed, self()});
        {tcp_error, Sock, _} ->
            gen_tcp:close(Sock),
            gen_server:cast(Server, {connection_closed, self()})
    after 60000 ->
        gen_tcp:close(Sock),
        gen_server:cast(Server, {connection_closed, self()})
    end.
```

---

## ตัวอย่างจริง: TCP Chat Server

```erlang
%% ไฟล์: chat_server.erl
-module(chat_server).
-behaviour(gen_server).

-export([start_link/1, broadcast/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
         terminate/2, code_change/3]).

-record(state, {
    listen_sock,
    clients = #{} :: #{pid() => #{nick => binary(), sock => term()}}
}).

start_link(Port) ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [Port], []).

broadcast(Sender, Message) ->
    gen_server:cast(?MODULE, {broadcast, Sender, Message}).

init([Port]) ->
    {ok, LSock} = gen_tcp:listen(Port, [binary, {active, false}, {reuseaddr, true}, {packet, line}]),
    Self = self(),
    spawn_link(fun() -> acceptor(LSock, Self) end),
    io:format("Chat server on port ~p~n", [Port]),
    {ok, #state{listen_sock = LSock}}.

acceptor(LSock, Server) ->
    {ok, Sock} = gen_tcp:accept(LSock),
    gen_server:cast(Server, {join, Sock}),
    acceptor(LSock, Server).

handle_cast({join, Sock}, #state{clients = C} = State) ->
    gen_tcp:send(Sock, <<"Enter your nickname: \n">>),
    Self = self(),
    Handler = spawn_link(fun() -> client_handler(Sock, Self) end),
    gen_tcp:controlling_process(Sock, Handler),
    Handler ! start,
    Nick = <<"anon", (integer_to_binary(map_size(C)))/binary>>,
    {noreply, State#state{clients = C#{Handler => #{nick => Nick, sock => Sock}}}};

handle_cast({set_nick, Pid, Nick}, #state{clients = C} = State) ->
    case maps:get(Pid, C, undefined) of
        undefined -> {noreply, State};
        Info ->
            OldNick = maps:get(nick, Info),
            Msg = <<OldNick/binary, " is now known as ", Nick/binary, "\n">>,
            do_broadcast(Msg, State),
            {noreply, State#state{clients = C#{Pid => Info#{nick => Nick}}}}
    end;

handle_cast({broadcast, Pid, Msg}, #state{clients = C} = State) ->
    case maps:get(Pid, C, undefined) of
        #{nick := Nick} ->
            Full = <<"[", Nick/binary, "] ", Msg/binary>>,
            do_broadcast(Full, State),
            {noreply, State};
        undefined ->
            {noreply, State}
    end;

handle_cast({leave, Pid}, #state{clients = C} = State) ->
    case maps:get(Pid, C, undefined) of
        #{nick := Nick} ->
            do_broadcast(<<Nick/binary, " left the chat\n">>, State),
            {noreply, State#state{clients = maps:remove(Pid, C)}};
        undefined ->
            {noreply, State}
    end.

handle_call(_, _, State) -> {reply, ok, State}.
handle_info({'EXIT', Pid, _}, #state{clients = C} = State) ->
    {noreply, State#state{clients = maps:remove(Pid, C)}}.
handle_info(_, State) -> {noreply, State}.
terminate(_, #state{listen_sock = S}) -> gen_tcp:close(S).
code_change(_, S, _) -> {ok, S}.

do_broadcast(Msg, #state{clients = C}) ->
    [gen_tcp:send(maps:get(sock, Info), Msg) || Info <- maps:values(C)].

%% Per-client handler
client_handler(Sock, Server) ->
    receive start -> ok end,
    inet:setopts(Sock, [{active, once}]),
    client_loop(Sock, Server).

client_loop(Sock, Server) ->
    receive
        {tcp, Sock, <<"/nick ", Rest/binary>>} ->
            Nick = string:trim(Rest),
            gen_server:cast(Server, {set_nick, self(), Nick}),
            inet:setopts(Sock, [{active, once}]),
            client_loop(Sock, Server);
        {tcp, Sock, Data} ->
            Msg = string:trim(Data),
            gen_server:cast(Server, {broadcast, self(), Msg}),
            inet:setopts(Sock, [{active, once}]),
            client_loop(Sock, Server);
        {tcp_closed, Sock} ->
            gen_server:cast(Server, {leave, self()});
        {tcp_error, Sock, _} ->
            gen_tcp:close(Sock),
            gen_server:cast(Server, {leave, self()})
    end.
```

```bash
# รัน chat server
$ erl -eval "chat_server:start_link(4000)."

# Connect (terminal 1)
$ telnet localhost 4000
Enter your nickname:
/nick Alice
[Alice] hello world!

# Connect (terminal 2)
$ telnet localhost 4000
/nick Bob
[Alice] hello world!
[Bob] hi Alice!
```

---

## สรุป Part 33

| Module | Protocol | ใช้เมื่อ |
|--------|---------|---------|
| `gen_tcp` | TCP | Reliable, ordered streams |
| `gen_udp` | UDP | Low-latency, stateless |
| `ssl` | TLS/SSL | Encrypted TCP |
| `inet` | Utilities | DNS, IP, socket ops |

| Option | Effect |
|--------|--------|
| `{active, false}` | Pull mode, `recv/2,3` |
| `{active, true}` | Push all to mailbox |
| `{active, once}` | Push one, then passive |
| `{packet, line}` | Line-based framing |
| `{packet, N}` | N-byte length prefix |

---

*[← Part 32: Distributed Patterns](part_32_distributed_patterns.md) | [Part 34: HTTP Client →](part_34_http_client.md)*
