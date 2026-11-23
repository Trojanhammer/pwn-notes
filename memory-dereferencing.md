# Computer Memory (RAM & Dereferencing) — Notes

## 1. Camne RAM Disusun?
Unlike CPU registers (`rax`, `rdi`) yang ada nama sebab jumlahnya sikit, RAM terlalu besar — so setiap kotak diberi **nombor address** macam deretan lokar berkunci.

### 1 Address = 1 Byte
Each address only holds **1 byte**. Nak simpan nombor 64-bit (8 bytes), dia makan **8 lokar berturutan**:

```text
Address:  │ 0x401000 │ 0x401001 │ 0x401002 │ ... │ 0x401007 │
          ├──────────┼──────────┼──────────┼─────┼──────────┤
Data:     │   0x2a   │   0x00   │   0x00   │ ... │   0x00   │
          └──────────┴──────────┴──────────┴─────┴──────────┘
          ◄──────────────── 8 bytes (64-bit value = 42) ─────►
```

### Virtual Address Space (127TB vs 16GB RAM fizikal)
Walaupun PC ada 16GB physical RAM, every program gets its own virtual address map seluas **127 Terabytes**:

```text
Virtual Address Space
┌──────────────────────────────────────────────────────────────┐
│  0x0000  →  0xffff                                           │
│  ❌ UNMAPPED — locked by Kernel (prevent NULL pointer bugs)  │
├──────────────────────────────────────────────────────────────┤
│  0x10000  →  0x7fffffffffff  (~127 TB)                       │
│  ✅ User Space                                               │
│     • Only parts the program actually uses get mapped to RAM │
│     • Everything else unmapped → segfault kalau sentuh       │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. `[ ]` — Dereference (Access RAM)
`[ ]` dalam Intel syntax means **"open the RAM locker at this address and get/write its contents"** — same as `*ptr` in C/C++.

### Tanpa `[ ]` vs Dengan `[ ]`
```text
   [ CPU register: rax = 133700 ]
          │                    │
          │ mov rdi, rax       │ mov rdi, [rax]
          │ 0 jumps            │ 1 jump to RAM
          ▼                    ▼
     rdi = 133700       ┌──────────────────┐
                        │ RAM addr 133700  │
                        │ contents = 42    │ ──► rdi = 42
                        └──────────────────┘
```

* **`mov rdi, rax`** — copy nombor `133700` tu sendiri masuk rdi
* **`mov rdi, [rax]`** — treat `133700` as an address, go open that locker, grab the contents (`42`)

### `[133700]` (fixed address) vs `[rax]` (pointer)

Kalau `rax = 1234000`, then `[rax]` is exactly the same as `[1234000]` — both go to address `1234000` and grab whatever's stored there:

```asm
mov rax, 1234000      # rax = 1234000
mov rdi, [rax]        # same as: mov rdi, [1234000]
                      # → go to RAM address 1234000, load its contents into rdi
```

So `[ ]` works the same way whether you put a register or a literal address inside — it always means "go to this address in RAM and read/write there". The difference is just flexibility: literal address is fixed, `[rax]` changes depending on what rax holds at runtime.

---

## 3. Read vs Write to RAM
`mov` always: **`mov DESTINATION (left), SOURCE (right)`**

| `[ ]` position | Example | Meaning |
| :--- | :--- | :--- |
| **Right (read)** | `mov rdi, [133700]` | Open addr `133700`, load contents into `rdi` |
| **Left (write)** | `mov [133700], 42` | Go to addr `133700`, write value `42` there |
| **No `[ ]`, number on left** | `mov 133700, 42` | ❌ Error — bare number bukan valid destination |

---

## 4. Offset: `[rax + 8]`
Kadang data tak duduk tepat kat starting address — it's shifted a few bytes forward, like array elements or struct fields in C/C++.

```text
Address:    rax          rax+8         rax+16
         ┌──────────┬──────────────┬──────────────┐
Data:    │ Element 0│  Element 1   │  Element 2   │
         │ (8 bytes)│  (8 bytes)   │  (8 bytes)   │
         └──────────┴──────────────┴──────────────┘
                            ▲
                            └── mov rdi, [rax + 8]  ← grab Element 1
```

```asm
mov rdi, [rax + 8]    # go to (rax + 8 bytes), load contents into rdi
```

---

## 5. Double Dereference (`**ptr`)
Satu `[ ]` = **one jump** je. Kalau nilai sebenar hidden in 2 layers (locker 1 stores address of locker 2, locker 2 stores the actual value), kena **dereference twice**.

```text
┌──────────────┐    1. mov rdi, [rax]    ┌────────────────────┐
│ rax = addr_2 │ ──────────────────────► │ RAM addr_2         │
└──────────────┘                         │ contents = addr_1  │
                                         └─────────┬──────────┘
                                                   │
                                                   │ 2. mov rdi, [rdi]
                                                   ▼
┌──────────────┐                         ┌────────────────────┐
│ rdi = 42 ✓  │ ◄────────────────────── │ RAM addr_1         │
└──────────────┘                         │ contents = 42      │
                                         └────────────────────┘
```

```asm
.intel_syntax noprefix
mov rdi, [rax]    # jump 1: open addr in rax, get second address
mov rdi, [rdi]    # jump 2: open second addr, get actual value
mov rax, 60
syscall
```
