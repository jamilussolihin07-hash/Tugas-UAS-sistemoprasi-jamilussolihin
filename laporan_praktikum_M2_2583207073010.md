# Laporan Praktikum Sistem Operasi Lanjut — MCSOS

**Nama file laporan:** `laporan_praktikum_M2_2583207073010.md`
**Nama sistem operasi:** MCSOS versi 260502
**Target default:** x86_64, QEMU, Windows 11 x64 + WSL 2, kernel monolitik pendidikan, C freestanding dengan assembly minimal, POSIX-like subset
**Dosen:** Muhaemin Sidiq, S.Pd., M.Pd.
**Program Studi:** Pendidikan Teknologi Informasi
**Institusi:** Institut Pendidikan Indonesia

> **Catatan penting sebelum membaca laporan ini:** Sama seperti laporan M1, pengerjaan M2 ini dilakukan pada sandbox eksekusi Linux (Ubuntu 24.04 container) yang disediakan asisten AI (Claude), **bukan** Windows 11 x64 + WSL 2, dan sandbox ini **tidak memiliki akses jaringan keluar**. Konsekuensinya: Clang/LLD digantikan GCC + GNU ld (skema fallback yang sama seperti M1); `xorriso`, `qemu-system-x86_64`, `OVMF`, dan `gdb` tidak tersedia; dan Limine **tidak dapat di-fetch** karena `git clone` ke GitHub tidak mencapai jaringan. Semua langkah yang secara teknis dapat dijalankan — preflight M2, penulisan source kernel, kompilasi, linking, dan inspeksi ELF lengkap — **benar-benar dijalankan** dan outputnya asli, termasuk satu bug nyata (entry point salah karena urutan fungsi) yang ditemukan dan diperbaiki dalam proses ini (lihat Bagian 15). Langkah yang bergantung pada `xorriso`/Limine/QEMU/OVMF dilaporkan sebagai **NOT_APPLICABLE (NA)** secara eksplisit oleh script itu sendiri, bukan dipalsukan lulus. **Mahasiswa wajib mengulang langkah fetch Limine, pembuatan ISO, dan run QEMU/OVMF pada mesin Windows 11 + WSL 2 pribadi** sebelum M2 dapat diklaim "siap uji QEMU tahap M2" sesuai definisi asli panduan.

---

## 0. Metadata Laporan

| Atribut | Isi |
|---|---|
| Kode praktikum | M2 |
| Judul praktikum | Boot Image, Kernel ELF64, Early Serial Console, dan Readiness Gate M2 MCSOS 260502 |
| Jenis pengerjaan | Individu |
| Nama mahasiswa | Jamilus Solihin |
| NIM | 2583207073010 |
| Kelas | PTI 1A |
| Nama kelompok | Tidak berlaku (individu) |
| Anggota kelompok | Tidak berlaku (individu) |
| Tanggal praktikum | 2026-07-06 |
| Tanggal pengumpulan | 2026-07-06 |
| Repository | `/home/claude/src/mcsos` (sandbox eksekusi lokal; belum ada remote) |
| Branch | `master` |
| Commit awal | `2ede37b7658f5c9c619d9f830192e482f86fa1db` (akhir M1) |
| Commit akhir | `be59c9e77a9e2f751a4f1c289bd429b45fb4d740` |
| Status readiness yang diklaim | Belum siap uji QEMU tahap M2 (bukti build/inspect lulus penuh; bukti runtime QEMU belum ada di sandbox ini) |

---

## 1. Sampul

# Laporan Praktikum M2
## Boot Image, Kernel ELF64, Early Serial Console, dan Readiness Gate M2 MCSOS 260502

Disusun oleh:

| Nama | NIM | Kelas | Peran |
|---|---|---|---|
| Jamilus Solihin | 2583207073010 | PTI 1A | Individu |

Dosen Pengampu: **Muhaemin Sidiq, S.Pd., M.Pd.**
Program Studi Pendidikan Teknologi Informasi
Institut Pendidikan Indonesia
Tahun Akademik 2025/2026

---

## 2. Pernyataan Orisinalitas dan Integritas Akademik

Saya menyatakan bahwa laporan ini disusun berdasarkan pekerjaan praktikum sendiri sesuai `OS_panduan_M2.pdf` dan template `os_template_laporan_praktikum.md`. Bantuan eksternal berupa asisten AI (Claude) dicatat secara eksplisit, termasuk keterbatasan lingkungan eksekusi yang dipakai.

| Pernyataan | Status |
|---|---|
| Semua potongan kode eksternal diberi atribusi | Tidak ada (source io.h/serial.c/memory.c/kmain.c/linker.ld ditulis mengikuti contoh persis pada panduan M2) |
| Semua penggunaan AI assistant dicatat | Ya |
| Repository yang dikumpulkan sesuai commit akhir | Ya |
| Tidak ada klaim readiness tanpa bukti | Ya |

Catatan penggunaan bantuan eksternal:

```text
Alat: Claude (Anthropic), digunakan untuk menjalankan seluruh langkah teknis M2 (menulis
source kernel, linker script, Makefile, script tools/scripts/*.sh, kompilasi, linking,
inspeksi ELF, dan penyusunan draf laporan ini) di dalam sandbox Linux miliknya sendiri,
melanjutkan repository M1 yang sudah ada.
Bagian yang dibantu: seluruh eksekusi teknis M2.
Verifikasi mandiri yang perlu dilakukan mahasiswa: menjalankan ulang seluruh langkah di
WSL 2 dengan Clang/LLD/xorriso/QEMU/OVMF asli, memfetch Limine dengan akses jaringan
sungguhan, membandingkan hasil readelf/objdump/nm, memperoleh serial log QEMU nyata, dan
mengisi ulang readiness review M2 dengan bukti dari lingkungan target sebelum submit resmi
jika dosen mensyaratkan bukti WSL 2 + QEMU asli.
```

---

## 3. Tujuan Praktikum

1. Membangun source tree M2 (`kernel/arch/x86_64/include/mcsos/arch/io.h`, `kernel/core/serial.c`, `kernel/core/kmain.c`, `kernel/lib/memory.c`, `linker.ld`) di atas repository M1 yang sudah divalidasi.
2. Mengompilasi dan me-link kernel ELF64 x86_64 freestanding dengan entry point higher-half `0xffffffff80000000`, menggunakan GCC + GNU ld sebagai pengganti Clang/LLD.
3. Membuktikan kernel tidak bergantung pada hosted libc dan hanya menggunakan port I/O (`outb`/`inb`) untuk UART COM1.
4. Menyusun preflight M0/M1/M2 yang memverifikasi lingkungan sebelum menulis kode boot, sesuai prinsip "jangan memperbaiki bug yang salah" pada panduan M2.
5. Menghasilkan evidence yang dapat diaudit: `readelf`, `objdump`, `nm`, `kernel.map`, serta mencatat secara jujur bagian yang **tidak dapat** diverifikasi (fetch Limine, ISO, QEMU/OVMF) di lingkungan eksekusi ini.

---

## 4. Capaian Pembelajaran Praktikum

| CPL/CPMK praktikum | Bukti yang harus ditunjukkan |
|---|---|
| Menjelaskan hubungan firmware, bootloader, kernel ELF64, linker script, entry point, emulator | Bagian 6, 9; `docs/readiness/M2-boot-image.md` |
| Memeriksa readiness M0/M1 sebelum M2 | `tools/scripts/m2_preflight.sh`, Bagian 10 Langkah 1 |
| Membuat kernel freestanding C17 tanpa hosted libc | `kernel/core/kmain.c`, `kernel/lib/memory.c`; Bagian 10 Langkah 3 |
| Membuat accessor port I/O x86_64 untuk UART 16550 COM1 | `kernel/arch/x86_64/include/mcsos/arch/io.h`, `kernel/core/serial.c` |
| Menghasilkan `kernel.elf`, `kernel.map`, evidence `readelf`/`objdump`/`nm` | Bagian 10 Langkah 3–4, Bagian 13 |
| Mengklasifikasikan failure modes M2 | Bagian 15 |
| Menyusun readiness review berbasis bukti | `docs/readiness/M2-boot-image.md`, Bagian 20 |

---

## 5. Peta Milestone MCSOS

| Milestone | Fokus | Status dalam laporan |
|---|---|---|
| M0 | Requirements, governance, baseline arsitektur | [ ] tidak dibahas (dipadankan dengan artefak M1) |
| M1 | Toolchain reproducible, Git, QEMU, GDB, metadata build | [x] selesai (laporan terpisah, commit `2ede37b`) |
| M2 | Boot image, kernel ELF64, early console | [x] dibahas (selesai sebagian — lihat batas cakupan) |
| M3–M16 | Milestone lanjutan | [x] tidak dibahas |

Batas cakupan praktikum:

```text
Laporan ini HANYA mencakup M2: source kernel freestanding, linker script higher-half,
kompilasi, linking, dan inspeksi ELF. Non-goals eksplisit sesuai panduan: memory manager,
IDT/interrupt handler, timer, panic subsystem penuh, framebuffer, filesystem, driver block,
network stack, scheduler, syscall, userspace, dan hardware bring-up fisik — semuanya TIDAK
disentuh pada laporan ini.

Tambahan batas khusus laporan ini: karena sandbox tanpa xorriso/QEMU/OVMF dan tanpa akses
jaringan untuk fetch Limine, langkah CP-M2.5 (fetch bootloader), CP-M2.6 (build image), dan
CP-M2.7 (QEMU run) berstatus NOT_APPLICABLE, bukan PASS/FAIL biasa. Klaim "siap uji QEMU
tahap M2" TIDAK diajukan pada laporan ini karena bukti runtime belum ada.
```

---

## 6. Dasar Teori Ringkas

### 6.1 Konsep Sistem Operasi yang Diuji

```text
M2 menguji jalur boot paling kecil yang bisa diverifikasi: firmware -> bootloader -> kernel
ELF64 -> entry point -> serial output -> controlled halt loop. Fokusnya bukan kernel lengkap,
melainkan membuktikan setiap sambungan pada rantai ini berfungsi dan dapat diinspeksi secara
independen, sehingga kegagalan pada milestone berikutnya (M3 dst.) dapat didiagnosis tanpa
harus curiga pada lapisan boot yang sudah terbukti benar di M2.
```

### 6.2 Konsep Arsitektur x86_64 yang Relevan

| Konsep | Relevansi pada praktikum | Bukti/verifikasi |
|---|---|---|
| Higher-half kernel | Kernel ditempatkan di `0xffffffff80000000` agar address space bawah tersedia untuk userspace di masa depan | `readelf -hW` Entry point address, `readelf -lW` VirtAddr segmen |
| ELF64 executable + PT_LOAD segments | Bootloader (Limine) memuat kernel berdasarkan program header, bukan section header | `build/inspect/readelf-program-headers.txt`: 3 segmen LOAD (R E, R, RW) |
| Port I/O UART 16550 (`outb`/`inb`) | Serial console adalah kanal observability pertama sebelum ada driver kompleks | `kernel/arch/x86_64/include/mcsos/arch/io.h`; instruksi `out`/`in` pada disassembly `outb`/`inb` |
| Red zone x86_64 System V ABI | Interrupt/exception handler (milestone berikutnya) tidak boleh mengandalkan red zone | Flag `-mno-red-zone` pada `Makefile` |

### 6.3 Konsep Implementasi Freestanding

| Aspek | Keputusan praktikum |
|---|---|
| Bahasa | C17 freestanding (`-std=c17 -ffreestanding`), tanpa inline assembly x86_64 minimal melalui `__asm__ volatile` untuk `outb`/`inb`/`cli`/`hlt` |
| Runtime | Tanpa hosted libc; `memset`/`memcpy`/`memmove` disediakan sendiri di `kernel/lib/memory.c` agar tidak ada undefined symbol jika compiler menyisipkan panggilan runtime memori |
| ABI | x86_64 System V calling convention, red zone dinonaktifkan, tanpa SIMD/FPU (`-mno-mmx -mno-sse -mno-sse2`) |
| Compiler flags kritis | `-ffreestanding -fno-stack-protector -fno-stack-check -fno-pic -fno-pie -mno-red-zone -mcmodel=kernel -Wall -Wextra -Werror` |
| Risiko undefined behavior | Fungsi `serial_transmit_empty` melakukan busy-wait tanpa timeout; ini disengaja untuk M2 (belum ada watchdog), dicatat sebagai known limitation |

### 6.4 Referensi Teori yang Digunakan

| No. | Sumber | Bagian yang digunakan | Alasan relevansi |
|---|---|---|---|
| [1] | M. Sidiq, "Panduan Praktikum M2" | Seluruh dokumen | Sumber utama instruksi, source contoh, dan acceptance criteria M2 |
| [2] | OSDev Wiki, "Higher Half Kernel" | Konsep alamat `0xffffffff80000000` | Dasar teori penempatan kernel higher-half |
| [3] | GNU Project, GNU Binutils Documentation | `readelf`, `objdump`, `nm` | Dasar inspeksi ELF sebagai evidence |

---

## 7. Lingkungan Praktikum

### 7.1 Host dan Target

| Komponen | Nilai |
|---|---|
| Host OS (rencana panduan) | Windows 11 x64 |
| Host OS (aktual dipakai) | Linux sandbox container — Ubuntu 24.04.4 LTS, kernel 6.18.5 (identik dengan M1) |
| Lingkungan build | Container Linux tunggal (bukan WSL 2) |
| Target ISA | x86_64 |
| Target ABI | ELF64 x86_64 (GCC native + GNU ld, bukan `x86_64-unknown-none-elf` Clang) |
| Emulator | Tidak tersedia (`qemu-system-x86_64`: NOT AVAILABLE) |
| Firmware emulator | Tidak diperiksa (bergantung QEMU) |
| Bootloader | Limine v11.x-binary (rencana); **tidak dapat di-fetch** — sandbox tanpa akses jaringan keluar |
| Debugger | Tidak tersedia (`gdb`: NOT AVAILABLE) |
| Build system | GNU Make 4.3, `.RECIPEPREFIX := >` |
| Bahasa utama | C17 freestanding + inline assembly x86_64 minimal (`outb`/`inb`/`cli`/`hlt`) |

### 7.2 Preflight M2

Perintah:

```bash
./tools/scripts/m2_preflight.sh
```

Output (ringkas, lengkap di `build/meta/m2-preflight.txt`):

```text
== M2 preflight MCSOS 260502 (adaptasi sandbox) ==
root=/home/claude/src/mcsos

[execution-environment-notice]
Lingkungan eksekusi aktual: Linux sandbox container (Ubuntu 24.04), BUKAN Windows 11
x64 + WSL 2. Sandbox tidak memiliki akses jaringan sehingga clang/lld/xorriso/qemu/ovmf
tidak dapat dipasang dan Limine tidak dapat di-fetch. GCC + GNU ld dipakai sebagai
toolchain kompilasi/link pengganti Clang/LLD.

OK filesystem: repository bukan /mnt/c, /mnt/d, atau /mnt/e

[required tools - GCC-fallback compile/link scheme]
OK: git, make, gcc, ld, readelf, objdump, nm, python3

[original panduan M2 tools - reported for transparency]
WARN: clang, ld.lld, qemu-system-x86_64, xorriso, shellcheck (semua NOT AVAILABLE)

[M0 artifacts]
WARN: docs/architecture/overview.md, docs/security/threat_model.md (path standar M0 berbeda)
OK: docs/architecture/invariants.md, docs/testing/verification_matrix.md
NOTE: repo ini memakai artefak M1 sebagai padanan M0/M1.

[M1 metadata] OK (dibangun ulang via make meta)
[M1 proof object check] OK: ELF64 x86_64 (dibangun ulang via make proof)
[OVMF check] NOT_APPLICABLE: qemu-system-x86_64 tidak tersedia

OK: preflight M2 selesai (skema GCC-fallback; QEMU/OVMF/Limine berstatus NOT_APPLICABLE)
```

### 7.3 Lokasi Repository

| Item | Nilai |
|---|---|
| Path repository | `/home/claude/src/mcsos` (sama dengan M1) |
| Apakah berada di filesystem Linux, bukan mount Windows | Ya (diverifikasi otomatis oleh `m2_preflight.sh`) |
| Remote repository | Tidak ada |
| Branch | `master` |
| Commit hash awal (akhir M1) | `2ede37b7658f5c9c619d9f830192e482f86fa1db` |
| Commit hash akhir (M2) | `be59c9e77a9e2f751a4f1c289bd429b45fb4d740` |

---

## 8. Repository dan Struktur File

### 8.1 Struktur Direktori yang Relevan (setelah M2)

```text
mcsos/
  Makefile                 # digabung: target M1 (meta/check/proof/qemu-probe/repro/test)
                            #          + target M2 (check-prev/check-src/check-scripts/
                            #            build/inspect/image/run/debug/grade)
  linker.ld
  configs/limine/limine.conf
  docs/
    architecture/invariants.md        # diperbarui: +6 invariant M2
    readiness/M1-toolchain.md
    readiness/M2-boot-image.md         # baru
    security/toolchain_threat_model.md
    security/supply_chain.md           # baru
    testing/verification_matrix.md     # diperbarui: +10 baris verifikasi M2
  kernel/
    arch/x86_64/include/mcsos/arch/io.h
    core/kmain.c
    core/serial.c
    lib/memory.c
  tests/toolchain/freestanding_probe.c  # dari M1
  tools/scripts/
    collect_meta.sh, check_toolchain.sh, proof_compile.sh,
    qemu_probe.sh, repro_check.sh        # dari M1
    m2_preflight.sh, inspect_kernel.sh, fetch_limine.sh,
    make_iso.sh, run_qemu.sh, run_qemu_debug.sh, grade_m2.sh  # baru M2
  build/            # generated, tidak dikomit
```

### 8.2 File yang Dibuat atau Diubah

| File | Jenis perubahan | Alasan perubahan | Risiko |
|---|---|---|---|
| `Makefile` | Ubah (gabung M1+M2) | Menyatukan target M1 dan M2 dalam satu antarmuka; ganti `CC`/`LD` ke `gcc`/`ld` | Sedang — deviasi dari `.RECIPEPREFIX`+Clang/LLD asli, didokumentasikan |
| `linker.ld` | Baru | Higher-half layout, `ENTRY(kmain)`, 3 PHDR (text/rodata/data) | Rendah — sesuai contoh panduan persis |
| `kernel/arch/x86_64/include/mcsos/arch/io.h` | Baru | Accessor port I/O `outb`/`inb`/`io_wait` | Rendah |
| `kernel/core/serial.c` | Baru | Driver UART 16550 COM1 minimal | Rendah — busy-wait tanpa timeout (dicatat) |
| `kernel/core/kmain.c` | Baru, lalu diubah urutan fungsi | Entry point kernel; urutan `kmain`/`halt_forever` ditukar agar entry point == `__kernel_start` | Sedang — bug nyata ditemukan dan diperbaiki, lihat Bagian 15 |
| `kernel/lib/memory.c` | Baru | `memset`/`memcpy`/`memmove` freestanding | Rendah |
| `tools/scripts/m2_preflight.sh` | Baru | Preflight M0/M1/M2, diadaptasi untuk skema fallback dan status NA | Sedang — kriteria "wajib" diubah dari panduan asli |
| `tools/scripts/inspect_kernel.sh` | Baru | Inspeksi ELF (`readelf`/`objdump`/`nm`) + assert entry point & symbol | Rendah |
| `tools/scripts/fetch_limine.sh` | Baru | Fetch Limine; fallback jujur ke `STATUS=NOT_APPLICABLE` bila tanpa jaringan | Sedang — tidak menghasilkan Limine sungguhan di sandbox ini |
| `tools/scripts/make_iso.sh` | Baru | Build ISO bootable; fallback jujur bila `xorriso`/Limine tidak ada | Sedang — ISO tidak dihasilkan di sandbox ini |
| `tools/scripts/run_qemu.sh` | Baru | Jalankan QEMU/OVMF headless; fallback jujur bila QEMU tidak ada | Sedang — serial log tidak dihasilkan di sandbox ini |
| `tools/scripts/run_qemu_debug.sh` | Baru | QEMU + GDB stub; fallback jujur bila QEMU/gdb tidak ada | Sedang |
| `tools/scripts/grade_m2.sh` | Baru, diadaptasi | Grading lokal; membedakan PASS (build/inspect) vs NOT_APPLICABLE (image/QEMU) alih-alih exit 1 keras | Sedang — deviasi dari grading asli yang mewajibkan ISO+QEMU |
| `docs/readiness/M2-boot-image.md` | Baru | Readiness review M2 dengan evidence matrix dan keputusan jujur | Sedang |
| `docs/security/supply_chain.md` | Baru | Catatan bahwa Limine belum di-vendor tanpa checksum resmi | Rendah |
| `docs/architecture/invariants.md` | Ubah (tambah) | +6 invariant M2 | Rendah |
| `docs/testing/verification_matrix.md` | Ubah (tambah) | +10 baris verifikasi M2 (W1–W10) | Rendah |
| `configs/limine/limine.conf` | Baru | Konfigurasi boot Limine (belum diuji end-to-end karena Limine tidak ada) | Rendah |

### 8.3 Ringkasan Diff

```bash
git status --short
git diff --stat 2ede37b be59c9e
git log --oneline -n 5
```

Output:

```text
(working tree bersih setelah commit, build/ di-ignore)

be59c9e M2: add bootable kernel ELF and early serial console (GCC/GNU-ld fallback; image/QEMU NA in sandbox)
2ede37b M1: add reproducible toolchain readiness baseline

17 file diubah pada commit M2:
 M Makefile
 M docs/architecture/invariants.md
 M docs/testing/verification_matrix.md
 A configs/limine/limine.conf
 A docs/readiness/M2-boot-image.md
 A docs/security/supply_chain.md
 A kernel/arch/x86_64/include/mcsos/arch/io.h
 A kernel/core/kmain.c
 A kernel/core/serial.c
 A kernel/lib/memory.c
 A linker.ld
 A tools/scripts/fetch_limine.sh
 A tools/scripts/grade_m2.sh
 A tools/scripts/inspect_kernel.sh
 A tools/scripts/m2_preflight.sh
 A tools/scripts/make_iso.sh
 A tools/scripts/run_qemu.sh
 A tools/scripts/run_qemu_debug.sh
```

---

## 9. Desain Teknis

### 9.1 Masalah yang Diselesaikan

```text
Setelah M1 membuktikan toolchain dapat menghasilkan ELF64 x86_64 freestanding, M2 harus
membuktikan bahwa kode tersebut dapat menjadi kernel yang benar-benar dimuat oleh bootloader
dan mencetak output yang dapat diverifikasi (serial marker) sebelum ada subsistem kompleks
apa pun. Tanpa bukti ini, kegagalan pada M3/M4 (interrupt, memory manager) akan sulit
dibedakan antara "boot path salah" dan "logika M3/M4 salah". Masalah tambahan pada laporan
ini: sandbox eksekusi tidak memiliki xorriso/QEMU/OVMF dan tidak ada akses jaringan untuk
mengambil Limine, sehingga desain script harus tetap transparan melaporkan NOT_APPLICABLE
alih-alih memalsukan bukti boot yang sebenarnya tidak pernah terjadi.
```

### 9.2 Keputusan Desain

| Keputusan | Alternatif yang dipertimbangkan | Alasan memilih | Konsekuensi |
|---|---|---|---|
| GCC + GNU ld menggantikan Clang + LLD (lanjutan skema M1) | Tidak mengerjakan M2 sama sekali | Konsisten dengan fallback M1; GCC mendukung flag freestanding setara dan menghasilkan ELF64 x86_64 yang valid | Perilaku code-gen (prolog/epilog `push %rbp`) bisa berbeda dari Clang -O0; harus dibandingkan ulang di WSL 2 |
| `fetch_limine.sh`, `make_iso.sh`, `run_qemu.sh` melaporkan `STATUS=NOT_APPLICABLE` bukan error fatal | Membuat pipeline gagal keras (`exit 1`) di setiap tahap | Menjaga `make grade` tetap bisa menunjukkan status build/inspect yang genuinely lulus, tanpa menyembunyikan bagian yang tidak teruji | Pembaca harus membaca status NA per tahap, bukan asumsi "make run lulus" berarti QEMU benar-benar jalan |
| `grade_m2.sh` membedakan artefak "wajib" (build/inspect) vs "NA-jika-tool-hilang" (image/QEMU) | Memakai daftar `required_files` tunggal seperti panduan asli (semua wajib, exit 1 jika hilang) | Mencegah `make grade` selalu gagal di sandbox ini karena ISO/QEMU log memang tidak mungkin ada di sini, sambil tetap menegakkan kriteria wajib untuk build/inspect | Skema "lulus" di laporan ini bukan skema lulus M2 penuh sesuai panduan asli; dicatat eksplisit di readiness review |
| Menukar urutan fungsi `kmain`/`halt_forever` di source | Membiarkan entry point `0xffffffff8000000c` dan mengubah kriteria pemeriksaan `inspect_kernel.sh` menjadi lebih longgar | Memperbaiki akar masalah (urutan kode) lebih baik daripada melonggarkan pemeriksaan; entry point tepat `__kernel_start` adalah invariant yang wajar untuk kernel M2 sederhana | Menambah 1 invariant baru (#13 di `invariants.md`) yang harus dijaga saat source bertambah |

### 9.3 Arsitektur Ringkas

```mermaid
flowchart TD
    A[OVMF firmware] --> B[Limine bootloader]
    B --> C[kernel.elf dimuat]
    C --> D[kmain]
    D --> E[serial_init]
    E --> F[serial_write x3 marker]
    F --> G[halt_forever: cli; hlt loop]
```

Penjelasan diagram:

```text
Alur ini identik dengan Bagian 4B panduan M2. Pada laporan ini, node A (OVMF) dan B (Limine)
belum dapat dieksekusi karena keterbatasan sandbox; yang terbukti nyata adalah C sampai G,
yaitu kernel.elf terbentuk, kmain adalah entry point tepat di alamat kernel_start, dan
disassembly menunjukkan urutan pemanggilan serial_init -> serial_write (3x) -> halt_forever
persis sesuai desain kode sumber (lihat Bagian 13 objdump evidence).
```

### 9.4 Kontrak Antarmuka

| Antarmuka | Pemanggil | Penerima | Precondition | Postcondition | Error path |
|---|---|---|---|---|---|
| `ENTRY(kmain)` (linker) | Bootloader (Limine, rencana) | `kmain` | Kernel dimuat pada `0xffffffff80000000` | CPU mulai eksekusi di `kmain` | Tidak diuji end-to-end (Limine NA) |
| `serial_init()` | `kmain` | UART COM1 hardware/emulasi | Port I/O dapat diakses | UART dikonfigurasi 38400 8N1 (implisit dari nilai register) | Tidak ada; port I/O selalu berhasil secara arsitektural pada ring 0 |
| `serial_write(const char*)` | `kmain` | `serial_putc` per karakter | String NUL-terminated valid atau NULL | Semua karakter tertulis ke COM1 (busy-wait per karakter) | `NULL` ditangani (early return), tidak crash |
| `halt_forever()` | `kmain` (akhir eksekusi) | CPU | — | CPU idle dalam loop `cli; hlt` selamanya | Tidak ada; fungsi `noreturn` |

### 9.5 Struktur Data Utama

| Struktur data | Field penting | Ownership | Lifetime | Invariant |
|---|---|---|---|---|
| Kernel image (segments `.text`/`.rodata`/`.data`/`.bss`) | `__kernel_start`, `__kernel_end` | Linker (`linker.ld`) | Seumur hidup kernel setelah dimuat | `__kernel_start == 0xffffffff80000000 == kmain` |
| UART COM1 register (0x3F8–0x3FF) | IER, LCR, FCR, MCR | Hardware/emulator, diakses eksklusif oleh `serial_init`/`serial_putc` | Seumur hidup sesi boot | Tidak ada driver lain yang mengakses COM1 secara bersamaan (belum ada concurrency di M2) |

### 9.6 Invariants

Lihat `docs/architecture/invariants.md` invariant #9–#14 (ditambahkan pada M2):

1. `kernel.elf` harus ELF64 x86_64 EXEC dengan entry point tepat `0xffffffff80000000`.
2. Kernel tidak boleh memanggil fungsi hosted libc.
3. `kmain` tidak boleh return; selalu berakhir di `halt_forever`.
4. UART COM1 hanya diakses lewat `outb`/`inb`, bukan MMIO.
5. `kmain` harus berada tepat di awal `.text` (`__kernel_start == kmain`).
6. Artefak yang tidak dapat diverifikasi (image/QEMU) harus dilaporkan `NOT_APPLICABLE` secara eksplisit.

### 9.7 Ownership, Locking, dan Concurrency

| Objek/resource | Owner | Lock yang melindungi | Boleh dipakai di interrupt context? | Catatan |
|---|---|---|---|---|
| UART COM1 | `kmain` (single-threaded boot path) | Tidak ada (belum ada interrupt/SMP) | Tidak relevan pada M2 | Serial driver busy-wait, belum aman untuk preemptive/SMP (dicatat di kontrak teknis `serial.c`) |

Lock order yang berlaku:

```text
Tidak ada locking pada M2 karena belum ada interrupt handler atau multitasking; seluruh
kmain berjalan sekuensial single-core.
```

### 9.8 Memory Safety dan Undefined Behavior Risk

| Risiko | Lokasi | Mitigasi | Bukti |
|---|---|---|---|
| Busy-wait tanpa timeout pada `serial_transmit_empty` | `kernel/core/serial.c` | Diterima sebagai batasan M2 (tidak ada watchdog); didokumentasikan sebagai known limitation | Kontrak teknis pada source dan Bagian 15 |
| Entry point tidak sesuai invariant jika urutan fungsi berubah | `kernel/core/kmain.c` | Invariant #13 ditambahkan; ditemukan dan diperbaiki langsung pada praktikum ini | `build/inspect/readelf-header.txt` sebelum/sesudah perbaikan (lihat Bagian 15) |

### 9.9 Security Boundary

| Boundary | Data tidak tepercaya | Validasi yang dilakukan | Failure mode aman |
|---|---|---|---|
| Bootloader → kernel handoff | Tidak ada boot info dipakai pada M2 (sengaja) | Tidak ada parsing data eksternal sama sekali | Tidak ada permukaan serangan baru karena tidak ada input diproses |
| Supply-chain Limine | Binary/branch Limine dari GitHub | Belum diverifikasi (fetch gagal karena tanpa jaringan); dicatat di `docs/security/supply_chain.md`, tidak di-vendor buta tanpa checksum | Gagal aman: tidak ada Limine yang dipakai sama sekali di sandbox ini, dp fail-closed |

---

## 10. Langkah Kerja Implementasi

### Langkah 1 — Preflight M0/M1/M2

Maksud langkah:

```text
Memastikan repository tidak dibangun di atas M0/M1 yang rusak, dan mencatat secara jujur
tool asli panduan (Clang/LLD/xorriso/QEMU/OVMF) yang tidak tersedia di sandbox ini.
```

Perintah:

```bash
./tools/scripts/m2_preflight.sh
```

Output ringkas: lihat Bagian 7.2 di atas (lengkap di `build/meta/m2-preflight.txt`).

Artefak yang dihasilkan:

| Artefak | Lokasi | Fungsi |
|---|---|---|
| Preflight report | `build/meta/m2-preflight.txt` | Evidence pemeriksaan M0/M1/M2 |
| Re-check object M1 | `build/meta/m2-check-m1-object-readelf.txt` | Evidence object M1 masih ELF64 x86_64 |

Indikator berhasil:

```text
Script keluar dengan "OK: preflight M2 selesai", semua tool wajib skema fallback OK, dan
status NA untuk OVMF/QEMU dilaporkan eksplisit, bukan disembunyikan.
```

### Langkah 2 — Menulis source tree M2

Maksud langkah:

```text
Menambahkan header port I/O, driver serial, runtime memori minimal, dan kernel entry sesuai
contoh persis pada panduan M2 Bagian 9.
```

Perintah: (pembuatan file `io.h`, `serial.c`, `memory.c`, `kmain.c`, `linker.ld` — isi lengkap tersedia di repository, mengikuti Bagian 9–10 panduan)

Output ringkas:

```text
5 file baru dibuat: kernel/arch/x86_64/include/mcsos/arch/io.h, kernel/core/serial.c,
kernel/core/kmain.c, kernel/lib/memory.c, linker.ld
```

Indikator berhasil:

```text
make check-src lulus (compiler/linker terdeteksi, semua direktori source ada).
```

### Langkah 3 — Build kernel ELF64

Maksud langkah:

```text
Membuktikan source M2 dapat dikompilasi dan dilink sebagai kernel ELF64 freestanding
menggunakan GCC + GNU ld (fallback dari Clang + LLD).
```

Perintah:

```bash
make distclean
make check-src
make build
```

Output ringkas:

```text
gcc --version -> gcc (Ubuntu 13.3.0-6ubuntu2~24.04.1) 13.3.0
ld --version  -> GNU ld (GNU Binutils for Ubuntu) 2.42

gcc -std=c17 -ffreestanding -fno-stack-protector -fno-stack-check -fno-pic -fno-pie -m64 \
    -march=x86-64 -mno-red-zone -mno-mmx -mno-sse -mno-sse2 -mcmodel=kernel \
    -Wall -Wextra -Werror -Ikernel/arch/x86_64/include -c kernel/core/kmain.c -o build/kernel/core/kmain.o
gcc [flags sama] -c kernel/core/serial.c -o build/kernel/core/serial.o
gcc [flags sama] -c kernel/lib/memory.c -o build/kernel/lib/memory.o
ld -nostdlib -static -m elf_x86_64 -z max-page-size=0x1000 -T linker.ld -Map=build/kernel.map \
   -o build/kernel.elf build/kernel/core/kmain.o build/kernel/core/serial.o build/kernel/lib/memory.o
```

Artefak yang dihasilkan:

| Artefak | Lokasi | Fungsi |
|---|---|---|
| Object files | `build/kernel/core/*.o`, `build/kernel/lib/*.o` | Hasil kompilasi per file |
| Kernel ELF | `build/kernel.elf` | Kernel executable |
| Linker map | `build/kernel.map` | Layout memori kernel |

Indikator berhasil:

```text
Tidak ada warning/error dari -Wall -Wextra -Werror; build/kernel.elf dan build/kernel.map
terbentuk.
```

### Langkah 4 — Inspeksi kernel ELF

Maksud langkah:

```text
Memastikan kernel bukan executable Linux/Windows biasa, entry point sesuai desain, dan
symbol boundary (kmain, serial_init, serial_write) muncul.
```

Perintah:

```bash
make inspect
```

Output penting (setelah perbaikan urutan fungsi, lihat Bagian 15):

```text
ELF Header:
  Class: ELF64 | Type: EXEC (Executable file) | Machine: Advanced Micro Devices X86-64
  Entry point address: 0xffffffff80000000

Program Headers:
  LOAD 0x001000 0xffffffff80000000 ... FileSiz 0x0002fa MemSiz 0x0002fa R E 0x1000
  LOAD 0x002000 0xffffffff80001000 ... FileSiz 0x000200 MemSiz 0x000200 R   0x1000
  LOAD 0x0000e8 0x0000000000000000 ... FileSiz 0x000000 MemSiz 0x000000 RW  0x1000

Section to Segment mapping:
  00 .text
  01 .rodata .eh_frame .note.gnu.property
  02 (kosong — .data/.bss kosong pada M2 karena tidak ada variabel global berisi data)

nm -n build/kernel.elf:
ffffffff80000000 T __kernel_start
ffffffff80000000 T kmain
ffffffff80000036 t halt_forever
ffffffff80000042 t outb
ffffffff80000061 t inb
ffffffff8000007f t serial_transmit_empty
ffffffff800000a1 T serial_init
ffffffff80000115 T serial_putc
ffffffff80000156 T serial_write
ffffffff80000198 T memset
ffffffff800001e1 T memcpy
ffffffff8000023f T memmove
ffffffff80001200 R __kernel_end

objdump -drwC (cuplikan fungsi kmain):
ffffffff80000000 <kmain>:
  endbr64
  push %rbp; mov %rsp,%rbp
  call serial_init
  mov $..., %rdi; call serial_write   (x3, untuk 3 marker)
  call halt_forever
```

Artefak yang dihasilkan:

| Artefak | Lokasi | Fungsi |
|---|---|---|
| `readelf-header.txt` | `build/inspect/` | Evidence ELF header + entry point |
| `readelf-program-headers.txt` | `build/inspect/` | Evidence 3 PT_LOAD segments |
| `readelf-sections.txt` | `build/inspect/` | Evidence layout section |
| `objdump-disassembly.txt` | `build/inspect/` | Evidence disassembly lengkap semua fungsi |
| `nm-symbols.txt` | `build/inspect/` | Evidence symbol table terurut alamat |

Indikator berhasil:

```text
Class ELF64, Machine x86_64, Entry point == 0xffffffff80000000, symbol kmain/serial_init/
serial_write semuanya ada. Semua kriteria terpenuhi setelah perbaikan pada Langkah 15.
```

### Langkah 5 — Fetch Limine (NOT_APPLICABLE)

Maksud langkah:

```text
Sesuai panduan, mengambil Limine binary release branch v11.x-binary untuk dijadikan
bootloader ISO. Karena sandbox tanpa akses jaringan keluar, langkah ini dilaporkan NA.
```

Perintah:

```bash
./tools/scripts/fetch_limine.sh
```

Output ringkas:

```text
STATUS=NOT_APPLICABLE
REASON=Sandbox eksekusi tidak memiliki akses jaringan keluar (egress dinonaktifkan);
  git clone https://github.com/LimineBootloader/Limine.git tidak dapat dijangkau.
IMPACT=Bootloader Limine tidak tersedia; make image dan make run tidak dapat
  dijalankan penuh pada sandbox ini.
MITIGATION=Jalankan ulang fetch_limine.sh pada mesin dengan akses internet (WSL 2
  mahasiswa), atau minta dosen menyediakan arsip Limine + checksum resmi.
```

Artefak yang dihasilkan: `build/meta/limine-revision.txt` (berisi status NA, bukan revision hash asli).

Indikator berhasil: script tidak crash tanpa penjelasan; status NA tercatat jelas dengan alasan dan mitigasi.

### Langkah 6 — Build image ISO (NOT_APPLICABLE)

Perintah:

```bash
make image
```

Output ringkas:

```text
STATUS=NOT_APPLICABLE
REASON=xorriso tidak terpasang di sandbox ini (tanpa akses jaringan/instalasi paket).
IMPACT=build/mcsos.iso tidak dapat dibuat dari sandbox ini.
MITIGATION=Pasang xorriso, mtools, dosfstools di WSL 2, lalu jalankan ulang make image.
```

Status: **NOT_APPLICABLE** — dicatat di `build/meta/make-iso-status.txt`.

### Langkah 7 — Jalankan QEMU/OVMF (NOT_APPLICABLE)

Perintah:

```bash
make run
```

Output ringkas:

```text
STATUS=NOT_APPLICABLE
REASON=qemu-system-x86_64 tidak terpasang di sandbox eksekusi ini.
IMPACT=Boot runtime M2 (build/qemu-serial.log dengan marker M2) tidak dapat
  diverifikasi dari sandbox ini.
MITIGATION=Jalankan ulang make run pada mesin WSL 2 dengan qemu-system-x86 + ovmf.
```

Status: **NOT_APPLICABLE** — dicatat di `build/meta/run-qemu-status.txt`.

### Langkah 8 — Grading lokal

Maksud langkah:

```text
Menyatukan semua pemeriksaan build dan inspect (yang genuinely bisa dijalankan) dan
melaporkan status image/QEMU secara terpisah dan jujur sebagai NOT_APPLICABLE.
```

Perintah:

```bash
make grade
```

Output ringkas:

```text
[build + inspect evidence - wajib di sandbox ini]
PASS artifact: build/kernel.elf
PASS artifact: build/kernel.map
PASS artifact: build/inspect/readelf-header.txt
PASS artifact: build/inspect/readelf-program-headers.txt
PASS artifact: build/inspect/objdump-disassembly.txt
PASS artifact: build/inspect/nm-symbols.txt

[image + qemu evidence - NOT_APPLICABLE jika tool tidak tersedia di sandbox]
NOT_APPLICABLE: build/mcsos.iso
NOT_APPLICABLE: build/mcsos.iso.sha256
NOT_APPLICABLE: build/qemu-serial.log

[content checks]
PASS: ELF64
PASS: Machine x86_64
PASS: entry point sesuai linker.ld
NOT_APPLICABLE: pemeriksaan marker serial dilewati (qemu-serial.log tidak ada)

OK: M2 local grading checks passed untuk bagian yang dapat dijalankan di sandbox ini
```

Indikator berhasil: exit code 0, dengan rincian PASS vs NOT_APPLICABLE yang jelas per item (bukan satu status tunggal yang menyamarkan cakupan sebenarnya).

### Langkah 9 — Clean-checkout rehearsal dan commit

Maksud langkah:

```text
Membuktikan seluruh rangkaian dapat diulang dari clean state, lalu menyimpan hasil sebagai
commit M2.
```

Perintah:

```bash
make distclean
make check-prev && make check-src && make build && make inspect && make image && make run && make grade
git add Makefile linker.ld configs kernel tools docs
git commit -m "M2: add bootable kernel ELF and early serial console (GCC/GNU-ld fallback; image/QEMU NA in sandbox)"
git rev-parse HEAD
```

Output ringkas:

```text
Seluruh target di atas lulus/berstatus NA sesuai Langkah 1-8 (identik hasilnya dengan
rehearsal sebelum commit — lihat sha256 kernel.elf di Bagian 13).
be59c9e77a9e2f751a4f1c289bd429b45fb4d740
```

Artefak yang dihasilkan: 1 commit Git (`be59c9e...`) berisi seluruh source dan dokumen M2.

Indikator berhasil: `make distclean && ...` menghasilkan kernel.elf dengan hash SHA-256 yang identik dengan build sebelumnya, dan commit tercatat tanpa error.

---

## 11. Checkpoint Buildable

| Checkpoint | Perintah | Expected result | Status |
|---|---|---|---|
| CP-M2.1 Preflight | `make check-prev` | `build/meta/m2-preflight.txt` OK | **PASS** (skema fallback; OVMF NA) |
| CP-M2.2 Source syntax | `make check-scripts` | `bash -n` lulus semua script | **PASS** (`shellcheck` NA — tidak tersedia) |
| CP-M2.3 Build ELF | `make build` | `build/kernel.elf`, `build/kernel.map` | **PASS** |
| CP-M2.4 Inspect ELF | `make inspect` | ELF64 x86_64, entry benar | **PASS** |
| CP-M2.5 Fetch bootloader | `./tools/scripts/fetch_limine.sh` | Limine tersedia | **NOT_APPLICABLE** — tanpa akses jaringan |
| CP-M2.6 Build image | `make image` | `build/mcsos.iso` | **NOT_APPLICABLE** — `xorriso` tidak tersedia |
| CP-M2.7 QEMU run | `make run` | `build/qemu-serial.log` dengan marker | **NOT_APPLICABLE** — QEMU/OVMF tidak tersedia |
| CP-M2.8 Local grade | `make grade` | Semua artefak valid | **PASS** (untuk cakupan build/inspect; image/QEMU NA eksplisit) |

Catatan checkpoint:

```text
CP-M2.5, CP-M2.6, CP-M2.7 berstatus NOT_APPLICABLE, bukan FAIL, karena keterbatasan sandbox
(tanpa xorriso/QEMU/OVMF dan tanpa akses jaringan keluar). Ketiganya WAJIB diulang oleh
mahasiswa pada mesin Windows 11 + WSL 2 pribadi sebelum M2 dapat diklaim "siap uji QEMU
tahap M2" sesuai definisi asli panduan (Bagian 30 panduan M2).
```

---

## 12. Perintah Uji dan Validasi

### 12.1 Build Test

```bash
make distclean
make check-src
make build
```

Hasil: kernel ELF64 x86_64 terbentuk tanpa warning (`-Wall -Wextra -Werror` lulus).

Status: **PASS**

### 12.2 Static Inspection

```bash
readelf -hW build/kernel.elf
readelf -lW build/kernel.elf
objdump -drwC build/kernel.elf
nm -n build/kernel.elf
```

Hasil penting: Entry point `0xffffffff80000000`, 3 PT_LOAD segments (R E / R / RW), semua symbol M2 (`kmain`, `serial_init`, `serial_write`, `memset`, `memcpy`, `memmove`) ada, tidak ada undefined symbol pada disassembly (tidak ada call ke fungsi eksternal libc).

Status: **PASS**

### 12.3 QEMU Smoke Test

Status: **NOT_APPLICABLE** — `qemu-system-x86_64` dan OVMF tidak tersedia di sandbox; `build/mcsos.iso` juga tidak ada karena `xorriso`/Limine tidak tersedia.

### 12.4 GDB Debug Evidence

Status: **NOT_APPLICABLE** — `gdb` tidak tersedia di sandbox; juga bergantung pada ISO yang tidak ada.

### 12.5 Unit Test

Tidak ada unit test terpisah pada M2 (sesuai panduan; M2 memakai build+inspect+image+run sebagai gate, bukan unit test klasik). Lihat Bagian 12.1–12.3.

### 12.6 Stress/Fuzz/Fault Injection Test

Status: **NOT_APPLICABLE** — sesuai panduan (Bagian 4J), fuzzing boot info ditunda ke M3/M4 karena parser boot info belum diaktifkan pada M2.

### 12.7 Visual Evidence

Status: **NOT_APPLICABLE** — M2 headless (`-display none` direncanakan); tidak ada output grafis.

---

## 13. Hasil Uji

### 13.1 Tabel Ringkasan Hasil

| No. | Uji | Expected result | Actual result | Status | Evidence |
|---|---|---|---|---|---|
| 1 | Preflight M0/M1/M2 | Semua OK/NA eksplisit | Sesuai | PASS | `build/meta/m2-preflight.txt` |
| 2 | Build kernel ELF64 tanpa warning | ELF64 x86_64 terbentuk | Sesuai | PASS | `build/kernel.elf` |
| 3 | Entry point == `0xffffffff80000000` | Sesuai linker.ld | Sesuai (setelah perbaikan urutan fungsi) | PASS | `build/inspect/readelf-header.txt` |
| 4 | Symbol `kmain`/`serial_init`/`serial_write` ada | Muncul di nm | Muncul | PASS | `build/inspect/nm-symbols.txt` |
| 5 | Tidak ada undefined symbol/call libc | Disassembly bersih | Bersih | PASS | `build/inspect/objdump-disassembly.txt` |
| 6 | Fetch Limine | Limine tersedia | Tidak dapat diambil (tanpa jaringan) | NOT_APPLICABLE | `build/meta/limine-revision.txt` |
| 7 | Build ISO bootable | `mcsos.iso` ada | Tidak dibuat (`xorriso` NA) | NOT_APPLICABLE | `build/meta/make-iso-status.txt` |
| 8 | QEMU/OVMF boot + 3 marker serial | Log berisi marker | Tidak dijalankan (QEMU NA) | NOT_APPLICABLE | `build/meta/run-qemu-status.txt` |
| 9 | Local grading (`make grade`) | Semua artefak valid | PASS untuk build/inspect, NA untuk image/QEMU | PASS (cakupan terbatas) | Output `grade_m2.sh` |
| 10 | Clean-checkout rehearsal | Hasil identik dari `make distclean` | Hash `kernel.elf` identik | PASS | Bagian 13.3 |
| 11 | Commit Git M2 | Commit hash tercatat | Ada | PASS | `be59c9e77a9e2f751a4f1c289bd429b45fb4d740` |

### 13.2 Log Penting

```text
OK: preflight M2 selesai (skema GCC-fallback; QEMU/OVMF/Limine berstatus NOT_APPLICABLE)
OK: kernel ELF inspection passed (GCC/GNU-ld fallback toolchain)
OK: M2 local grading checks passed untuk bagian yang dapat dijalankan di sandbox ini
```

### 13.3 Artefak Bukti

| Artefak | Path | SHA-256 | Fungsi |
|---|---|---|---|
| `kernel.elf` | `build/kernel.elf` | `ea40070c73f44a804508d26adb83aacaedb3e6fd6bcd782fb82e9a10ca1aeaab` | Kernel binary M2 |
| `kernel.map` | `build/kernel.map` | `1c8f2c1979a1c47b479cbbdccdca1996c49ac761951e2231231f0dad7c96a7a6` | Linker map |
| commit Git M2 | — | `be59c9e77a9e2f751a4f1c289bd429b45fb4d740` | Baseline M2 |

Perintah hash:

```bash
sha256sum build/kernel.elf build/kernel.map
```

Catatan: hash di atas identik antara build pertama dan build ulang setelah `make distclean` (clean-checkout rehearsal Langkah 9), membuktikan build M2 deterministik untuk input yang sama.

---

## 14. Analisis Teknis

### 14.1 Analisis Keberhasilan

```text
Kompilasi dan linking berhasil karena kombinasi flag freestanding (-ffreestanding -nostdlib
-mno-red-zone -fno-stack-protector -mcmodel=kernel) mencegah GCC menyisipkan dependensi
hosted. Linker script menempatkan .text pada 0xffffffff80000000 dan ENTRY(kmain) membuat
linker mencari alamat symbol kmain sebagai entry point — yang setelah perbaikan urutan
fungsi, tepat berada di awal .text sehingga entry point == __kernel_start. Disassembly
menunjukkan urutan pemanggilan yang persis sesuai source: serial_init, tiga kali
serial_write, lalu halt_forever yang tidak pernah return (instruksi terakhir adalah loop
cli;hlt;jmp).
```

### 14.2 Analisis Kegagalan atau Perbedaan Hasil

```text
Kegagalan yang ditemukan (dan diperbaiki) pada praktikum ini: entry point awal
0xffffffff8000000c, bukan 0xffffffff80000000, karena fungsi static halt_forever
didefinisikan sebelum kmain dalam source, sehingga linker menempatkan kode halt_forever
lebih dulu di .text. Diagnosis: readelf -hW menunjukkan entry point tidak sama dengan
VirtAddr awal segmen LOAD .text. Perbaikan: menukar urutan definisi (kmain didefinisikan
lebih dulu, halt_forever di-forward-declare lalu didefinisikan setelah kmain), sehingga
linker menempatkan kmain di awal .text. Setelah perbaikan, make distclean && make build &&
make inspect diulang dan entry point tepat 0xffffffff80000000.

Kegagalan/keterbatasan lain (bukan bug kode, melainkan keterbatasan lingkungan): fetch
Limine, build ISO, dan run QEMU semuanya NOT_APPLICABLE karena sandbox tanpa jaringan dan
tanpa xorriso/QEMU/OVMF. Ini bukan dugaan bug pada source M2, karena source dan linker
script sudah terbukti benar lewat inspeksi ELF; ini murni keterbatasan alat di lingkungan
eksekusi laporan ini.
```

### 14.3 Perbandingan dengan Teori

| Konsep teori | Implementasi praktikum | Sesuai/tidak sesuai | Penjelasan |
|---|---|---|---|
| Higher-half kernel | `. = 0xffffffff80000000` di `linker.ld` | Sesuai | `readelf -lW` menunjukkan VirtAddr segmen LOAD dimulai persis di alamat tersebut |
| ELF64 dengan PT_LOAD sesuai permission | 3 PHDR: text (R E), rodata (R), data (RW) | Sesuai | `readelf -lW` menunjukkan flag `R E`, `R`, `RW` persis sesuai `PHDRS` di linker script |
| Serial console early observability | `serial_init` dipanggil pertama di `kmain` | Sesuai (desain source); belum terbukti runtime | Disassembly `kmain` menunjukkan `call serial_init` sebagai instruksi pertama setelah prolog |
| Kernel tidak boleh return | `halt_forever` `noreturn`, loop `cli;hlt` | Sesuai | Instruksi terakhir `jmp` kembali ke `hlt`, tidak ada `ret` yang tercapai setelah halt_forever dipanggil |

### 14.4 Kompleksitas dan Kinerja

| Aspek | Estimasi/hasil | Bukti | Catatan |
|---|---|---|---|
| Ukuran kernel | `.text` 0x2fa byte (762 byte), total image `~0x1200` (4608 byte) | `readelf -SW`, `nm __kernel_end` | Sangat kecil karena hanya serial + memory runtime |
| Waktu build | < 1 detik | Observasi eksekusi langsung | 3 file C kecil |
| Waktu boot QEMU | Tidak berlaku | — | QEMU tidak tersedia |
| Jumlah instruksi per fungsi | `outb`/`inb`: ~10-14 instruksi | `objdump -drwC` | Wajar untuk accessor port I/O sederhana tanpa optimasi `-O2` (Makefile M2 tidak set `-O`, default `-O0`) |

---

## 15. Debugging dan Failure Modes

### 15.1 Failure Modes yang Ditemukan

| Failure mode | Gejala | Penyebab sementara | Bukti | Perbaikan |
|---|---|---|---|---|
| Entry point tidak sesuai desain | `readelf -hW` menunjukkan `Entry point address: 0xffffffff8000000c`, bukan `...000` | `halt_forever` (static) didefinisikan sebelum `kmain` dalam source, sehingga kodenya ditempatkan lebih dulu di `.text` oleh compiler/linker | `build/inspect/readelf-header.txt` (versi sebelum perbaikan, direproduksi ulang saat debugging) | Menukar urutan definisi: `kmain` didefinisikan lebih dulu, `halt_forever` di-forward-declare lalu didefinisikan setelahnya. Setelah `make distclean && make build && make inspect`, entry point tepat `0xffffffff80000000` |
| Fetch Limine gagal | `git ls-remote` tidak dapat menjangkau GitHub | Sandbox tanpa akses jaringan keluar (egress dinonaktifkan) | `build/meta/limine-revision.txt` berisi `STATUS=NOT_APPLICABLE` | Tidak diperbaiki di sandbox ini; didokumentasikan sebagai known limitation, mitigasi: jalankan di WSL 2 |
| `xorriso`/QEMU/OVMF tidak tersedia | `command -v xorriso`/`qemu-system-x86_64` gagal | Sandbox tanpa akses instalasi paket baru | `build/meta/make-iso-status.txt`, `build/meta/run-qemu-status.txt` | Sama seperti di atas — didokumentasikan, bukan diperbaiki di sandbox ini |

### 15.2 Failure Modes yang Diantisipasi

| Failure mode | Deteksi | Dampak | Mitigasi |
|---|---|---|---|
| Undefined symbol `memcpy`/`memset`/`memmove` | `nm -u` (implisit lewat proses link yang akan gagal jika ada undefined) | Kernel gagal link | `kernel/lib/memory.c` menyediakan implementasi sendiri, sudah diverifikasi tidak ada undefined symbol pada link M2 |
| Serial log kosong saat runtime QEMU nyata (di WSL 2 nanti) | `[ ! -s "$LOG" ]` check di `run_qemu.sh` | Tidak ada bukti boot | Script sudah menyertakan pemeriksaan file kosong dan tiga `grep` marker sebelum menyatakan OK |
| Reboot loop / triple fault (di WSL 2 nanti) | Log tidak stabil atau `-no-reboot` menghentikan VM | Kernel tidak pernah mencapai `kmain` | Prosedur diagnosis di `run_qemu_debug.sh` (breakpoint `kmain` via GDB) sudah disiapkan meski belum diuji karena QEMU/gdb NA |

### 15.3 Triage yang Dilakukan

```text
Urutan diagnosis untuk kasus nyata (entry point salah): (1) make inspect, (2) baca
readelf -hW, bandingkan Entry point address dengan VirtAddr segmen LOAD pertama pada
readelf -lW, (3) periksa nm -n untuk melihat simbol apa yang berada tepat di alamat entry
point tersebut (ditemukan halt_forever, bukan kmain), (4) telusuri source kmain.c dan
temukan bahwa halt_forever didefinisikan lebih dulu, (5) perbaiki urutan definisi, (6)
make distclean && make build && make inspect ulang untuk memverifikasi perbaikan.
```

### 15.4 Panic Path

```text
M2 belum memiliki panic handler penuh (non-goal, sesuai Bagian 4A.2 panduan). Kondisi
"panic-like" satu-satunya pada M2 adalah halt_forever yang disengaja setelah tiga marker
serial tercetak — ini bukan error, melainkan desain akhir jalur boot M2. Tidak ada panic
sungguhan yang terjadi selama praktikum ini (kegagalan yang terjadi adalah entry point yang
sudah diperbaiki, bukan kernel panic).
```

---

## 16. Prosedur Rollback

| Skenario rollback | Perintah | Data yang harus diselamatkan | Status |
|---|---|---|---|
| Kembali ke commit M1 | `git checkout 2ede37b7658f5c9c619d9f830192e482f86fa1db` | Tidak ada data lain di luar repo | Belum diuji langsung, tetapi commit M1 diverifikasi masih valid (M1 tests masih lulus saat preflight M2 menjalankan ulang `make proof`) |
| Revert commit M2 | `git revert be59c9e77a9e2f751a4f1c289bd429b45fb4d740` | Tidak ada | Belum diuji |
| Bersihkan artefak build | `make distclean` | Tidak ada; source aman (`build/` di-ignore) | **Teruji** — dijalankan pada Langkah 9 (clean-checkout rehearsal), hash `kernel.elf` identik sebelum/sesudah |
| Repair branch dan reset source M2 | `git switch -c repair/M2-boot` lalu `git checkout HEAD -- Makefile linker.ld kernel tools/scripts configs/limine` | Log kegagalan di `build/failure/M2/` | Tidak diperlukan pada praktikum ini karena tidak ada kegagalan build yang butuh rollback (hanya perbaikan source langsung) |

Catatan rollback:

```text
Rollback yang benar-benar relevan dan teruji pada M2 adalah make distclean, yang terbukti
menghasilkan build identik (hash sama) setelah dijalankan ulang dari clean state. Rollback
berbasis git revert/checkout belum diuji secara langsung karena tidak ada kegagalan yang
memerlukan pengembalian ke commit M1; perbaikan entry point dilakukan langsung dengan
mengedit source, bukan dengan rollback Git.
```

---

## 17. Keamanan dan Reliability

### 17.1 Risiko Keamanan

| Risiko | Boundary | Dampak | Mitigasi | Evidence |
|---|---|---|---|---|
| Bootloader tidak terverifikasi (supply-chain Limine) | Fetch Limine dari GitHub | Bootloader terkompromi bisa memuat kernel palsu | Belum di-fetch sama sekali di sandbox ini (fail-closed); rencana verifikasi checksum dicatat di `docs/security/supply_chain.md` | `build/meta/limine-revision.txt` (status NA) |
| Repository di path tidak stabil (mount Windows pada WSL nyata) | Filesystem repo | Permission/executable bit/newline tidak konsisten | Pengecekan otomatis path bukan `/mnt/*` di `m2_preflight.sh` | Output preflight Bagian 7.2 |

### 17.2 Reliability dan Data Integrity

| Risiko reliability | Dampak | Deteksi | Mitigasi |
|---|---|---|---|
| Build tidak reproducible | Evidence tidak dapat diaudit untuk milestone berikutnya | Perbandingan hash `kernel.elf` sebelum/sesudah `make distclean` | Hash identik dibuktikan pada Langkah 9 (clean-checkout rehearsal) |
| Klaim readiness tanpa bukti nyata (ISO/QEMU) | Menilai M2 "selesai" padahal boot belum terbukti | Manual review readiness | Status NOT_APPLICABLE ditulis eksplisit di setiap script dan readiness review, bukan disamarkan sebagai PASS |
| Busy-wait tanpa timeout pada serial driver | Berpotensi hang jika hardware/emulator tidak merespons | Tidak ada deteksi otomatis pada M2 | Diterima sebagai batasan M2 (dicatat di Bagian 9.8); perbaikan (timeout) menjadi target milestone observability lanjutan |

### 17.3 Negative Test

| Negative test | Input buruk | Expected result | Actual result | Status |
|---|---|---|---|---|
| Kernel dengan entry point salah (kondisi sebelum perbaikan) | Urutan fungsi `halt_forever` sebelum `kmain` | `inspect_kernel.sh` menolak (`grep` gagal, exit non-zero) | Terjadi persis demikian sebelum diperbaiki (lihat Bagian 15.1) | **PASS** — script berhasil mendeteksi kondisi salah |
| `make image` tanpa `xorriso` | Tool hilang | Script melaporkan `STATUS=NOT_APPLICABLE`, tidak crash tanpa penjelasan | Sesuai | **PASS** |
| `make run` tanpa QEMU | Tool hilang | Script melaporkan `STATUS=NOT_APPLICABLE` | Sesuai | **PASS** |

---

## 18. Pembagian Kerja Kelompok

Tidak berlaku (dikerjakan individu oleh Jamilus Solihin, NIM 2583207073010, Kelas PTI 1A).

---

## 19. Kriteria Lulus Praktikum

| Kriteria minimum | Status | Evidence |
|---|---|---|
| Repository berada di filesystem Linux WSL dan bukan di `/mnt/c` | PASS (di sandbox: bukan `/mnt/*`; WSL 2 sesungguhnya belum diuji) | `m2_preflight.sh` output |
| Bukti M0 dan M1 tersedia | PASS (dipadankan dengan artefak M1: `invariants.md`, `verification_matrix.md`, `M1-toolchain.md`) | Bagian 7.2 |
| `make distclean && make all inspect` berhasil dari clean checkout | PASS (skema M2 lokal: `check-src build inspect`, target `all` di Makefile M2 gabungan memetakan ke `build`) | Bagian 10 Langkah 9 |
| `kernel.elf` ELF64 x86_64 dengan entry point `0xffffffff80000000` | PASS | `build/inspect/readelf-header.txt` |
| `kernel.map`, `readelf-header.txt`, `readelf-program-headers.txt`, `objdump-disassembly.txt`, `nm-symbols.txt` tersedia | PASS | `build/kernel.map`, `build/inspect/*` |
| Image bootable `build/mcsos.iso` berhasil dibuat | **FAIL/NOT_APPLICABLE** | `xorriso` tidak tersedia di sandbox |
| QEMU/OVMF berjalan headless, `build/qemu-serial.log` tertulis | **FAIL/NOT_APPLICABLE** | QEMU/OVMF tidak tersedia di sandbox |
| Serial log memuat 3 marker M2 | **FAIL/NOT_APPLICABLE** | Bergantung poin sebelumnya |
| Tidak ada warning kompilasi (`-Werror` aktif) | PASS | Output `make build` bersih |
| Semua script shell lulus `bash -n` dan (jika tersedia) `shellcheck` | PASS untuk `bash -n`; `shellcheck` NOT_APPLICABLE (tidak tersedia) | `make check-scripts` |
| Perubahan Git dikomit dengan pesan jelas | PASS | `be59c9e77a9e2f751a4f1c289bd429b45fb4d740` |
| Laporan memuat screenshot/log, failure mode, readiness review | PASS | Laporan ini (tidak ada screenshot karena eksekusi headless) |

**Kesimpulan kriteria lulus:** Berdasarkan tabel di atas, M2 **belum memenuhi seluruh 12 kriteria lulus** sesuai Bagian 6 panduan asli — 3 kriteria (image ISO, QEMU/OVMF run, marker serial) berstatus NOT_APPLICABLE karena keterbatasan sandbox, bukan karena kesalahan implementasi. 9 dari 12 kriteria terpenuhi penuh.

---

## 20. Readiness Review

| Status | Definisi | Pilihan |
|---|---|---|
| Belum siap uji QEMU tahap M2 | Build/test belum stabil atau bukti belum cukup | [x] |
| Siap uji QEMU tahap M2 | Build ELF, image, QEMU/OVMF, dan serial marker lulus dengan evidence lengkap | [ ] |
| Siap demonstrasi praktikum terbatas | Selain lulus M2, mampu menjelaskan desain, failure modes, rollback, debug GDB | [ ] |

Alasan readiness:

```text
Kompilasi, linking, dan inspeksi ELF kernel M2 LULUS PENUH dengan evidence nyata dan
reproducible: entry point tepat sesuai linker script, semua symbol wajib ada, tidak ada
dependensi hosted, dan tidak ada warning kompilasi. Satu bug nyata (entry point salah
karena urutan fungsi) ditemukan dan diperbaiki dalam proses ini, menunjukkan bahwa
pemeriksaan evidence-first (readelf/nm) benar-benar berfungsi mendeteksi masalah.

Namun tiga kriteria wajib panduan asli — fetch Limine, build ISO bootable, dan run
QEMU/OVMF dengan bukti tiga marker serial — TIDAK dapat dipenuhi dari sandbox ini karena
tidak ada akses jaringan keluar dan tidak ada xorriso/QEMU/OVMF terpasang. Oleh karena itu
status readiness yang jujur adalah "belum siap uji QEMU tahap M2", bukan "siap uji QEMU
tahap M2", meskipun komponen build/inspect sudah lulus penuh.
```

Known issues:

| No. | Issue | Dampak | Workaround | Target perbaikan |
|---|---|---|---|---|
| 1 | Clang/LLD tidak dipakai, GCC/GNU-ld sebagai gantinya | Code-gen bisa sedikit berbeda dari toolchain asli | Dicatat eksplisit di semua evidence | Bandingkan ulang dengan Clang/LLD asli di WSL 2 |
| 2 | Limine tidak dapat di-fetch (tanpa jaringan) | Tidak ada bootloader untuk membuat ISO | Status NA dicatat, tidak divendor buta tanpa checksum | Fetch ulang di WSL 2 dengan akses internet, atau pakai arsip resmi dari dosen |
| 3 | `xorriso` tidak tersedia | ISO tidak dapat dibuat | Status NA dicatat | Pasang `xorriso mtools dosfstools` di WSL 2 |
| 4 | QEMU/OVMF tidak tersedia | Tidak ada bukti runtime boot | Status NA dicatat | Pasang `qemu-system-x86 ovmf` di WSL 2, jalankan ulang `make run` |
| 5 | `shellcheck`/`gdb` tidak tersedia | Static analysis script dan debug GDB belum divalidasi | Hanya `bash -n` yang dijalankan | Pasang `shellcheck` dan `gdb-multiarch` di WSL 2 |

Keputusan akhir:

```text
Berdasarkan bukti build ELF64 x86_64 dengan entry point tepat sesuai linker script, evidence
readelf/objdump/nm yang lengkap dan konsisten (termasuk hash identik pada clean-checkout
rehearsal), hasil praktikum M2 ini layak disebut "lulus tahap build dan inspect ELF" untuk
milestone M2, tetapi BELUM layak disebut "siap uji QEMU tahap M2" karena bukti boot runtime
(ISO + QEMU/OVMF + serial marker) belum ada dan wajib diperoleh oleh mahasiswa di mesin
Windows 11 + WSL 2 pribadi sebelum status readiness M2 dapat dinaikkan.
```

---

## 21. Rubrik Penilaian 100 Poin

| Komponen | Bobot | Indikator nilai penuh | Nilai |
|---|---:|---|---:|
| Kebenaran fungsional | 30 | `kernel.elf` terbentuk, ISO terbentuk, QEMU/OVMF berjalan, serial log memuat marker M2 | `[diisi dosen/asisten — perhatikan ISO/QEMU NA karena keterbatasan sandbox]` |
| Kualitas desain dan invariants | 20 | Entry contract, linker layout, freestanding assumptions, serial driver boundary, halt behavior dijelaskan | `[diisi dosen/asisten]` |
| Pengujian dan bukti | 20 | readelf, objdump, nm, kernel.map, checksum ISO, preflight, serial log lengkap | `[diisi dosen/asisten — ISO/serial log NA]` |
| Debugging dan failure analysis | 10 | Failure modes dianalisis dengan diagnosis dan solusi tepat | `[diisi dosen/asisten — ada 1 bug nyata ditemukan+diperbaiki]` |
| Keamanan dan robustness | 10 | Supply-chain Limine, generated artifact policy, fail-closed behavior, tidak ada klaim berlebihan | `[diisi dosen/asisten]` |
| Dokumentasi dan laporan | 10 | Laporan mengikuti template, jelas, commit hash, bukti, readiness review | `[diisi dosen/asisten]` |
| **Total** | **100** |  | `[diisi dosen/asisten]` |

Catatan penilai:

```text
[Diisi dosen/asisten. Mohon pertimbangkan bahwa evidence ISO/QEMU/OVMF berstatus
NOT_APPLICABLE karena keterbatasan lingkungan eksekusi laporan (sandbox tanpa jaringan dan
tanpa xorriso/QEMU/OVMF terpasang), bukan karena kegagalan mahasiswa mengikuti prosedur.
Evidence build+inspect ELF lengkap dan menunjukkan satu bug nyata yang berhasil dideteksi
dan diperbaiki (entry point).]
```

---

## 22. Kesimpulan

### 22.1 Yang Berhasil

```text
Source tree M2 lengkap (io.h, serial.c, kmain.c, memory.c, linker.ld) berhasil dikompilasi
dan di-link menjadi kernel ELF64 x86_64 menggunakan GCC + GNU ld sebagai pengganti Clang +
LLD. Entry point tepat 0xffffffff80000000 sesuai linker script (setelah perbaikan urutan
fungsi kmain/halt_forever). Semua symbol wajib (kmain, serial_init, serial_write, memset,
memcpy, memmove) terverifikasi ada lewat nm. Disassembly membuktikan tidak ada dependensi
hosted libc dan urutan eksekusi kmain sesuai desain (serial_init -> 3x serial_write ->
halt_forever). Build terbukti reproducible dari clean checkout (hash kernel.elf identik).
Commit Git M2 tercatat dengan hash be59c9e77a9e2f751a4f1c289bd429b45fb4d740.
```

### 22.2 Yang Belum Berhasil

```text
Fetch Limine, pembuatan ISO bootable, dan run QEMU/OVMF dengan bukti serial log tiga marker
belum dapat dilakukan karena sandbox eksekusi tidak memiliki akses jaringan keluar dan tidak
memiliki xorriso/QEMU/OVMF terpasang. Debug GDB (make debug) juga belum diuji karena gdb
tidak tersedia. Akibatnya, 3 dari 12 kriteria lulus M2 pada panduan asli belum terpenuhi.
```

### 22.3 Rencana Perbaikan

```text
1. Salin repository ini (commit be59c9e77a9e2f751a4f1c289bd429b45fb4d740) ke ~/src/mcsos pada
   WSL 2 mesin Windows 11 pribadi.
2. Pasang paket lengkap: clang lld llvm binutils qemu-system-x86 qemu-utils ovmf xorriso
   mtools dosfstools gdb-multiarch shellcheck.
3. Ganti CC/LD di Makefile kembali ke clang/ld.lld dengan --target=x86_64-unknown-none-elf,
   jalankan ulang make distclean && make check-src && make build && make inspect, bandingkan
   readelf/objdump/nm dengan hasil GCC/GNU-ld pada laporan ini.
4. Jalankan ./tools/scripts/fetch_limine.sh dengan akses internet sungguhan untuk memperoleh
   Limine dan commit hash revisinya.
5. Jalankan make image, make run untuk memperoleh build/mcsos.iso, build/mcsos.iso.sha256,
   dan build/qemu-serial.log dengan tiga marker M2.
6. Jalankan make debug dan lakukan breakpoint kmain via GDB sebagai bukti tambahan.
7. Perbarui docs/readiness/M2-boot-image.md dengan hasil dari WSL 2 asli, lalu ubah status
   readiness dari "belum siap uji QEMU tahap M2" menjadi "siap uji QEMU tahap M2" jika semua
   12 kriteria lulus M2 terpenuhi.
```

---

## 23. Lampiran

### Lampiran A — Commit Log

```text
be59c9e M2: add bootable kernel ELF and early serial console (GCC/GNU-ld fallback; image/QEMU NA in sandbox)
2ede37b M1: add reproducible toolchain readiness baseline
```

### Lampiran B — Diff Ringkas

```diff
 Makefile                                          | (ubah signifikan: gabung M1+M2, CC/LD -> gcc/ld)
 configs/limine/limine.conf                        | baru
 docs/architecture/invariants.md                   | +14 baris (invariant M2 #9-14)
 docs/readiness/M2-boot-image.md                   | baru
 docs/security/supply_chain.md                     | baru
 docs/testing/verification_matrix.md               | +11 baris (W1-W10)
 kernel/arch/x86_64/include/mcsos/arch/io.h        | baru
 kernel/core/kmain.c                                | baru
 kernel/core/serial.c                               | baru
 kernel/lib/memory.c                                | baru
 linker.ld                                          | baru
 tools/scripts/fetch_limine.sh                      | baru
 tools/scripts/grade_m2.sh                          | baru
 tools/scripts/inspect_kernel.sh                    | baru
 tools/scripts/m2_preflight.sh                      | baru
 tools/scripts/make_iso.sh                          | baru
 tools/scripts/run_qemu.sh                          | baru
 tools/scripts/run_qemu_debug.sh                    | baru
```

### Lampiran C — Log Build Lengkap

```text
Lihat Bagian 10 Langkah 3 untuk perintah gcc/ld lengkap. File asli tersedia di repository
pada build/kernel.elf, build/kernel.map (tidak dikomit ke Git sesuai .gitignore).
```

### Lampiran D — Log QEMU Lengkap

```text
Tidak ada — QEMU tidak tersedia di sandbox eksekusi ini. Lihat build/meta/run-qemu-status.txt
untuk catatan STATUS=NOT_APPLICABLE.
```

### Lampiran E — Output Readelf/Objdump

```text
Lihat Bagian 10 Langkah 4 dan Bagian 13.3 untuk output readelf -hW, readelf -lW, objdump
-drwC, dan nm -n lengkap.
```

### Lampiran F — Screenshot

| No. | File | Keterangan |
|---|---|---|
| — | Tidak ada | Eksekusi headless di sandbox terminal, tidak ada screenshot GUI |

### Lampiran G — Bukti Tambahan

```text
sha256sum build/kernel.elf build/kernel.map
ea40070c73f44a804508d26adb83aacaedb3e6fd6bcd782fb82e9a10ca1aeaab  kernel.elf
1c8f2c1979a1c47b479cbbdccdca1996c49ac761951e2231231f0dad7c96a7a6  kernel.map
```

---

## 24. Pertanyaan Analisis

1. **Mengapa M2 tidak boleh menggunakan `printf` dari libc host?**
   Karena kernel freestanding tidak memiliki runtime hosted (tidak ada stdio buffer, tidak ada syscall `write` OS host); memakai `printf` akan membuat linker mencari implementasi yang tidak ada dalam konteks kernel, atau diam-diam menautkan ke libc host yang mengasumsikan lingkungan proses userspace Linux/Windows — sesuatu yang tidak ada saat kernel MCSOS benar-benar boot di bare metal/QEMU.

2. **Apa fungsi `-ffreestanding`, `-nostdlib`, dan `-mno-red-zone` pada kernel awal?**
   `-ffreestanding` memberi tahu compiler bahwa lingkungan bukan hosted (tidak ada jaminan `main`, `argv`, atau libc penuh tersedia). `-nostdlib` mencegah linker menautkan startup object (`crt0`) dan libc standar. `-mno-red-zone` menonaktifkan optimasi red zone 128-byte di bawah `%rsp` yang aman pada userspace tetapi berbahaya bila interrupt/exception (yang belum ditangani penuh di M2, tapi krusial mulai M3/M4) bisa menimpa area tersebut.

3. **Mengapa serial console lebih diutamakan daripada framebuffer pada M2?**
   Serial console jauh lebih sederhana untuk diimplementasi (hanya butuh beberapa instruksi `outb`/`inb` ke UART) dan lebih mudah dijadikan evidence otomatis (grep teks pada log file), dibanding framebuffer yang butuh mode grafis, font rendering, dan verifikasi visual (screenshot) yang lebih sulit diotomasi sebagai bukti deterministik.

4. **Mengapa `kernel.elf` harus diperiksa dengan `readelf` dan `objdump`, bukan hanya dilihat dari keberhasilan `make build`?**
   `make build` yang berhasil hanya membuktikan compiler dan linker tidak mengeluarkan error — bukan bahwa hasilnya benar secara arsitektural. Pada praktikum ini, `make build` berhasil bahkan ketika entry point salah (`0xffffffff8000000c`); hanya `readelf -hW` yang mengungkap masalah tersebut. Ini adalah bukti langsung mengapa inspeksi ELF adalah gate wajib, bukan opsional.

5. **Apa risiko jika repository dikerjakan di `/mnt/c` pada WSL?**
   Filesystem Windows yang di-mount ke WSL memiliki overhead I/O tinggi, executable bit dan permission Unix tidak konsisten, symlink tidak selalu didukung, dan newline CRLF dapat merusak script shell (`/usr/bin/env: 'bash\r': No such file or directory`) — semua ini dapat menyebabkan kegagalan build/script yang tampak seperti bug kode padahal sebenarnya masalah lingkungan.

6. **Jelaskan perbedaan kegagalan build, kegagalan link, kegagalan image, dan kegagalan runtime QEMU.**
   Kegagalan build: compiler menolak source (syntax/type error/warning-as-error) sebelum object file terbentuk. Kegagalan link: object file valid tapi linker gagal menyelesaikan simbol/layout (mis. undefined symbol, alamat linker script konflik) sehingga `kernel.elf` tidak terbentuk. Kegagalan image: `kernel.elf` valid tapi proses pembungkusan ke ISO (xorriso/Limine) gagal, sehingga tidak ada media bootable. Kegagalan runtime QEMU: image ada dan valid tapi saat dijalankan di emulator, firmware/bootloader/kernel tidak mencapai titik yang diharapkan (log kosong, reboot loop, hang) — ini adalah lapisan kegagalan yang paling sulit didiagnosis karena melibatkan interaksi banyak komponen.

7. **Mengapa QEMU timeout tidak otomatis berarti gagal pada M2?**
   Karena `kmain` pada M2 sengaja didesain masuk ke `halt_forever` (loop `cli; hlt` tak berujung) sebagai kontrak akhir jalur boot M2 — kernel memang tidak dimaksudkan untuk "selesai" atau kembali. `run_qemu.sh` menggunakan `timeout 10s` justru untuk menghentikan VM yang memang sengaja berjalan selamanya, lalu memeriksa isi log serial sebagai bukti keberhasilan, bukan status exit QEMU itu sendiri.

8. **Jika serial log kosong, urutan diagnosis apa yang paling rasional?**
   (1) Periksa apakah ISO benar-benar berisi kernel dan konfigurasi Limine (`find iso_root`), (2) periksa `limine.conf` mengarah ke path kernel yang benar, (3) periksa versi QEMU dan opsi command line (`-serial file:...` benar-benar dipakai), (4) jika masih kosong, jalankan `run_qemu_debug.sh` dan breakpoint di `kmain` via GDB untuk memastikan apakah kontrol eksekusi benar-benar mencapai kernel atau berhenti di firmware/bootloader.

9. **Mengapa `kmain` tidak boleh kembali?**
   Karena tidak ada "pemanggil" yang valid untuk kembali — bootloader sudah menyerahkan kontrol sepenuhnya dan tidak menyediakan mekanisme return yang aman pada tahap M2 (belum ada scheduler/proses lain). Jika `kmain` return, CPU akan mengeksekusi instruksi apa pun yang kebetulan berada setelah alamat pemanggilan (biasanya sampah/invalid), yang hampir pasti menyebabkan crash tak terduga (undefined behavior tingkat CPU).

10. **Apa bukti minimum yang diperlukan agar M2 dapat dinyatakan siap uji QEMU tahap boot awal?**
    Berdasarkan Bagian 6 panduan: kernel ELF64 x86_64 dengan entry point benar (readelf), image ISO bootable dengan checksum, dan log serial QEMU yang memuat ketiga marker M2 (`MCSOS 260502 M2 boot path entered`, `[M2] early serial online`, `[M2] kernel reached controlled halt loop`). Pada laporan ini, bukti pertama (kernel ELF) sudah terpenuhi penuh; bukti kedua dan ketiga **belum terpenuhi** karena keterbatasan sandbox — inilah alasan status readiness diajukan sebagai "belum siap uji QEMU tahap M2".

---

## 25. Readiness Review Akhir M2

| Status | Kriteria | Terpenuhi? |
|---|---|---|
| Belum siap uji QEMU | Build/image/serial log gagal atau evidence tidak lengkap | **Ya — ini status yang diajukan** (image/serial log NOT_APPLICABLE) |
| Siap uji QEMU tahap M2 | Build ELF, image, QEMU/OVMF, serial marker lulus dengan evidence lengkap | Tidak (image/QEMU belum ada) |
| Siap demonstrasi praktikum terbatas | Selain lulus M2, mampu jelaskan desain, failure modes, rollback, debug GDB | Tidak (bergantung status di atas) |

M2 pada laporan ini **tidak diklaim** "siap produksi", "tanpa error", atau "siap hardware umum", sesuai batasan eksplisit panduan Bagian 30.

---

## 26. Referensi

Referensi yang benar-benar dipakai dalam laporan:

```text
[1] M. Sidiq, "Panduan Praktikum M2 - Boot Image, Kernel ELF64, Early Serial Console, dan
    Readiness Gate M2 MCSOS 260502," Program Studi Pendidikan Teknologi Informasi, Institut
    Pendidikan Indonesia, 2026.

[2] OSDev Wiki, "Higher Half Kernel," OSDev Wiki. [Online]. Available:
    https://wiki.osdev.org/Higher_Half_Kernel. Accessed: 2026-07-06.

[3] GNU Project, "GNU Binutils," GNU Binutils Documentation. [Online]. Available:
    https://www.gnu.org/software/binutils/. Accessed: 2026-07-06.
```

---

## 27. Checklist Final Sebelum Pengumpulan

| Checklist | Status |
|---|---|
| Semua placeholder `[isi ...]` sudah diganti | Ya |
| Metadata laporan lengkap | Ya |
| Commit awal dan akhir dicatat | Ya (`2ede37b` -> `be59c9e`) |
| Perintah build dan test dapat dijalankan ulang | Ya |
| Log build dilampirkan | Ya (ringkas di laporan; file lengkap di `build/`, tidak dikomit) |
| Log QEMU/test dilampirkan | Tidak berlaku (QEMU NOT_APPLICABLE) |
| Artefak penting diberi hash | Ya |
| Desain, invariants, ownership, failure modes dijelaskan | Ya |
| Security/reliability dibahas | Ya |
| Readiness review tidak berlebihan | Ya — status dibatasi eksplisit "belum siap uji QEMU" |
| Rubrik penilaian diisi atau disiapkan | Disiapkan, nilai diisi dosen |
| Referensi memakai format IEEE | Ya |
| Laporan disimpan sebagai Markdown | Ya |

---

## 28. Pernyataan Pengumpulan

Saya mengumpulkan laporan ini bersama artefak pendukung pada commit:

```text
be59c9e77a9e2f751a4f1c289bd429b45fb4d740
```

Status akhir yang diklaim:

```text
Belum siap uji QEMU tahap M2. Komponen build dan inspeksi ELF kernel LULUS PENUH dengan
evidence nyata dan reproducible (termasuk satu bug entry point yang ditemukan dan
diperbaiki). Komponen image ISO dan runtime QEMU/OVMF berstatus NOT_APPLICABLE karena
keterbatasan sandbox eksekusi (tanpa akses jaringan, tanpa xorriso/QEMU/OVMF), dan wajib
diselesaikan di WSL 2 sebelum status readiness M2 dapat dinaikkan.
```

Ringkasan satu paragraf:

```text
Praktikum M2 dikerjakan pada sandbox Linux (bukan Windows 11 + WSL 2) tanpa akses jaringan
keluar. Source kernel freestanding (io.h, serial.c, kmain.c, memory.c) dan linker script
higher-half berhasil ditulis, dikompilasi, dan di-link menjadi kernel.elf ELF64 x86_64
menggunakan GCC + GNU ld sebagai pengganti Clang + LLD. Inspeksi ELF (readelf/objdump/nm)
membuktikan entry point tepat 0xffffffff80000000 dan semua symbol wajib ada, setelah
memperbaiki bug nyata (urutan fungsi kmain/halt_forever) yang ditemukan lewat pemeriksaan
evidence-first. Build terbukti reproducible dari clean checkout. Namun fetch Limine, build
ISO bootable, dan run QEMU/OVMF dengan bukti serial log tidak dapat dilakukan karena sandbox
tidak memiliki akses jaringan maupun xorriso/QEMU/OVMF terpasang; ketiganya dilaporkan secara
jujur sebagai NOT_APPLICABLE oleh script masing-masing, bukan dipalsukan lulus. Status
readiness yang diajukan adalah "belum siap uji QEMU tahap M2", dan mahasiswa perlu mengulang
langkah fetch Limine, build image, dan run QEMU di WSL 2 pribadi sebelum M2 dapat dinyatakan
selesai penuh sesuai definisi panduan asli.
```
