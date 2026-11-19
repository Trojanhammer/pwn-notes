# OS Architecture, Kernel Isolation & Secure Boot — Notes

## 1. Monolithic Kernel vs. Microkernel

Perbezaan paling besar: **kat mana drivers dan system services jalan** — dalam kernel space ke user space.

### A. Monolithic Kernel (Linux, FreeBSD)
Semua benda — file system, network stack, hardware drivers — duduk dalam satu kernel space yang sama.

```text
+-------------------------------------------------------------------+
|                        USER SPACE (Ring 3)                        |
|   [ Application A ]       [ Application B ]       [ Browser ]     |
+---------------------------------+---------------------------------+
                                  |  System Calls (Syscalls)
==================================v==================================
|                       KERNEL SPACE (Ring 0)                       |
|  +--------------+  +--------------+  +-------------------------+  |
|  | File System  |  | Network/WiFi |  | GPU & Hardware Drivers  |  |
|  +--------------+  +--------------+  +-------------------------+  |
|  +-------------------------------------------------------------+  |
|  |           Memory Management  &  CPU Scheduler               |  |
|  +-------------------------------------------------------------+  |
+-------------------------------------------------------------------+
```

* **Pros:** Laju — semua komponen dalam kernel boleh communicate terus tanpa context switching.
* **Cons:** Attack surface besar. Satu bug (buffer overflow / UAF) dalam mana-mana driver = penyerang dapat full control OS.

---

### B. Microkernel (Nintendo Horizon OS, QNX, seL4)
Kernel dikecilkan — handle RAM, CPU scheduling, dan IPC je. Drivers dan file system diasingkan ke user space masing-masing.

```text
+-------------------------------------------------------------------+
|                        USER SPACE (Ring 3)                        |
|  +-------------+   +-------------+   +-----------+   +---------+  |
|  | App / Game  |   | File System |   | Bluetooth |   |   GPU   |  |
|  |  (Sandbox)  |   |   Service   |   |  Service  |   | Driver  |  |
|  +------+------+   +------+------+   +-----+-----+   +----+----+  |
|         |                 ^                ^              ^        |
+---------|-----------------|----------------|--------------|--------+
          | (1) Send msg    | (2) Route      |              |
          +------->---------+                |              |
====================|========================|==============|=========
|                   v      MICROKERNEL (Ring 0)                      |
|         [ IPC Message Router ]  +  [ RAM & CPU ]                   |
+-------------------------------------------------------------------+
```

* **Pros:** Kalau browser atau bluetooth process kena exploit, penyerang stuck dalam kotak process tu je — tak boleh reach microkernel atau memory app lain.
* **Cons:** Kalau file system service ada bug waktu process IPC message dari app biasa, penyerang boleh ambil alih service tu — tak perlu tembus microkernel pun.

---

## 2. Defense-in-Depth: Lapisan Security Sistem Moden

Camne exploit chain kena pergi layer by layer:

```text
  Userland Apps / Browser          (Ring 3)
            │
     kena kernel exploit
            ▼
  OS Kernel (FreeBSD/Linux)        (Ring 0)
            │
     kena hypervisor exploit
            ▼
  Hypervisor / Secure Monitor      (Ring -1 / EL2)
            │
     dikunci dalam silikon
            ▼
  Hardware Security Processor & BootROM
```

* **Runtime exploit:** Inject code ke RAM je. Reboot = hilang semua, sebab takde apa berubah dalam storage.
* **Persistent modification:** Tukar benda dalam SSD/NAND secara kekal. Perkakasan moden halang ni via Hardware Root of Trust.

---

## 3. Hardware Root of Trust & Secure Boot

Camne peranti pastikan OS yang boot tu genuine dan tak kena tamper?

```text
[ VENDOR SERVER ]                      [ CHIP DALAM DEVICE ]
  Private Key                            Public Key
  (Rahsia, vendor pegang)                (Burned dalam BootROM — Read Only)
         │                                        │
         ▼                                        ▼
  Sign firmware                          1. Power on → BootROM bangun
  → dapat digital signature              2. Check signature bootloader guna public key
         │                                        │
         +---------> [ SSD / FLASH ] -------------+
                       Simpan firmware             │
                       + signature        +--------+--------+
                                          |                 |
                                     [ VALID ]         [ INVALID ]
                                          │                 │
                                          ▼                 ▼
                                    Boot OS            Halt / Sekat
```

### Konsep Penting

1. **BootROM:** Code paling awal yang dibakar terus dalam silikon masa kilang. Read-only — tak boleh padam atau update via software.

2. **Public vs Private Key:**
   * **Private key (vendor pegang):** Satu-satunya cara nak sign firmware rasmi.
   * **Public key (dalam BootROM):** Untuk verify signature je. Walaupun orang berjaya dump BootROM, dapat public key pun tak guna — tak boleh sign firmware palsu.

3. **Chain of Trust:** BootROM verify Bootloader → Bootloader verify Hypervisor → Hypervisor verify Kernel. Satu byte berubah tanpa signature sah = rantaian putus = sistem refuse boot.
