# Processes & Jobs — Notes

## 1. Tengok Running Processes (`ps`)
Every time kita run program, Linux create satu **Process** dengan unique ID dipanggil **PID**.

* **`ps`** (default): Cuma tunjuk processes dalam current terminal je — kurang useful.
* **`ps -ef`**: `-e` (every process) + `-f` (full format). Bagus untuk tengok **PPID** (parent process yang launch dia).
* **`ps aux`**: `a` (all users) + `x` (background processes) + `u` (user-readable). Bagus untuk check `%CPU`, `%MEM`, dan `STAT`.
* **`ww`**: Tambah kalau nama program panjang sangat sampai kena truncate kat hujung screen.

```bash
ps -efww                    # list semua processes + full path tanpa truncate
ps auxww | grep challenge   # cari specific process
```

### Contoh Output `ps aux`
```text
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   1056   752 ?        Ss   02:41   0:00 /sbin/init
root         7  0.0  0.0   7136  2812 ?        S    02:41   0:00 /bin/sleep 6h
user       104  0.0  0.0   4716  3556 pts/0    Ss   02:41   0:00 /bin/bash
user       110  0.0  0.0 233508  5528 pts/0    S    02:41   0:00 /bin/bash --login
user       127  0.0  0.0 232860  4924 pts/1    Ss+  02:41   0:00 /bin/bash --login
user       144  0.0  0.0 235228  4436 pts/0    R+   02:44   0:00 ps aux
```
* `Ss` — Sleeping, tapi dia pula jadi parent/session leader untuk processes lain
* `S` — Sleeping (tunggu input atau masa)
* `Ss+` — Sleeping + session leader + foreground
* `R+` — Running + foreground (ps sendiri tengah jalan)
* `Sl` — Sleeping + multi-threaded
* `T` — Stopped/suspended (kena `Ctrl+Z`)

### Columns Penting
* **`PID`:** Unique ID process (cth: `1337`)
* **`PPID`:** ID parent process yang spawn dia
* **`TTY`:** Terminal mana process tu duduk (`pts/0` = terminal kita, `?` = background daemon)
* **`STAT`:** Status process:
  * `R` — Running (actively using CPU)
  * `S` — Sleeping (active, tapi tengah tunggu input/masa)
  * `T` — Stopped (suspended sebab `Ctrl+Z`)
  * `+` — Foreground (kunci keyboard kita). Bila masuk background, tanda `+` hilang.

---

## 2. Kill & Send Signal (`kill`)
Kalau program hang atau degil, hantar **Signal** guna `kill`.

| Signal | No. | Command | Maksud |
| :--- | :--- | :--- | :--- |
| **`SIGINT`** | `2` | `Ctrl+C` / `kill -2 <PID>` | Interrupt — minta stop cara baik (boleh di-ignore). |
| **`SIGTERM`** | `15` | `kill <PID>` (default) | Terminate — suruh close cleanly (kalau degil boleh buat tak tahu). |
| **`SIGKILL`** | `9` | `kill -9 <PID>` | Force kill — kernel terus bunuh, program tak boleh block atau ignore. |

```bash
ps aux | grep misbehaving   # step 1: cari PID (cth: dapat 4521)
kill 4521                   # step 2: cuba SIGTERM dulu
kill -9 4521                # step 3: kalau masih hidup, force kill
```

---

## 3. Foreground, Background & `jobs`
* **Foreground:** Process duduk depan, **kunci terminal** sampai habis. Ada tanda `+` dalam STAT.
* **Background:** Process jalan senyap belakang, **terminal bebas** untuk type command lain.

| Command | Fungsi |
| :--- | :--- |
| `Ctrl+Z` | Freeze/suspend current foreground process (STAT jadi `T`) |
| `jobs` | List semua stopped/background processes dalam current terminal |
| `bg` | Sambung process yang kena `Ctrl+Z` kat background |
| `fg` | Tarik process dari background/stopped ke foreground |
| `&` | Launch program terus ke background dari awal |

```bash
# Cara 1: Ctrl+Z + bg
/challenge/run              # start foreground (terminal locked)
# Ctrl+Z                    # freeze → [1]+ Stopped (STAT jadi T)
bg                          # sambung kat background → [1]+ /challenge/run &
/challenge/run              # boleh run process kedua sekarang

# Cara 2: terus background dari awal
sleep 100 &                 # → [1] 1234
sleep 200 &                 # → [2] 1235
sleep 300 &                 # → [3] 1236

jobs                        # tengok senarai semua
# output:
# [1]   Running    sleep 100 &
# [2]-  Running    sleep 200 &
# [3]+  Running    sleep 300 &
# (tanda + = job paling recent, - = job sebelum tu)

fg %2                       # tarik job 2 (sleep 200) ke foreground
# Ctrl+Z                    # freeze dia → STAT jadi T
bg %2                       # sambung balik job 2 kat background
fg %1                       # tarik job 1 ke foreground

# fg tanpa nombor = amek job yang ada tanda +
# tanda + = "current job" (job paling recent/last yang kita interact)
fg                          # sama macam fg %3 (dalam contoh ni sleep 300 ada +)
```

*(Process kat background tak boleh baca keyboard input. Kalau dia cuba, Linux auto-freeze dia jadi `Stopped` sampai kita `fg` balik.)*

---

## 4. Exit Codes (`$?`)
Every program yang habis jalan return satu nombor status — **Exit Code**: `0` = success, `1–255` = error.

* **`$?`:** Bash variable yang simpan exit code **command paling terakhir**.
* **Penting:** `$?` overwrite setiap kali kita run command baru (termasuk `echo`!).

```bash
/challenge/get-code         # program keluar dengan code rawak (cth: 52)
/challenge/submit-code $?   # terus hantar $? sebelum command lain overwrite!

# kalau nak save dulu
/challenge/get-code
SAVED_CODE=$?               # save dalam variable sendiri
echo "Kod tadi: $SAVED_CODE"  # SAVED_CODE takkan hilang
```
