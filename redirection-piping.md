# Redirection & Piping — Quick Notes

## 1. File Descriptors (FD) — 3 Saluran Komunikasi Program
Setiap program dalam Linux mempunyai 3 saluran data bernombor:
* **`0` (`stdin`):** Standard Input (data masuk dari papan kekunci / fail).
* **`1` (`stdout`):** Standard Output (hasil output biasa ke skrin).
* **`2` (`stderr`):** Standard Error (mesej ralat / arahan sistem ke skrin).

*(Nota: Jika menulis `>` tanpa nombor, Linux secara automatik anggap ia adalah `1>`.)*

---

## 2. Output Redirection (`>` dan `>>`)
Menyalurkan output program ke dalam fail (bukannya tercetak di skrin).

* **`>` (Overwrite / Timpa):** Cipta fail baharu atau **padam isi lama** jika fail sudah wujud.
* **`>>` (Append / Tambah):** Sambung teks baharu di **baris paling bawah** tanpa memadam isi lama.

```bash
echo pwn > outfile         # Tulis "pwn" (isi lama dipadam)
echo college >> outfile    # Tambah "college" di bawah "pwn"
cat outfile                # Output: pwn \n college

/challenge/run > myflag    # Simpan output program ke fail myflag
```

---

## 3. Error Redirection (`2>`)
Menangkap saluran ralat (`FD 2`) secara berasingan daripada output biasa (`FD 1`).

```bash
/challenge/run 2> errors.log                      # Simpan mesej error sahaja ke errors.log
/challenge/run > output.log 2> errors.log         # Asingkan output biasa & error ke 2 fail berbeza
```

---

## 4. Input Redirection (`<`)
Menyuapkan isi fail masuk ke dalam mulut program (`stdin`) seolah-olah kita menaipnya di papan kekunci.

```bash
echo yo > message          # Simpan teks "yo" ke fail message
rev < message              # Suapkan fail message ke program rev → Output: oy
/challenge/run < PWN       # Suapkan isi fail PWN sebagai input kepada /challenge/run
```

---

## 5. Piping (`|`) — Sambung Output Terus Ke Program Lain
"Buang orang tengah" — sambungkan terus `stdout` (`1`) arahan kiri menjadi `stdin` (`0`) arahan kanan tanpa perlu simpan fail sementara.

```bash
echo yes-yes | grep yes               # Tapis perkataan "yes" terus dari output echo
/challenge/run | grep "pwn.college{"  # Cari flag terus dari ribuan baris output tanpa buat fail
```

---

## 6. Menggabungkan Error Ke Dalam Pipe (`2>&1`)
Operator paip `|` **hanya menyambungkan `stdout` (`1`)** dan tidak menyambungkan `stderr` (`2`). Untuk menapis mesej ralat menggunakan `grep`, gabungkan saluran `2` masuk ke saluran `1` dahulu.

```bash
# 2>&1 = Alihkan stderr (2) masuk ke tempat yang sama dengan stdout (1)
/challenge/run 2>&1 | grep "pwn.college{"
```
*(Susunan penting: `2>&1` mesti ditulis **sebelum** simbol paip `|`!)*

---

## 7. Penapisan Teks Lanjutan (`grep -v` & `sed`)

### `grep -v` (Invert Match — Tapis Keluar Yang Tak Nak)
```bash
/challenge/run | grep -v DECOY    # Buang semua baris yang ada perkataan "DECOY"
```

### `sed` (Stream Editor — Cari & Ganti Dalam Baris Sama)
Sintaks: `sed "s/lama/baru/g"` (`s` = substitute, `g` = global)

```bash
echo "pwn.college{SAMPAHflagSAMPAH}" | sed "s/SAMPAH//g"
# Output: pwn.college{flag}
```

---

## 8. `tee` — Penyambung Paip Simpang Tiga ("T-Splitter")
Membelah satu aliran data kepada dua laluan serentak: satu salinan disimpan ke fail, satu lagi diteruskan ke `stdout`.

```bash
command_1 | tee cmd1_output | command_2    # Debug: simpan salinan sambil teruskan ke program lain
echo hi | tee pwn college                  # Salin output ke dua fail sekaligus + cetak ke skrin
```

---

## 9. Process Substitution (`<(...)` & `>(...)`)
Menyamar output/input sesuatu arahan supaya kelihatan seperti fail sementara untuk program yang wajib menerima argumen fail.

```bash
# <(...) — Baca output arahan sebagai fail
diff <(ls /bin) <(ls /usr/bin)             # Bandingkan output dua arahan tanpa buat fail sementara

# >(...) — Tulis ke dalam arahan sebagai fail
/challenge/hack | tee >(/challenge/the) | /challenge/planet

# Asingkan stdout & stderr ke dua program berbeza
/challenge/hack 2> >(/challenge/the) | /challenge/planet
/challenge/hack > >(/challenge/planet) 2> >(/challenge/the)
```

---

## 10. FIFO (`mkfifo`) — Named Pipe Kekal Dalam Memori
FIFO (*First-In, First-Out*) ialah fail paip kekal yang menghubungkan dua proses secara langsung dalam RAM tanpa menyimpan data ke cakera keras.

* Pada `ls -l`, bermula dengan huruf **`p`** (`prw-r--r--`).
* **Sifat *Blocking*:** Proses yang menulis ke FIFO akan menunggu sehingga ada proses lain datang membaca.

```bash
mkfifo /tmp/flag_fifo              # Cipta fail paip FIFO
/challenge/run > /tmp/flag_fifo &  # Hantar output ke FIFO di latar belakang (& = background)
cat /tmp/flag_fifo                 # Baca data dari FIFO (sekali baca terus hilang)
```
