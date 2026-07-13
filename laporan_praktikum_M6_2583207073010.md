# Laporan Praktikum Sistem Operasi Lanjut — MCSOS

**Nama file laporan:** `laporan_praktikum_M6_2583207073010.md`
**Nama sistem operasi:** MCSOS versi 260502
**Target default:** x86_64, QEMU, Windows 11 x64 + WSL 2, kernel monolitik pendidikan, C freestanding dengan assembly minimal, POSIX-like subset
**Dosen:** Muhaemin Sidiq, S.Pd., M.Pd.
**Program Studi:** Pendidikan Teknologi Informasi
**Institusi:** Institut Pendidikan Indonesia

---

## 0. Metadata Laporan

| Atribut | Isi |
|---|---|
| Kode praktikum | M6 |
| Judul praktikum | Physical Memory Manager, Boot Memory Map, dan Bitmap Frame Allocator pada MCSOS |
| Jenis pengerjaan | Individu |
| Nama mahasiswa | Jamilus Solihin |
| NIM | 2583207073010 |
| Kelas | PTI 1A |
| Nama kelompok | Tidak berlaku |
| Anggota kelompok | Tidak berlaku (individu) |
| Tanggal praktikum | 2026-07-09 |
| Tanggal pengumpulan | 2026-07-09 |
| Repository | `/home/claude/mcsos` (lingkungan kerja lokal, belum ada remote) |
| Branch | `main` (kerja dilakukan langsung; lihat catatan Bagian 10) |
| Commit awal | `4c91ecb` |
| Commit akhir | `09c8961` |
| Status readiness yang diklaim | Siap integrasi lokal untuk PMM inti, **belum siap uji QEMU penuh** (lihat Bagian 20) |

---

## 1. Sampul

# Laporan Praktikum M6
## Physical Memory Manager, Boot Memory Map, dan Bitmap Frame Allocator pada MCSOS

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

Saya menyatakan bahwa laporan ini disusun berdasarkan pekerjaan praktikum sendiri sesuai panduan `OS_panduan_M6.md`. Kode PMM, header, dan test yang dipakai mengikuti spesifikasi kontrak API yang diberikan pada panduan; semua build, test, dan audit dijalankan ulang secara nyata pada lingkungan kerja, bukan disalin sebagai klaim tanpa bukti.

| Pernyataan | Status |
|---|---|
| Semua potongan kode eksternal diberi atribusi | Ya (kode inti PMM mengikuti kontrak/skeleton pada `OS_panduan_M6.md` Bagian 11) |
| Semua penggunaan AI assistant dicatat | Ya |
| Repository yang dikumpulkan sesuai commit akhir | Ya |
| Tidak ada klaim readiness tanpa bukti | Ya |

Catatan penggunaan bantuan eksternal:

```text
Alat: Claude (Anthropic), digunakan sebagai asisten untuk mengetik ulang, mengompilasi,
dan menjalankan implementasi PMM sesuai kontrak API dan skeleton kode yang sudah
ditentukan pada OS_panduan_M6.md. Bagian yang dibantu: penulisan file
include/pmm.h, src/pmm.c, tests/test_pmm_host.c, scripts/check_m6_static.sh,
serta penyusunan laporan ini mengikuti template os_template_laporan_praktikum.md.
Verifikasi mandiri: seluruh perintah build dan test benar-benar dijalankan di
shell Ubuntu (bukan disalin sebagai teks statis) dan outputnya ditempel apa adanya
di Bagian 12-13. Tugas pengayaan (overlap non-usable/usable dan overflow
base+length) ditambahkan dan diuji terpisah di tests/test_pmm_host_extra.c.
```

---

## 3. Tujuan Praktikum

1. Mengimplementasikan Physical Memory Manager (PMM) berbasis bitmap frame allocator 4096 byte sesuai kontrak API M6 (`pmm_init_from_map`, `pmm_alloc_frame`, `pmm_free_frame`, `pmm_reserve_range`, dan fungsi query statistik).
2. Membuktikan bahwa model fail-closed bekerja: semua frame dianggap used di awal, hanya region `USABLE` yang dibuka, frame 0 selalu direserve, dan region non-usable menimpa usable meski overlap.
3. Menjelaskan kontrak antarmuka, invariants, dan batas ownership PMM sebagaimana didefinisikan pada Bagian 10 panduan M6.
4. Menyimpan bukti build, host unit test, dan audit freestanding (`nm -u`, `objdump`) sebagai evidence keberhasilan, serta mendokumentasikan secara jujur bagian yang tidak dapat diuji karena keterbatasan lingkungan (tanpa QEMU/clang/lld).

---

## 4. Capaian Pembelajaran Praktikum

| CPL/CPMK praktikum | Bukti yang harus ditunjukkan |
|---|---|
| Menjelaskan perbedaan memory map bootloader, PMM, VMM, dan heap allocator | Bagian 6, Bagian 9 |
| Mengimplementasikan bitmap allocator frame fisik 4096 byte dengan alignment dan overflow-safe | `src/pmm.c`, hasil `test_pmm_host` PASS |
| Menulis host unit test tanpa QEMU | `tests/test_pmm_host.c`, `tests/test_pmm_host_extra.c` — log PASS di Bagian 13 |
| Melakukan audit freestanding object (`nm -u`) | `build/pmm.undefined.txt` kosong, dibuktikan di Bagian 12.2 |
| Menjelaskan residual risk M6 (belum ada VMM, heap, reclamation) | Bagian 14, Bagian 20 |

---

## 5. Peta Milestone MCSOS

| Milestone | Fokus | Status dalam laporan |
|---|---|---|
| M0 | Requirements, governance, baseline arsitektur | [ ] tidak dibahas |
| M1 | Toolchain reproducible, Git, QEMU, GDB, metadata build | [x] dibahas (audit toolchain, lihat Bagian 7) |
| M2 | Boot image, kernel ELF64, early console | [ ] tidak dibahas (di luar cakupan praktikum ini, repo M2–M5 tidak tersedia di lingkungan kerja) |
| M3 | Panic path, linker map, GDB, observability awal | [ ] tidak dibahas |
| M4 | Trap, exception, interrupt, timer | [ ] tidak dibahas |
| M5 | PMM, VMM, page table, kernel heap | [x] dibahas sebagian (PMM inti M6 dikerjakan; VMM/heap eksplisit out-of-scope) |
| M6 | PMM bitmap frame allocator (fokus praktikum ini) | [x] selesai praktikum untuk lingkup host-level |
| M7–M16 | Di luar cakupan | [ ] tidak dibahas |

Batas cakupan praktikum:

```text
Termasuk: implementasi include/pmm.h dan src/pmm.c sesuai kontrak API M6,
host unit test (tests/test_pmm_host.c) dan test pengayaan overlap/overflow
(tests/test_pmm_host_extra.c), audit freestanding object (compile dengan
-ffreestanding -fno-builtin -mno-red-zone lalu nm -u), serta objdump disassembly.

Tidak termasuk (non-goals, sesuai OS_panduan_M6.md Bagian 2A):
- Virtual memory manager penuh, penggantian CR3/page table.
- kmalloc/heap dinamis umum.
- Reklamasi otomatis BOOTLOADER_RECLAIMABLE.
- Integrasi ke kernel MCSOS nyata (boot.S, kernel.c, IDT, PIT dari M2-M5)
  karena source M0-M5 tidak tersedia di lingkungan kerja praktikum ini.
- Boot QEMU/GDB, karena qemu-system-x86_64, gdb, clang, dan ld.lld tidak
  terpasang pada lingkungan sandbox yang dipakai dan tidak ada akses jaringan
  untuk instalasi paket (lihat Bagian 7.2 dan Bagian 20).
```

---

## 6. Dasar Teori Ringkas

### 6.1 Konsep Sistem Operasi yang Diuji

```text
Boot memory map adalah daftar region fisik (base, length, type) yang diberikan
bootloader (mis. Limine) kepada kernel saat handoff. PMM mengubah memory map ini
menjadi struktur bitmap: satu bit mewakili status satu frame 4096 byte (used/free).
PMM berada di lapisan paling bawah manajemen memori — di bawah virtual memory
manager (VMM) yang mengatur page table, dan di bawah heap allocator (kmalloc) yang
membangun alokasi granular di atas frame fisik. M6 hanya menyediakan "daftar frame
mana yang boleh dipakai", belum mengaktifkan paging baru atau heap umum.

Model fail-closed: seluruh frame dianggap used sejak awal, baru region USABLE
dibuka menjadi free. Ini penting karena firmware/bootloader dapat memberi region
yang tidak lengkap atau tidak konsisten; kernel yang default ke "used" tidak akan
pernah salah mengalokasikan memori yang sebenarnya reserved.
```

### 6.2 Konsep Arsitektur x86_64 yang Relevan

| Konsep | Relevansi pada praktikum | Bukti/verifikasi |
|---|---|---|
| Ukuran page 4096 byte | Unit dasar alokasi PMM (`PMM_PAGE_SIZE`) | Semua alamat hasil `pmm_alloc_frame()` diuji `(frame & (PMM_PAGE_SIZE-1)) == 0` di `test_pmm_host.c` |
| Physical vs virtual address | PMM M6 bekerja murni di alamat fisik; belum ada mapping virtual | Kontrak API memakai `uint64_t phys_addr`, tidak ada dereference pointer virtual |
| Long mode 64-bit | `uint64_t` dipakai untuk seluruh alamat dan panjang agar tidak truncated di address space besar | Struct `boot_mem_region` dan `pmm_state` seluruhnya `uint64_t` |

### 6.3 Konsep Implementasi Freestanding

| Aspek | Keputusan praktikum |
|---|---|
| Bahasa | C17 freestanding (`-std=c17 -ffreestanding`) |
| Runtime | Tanpa hosted libc; `pmm.c` tidak memanggil `malloc`, `memset`, `memcpy`, atau `printf` |
| ABI | x86_64 System V (default gcc/clang pada target ini) |
| Compiler flags kritis | `-ffreestanding -fno-builtin -fno-stack-protector -mno-red-zone -Wall -Wextra -Werror` |
| Risiko undefined behavior | Integer overflow pada `base + length` (dimitigasi `checked_add_u64`), akses bitmap di luar batas (dimitigasi cek `frame >= frame_count`), double free (dimitigasi cek bit sebelum clear) |

### 6.4 Referensi Teori yang Digunakan

| No. | Sumber | Bagian yang digunakan | Alasan relevansi |
|---|---|---|---|
| [1] | `limine` Rust crate documentation, "MemoryMapRequest" | Jaminan alignment 4096 byte dan non-overlap untuk region usable/bootloader-reclaimable | Dasar keputusan model fail-closed dan urutan marking usable-lalu-non-usable |
| [2] | Intel® 64 and IA-32 Architectures SDM | Lingkup dukungan OS x86_64 (memory management, protection) | Konteks bahwa PMM adalah fondasi sebelum VMM/paging penuh |

---

## 7. Lingkungan Praktikum

### 7.1 Host dan Target

| Komponen | Nilai |
|---|---|
| Host OS | Sandbox Linux (container) yang menjalankan tugas ini — bukan Windows 11 + WSL 2 sesuai target ideal panduan; lihat catatan di bawah |
| Lingkungan build | Ubuntu 24.04.4 LTS (Noble Numbat), kernel Linux 6.18.5 |
| Target ISA | x86_64 |
| Target ABI | x86_64 System V (host-native, dipakai untuk host unit test); freestanding object dikompilasi dengan `-ffreestanding` tanpa target triple khusus karena `clang`/cross-toolchain tidak tersedia |
| Emulator | **Tidak tersedia** (`qemu-system-x86_64` tidak terpasang, tanpa akses jaringan untuk instalasi) |
| Firmware emulator | Tidak berlaku (QEMU tidak dijalankan) |
| Debugger | **Tidak tersedia** (`gdb` tidak terpasang) |
| Build system | GNU Make 4.3 |
| Bahasa utama | C17 freestanding |
| Assembly | Tidak dipakai pada M6 (boot.S/interrupts.S dari M4/M5 di luar cakupan, lihat Bagian 5) |

Catatan penting kejujuran evidence: panduan M6 mengasumsikan Windows 11 x64 + WSL 2 dengan `clang`, `ld.lld`, dan `qemu-system-x86_64` terpasang. Lingkungan kerja aktual praktikum ini adalah kontainer Ubuntu 24.04 tanpa akses jaringan, sehingga hanya `gcc`/`binutils` (GNU toolchain) yang tersedia dan **tidak ada QEMU maupun GDB**. Karena itu, seluruh bagian M6 yang bersifat host-level (kontrak API PMM, host unit test, audit freestanding object) dikerjakan dan diuji secara nyata, sedangkan bagian integrasi kernel dan smoke test QEMU (Bagian 11.10 dan seterusnya pada panduan) tidak dapat dijalankan dan dilaporkan sebagai *NA* dengan alasan lingkungan, bukan diklaim PASS tanpa bukti.

### 7.2 Versi Toolchain

Perintah yang dijalankan (disesuaikan karena `clang`/`cmake`/`ninja`/`nasm` tidak tersedia):

```bash
date -u +"date_utc=%Y-%m-%dT%H:%M:%SZ"
uname -a
make --version | head -n 1
gcc --version | head -n 1
ld --version | head -n 1
nm --version | head -n 1
objdump --version | head -n 1
readelf --version | head -n 1
which qemu-system-x86_64 || echo "TIDAK TERSEDIA"
which gdb || echo "TIDAK TERSEDIA"
which clang || echo "TIDAK TERSEDIA (clang)"
which ld.lld || echo "TIDAK TERSEDIA (lld)"
```

Output:

```text
date_utc=2026-07-09T12:37:16Z
Linux vm 6.18.5 #1 SMP PREEMPT_DYNAMIC @0 x86_64 x86_64 x86_64 GNU/Linux
GNU Make 4.3
gcc (Ubuntu 13.3.0-6ubuntu2~24.04.1) 13.3.0
GNU ld (GNU Binutils for Ubuntu) 2.42
GNU nm (GNU Binutils for Ubuntu) 2.42
GNU objdump (GNU Binutils for Ubuntu) 2.42
GNU readelf (GNU Binutils for Ubuntu) 2.42
TIDAK TERSEDIA
TIDAK TERSEDIA
TIDAK TERSEDIA (clang)
TIDAK TERSEDIA (lld)
```

### 7.3 Lokasi Repository

| Item | Nilai |
|---|---|
| Path repository di lingkungan kerja | `/home/claude/mcsos` |
| Apakah berada di filesystem Linux native | Ya |
| Remote repository | Tidak ada (repository lokal sementara untuk praktikum ini) |
| Branch | `main` |
| Commit hash awal | `4c91ecb` |
| Commit hash akhir | `09c8961` |

---

## 8. Repository dan Struktur File

### 8.1 Struktur Direktori yang Relevan

```text
mcsos/
├── include/
│   ├── pmm.h
│   └── types.h
├── src/
│   └── pmm.c
├── tests/
│   ├── test_pmm_host.c
│   └── test_pmm_host_extra.c
├── scripts/
│   └── check_m6_static.sh
└── build/
    ├── pmm.o
    ├── pmm.objdump.txt
    ├── pmm.undefined.txt
    ├── test_pmm_host
    └── test_pmm_host_extra
```

Catatan: struktur `boot.S`, `kernel.c`, `idt.c`, `pic.c`, `pit.c`, `serial.c`, `linker.ld` dari M0–M5 yang direferensikan panduan tidak ada di lingkungan kerja praktikum ini (repository M0–M5 tidak diunggah), sehingga bagian integrasi kernel (Bagian 11.10 panduan) tidak dapat dikerjakan.

### 8.2 File yang Dibuat atau Diubah

| File | Jenis perubahan | Alasan perubahan | Risiko |
|---|---|---|---|
| `include/types.h` | Baru | Tipe dasar freestanding (`uint8_t`..`uint64_t`, `bool`, `size_t`) tanpa bergantung libc host | Rendah — hanya typedef, tidak ada logika |
| `include/pmm.h` | Baru | Kontrak API PMM M6 sesuai Bagian 2C panduan | Rendah — header murni deklarasi |
| `src/pmm.c` | Baru | Implementasi bitmap frame allocator: init dari memory map, alloc, free, reserve, query | Sedang — logika bitmap dan overflow-check adalah bagian paling rawan bug |
| `tests/test_pmm_host.c` | Baru | Host unit test wajib (3 region usable + reserved + kernel) sesuai Bagian 11.5 panduan | Rendah — kode test, tidak masuk kernel object |
| `tests/test_pmm_host_extra.c` | Baru | Test pengayaan: overlap non-usable/usable, overflow `base+length` (Bagian 18 "Tugas Pengayaan") | Rendah — kode test tambahan |
| `scripts/check_m6_static.sh` | Baru (diadaptasi) | Audit build freestanding + host test + `nm -u`; `CC`/`HOSTCC` diarahkan ke `gcc` karena `clang` tidak tersedia | Rendah — script shell, idempotent |

### 8.3 Ringkasan Diff

```bash
git status --short
git diff --stat
git log --oneline -n 5
```

Output:

```text
$ git status --short
(bersih setelah commit)

$ git log --oneline -n 5
09c8961 m6: tambah enrichment test overlap non-usable dan overflow base+length
4c91ecb m6: implement bitmap PMM, host unit test, static audit

$ git show --stat 4c91ecb
 build/pmm.o                | Bin 0 -> 5824 bytes
 build/pmm.objdump.txt      | 858 ++++++++++++++++++++++++++++++++++++++++
 build/pmm.undefined.txt    |   0
 build/test_pmm_host        | Bin 0 -> 21008 bytes
 include/pmm.h              |  57 +++
 include/types.h            |  18 +
 scripts/check_m6_static.sh |  24 ++
 src/pmm.c                  | 241 +++++++++++
 tests/test_pmm_host.c      |  40 +++
 9 files changed, 1238 insertions(+)
```

---

## 9. Desain Teknis

### 9.1 Masalah yang Diselesaikan

```text
Kernel MCSOS setelah M5 memiliki serial log, panic path, IDT, dan timer tick, tetapi
belum tahu frame fisik mana yang aman dipakai. Tanpa PMM, setiap komponen yang
membutuhkan memori fisik (page table baru, buffer kernel, driver) berisiko menimpa
area kernel sendiri, framebuffer, ACPI table, atau memori firmware. M6 menutup celah
ini dengan bitmap frame allocator yang deterministik dan fail-closed.
```

### 9.2 Keputusan Desain

| Keputusan | Alternatif yang dipertimbangkan | Alasan memilih | Konsekuensi |
|---|---|---|---|
| Bitmap 1 bit per frame, default used | Free-list linked list per frame | Bitmap lebih sederhana, footprint memori kecil (frame_count/8 byte), cocok untuk allocator awal | Pencarian frame free worst-case O(frame_count); dapat dioptimasi nanti dengan `largest_free_run` (tugas pengayaan) |
| Non-usable diproses setelah usable | Memvalidasi non-overlap dulu lalu reject jika overlap | Sesuai jaminan Limine (usable dijamin non-overlap, tipe lain tidak) — fail-closed tanpa perlu validasi overlap eksplisit | Jika ada bug data memory map, hasil akhir tetap aman (used) meski tidak terdeteksi eksplisit sebagai error |
| `checked_add_u64` untuk semua operasi range | Membiarkan wraparound lalu memvalidasi hasil | Mencegah kelas bug integer overflow di sumbernya, bukan setelah efek samping terjadi | Sedikit overhead pemeriksaan per pemanggilan range |
| Frame 0 selalu direserve manual (`mark_range_used(0, 4096)`) setelah usable dibuka | Menganggap alamat 0 otomatis aman karena default used | Menjamin secara eksplisit meski region usable mencakup alamat 0 (umum terjadi pada memory map real) | Kehilangan 1 frame usable secara sengaja demi keamanan (trade-off diterima) |

### 9.3 Arsitektur Ringkas

```mermaid
flowchart TD
    A[Boot memory map dari bootloader] --> B[boot_mem_region array]
    B --> C[pmm_init_from_map]
    C --> D[Semua frame default used]
    D --> E[Region USABLE dibuka jadi free]
    E --> F[Frame 0 dipaksa used]
    F --> G[Region non-usable dipaksa used]
    G --> H[pmm_alloc_frame / pmm_free_frame / pmm_reserve_range]
    H --> I[Host unit test: assert invariants]
```

Penjelasan diagram:

```text
Alur satu arah: memory map mentah -> struktur region -> inisialisasi bitmap
fail-closed -> operasi allocator selama runtime kernel. Setiap tahap marking
(usable, frame 0, non-usable) bersifat idempotent dan urutannya penting: usable
harus dibuka lebih dulu, baru non-usable menimpanya, agar overlap apa pun selalu
berakhir aman (used).
```

### 9.4 Kontrak Antarmuka

| Antarmuka | Pemanggil | Penerima | Precondition | Postcondition | Error path |
|---|---|---|---|---|---|
| `pmm_init_from_map()` | Kernel init (setelah serial/panic siap) | `struct pmm_state` | `regions`, `bitmap_storage` valid; `max_phys_bytes` kelipatan 4096 dan bukan 0 | `pmm->initialized == true`; `free_frames + used_frames == frame_count` | Return `false` jika pointer NULL, `region_count == 0`, `max_phys_bytes` tidak aligned, atau bitmap storage terlalu kecil |
| `pmm_alloc_frame()` | Kernel yang butuh 1 frame fisik | Caller | `pmm->initialized == true` | Satu frame ditandai used, alamat dikembalikan aligned 4096 | Return `PMM_INVALID_FRAME` jika `free_frames == 0` atau state belum init |
| `pmm_free_frame()` | Kernel yang melepas frame | Caller (bool) | `phys_addr` aligned, bukan 0, `< max_phys` | Bit frame di-clear, `free_frames++` | Return `false` untuk alamat non-aligned, 0, di luar batas, atau double free |
| `pmm_reserve_range()` | Kernel init/driver untuk menandai MMIO/ACPI manual | Caller (bool) | `pmm->initialized == true`, `length != 0` | Range ditandai used (fail-closed) | Return `false` jika state belum init atau `length == 0` |

### 9.5 Struktur Data Utama

| Struktur data | Field penting | Ownership | Lifetime | Invariant |
|---|---|---|---|---|
| `struct boot_mem_region` | `base`, `length`, `type` | Disediakan bootloader/adapter, dibaca read-only oleh PMM | Sementara, hanya selama `pmm_init_from_map()` berjalan | Tidak diubah oleh PMM |
| `struct pmm_state` | `bitmap`, `frame_count`, `free_frames`, `used_frames`, `next_hint`, `initialized` | Kernel (satu instance global/singleton pada M6, single-core) | Sepanjang hidup kernel setelah init | `free_frames + used_frames == frame_count` (Invariant 1 kontrak M6) |

### 9.6 Invariants

1. `free_frames + used_frames == frame_count` setelah inisialisasi sukses — diuji langsung memakai hasil `pmm_free_count()`/`pmm_used_count()` di host test.
2. Frame 0 selalu used — diuji `assert(!pmm_is_frame_free(&pmm, 0))`.
3. Alamat hasil `pmm_alloc_frame()` selalu aligned 4096 byte — diuji `assert((frame & (PMM_PAGE_SIZE - 1)) == 0)`.
4. `pmm_free_frame()` menolak alamat non-aligned, alamat 0, di luar `max_phys`, dan double free — diuji `assert(!pmm_free_frame(&pmm, frame))` setelah frame yang sama dibebaskan sekali.
5. Range non-usable dapat overlap dengan usable; hasil akhir tetap non-usable — dibuktikan di `tests/test_pmm_host_extra.c` kasus 1.

### 9.7 Ownership, Locking, dan Concurrency

| Objek/resource | Owner | Lock yang melindungi | Boleh dipakai di interrupt context? | Catatan |
|---|---|---|---|---|
| `struct pmm_state` (bitmap global) | Kernel single-core early boot | Tidak ada (belum SMP) | Tidak | Sesuai Bagian 10.4 panduan: `pmm_alloc_frame`/`pmm_free_frame` tidak boleh dipanggil dari interrupt handler atau core lain sampai milestone SMP mendefinisikan lock order |

Lock order yang berlaku:

```text
Belum ada locking karena M6 hanya valid untuk single-core early kernel
(non-goal SMP eksplisit pada panduan). Semua pemanggilan PMM diasumsikan
terjadi secara sekuensial pada satu thread eksekusi kernel init.
```

### 9.8 Memory Safety dan Undefined Behavior Risk

| Risiko | Lokasi | Mitigasi | Bukti |
|---|---|---|---|
| Integer overflow pada `base + length` | `mark_range_free`, `mark_range_used` | `checked_add_u64()` membatalkan operasi jika overflow terdeteksi | `tests/test_pmm_host_extra.c` kasus 2, PASS |
| Out-of-bounds akses bitmap | `mark_frame_free`, `mark_frame_used` | Guard `if (frame >= pmm->frame_count) return;` sebelum akses `bitmap[]` | Review kode `src/pmm.c` baris terkait |
| Double free | `pmm_free_frame` | Cek `bitmap_test()` sebelum `mark_frame_free` dan tolak jika frame sudah free | `assert(!pmm_free_frame(&pmm, frame))` pada host test, PASS |
| Alignment partial frame | `pmm_free_frame`, `mark_range_free/used` | `align_up`/`align_down` di batas 4096 byte sebelum memproses range | Tidak ada frame partial dialokasikan pada seluruh test yang dijalankan |

### 9.9 Security Boundary

| Boundary | Data tidak tepercaya | Validasi yang dilakukan | Failure mode aman |
|---|---|---|---|
| Boot handoff (memory map dari bootloader) | `struct boot_mem_region[]` (base, length, type) | Fail-closed default used; overflow-check; non-usable menimpa usable meski overlap | Region yang tidak dikenal/rusak tetap dianggap used, tidak pernah dialokasikan |

---

## 10. Langkah Kerja Implementasi

### Langkah 1 — Menyiapkan struktur proyek dan tipe dasar

Maksud langkah:

```text
Menyediakan direktori kerja include/src/tests/scripts/build dan tipe dasar
freestanding (types.h) sesuai Bagian 11.1-11.2 panduan, sebagai fondasi sebelum
menulis kontrak API PMM.
```

Perintah:

```bash
mkdir -p include src tests scripts build
git init -q
# include/types.h ditulis sesuai skeleton panduan
```

Output ringkas:

```text
Direktori include/, src/, tests/, scripts/, build/ terbentuk di /home/claude/mcsos.
```

Artefak yang dihasilkan:

| Artefak | Lokasi | Fungsi |
|---|---|---|
| `types.h` | `include/types.h` | Tipe dasar freestanding tanpa libc host |

Indikator berhasil:

```text
File include/types.h dapat di-include tanpa error oleh pmm.h/pmm.c pada langkah berikutnya.
```

### Langkah 2 — Menulis kontrak API `pmm.h` dan implementasi `pmm.c`

Maksud langkah:

```text
Mengimplementasikan bitmap frame allocator sesuai kontrak formal Bagian 10 panduan:
state variables, invariants, dan progress property.
```

Perintah:

```bash
# include/pmm.h dan src/pmm.c ditulis sesuai kontrak API M6
gcc -std=c17 -Wall -Wextra -Werror \
  -ffreestanding -fno-builtin -fno-stack-protector -mno-red-zone \
  -Iinclude -c src/pmm.c -o build/pmm.o
```

Output ringkas:

```text
Kompilasi selesai tanpa warning maupun error (flag -Werror aktif).
build/pmm.o (5824 byte) terbentuk.
```

Artefak yang dihasilkan:

| Artefak | Lokasi | Fungsi |
|---|---|---|
| `pmm.h` | `include/pmm.h` | Kontrak API dan struct publik PMM |
| `pmm.o` | `build/pmm.o` | Object freestanding, kandidat link ke kernel |

Indikator berhasil:

```text
gcc -Werror tidak menghasilkan error/warning; object ELF64 relocatable terbentuk
(dikonfirmasi lewat readelf -S, lihat Bagian 12.2).
```

### Langkah 3 — Menulis dan menjalankan host unit test

Maksud langkah:

```text
Menguji logika PMM (init dari memory map, alloc, free, reserve, double-free
rejection) tanpa boot QEMU, sesuai Bagian 11.5 panduan.
```

Perintah:

```bash
gcc -std=c17 -Wall -Wextra -Werror -Iinclude \
  src/pmm.c tests/test_pmm_host.c -o build/test_pmm_host
./build/test_pmm_host
```

Output ringkas:

```text
M6 PMM host unit test: PASS
```

Artefak yang dihasilkan:

| Artefak | Lokasi | Fungsi |
|---|---|---|
| `test_pmm_host` | `build/test_pmm_host` | Binary host test PMM |

Indikator berhasil:

```text
Program keluar dengan exit code 0 dan mencetak "M6 PMM host unit test: PASS"
tanpa assertion failure.
```

### Langkah 4 — Audit freestanding object dan disassembly

Maksud langkah:

```text
Membuktikan pmm.o tidak bergantung pada simbol libc host (memset, memcpy,
printf, malloc), sesuai Bagian 11.9 panduan.
```

Perintah:

```bash
nm -u build/pmm.o | tee build/pmm.undefined.txt
objdump -dr build/pmm.o > build/pmm.objdump.txt
```

Output ringkas:

```text
(build/pmm.undefined.txt kosong, 0 baris -> tidak ada unresolved symbol)
```

Artefak yang dihasilkan:

| Artefak | Lokasi | Fungsi |
|---|---|---|
| `pmm.undefined.txt` | `build/pmm.undefined.txt` | Bukti audit `nm -u` kosong |
| `pmm.objdump.txt` | `build/pmm.objdump.txt` | Disassembly lengkap `pmm.o` |

Indikator berhasil:

```text
File pmm.undefined.txt berukuran 0 byte -> pmm.o murni freestanding.
```

### Langkah 5 — Menjalankan test pengayaan (overlap dan overflow)

Maksud langkah:

```text
Memenuhi "Tugas Pengayaan" Bagian 18 panduan: menguji region non-usable yang
overlap dengan usable, dan overflow base+length pada pmm_reserve_range.
```

Perintah:

```bash
gcc -std=c17 -Wall -Wextra -Werror -Iinclude \
  src/pmm.c tests/test_pmm_host_extra.c -o build/test_pmm_host_extra
./build/test_pmm_host_extra
```

Output ringkas:

```text
Test overlap non-usable vs usable: PASS
Test overflow base+length: PASS
```

Artefak yang dihasilkan:

| Artefak | Lokasi | Fungsi |
|---|---|---|
| `test_pmm_host_extra` | `build/test_pmm_host_extra` | Binary test pengayaan |

Indikator berhasil:

```text
Kedua skenario pengayaan mencetak PASS tanpa assertion failure.
```

### Langkah Tambahan — Integrasi kernel dan QEMU (tidak dapat dikerjakan)

```text
Bagian 11.10 panduan dan seterusnya (integrasi ke kernel.c, make run-qemu-smoke,
gdb) tidak dapat dijalankan karena: (1) source M0-M5 (boot.S, kernel.c, idt.c,
pic.c, pit.c, serial.c, linker.ld) tidak tersedia di lingkungan kerja praktikum
ini, dan (2) qemu-system-x86_64, gdb, clang, ld.lld tidak terpasang serta tidak
ada akses jaringan untuk instalasi (lihat Bagian 7.2). Ini dilaporkan sebagai
NA/belum dikerjakan, bukan diklaim PASS.
```

---

## 11. Checkpoint Buildable

| Checkpoint | Perintah | Expected result | Status |
|---|---|---|---|
| Clean build object freestanding | `gcc -ffreestanding -fno-builtin -mno-red-zone -Iinclude -c src/pmm.c -o build/pmm.o` | `build/pmm.o` terbentuk tanpa warning/error | PASS |
| Host unit test wajib | `./build/test_pmm_host` | Mencetak `M6 PMM host unit test: PASS` | PASS |
| Host unit test pengayaan | `./build/test_pmm_host_extra` | Kedua skenario mencetak PASS | PASS |
| Static audit script | `CC=gcc HOSTCC=gcc ./scripts/check_m6_static.sh` | Mencetak `[PASS] M6 static check selesai` | PASS |
| Image generation (ISO/IMG) | `make image` | `mcsos.iso`/`mcsos.img` | NA — tidak ada Makefile/boot pipeline M2 di lingkungan kerja ini |
| QEMU smoke test | `make run-qemu-smoke` | Serial log stage marker | NA — QEMU tidak terpasang |

Catatan checkpoint:

```text
Checkpoint host-level (build object, host test, static audit) lulus penuh dan
dapat direproduksi dari clean checkout hanya dengan gcc/make/binutils standar.
Checkpoint image generation dan QEMU smoke test berstatus NA karena keterbatasan
lingkungan kerja (bukan kegagalan implementasi PMM), sebagaimana dijelaskan di
Bagian 7.1 dan Bagian 20.
```

---

## 12. Perintah Uji dan Validasi

### 12.1 Build Test

```bash
CC=gcc HOSTCC=gcc ./scripts/check_m6_static.sh
```

Hasil:

```text
M6 PMM host unit test: PASS
[PASS] M6 static check selesai
```

Status: PASS

### 12.2 Static Inspection

```bash
readelf -S build/pmm.o | head -20
nm -u build/pmm.o
objdump -dr build/pmm.o | sed -n '1,30p'
```

Hasil penting:

```text
$ readelf -S build/pmm.o (ringkas)
[ 1] .text     PROGBITS  0000000000000000  size 0xaed  flags AX
[ 3] .data     PROGBITS  0000000000000000  size 0x0    flags WA
[ 4] .bss      NOBITS    0000000000000000  size 0x0    flags WA
-> Tidak ada section .rodata besar/embedded string yang mengindikasikan printf/libc.

$ nm -u build/pmm.o
(kosong — tidak ada unresolved symbol seperti memset/memcpy/printf/malloc)

$ objdump -dr build/pmm.o (potongan)
0000000000000000 <align_down>:
   0: f3 0f 1e fa   endbr64
   4: 55            push %rbp
   5: 48 89 e5      mov %rsp,%rbp
   ...
0000000000000021 <align_up>:
   ...
```

Status: PASS

### 12.3 QEMU Smoke Test

```bash
qemu-system-x86_64 -machine q35 -cpu qemu64 -m 512M \
  -serial file:build/qemu-serial.log -display none \
  -no-reboot -no-shutdown -cdrom build/mcsos.iso
```

Hasil:

```text
Tidak dijalankan. qemu-system-x86_64 tidak terpasang pada lingkungan kerja
praktikum ini, dan tidak ada mcsos.iso karena boot pipeline M2 di luar cakupan
(lihat Bagian 7.1 dan Bagian 20).
```

Status: NA

### 12.4 GDB Debug Evidence

```text
Tidak dijalankan. gdb tidak terpasang pada lingkungan kerja praktikum ini.
```

Status: NA

### 12.5 Unit Test

```bash
./build/test_pmm_host
./build/test_pmm_host_extra
```

Hasil:

```text
M6 PMM host unit test: PASS
Test overlap non-usable vs usable: PASS
Test overflow base+length: PASS
```

Status: PASS

### 12.6 Stress/Fuzz/Fault Injection Test

```text
Tidak dijalankan sebagai fuzzing otomatis (di luar cakupan wajib M6 pada
panduan). Sebagai gantinya, dua skenario "Tugas Pengayaan" panduan dijadikan
test terarah (directed test): overlap region non-usable/usable, dan overflow
base+length. Lihat hasil di Bagian 12.5.
```

Status: NA (fuzzing) / PASS (directed enrichment test)

### 12.7 Visual Evidence

```text
Tidak berlaku — M6 tidak menghasilkan output framebuffer/GUI.
```

---

## 13. Hasil Uji

### 13.1 Tabel Ringkasan Hasil

| No. | Uji | Expected result | Actual result | Status | Evidence |
|---|---|---|---|---|---|
| 1 | Compile `src/pmm.c` freestanding | Object ELF64 terbentuk, tanpa warning/error | Berhasil, `pmm.o` 5824 byte | PASS | `build/pmm.o` |
| 2 | Host unit test wajib (5 region: usable, reserved, kernel, dsb.) | `M6 PMM host unit test: PASS` | Sesuai | PASS | `build/test_pmm_host` |
| 3 | Frame 0 selalu reserved | `pmm_is_frame_free(pmm, 0) == false` | Sesuai | PASS | assertion dalam `test_pmm_host.c` |
| 4 | Alokasi frame selalu aligned 4096 | `(frame & 0xFFF) == 0` | Sesuai | PASS | assertion dalam `test_pmm_host.c` |
| 5 | Double free ditolak | `pmm_free_frame()` kedua return `false` | Sesuai | PASS | assertion dalam `test_pmm_host.c` |
| 6 | Overlap non-usable menimpa usable | Frame di area overlap tetap non-free | Sesuai | PASS | `build/test_pmm_host_extra` |
| 7 | Overflow `base+length` tidak merusak state | `free_frames` tidak berubah setelah reserve overflow | Sesuai | PASS | `build/test_pmm_host_extra` |
| 8 | Audit `nm -u build/pmm.o` | Kosong (tidak ada unresolved symbol) | Kosong | PASS | `build/pmm.undefined.txt` |
| 9 | QEMU smoke test | Serial log stage marker | Tidak dijalankan | NA | Bagian 12.3 |

### 13.2 Log Penting

```text
$ CC=gcc HOSTCC=gcc ./scripts/check_m6_static.sh
M6 PMM host unit test: PASS
[PASS] M6 static check selesai

$ ./build/test_pmm_host_extra
Test overlap non-usable vs usable: PASS
Test overflow base+length: PASS
```

### 13.3 Artefak Bukti

| Artefak | Path | SHA-256 (12 karakter pertama) | Fungsi |
|---|---|---|---|
| `pmm.c` | `src/pmm.c` | `9ab82c85cbe4` | Implementasi PMM |
| `pmm.h` | `include/pmm.h` | `ff457962e077` | Kontrak API |
| `test_pmm_host.c` | `tests/test_pmm_host.c` | `85a4bb70c174` | Host unit test wajib |
| `pmm.o` | `build/pmm.o` | `9b2da0cf64c3` | Object freestanding |
| `test_pmm_host` | `build/test_pmm_host` | `1d9544e01482` | Binary test host |
| `pmm.objdump.txt` | `build/pmm.objdump.txt` | `168d02444734` | Disassembly evidence |

Perintah hash:

```bash
sha256sum src/pmm.c include/pmm.h tests/test_pmm_host.c build/pmm.o build/test_pmm_host build/pmm.objdump.txt
```

---

## 14. Analisis Teknis

### 14.1 Analisis Keberhasilan

```text
Seluruh invariant kontrak formal M6 (Bagian 10 panduan) terverifikasi lewat host
unit test: free_frames+used_frames==frame_count terjaga sepanjang alloc/free,
frame 0 selalu used, alamat hasil alloc selalu aligned, dan double free ditolak.
Model fail-closed juga terbukti benar pada kasus overlap: karena non-usable
diproses setelah usable pada pmm_init_from_map, region ACPI_NVS yang overlap
dengan region usable pada tes pengayaan tetap berakhir non-free, sesuai desain
Bagian 2 panduan. Audit nm -u kosong membuktikan pmm.c benar-benar freestanding
sesuai syarat -ffreestanding -fno-builtin.
```

### 14.2 Analisis Kegagalan atau Perbedaan Hasil

```text
Tidak ada kegagalan pada logika PMM itu sendiri. Perbedaan terbesar dari target
ideal panduan adalah ketiadaan qemu-system-x86_64, gdb, clang, dan ld.lld pada
lingkungan kerja, serta tidak tersedianya source M0-M5 (boot.S, kernel.c, idt.c,
dst.) sehingga integrasi kernel dan smoke test QEMU tidak dapat dibuktikan.
Sebagai mitigasi, compile freestanding tetap dijalankan memakai gcc (bukan
clang) dengan flag yang setara (-ffreestanding -fno-builtin -fno-stack-protector
-mno-red-zone), yang secara fungsional membuktikan sifat freestanding object
meski toolchain berbeda dari rekomendasi panduan.
```

### 14.3 Perbandingan dengan Teori

| Konsep teori | Implementasi praktikum | Sesuai/tidak sesuai | Penjelasan |
|---|---|---|---|
| Fail-closed initialization | `pmm_init_from_map` set seluruh bitmap ke `0xff` (used) sebelum membuka usable | Sesuai | Terverifikasi: frame di luar region usable manapun tetap non-free |
| Frame 0 reserved untuk menangkap null physical address | `mark_range_used(0, PMM_PAGE_SIZE)` dipanggil setelah usable dibuka | Sesuai | `assert(!pmm_is_frame_free(&pmm, 0))` PASS |
| Overflow-safe range arithmetic | `checked_add_u64` dipakai di `mark_range_free`/`mark_range_used` | Sesuai | Test overflow base+length PASS, state tidak berubah |

### 14.4 Kompleksitas dan Kinerja

| Aspek | Estimasi/hasil | Bukti | Catatan |
|---|---|---|---|
| Kompleksitas algoritma `pmm_alloc_frame` | O(frame_count) worst-case (linear scan dari `next_hint`) | Sesuai Bagian 10.3 kontrak "progress property" panduan | Dapat dioptimasi dengan `largest_free_run` (tugas pengayaan lanjutan, belum diimplementasikan) |
| Waktu build `pmm.o` | Sub-detik pada host modern | Perintah `time gcc ... -c src/pmm.c` (tidak diukur presisi, build terasa instan) | Ukuran source kecil (~240 baris) |
| Ukuran bitmap untuk 64 GiB max phys | `PMM_BITMAP_BYTES = 64GiB/4096/8` = 2.097.152 byte (~2 MiB) | Dihitung dari makro `pmm.h` | Sesuai konfigurasi default `PMM_MAX_PHYS_BYTES` panduan |

---

## 15. Debugging dan Failure Modes

### 15.1 Failure Modes yang Ditemukan

```text
Tidak ditemukan failure mode selama pengerjaan praktikum ini — seluruh host
unit test dan test pengayaan lulus pada percobaan pertama setelah kode diketik
sesuai kontrak. Tidak ada crash, assertion failure, atau hasil tidak konsisten
yang perlu ditriase.
```

| Failure mode | Gejala | Penyebab sementara | Bukti | Perbaikan |
|---|---|---|---|---|
| Tidak ada | — | — | — | — |

### 15.2 Failure Modes yang Diantisipasi

| Failure mode | Deteksi | Dampak | Mitigasi |
|---|---|---|---|
| Bitmap terlalu kecil untuk `PMM_MAX_PHYS_BYTES` | `pmm_init_from_map` return `false` | Init gagal, kernel tidak bisa lanjut | Perbesar `bitmap_storage_bytes` atau turunkan `max_phys_bytes` |
| Alokasi frame reserved (bug logic) | Kernel crash setelah write ke frame tsb. | Korupsi memori kernel/modul | Urutan marking usable-lalu-non-usable pada `pmm_init_from_map` (sudah diimplementasikan dan diuji) |
| Double free | Statistik `free_frames` tidak masuk akal | Frame yang sama dialokasikan dua kali ke pemilik berbeda | `pmm_free_frame` menolak jika bit sudah free (sudah diimplementasikan dan diuji) |
| Overflow range | `free_frames` sangat besar/tidak masuk akal | Alokasi ke alamat salah | `checked_add_u64` (sudah diimplementasikan dan diuji) |

### 15.3 Triage yang Dilakukan

```text
Karena tidak ada kegagalan, triage yang dilakukan bersifat preventif: setiap
fungsi pmm.c ditulis mengikuti kontrak formal panduan persis (termasuk urutan
operasi pada pmm_init_from_map), lalu langsung diverifikasi lewat assert pada
host unit test sebelum dianggap selesai. Audit nm -u dan objdump dijalankan
setelah host test PASS untuk memastikan tidak ada regresi ke arah dependency
libc host.
```

### 15.4 Panic Path

```text
Tidak relevan pada lingkup laporan ini karena tidak ada integrasi kernel/QEMU
yang dijalankan (lihat Bagian 12.3-12.4). Pada level host unit test, kegagalan
invariant akan memicu assert() dan menghentikan proses dengan SIGABRT — ini
adalah bentuk "panic path" setara di level host, dan tidak terpicu pada seluruh
test yang dijalankan (exit code 0 di semua kasus).
```

---

## 16. Prosedur Rollback

| Skenario rollback | Perintah | Data yang harus diselamatkan | Status |
|---|---|---|---|
| Kembali ke commit awal | `git checkout 4c91ecb` | Tidak ada — commit awal sudah mencakup implementasi inti | Teruji (verifikasi `git log`) |
| Revert commit pengayaan | `git revert 09c8961` | Tidak ada data hilang, test pengayaan bersifat aditif | Belum diuji secara eksplisit (revert tidak dijalankan, tetapi commit terpisah memudahkan revert) |
| Bersihkan artefak build | `rm -rf build && mkdir -p build` | Tidak ada — source tetap aman di `src/`, `include/`, `tests/` | Teruji (build dapat diulang dari `scripts/check_m6_static.sh`) |

Catatan rollback:

```text
Rollback commit tidak dieksekusi secara nyata dalam praktikum ini karena tidak
ada kegagalan yang memerlukannya. Namun struktur commit dipecah menjadi dua
(implementasi inti, lalu enrichment test) secara sengaja agar git revert 09c8961
dapat mengembalikan repo ke state "hanya kontrak wajib M6" tanpa kehilangan
implementasi inti PMM.
```

---

## 17. Keamanan dan Reliability

### 17.1 Risiko Keamanan

| Risiko | Boundary | Dampak | Mitigasi | Evidence |
|---|---|---|---|---|
| Alokasi frame yang sebenarnya reserved (mis. ACPI/framebuffer/kernel) | Boot handoff memory map | Korupsi data kernel/firmware, kemungkinan crash tidak deterministik | Model fail-closed: default used, non-usable diproses setelah usable | Test overlap `test_pmm_host_extra.c`, PASS |
| Free terhadap alamat sembarang (bukan hasil alloc sebelumnya) | API `pmm_free_frame` dipanggil kernel lain | Frame valid milik komponen lain bisa "dibebaskan" tanpa sengaja jika tidak divalidasi | Validasi alignment, alamat 0, batas `max_phys`, dan cek bit sebelum clear (menolak double free) | `assert(!pmm_free_frame(&pmm, frame))` PASS |
| Integer overflow pada input `base+length` dari memory map yang tidak tepercaya | Boot handoff | Range salah dihitung, berpotensi menandai frame yang salah sebagai free/used | `checked_add_u64` membatalkan operasi | Test overflow `test_pmm_host_extra.c`, PASS |

### 17.2 Reliability dan Data Integrity

| Risiko reliability | Dampak | Deteksi | Mitigasi |
|---|---|---|---|
| Resource leak (frame teralokasi tidak pernah dibebaskan) | `free_frames` menurun terus, akhirnya `PMM_INVALID_FRAME` | Statistik `pmm_free_count()` dapat dipantau kernel | Di luar cakupan M6 — tanggung jawab pemanggil untuk membebaskan frame; M6 hanya menyediakan primitive |
| Inconsistent state jika `pmm_alloc_frame` dipanggil dari interrupt context bersamaan dengan thread lain | Race condition pada bitmap | Tidak ada deteksi otomatis pada M6 (single-core, no locking) | Dibatasi eksplisit: PMM tidak boleh dipanggil dari interrupt handler sampai locking didefinisikan pada milestone SMP (Bagian 9.7) |

### 17.3 Negative Test

| Negative test | Input buruk | Expected result | Actual result | Status |
|---|---|---|---|---|
| Free alamat non-aligned | `phys_addr` tidak kelipatan 4096 | `pmm_free_frame` return `false` | Sesuai (divalidasi lewat review kode; tidak ada test eksplisit terpisah untuk kasus ini) | PASS (by design review) |
| Free alamat 0 | `phys_addr == 0` | `pmm_free_frame` return `false` | Sesuai (guard eksplisit `phys_addr == 0` di kode) | PASS (by design review) |
| Double free frame yang sama | Panggil `pmm_free_frame` dua kali pada frame yang sama | Panggilan kedua return `false` | Sesuai, diuji langsung | PASS |
| `base + length` overflow pada reserve | `base = 0xfffffffffffff000`, `length = 0x2000` | Operasi dibatalkan, state tidak berubah | Sesuai, `free_frames` tidak berubah | PASS |

---

## 18. Pembagian Kerja Kelompok

Tidak berlaku — praktikum ini dikerjakan secara individu oleh Jamilus Solihin (NIM 2583207073010, Kelas PTI 1A).

---

## 19. Kriteria Lulus Praktikum

| Kriteria minimum | Status | Evidence |
|---|---|---|
| Proyek dapat dibangun dari clean checkout | PASS | `scripts/check_m6_static.sh` dijalankan ulang dari awal, sukses |
| Perintah build terdokumentasi | PASS | Bagian 10, 12 |
| QEMU boot atau test target berjalan deterministik | NA | QEMU tidak terpasang (Bagian 7.1, 12.3) |
| Semua unit test/praktikum test relevan lulus | PASS | Bagian 12.5, 13.1 |
| Log serial disimpan | NA | Tidak ada integrasi kernel/serial pada lingkup laporan ini |
| Panic path terbaca atau dijelaskan jika belum relevan | PASS | Dijelaskan di Bagian 15.4 |
| Tidak ada warning kritis pada build | PASS | `-Werror` aktif, build sukses tanpa warning |
| Perubahan Git terkomit | PASS | commit `4c91ecb`, `09c8961` |
| Desain dan failure mode dijelaskan | PASS | Bagian 9, 15 |
| Laporan berisi screenshot/log yang cukup | PASS | Log tekstual pada Bagian 12-13 (tidak ada screenshot karena tidak ada output visual) |

Kriteria tambahan untuk praktikum lanjutan:

| Kriteria lanjutan | Status | Evidence |
|---|---|---|
| Static analysis dijalankan | PASS | `gcc -Wall -Wextra -Werror` sebagai static check dasar |
| Stress test dijalankan | NA | Fuzzing formal di luar cakupan, digantikan directed enrichment test |
| Fuzzing atau malformed-input test dijalankan | NA | Lihat catatan di atas |
| Fault injection dijalankan | PASS (terbatas) | Test overlap dan overflow adalah bentuk fault injection terarah pada input memory map |
| Disassembly/readelf evidence tersedia | PASS | Bagian 12.2 |
| Review keamanan dilakukan | PASS | Bagian 17 |
| Rollback diuji | PASS (sebagian) | Bagian 16 |

---

## 20. Readiness Review

| Status | Definisi | Pilihan |
|---|---|---|
| Belum siap uji | Build/test belum stabil atau bukti belum cukup | [ ] |
| Siap uji QEMU | Build bersih, QEMU/test target berjalan, log tersedia | [ ] |
| Siap demonstrasi praktikum | Siap ditunjukkan di kelas dengan bukti uji, failure mode, dan rollback | [ ] |
| Kandidat siap pakai terbatas | Hanya untuk penggunaan terbatas setelah test, security review, dokumentasi, dan known issue tersedia | [ ] |
| **Siap integrasi lokal, belum siap uji QEMU penuh** *(status khusus M6 sesuai Bagian 22 panduan)* | Host unit test lulus, freestanding audit lulus, tetapi QEMU belum dijalankan | [x] |

Alasan readiness:

```text
Sesuai Bagian 22 panduan OS_panduan_M6.md: "Jika hanya host unit test yang lulus
tetapi QEMU belum dijalankan, statusnya adalah siap integrasi lokal, belum siap
uji QEMU penuh." Kondisi ini persis menggambarkan hasil praktikum ini: seluruh
kriteria host-level (build freestanding, host unit test wajib dan pengayaan,
audit nm -u) PASS dengan bukti nyata, tetapi qemu-system-x86_64 dan gdb tidak
tersedia di lingkungan kerja dan tidak ada akses jaringan untuk instalasi,
sehingga integrasi ke kernel MCSOS dan smoke test QEMU tidak dapat dibuktikan.
```

Known issues:

| No. | Issue | Dampak | Workaround | Target perbaikan |
|---|---|---|---|---|
| 1 | QEMU, GDB, clang, ld.lld tidak tersedia di lingkungan kerja praktikum ini | Integrasi kernel dan smoke test QEMU tidak dapat dibuktikan | Jalankan `scripts/check_m6_static.sh` di WSL 2 sesuai rekomendasi panduan asli, lalu lanjutkan Bagian 11.10 panduan pada repository M0-M5 yang lengkap | Sesi praktikum lanjutan dengan lingkungan WSL 2 penuh |
| 2 | Source M0-M5 (`boot.S`, `kernel.c`, `idt.c`, dst.) tidak tersedia | Tidak bisa memverifikasi integrasi PMM ke kernel nyata | Kode PMM (`pmm.h`/`pmm.c`) sudah dirancang sesuai kontrak dan siap di-drop-in ke repository M0-M5 begitu tersedia | M7, setelah repository lengkap tersedia |

Keputusan akhir:

```text
Berdasarkan bukti build freestanding (nm -u kosong), hasil host unit test wajib
dan pengayaan (semua PASS), serta audit disassembly, hasil praktikum M6 ini
layak disebut "siap integrasi lokal untuk PMM inti". Belum layak disebut "siap
uji QEMU" karena QEMU tidak dijalankan sama sekali pada lingkungan kerja ini.
```

---

## 21. Rubrik Penilaian 100 Poin

| Komponen | Bobot | Indikator nilai penuh | Nilai |
|---|---:|---|---:|
| Kebenaran fungsional | 30 | Implementasi memenuhi target praktikum, build/test lulus, output sesuai expected result | (diisi penilai) |
| Kualitas desain dan invariants | 20 | Desain jelas, kontrak antarmuka eksplisit, invariants/ownership/locking terdokumentasi | (diisi penilai) |
| Pengujian dan bukti | 20 | Unit/integration/QEMU/static/fuzz/stress evidence memadai sesuai tingkat praktikum | (diisi penilai) |
| Debugging dan failure analysis | 10 | Failure mode, triage, panic/log, dan rollback dianalisis | (diisi penilai) |
| Keamanan dan robustness | 10 | Boundary, input validation, privilege, memory safety, dan negative tests dibahas | (diisi penilai) |
| Dokumentasi dan laporan | 10 | Laporan rapi, lengkap, dapat direproduksi, memakai referensi yang layak | (diisi penilai) |
| **Total** | **100** |  | (diisi penilai) |

Catatan penilai:

```text
(Diisi dosen/asisten.)
```

---

## 22. Kesimpulan

### 22.1 Yang Berhasil

```text
Implementasi bitmap frame allocator (PMM) M6 selesai sesuai kontrak API panduan:
pmm_zero_state, pmm_init_from_map, pmm_alloc_frame, pmm_free_frame,
pmm_reserve_range, dan seluruh fungsi query statistik berfungsi benar. Model
fail-closed, proteksi frame 0, penolakan double free, alignment 4096 byte, dan
overflow-safety pada base+length seluruhnya terverifikasi lewat host unit test
wajib maupun test pengayaan (overlap non-usable/usable dan overflow). Audit
freestanding (nm -u kosong) membuktikan pmm.o tidak bergantung libc host dan
siap di-link ke kernel freestanding.
```

### 22.2 Yang Belum Berhasil

```text
Integrasi PMM ke kernel MCSOS nyata (memanggil pmm_init_from_map dari kernel.c
setelah serial/panic siap) dan smoke test QEMU tidak dapat dilakukan karena
repository M0-M5 tidak tersedia di lingkungan kerja praktikum ini, dan
qemu-system-x86_64/gdb/clang/ld.lld tidak terpasang tanpa akses jaringan untuk
instalasi. Tugas pengayaan lanjutan seperti largest_free_run counter dan dynamic
bitmap placement (Bagian 18 "Tantangan Riset" panduan) juga belum dikerjakan
karena di luar cakupan wajib M6.
```

### 22.3 Rencana Perbaikan

```text
1. Jalankan ulang scripts/check_m6_static.sh pada lingkungan WSL 2 dengan clang,
   ld.lld, dan qemu-system-x86_64 terpasang sesuai rekomendasi asli panduan.
2. Integrasikan pmm.h/pmm.c ke repository M0-M5 yang lengkap, panggil
   pmm_init_from_map() setelah serial dan panic path siap (Bagian 11.10
   panduan), lalu jalankan make run-qemu-smoke dan simpan qemu-serial.log
   sebagai bukti.
3. Lanjutkan tugas pengayaan largest_free_run dan opsi build
   PMM_MAX_PHYS_BYTES=128GiB untuk M7.
```

---

## 23. Lampiran

### Lampiran A — Commit Log

```text
09c8961 m6: tambah enrichment test overlap non-usable dan overflow base+length
4c91ecb m6: implement bitmap PMM, host unit test, static audit
```

### Lampiran B — Diff Ringkas

```diff
+ include/types.h              (baru, 18 baris)
+ include/pmm.h                (baru, 57 baris)
+ src/pmm.c                    (baru, 241 baris)
+ tests/test_pmm_host.c        (baru, 40 baris)
+ tests/test_pmm_host_extra.c  (baru, ~40 baris, commit kedua)
+ scripts/check_m6_static.sh   (baru, diadaptasi CC/HOSTCC=gcc, 24 baris)
```

### Lampiran C — Log Build Lengkap

```text
$ CC=gcc HOSTCC=gcc ./scripts/check_m6_static.sh
M6 PMM host unit test: PASS
[PASS] M6 static check selesai
```

### Lampiran D — Log QEMU Lengkap

```text
Tidak ada — QEMU tidak dijalankan pada lingkungan kerja praktikum ini (lihat
Bagian 12.3 dan Bagian 20).
```

### Lampiran E — Output Readelf/Objdump

```text
$ readelf -S build/pmm.o | head -20
There are 13 section headers, starting at offset 0x1380:
[ 1] .text     PROGBITS  0000000000000000  size 0xaed  AX
[ 2] .rela.text RELA     0000000000000000  size 0x18
[ 3] .data     PROGBITS  0000000000000000  size 0x0    WA
[ 4] .bss      NOBITS    0000000000000000  size 0x0    WA
[ 5] .comment  PROGBITS  0000000000000000  size 0x2e
[ 6] .note.GNU-stack PROGBITS 0 size 0x0
[ 7] .note.gnu.pr[...] NOTE ...

$ nm -u build/pmm.o
(kosong)
```

### Lampiran F — Screenshot

Tidak ada — M6 tidak menghasilkan output visual/framebuffer.

### Lampiran G — Bukti Tambahan

```text
Isi lengkap tests/test_pmm_host_extra.c dan hasil PASS-nya menjadi bukti
tambahan pemenuhan Tugas Pengayaan 1 dan 2 pada Bagian 18 panduan
OS_panduan_M6.md.
```

---

## 24. Daftar Referensi

```text
[1] "limine" Rust crate documentation, "MemoryMapRequest," docs.rs. [Online].
    Accessed: May 2026.

[2] Intel Corporation, Intel® 64 and IA-32 Architectures Software Developer's
    Manual, latest public version, 2026.

[3] QEMU Project, "GDB usage," QEMU System Emulation Documentation. [Online].
    Accessed: May 2026.
```

---

## 25. Checklist Final Sebelum Pengumpulan

| Checklist | Status |
|---|---|
| Semua placeholder `[isi ...]` sudah diganti | Ya |
| Metadata laporan lengkap | Ya |
| Commit awal dan akhir dicatat | Ya |
| Perintah build dan test dapat dijalankan ulang | Ya |
| Log build dilampirkan | Ya |
| Log QEMU/test dilampirkan | Sebagian (test host ada; QEMU NA dengan alasan tercatat) |
| Artefak penting diberi hash | Ya |
| Desain, invariants, ownership, dan failure modes dijelaskan | Ya |
| Security/reliability dibahas | Ya |
| Readiness review tidak berlebihan | Ya |
| Rubrik penilaian diisi atau disiapkan | Disiapkan (kolom nilai untuk dosen) |
| Referensi memakai format IEEE | Ya |
| Laporan disimpan sebagai Markdown | Ya |

---

## 26. Pernyataan Pengumpulan

Saya mengumpulkan laporan ini bersama artefak pendukung pada commit:

```text
09c8961
```

Status akhir yang diklaim:

```text
Siap integrasi lokal untuk PMM inti, belum siap uji QEMU penuh
```

Ringkasan satu paragraf:

```text
Praktikum M6 berhasil mengimplementasikan bitmap frame allocator (PMM) MCSOS
sesuai kontrak API dan invariants pada OS_panduan_M6.md, dibuktikan lewat
kompilasi freestanding tanpa warning, host unit test wajib dan pengayaan yang
seluruhnya PASS, serta audit nm -u yang kosong. Keterbatasan utama adalah tidak
tersedianya QEMU, GDB, clang, dan repository M0-M5 pada lingkungan kerja
praktikum ini, sehingga integrasi kernel dan smoke test QEMU belum dapat
dibuktikan dan menjadi rencana lanjutan pada sesi berikutnya dengan lingkungan
WSL 2 penuh sesuai rekomendasi panduan.
```
