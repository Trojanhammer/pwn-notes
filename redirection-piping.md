# Redirection & Piping — Notes

## 1. File Descriptors (FD)
Every program ada 3 data channels:
* **`0` (`stdin`):** Input — from keyboard or file.
* **`1` (`stdout`):** Normal output — print to screen.
* **`2` (`stderr`):** Error messages — print to screen jugak tapi saluran lain.

*(Kalau tulis `>` tanpa nombor, Linux auto anggap `1>`.)*

---

## 2. Output Redirection (`>` dan `>>`)
Redirect output program ke fail instead of screen.

* **`>`** — overwrite (padam isi lama kalau fail dah ada)
* **`>>`** — append (sambung kat bawah, tak padam isi lama)

```bash
echo pwn > outfile         # tulis "pwn" ke outfile (isi lama dipadam)
echo college >> outfile    # sambung "college" kat bawah
cat outfile                # output: pwn \n college

/challenge/run > myflag    # save output program ke fail myflag
```

---

## 3. Error Redirection (`2>`)
Capture stderr je, berasingan dari stdout.

```bash
/challenge/run 2> errors.log                    # save error je ke errors.log
/challenge/run > output.log 2> errors.log       # asingkan output & error ke 2 fail
```

---

## 4. Input Redirection (`<`)
Feed isi fail masuk ke stdin program — macam kita type sendiri tapi dari fail.

```bash
echo yo > message      # save "yo" ke fail message
rev < message          # feed fail message ke rev → output: oy
/challenge/run < PWN   # feed isi fail PWN sebagai input
```

---

## 5. Piping (`|`)
Connect terus stdout kiri jadi stdin kanan — tak perlu save fail sementara.

```bash
echo yes-yes | grep yes               # filter "yes" terus dari output echo
/challenge/run | grep "pwn.college{"  # cari flag dari ribuan baris output
```

---

## 6. Gabungkan Error Ke Dalam Pipe (`2>&1`)
Pipe `|` only connect stdout (`1`), tak connect stderr (`2`). Kalau nak filter error guna grep, kena gabung dulu.

```bash
# 2>&1 = redirect stderr (2) masuk sama dengan stdout (1)
/challenge/run 2>&1 | grep "pwn.college{"
```
*(Penting: `2>&1` kena tulis SEBELUM `|`)*

---

## 7. `grep -v` & `sed`

### `grep -v` — Buang baris yang ada keyword
```bash
/challenge/run | grep -v DECOY    # buang semua baris yang ada "DECOY"
```

### `sed` — Find & replace dalam baris
Syntax: `sed "s/lama/baru/g"` (`s` = substitute, `g` = all occurrences in line)

```bash
echo "pwn.college{SAMPAHflagSAMPAH}" | sed "s/SAMPAH//g"
# output: pwn.college{flag}
```

---

## 8. `tee` — T-Splitter
Split satu stream jadi dua: satu ke fail, satu terus ke stdout.

```bash
command_1 | tee cmd1_output | command_2    # save salinan sambil teruskan ke program lain
echo hi | tee pwn college                  # copy output ke dua fail + print ke screen
```

---

## 9. Process Substitution (`<(...)` & `>(...)`)
Buat output/input sesuatu command nampak macam temporary file — untuk program yang kena terima file argument.

```bash
# <(...) — read output command as file
diff <(ls /bin) <(ls /usr/bin)             # compare output dua command tanpa buat fail

# >(...) — write ke command as file
/challenge/hack | tee >(/challenge/the) | /challenge/planet

# asingkan stdout & stderr ke dua program berbeza
/challenge/hack 2> >(/challenge/the) | /challenge/planet
/challenge/hack > >(/challenge/planet) 2> >(/challenge/the)
```

---

## 10. FIFO (`mkfifo`) — Named Pipe In RAM
FIFO = persistent pipe file yang connect dua process terus dalam RAM, tak save ke disk.

* Dalam `ls -l`, starts with **`p`** (`prw-r--r--`)
* **Blocking:** Process yang write ke FIFO akan tunggu sampai ada process lain yang baca

```bash
mkfifo /tmp/flag_fifo              # buat FIFO
/challenge/run > /tmp/flag_fifo &  # hantar output ke FIFO kat background (& = jangan block terminal)
cat /tmp/flag_fifo                 # baca dari FIFO (sekali baca terus hilang)
```
