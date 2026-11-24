# Software Introspection (`objdump`, `strace` & `gdb`) — Notes

## 1. Quick Tools: `objdump` & `strace`

Before buka debugger berat, ada 2 tools untuk intip binary:

* **`objdump -d -M intel ./program`** — convert binary (ELF) back to Assembly Intel syntax. Static analysis — tak perlu run program pun. Good for getting a quick overview.
* **`gdb ./program`** — runtime analysis. Boleh freeze CPU mid-execution and check real-time register values and RAM contents.

```bash
$ objdump -d -M intel ./program
0000000000401000 <_start>:
  401000: 48 c7 c7 2a 00 00 00    mov    rdi, 0x2a    # rdi = 42
  401007: 48 c7 c0 3c 00 00 00    mov    rax, 0x3c    # rax = 60 (sys_exit)
  40100e: 0f 05                   syscall
```

* **`strace ./program`** — spy on every syscall the program makes at runtime, with arguments and return values.

```bash
$ strace ./program
execve("./program", ["./program"], 0x... /* 20 vars */) = 0
# ^ OS loads the program

alarm(5) = 0
# ^ program set a 5-second timer (maybe to auto-fail if too slow)

write(1, "Enter the flag: ", 16) = 16
# ^ print "Enter the flag: " to stdout (fd=1), 16 bytes

read(0, "test\n", 128) = 5
# ^ read input from stdin (fd=0), got "test\n" (5 bytes)

exit(1) = ?
# ^ exit with code 1 (wrong answer)
```

Strace berguna untuk nampak exactly what the program expects — timing, input format, which fd it reads from — without needing to reverse engineer everything.

---

## 2. GDB Basics — Start & Control

**GDB** = real-time microscope. Boleh freeze CPU kat mana-mana instruction and inspect/modify registers atau RAM.

| Command | Short | What it does |
| :--- | :--- | :--- |
| `gdb ./program` | — | Load binary into GDB (belum run lagi) |
| `starti` | — | Run and **immediately pause at the very first instruction** (`_start`) |
| `run` | `r` | Run from start until end or breakpoint |
| `run arg1 arg2` | `r arg1 arg2` | Run with command-line arguments |
| `run < input.txt` | `r < input.txt` | Run with file fed as stdin |
| `continue` | `c` | Resume dari pause |
| `stepi` | `si` | Move forward **1 Assembly instruction** |
| `quit` | `q` | Keluar GDB |

---

## 3. Disassemble & Breakpoints

Daripada tekan `si` berpuluh kali dari awal, baca the code map dulu, set breakpoint kat target address, pastu `run` terus sampai sana.

```gdb
set disassembly-flavor intel    # switch to Intel syntax (mov rax, 60)
disassemble main                # show Assembly for main / _start
x/10i $rip                      # show next 10 instructions from current position
break *0x40100e                 # set breakpoint at address (wajib ada * before hex!)
b *0x40100e                     # same, short form
```

### Workflow: `disassemble` → `break` → `run`
```gdb
(gdb) disassemble main
   0x401000 <+0>:   mov rdi, 0x0
   0x401007 <+7>:   add rdi, CENSORED
=> 0x40100e <+14>:  mov rdi, 0x0
   0x401015 <+21>:  mov rax, 0x3c
   0x40101c <+28>:  syscall

(gdb) break *0x40100e    # pause right after secret value loads into $rdi
(gdb) run                # program runs fast, auto-pauses at 0x40100e
(gdb) print $rdi         # read the secret value without pressing si many times
```

### `int3` — Software Breakpoint
* `int3` (machine code `0xcc`) = built-in trap instruction dalam x86-64
* Bila CPU hit `int3`, dia trigger **`SIGTRAP`** → program auto-pauses in GDB
* Behind the scenes, `break *0x40100e` sebenarnya inject 1 byte `0xcc` kat address tu dalam RAM

---

## 4. Read & Modify Registers

Dalam GDB, every register name kena ada **`$`** prefix (`$rax`, `$rdi`, `$rsp`, `$rip`).

```gdb
info registers          # (i r) show all registers
print $rdi              # (p $rdi) show one register
set $rdi = 1337         # force-change rdi to 1337
set $rax = 0x1337       # same in hex
print $rdi              # confirm ($1 = 1337)
```

---

## 5. Examine Memory (`x/`)

* `print $rsp` — prints the value **inside the register** (cth: address number `0x7fffffffeb10`) — no dereference
* `x $rsp` — **dereferences** it: goes to that address in RAM and reads what's there

### Formula: `x/<count><format><size> <address>`

**Format:**
* `x` = hex, `d` = decimal, `s` = string, `a` = address, `i` = instruction

**Size:**
* `b` = 1 byte, `h` = 2 bytes, `w` = 4 bytes, `g` = 8 bytes (64-bit — most common)

```gdb
x/gx $rsp               # read 1 x 64-bit value at top of stack (argc)
x/10a $rsp              # show 10 stack slots (argc, argv[0] at +8, argv[1] at +16...)
x/s *(char**)($rsp+16)  # dereference argv[1] address to read the actual string
x/gx 0x404000           # read 64-bit value at fixed address
```

### Stack Layout Note
Setiap pointer 64-bit makan **8 bytes** (`+0x8`). Values stored in **little-endian** (smallest byte at lowest address). So `$rsp+0` = argc, `$rsp+8` = argv[0], `$rsp+16` = argv[1], and so on.

---

## 6. Workflow Summary

1. `starti` atau `run` → freeze at breakpoint / `int3` `SIGTRAP`
2. `x/10a $rsp` → inspect secret values on the stack
3. `set $rdi = <value>` atau `continue` → inject value into register or resume
