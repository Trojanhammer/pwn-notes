# Stack, Heap & CPU Registers — Notes

## 1. Registers vs RAM — Mana Diorang Duduk?

```text
┌─────────────────────────────────┐         ┌─────────────────────────────────┐
│         CPU Chip (super fast)   │         │         RAM (big capacity)      │
│                                 │         │                                 │
│  ┌─────────────────────────┐    │         │  Address          Contents      │
│  │ REGISTERS               │    │         │  ┌────────────┬──────────────┐  │
│  │ • rax : 60              │    │         │  │ 1337000    │      3       │  │
│  │ • rdi : 0               │    │ ◄─────► │  ├────────────┼──────────────┤  │
│  │ • rsp : 1337000         │    │  data   │  │ 1337008    │  0x7ffff..   │  │
│  │ • rip : 0x401000        │    │   bus   │  ├────────────┼──────────────┤  │
│  └─────────────────────────┘    │         │  │ 1337016    │  0x7ffff..   │  │
└─────────────────────────────────┘         └─────────────────────────────────┘
```

* **Registers (`rax`, `rdi`, `rsp`, `rip`)** — duduk terus dalam CPU chip. Super fast, jumlah sikit, access guna nama je (`mov rdi, rsp`) **tanpa `[ ]`**.
* **RAM (Stack & Heap)** — duduk luar CPU. Capacity besar, guna nombor address, **wajib `[ ]`** untuk baca/tulis (`mov rdi, [rsp]`).

---

## 2. Stack vs Heap dalam Memory Map

```text
High Address (0x7fffffffffff)
┌──────────────────────────────────────────────────────────┐
│  STACK  (grows DOWN ▼ — address makin kecil)             │
│    │    • local variables, arguments, return addresses   │
│    ▼    • auto-managed by CPU (push / pop / call / ret)  │
├──────────────────────────────────────────────────────────┤
│                   ... empty space ...                    │
├──────────────────────────────────────────────────────────┤
│    ▲    • dynamic data (malloc / new)                    │
│    │    • kena urus manual (free / delete)               │
│  HEAP   (grows UP ▲ — address makin besar)               │
├──────────────────────────────────────────────────────────┤
│  DATA / BSS (global variables)                           │
├──────────────────────────────────────────────────────────┤
│  TEXT / CODE (program code — cth: 0x401000)              │
└──────────────────────────────────────────────────────────┘
Low Address (0x10000)
```

| | **Stack** | **Heap** |
| :--- | :--- | :--- |
| **Grows** | Down (high → low address) | Up (low → high address) |
| **Managed by** | CPU & compiler auto (`rsp`) | Programmer manual (`malloc`/`free`) |
| **Speed** | Super fast (just shift `rsp`) | Slower (kena cari ruang kosong) |
| **Size** | Limited (~8MB Linux — overflow = crash) | Besar (ikut free RAM) |

---

## 3. Stack, `rsp` & Program Arguments

Stack works like a **plate stack** — **LIFO** (Last In, First Out). `rsp` (Stack Pointer) always holds the **address of the top of the stack** at that moment.

Baca items without popping guna offset:
* `[rsp]` = top item
* `[rsp + 8]` = item kedua
* `[rsp + 16]` = item ketiga

### Arguments atas Stack bila program start
Bila run `/tmp/program hello world`, Linux Kernel auto susun arguments atas Stack before our code runs:

```text
Address     Stack contents    Meaning
┌─────────┬────────────────┐
│ 1337000 │       3        │ ◄── rsp  (argc = number of args)
├─────────┼────────────────┤
│ 1337008 │ ptr → program  │ ◄── [rsp + 8]  → "/tmp/program"
├─────────┼────────────────┤
│ 1337016 │ ptr → "hello"  │ ◄── [rsp + 16] → "hello"
├─────────┼────────────────┤
│ 1337024 │ ptr → "world"  │ ◄── [rsp + 24] → "world"
└─────────┴────────────────┘
```

---

## 4. `pop` (and `push`)

Instead of manually writing `mov rdi, [rsp]` then `add rsp, 8`, CPU ada built-in instruction **`pop`** yang buat both steps at once.

### Apa jadi masa `pop rdi`
```text
BEFORE pop rdi:                       AFTER pop rdi:
  rsp = 1337000                         rsp = 1337008  (auto +8)
  rdi = 0                               rdi = 3        (grabbed from [1337000])

Address    Contents                   Address    Contents
┌────────┬──────┐                     ┌────────┬──────┐
│1337000 │  3   │ ◄── rsp             │1337000 │  3   │ (already popped)
├────────┼──────┤                     ├────────┼──────┤
│1337008 │  99  │                     │1337008 │  99  │ ◄── rsp now here
└────────┴──────┘                     └────────┴──────┘
```

### `push` vs `pop`
* **`push rax`** — tambah item ke stack: `rsp - 8` dulu (grows down), then write `rax` into `[rsp]`
* **`pop rdi`** — ambil & buang item dari stack: read `[rsp]` into `rdi`, then `rsp + 8` (shrink balik)

Sebab stack grows down, **buang item = `rsp` naik (+8)** ke address lebih tinggi. Push = turun, pop = naik.

```asm
.intel_syntax noprefix
pop rdi           # grab top of stack into rdi, auto rsp + 8
mov rax, 60       # sys_exit
syscall
```
