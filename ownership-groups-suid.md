# File Ownership, Groups & SUID — Notes

## 1. Users & Ownership (`chown`)
Setiap fail dalam Linux ada **Owner** (pemilik individu) dan **Group** (kumpulan pemilik).
* **`root` (UID 0):** Superuser. Boleh baca/ubah apa sahaja dalam sistem.
* **`chown`:** Tukar owner fail. Biasanya hanya `root` boleh buat ni.

```bash
chown hacker /flag        # Tukar owner /flag ke user 'hacker'
chown hacker:hacker file  # Tukar owner DAN group serentak
```

---

## 2. Groups & `id` (`chgrp`)
Group guna untuk kongsi access resource kepada beberapa user tanpa kena bagi kuasa `root` penuh. Contoh: group `video` untuk monitor, `sudo` untuk admin access, `docker` untuk containers.

```bash
hacker@dojo:~$ id
uid=1000(hacker) gid=1000(hacker) groups=1000(hacker)
```

```bash
chgrp hacker /flag        # Tukar group /flag kepada 'hacker'
chgrp grp_abc /flag       # Tukar ikut nama group sebenar dari output 'id'
```
*(Nama group tak semestinya sama dengan username — guna `id` dulu untuk confirm nama group sebenar.)*

---

## 3. SUID — Pinjam Kuasa Owner Fail
SUID allow normal user run program guna kuasa **owner asal fail tu** (contoh: `root`), bukan kuasa user yang run.

* Dalam `ls -l`, huruf **`s`** gantikan `x` di slot Owner — contoh: `-rwsr-xr-x`
* User tetap kena ada `x` permission kat slot Others untuk boleh start program tu.

### Camne Set SUID

Dalam format octal 4-digit, digit paling depan = **Special Permission Bit** (`4` = SUID, `2` = SGID, `1` = Sticky Bit):

| Cara | Command | Output `ls -l` |
| :--- | :--- | :--- |
| **Octal** | `chmod 4755 /challenge/getroot` | `-rwsr-xr-x` |
| **Symbolic** | `chmod u+s /challenge/getroot` | `-rwsr-xr-x` |

```bash
chmod 4755 /challenge/getroot   # Set SUID — 4 (SUID) + 755 (rwxr-xr-x)
/challenge/getroot              # Run → dapat root shell (#) → boleh cat /flag
```

*(Kalau fail SUID milik `root` boleh di-modify oleh attacker, terus jadi pintu masuk untuk dapat full `root` access.)*
