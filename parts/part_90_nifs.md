# Part 90: NIFs and Native Code

## สารบัญ
1. [NIF Basics](#basics)
2. [Writing Safe NIFs](#safe-nifs)
3. [Dirty Schedulers](#dirty)
4. [Enif Resource Objects](#resources)
5. [Port Drivers](#port-drivers)
6. [ตัวอย่างจริง: High-Performance NIF](#example)

---

## NIF Basics

```c
/* my_nif.c — Native Implemented Functions in C */

#include <erl_nif.h>
#include <string.h>
#include <stdlib.h>

/* Simple NIF: add two integers */
static ERL_NIF_TERM add_nif(ErlNifEnv* env, int argc, const ERL_NIF_TERM argv[]) {
    int a, b;
    
    if (!enif_get_int(env, argv[0], &a) ||
        !enif_get_int(env, argv[1], &b)) {
        return enif_make_badarg(env);
    }
    
    return enif_make_int(env, a + b);
}

/* NIF: count bytes matching a pattern */
static ERL_NIF_TERM count_bytes_nif(ErlNifEnv* env, int argc, const ERL_NIF_TERM argv[]) {
    ErlNifBinary bin;
    int target_byte;
    long count = 0;
    
    if (!enif_inspect_binary(env, argv[0], &bin) ||
        !enif_get_int(env, argv[1], &target_byte)) {
        return enif_make_badarg(env);
    }
    
    for (size_t i = 0; i < bin.size; i++) {
        if (bin.data[i] == (unsigned char)target_byte) {
            count++;
        }
    }
    
    return enif_make_long(env, count);
}

/* Register NIFs */
static ErlNifFunc nif_funcs[] = {
    {"add",         2, add_nif,         0},
    {"count_bytes", 2, count_bytes_nif, 0}
};

ERL_NIF_INIT(my_nif, nif_funcs, NULL, NULL, NULL, NULL)
```

```erlang
%% my_nif.erl — Erlang wrapper

-module(my_nif).
-export([add/2, count_bytes/2]).
-on_load(init/0).

init() ->
    %% Load the .so shared library
    SoPath = filename:join(code:priv_dir(my_app), "my_nif"),
    erlang:load_nif(SoPath, 0).

%% Stub functions (replaced by NIF at runtime)
add(_A, _B) ->
    erlang:nif_error(nif_not_loaded).

count_bytes(_Binary, _Byte) ->
    erlang:nif_error(nif_not_loaded).
```

```makefile
# c_src/Makefile
CC = gcc
CFLAGS = -fPIC -shared -O2 -Wall
ERL_INCLUDE = $(shell erl -eval 'io:format("~s", [code:root_dir()])' -noshell -s init stop)/erts-$(shell erl -eval 'io:format("~s", [erlang:system_info(version)])' -noshell -s init stop)/include

my_nif.so: my_nif.c
    $(CC) $(CFLAGS) -I$(ERL_INCLUDE) -o ../priv/my_nif.so my_nif.c
```

```erlang
%% rebar.config for NIF compilation
nif_rebar_config() ->
    [{pre_hooks, [
        {compile, "make -C c_src"}
    ]},
     {post_hooks, [
        {clean, "make -C c_src clean"}
     ]}].
```

---

## Writing Safe NIFs

```c
/* Critical NIF rules to avoid crashing the VM */

/* 1. NIFs MUST return quickly (< 1ms) */
/* For long operations, use dirty schedulers or ports */

/* 2. Never call blocking syscalls in NIFs */
/* No: sleep(), malloc large, file I/O without dirty */

/* 3. Correct memory management */
static ERL_NIF_TERM safe_binary_nif(ErlNifEnv* env, int argc, const ERL_NIF_TERM argv[]) {
    ErlNifBinary input, output;
    
    if (!enif_inspect_binary(env, argv[0], &input)) {
        return enif_make_badarg(env);
    }
    
    /* Allocate output binary - automatically freed if we return an error */
    if (!enif_alloc_binary(input.size, &output)) {
        return enif_make_atom(env, "enomem");
    }
    
    /* Process */
    for (size_t i = 0; i < input.size; i++) {
        output.data[i] = input.data[i] ^ 0xFF;  /* XOR each byte */
    }
    
    /* enif_make_binary transfers ownership - no need to free */
    return enif_make_binary(env, &output);
}

/* 4. Handle all error cases */
static ERL_NIF_TERM safe_parse_nif(ErlNifEnv* env, int argc, const ERL_NIF_TERM argv[]) {
    char buf[256];
    int len;
    
    /* Validate input */
    if (!enif_is_binary(env, argv[0])) {
        return enif_make_tuple2(env,
            enif_make_atom(env, "error"),
            enif_make_atom(env, "not_binary"));
    }
    
    len = enif_get_string(env, argv[0], buf, sizeof(buf), ERL_NIF_LATIN1);
    if (len <= 0) {
        return enif_make_tuple2(env,
            enif_make_atom(env, "error"),
            enif_make_atom(env, "string_too_long"));
    }
    
    /* Process buf... */
    return enif_make_tuple2(env,
        enif_make_atom(env, "ok"),
        enif_make_string(env, buf, ERL_NIF_LATIN1));
}

/* 5. Thread-safe atom caching */
static ERL_NIF_TERM atom_ok;
static ERL_NIF_TERM atom_error;

static int load(ErlNifEnv* env, void** priv, ERL_NIF_TERM load_info) {
    atom_ok = enif_make_atom(env, "ok");
    atom_error = enif_make_atom(env, "error");
    return 0;
}

ERL_NIF_INIT(my_safe_nif, nif_funcs, load, NULL, NULL, NULL)
```

---

## Dirty Schedulers

```c
/* For CPU-intensive NIFs: use dirty CPU scheduler */
/* For I/O-blocking NIFs: use dirty I/O scheduler */

#include <erl_nif.h>
#include <unistd.h>   /* for sleep */

/* CPU-intensive operation */
static ERL_NIF_TERM heavy_compute_nif(ErlNifEnv* env, int argc, const ERL_NIF_TERM argv[]) {
    unsigned long n;
    
    if (!enif_get_ulong(env, argv[0], &n)) {
        return enif_make_badarg(env);
    }
    
    /* CPU-intensive: compute prime sieve up to n */
    unsigned char* sieve = calloc(n + 1, 1);
    sieve[0] = sieve[1] = 1;
    
    for (unsigned long i = 2; i * i <= n; i++) {
        if (!sieve[i]) {
            for (unsigned long j = i * i; j <= n; j += i) {
                sieve[j] = 1;
            }
        }
    }
    
    long prime_count = 0;
    for (unsigned long i = 2; i <= n; i++) {
        if (!sieve[i]) prime_count++;
    }
    
    free(sieve);
    
    return enif_make_long(env, prime_count);
}

/* I/O blocking operation */
static ERL_NIF_TERM read_file_nif(ErlNifEnv* env, int argc, const ERL_NIF_TERM argv[]) {
    char path[4096];
    
    if (enif_get_string(env, argv[0], path, sizeof(path), ERL_NIF_LATIN1) <= 0) {
        return enif_make_badarg(env);
    }
    
    /* This would block — use dirty I/O scheduler */
    FILE* f = fopen(path, "rb");
    if (!f) {
        return enif_make_tuple2(env, enif_make_atom(env, "error"),
                                 enif_make_atom(env, "enoent"));
    }
    
    /* ... read file ... */
    fclose(f);
    
    return enif_make_atom(env, "ok");
}

/* Register with dirty scheduler flags */
static ErlNifFunc dirty_funcs[] = {
    {"heavy_compute", 1, heavy_compute_nif, ERL_NIF_DIRTY_JOB_CPU_BOUND},
    {"read_file",     1, read_file_nif,     ERL_NIF_DIRTY_JOB_IO_BOUND}
};

ERL_NIF_INIT(dirty_nif, dirty_funcs, NULL, NULL, NULL, NULL)
```

```erlang
%% dirty_nif.erl
-module(dirty_nif).
-export([heavy_compute/1, read_file/1]).
-on_load(init/0).

init() ->
    erlang:load_nif(filename:join(code:priv_dir(my_app), "dirty_nif"), 0).

heavy_compute(_N) -> erlang:nif_error(nif_not_loaded).
read_file(_Path) -> erlang:nif_error(nif_not_loaded).

%% Usage: runs on dirty CPU scheduler, doesn't block BEAM schedulers
primes_up_to(N) ->
    heavy_compute(N).
```

---

## Enif Resource Objects

```c
/* Manage C resources with automatic cleanup */

#include <erl_nif.h>
#include <stdio.h>

/* Resource type for file handles */
static ErlNifResourceType* FILE_RES_TYPE = NULL;

/* Destructor: called when resource is GC'd */
static void file_destructor(ErlNifEnv* env, void* obj) {
    FILE** fp = (FILE**)obj;
    if (*fp != NULL) {
        fclose(*fp);
        *fp = NULL;
    }
}

static int load(ErlNifEnv* env, void** priv, ERL_NIF_TERM load_info) {
    FILE_RES_TYPE = enif_open_resource_type(
        env, NULL, "file_handle",
        file_destructor,
        ERL_NIF_RT_CREATE | ERL_NIF_RT_TAKEOVER,
        NULL
    );
    return FILE_RES_TYPE != NULL ? 0 : -1;
}

static ERL_NIF_TERM open_file_nif(ErlNifEnv* env, int argc, const ERL_NIF_TERM argv[]) {
    char path[4096];
    
    if (enif_get_string(env, argv[0], path, sizeof(path), ERL_NIF_LATIN1) <= 0) {
        return enif_make_badarg(env);
    }
    
    /* Allocate resource */
    FILE** res = enif_alloc_resource(FILE_RES_TYPE, sizeof(FILE*));
    *res = fopen(path, "r");
    
    if (!*res) {
        enif_release_resource(res);
        return enif_make_tuple2(env, enif_make_atom(env, "error"),
                                 enif_make_atom(env, "enoent"));
    }
    
    /* Transfer ownership to Erlang (GC will call destructor) */
    ERL_NIF_TERM term = enif_make_resource(env, res);
    enif_release_resource(res);  /* Release our own reference */
    
    return enif_make_tuple2(env, enif_make_atom(env, "ok"), term);
}

static ERL_NIF_TERM read_line_nif(ErlNifEnv* env, int argc, const ERL_NIF_TERM argv[]) {
    FILE** res;
    char buf[4096];
    
    if (!enif_get_resource(env, argv[0], FILE_RES_TYPE, (void**)&res)) {
        return enif_make_badarg(env);
    }
    
    if (fgets(buf, sizeof(buf), *res) == NULL) {
        return enif_make_atom(env, "eof");
    }
    
    return enif_make_tuple2(env, enif_make_atom(env, "ok"),
                             enif_make_string(env, buf, ERL_NIF_LATIN1));
}
```

---

## ตัวอย่างจริง: High-Performance NIF

```c
/* fast_json_scanner.c — fast JSON token scanner NIF */

#include <erl_nif.h>
#include <string.h>

typedef enum {
    TOKEN_OBJECT_START,
    TOKEN_OBJECT_END,
    TOKEN_ARRAY_START,
    TOKEN_ARRAY_END,
    TOKEN_STRING,
    TOKEN_NUMBER,
    TOKEN_TRUE,
    TOKEN_FALSE,
    TOKEN_NULL
} TokenType;

static ERL_NIF_TERM count_tokens_nif(ErlNifEnv* env, int argc, const ERL_NIF_TERM argv[]) {
    ErlNifBinary json;
    
    if (!enif_inspect_binary(env, argv[0], &json)) {
        return enif_make_badarg(env);
    }
    
    long token_count = 0;
    int in_string = 0;
    
    for (size_t i = 0; i < json.size; i++) {
        unsigned char c = json.data[i];
        
        if (in_string) {
            if (c == '"' && (i == 0 || json.data[i-1] != '\\')) {
                in_string = 0;
                token_count++;  /* End of string token */
            }
        } else {
            switch (c) {
                case '"': in_string = 1; break;
                case '{': case '}': case '[': case ']': case ':': case ',':
                    token_count++;
                    break;
                default:
                    if (c != ' ' && c != '\t' && c != '\n' && c != '\r') {
                        /* Start of number, true, false, or null */
                        /* Skip until delimiter */
                        while (i + 1 < json.size) {
                            unsigned char next = json.data[i + 1];
                            if (next == ',' || next == '}' || next == ']' ||
                                next == ' ' || next == '\n') {
                                break;
                            }
                            i++;
                        }
                        token_count++;
                    }
            }
        }
    }
    
    return enif_make_long(env, token_count);
}

static ErlNifFunc nif_funcs[] = {
    {"count_tokens", 1, count_tokens_nif, ERL_NIF_DIRTY_JOB_CPU_BOUND}
};

ERL_NIF_INIT(fast_json_scanner, nif_funcs, NULL, NULL, NULL, NULL)
```

```erlang
%% fast_json_scanner.erl
-module(fast_json_scanner).
-export([count_tokens/1]).
-on_load(init/0).

init() ->
    SoPath = filename:join(code:priv_dir(my_app), "fast_json_scanner"),
    erlang:load_nif(SoPath, 0).

count_tokens(_Json) ->
    erlang:nif_error(nif_not_loaded).

%% Benchmark comparison:
%% Pure Erlang token counting: ~1M tokens/sec
%% NIF token counting: ~50M tokens/sec (50x faster)
```

---

## สรุป Part 90

| Feature | Description |
|---------|-------------|
| Basic NIF | C function called from Erlang |
| Dirty CPU | Long computation without blocking schedulers |
| Dirty I/O | Blocking I/O without blocking schedulers |
| Resources | C objects managed by BEAM GC |
| Safety | Always return quickly, no blocking syscalls |

**When to use NIFs**:
- CPU-intensive algorithms (crypto, compression, image processing)
- Interfacing with existing C libraries
- Performance-critical hot paths (>50x speedup possible)

**When NOT to use NIFs**:
- Simple logic (Erlang is fast enough)
- When correctness matters more than speed
- Without crash isolation (NIF crash = VM crash)

---

*[← Part 89: Production Recipes](part_89_production_recipes.md) | [Part 91: Parse Transforms →](part_91_parse_transforms.md)*
