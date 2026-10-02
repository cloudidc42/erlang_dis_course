# Part 1: บทนำ Erlang — ภาษาแห่งความเสถียรและ Distributed Computing

## สารบัญ
1. [Erlang คืออะไร?](#erlang-คืออะไร)
2. [ประวัติศาสตร์ของ Erlang](#ประวัติศาสตร์ของ-erlang)
3. [ทำไมต้องใช้ Erlang?](#ทำไมต้องใช้-erlang)
4. [จุดแข็งของ Erlang](#จุดแข็งของ-erlang)
5. [ระบบที่ใช้ Erlang จริงๆ](#ระบบที่ใช้-erlang-จริงๆ)
6. [BEAM Virtual Machine](#beam-virtual-machine)
7. [Erlang vs ภาษาอื่น](#erlang-vs-ภาษาอื่น)
8. [OTP คืออะไร?](#otp-คืออะไร)
9. [Elixir vs Erlang](#elixir-vs-erlang)
10. [โปรแกรมแรกของคุณ](#โปรแกรมแรกของคุณ)
11. [แนวคิดหลักของ Erlang](#แนวคิดหลักของ-erlang)
12. [สรุป](#สรุป)

---

## Erlang คืออะไร?

**Erlang** (อ่านว่า "เออร์แลง") เป็นภาษาโปรแกรมมิ่งที่ถูกสร้างขึ้นเพื่อ:
- ระบบ **Distributed** (กระจาย)
- ระบบ **Fault-tolerant** (ทนต่อความผิดพลาด)
- ระบบ **Real-time** (ประมวลผลแบบเรียลไทม์)
- ระบบที่ต้องมี **High Availability** (ใช้งานได้ตลอดเวลา)

Erlang เป็นภาษา **Functional Programming** ที่มีความสามารถในการจัดการ Concurrency ได้ดีมาก โดยใช้ **Actor Model** เป็นพื้นฐาน

### ลักษณะเด่น

```
Erlang = Functional + Concurrent + Distributed + Fault-tolerant
```

- **Immutable Data**: ข้อมูลไม่สามารถเปลี่ยนแปลงได้หลังจากสร้าง
- **Pattern Matching**: จับคู่รูปแบบข้อมูลได้อย่างทรงพลัง
- **Lightweight Processes**: สร้าง process ได้นับล้านตัวด้วยต้นทุนต่ำมาก
- **Message Passing**: processes สื่อสารกันผ่านการส่ง message เท่านั้น
- **"Let it Crash"**: ปล่อยให้ระบบล้มเหลวและฟื้นตัวเองได้

---

## ประวัติศาสตร์ของ Erlang

### Timeline

```
1986  → Joe Armstrong เริ่มพัฒนา Erlang ที่ Ericsson
         เป้าหมาย: สร้างภาษาสำหรับระบบโทรคมนาคม
         
1987  → Erlang เวอร์ชันแรกพร้อมใช้งาน
         ใช้ใน AXE telephone switch

1990s → ใช้งานในระบบ AXD301 ATM switch
         Uptime 99.9999999% (เก้าตัวเก้า!)
         
1998  → Ericsson ปล่อย Erlang เป็น Open Source
         (หลังจากที่ห้ามใช้ภายในและย้อนกลับมาเปิด)

2007  → Robert Virding, Claes Wikström, Mike Williams
         ได้รับ ACM Software System Award

2012  → Erlang/OTP 15B - การปรับปรุงครั้งใหญ่

2014  → Erlang/OTP 17 - เพิ่ม Maps

2016  → Erlang/OTP 19 - แก้ไข TLS, Logger

2019  → Erlang/OTP 22 - เพิ่ม logger module

2021  → Erlang/OTP 24 - JIT Compiler (ประสิทธิภาพเพิ่มขึ้น 30-40%)

2023  → Erlang/OTP 26 - ปรับปรุง performance และ type system

2024  → Erlang/OTP 27 - เพิ่ม type checker ในตัว
```

### ผู้สร้าง Erlang

**Joe Armstrong** (1950-2019) — "Father of Erlang"
- เขียน PhD thesis: "Making reliable distributed systems in the presence of software errors"
- คำพูดโด่งดัง: *"The problem with object-oriented languages is they've got all this implicit environment that they carry around with them. You wanted a banana but what you got was a gorilla holding the banana and the entire jungle."*

**Mike Williams** — พัฒนา BEAM VM ตัวแรก

**Robert Virding** — นักพัฒนาหลัก ออกแบบ Standard Library

---

## ทำไมต้องใช้ Erlang?

### ปัญหาที่ Erlang แก้ได้

```
ปัญหาระบบ Traditional:
━━━━━━━━━━━━━━━━━━━━━
Thread 1 ──→ Shared Memory ←── Thread 2
                   ↓
              Race Condition
              Deadlock
              Memory Corruption
              
ทางแก้ของ Erlang:
━━━━━━━━━━━━━━━━
Process 1 ──→ Mailbox    Process 2 ──→ Mailbox
                ↕                          ↕
         ส่ง Message               ส่ง Message
         
ไม่มี Shared Memory = ไม่มี Race Condition!
```

### เปรียบเทียบความยาก

| ภาษา | Concurrent Programming |
|------|------------------------|
| C/C++ | ยากมาก (mutex, semaphore, deadlock) |
| Java | ยาก (synchronized, volatile) |
| Python | ยาก (GIL, asyncio, threading) |
| Go | ง่ายกว่า (goroutines, channels) |
| **Erlang** | **ง่ายที่สุด** (process, message passing) |

---

## จุดแข็งของ Erlang

### 1. Fault Tolerance - "Let It Crash"

แนวคิด "Let It Crash" เป็นหัวใจสำคัญของ Erlang:

```erlang
% แทนที่จะป้องกันทุกข้อผิดพลาด...
try_something_risky() ->
    try
        risky_operation()
    catch
        _:_ -> handle_error()  % ซับซ้อนและอาจมีข้อผิดพลาด
    end.

% Erlang ทำแบบนี้:
try_something_risky() ->
    risky_operation().  % ปล่อยให้ crash!
    % Supervisor จะ restart process ให้อัตโนมัติ
```

**ผลลัพธ์**: ระบบฟื้นตัวเองได้โดยอัตโนมัติ

### 2. Concurrency แบบ Lightweight

```
Java Thread:       ~1 MB stack, สร้างได้ไม่กี่พัน
OS Thread:         ~8 MB stack, แพง
Erlang Process:    ~300 bytes เริ่มต้น, สร้างได้นับล้าน!
```

### 3. Soft Real-time

Erlang มี Garbage Collector แบบ per-process:
- GC ไม่หยุดทั้งระบบ
- แต่ละ Process มี GC ของตัวเอง
- Latency ต่ำและสม่ำเสมอ

### 4. Hot Code Reloading

```erlang
% อัปเดตโค้ดโดยไม่ต้องหยุดระบบ!
% ระบบที่มี uptime 99.9% สามารถ deploy ใหม่ได้
% โดยผู้ใช้ไม่รู้สึกอะไรเลย
```

### 5. Distributed by Default

```erlang
% เชื่อมต่อ Node อื่นได้ง่ายมาก
net_kernel:connect_node('node2@server2.example.com').

% ส่ง Message ข้าม Node เหมือนกับส่งภายใน Node เดียว
{remote_process, 'node2@server2.example.com'} ! {hello, world}.
```

---

## ระบบที่ใช้ Erlang จริงๆ

### WhatsApp

```
ข้อมูล:
- 2 พันล้าน users
- วิศวกร: ~50 คน (ก่อน Facebook ซื้อ)
- ทำไมต้องใช้คนน้อย? เพราะ Erlang ดูแลตัวเองได้
- Memory: แต่ละ connection ใช้แค่ ~300 bytes

Quote จาก Rick Reed (WhatsApp CTO):
"We had 2 million connections on a single server"
```

### Ericsson

```
ระบบ AXD301 ATM Switch:
- Uptime: 99.9999999% (ไม่หยุดทำงาน 31 ปี!)
- 9 เก้า = ไม่เกิน 31.5 milliseconds downtime ต่อปี
- ใช้งานจริงในเครือข่ายโทรคมนาคมทั่วโลก
```

### Discord

```
Discord ใช้ Elixir (ทำงานบน BEAM เหมือน Erlang):
- รองรับ 5 ล้าน concurrent users
- ต้นทุนเซิร์ฟเวอร์ลดลง 25% หลังเปลี่ยนจาก Python
```

### RabbitMQ

```
Message Broker ที่ดังที่สุด:
- เขียนด้วย Erlang
- ใช้ในบริษัทหลายล้านแห่งทั่วโลก
- Netflix, Instagram, Reddit ใช้ RabbitMQ
```

### CouchDB / Riak

```
Distributed Databases เขียนด้วย Erlang:
- CouchDB: ใช้ HTTP REST API
- Riak: Distributed Key-Value Store
  รองรับ 10,000+ requests/second ต่อ Node
```

### Bet365

```
Betting Platform ที่ใหญ่ที่สุดในโลก:
- Erlang รองรับ peak load ในช่วงกีฬาสำคัญ
- ล้าน concurrent users พร้อมกัน
```

---

## BEAM Virtual Machine

BEAM (Bogdan/Björn's Erlang Abstract Machine) คือ VM ที่รัน Erlang

```
โครงสร้าง BEAM:

┌─────────────────────────────────────────────┐
│                  BEAM VM                     │
├─────────────────────────────────────────────┤
│  Scheduler 1  │  Scheduler 2  │  Scheduler N│
│  (CPU Core 1) │  (CPU Core 2) │  (CPU Core N)│
├─────────────────────────────────────────────┤
│  Process Pool (นับล้าน lightweight processes)│
├─────────────────────────────────────────────┤
│  GC per Process │ Atom Table │ Code Server   │
└─────────────────────────────────────────────┘
```

### Scheduler

```
BEAM ใช้ Preemptive Scheduling:
- แต่ละ process ได้รับ 2000 "reduction" ก่อนที่จะถูกสลับ
- 1 reduction ≈ 1 function call
- ทำให้ทุก process ได้รับ CPU time อย่างยุติธรรม

ผลลัพธ์:
- ไม่มี process ไหน "ครอง" CPU นานเกินไป
- Latency ต่ำและสม่ำเสมอ
- Real-time behavior ที่คาดเดาได้
```

### Memory Model

```
ต่าง Node: มี heap ของตัวเอง
Process A ─── copy ──→ Process B
              ↑
         ส่งแค่ data, ไม่ share memory
         
ข้อดี: ไม่มี race condition!
ข้อเสีย: การ copy ใหญ่ๆ อาจช้า (แต่มี optimization)
```

---

## Erlang vs ภาษาอื่น

### Erlang vs Python

| หัวข้อ | Python | Erlang |
|--------|--------|--------|
| Concurrency | GIL (Global Interpreter Lock) | Lightweight Processes |
| Fault Tolerance | Try/Except | Supervisor Trees |
| Distribution | ต้องใช้ library | Built-in |
| Hot Reload | ไม่รองรับ native | Built-in |
| Performance | ช้า (ถ้าไม่ใช้ C extension) | เร็วกว่า Python มาก |
| Learning Curve | ง่าย | ปานกลาง |

### Erlang vs Go

| หัวข้อ | Go | Erlang |
|--------|-----|--------|
| Concurrency | Goroutines (stack-based) | Processes (heap-based, per-process GC) |
| Distribution | ต้องเขียนเอง | Built-in |
| Fault Tolerance | ต้องเขียนเอง | Supervisor Trees |
| Pattern Matching | จำกัด | ทรงพลังมาก |
| Hot Reload | ไม่รองรับ | Built-in |

### Erlang vs Java

| หัวข้อ | Java | Erlang |
|--------|------|--------|
| Concurrency | Threads (heavyweight) | Processes (lightweight) |
| GC | Stop-the-world GC | Per-process GC (ไม่ stop ทั้งระบบ) |
| Memory | Shared Heap | Private Heap per Process |
| Distribution | RMI, gRPC | Built-in |
| Learning Curve | ปานกลาง | ปานกลาง |

---

## OTP คืออะไร?

**OTP** (Open Telecom Platform) เป็น Framework ที่มาพร้อมกับ Erlang ประกอบด้วย:

### Behaviours

```erlang
% GenServer - Generic Server
% ใช้สร้าง stateful process ทั่วไป
-behaviour(gen_server).

% Supervisor - Supervisor
% ใช้ดูแลและ restart processes
-behaviour(supervisor).

% gen_statem - Finite State Machine
% ใช้สร้าง state machine
-behaviour(gen_statem).

% gen_event - Event Manager
% ใช้จัดการ events
-behaviour(gen_event).

% application - Application
% ใช้สร้าง OTP Application
-behaviour(application).
```

### Libraries

```
stdlib    - Standard Library (lists, dict, etc.)
kernel    - OS interface, networking
sasl      - System Application Support Library (logging)
ssl       - TLS/SSL support
crypto    - Cryptography
mnesia    - Distributed Database
inets     - HTTP client/server
```

---

## Elixir vs Erlang

Elixir เป็นภาษาที่ทำงานบน BEAM เช่นเดียวกับ Erlang

```
Erlang → เขียน BEAM bytecode โดยตรง
Elixir → compile เป็น BEAM bytecode ผ่าน Erlang
```

### ความแตกต่าง

```erlang
% Erlang - ดั้งเดิม
-module(hello).
-export([greet/1]).

greet(Name) ->
    io:format("Hello, ~s!~n", [Name]).
```

```elixir
# Elixir - syntax สวยกว่า
defmodule Hello do
  def greet(name) do
    IO.puts("Hello, #{name}!")
  end
end
```

### เมื่อไรใช้ Erlang vs Elixir?

```
ใช้ Erlang เมื่อ:
✓ ต้องการ performance สูงสุด
✓ ทำงานกับ Telecom/embedded systems
✓ ต้องการ interop กับ Erlang libraries โดยตรง
✓ ทีมมีความรู้ Erlang อยู่แล้ว

ใช้ Elixir เมื่อ:
✓ ต้องการ syntax ที่อ่านง่ายกว่า
✓ พัฒนา Web Application (Phoenix Framework)
✓ ทีมมาจาก Ruby/Python background
✓ ต้องการ Macro/Metaprogramming ที่ทรงพลัง
```

---

## โปรแกรมแรกของคุณ

### Hello World ใน Erlang Shell

เปิด Erlang shell โดยพิมพ์ `erl` ใน terminal:

```
$ erl
Erlang/OTP 26 [erts-14.0] [64-bit] [smp:8:8] [ds:8:8:10] [async-threads:1]

Eshell V14.0 (press Ctrl+G to abort, type help(). for help)
1>
```

พิมพ์คำสั่งแรก:

```erlang
1> io:format("Hello, Erlang!~n").
Hello, Erlang!
ok
2>
```

### การออกจาก Shell

```erlang
% วิธีที่ 1: กด Ctrl+C แล้วกด a
% วิธีที่ 2:
q().
% วิธีที่ 3:
halt().
```

### โปรแกรมแรกใน File

สร้างไฟล์ `hello.erl`:

```erlang
%% ไฟล์: hello.erl
-module(hello).          %% ชื่อ module ต้องตรงกับชื่อไฟล์
-export([world/0]).      %% export function world/0 ให้เรียกจากภายนอกได้

%% Function hello:world/0
%% ไม่มี argument (arity = 0)
world() ->
    io:format("Hello, World!~n").
```

### Compile และ Run

```bash
# วิธีที่ 1: Compile ใน Shell
$ erl
1> c(hello).
{ok,hello}
2> hello:world().
Hello, World!
ok

# วิธีที่ 2: Compile จาก Command Line
$ erlc hello.erl
$ erl -noshell -eval "hello:world()" -s init stop
Hello, World!
```

### โปรแกรม "Hello, Name!"

```erlang
%% ไฟล์: greeting.erl
-module(greeting).
-export([hello/1, hello/0]).

%% Function ที่รับ Name เป็น argument
hello(Name) ->
    io:format("Hello, ~s!~n", [Name]).

%% Function ที่ไม่รับ argument - ใช้ค่า default
hello() ->
    hello("World").
```

```erlang
% ทดสอบใน Shell
1> c(greeting).
{ok,greeting}
2> greeting:hello("Thailand").
Hello, Thailand!
ok
3> greeting:hello().
Hello, World!
ok
```

---

## แนวคิดหลักของ Erlang

### 1. Everything is a Process

```
ใน Erlang ทุกอย่างเป็น Process:
- Web Request Handler → Process
- Database Connection → Process
- Timer → Process
- Background Job → Process

กว่า 1 ล้าน Process สามารถทำงานพร้อมกันได้!
```

### 2. Processes are Isolated

```
Process A    Process B    Process C
   │            │            │
   ▼            ▼            ▼
[Heap A]    [Heap B]    [Heap C]
   ↑            ↑            ↑
ต่างคนต่างอยู่ ไม่ share memory
ถ้า Process A crash → ไม่กระทบ B และ C
```

### 3. Message Passing

```erlang
% ส่ง Message
RecipientPid ! {hello, "World"}.

% รับ Message
receive
    {hello, Message} ->
        io:format("Got: ~s~n", [Message])
end.
```

### 4. Pattern Matching

```erlang
% Pattern Matching ที่ทรงพลัง
process_message({ok, Result}) ->
    io:format("Success: ~p~n", [Result]);
process_message({error, Reason}) ->
    io:format("Error: ~p~n", [Reason]);
process_message(unknown) ->
    io:format("Unknown message~n").
```

### 5. Immutability

```erlang
% ใน Erlang ตัวแปรเปลี่ยนค่าไม่ได้!
X = 5.       % ผูก X กับ 5
X = 5.       % OK - match กับค่าเดิม
% X = 6.    % ERROR! X already bound to 5

% แต่เราสามารถสร้างตัวแปรใหม่:
Y = X + 1.   % Y = 6, X ยังคงเป็น 5
```

---

## ความเข้าใจพื้นฐาน: The Erlang Way

### Traditional OOP (Object-Oriented)

```
Object-Oriented Thinking:
"ฉันมี Object ที่มี State และ Method"

โปรแกรม = Objects ที่ interact กัน
ปัญหา: Shared mutable state = Race Conditions
```

### Erlang Way

```
Erlang Thinking:
"ฉันมี Process ที่ communicate ผ่าน Messages"

โปรแกรม = Processes ที่ส่ง Messages หากัน
ข้อดี: ไม่มี Shared State = ไม่มี Race Conditions
```

### การคิดแบบ Erlang

```erlang
%% แทนที่จะคิดว่า:
%% "สร้าง object แล้วเรียก method"

%% คิดว่า:
%% "สร้าง process แล้วส่ง message"

% สร้าง Counter Process
CounterPid = spawn(counter, start, [0]).

% ส่ง message เพื่อเพิ่มค่า
CounterPid ! increment.

% ส่ง message เพื่อรับค่า
CounterPid ! {get, self()}.
receive
    {value, Count} ->
        io:format("Count: ~p~n", [Count])
end.
```

---

## สรุป Part 1

เราได้เรียนรู้:

1. **Erlang คืออะไร**: ภาษา Functional, Concurrent, Distributed ที่สร้างโดย Ericsson
2. **ประวัติศาสตร์**: จาก Telecom Switch สู่ WhatsApp และ Discord
3. **จุดแข็ง**: Fault Tolerance, Lightweight Processes, Hot Code Reload, Distributed
4. **BEAM VM**: VM ที่ใช้ run Erlang code
5. **OTP**: Framework สำหรับสร้างระบบ Production-grade
6. **แนวคิดหลัก**: Everything is a Process, Isolation, Message Passing, Immutability
7. **โปรแกรมแรก**: Hello World ใน Erlang

### ก้าวต่อไป

ใน **Part 2** เราจะเรียน:
- การติดตั้ง Erlang/OTP
- การตั้งค่า Development Environment
- การใช้ Erlang Shell (eshell)
- การใช้ Rebar3 Build Tool

---

## แบบฝึกหัด Part 1

**แบบฝึกหัดที่ 1**: เปิด Erlang shell และลองพิมพ์คำสั่งต่อไปนี้:
```erlang
1> 1 + 1.
2> "Hello" ++ " " ++ "World".
3> lists:sum([1, 2, 3, 4, 5]).
4> erlang:system_info(schedulers).
```

**แบบฝึกหัดที่ 2**: สร้างไฟล์ `my_first.erl` ที่:
- มี module ชื่อ `my_first`
- มี function `sum/2` ที่รับตัวเลข 2 ตัวแล้วบวกกัน
- มี function `factorial/1` ที่คำนวณ factorial

**แบบฝึกหัดที่ 3**: ค้นหาว่า Erlang ถูกนำไปใช้ในระบบอะไรอีกบ้าง และเขียนสรุปสั้นๆ

---

*[← กลับ README](../README.md) | [Part 2: การติดตั้ง →](part_02_installation.md)*
