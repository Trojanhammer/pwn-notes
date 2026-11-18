# Linux Basics & System Deep Dive — Quick Notes

## 1. Prompt & Shell (`bash`)
`hacker@dojo:~$`
* `hacker` = username
* `dojo` = hostname
* `~` = lokasi semasa (shorthand untuk home directory `/home/hacker`; bertukar jika `cd` ke folder lain)
* `$` = normal user, `#` = root/admin

### Apa itu `bash` & Shebang (`#!`)
* **`bash`** (*Bourne Again SHell*): Program penterjemah arahan (*shell*) yang membaca teks yang kita taip di terminal dan melaksanakannya.
* **`#!/bin/bash` (Shebang)**: Baris pertama dalam fail skrip yang memberitahu Linux penterjemah mana perlu digunakan.

---

## 2. Navigation & Path Types
```bash
pwd                 # print working directory (tunjuk lokasi penuh sekarang)
cd /some/dir        # tukar directory guna absolute path
cd                  # tanpa argument = terus balik ke home directory (/home/hacker)
cd ~                # sama seperti cd
```

### Path Types

| Type | Contoh | Bermula dari |
| :--- | :--- | :--- |
| **Absolute** | `/tmp/a/b/file` | root `/` |
| **Relative** | `a/b/file` | current directory |

### Special Path Symbols & Tips
```text
.   = current directory  → ./run   (wajib letak ./ sebab Linux tak auto cari dalam cwd)
..  = parent directory   → ../file (pergi satu folder ke atas)
~   = home directory     → ~/file  (sama dengan /home/hacker/file)
```

---

## 3. File Commands
### Membaca & Mencari Teks (`cat`, `grep`, `diff`, `man`)
```bash
cat file                                  # baca satu file
cat file1 file2                           # baca dua file sekaligus (concatenate)
cat "my file"                             # guna tanda petik jika nama file ada ruang kosong
cat -- -file                              # guna -- jika nama file bermula dengan tanda -
grep pwn.college /challenge/data.txt      # cari baris yang ada perkataan "pwn.college"
man ls                                    # buka buku manual rasmi untuk arahan ls
```

#### `diff` — Bandingkan dua fail
```bash
diff file1 file2
```
Contoh output perubahan (`c` = changed, `a` = added, `d` = deleted):
```text
2c2
< world        # < = baris asal dari file1
---
> universe     # > = baris baharu dari file2 (line 2 berubah)

1a2
> college      # selepas line 1 file1, file2 ada 1 line tambahan
```

### Menyenaraikan Fail (`ls`)
```bash
ls                  # list files dalam cwd
ls /challenge       # list files dalam folder lain
ls -a               # -a = all (tunjuk hidden files bermula dengan .)
ls -l               # -l = long (tunjuk details penuh)
ls -lah             # -h = human-readable (tukar 4096 bytes kepada 4.0K)
```

Contoh output `ls -la`:
```text
drwxr-xr-x  2 hacker hacker  4096 Nov 18 myfolder
-rw-r--r--  1 hacker hacker   123 Nov 18 myfile
lrwxrwxrwx  1 root   root       5 Nov 18 not-the-flag -> /flag
```
* Huruf pertama: `d` = directory, `-` = file biasa, `l` = symbolic link
* `rwxr-xr-x` = permissions (`owner` / `group` / `others`)
* `hacker hacker` = owner, group
* `4096` = size dalam bytes (untuk folder, ini ialah saiz blok metadata sistem fail)

### Mencipta, Memindah, & Memadam (`touch`, `mkdir`, `rm`, `mv`, `cp`)
```bash
touch /tmp/pwn            # buat file kosong
mkdir /tmp/myfolder       # buat directory baharu

rm file                   # delete file
rm -r /tmp/myfolder       # -r = recursive (wajib untuk padam folder & isinya)
mv oldname newname        # rename file dalam folder sama
mv file /tmp/             # move file ke folder lain
cp file /tmp/             # copy file
cp -r folder1 folder2     # copy keseluruhan folder & isinya
```

### `ln` — Symbolic Link (Symlink)
Symlink ialah fail jalan pintas (*shortcut*) yang menunjuk ke lokasi fail asal.
```bash
ln -s TARGET LINK_NAME

# Contoh:
ln -s /flag ~/not-the-flag
cat ~/not-the-flag        # baca isi /flag melalui symlink
file ~/not-the-flag       # output: symbolic link to /flag
```
*(Nota: `LINK_NAME` mesti belum wujud; jika sudah ada, padam dahulu guna `rm` sebelum buat `ln -s`.)*

---

## 4. Deep Dive: `$PATH`, SUID & Binary Inspection

### Susunan `$PATH` & `which`
* **`$PATH`**: Senarai folder (dipisahkan oleh `:`) yang dicari oleh Linux dari **kiri ke kanan** apabila kita menaip arahan tanpa laluan penuh.
```bash
env                       # lihat semua environment variables termasuk $PATH
which bash                # tunjuk lokasi bash pertama yang dijumpai dalam $PATH
which -a bash             # tunjuk semua versi bash yang wujud (cth: /run/dojo/bin/bash & /bin/bash)
```

### SUID & `exec-suid`
* **SUID (`4755` / `-rwsr-xr-x`)**: Membolehkan program dijalankan menggunakan kuasa **pemilik asal fail tersebut** (contohnya `root`), bukannya pengguna biasa (`hacker`).
* **`/usr/bin/exec-suid`**: Program pembungkus (*wrapper*) khas `pwn.college` yang membolehkan skrip Bash dijalankan dengan kuasa SUID `root` sambil mengunci pemboleh ubah berbahaya.
```bash
find / -perm -4000 2>/dev/null   # cari semua program berkuasa SUID dalam sistem
```

### Alatan Siasatan Binari & Proses
```bash
file /usr/bin/exec-suid          # kenal pasti jenis sebenar fail (ELF 64-bit, symlink, dll)
strings /usr/bin/exec-suid       # ekstrak teks boleh baca manusia daripada fail binari
xxd /usr/bin/exec-suid | head    # lihat hex & magic bytes (7f45 4c46 = .ELF)
ps aux                           # senaraikan semua proses yang sedang berjalan
strace /challenge/check          # intip system calls antara program dan kernel Linux
```
*(Elak guna `cat` pada fail binari — akan rosakkan paparan terminal. Jika tertekan, taip `reset`.)*

---

## 5. Asas Bash Scripting (`if ... fi`)
Dalam skrip Bash, blok `if` sentiasa ditutup dengan ejaan terbaliknya iaitu `fi`.

```bash
#!/usr/bin/exec-suid -- /bin/bash -p

# ! bermaksud NOT
# -f menyemak sama ada fail wujud
if ! [ -f /tmp/hack-the-planet ]
then
    echo "Fail tidak wujud!"
    exit 1
fi

# -q bermaksud quiet (cari teks tanpa cetak output ke skrin)
if ! grep -q "pwn.college{" /tmp/hack-the-planet
then
    echo "Tiada flag dalam fail ini!"
    exit 1
fi

cat /tmp/hack-the-planet
```
