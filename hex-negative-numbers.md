# Hex & Negative Numbers — Notes

## 1. Bit, Nibble, Byte & Hex

Computers store data as `0` and `1` (binary), tapi manusia baca dalam **Hex (Base 16)** sebab lebih pendek dan kemas.

* **1 Bit** — `0` atau `1`
* **1 Nibble** — **4 bits** = exactly **1 hex digit** (`0` to `f`)
* **1 Byte** — **8 bits** (2 nibbles) = exactly **2 hex digits** (`0x00` to `0xff`)

### Convert Binary → Hex (formula 8-4-2-1)
Every 4 bits dari kiri ke kanan ada fixed weight **`8 - 4 - 2 - 1`**:

```text
Binary:  1    1    1    1
Weight:  8  + 4  + 2  + 1  =  15  →  0xf
```

* `0–9` = kekal `0–9`
* `10` = **`a`**, `11` = **`b`**, `12` = **`c`**, `13` = **`d`**, `14` = **`e`**, `15` = **`f`**

Contoh: `1010` = `8 + 0 + 2 + 0` = `10` = **`0xa`**

---

## 2. Negative Numbers — Two's Complement

CPU tak ada simbol `-`, so dia guna **Two's Complement** — bayangkan macam odometer kereta yang pusing ke belakang dari `0`:

```text
0000 0010 (0x02)  =  +2
0000 0001 (0x01)  =  +1
0000 0000 (0x00)  =   0
1111 1111 (0xff)  =  -1   ← meter pusing balik
1111 1110 (0xfe)  =  -2
1111 1101 (0xfd)  =  -3
```

### 3 steps tukar positif jadi negatif (contoh: `-5` dalam 8-bit)
1. Tulis `5` dalam binary: `0000 0101`
2. **Flip semua bits**: `1111 1010`
3. **+1**: `1111 1010 + 1` = **`1111 1011`** (`0xfb` = **`-5`**)

### Quick read masa tengok Ghidra / GDB
* Nampak penuh `f` macam **`0xff`** (8-bit), **`0xffffffff`** (32-bit), atau **`0xffffffffffffffff`** (64-bit) → tu sebenarnya **`-1`**
* Any hex value yang start dengan digit **`8` to `f`** (leftmost bit = `1`) = **negative number** kalau baca as signed integer

---
