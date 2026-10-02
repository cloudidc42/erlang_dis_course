# Part 42: NIFs และ Ports — Integration กับ Native Code

## สารบัญ
1. [NIFs (Native Implemented Functions)](#nifs-native-implemented-functions)
2. [Port Driver](#port-driver)
3. [Port Programs (External Processes)](#port-programs-external-processes)
4. [C Nodes](#c-nodes)
5. [ตัวอย่างจริง: NIF สำหรับ Image Processing](#ตัวอย่างจริง-nif-สำหรับ-image-processing)

---

## NIFs (Native Implemented Functions)

```c
/* nif_example.c — ตัวอย่าง NIF */
#include <erl_nif.h>
#include <string.h>

/* NIF function: เพิ่ม 2 integers */
static ERL_NIF_TERM add_nif(ErlNifEnv* env, int argc, const ERL_NIF_TERM argv[]) {
    int a, b;
    if (!enif_get_int(env, argv[0], &a) || !enif_get_int(env, argv[1], &b)) {
        return enif_make_badarg(env);
    }
    return enif_make_int(env, a + b);
}

/* NIF function: reverse a binary */
static ERL_NIF_TERM reverse_binary_nif(ErlNifEnv* env, int argc, const ERL_NIF_TERM argv[]) {
    ErlNifBinary bin;
    if (!enif_inspect_binary(env, argv[0], &bin)) {
        return enif_make_badarg(env);
    }
    
    ErlNifBinary result;
    enif_alloc_binary(bin.size, &result);
    
    for (size_t i = 0; i < bin.size; i++) {
        result.data[i] = bin.data[bin.size - 1 - i];
    }
    
    return enif_make_binary(env, &result);
}

/* NIF table */
static ErlNifFunc nif_funcs[] = {
    {"add", 2, add_nif, 0},
    {"reverse_binary", 1, reverse_binary_nif, 0}
};

ERL_NIF_INIT(nif_example, nif_funcs, NULL, NULL, NULL, NULL)
```

```erlang
%% Erlang wrapper module
-module(nif_example).
-export([add/2, reverse_binary/1]).
-on_load(init/0).

init() ->
    %% Load NIF shared library
    PrivDir = code:priv_dir(my_app),
    erlang:load_nif(filename:join(PrivDir, "nif_example"), 0).

%% Fallback if NIF fails to load
add(_A, _B) ->
    error(nif_not_loaded).

reverse_binary(_Bin) ->
    error(nif_not_loaded).
```

```erlang
%% rebar.config สำหรับ compile C code
{pre_hooks, [{"(linux|darwin|solaris)", compile, "make -C c_src"}]}.
{post_hooks, [{"(linux|darwin|solaris)", clean, "make -C c_src clean"}]}.

%% c_src/Makefile
%% CFLAGS = -O2 -fPIC -shared
%% TARGET = ../priv/nif_example.so
%% all: $(TARGET)
%% $(TARGET): nif_example.c
%%     $(CC) $(CFLAGS) -I $(ERTS_INCLUDE_DIR) -o $@ $<
```

---

## Dirty NIFs (Long-Running)

```c
/* Dirty NIF: สำหรับงานที่ใช้เวลานาน
   ใช้ dirty scheduler เพื่อไม่บล็อก normal schedulers */

static ERL_NIF_TERM heavy_compute_nif(ErlNifEnv* env, int argc, const ERL_NIF_TERM argv[]) {
    unsigned long n;
    enif_get_ulong(env, argv[0], &n);
    
    /* Simulate heavy computation */
    unsigned long result = 0;
    for (unsigned long i = 0; i < n; i++) {
        result += i * i;
    }
    
    return enif_make_ulong(env, result);
}

static ErlNifFunc nif_funcs[] = {
    {"heavy_compute", 1, heavy_compute_nif, ERL_NIF_DIRTY_JOB_CPU_BOUND}
    /* ERL_NIF_DIRTY_JOB_IO_BOUND for I/O */
};
```

```erlang
%% NIF resource types: managing C resources safely
%% Automatic cleanup via destructor

%% In C:
/* ErlNifResourceType* resource_type;

static void destructor(ErlNifEnv* env, void* obj) {
    MyResource* r = (MyResource*)obj;
    free(r->data);  // cleanup
}

static int load(ErlNifEnv* env, void** priv, ERL_NIF_TERM info) {
    resource_type = enif_open_resource_type(env, NULL, "my_resource",
        destructor, ERL_NIF_RT_CREATE, NULL);
    return 0;
}

static ERL_NIF_TERM create_resource_nif(ErlNifEnv* env, int argc, const ERL_NIF_TERM argv[]) {
    MyResource* r = enif_alloc_resource(resource_type, sizeof(MyResource));
    r->data = malloc(1024);  // allocate
    ERL_NIF_TERM term = enif_make_resource(env, r);
    enif_release_resource(r);  // Erlang now owns it
    return term;
} */

%% NIFs ที่ควรหลีกเลี่ยง:
%% 1. NIF ที่ block > 1ms โดยไม่ใช้ dirty scheduler
%% 2. NIF ที่ allocate memory ขนาดใหญ่บน Erlang heap
%% 3. NIF ที่ไม่ handle errors อย่างถูกต้อง
%% 4. NIF ที่เรียก enif_* จาก thread ที่ไม่ใช่ NIF env
```

---

## Port Driver

```c
/* Port driver: หนักกว่า NIF แต่ crash-safe
   crash ใน driver ไม่ crash VM */

#include "erl_driver.h"
#include <string.h>

typedef struct {
    ErlDrvPort port;
    int count;
} DriverState;

static ErlDrvData start(ErlDrvPort port, char* command) {
    DriverState* state = driver_alloc(sizeof(DriverState));
    state->port = port;
    state->count = 0;
    return (ErlDrvData)state;
}

static void stop(ErlDrvData handle) {
    driver_free((char*)handle);
}

static void output(ErlDrvData handle, char* buf, ErlDrvSizeT len) {
    DriverState* state = (DriverState*)handle;
    state->count += len;
    
    /* Echo back with count */
    char response[64];
    int rlen = snprintf(response, sizeof(response), "count:%d", state->count);
    driver_output(state->port, response, rlen);
}

static ErlDrvEntry driver_entry = {
    .driver_name = "my_driver",
    .start = start,
    .stop = stop,
    .output = output,
    .extended_marker = ERL_DRV_EXTENDED_MARKER,
    .major_version = ERL_DRV_EXTENDED_MAJOR_VERSION,
    .minor_version = ERL_DRV_EXTENDED_MINOR_VERSION
};

DRIVER_INIT(my_driver) {
    return &driver_entry;
}
```

```erlang
%% Erlang side: using port driver
-module(my_driver).
-export([start/0, send/2, stop/1]).

start() ->
    erl_ddll:load_driver(code:priv_dir(my_app), "my_driver"),
    open_port({spawn, "my_driver"}, [binary]).

send(Port, Data) ->
    port_command(Port, Data),
    receive
        {Port, {data, Response}} -> Response
    after 5000 ->
        error(timeout)
    end.

stop(Port) ->
    port_close(Port).
```

---

## Port Programs (External Processes)

```erlang
%% Port program: ปลอดภัยที่สุด — ทำงานใน separate OS process
%% crash ใน external process ไม่กระทบ Erlang VM

%% เปิด port ไปยัง external program
start_port() ->
    open_port({spawn, "python3 my_script.py"}, [
        binary,
        {packet, 4},  %% length-prefixed packets
        use_stdio,
        exit_status
    ]).

%% Send/receive
call_port(Port, Request) ->
    Encoded = encode_request(Request),
    port_command(Port, Encoded),
    receive
        {Port, {data, Data}} ->
            decode_response(Data);
        {Port, {exit_status, Status}} ->
            {error, {exited, Status}}
    after 10000 ->
        {error, timeout}
    end.

encode_request({method, Args}) ->
    jsx:encode(#{method => method, args => Args}).

decode_response(Data) ->
    jsx:decode(Data, [return_maps]).

%% Port options:
%% {spawn_executable, Path}  — absolute path, safer
%% {args, [...]}             — argument list (no shell injection)
%% {env, [{"VAR", "val"}]}  — environment variables
%% {cd, Dir}                 — working directory
%% stream | {packet, N}      — data framing
%% binary | list             — data format
%% exit_status               — receive {exit_status, N}
```

```python
# Python side: my_script.py
import sys
import struct
import json

def read_msg():
    """Read 4-byte length-prefixed message"""
    header = sys.stdin.buffer.read(4)
    if not header:
        return None
    length = struct.unpack('>I', header)[0]
    return json.loads(sys.stdin.buffer.read(length))

def write_msg(data):
    """Write 4-byte length-prefixed message"""
    encoded = json.dumps(data).encode()
    sys.stdout.buffer.write(struct.pack('>I', len(encoded)))
    sys.stdout.buffer.write(encoded)
    sys.stdout.buffer.flush()

while True:
    msg = read_msg()
    if msg is None:
        break
    method = msg.get('method')
    args = msg.get('args', [])
    
    if method == 'process':
        result = {'ok': True, 'data': args[0] * 2}
    else:
        result = {'error': 'unknown_method'}
    
    write_msg(result)
```

---

## C Nodes

```c
/* C Node: external C process ที่ act เหมือน Erlang node
   ใช้ ei library (Erlang Interface) */

#include "ei.h"
#include <sys/socket.h>
#include <netdb.h>

int main() {
    ei_cnode ec;
    struct in_addr addr;
    inet_aton("127.0.0.1", &addr);
    
    /* Initialize C node */
    ei_connect_init(&ec, "cnode", "secret_cookie", 0);
    
    /* Accept connections from Erlang */
    int port = 12345;
    int listen_fd = ei_listen(&ec, &port, 5);
    
    ErlConnect conn;
    int fd = ei_accept(&ec, listen_fd, &conn);
    
    /* Receive and handle messages */
    ei_x_buff buf;
    ei_x_new(&buf);
    
    while (1) {
        erlang_msg msg;
        int got = ei_xreceive_msg(fd, &msg, &buf);
        
        if (got == ERL_MSG) {
            /* Decode and respond */
            int index = 0;
            int arity;
            char atom[256];
            
            ei_decode_version(buf.buff, &index, NULL);
            ei_decode_tuple_header(buf.buff, &index, &arity);
            ei_decode_atom(buf.buff, &index, atom);
            
            /* Build response */
            ei_x_buff reply;
            ei_x_new_with_version(&reply);
            ei_x_encode_atom(&reply, "ok");
            
            ei_send(fd, &msg.from, reply.buff, reply.buffsz);
            ei_x_free(&reply);
        }
    }
}
```

```erlang
%% Erlang side: communicate with C node
connect_cnode() ->
    Node = 'cnode@localhost',
    net_adm:ping(Node),
    {any, Node} ! {self(), hello},
    receive
        Response -> Response
    after 5000 ->
        timeout
    end.
```

---

## ตัวอย่างจริง: NIF สำหรับ Image Processing

```c
/* image_nif.c: fast pixel manipulation via NIF */
#include <erl_nif.h>
#include <stdint.h>
#include <string.h>

/* Convert RGB to grayscale */
static ERL_NIF_TERM to_grayscale_nif(ErlNifEnv* env, int argc, const ERL_NIF_TERM argv[]) {
    ErlNifBinary input;
    if (!enif_inspect_binary(env, argv[0], &input)) {
        return enif_make_badarg(env);
    }
    
    if (input.size % 3 != 0) {
        return enif_make_badarg(env);
    }
    
    size_t pixel_count = input.size / 3;
    ErlNifBinary output;
    enif_alloc_binary(pixel_count, &output);
    
    for (size_t i = 0; i < pixel_count; i++) {
        uint8_t r = input.data[i*3];
        uint8_t g = input.data[i*3+1];
        uint8_t b = input.data[i*3+2];
        /* ITU-R BT.601 luminance formula */
        output.data[i] = (uint8_t)(0.299*r + 0.587*g + 0.114*b);
    }
    
    return enif_make_binary(env, &output);
}

/* Apply brightness adjustment */
static ERL_NIF_TERM adjust_brightness_nif(ErlNifEnv* env, int argc, const ERL_NIF_TERM argv[]) {
    ErlNifBinary input;
    int delta;
    
    if (!enif_inspect_binary(env, argv[0], &input) ||
        !enif_get_int(env, argv[1], &delta)) {
        return enif_make_badarg(env);
    }
    
    ErlNifBinary output;
    enif_alloc_binary(input.size, &output);
    
    for (size_t i = 0; i < input.size; i++) {
        int val = (int)input.data[i] + delta;
        if (val < 0) val = 0;
        if (val > 255) val = 255;
        output.data[i] = (uint8_t)val;
    }
    
    return enif_make_binary(env, &output);
}

static ErlNifFunc nif_funcs[] = {
    {"to_grayscale", 1, to_grayscale_nif, ERL_NIF_DIRTY_JOB_CPU_BOUND},
    {"adjust_brightness", 2, adjust_brightness_nif, ERL_NIF_DIRTY_JOB_CPU_BOUND}
};

ERL_NIF_INIT(image_nif, nif_funcs, NULL, NULL, NULL, NULL)
```

```erlang
%% image_nif.erl
-module(image_nif).
-export([to_grayscale/1, adjust_brightness/2]).
-on_load(init/0).

init() ->
    PrivDir = code:priv_dir(my_app),
    Path = filename:join(PrivDir, "image_nif"),
    erlang:load_nif(Path, 0).

to_grayscale(_RgbBinary) -> error(nif_not_loaded).
adjust_brightness(_Binary, _Delta) -> error(nif_not_loaded).

%% High-level API
process_image(RgbBinary, Options) ->
    Steps = maps:get(steps, Options, []),
    lists:foldl(fun apply_step/2, RgbBinary, Steps).

apply_step(grayscale, Bin) -> to_grayscale(Bin);
apply_step({brightness, D}, Bin) -> adjust_brightness(Bin, D);
apply_step(_, Bin) -> Bin.
```

---

## สรุป Part 42

| Method | Safety | Performance | Use Case |
|--------|--------|-------------|----------|
| NIF | Crash VM | Fastest | CPU-bound, no blocking |
| Dirty NIF | Crash VM | Fast | Long CPU/IO |
| Port Driver | Crash safe | Fast | Complex I/O |
| Port Program | Fully safe | Slowest | External tools |
| C Node | Fully safe | Medium | Protocol integration |

**Rule of thumb**: ใช้ Port Program ก่อน → Port Driver ถ้าต้องการความเร็ว → NIF เมื่อจำเป็นจริงๆ

---

*[← Part 41: Release Management](part_41_releases.md) | [Part 43: Security →](part_43_security.md)*
