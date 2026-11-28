# Assembly Control Flow & Comparisons — Notes

## 1. Comparing Values (`cmp`)
CPU tak ada `if/else` terus. Instead, guna **`cmp`** untuk compare dua values.

* `cmp a, b` does `a - b` temporarily — tak ubah nilai `a` atau `b`
* Result updates **RFLAGS register** (cth: *Zero Flag (ZF)* set to 1 kalau `a == b`)

```asm
cmp eax, 10      # compare eax dengan 10
cmp rdi, rsi     # compare dua registers
cmp rax, 0       # check kalau rax == 0
```

---

## 2. Comparing Characters
At machine level, computer tak kenal huruf — semua nombor based on **ASCII**.

* Each character = **1 byte**, so guna `BYTE PTR` dan 8-bit registers (`al`, `bl`, `dil`)
* Contoh: `'A'` = `0x41` = 65, `'a'` = `0x61` = 97, `'0'` = `0x30` = 48

```asm
cmp BYTE PTR [rdi], 0x41   # check kalau huruf pertama ialah 'A'
cmp BYTE PTR [rdi], 0x0    # check kalau null terminator (end of string)
cmp BYTE PTR [rdi], 0x61   # check kalau huruf 'a'
```

---

## 3. Conditional Jumps
Lepas `cmp` update flags, CPU guna **conditional jump** untuk decide mana nak pergi — macam `if/else`.

| Instruction | Meaning | When it jumps |
| :--- | :--- | :--- |
| `je` / `jz` | Jump if Equal / Zero | Nilai **sama** (`ZF = 1`) |
| `jne` / `jnz` | Jump if Not Equal / Not Zero | Nilai **tak sama** (`ZF = 0`) |
| `jg` / `ja` | Jump if Greater / Above | Nilai pertama **lebih besar** |
| `jl` / `jb` | Jump if Less / Below | Nilai pertama **lebih kecil** |
| `jge` | Jump if Greater or Equal | Lebih besar atau sama |
| `jle` | Jump if Less or Equal | Lebih kecil atau sama |
| `jmp` | Unconditional Jump | Always jump, no condition |

```asm
# contoh 1: simple equal check
cmp eax, 5
je  equal       # eax == 5 → lompat ke equal
jne not_equal   # eax != 5 → lompat ke not_equal

# contoh 2: range check
cmp rdi, 10
jl  too_small   # rdi < 10 → too_small
jg  too_big     # rdi > 10 → too_big
                # kalau reach sini, rdi == 10

# contoh 3: null check
cmp rax, 0
je  is_null     # pointer is null, jangan dereference!
mov rdi, [rax]  # safe to dereference
```

---

## 4. Comparing Strings
String dalam memory = array of bytes duduk bersebelahan, **mesti end dengan `0x0` (null terminator)**.

```text
String "hi" dalam memory:
Address:  0x1000   0x1001   0x1002
Content:  0x68     0x69     0x00
           'h'      'i'    null terminator
```

Program compare byte by byte — `[rdi]` vs `[rsi]`:

```asm
# manual byte-by-byte compare
mov al, BYTE PTR [rdi]     # load 1 byte dari string 1
cmp al, BYTE PTR [rsi]     # compare dengan string 2
jne different              # tak sama → strings different
cmp al, 0x0                # check null terminator
je  same                   # sama dan dah habis → strings match
inc rdi                    # next char string 1
inc rsi                    # next char string 2
jmp loop                   # repeat
```

---

## 5. Array of Pointers
Dalam memory, **array of pointers** = setiap slot 8 bytes (64-bit), each slot holds an **address** yang point ke data sebenar.

```text
Array of string pointers kat address 0x404000:

Address:   Value (pointer)    Points to:
0x404000:  0x401100     →     "hello\0"  (kat 0x401100)
0x404008:  0x401110     →     "world\0"  (kat 0x401110)
0x404010:  0x401120     →     "pwn\0"    (kat 0x401120)
0x404018:  0x000000     →     NULL (end of array)
```

Setiap element = 8 bytes, so untuk access element ke-N: `base + (N * 8)`

```asm
# rdi = base address of pointer array
mov rax, [rdi]          # load pointer dari slot 0 (first string address)
mov rax, [rdi + 8]      # load pointer dari slot 1 (second string)
mov rax, [rdi + 16]     # load pointer dari slot 2 (third string)

# access actual string content (double dereference)
mov rax, [rdi]          # rax = address of first string
mov bl,  BYTE PTR [rax] # bl = first character of that string ('h')
```

Lukisan double dereference:
```text
rdi = 0x404000
         │
         ▼
   [0x404000] = 0x401100   ← pointer to string
                   │
                   ▼
             [0x401100] = 0x68 ('h')   ← actual data
```

---

## 6. Stack Strings (Reverse Engineering)
Compiler selalu build secret strings terus kat stack instead of storing them plaintext in `.data` — susah nak nampak dalam static analysis.

```asm
# compiler build string "ABC" kat stack
mov BYTE PTR [rsp],     0x41   # 'A'
mov BYTE PTR [rsp + 1], 0x42   # 'B'
mov BYTE PTR [rsp + 2], 0x43   # 'C'
mov BYTE PTR [rsp + 3], 0x00   # null terminator
```

Little Endian note: kalau store multi-byte value sekaligus, bytes stored backwards:
```asm
mov DWORD PTR [rsp], 0x43424100   # stores: 00 41 42 43 → "ABC\0" forward
```

**Cara baca dalam Ghidra/GDB:** Collect hex values in order by address → convert to ASCII.

---

## 7. Conditionals Without Jumps
Conditional jumps (`je`, `jne`) boleh slow down CPU (*branch misprediction*). Modern code selalu replace dengan:

* **`cmovcc`** — conditional move, copy data only if condition met, no jump
  ```asm
  cmp eax, ebx
  cmove ecx, edx    # kalau eax == ebx, copy edx → ecx (no jump)
  cmovg ecx, edx    # kalau eax > ebx, copy edx → ecx
  ```

* **`setcc`** — set 1 atau 0 dalam 8-bit register based on `cmp` result
  ```asm
  cmp eax, 10
  sete al           # kalau eax == 10, al = 1. Otherwise al = 0.
  setg al           # kalau eax > 10, al = 1. Otherwise al = 0.
  ```

---

## 8. Walking Pointer Pattern
Common pattern dalam loops — rdi jalan satu-satu through string sambil rsi count.

```asm
mov rdi, [rsp+16]   # rdi = address of first argument (argv[1])
mov rsi, 0          # rsi = counter = 0
```

`[rsp+16]` holds the **address** of the string, bukan string tu sendiri. So lepas `mov rdi, [rsp+16]`, rdi = something like `0x7fffffffed60`.

```text
STACK                          MEMORY
─────────────────────          ──────────────────────────────────────────
[rsp]    │   3        │  argc
[rsp+8]  │ 0xed40     │ ──────────────────────────► "/tmp/prog\0"
[rsp+16] │ 0xed60     │ ──┐  ← rdi = 0xed60
─────────────────────     │
                          │
                          ▼
                     0xed60  0xed61  0xed62  0xed63  0xed64  0xed65
                    ┌───────┬───────┬───────┬───────┬───────┬───────┐
                    │  'h'  │  'e'  │  'l'  │  'l'  │  'o'  │  \0  │
                    │ 0x68  │ 0x65  │ 0x6c  │ 0x6c  │ 0x6f  │ 0x00 │
                    └───────┴───────┴───────┴───────┴───────┴───────┘
                       ▲
                       rdi starts here, walks →→→→→→→→→→→→→→→→→→ until \0

LOOP TRACE:
iter 1: rdi=0xed60  [rdi]='h'  rsi: 0→1   inc rdi →
iter 2: rdi=0xed61  [rdi]='e'  rsi: 1→2   inc rdi →
iter 3: rdi=0xed62  [rdi]='l'  rsi: 2→3   inc rdi →
iter 4: rdi=0xed63  [rdi]='l'  rsi: 3→4   inc rdi →
iter 5: rdi=0xed64  [rdi]='o'  rsi: 4→5   inc rdi →
iter 6: rdi=0xed65  [rdi]=0x00 ──────────────────── je exit_loop ✓ (rsi = 5)
```

rdi = **walking pointer** (jalan char by char), rsi = **counter** (berapa langkah = panjang string).

## 8. Recognizing Loops
Loop berlaku bila ada instruction yang **jump backwards** — repeat the same operation. Asas kepada `for` dan `while`.

```asm
# for loop equivalent: i = 0; i < 10; i++
mov rcx, 0          # i = 0
loop_start:
  cmp rcx, 10
  jge loop_done     # i >= 10 → exit
  
  # loop body here
  inc rcx           # i++
  jmp loop_start    # repeat

loop_done:
```

---

## 9. Full Program Example — String Compare
Program lengkap yang compare input string dengan password, exit dengan code 0 kalau match.

```asm
.intel_syntax noprefix
.global _start

# Layout:
# argv[0] = program name  → [rsp + 8]
# argv[1] = user input    → [rsp + 16]

_start:
    mov rdi, [rsp + 16]     # rdi = pointer to argv[1] (user input)

    # build password "pwnm3" on stack
    mov BYTE PTR [rsp - 8],  0x70   # 'p'
    mov BYTE PTR [rsp - 7],  0x77   # 'w'
    mov BYTE PTR [rsp - 6],  0x6e   # 'n'
    mov BYTE PTR [rsp - 5],  0x6d   # 'm'
    mov BYTE PTR [rsp - 4],  0x33   # '3'
    mov BYTE PTR [rsp - 3],  0x00   # null terminator
    lea rsi, [rsp - 8]      # rsi = pointer to password on stack

compare_loop:
    mov al, BYTE PTR [rdi]  # load 1 byte from input
    cmp al, BYTE PTR [rsi]  # compare with password byte
    jne wrong               # mismatch → fail
    cmp al, 0x0             # check null terminator
    je  correct             # both reached null → match!
    inc rdi                 # next input char
    inc rsi                 # next password char
    jmp compare_loop        # repeat

correct:
    mov rdi, 0              # exit code 0 = success
    mov rax, 60
    syscall

wrong:
    mov rdi, 1              # exit code 1 = wrong password
    mov rax, 60
    syscall
```

```bash
# build & test
as -o prog.o prog.s && ld -o prog prog.o

./prog pwnm3   # → exit code 0 (correct)
echo $?        # 0

./prog hello   # → exit code 1 (wrong)
echo $?        # 1
```
