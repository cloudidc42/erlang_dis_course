# Part 2: การติดตั้งและตั้งค่าสภาพแวดล้อม Erlang

## สารบัญ
1. [ข้อกำหนดระบบ](#ข้อกำหนดระบบ)
2. [ติดตั้งบน Linux (Ubuntu/Debian)](#ติดตั้งบน-linux-ubuntudebian)
3. [ติดตั้งบน macOS](#ติดตั้งบน-macos)
4. [ติดตั้งบน Windows](#ติดตั้งบน-windows)
5. [ติดตั้งด้วย ASDF (แนะนำ)](#ติดตั้งด้วย-asdf-แนะนำ)
6. [ติดตั้งด้วย Docker](#ติดตั้งด้วย-docker)
7. [ตรวจสอบการติดตั้ง](#ตรวจสอบการติดตั้ง)
8. [Erlang Shell (eshell)](#erlang-shell-eshell)
9. [ติดตั้ง Rebar3](#ติดตั้ง-rebar3)
10. [ตั้งค่า Editor](#ตั้งค่า-editor)
11. [โปรเจค Erlang แรก](#โปรเจค-erlang-แรก)
12. [สรุป](#สรุป)

---

## ข้อกำหนดระบบ

### Minimum Requirements
```
OS: Linux, macOS, Windows 10+
RAM: 512 MB (แนะนำ 2+ GB สำหรับ development)
Disk: 500 MB สำหรับ Erlang/OTP
CPU: x86_64 หรือ ARM64
```

### Recommended Development Setup
```
OS: Ubuntu 22.04 LTS, macOS 13+, หรือ WSL2 บน Windows
RAM: 8+ GB
Disk: 20+ GB
CPU: Multi-core (Erlang ใช้ทุก core ได้เต็มที่)
Editor: VS Code, IntelliJ IDEA, Emacs, หรือ Vim
```

---

## ติดตั้งบน Linux (Ubuntu/Debian)

### วิธีที่ 1: ผ่าน Package Manager (ง่ายแต่อาจไม่ใช่เวอร์ชันล่าสุด)

```bash
# อัพเดท package list
sudo apt update

# ติดตั้ง Erlang
sudo apt install erlang erlang-dev erlang-doc

# ตรวจสอบเวอร์ชัน
erl -version
```

### วิธีที่ 2: ผ่าน Erlang Solutions Repository (แนะนำ - ได้เวอร์ชันล่าสุด)

```bash
# ดาวน์โหลด repository key
wget -O- https://packages.erlang-solutions.com/ubuntu/erlang_solutions.asc \
    | sudo apt-key add -

# เพิ่ม repository
echo "deb https://packages.erlang-solutions.com/ubuntu $(lsb_release -cs) contrib" \
    | sudo tee /etc/apt/sources.list.d/erlang-solutions.list

# อัพเดทและติดตั้ง
sudo apt update
sudo apt install esl-erlang

# ตรวจสอบ
erl -version
```

### วิธีที่ 3: Compile จาก Source (เพื่อ customization สูงสุด)

```bash
# ติดตั้ง dependencies
sudo apt install build-essential autoconf libncurses5-dev \
    libssl-dev libwxwidgets-common-dev libgl1-mesa-dev \
    libglu1-mesa-dev libpng-dev m4 unixodbc-dev

# ดาวน์โหลด source code
wget https://github.com/erlang/otp/releases/download/OTP-26.2/otp_src_26.2.tar.gz
tar -xzf otp_src_26.2.tar.gz
cd otp_src_26.2

# Configure
./configure --with-ssl --enable-hipe

# Compile (ใช้เวลาสักครู่)
make -j$(nproc)

# ติดตั้ง
sudo make install

# ตรวจสอบ
erl -version
```

### ติดตั้นบน CentOS/RHEL/Fedora

```bash
# Fedora
sudo dnf install erlang

# CentOS/RHEL - ต้องเพิ่ม EPEL repository ก่อน
sudo dnf install epel-release
sudo dnf install erlang

# หรือจาก Erlang Solutions
curl -s https://packagecloud.io/install/repositories/rabbitmq/erlang/script.rpm.sh \
    | sudo bash
sudo yum install erlang
```

---

## ติดตั้งบน macOS

### วิธีที่ 1: Homebrew (แนะนำ)

```bash
# ติดตั้ง Homebrew ก่อน (ถ้ายังไม่มี)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง Erlang
brew install erlang

# ตรวจสอบ
erl -version
```

### วิธีที่ 2: MacPorts

```bash
sudo port install erlang +ssl
```

### วิธีที่ 3: Installer จาก Erlang Solutions

```bash
# ดาวน์โหลด .pkg installer จาก:
# https://www.erlang-solutions.com/downloads/

# เปิด .pkg file และทำตาม wizard
# ตรวจสอบ
erl -version
```

---

## ติดตั้งบน Windows

### วิธีที่ 1: Official Installer

```
1. ไปที่ https://www.erlang.org/downloads
2. ดาวน์โหลด Windows installer (.exe)
3. เรียกใช้ installer
4. ทำตาม wizard
5. เพิ่ม C:\Program Files\erl-26\bin ใน PATH
```

### วิธีที่ 2: Chocolatey

```powershell
# เปิด PowerShell เป็น Administrator
choco install erlang
```

### วิธีที่ 3: WSL2 (แนะนำสำหรับ Development)

```powershell
# เปิด PowerShell เป็น Administrator
wsl --install

# เลือก Ubuntu
# รีสตาร์ทเครื่อง
# เปิด Ubuntu
# ทำตามขั้นตอน Linux ด้านบน
```

---

## ติดตั้งด้วย ASDF (แนะนำ)

ASDF เป็น version manager ที่ช่วยจัดการ Erlang หลายเวอร์ชัน

### ติดตั้ง ASDF

```bash
# Clone ASDF
git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.14.0

# เพิ่มใน shell config
echo '. "$HOME/.asdf/asdf.sh"' >> ~/.bashrc
echo '. "$HOME/.asdf/completions/asdf.bash"' >> ~/.bashrc
source ~/.bashrc

# ตรวจสอบ
asdf --version
```

### ติดตั้ง Erlang Plugin

```bash
# ติดตั้น plugin
asdf plugin-add erlang https://github.com/asdf-vm/asdf-erlang.git

# ติดตั้ง dependencies (Ubuntu)
sudo apt install autoconf libssl-dev libwxwidgets-common-dev \
    libwxgtk3.0-gtk3-dev libgl1-mesa-dev libglu1-mesa-dev libpng-dev

# แสดงเวอร์ชันที่มี
asdf list-all erlang

# ติดตั้น Erlang 26.2
asdf install erlang 26.2

# ตั้งค่าเป็น version เริ่มต้น
asdf global erlang 26.2

# ตรวจสอบ
erl -version
```

### ใช้ Version ต่างๆ ต่อ Project

```bash
# ใน project folder
cd my_erlang_project

# ใช้ Erlang 26.2 สำหรับ project นี้
asdf local erlang 26.2

# หรือใช้ .tool-versions file
echo "erlang 26.2" > .tool-versions

# ตรวจสอบ
asdf current erlang
```

---

## ติดตั้งด้วย Docker

### Erlang Official Docker Image

```bash
# Pull image
docker pull erlang:26

# รัน Erlang shell
docker run -it erlang:26 erl

# รัน Erlang script
docker run -it --rm -v $(pwd):/app erlang:26 erl -noshell -eval \
    "c('/app/hello'), hello:world(), halt()."
```

### Dockerfile สำหรับ Development

```dockerfile
# Dockerfile.dev
FROM erlang:26

# ติดตั้น development tools
RUN apt-get update && apt-get install -y \
    git \
    curl \
    vim \
    && rm -rf /var/lib/apt/lists/*

# ติดตั้น Rebar3
RUN wget -q https://s3.amazonaws.com/rebar3/rebar3 -O /usr/local/bin/rebar3 \
    && chmod +x /usr/local/bin/rebar3

WORKDIR /app

CMD ["/bin/bash"]
```

```bash
# Build image
docker build -f Dockerfile.dev -t erlang-dev .

# Run container
docker run -it --rm -v $(pwd):/app erlang-dev

# ใน container
rebar3 new app my_app
cd my_app
rebar3 shell
```

### Docker Compose สำหรับ Project

```yaml
# docker-compose.yml
version: '3.8'

services:
  erlang:
    image: erlang:26
    volumes:
      - .:/app
    working_dir: /app
    command: bash -c "rebar3 shell"
    ports:
      - "8080:8080"
    
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp_dev
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
```

---

## ตรวจสอบการติดตั้ง

### คำสั่งตรวจสอบ

```bash
# ตรวจสอบ Erlang version
erl -version
# Output: Erlang (SMP,ASYNC_THREADS,HIPE) (BEAM) emulator version 14.0

# ตรวจสอบ Erlang/OTP version ละเอียด
erl -eval 'erlang:display(erlang:system_info(otp_release)), halt().' -noshell
# Output: "26"

# ตรวจสอบ compiler
erlc --version
# Output: Erlang/OTP 26 [erts-14.0] ...

# ตรวจสอบ tools
which erl erlc escript
```

### ทดสอบใน Shell

```bash
$ erl
Erlang/OTP 26 [erts-14.0] [source] [64-bit] [smp:8:8] [ds:8:8:10] [async-threads:1] [jit:ns]

Eshell V14.0 (press Ctrl+G to abort, type help(). for help)
1> erlang:system_info(otp_release).
"26"
2> erlang:system_info(schedulers).
8
3> erlang:system_info(wordsize).
8
4> halt().
```

---

## Erlang Shell (eshell)

Erlang Shell หรือ eshell เป็นเครื่องมือสำคัญสำหรับ development และ debugging

### เปิด Shell

```bash
$ erl
```

### คำสั่งพื้นฐาน

```erlang
% แสดงความช่วยเหลือ
1> help().

% ดู history
2> h().  % หรือกด Ctrl+P (previous) / Ctrl+N (next)

% ล้างหน้าจอ
3> c:cls().

% Compile module
4> c(module_name).

% Load module
5> l(module_name).

% แสดง module info
6> module_name:module_info().

% ออกจาก shell
7> q().
% หรือ halt().
```

### Shell Variables

```erlang
% ตัวแปรใน Shell ต้องเริ่มด้วยตัวพิมพ์ใหญ่
1> X = 10.
10
2> Y = 20.
20
3> X + Y.
30

% ตัวแปรถูก bind เพียงครั้งเดียว
4> X = 10.  % OK - match กับค่าเดิม
10
5> X = 15.  % ERROR!
** exception error: no match of right hand side value 15

% ใช้ f() เพื่อ forget variable
6> f(X).
ok
7> X = 15.  % OK แล้ว
15

% Forget ทั้งหมด
8> f().
ok
```

### Shell History

```erlang
% ดู expression ก่อนหน้า
v(1)   % ผลลัพธ์ของ expression ที่ 1
v(-1)  % ผลลัพธ์ของ expression ก่อนหน้า
```

### Keyboard Shortcuts

```
Ctrl+A  - ไปต้นบรรทัด
Ctrl+E  - ไปท้ายบรรทัด
Ctrl+P  - expression ก่อนหน้า
Ctrl+N  - expression ถัดไป
Ctrl+K  - ลบตั้งแต่ cursor ถึงท้ายบรรทัด
Ctrl+U  - ลบทั้งบรรทัด
Ctrl+C  - เมนู (จากนั้น a เพื่อออก)
Ctrl+G  - เมนู Job Control
Tab     - autocomplete
```

### Multi-line Expressions

```erlang
% พิมพ์ expression หลายบรรทัดได้โดยไม่ต้องกด Enter จนกว่าจะพิมพ์ . (dot)
1> lists:foldl(
1>     fun(X, Acc) -> X + Acc end,
1>     0,
1>     [1, 2, 3, 4, 5]).
15
```

### Shell ที่มีประโยชน์

```erlang
% รัน script โดยไม่เปิด interactive shell
$ erl -noshell -eval "io:format(\"Hello~n\"), halt()."

% รัน escript file
$ cat hello.escript
#!/usr/bin/env escript

main(_Args) ->
    io:format("Hello from escript!~n").

$ escript hello.escript
Hello from escript!
```

---

## ติดตั้ง Rebar3

Rebar3 เป็น build tool มาตรฐานของ Erlang

### ติดตั้ง Rebar3

```bash
# วิธีที่ 1: ดาวน์โหลด binary โดยตรง
curl https://s3.amazonaws.com/rebar3/rebar3 -o rebar3
chmod +x rebar3
sudo mv rebar3 /usr/local/bin/

# วิธีที่ 2: Homebrew (macOS)
brew install rebar3

# วิธีที่ 3: Compile จาก source
git clone https://github.com/erlang/rebar3.git
cd rebar3
./bootstrap
sudo cp rebar3 /usr/local/bin/

# ตรวจสอบ
rebar3 --version
# rebar 3.22.0 on Erlang/OTP 26
```

### คำสั่ง Rebar3 พื้นฐาน

```bash
# สร้าง project ใหม่
rebar3 new app my_app        # OTP Application
rebar3 new lib my_lib        # Library
rebar3 new release my_release # Release
rebar3 new escript my_script  # Escript

# Build project
rebar3 compile

# รัน tests
rebar3 eunit
rebar3 ct

# รัน shell กับ project loaded
rebar3 shell

# สร้าง release
rebar3 release

# Clean build
rebar3 clean

# ดู dependencies
rebar3 tree

# อัพเดท dependencies
rebar3 upgrade
```

---

## ตั้งค่า Editor

### VS Code

```bash
# ติดตั้น VS Code
# https://code.visualstudio.com/

# ติดตั้น Erlang extension
# เปิด VS Code แล้วกด Ctrl+P
# พิมพ์: ext install erlang-ls.erlang-ls
# หรือ: ext install pgourlain.erlang

# หรือจาก command line
code --install-extension erlang-ls.erlang-ls
```

#### VS Code Settings สำหรับ Erlang

```json
// .vscode/settings.json
{
    "erlang.erlangPath": "/usr/local/lib/erlang/bin",
    "erlang.erlangLsPath": "/usr/local/bin/erlang_ls",
    "[erlang]": {
        "editor.tabSize": 4,
        "editor.insertSpaces": true,
        "editor.rulers": [80, 100]
    },
    "files.associations": {
        "*.erl": "erlang",
        "*.hrl": "erlang",
        "*.app.src": "erlang"
    }
}
```

#### ติดตั้ง Erlang Language Server

```bash
# Erlang Language Server ให้ features เช่น:
# - Code completion
# - Go to definition
# - Find references
# - Diagnostics

# ติดตั้นผ่าน rebar3
git clone https://github.com/erlang-ls/erlang_ls.git
cd erlang_ls
rebar3 escriptize
cp _build/default/bin/erlang_ls /usr/local/bin/

# สร้าง config file
cat > erlang_ls.config << 'EOF'
apps_dirs:
  - "apps/*"
  - "."
deps_dirs:
  - "_build/default/lib"
include_dirs:
  - "include"
  - "_build/default/lib/*/include"
EOF
```

### IntelliJ IDEA / Android Studio

```bash
# ติดตั้น Erlang plugin
# File → Settings → Plugins
# ค้นหา "Erlang"
# ติดตั้น "Erlang" plugin โดย Igor Fedorov
```

### Emacs

```lisp
;; ใน ~/.emacs หรือ ~/.emacs.d/init.el
;; ติดตั้น packages ที่จำเป็น

;; เพิ่ม MELPA repository
(require 'package)
(add-to-list 'package-archives
             '("melpa" . "https://melpa.org/packages/") t)
(package-initialize)

;; ติดตั้น erlang-mode
;; M-x package-install RET erlang RET

;; Configuration
(require 'erlang-start)
(setq erlang-root-dir "/usr/local/lib/erlang")
(add-to-list 'exec-path "/usr/local/lib/erlang/bin")

;; Auto-complete
(require 'company)
(add-hook 'erlang-mode-hook 'company-mode)
```

### Vim/Neovim

```vim
" ใน ~/.vimrc หรือ ~/.config/nvim/init.vim
" ติดตั้น vim-erlang plugins

" ใช้ vim-plug
Plug 'vim-erlang/vim-erlang-runtime'
Plug 'vim-erlang/vim-erlang-compiler'
Plug 'vim-erlang/vim-erlang-omnicomplete'
Plug 'vim-erlang/vim-erlang-tags'

" Configuration
autocmd FileType erlang setlocal tabstop=4 shiftwidth=4 expandtab
let g:erlang_tags_auto_update = 1
```

---

## โปรเจค Erlang แรก

### สร้าง OTP Application

```bash
# สร้าง project
rebar3 new app hello_world
cd hello_world

# โครงสร้าง directory
tree .
.
├── LICENSE
├── README.md
├── rebar.config
└── src
    ├── hello_world.app.src
    ├── hello_world_app.erl
    └── hello_world_sup.erl
```

### ดู rebar.config

```erlang
%% rebar.config
{erl_opts, [debug_info]}.
{deps, []}.

{shell, [
    % {config, "config/sys.config"},
    {apps, [hello_world]}
]}.
```

### ดู hello_world.app.src

```erlang
%% src/hello_world.app.src
{application, hello_world,
 [{description, "An OTP application"},
  {vsn, "0.1.0"},
  {registered, []},
  {mod, {hello_world_app, []}},
  {applications,
   [kernel,
    stdlib
   ]},
  {env,[]},
  {modules, []},

  {licenses, ["Apache-2.0"]},
  {links, []}
 ]}.
```

### เพิ่ม Module แรก

```bash
# สร้าง module ใหม่
cat > src/hello_world.erl << 'EOF'
-module(hello_world).
-export([greet/0, greet/1]).

greet() ->
    greet("World").

greet(Name) ->
    io:format("Hello, ~s!~n", [Name]).
EOF
```

### Compile และ Test

```bash
# Compile
rebar3 compile

# รัน shell
rebar3 shell

# ใน shell
1> hello_world:greet().
Hello, World!
ok
2> hello_world:greet("Thailand").
Hello, Thailand!
ok
3> q().
```

### เพิ่ม Dependencies

```erlang
%% rebar.config - เพิ่ม dependency
{erl_opts, [debug_info]}.
{deps, [
    {jsx, "3.1.0"}   %% JSON library
]}.
```

```bash
# ดาวน์โหลด dependencies
rebar3 deps

# หรือ compile (จะดาวน์โหลดอัตโนมัติ)
rebar3 compile
```

---

## สรุป Part 2

เราได้ติดตั้งและตั้งค่า:

1. **Erlang/OTP**: ติดตั้นได้หลายวิธี (package manager, ASDF, Docker)
2. **Erlang Shell**: เครื่องมือสำคัญสำหรับ exploration
3. **Rebar3**: Build tool มาตรฐาน
4. **Editor**: VS Code, IntelliJ, Emacs, หรือ Vim
5. **โปรเจคแรก**: สร้างด้วย `rebar3 new app`

### Quick Reference

```bash
# ติดตั้น (Ubuntu)
sudo apt install esl-erlang

# ตรวจสอบ
erl -version

# เปิด Shell
erl

# สร้าง Project
rebar3 new app my_app
cd my_app
rebar3 shell
```

### ก้าวต่อไป

ใน **Part 3** เราจะเรียน Syntax พื้นฐานของ Erlang:
- Atoms, Variables, Numbers
- Strings และ Binaries  
- Lists, Tuples, Maps
- การเขียน Comments
- Expressions และ Statements

---

## แบบฝึกหัด Part 2

**แบบฝึกหัดที่ 1**: ติดตั้ง Erlang และ Rebar3 บนเครื่องของคุณ

**แบบฝึกหัดที่ 2**: ลองคำสั่งต่อไปนี้ใน Erlang Shell:
```erlang
% ดู scheduler info
erlang:system_info(schedulers).
erlang:system_info(schedulers_online).

% ดู memory info
erlang:memory().

% สร้าง process ง่ายๆ
Pid = spawn(fun() -> io:format("Hello from process ~p~n", [self()]) end).
```

**แบบฝึกหัดที่ 3**: สร้าง Rebar3 project ชื่อ `calculator` และเพิ่ม functions:
- `add(A, B)` - บวก
- `subtract(A, B)` - ลบ
- `multiply(A, B)` - คูณ
- `divide(A, B)` - หาร (จัดการกรณี B = 0 ด้วย)

---

*[← Part 1: บทนำ](part_01_introduction.md) | [Part 3: Syntax พื้นฐาน →](part_03_basic_syntax.md)*
