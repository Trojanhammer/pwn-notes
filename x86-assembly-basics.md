# x86-64 Assembly Basics — Notes

## 1. Registers & `mov`

### Register tu apa?
CPU think in simple terms — move data, change data, make decisions based on data. Semua tu perlukan tempat simpan data, and that's what registers are for.

Bayangkan camni:
* **RAM** = almari besar kat tepi bilik. Banyak ruang, tapi kena jalan pergi ambil dulu.
* **Register** = atas meja CPU terus depan mata. Sikit je ruang, tapi **instant access**.

CPU kena angkut data dari RAM ke register dulu before boleh process apa-apa. Register tu the most expensive + fastest hardware dalam komputer — sebab tu dia constrained gila, x86-64 ada **16 general purpose registers** je yang code boleh guna, plus a few dozen lagi for special purposes.

`mov dst, src` bukan "pindah" — dia **copy** nilai dari kanan ke kiri. Nilai asal kekal tak hilang.

### Registers Penting (Syscall & Arguments)
* **`rax`** — syscall number (cth: `60` = `sys_exit`) atau return value
* **`rdi`** — argument ke-1
* **`rsi`** — argument ke-2
* **`rdx`** — argument ke-3

Cara ingat urutan: **DI → SI → DX** = arg **1 → 2 → 3**

```asm
mov rax, 60       # set rax = 60
mov rdi, 42       # set rdi = 42
mov rdi, rsi      # copy nilai rsi masuk rdi (rsi tak hilang)
```

---

## 2. CPU vs Kernel & Segfault

CPU runs sendiri baris demi baris tanpa henti. Kernel **tak check** setiap instruction — kalau buat camtu komputer jadi lembab gila.

### Bila je Kernel masuk campur?
1. **Program buat `syscall`** — program ketuk pintu Kernel (Ring 3 → Ring 0) minta servis macam exit, write, read.
2. **CPU langgar memori unmapped** — MMU (hardware) bunyikan alarm, Kernel bangun dan bunuh program → **Segmentation fault**.

### Kenapa tak semua memori dimapped?
Setiap program ada **Virtual Address Space** sendiri. Kernel only connect RAM fizikal (4096 bytes per page) kat kawasan yang program tu **betul-betul guna** je — semua address lain kosong/unmapped. Kalau program terkeluar dari kawasan dia, terus segfault. Sebab tu satu program tak boleh baca memori program lain.

### Apa jadi kalau `mov rax, 60` tanpa `syscall`?
```text
Virtual Address Space
┌──────────────────────────────────────────────────────┐
│  0x0000000000000000  →  UNMAPPED (sentuh = segfault) │
├──────────────────────────────────────────────────────┤
│  0x401000:  mov rax, 60   (7 bytes)                  │
│  0x401007:  00 00         (kosong = add [rax], al)   │
│             CPU cuba tulis ke addr 0x3c → UNMAPPED   │
│             💥 Segmentation fault                    │
└──────────────────────────────────────────────────────┘
```

Tanpa `syscall`, CPU tak tahu nak stop. Lepas last instruction, dia baca bytes kosong `00 00` kat address seterusnya — tu translate jadi `add [rax], al` dalam machine code — cuba tulis ke address `60` (`0x3c`) yang unmapped → segfault.

*(Same goes kalau guna syscall lain macam `sys_write` — lepas Kernel siap print, dia return control balik ke CPU. Kalau takde `exit` kat hujung, CPU terus lari ke memori kosong dan segfault.)*

### Contoh program betul (`exit(42)`)
```asm
mov rdi, 42       # arg 1: exit code = 42
mov rax, 60       # syscall 60 = sys_exit
syscall           # ketuk Kernel, program close elok
```
```bash
./program
echo $?           # output: 42
```

---

## 3. Build Assembly: `as` & `ld`

CPU tak boleh run `.s` file terus — kena convert ke Linux binary (**ELF — Executable and Linkable Format**).

**3 steps:**
1. **Tulis `.s`** — letak `.intel_syntax noprefix` kat atas (tak payah guna `%` dan `$` style AT&T)
2. **Assemble** (`as` → `.o`) — convert teks jadi machine code. Belum boleh run lagi, belum ada entry point
3. **Link** (`ld` → executable) — wrap jadi ELF dengan `_start` supaya Kernel tahu nak start dari mana

```bash
cat << 'EOF' > program.s
.intel_syntax noprefix
.global _start
_start:
    mov rdi, 42
    mov rax, 60
    syscall
EOF

as -o program.o program.s    # assemble
ld -o program program.o      # link

./program
echo $?                      # output: 42
```

