# Dokumentasi Lengkap Proyek SIJEMPOL / SILAPONTI-KAMTIB

**Untuk: pemilik baru proyek (serah terima dari pengembang sebelumnya)**
**Ditulis dengan asumsi pembaca belum pernah membaca kode program sama sekali.**

Tanggal dokumen: 24 Agustus 2026

---

## DAFTAR ISI

1. Ringkasan Satu Paragraf
2. Nama Aplikasi dan Kenapa Ada Dua Nama
3. Untuk Apa Aplikasi Ini Dipakai
4. Siapa Saja yang Memakai
5. Kamus Istilah (Wajib Dibaca Dulu)
6. Bahasa Pemrograman dan Teknologi yang Dipakai
7. Peta Tiga Proyek dan Cara Mereka Saling Bicara
8. Anatomi Proyek 1 — sijempol-be (Otak / Server)
9. Anatomi Proyek 2 — sijempol-fe (Website Admin)
10. Anatomi Proyek 3 — sijempol-scan (Kiosk Scan QR)
11. Database: Tabel demi Tabel
12. ALUR A — Login dan Sesi
13. ALUR B — Data Petugas
14. ALUR C — Jenis Piket dan Regu
15. ALUR D — Menyusun Jadwal Piket (Matriks Excel)
16. ALUR E — Aktivasi Jadwal (jantung sistem)
17. ALUR F — Pengingat WhatsApp
18. ALUR G — Konfirmasi Kesiapan H-1
19. ALUR H — Presensi Scan QR (inti yang Anda tanyakan)
20. ALUR I — Presensi Manual dan Koreksi
21. ALUR J — Laporan Pengawalan + Cetak PDF
22. ALUR K — Laporan Penggeledahan + Upload Foto
23. ALUR L — Dashboard dan Mode TV
24. ALUR M — Laporan Rekap Bulanan
25. Hak Akses per Role
26. Cara Menjalankan di Komputer Anda
27. Cara Deploy ke Server (Docker)
28. Verifikasi "95% Selesai" — Apa yang Benar-Benar Kurang
29. Temuan Teknis / Risiko yang Perlu Anda Tahu
30. Rencana Kerja yang Saya Sarankan
31. Lampiran: Daftar Lengkap Endpoint API
32. Lampiran: Peta File Penting

---

## 1. Ringkasan Satu Paragraf

SIJEMPOL adalah **sistem penjadwalan piket + presensi + pelaporan** untuk
**Lembaga Pemasyarakatan (Lapas) Kelas IIA Pontianak**, di bawah Kementerian
Imigrasi dan Pemasyarakatan RI. Admin/Operator memakai sebuah **website** untuk
menyusun jadwal piket petugas dalam bentuk tabel mirip Excel. Begitu jadwal
"diaktifkan", sistem otomatis **mengirim pengingat lewat WhatsApp** ke setiap
petugas, dan petugas bisa **membalas WhatsApp** dengan "SIAP" atau
"BERHALANGAN" — balasan itu langsung tercatat di sistem. Pada hari-H, petugas
tinggal **menempelkan kartu QR / NIP ke mesin scan** yang berdiri di pos, dan
kehadirannya otomatis tercatat. Selain itu ada modul **laporan pengawalan WBP**
dan **laporan penggeledahan blok hunian** yang bisa dicetak jadi PDF berkop
surat resmi. Semuanya terdiri dari **tiga proyek terpisah** yang harus jalan
bersamaan.

---

## 2. Nama Aplikasi dan Kenapa Ada Dua Nama

Anda akan menemukan **dua nama** di dalam kode. Ini normal, bukan bug:

| Nama | Artinya | Dipakai di mana |
| --- | --- | --- |
| **SIJEMPOL** | Nama internal/teknis (nama folder, nama database, nama Docker image, nama repositori GitHub) | `sijempol-be`, `sijempol-fe`, `sijempol-scan`, `APP_NAME=SIJEMPOL` |
| **SILAPONTI - KAMTIB** | Nama yang **dilihat pengguna** di layar | Judul di sidebar, judul di kiosk scan, pesan error |

Kepanjangan yang tertulis di layar kiosk: *"Sistem Informasi Jadwal dan
Monitoring Piket"*. "SILAPONTI" kemungkinan singkatan dari *Sistem Informasi
Lapas Pontianak*, dan "KAMTIB" = *Keamanan dan Ketertiban* — nama seksi di
Lapas yang mengurusi piket, pengawalan, dan penggeledahan.

Bukti perubahan nama ada di riwayat git proyek scan:
`82e7363 chore: rebrand application from SIJEMPOL to SILAPONTI - KAMTIB`.

> **Catatan untuk Anda:** rebranding ini **belum tuntas**. Di dalam kode backend
> masih ada teks yang tersimpan ke database berbunyi "Presensi otomatis melalui
> SIJEMPOL Scan." (`sijempol-be/app/Models/KehadiranPiket.php:152`). Kalau
> instansi mau konsisten memakai SILAPONTI, teks ini perlu diganti.

---

## 3. Untuk Apa Aplikasi Ini Dipakai

Bayangkan sebuah Lapas. Setiap hari harus ada petugas jaga: jaga menara, jaga
pintu utama (P2U), perwira piket, regu pengamanan, dan seterusnya. Dulu
jadwalnya dibuat di Excel, ditempel di papan, lalu petugas diingatkan via grup
WhatsApp secara manual, dan absennya pakai buku tulis.

Aplikasi ini menggantikan semua itu dengan **7 pekerjaan utama**:

1. **Menyimpan data petugas** — NIP, nama, jabatan, unit kerja, nomor WhatsApp,
   status (Aktif / Cuti / Dinas Luar).
2. **Menyusun jadwal piket bulanan** dalam bentuk matriks (baris = petugas,
   kolom = tanggal), lengkap dengan kode shift (P = pagi, S = siang, M = malam,
   L = libur, dst).
3. **Mengirim pengingat WhatsApp otomatis** — H-1 hari dan H-1 jam sebelum
   tugas dimulai.
4. **Menerima konfirmasi kesiapan** — petugas balas WA "SIAP" atau
   "BERHALANGAN sakit", sistem membacanya sendiri.
5. **Mencatat kehadiran hari-H** — lewat scan QR/barcode di pos jaga, atau
   diinput manual oleh operator.
6. **Membuat laporan resmi** — Laporan Pengawalan WBP (mengantar narapidana ke
   sidang/rumah sakit) dan Laporan Penggeledahan (razia blok hunian, lengkap
   dengan foto barang sitaan).
7. **Menampilkan dashboard** ringkasan harian, termasuk **mode TV** untuk
   dipajang di layar besar di ruang kontrol.

---

## 4. Siapa Saja yang Memakai

Ada **3 jenis akun** (role) di sistem, ditambah petugas lapangan yang tidak
punya akun sama sekali.

| Role | Siapa | Bisa apa |
| --- | --- | --- |
| **Admin** | Kepala/penanggung jawab sistem | Semua. Termasuk membuat akun pengguna, menghapus data, mengubah jam notifikasi, mengubah template surat |
| **Operator** | Staf KAMTIB sehari-hari | Semua operasional (jadwal, presensi, laporan) **kecuali** menghapus data & menu Pengaturan |
| **TV** | Akun khusus layar monitor | **Hanya** dashboard. Tidak ada sidebar, tidak bisa buka menu lain |
| *(petugas piket)* | Anggota regu di lapangan | **Tidak punya akun.** Mereka hanya menerima WA, membalas WA, dan scan QR |

Poin penting yang sering disalahpahami: **petugas piket tidak login ke mana
pun.** Mereka tidak punya aplikasi HP. Interaksi mereka cuma dua: (a) balas
pesan WhatsApp, (b) tempelkan kartu ke mesin scan. Ini keputusan desain yang
tepat untuk lingkungan Lapas.

Definisi role ada di `sijempol-be/app/Enums/UserRole.php`:

```php
enum UserRole: string
{
    case Admin = 'Admin';
    case Operator = 'Operator';
    case TV = 'TV';
}
```

`enum` artinya "daftar pilihan yang sah". Kalau ada kode yang mencoba menyimpan
role `"Kepala"`, program langsung menolak — bukan menyimpan data ngawur.

---

## 5. Kamus Istilah (Wajib Dibaca Dulu)

Kode ini ditulis campur Indonesia-Inggris. Kalau Anda tidak paham istilahnya,
kode akan terlihat jauh lebih rumit dari sebenarnya.

### 5.1 Istilah domain (dunia Lapas)

| Istilah | Arti |
| --- | --- |
| **Piket** | Tugas jaga bergiliran |
| **Jenis Piket** | Kategori piket, misalnya "Pengawas Piket", "Regu Pengamanan", "P2U". Di kode, tabel inilah yang **juga** menyimpan periode & status jadwal |
| **Regu** | Kelompok petugas yang bertugas bersama (Regu I, Regu II, ...) |
| **Petugas** | Pegawai yang dijadwalkan piket |
| **Shift** | Giliran waktu: Pagi / Siang / Malam |
| **P2U** | Pintu Pengamanan Utama — pos gerbang depan Lapas |
| **KPLP** | Kepala Pengamanan Lembaga Pemasyarakatan |
| **WBP** | Warga Binaan Pemasyarakatan (istilah resmi untuk narapidana) |
| **Pengawalan** | Mengawal WBP keluar Lapas (sidang, berobat, pemindahan) |
| **Penggeledahan** | Razia/pemeriksaan blok & kamar hunian |
| **Blok / Kamar Hunian** | Lokasi sel di dalam Lapas |
| **Barang Sitaan** | Barang terlarang yang ditemukan saat razia |
| **Surat Perintah** | Dokumen resmi dasar pengawalan |
| **Roster** | Urutan/susunan petugas di dalam matriks jadwal |
| **H-1** | Satu hari sebelum hari tugas |

### 5.2 Istilah teknis (dunia programmer)

| Istilah | Penjelasan untuk orang awam |
| --- | --- |
| **Backend / BE** | "Otak" aplikasi. Program yang jalan di server, menyimpan data, menghitung, dan mengatur aturan. Tidak punya tampilan |
| **Frontend / FE** | "Wajah" aplikasi. Yang Anda lihat & klik di browser |
| **API** | "Loket" tempat frontend meminta data ke backend. Contoh: frontend bertanya `GET /api/v1/petugas`, backend menjawab daftar petugas |
| **Endpoint** | Satu alamat loket. Misal `/api/v1/auth/login` |
| **JSON** | Format teks untuk bertukar data. Bentuknya `{"nama": "Budi", "nip": "123"}` |
| **Database** | Gudang data permanen. Di sini pakai PostgreSQL |
| **Tabel** | Satu "sheet Excel" di dalam database. Misal tabel `petugas` |
| **Kolom / Field** | Satu jenis informasi di tabel. Misal kolom `nip` |
| **Baris / Record / Row** | Satu data. Misal satu orang petugas |
| **Migration** | Resep untuk membuat/mengubah tabel. Dijalankan sekali, tersimpan riwayatnya |
| **Model** | Perwakilan satu tabel di dalam kode. `Petugas.php` mewakili tabel `petugas` |
| **Controller** | Petugas loket. Menerima permintaan, menyuruh Model bekerja, mengembalikan jawaban |
| **Service** | Manajer. Dipakai untuk pekerjaan rumit yang butuh banyak langkah |
| **Middleware** | Satpam di depan pintu. Mengecek "boleh masuk atau tidak" sebelum permintaan sampai ke Controller |
| **Route** | Daftar alamat + siapa yang menanganinya |
| **Form Request** | Formulir pemeriksa. Memvalidasi isian sebelum masuk ke Controller |
| **Resource** | Penyaring jawaban. Menentukan field mana yang boleh dikirim keluar (password tidak pernah keluar) |
| **Queue / Antrean** | Daftar pekerjaan yang dikerjakan belakangan, di latar belakang. Supaya pengguna tidak menunggu |
| **Job** | Satu pekerjaan di dalam antrean. Contoh: "kirim WA ke Pak Budi" |
| **Worker** | Program yang jalan terus-menerus mengambil Job dari antrean lalu mengerjakannya |
| **Token** | Kartu identitas digital. Setelah login, frontend menyimpan token dan menempelkannya di setiap permintaan |
| **Transaction (DB)** | "Semua berhasil atau semua batal." Mencegah data setengah jadi |
| **Lock** | Kunci sementara pada baris data supaya dua orang tidak mengubahnya bersamaan |
| **Webhook** | Kebalikan API. Layanan luar yang "menelepon" backend kita saat ada kejadian |
| **Enum** | Daftar pilihan sah dan terbatas. Misal status kehadiran hanya boleh Hadir/Sakit/Izin/Alpa |
| **UUID** | Nomor identitas acak panjang, contoh `9b1d...-c3f2`. Dipakai supaya ID tidak bisa ditebak |
| **Idempotent** | Dikerjakan sekali atau seratus kali, hasilnya sama. Penting supaya scan dua kali tidak jadi dua absen |
| **Repository (git)** | Folder proyek yang riwayat perubahannya dicatat |
| **Hook (React)** | Fungsi khusus React yang namanya diawali `use...`, tempat menyimpan data & logika sebuah halaman |

---

## 6. Bahasa Pemrograman dan Teknologi yang Dipakai

### 6.1 Ringkas

| Proyek | Bahasa | Framework utama | Port default |
| --- | --- | --- | --- |
| `sijempol-be` | **PHP 8.3+** | **Laravel 13** | 8000 |
| `sijempol-fe` | **TypeScript** (turunan JavaScript) | **React 19** + Vite 6 + Tailwind CSS 4 | 3000 |
| `sijempol-scan` | **TypeScript** | **React 19** + Vite 8 | 3005 |

### 6.2 Rinci — Backend (`sijempol-be`)

Sumber: `sijempol-be/composer.json`

| Paket | Versi | Untuk apa |
| --- | --- | --- |
| `php` | ^8.3 | Bahasa pemrogramannya |
| `laravel/framework` | ^13.8 | Kerangka kerja utama: routing, database, queue, validasi |
| `laravel/sanctum` | ^4.3 | Sistem token login |
| `barryvdh/laravel-dompdf` | ^3.1 | **Mengubah halaman HTML jadi file PDF** — untuk cetak jadwal & surat pengawalan |
| `livewire/livewire`, `livewire/flux` | ^4.3 / ^2.15 | Terpasang tapi **praktis tidak dipakai** — sisa template Laravel |
| `pestphp/pest` | ^4.7 | Alat pengujian otomatis (test) |
| `laravel/pint` | ^1.27 | Perapi format kode |

- Database: **PostgreSQL** (`.env.example` → `DB_CONNECTION=pgsql`).
- Zona waktu: **Asia/Pontianak** (`APP_TIMEZONE=Asia/Pontianak`). Ini penting —
  semua perhitungan "hari ini" mengikuti waktu Pontianak, bukan UTC.
- Queue: **database** (`QUEUE_CONNECTION=database`), artinya antrean pekerjaan
  disimpan di tabel `jobs`, bukan Redis.

### 6.3 Rinci — Frontend Admin (`sijempol-fe`)

Sumber: `sijempol-fe/package.json`

| Paket | Untuk apa |
| --- | --- |
| `react` 19 + `react-dom` | Kerangka pembuat tampilan |
| `react-router-dom` 7 | Mengatur perpindahan halaman tanpa reload |
| `vite` 6 | Alat build & server pengembangan |
| `tailwindcss` 4 | Sistem styling (warna, jarak, ukuran) langsung di dalam HTML |
| `lucide-react` | Kumpulan ikon |
| `@dnd-kit/*` | Drag & drop (menyeret baris petugas di matriks jadwal) |
| `jspreadsheet-ce` | Komponen tabel mirip Excel |
| `motion` | Animasi |
| `express` | Server kecil untuk mode produksi (opsional) |
| `@google/genai` | **Terpasang tapi tidak dipakai** — sisa template Google AI Studio. Aman dihapus |

Catatan kecil: nama paket di `package.json` masih `"react-example"` — sisa
template yang belum diganti. Tidak memengaruhi jalannya aplikasi.

### 6.4 Rinci — Kiosk Scan (`sijempol-scan`)

| Paket | Untuk apa |
| --- | --- |
| `react` 19 | Tampilan |
| `html5-qrcode` ^2.3.8 | **Membaca QR lewat kamera laptop/HP** |
| `lucide-react` | Ikon |
| `vite` 8 | Build & server |
| `@vitejs/plugin-basic-ssl` | Menyiapkan HTTPS lokal (kamera browser wajib HTTPS kalau bukan localhost) |
| `oxlint` | Pemeriksa kualitas kode |

### 6.5 Layanan luar

| Layanan | Untuk apa |
| --- | --- |
| **GOWA (Go-WhatsApp)** | Program terpisah (**bukan** bagian dari 3 proyek ini) yang benar-benar mengirim & menerima pesan WhatsApp. Backend menghubunginya lewat HTTP. Dikonfigurasi via `GOWA_URL`, `GOWA_DEVICE_ID`, `GOWA_USERNAME`, `GOWA_PASSWORD`, `GOWA_WEBHOOK_SECRET` |

> **PENTING:** GOWA **tidak ada di dalam ketiga folder ini.** Anda harus
> menjalankannya terpisah, lalu memasang (pairing) nomor WhatsApp instansi ke
> situ. Tanpa GOWA hidup, fitur pengingat & konfirmasi otomatis mati total —
> backend akan memakai kelas pengganti `UnavailableWhatsAppSender`, lihat
> `sijempol-be/app/Providers/AppServiceProvider.php:27-42`.

---

## 7. Peta Tiga Proyek dan Cara Mereka Saling Bicara

```
   [ Admin / Operator ]                     [ Petugas di pos jaga ]
   buka browser di PC                       tempel kartu QR / NIP
            |                                          |
            v                                          v
  +---------------------+                   +----------------------+
  |    sijempol-fe      |                   |    sijempol-scan     |
  |  React, port 3000   |                   |  React, port 3005    |
  |  Website admin      |                   |  Kiosk layar penuh   |
  +----------+----------+                   +----------+-----------+
             |                                         |
             | HTTP + JSON (pakai token login)         | HTTP + JSON (TANPA login)
             | ke /api/v1/...                          | ke /api/v1/kehadiran/scan
             |                                         |
             +--------------------+--------------------+
                                  |
                                  v
                   +------------------------------+
                   |        sijempol-be           |
                   |    Laravel 13, port 8000     |
                   |    "otak" semua aturan       |
                   +---+--------------+-----------+
                       |              |
        +--------------+              +-----------------+
        v                                               v
+----------------+                          +------------------------+
|  PostgreSQL    |                          |   Antrean (queue) di   |
|  semua data    |                          |   tabel `jobs`         |
+----------------+                          +-----------+------------+
                                                        |
                                                        v
                                            +------------------------+
                                            |  Worker:               |
                                            |  php artisan queue:work|
                                            +-----------+------------+
                                                        |
                                                        v
                                            +------------------------+
                                            |   GOWA (Go-WhatsApp)   | --> HP Petugas
                                            |   layanan terpisah     | <-- balasan "SIAP"
                                            +-----------+------------+
                                                        | webhook
                                                        v
                                            /api/v1/webhooks/gowa/reports
```

**Cara membaca diagram ini:**

- Ketiga kotak atas adalah **program terpisah** yang bisa dijalankan di komputer
  berbeda. Mereka **tidak saling memanggil langsung**; semuanya lewat backend.
- Backend adalah **satu-satunya** yang boleh menyentuh database. Ini disebut
  arsitektur *monolith + API* — sederhana dan mudah dirawat.
- Panah ke GOWA searah untuk mengirim; GOWA "menelepon balik" backend lewat
  **webhook** saat ada balasan masuk dari petugas.

---

## 8. Anatomi Proyek 1 — `sijempol-be` (Otak / Server)

### 8.1 Struktur folder & artinya

```
sijempol-be/
├── app/                        <- SEMUA kode buatan sendiri ada di sini
│   ├── Console/Commands/       <- Perintah yang dijalankan lewat terminal
│   ├── Contracts/              <- "Janji"/kontrak: WhatsAppSender.php
│   ├── Data/                   <- Objek pembawa data sederhana
│   ├── Enums/                  <- Daftar pilihan sah (status, role, shift)
│   ├── Exceptions/             <- Jenis-jenis error khusus aplikasi ini
│   ├── Http/
│   │   ├── Controllers/Api/    <- Petugas loket tiap modul
│   │   ├── Middleware/         <- Satpam (admin, active, tv-dashboard-only)
│   │   ├── Requests/           <- Formulir validasi tiap aksi
│   │   ├── Resources/          <- Penyaring bentuk jawaban JSON
│   │   └── Responses/          <- ApiResponse.php: pembungkus jawaban standar
│   ├── Jobs/                   <- Pekerjaan latar belakang (kirim WA)
│   ├── Models/                 <- Perwakilan tabel database
│   ├── Providers/              <- Pengaturan awal aplikasi (rate limit, dsb)
│   └── Services/               <- Manajer alur rumit (Jadwal, Dashboard, PDF)
├── bootstrap/app.php           <- Titik rakit aplikasi: middleware & error
├── config/                     <- File pengaturan
├── database/
│   ├── factories/              <- Pembuat data palsu untuk testing
│   ├── migrations/             <- Resep pembuatan tabel (33 file)
│   └── seeders/                <- Pengisi data awal / demo
├── resources/views/pdf/        <- Template HTML yang jadi PDF
├── routes/api.php              <- DAFTAR SEMUA ALAMAT API  ** penting **
├── storage/                    <- File upload (foto), log, cache
├── tests/                      <- Pengujian otomatis
├── .env.example                <- Contoh file konfigurasi rahasia
├── docker-compose.yml          <- Resep menjalankan di server
├── BACKEND_README.md           <- Rancangan awal produk (823 baris)
├── architecture.md             <- Ringkasan arsitektur (314 baris)
├── task-list.md                <- Checklist progres (267 baris)  ** penting **
└── AGENTS.md                   <- Aturan kerja untuk AI coding agent
```

### 8.2 Alur permintaan di backend — "jalur pipa" yang selalu sama

Setiap kali frontend meminta sesuatu, permintaan itu melewati jalur yang persis
sama. Pahami ini sekali, Anda paham seluruh backend:

```
Permintaan masuk
   |
   v
[1] routes/api.php          -> "alamat ini ditangani siapa?"
   |
   v
[2] Middleware              -> "sudah login? masih aktif? role-nya boleh?"
   |
   v
[3] Form Request            -> "isian formulirnya lengkap dan benar?"
   |
   v
[4] Controller              -> "oke, panggil Model/Service"
   |
   v
[5] Model / Service         -> baca-tulis database, hitung, atur transaksi
   |
   v
[6] API Resource            -> "field mana yang boleh dikirim keluar?"
   |
   v
[7] ApiResponse             -> dibungkus {success, message, data}
   |
   v
Jawaban keluar
```

**Contoh nyata**, mencatat presensi manual:

| Langkah | File |
| --- | --- |
| 1 | `routes/api.php:87` → `Route::post('/kehadiran', [KehadiranController::class, 'store'])` |
| 2 | `auth:sanctum`, `active`, `tv-dashboard-only` (didaftarkan di `bootstrap/app.php:33-37`) |
| 3 | `app/Http/Requests/Kehadiran/StoreKehadiranRequest.php` |
| 4 | `app/Http/Controllers/Api/KehadiranController.php:35-45` |
| 5 | `app/Models/KehadiranPiket.php:96-118` (`createFromPayload`) |
| 6 | `app/Http/Resources/KehadiranResource.php` |
| 7 | `app/Http/Responses/ApiResponse.php` |

### 8.3 Bentuk jawaban standar

Semua jawaban API punya bentuk yang sama. Ini memudahkan frontend.

Berhasil:
```json
{ "success": true, "message": "Data berhasil disimpan", "data": { } }
```

Gagal validasi:
```json
{
  "success": false,
  "message": "Validasi gagal.",
  "errors": { "nama": ["Nama wajib diisi"] }
}
```

Daftar berhalaman:
```json
{
  "success": true,
  "data": [ ],
  "meta": { "page": 1, "limit": 20, "total": 100, "totalPages": 5 }
}
```

### 8.4 Penanganan error terpusat

File `bootstrap/app.php:41-110` mengubah **semua** error jadi JSON yang rapi.
Contoh potongan:

```php
$exceptions->render(function (Throwable $exception, Request $request): ?JsonResponse {
    if (! $request->is('api/*')) {
        return null;
    }

    return ApiResponse::error('Terjadi kesalahan server.', status: 500);
});
```

Artinya: kalau terjadi error tak terduga, pengguna hanya melihat pesan
"Terjadi kesalahan server." — **bukan** detail teknis yang bisa membocorkan
struktur database. Ini praktik keamanan yang benar.

### 8.5 Middleware (satpam) yang ada

| Nama alias | File | Tugas |
| --- | --- | --- |
| `auth:sanctum` | bawaan Laravel | Wajib punya token login yang valid |
| `active` | `EnsureUserIsActive.php` | Tolak akun berstatus Nonaktif |
| `admin` | `EnsureUserIsAdmin.php` | Hanya role Admin yang lewat |
| `tv-dashboard-only` | `EnsureTvDashboardOnly.php` | Akun TV hanya boleh `/auth/me`, `/auth/logout`, `/dashboard/summary` |
| (selalu jalan) | `AddRequestCorrelationId.php` | Memberi nomor unik ke setiap permintaan supaya mudah dilacak di log |

Isi `EnsureTvDashboardOnly.php` — perhatikan betapa sederhananya:

```php
if (
    $request->user()?->role === UserRole::TV
    && ! $request->is(
        'api/v1/auth/me',
        'api/v1/auth/logout',
        'api/v1/dashboard/summary',
    )
) {
    return ApiResponse::error('Role TV hanya dapat mengakses dashboard.', status: 403);
}

return $next($request);
```

Baca sebagai kalimat: *"Kalau yang login adalah akun TV DAN alamat yang dituju
bukan salah satu dari tiga alamat ini, tolak dengan pesan 403. Kalau tidak,
teruskan."* `$next($request)` artinya "silakan lanjut ke satpam berikutnya".

### 8.6 Rate limit (pembatas percobaan)

Ada di `app/Providers/AppServiceProvider.php:92-113`:

| Nama | Batas | Kenapa |
| --- | --- | --- |
| `login` | 5x per menit per kombinasi username+IP | Mencegah penebakan password |
| `scan` | 60x per menit per IP | Mencegah kiosk membanjiri server |
| `whatsapp-send` | 1 pesan per 20 detik (dari `WHATSAPP_MESSAGE_INTERVAL_SECONDS`) | **Sangat penting** — WhatsApp akan memblokir nomor yang mengirim terlalu cepat |

---

## 9. Anatomi Proyek 2 — `sijempol-fe` (Website Admin)

### 9.1 Prinsip penataan: "Feature-based"

Proyek ini disusun **per fitur**, bukan per jenis file. Aturannya tertulis di
`sijempol-fe/src/features/ARCHITECTURE.md`. Setiap folder fitur berisi:

```
src/features/<nama-fitur>/
├── api/__request.ts      <- Semua panggilan API milik fitur ini
├── hooks/useXxx.ts       <- Otak halaman: data, loading, error, aksi
├── types/__models.ts     <- Bentuk data (TypeScript)
├── components/           <- Tampilan
├── lib/                  <- Fungsi bantu murni (kalau perlu)
└── index.ts              <- SATU-SATUNYA pintu keluar untuk fitur lain
```

**Aturan emas:** fitur A boleh mengimpor dari `features/B/index.ts`, tapi
**tidak boleh** langsung menusuk ke `features/B/components/Sesuatu.tsx`. Ini
menjaga supaya perubahan di dalam sebuah fitur tidak merusak fitur lain.

### 9.2 Daftar fitur yang ada

| Folder | Halaman | Baris kode terbesar |
| --- | --- | --- |
| `auth` | Login | `LoginView.tsx` (238) |
| `dashboard` | Dashboard | `DashboardView.tsx` (323) |
| `petugas` | Data Petugas | `PetugasView.tsx` (526) |
| `regu` | Manajemen Regu | `ReguView.tsx` (320) |
| `jadwal-piket` | Jadwal Piket (**fitur terbesar**) | `JadwalPiketView.tsx` (1.424) |
| `pengingat` | Pengingat WhatsApp | `PengingatView.tsx` (263) |
| `konfirmasi-petugas` | Konfirmasi Petugas | `KonfirmasiView.tsx` (195) |
| `kehadiran` | Kehadiran | `KehadiranView.tsx` (224) |
| `laporan-pengawalan` | Laporan Pengawalan | `LaporanPengawalanForm.tsx` (827) |
| `laporan-penggeledahan` | Laporan Penggeledahan | `LaporanPenggeledahanForm.tsx` (1.037) |
| `laporan-rekap` | Laporan Rekap | `LaporanRekapView.tsx` (37) |
| `manajemen-pengguna` | Manajemen Pengguna | `UserManagementView.tsx` (150) |
| `template-pengawalan` | Template Pengawalan | `TemplatePengawalanView.tsx` |
| `panduan` | Panduan (halaman statis) | `PanduanView.tsx` |

Total kode frontend: **~18.755 baris**.

### 9.3 Daftar alamat halaman

Ada di `src/app/routes.ts:1-18`:

```typescript
export const APP_PATHS = {
  login: '/login',
  dashboard: '/dashboard',
  petugas: '/petugas',
  jadwal: '/jadwal',
  regu: '/regu',
  reguPengamanan: '/regu/pengamanan',
  reguP2u: '/regu/p2u',
  pengingat: '/pengingat',
  konfirmasi: '/konfirmasi',
  kehadiran: '/kehadiran',
  laporanPengawalan: '/laporan/pengawalan',
  laporanPenggeledahan: '/laporan/penggeledahan',
  laporanRekap: '/laporan/rekap',
  users: '/users',
  templatePengawalan: '/template-pengawalan',
  panduan: '/panduan',
} as const;
```

`as const` di TypeScript artinya "nilai-nilai ini tetap, tidak boleh diubah
saat program jalan". Manfaatnya: kalau Anda salah ketik `APP_PATHS.jadwl`,
editor langsung memberi tanda merah — tidak menunggu sampai aplikasi rusak.

### 9.4 Penjaga halaman (guard)

Di `src/app/App.tsx:45-70` ada empat komponen penjaga:

```typescript
function RequireAuth() {
  const { user } = useAuth();
  const location = useLocation();
  return user ? <Outlet /> : <Navigate to={APP_PATHS.login} replace state={{ from: location }} />;
}
```

Baca sebagai: *"Ambil siapa yang sedang login. Kalau ada (`user` terisi),
tampilkan halaman yang diminta (`<Outlet />`). Kalau tidak ada, lempar ke
halaman login sambil mengingat halaman yang tadi mau dibuka
(`state={{ from: location }}`)"* — sehingga setelah login, pengguna
dikembalikan ke halaman yang tadi diinginkan.

Empat penjaga tersebut:

| Penjaga | Baris | Fungsi |
| --- | --- | --- |
| `RequireAuth` | `App.tsx:45` | Wajib login |
| `PublicOnly` | `App.tsx:51` | Kalau sudah login, jangan lihat halaman login lagi |
| `RequireAdmin` | `App.tsx:57` | Hanya Admin |
| `RequireInteractiveRole` | `App.tsx:64` | Akun TV dilempar balik ke dashboard |

> **Catatan keamanan penting:** penjaga di frontend ini **hanya kosmetik** —
> supaya menu tidak terlihat. Keamanan sebenarnya ada di backend
> (`middleware admin`, `tv-dashboard-only`). Ini benar dan memang begitu
> seharusnya. Jangan pernah mengandalkan frontend saja.

### 9.5 Jembatan ke backend: `src/shared/api/__request.ts`

Ini file **paling penting** di frontend (121 baris). Semua fitur memanggil
backend lewat sini.

**(a) Alamat backend dan kunci token**
```typescript
const API_BASE_URL = (import.meta.env?.VITE_API_BASE_URL || '/api/v1').replace(/\/$/, '');
const TOKEN_KEY = 'silaponti.auth.token';
```
`import.meta.env.VITE_API_BASE_URL` dibaca dari file `.env`. Kalau kosong,
dipakai `/api/v1`. `.replace(/\/$/, '')` menghapus garis miring di ujung supaya
tidak terjadi alamat ganda seperti `//petugas`.

**(b) Penyimpan token**
```typescript
export const session = {
  getToken: () => sessionStorage.getItem(TOKEN_KEY),
  setToken: (token: string) => sessionStorage.setItem(TOKEN_KEY, token),
  clear: () => sessionStorage.removeItem(TOKEN_KEY),
  hasToken: () => Boolean(sessionStorage.getItem(TOKEN_KEY)),
};
```
Perhatikan: **`sessionStorage`**, bukan `localStorage`. Bedanya: `sessionStorage`
otomatis terhapus saat tab browser ditutup. Ini pilihan sadar demi keamanan —
kalau petugas lupa logout di komputer bersama, tokennya hilang begitu tab
ditutup.

**(c) Menempelkan token ke setiap permintaan**
```typescript
if (options.authenticated !== false) {
  const token = session.getToken();
  if (token) headers.set('Authorization', `Bearer ${token}`);
}
```
Semua permintaan otomatis membawa token, **kecuali** yang secara khusus diberi
`authenticated: false` (yaitu login).

**(d) Menangani token kedaluwarsa**
```typescript
if (response.status === 401) {
  session.clear();
  window.dispatchEvent(new CustomEvent('silaponti:auth-expired'));
}
```
Kalau backend menjawab 401 ("tidak dikenali"), token dihapus dan seluruh
aplikasi "diberi tahu" lewat event. Yang mendengarkan event ini ada di
`useAuth.ts:47-55`, yang akan mengosongkan data user sehingga pengguna otomatis
kembali ke halaman login. Rapi.

**(e) Mengambil SEMUA halaman sekaligus**
```typescript
export async function getAll<T>(path: string, params = {}): Promise<T[]> {
  const items: T[] = [];
  let page = 1;
  let totalPages = 1;

  do {
    const response = await request<T[]>(`${path}${queryString({ ...params, page, limit: 100 })}`);
    items.push(...response.data);
    totalPages = response.meta?.totalPages ?? 1;
    page++;
  } while (page <= totalPages);

  return items;
}
```
Backend membatasi 100 data per permintaan. Fungsi ini memanggil berulang sampai
semua halaman terkumpul. Dipakai untuk mengisi dropdown pilihan petugas.

> **Risiko:** kalau petugas nanti ada 5.000 orang, fungsi ini akan memanggil
> backend 50 kali berturut-turut setiap halaman dibuka. Untuk skala Lapas
> (ratusan orang) masih aman.

---

## 10. Anatomi Proyek 3 — `sijempol-scan` (Kiosk Scan QR)

Proyek ini **sengaja dibuat kecil**: hanya 4 file kode, total ~484 baris.

```
sijempol-scan/src/
├── main.tsx                     (10 baris)  <- titik mulai
├── App.tsx                      (182 baris) <- seluruh logika kiosk
├── components/QRScanner.tsx     (144 baris) <- pembungkus kamera
└── index.css                    (148 baris) <- tampilan
```

### 10.1 Kenapa dipisah dari website admin?

Empat alasan, semuanya masuk akal:

1. **Beda perangkat.** Kiosk ditempel di pos jaga; website admin ada di meja staf.
2. **Beda kebutuhan login.** Kiosk **tidak login sama sekali** — tidak boleh ada
   token admin nongkrong di komputer pos jaga yang bisa diakses siapa saja.
3. **Beda mode tampilan.** Kiosk layar penuh, tombol besar, tanpa menu.
4. **Ringan.** Tidak perlu memuat 18 ribu baris kode admin hanya untuk scan.

### 10.2 Dua cara input yang didukung

Ini detail yang sering terlewat, padahal penting untuk operasional:

| Cara | Perangkat | Cara kerja teknis |
| --- | --- | --- |
| **Kamera** | Laptop/HP/webcam | Library `html5-qrcode` membaca gambar QR |
| **Scanner USB** | Alat scan barcode/QR yang dicolok USB | Alat ini **berperilaku seperti keyboard**: dia "mengetik" NIP lalu menekan Enter |

Cara kedua ditangani oleh kode "penyadap ketikan" di `App.tsx:57-85`. Ini
bagian paling menarik dari proyek scan — mari kita bedah baris per baris.

```typescript
useEffect(() => {
  let barcodeBuffer = '';
  let lastKeyTime = Date.now();

  const handleKeyDown = (event: KeyboardEvent) => {
    // (1) Kalau kursor sedang di dalam kotak isian, jangan ganggu
    if (event.target instanceof HTMLInputElement || ...) {
      return;
    }

    const currentTime = Date.now();

    // (2) Kalau jeda antar tombol > 50 milidetik, berarti ini manusia mengetik
    //     -> buang isi buffer
    if (currentTime - lastKeyTime > 50) {
      barcodeBuffer = '';
    }

    // (3) Kalau Enter ditekan, berarti scanner selesai -> kirim
    if (event.key === 'Enter') {
      if (barcodeBuffer) {
        void handleScanSuccess(barcodeBuffer);
        barcodeBuffer = '';
      }
    } else if (event.key.length === 1) {
      // (4) Karakter biasa -> tumpuk ke buffer
      barcodeBuffer += event.key;
    }
    lastKeyTime = currentTime;
  };

  window.addEventListener('keydown', handleKeyDown);
  return () => window.removeEventListener('keydown', handleKeyDown);
}, []);
```

**Penjelasan awam:** scanner USB "mengetik" NIP dengan kecepatan sangat tinggi
(kurang dari 50 milidetik antar huruf), sedangkan manusia jauh lebih lambat.
Kode ini memakai selisih waktu itu untuk membedakan mana ketikan scanner dan
mana ketikan orang. Kalau ketikan cepat lalu diakhiri Enter → itu hasil scan,
langsung dikirim ke server.

Baris terakhir `return () => window.removeEventListener(...)` adalah
"bersih-bersih": ketika komponen ditutup, penyadap dilepas supaya tidak
menumpuk di memori.

### 10.3 Pencegah scan ganda

`App.tsx:33-37`:

```typescript
const latestScan = latestScanRef.current;
if (latestScan?.text === nip && newItem.timestamp.getTime() - latestScan.timestamp < 3000) {
  return;
}
latestScanRef.current = { text: nip, timestamp: newItem.timestamp.getTime() };
```

Artinya: *"Kalau NIP yang baru saja di-scan sama dengan NIP sebelumnya DAN
jaraknya kurang dari 3 detik (3000 milidetik), abaikan."* Ini mencegah satu
kartu terbaca 5 kali karena kamera membaca 10 frame per detik.

Perhatikan bahwa **pengamanan ini ada di DUA tempat**: di sini (frontend, cepat)
dan di backend (`KehadiranPiket::recordScanByNip`, permanen). Prinsip ini
disebut *defense in depth* — kalau salah satu jebol, yang lain menahan.

---

## 11. Database: Tabel demi Tabel

Semua tabel dibuat oleh file di `sijempol-be/database/migrations/` (33 file).
ID tabel domain memakai **UUID** (nomor acak panjang), bukan angka berurutan —
supaya orang luar tidak bisa menebak `/petugas/2` untuk mengintip data.

### 11.1 `users` — akun yang bisa login

`0001_01_01_000000_create_users_table.php`

| Kolom | Isi |
| --- | --- |
| `id` | Angka berurutan (satu-satunya tabel yang begini) |
| `nama` | Nama lengkap |
| `username` | Nama login, **unik** |
| `password` | Kata sandi yang sudah di-hash (diacak satu arah, tidak bisa dibalik) |
| `role` | `Admin` / `Operator` / `TV` |
| `status` | `Aktif` / `Nonaktif` |

### 11.2 `petugas` — pegawai yang dijadwalkan

`2026_08_09_000002_create_petugas_table.php`

| Kolom | Isi | Catatan |
| --- | --- | --- |
| `id` | UUID | |
| `nip` | Nomor Induk Pegawai | **Unik.** Inilah yang dicetak jadi QR |
| `nama` | Nama lengkap | |
| `jabatan` | Jabatan | |
| `unit_kerja` | Unit kerja | |
| `whatsapp` | Nomor WA | Boleh kosong, tapi kalau kosong pengingat gagal |
| `status` | `Aktif` / `Cuti` / `Dinas Luar` | Ada index untuk pencarian cepat |
| `regu_id` | Regu "rumah" petugas | Ditambahkan belakangan (migration 000022) |

### 11.3 `jenis_piket` — kategori piket **dan** wadah jadwal

`2026_08_09_000003_create_jenis_piket_table.php`

Tabel ini paling banyak kolomnya karena merangkap dua peran.

| Kolom | Isi |
| --- | --- |
| `nama` | "Pengawas Piket", "Regu Pengamanan", dsb |
| `deskripsi` | Keterangan |
| `minimal_petugas_per_hari` | Standar minimal, dipakai untuk peringatan |
| `periode_mulai`, `periode_selesai` | Rentang tanggal jadwal |
| `status` | `Belum Dibuat` / `Draft` / `Aktif` / `Selesai` |
| `waktu_notifikasi` | Jam kirim WA default (08:00) |
| `waktu_notifikasi_pagi` | Jam kirim untuk shift pagi (08:00) |
| `waktu_notifikasi_siang` | Jam kirim untuk shift siang (12:00) |
| `waktu_notifikasi_malam` | Jam kirim untuk shift malam (19:00) |
| `menggunakan_shift` | true/false — apakah piket ini bershift |
| `tipe_regu` | Slug kategori regu, misal `pengamanan`, `p2u` |
| `unit_kerja` | JSON daftar unit kerja yang relevan |
| `urutan_roster` | JSON urutan baris petugas/regu di matriks |

> Kolom bertipe **JSON** artinya isinya bisa berupa daftar atau objek bersarang,
> bukan cuma satu nilai. Praktis, tapi lebih sulit dicari/diagregasi.

### 11.4 `regu` dan `regu_anggota`

`regu`: `id`, `nama`, `tipe` (slug kategori), `jenis_piket_id`,
`commander_petugas_id` (ditambah migration `2026_08_12_222312`).

`regu_anggota`: tabel penghubung (pivot). Isinya cuma `regu_id` + `petugas_id`,
dengan **unique constraint** pada pasangan keduanya — artinya satu petugas tidak
bisa terdaftar dua kali di regu yang sama.

### 11.5 `jadwal_assignments` — inti jadwal

`2026_08_09_000008_create_jadwal_assignments_table.php`

| Kolom | Isi |
| --- | --- |
| `jenis_piket_id` | Jadwal mana |
| `petugas_id` | Siapa |
| `tanggal` | Kapan |
| `shift` | Kode shift (`p`, `s`, `m`, `l`, `i`, ...) atau kosong |

Satu baris = **satu sel di matriks jadwal**. Kalau jadwal 30 hari untuk 40
petugas terisi penuh, tabel ini berisi 1.200 baris untuk satu jadwal saja.

Tiga index dipasang supaya pencarian cepat:
```php
$table->index(['jenis_piket_id', 'tanggal']);
$table->index(['petugas_id', 'tanggal']);
$table->index(['tanggal', 'jenis_piket_id'], 'jadwal_dashboard_date_index');
```
Index itu seperti daftar isi buku: tanpa itu, database harus membaca seluruh
tabel setiap kali mencari.

### 11.6 `pengingat_logs` — catatan pengiriman WA

| Kolom | Isi |
| --- | --- |
| `petugas_id`, `jenis_piket_id`, `tanggal_piket` | Untuk siapa & kapan |
| `waktu_pengingat` | `H-1 Hari` atau `H-1 Jam` (migration `2026_08_14_184142`) |
| `whatsapp` | Nomor tujuan (disalin saat itu, jadi riwayat tidak berubah) |
| `pesan` | Isi pesan lengkap (disimpan sebagai arsip) |
| `status_kirim` | `Menunggu` / `Diterima Server` / `Terkirim` / `Gagal` |
| `wa_message_id` | ID pesan dari WhatsApp, **unik** |
| `ack` | Kode status pengiriman dari WhatsApp (-1, 1, 2, ...) |
| `error_message` | Alasan gagal |

Unique constraint: `(petugas_id, jenis_piket_id, tanggal_piket)` + kolom
`waktu_pengingat`. Ini mencegah satu petugas dikirimi pengingat yang sama
berkali-kali.

### 11.7 `konfirmasi_petugas` — jawaban kesiapan H-1

| Kolom | Isi |
| --- | --- |
| `status_konfirmasi` | `Menunggu` / `Hadir` / `Berhalangan` |
| `waktu_konfirmasi` | Kapan dijawab |
| `alasan` | Alasan kalau berhalangan |
| `metode` | `WhatsApp` (otomatis) atau `Manual` (diinput operator) |

### 11.8 `kehadiran_piket` — realisasi hari-H

| Kolom | Isi |
| --- | --- |
| `status_kehadiran` | `Hadir` / `Sakit` / `Izin` / `Alpa` |
| `waktu_presensi` | Jam scan / jam input |
| `keterangan` | Catatan bebas |

Unique constraint `(petugas_id, jenis_piket_id, tanggal_piket)` — **inilah yang
menjamin satu petugas hanya punya satu catatan absen per piket per hari**,
berapa kali pun dia scan.

### 11.9 `riwayat_perubahan` — jejak audit jadwal

Mencatat siapa (`user_id`) melakukan apa (`aksi`: Simpan Draft / Aktivasi /
Reset / Nonaktifkan) pada jadwal mana, dengan alasan apa (`detail`) dan kapan.

### 11.10 `laporan_pengawalan`

Tabel besar (30+ kolom) berisi data surat perintah, identitas WBP (nama, no
register, umur, jenis kelamin, perkara, pidana, tanggal ekspirasi, alamat),
kategori pengawalan, dasar hukum, dan dua kolom JSON:
`petugas_pendamping_json` & `petugas_pemberi_perintah_json`.

### 11.11 `laporan_penggeledahan`

Berisi nomor laporan, tanggal, waktu mulai/selesai, blok & kamar, dasar
pelaksanaan, plus empat kolom JSON: `petugas_penggeledah_json`,
`barang_sitaan_json`, `dokumentasi_foto_json` (daftar path foto),
`area_penggeledahan_json` (blok bertingkat berisi daftar kamar).

### 11.12 Tabel bawaan Laravel

| Tabel | Fungsi |
| --- | --- |
| `personal_access_tokens` | Token login Sanctum |
| `jobs`, `failed_jobs`, `job_batches` | Antrean pekerjaan latar belakang |
| `cache`, `cache_locks` | Cache |
| `sessions` | Sesi web (tidak dipakai untuk API) |
| `migration_actions` | **Buatan sendiri** — mencatat aksi tiap migration untuk audit |

---

# BAGIAN II — ALUR KERJA LENGKAP

Bagian ini menelusuri setiap alur dari klik pertama sampai data tersimpan.
Setiap langkah menyebut **file dan baris** supaya Anda bisa membukanya sendiri.

---

## ALUR A — Login dan Sesi

### Cerita dari sisi pengguna

Operator membuka `http://alamat-server:3000`, melihat halaman login berlogo
Kemenimipas, mengetik username dan password, klik "Masuk". Kalau benar, dia
langsung berada di Dashboard.

### Yang terjadi di balik layar

**Langkah 1 — Pengguna menekan tombol Masuk**
`sijempol-fe/src/features/auth/components/LoginView.tsx` memanggil fungsi
`login()` dari hook `useAuth`.

**Langkah 2 — Hook memanggil API**
`sijempol-fe/src/features/auth/hooks/useAuth.ts:57-62`:
```typescript
const login = useCallback(async (username: string, password: string) => {
  const loggedInUser = await authApi.login(username, password);
  setUser(loggedInUser);
  setError(null);
  return loggedInUser;
}, []);
```

**Langkah 3 — Permintaan dikirim**
`sijempol-fe/src/features/auth/api/__request.ts:5-14`:
```typescript
async login(username: string, password: string): Promise<SystemUser> {
  const response = await request<{ user: SystemUser; token: string }>('/auth/login', {
    method: 'POST',
    authenticated: false,     // <- login belum punya token, jadi jangan tempel token
    body: { username, password },
  });
  session.setToken(response.data.token);   // <- simpan token
  return response.data.user;
},
```

**Langkah 4 — Backend menerima**
`sijempol-be/routes/api.php:20-22`:
```php
Route::post('/login', [AuthController::class, 'login'])
    ->middleware('throttle:login');
```
`throttle:login` = pembatas 5 percobaan per menit (didefinisikan di
`AppServiceProvider.php:94-104`). Jadi orang tidak bisa menebak password
ribuan kali.

**Langkah 5 — Validasi**
`sijempol-be/app/Http/Requests/Auth/LoginRequest.php` memeriksa username &
password terisi.

**Langkah 6 — Controller memeriksa & menerbitkan token**
`sijempol-be/app/Http/Controllers/Api/AuthController.php`. Password
dibandingkan dengan versi hash di database. Kalau cocok, Sanctum menerbitkan
token acak panjang. Token itu dicatat di tabel `personal_access_tokens`.

**Langkah 7 — Frontend menyimpan token**
Disimpan di `sessionStorage` dengan kunci `silaponti.auth.token`
(`shared/api/__request.ts:4`).

### Memulihkan sesi setelah refresh halaman

Kalau pengguna menekan F5, React memulai dari nol dan "lupa" siapa yang login.
Karena itu ada mekanisme pemulihan di `useAuth.ts:20-45`:

```typescript
useEffect(() => {
  let active = true;
  const restore = async () => {
    if (!session.hasToken()) {          // (1) tidak ada token -> selesai, tampilkan login
      if (active) setIsRestoring(false);
      return;
    }
    try {
      const restoredUser = await authApi.me();   // (2) ada token -> tanya backend "saya siapa?"
      if (active) setUser(restoredUser);
    } catch (restoreError) {
      // (3) token kedaluwarsa -> diam saja, biarkan pengguna login lagi
    } finally {
      if (active) setIsRestoring(false);
    }
  };
  void restore();
  return () => { active = false; };     // (4) bersih-bersih
}, []);
```

Variabel `active` mencegah bug klasik: kalau pengguna pindah halaman sementara
permintaan masih berjalan, jawaban yang telat datang tidak akan mengubah
tampilan yang sudah tidak ada. Ini praktik React yang benar.

Selama `isRestoring` bernilai true, `App.tsx:158` menampilkan layar skeleton —
supaya pengguna tidak sempat melihat halaman login berkedip.

### Logout

`authApi.logout()` memanggil `POST /auth/logout` (backend mencabut token dari
database), lalu **apa pun hasilnya** menghapus token lokal:
```typescript
async logout(): Promise<void> {
  try {
    await request('/auth/logout', { method: 'POST' });
  } finally {
    session.clear();     // <- 'finally' = dijalankan walau permintaan gagal
  }
}
```
Ini penting: kalau server mati, pengguna tetap harus bisa logout dari
browsernya.

---

## ALUR B — Data Petugas

Halaman: `/petugas` → `sijempol-fe/src/features/petugas/components/PetugasView.tsx`

### Yang bisa dilakukan

| Aksi | Endpoint | Siapa boleh |
| --- | --- | --- |
| Lihat daftar (dengan pencarian & filter status) | `GET /petugas` | Admin, Operator |
| Tambah | `POST /petugas` | Admin, Operator |
| Lihat detail | `GET /petugas/{id}` | Admin, Operator |
| Ubah | `PUT /petugas/{id}` | Admin, Operator |
| **Hapus** | `DELETE /petugas/{id}` | **Admin saja** |
| Import massal | (dari file, diproses di frontend) | Admin, Operator |

### Pencarian di backend

`sijempol-be/app/Models/Petugas.php:33-58`:
```php
->when($search, function (Builder $query, string $search): void {
    $query->where(function (Builder $query) use ($search): void {
        $query
            ->whereLike('nip', "%{$search}%")
            ->orWhereLike('nama', "%{$search}%")
            ->orWhereLike('jabatan', "%{$search}%")
            ->orWhereLike('unit_kerja', "%{$search}%");
    });
})
```
`when($search, ...)` artinya *"kalau ada kata kunci pencarian, tambahkan
kondisi ini; kalau tidak ada, lewati saja"*. Ini cara Laravel membangun query
secara bertahap tanpa menulis `if` bertumpuk.

`%{$search}%` = "mengandung", jadi mencari "bud" akan menemukan "Budi" dan
"Abdul Budiman".

### Perlindungan hapus

`Petugas.php:110-118`:
```php
public function isUsed(): bool
{
    return $this->regu()->exists()
        || $this->jadwalAssignments()->exists()
        || $this->pengingatLogs()->exists()
        || $this->konfirmasi()->exists()
        || $this->kehadiran()->exists();
}
```
Sebelum menghapus, sistem memeriksa apakah petugas ini sudah dipakai di regu,
jadwal, pengingat, konfirmasi, atau kehadiran. Kalau ya, penghapusan ditolak.
Ini melindungi integritas riwayat — Anda tidak akan punya catatan absen tanpa
pemilik.

### Import massal petugas

`sijempol-fe/src/features/petugas/components/bulkPetugasImport.ts` +
`BulkPetugasModal.tsx`. Operator menempel data dari Excel, sistem menguraikan
per baris, lalu mengirim satu per satu ke `POST /petugas`. Ada test untuk ini:
`src/tests/bulkPetugasImport.test.ts`.

---

## ALUR C — Jenis Piket dan Regu

### C.1 Membuat Jenis Piket

Halaman: `/jadwal` → tombol tambah → `JenisPiketModal.tsx`

Yang diisi:
- Nama (misal "Regu Pengamanan")
- Deskripsi
- Minimal petugas per hari
- Unit kerja terkait
- **Apakah beregu?** (`isBeregu`) dan `tipeRegu` (slug, misal `pengamanan`)
- **Menggunakan shift?** (`menggunakanShift`)
- Empat jam notifikasi (default, pagi, siang, malam)
- Periode mulai & selesai

Backend: `sijempol-be/app/Models/JenisPiket.php:59-68`
```php
public static function createFromPayload(array $attributes): self
{
    return DB::transaction(function () use ($attributes): self {
        $jenisPiket = self::query()->create(self::normalizePayload($attributes));
        $jenisPiket->ensureKategoriRegu();

        return $jenisPiket;
    });
}
```
Perhatikan `ensureKategoriRegu()`: kalau jenis piket ini beregu dan kategorinya
belum ada, sistem **otomatis mendaftarkan kategori regu baru** dan membuat regu
placeholder. Jadi operator tidak perlu menyiapkan master data dulu.

`DB::transaction(...)` membungkus keduanya: kalau pembuatan kategori gagal,
jenis piketnya juga batal. Tidak ada data setengah jadi.

### C.2 Mengatur Regu

Halaman: `/regu?tipe=pengamanan` → `ReguView.tsx`

| Aksi | Endpoint |
| --- | --- |
| Daftar regu | `GET /regu` |
| Daftar kategori regu | `GET /kategori-regu` |
| Tambah regu | `POST /regu` |
| Ubah nama/tipe | `PUT /regu/{id}` |
| **Ganti seluruh anggota sekaligus** | `PUT /regu/{id}/anggota` |
| Hapus regu | `DELETE /regu/{id}` (Admin) |

Endpoint "ganti anggota" bekerja secara **atomik**: seluruh daftar anggota
lama dihapus dan diganti daftar baru dalam satu transaksi. Kelebihannya:
tidak mungkin ada kondisi setengah jadi di mana sebagian anggota sudah diganti
dan sebagian belum.

### C.3 Kaitan Petugas <-> Regu

Ada dua jalur relasi yang sengaja dibuat:
1. `petugas.regu_id` — regu "rumah"/asal petugas (kolom langsung).
2. Tabel `regu_anggota` — keanggotaan riil, bisa banyak regu.

`Petugas.php:88-104` menjaga keduanya tetap sinkron:
```php
public function syncReguMembership(?string $reguId): void
{
    if ($reguId === null) {
        return;
    }

    $membershipExists = ReguAnggota::query()
        ->where('regu_id', $reguId)
        ->where('petugas_id', $this->id)
        ->exists();

    if (! $membershipExists) {
        ReguAnggota::query()->create([
            'regu_id' => $reguId,
            'petugas_id' => $this->id,
        ]);
    }
}
```
Baca: *"Kalau petugas diberi regu rumah, pastikan dia juga terdaftar sebagai
anggota regu itu — tapi jangan sentuh keanggotaan regu lainnya."*

---

## ALUR D — Menyusun Jadwal Piket (Matriks Excel)

Ini **fitur terbesar** di seluruh sistem. File utamanya
`sijempol-fe/src/features/jadwal-piket/components/JadwalPiketView.tsx`
sepanjang **1.424 baris**, dibantu `JadwalPiketMatrix.tsx` (809 baris).

### Cerita dari sisi pengguna

Panduan resmi ada di dalam aplikasi sendiri
(`sijempol-fe/src/features/panduan/components/PanduanView.tsx`), isinya:

1. **Pilih Jenis Piket & Periode** — pilih kategori, atur bulan/tahun atau
   tanggal lintas bulan.
2. **Daftarkan Petugas ke Matriks** — klik "Pilih Petugas", cari & filter unit,
   centang nama-nama, klik tambah. Setiap petugas jadi satu baris.
3. **Isi Tanggal Piket** — klik sel pertemuan Nama x Tanggal. Klik lagi untuk
   membatalkan. Bisa **drag mouse** atau **Shift+Klik** untuk mengisi cepat.
4. **Alat Bantu Isi Cepat** — dropdown di ujung baris untuk pola otomatis:
   tanggal ganjil/genap, selang 2/3 hari, akhir pekan, atau **menyalin pola
   dari petugas lain**.

### Komponen-komponen yang terlibat

| File | Tugas |
| --- | --- |
| `JadwalPiketView.tsx` | Pengatur utama, menyimpan seluruh state matriks |
| `JadwalPiketMatrix.tsx` | Menggambar tabel raksasa + menangani klik/drag |
| `JadwalPiketToolbar.tsx` | Tombol-tombol di atas tabel |
| `JadwalPiketCalendar.tsx` | Tampilan kalender alternatif |
| `list/JenisPiketTable.tsx` | Daftar jenis piket sebelum masuk editor |
| `editor/SchedulePeriodControls.tsx` | Pengatur rentang tanggal |
| `editor/ScheduleValidationPanel.tsx` | Panel peringatan konflik |
| `editor/ScheduleEditorSummary.tsx` | Ringkasan sebelum simpan |
| `modals/OfficerSelectionModal.tsx` | Dialog pilih petugas (387 baris) |
| `modals/ReguSelectionModal.tsx` | Dialog pilih regu |
| `modals/CopyPatternModal.tsx` | Dialog salin pola |
| `modals/KplpRotationModal.tsx` | Dialog rotasi KPLP |
| `modals/ScheduleSummaryModal.tsx` | Dialog konfirmasi akhir |

### Fungsi bantu (lib/) — logika murni yang bisa diuji

| File | Fungsi |
| --- | --- |
| `scheduleValidation.ts` | Mendeteksi konflik & kekurangan petugas |
| `applyKplpRotation.ts` | Menerapkan pola rotasi KPLP otomatis |
| `reconcileScheduleOfficers.ts` | Menyelaraskan daftar petugas di matriks dengan data terbaru |
| `resolveScheduleOfficers.ts` | Menentukan petugas mana yang muncul di baris |
| `rosterOrder.ts` | Mengatur urutan baris |
| `semesterChecklist.ts` | Checklist per semester |
| `finishScheduleSubmission.ts` | Langkah akhir sebelum kirim |
| `validateJenisPiketForm.ts` | Validasi form jenis piket |

Semua file ini punya **test otomatis** di `src/tests/`. Ini bagian paling
sehat dari proyek — logika rumit dipisah ke fungsi murni yang bisa diuji tanpa
membuka browser.

### Kode shift yang tersedia

`sijempol-be/app/Enums/JadwalShift.php` mendefinisikan 18 kode:

| Kode | Arti | Kirim WA? |
| --- | --- | --- |
| `p` | Pagi | Ya, jam pagi |
| `s` | Siang | Ya, jam siang |
| `m` | Malam | Ya, jam malam |
| `pm` | Pagi-Malam | Ya, jam malam |
| `pp` | Piket Pagi | Ya, jam pagi |
| `ps` | Piket Siang | Ya, jam siang |
| `fs` | Pagi-Siang | Ya, jam siang |
| `pmrs` | Piket Malam Rumah Sakit | Ya, jam malam |
| `i` | **Istirahat** | **Tidak** (dilewati total) |
| `l` | Libur | **Tidak** |
| `cd` | Cadangan | **Tidak** |
| `lp` | Lepas Piket | **Tidak** |
| `x_pw` | Perwira Piket | Ya, jam default |
| `duta` | Duta Layanan | Ya, jam default |
| `pw` | Penggeledahan Wanita | Ya, jam default |
| `x_prk` | Pencatatan Ruang Kunjungan | Ya, jam default |
| `sdp` | Pendaftaran SDP | Ya, jam default |
| `pt` | Barang Titipan | Ya, jam default |

Logika pemetaan ini ada di `JadwalService.php:364-380`:
```php
private function notificationTimeForShift(JenisPiket $jenisPiket, ?string $shift): ?string
{
    if ($jenisPiket->menggunakan_shift === false) {
        return in_array($shift, ['cd', 'i', 'l', 'lp'], true)
            ? null
            : (string) $jenisPiket->waktu_notifikasi;
    }

    return match ($shift) {
        'p', 'pp' => (string) $jenisPiket->waktu_notifikasi_pagi,
        's', 'ps', 'fs' => (string) $jenisPiket->waktu_notifikasi_siang,
        'm', 'pm', 'pmrs' => (string) $jenisPiket->waktu_notifikasi_malam,
        'cd', 'i', 'l', 'lp' => null,
        default => (string) $jenisPiket->waktu_notifikasi,
    };
}
```
`match` adalah versi modern dari `switch`. `null` berarti "jangan kirim
pengingat sama sekali" — masuk akal, karena orang libur tidak perlu diingatkan.

### Menyimpan draft

Frontend mengumpulkan seluruh isi matriks jadi satu daftar besar `assignments`
lalu mengirim sekaligus. `sijempol-fe/src/features/jadwal-piket/api/__request.ts:72-99`:

```typescript
async save(jenisPiket, assignments, isDraft, reason, urutanRoster) {
  const draft = await request<any>(`/jadwal/${jenisPiket.id}`, {
    method: 'PUT',
    body: {
      periodeMulai: jenisPiket.periodeMulai,
      periodeSelesai: jenisPiket.periodeSelesai,
      assignments,
      urutanRoster: urutanRoster ?? null,
      reason: reason || 'Pembaruan matriks jadwal dari frontend',
    },
  });
  if (isDraft) return mapJadwalDetail(draft.data);

  // kalau BUKAN draft, langsung lanjut aktivasi
  const activated = await request<any>(`/jadwal/${jenisPiket.id}/activate`, {
    method: 'POST',
    body: { reason: reason || 'Aktivasi jadwal dari frontend' },
  });
  return mapJadwalDetail(activated.data);
}
```

Perhatikan pola **dua langkah**: selalu simpan draft dulu, baru aktivasi.
Kalau simpan gagal, aktivasi tidak pernah terjadi.

### Backend menyimpan draft

`sijempol-be/app/Services/JadwalService.php:60-115`. Alurnya:

```php
DB::transaction(function () use ($jenisPiket, $user, $data): void {
    // (1) Kunci baris jenis_piket supaya tidak ada yang mengubah bersamaan
    $lockedJenisPiket = JenisPiket::query()
        ->lockForUpdate()
        ->findOrFail($jenisPiket->getKey());

    // (2) Jadwal yang sudah Aktif/Selesai tidak boleh diedit
    if (in_array($lockedJenisPiket->status, [JenisPiketStatus::Aktif, JenisPiketStatus::Selesai], true)) {
        throw JadwalStateException::cannotModify($lockedJenisPiket->status->value);
    }

    // (3) Perbarui periode & set status jadi Draft
    $lockedJenisPiket->update($attributes);

    // (4) Hapus pengingat & konfirmasi lama (draft belum boleh punya jadwal kirim)
    PengingatLog::query()->whereBelongsTo($lockedJenisPiket)->delete();
    KonfirmasiPetugas::query()->whereBelongsTo($lockedJenisPiket)->delete();

    // (5) Hapus semua assignment lama, lalu tulis ulang dari nol
    JadwalAssignment::query()->whereBelongsTo($lockedJenisPiket)->delete();

    foreach ($data['assignments'] as $assignment) {
        JadwalAssignment::query()->create([...]);
    }

    // (6) Bekukan komposisi regu saat ini
    $this->snapshotRegus($lockedJenisPiket);

    // (7) Catat di riwayat perubahan
    RiwayatPerubahan::query()->create([
        'user_id' => $user->id,
        'aksi' => 'Simpan Draft',
        'detail' => $data['reason'],
    ]);
}, attempts: 3);
```

**Yang perlu Anda pahami dari kode ini:**

- **`lockForUpdate()`** — mengunci baris. Kalau dua operator menekan Simpan
  bersamaan, yang kedua menunggu sampai yang pertama selesai. Tanpa ini, jadwal
  bisa tercampur aduk.
- **Hapus-lalu-tulis-ulang** — pendekatan sederhana dan aman. Tidak perlu
  menghitung "mana yang berubah"; buang semua, tulis ulang. Karena ada di dalam
  transaksi, tidak akan ada momen di mana jadwal kosong bagi pengguna lain.
- **`attempts: 3`** — kalau transaksi gagal karena bentrok, coba ulang sampai
  3 kali otomatis.
- **`snapshotRegus`** — komentar aslinya: *"Freeze the regu composition shown
  on the matrix at this point in time, so later regu rollings never alter the
  composition of this schedule."* Artinya: kalau bulan depan regu diacak ulang,
  jadwal bulan ini tetap menunjukkan susunan yang benar saat dibuat. Ini
  penting untuk arsip.

---

## ALUR E — Aktivasi Jadwal (jantung sistem)

Ini alur **paling penting dan paling rumit** di seluruh aplikasi. Sekali
diaktifkan, sistem membuat puluhan/ratusan pengingat dan konfirmasi sekaligus.

File: `sijempol-be/app/Services/JadwalService.php:124-227`

### Peta langkah

```
POST /jadwal/{jenisPiket}/activate
   |
   v
[MULAI TRANSAKSI DATABASE]
   |
   1. Kunci baris jenis_piket (lockForUpdate)
   2. Muat semua assignment + data petugasnya
   3. TOLAK kalau assignment kosong
   4. TOLAK kalau status bukan 'Draft'
   5. Ubah status jadi 'Aktif'
   6. Bekukan komposisi regu (snapshot)
   7. Catat 'Aktivasi' di riwayat_perubahan
   8. Untuk SETIAP assignment:
        - lewati kalau shift = 'i' (istirahat)
        - lewati kalau tanggalnya sudah lewat
        - buat 1 KonfirmasiPetugas berstatus 'Menunggu'
        - tentukan jam notifikasi dari shift
        - lewati kalau shift tidak perlu WA (libur/cadangan/lepas)
        - buat 2 PengingatLog: 'H-1 Hari' dan 'H-1 Jam'
        - kumpulkan yang perlu dikirim
   |
   v
[COMMIT TRANSAKSI]
   |
   v
9. Setelah commit: masukkan job kirim WA ke antrean (dengan delay)
   |
   v
10. Kembalikan jadwal terbaru + daftar peringatan
```

### Kode kuncinya, dijelaskan

**Penolakan awal** (`JadwalService.php:136-144`):
```php
if ($lockedJenisPiket->jadwalAssignments->isEmpty()) {
    throw JadwalStateException::emptyAssignments();
}

if ($lockedJenisPiket->status !== JenisPiketStatus::Draft) {
    throw JadwalStateException::cannotActivate($lockedJenisPiket->status->value);
}
```
Dua penjaga: jadwal kosong tidak bisa diaktifkan, dan jadwal yang sudah aktif
tidak bisa diaktifkan dua kali (mencegah pengingat dobel).

**Melewati shift istirahat** (`JadwalService.php:159-161`):
```php
foreach ($lockedJenisPiket->jadwalAssignments as $assignment) {
    if ($assignment->shift === 'i') {
        continue;
    }
```
`continue` = "lompati yang ini, lanjut ke berikutnya".

**Melewati tanggal yang sudah lewat** (`JadwalService.php:165-170`):
```php
// Assignments on past dates are historical; reminders and
// H-1 confirmations for them would only surface stale
// "failed"/"pending" records on the dashboard.
if ($assignment->tanggal->lt($today)) {
    continue;
}
```
Komentarnya jelas: kalau operator mengaktifkan jadwal bulan lalu, sistem tidak
akan mengirim pengingat untuk tanggal yang sudah lewat — itu hanya akan
mengotori dashboard dengan status "Gagal".

**Membuat konfirmasi** (`JadwalService.php:172-176`):
```php
KonfirmasiPetugas::query()->firstOrCreate([
    'petugas_id' => $petugas->id,
    'jenis_piket_id' => $lockedJenisPiket->id,
    'tanggal_piket' => $assignment->tanggal->toDateString(),
]);
```
`firstOrCreate` = "cari dulu; kalau sudah ada pakai yang ada, kalau belum ada
buat baru". Inilah yang membuat aktivasi **idempotent** — dijalankan dua kali
tidak menghasilkan data ganda.

**Membuat dua pengingat sekaligus** (`JadwalService.php:186-211`):
```php
foreach ([
    'H-1 Hari' => $assignment->tanggal->copy()->subDay()->setTimeFromTimeString($notificationTime),
    'H-1 Jam'  => $assignment->tanggal->copy()->setTimeFromTimeString($notificationTime)->subHour(),
] as $waktuPengingat => $sendAt) {
    $pengingat = PengingatLog::query()->firstOrCreate(
        [
            'petugas_id' => $petugas->id,
            'jenis_piket_id' => $lockedJenisPiket->id,
            'tanggal_piket' => $assignment->tanggal->toDateString(),
            'waktu_pengingat' => $waktuPengingat,
        ],
        [
            'whatsapp' => $petugas->whatsapp,
            'pesan' => $this->reminderMessage($lockedJenisPiket, $petugas, $assignment),
            'status_kirim' => $hasValidWhatsapp
                ? PengingatStatus::Menunggu
                : PengingatStatus::Gagal,
        ],
    );
    ...
}
```

Baca perlahan:
- Setiap petugas dapat **dua** pengingat: satu **sehari sebelumnya** pada jam
  notifikasi, satu lagi **satu jam sebelum** jam notifikasi di hari-H.
- `subDay()` = kurangi satu hari. `subHour()` = kurangi satu jam.
- `setTimeFromTimeString($notificationTime)` = pasang jamnya sesuai shift.
- Kalau nomor WA tidak valid, pengingat langsung berstatus **Gagal** — tapi
  **aktivasi tetap berhasil**. Ini keputusan desain yang tepat: satu nomor
  bermasalah tidak boleh menggagalkan jadwal 40 orang.

**Isi pesannya** (`JadwalService.php:345-353`):
```php
return "Yth. {$petugas->nama}, mengingatkan jadwal {$jenisPiket->nama} Anda pada {$assignment->tanggal->toDateString()}.\n"
    .'Balas: SIAP atau BERHALANGAN <alasan>.';
```
Contoh nyata:
> Yth. Budi Santoso, mengingatkan jadwal Regu Pengamanan Anda pada 2026-08-25.
> Balas: SIAP atau BERHALANGAN <alasan>.

> **Catatan:** format tanggal `2026-08-25` kurang ramah untuk pembaca Indonesia.
> Idealnya "Senin, 25 Agustus 2026". Ini perbaikan kecil yang layak dilakukan.

**Mengirim job SETELAH transaksi commit** (`JadwalService.php:218-226`):
```php
foreach ($scheduledReminders as $scheduledReminder) {
    $pendingDispatch = SendWhatsAppReminderJob::dispatch($scheduledReminder['id']);

    if ($scheduledReminder['sendAt']->isFuture()) {
        $pendingDispatch->delay($scheduledReminder['sendAt']);
    }

    $pendingDispatch->afterCommit();
}
```

**Ini detail arsitektur yang sangat penting.** Perhatikan bahwa `foreach` ini
berada **di luar** `DB::transaction(...)`. Kenapa?

Bayangkan kalau job dimasukkan ke antrean **di dalam** transaksi:
1. Job masuk antrean
2. Worker langsung mengambilnya (cepat sekali!)
3. Worker mencari `PengingatLog` dengan ID itu di database
4. Tapi transaksi **belum commit** → data belum ada → worker error

Dengan `afterCommit()` dan penempatan di luar transaksi, job baru masuk antrean
setelah data benar-benar tersimpan. Ini bug klasik yang sudah dihindari di sini.

`->delay($sendAt)` artinya: "jangan kerjakan sekarang, kerjakan pada waktu ini".
Jadi pengingat H-1 untuk tanggal 25 Agustus benar-benar dikirim tanggal 24
Agustus jam 08:00, bukan langsung saat aktivasi.

### Peringatan (warnings)

`JadwalService.php:311-340` mengumpulkan dua jenis peringatan:

```php
if ($petugas->status !== PetugasStatus::Aktif) {
    $warnings[] = [
        'code' => 'PETUGAS_TIDAK_AKTIF',
        'petugasId' => $petugas->id,
        'message' => "{$petugas->nama} berstatus {$petugas->status->value}.",
    ];
}

if (! $this->hasValidWhatsapp($petugas)) {
    $warnings[] = [
        'code' => 'WHATSAPP_TIDAK_VALID',
        'petugasId' => $petugas->id,
        'message' => "{$petugas->nama} tidak memiliki nomor WhatsApp yang valid.",
    ];
}
```

Peringatan **tidak menggagalkan** aktivasi — hanya ditampilkan ke operator.
Petugas yang sedang Cuti atau Dinas Luar tetap bisa dijadwalkan (kadang memang
perlu), tapi operator diberi tahu.

### Menonaktifkan jadwal

`JadwalService.php:265-296`. Berbeda dari Reset, `deactivate` **tidak
menghapus** assignment — hanya mengembalikan status ke Draft dan menghapus
semua pengingat & konfirmasi.

```php
// Delete every notification for this schedule. Delayed jobs become
// harmless because their reminder record no longer exists.
PengingatLog::query()->whereBelongsTo($lockedJenisPiket)->delete();
```

Komentarnya cerdas: job yang sudah terlanjur dijadwalkan di antrean tidak perlu
dibatalkan satu per satu — cukup hapus data pengingatnya, dan job akan
"menemukan tidak ada apa-apa" lalu berhenti sendiri. Lihat
`SendWhatsAppReminderJob.php:65-72`:
```php
try {
    $pengingat = $pengingatService->prepareForDelivery($this->pengingatLogId);
} catch (ModelNotFoundException) {
    return;      // <- data sudah dihapus, diam-diam selesai
}
```

### Empat status jadwal

```
 Belum Dibuat  --(simpan draft)-->  Draft  --(aktivasi)-->  Aktif
      ^                              |  ^                     |
      |                              |  |                     |
      +--------(reset)---------------+  +----(nonaktifkan)----+
                                                              |
                                                              v
                                                          Selesai
```

| Status | Bisa diedit? | Bisa diaktifkan? |
| --- | --- | --- |
| `Belum Dibuat` | Ya | Tidak (harus jadi Draft dulu) |
| `Draft` | Ya | **Ya** |
| `Aktif` | Tidak | Tidak (sudah aktif) |
| `Selesai` | Tidak | Tidak |

---

## ALUR F — Pengingat WhatsApp

### Rantai lengkap

```
PengingatLog dibuat (status: Menunggu)
   |
   v
SendWhatsAppReminderJob masuk antrean (tabel jobs) dengan delay
   |
   v  [menunggu sampai waktunya]
Worker `php artisan queue:work` mengambil job
   |
   v
PengingatService::prepareForDelivery()  <- kunci & ambil nomor WA TERBARU
   |
   v
GoWhatsAppSender::sendText()  --HTTP POST--> GOWA --> WhatsApp --> HP petugas
   |
   v
PengingatLog diperbarui: wa_message_id, ack, status
   |
   v (kalau belum pasti terkirim)
ReconcileWhatsAppDeliveryJob  <- cek ulang 5 detik kemudian
```

### Job pengirim: `app/Jobs/SendWhatsAppReminderJob.php`

**Pengaturan di atas kelas** (baris 21-29):
```php
class SendWhatsAppReminderJob implements ShouldBeUnique, ShouldQueue
{
    public int $tries = 120;      // coba sampai 120 kali
    public int $timeout = 60;     // maksimal 60 detik per percobaan
    public int $uniqueFor = 3600; // kunci "unik" berlaku 1 jam
```

- `ShouldQueue` = "kerjakan di latar belakang, jangan bikin pengguna menunggu".
- `ShouldBeUnique` = "kalau job untuk pengingat yang sama sudah ada di antrean,
  jangan tambah lagi". **Ini pencegah WA dobel.**
- `tries = 120` terdengar banyak, tapi masuk akal karena WhatsApp sering
  sementara tidak bisa dihubungi.

**Jeda antar percobaan** (baris 38-41):
```php
public function backoff(): array
{
    return [10, 30, 60];
}
```
Gagal pertama → tunggu 10 detik. Gagal kedua → 30 detik. Gagal ketiga dan
seterusnya → 60 detik. Namanya *exponential backoff* — tidak membanjiri
layanan yang sedang bermasalah.

**Pembatas kecepatan** (baris 46-52):
```php
public function middleware(): array
{
    return [
        (new RateLimited('whatsapp-send'))
            ->releaseAfter(config('sijempol.whatsapp.message_interval_seconds', 5)),
    ];
}
```
**Ini paling kritis untuk operasional.** WhatsApp memblokir nomor yang mengirim
pesan terlalu cepat. Setting default di `.env.example` adalah
`WHATSAPP_MESSAGE_INTERVAL_SECONDS=20`, artinya **satu pesan tiap 20 detik**.

> **Hitungan praktis:** kalau jadwal Anda punya 40 petugas x 30 hari x 2
> pengingat = 2.400 pesan. Dengan jeda 20 detik, itu 48.000 detik = **13 jam**
> untuk mengosongkan antrean. Tapi karena setiap pesan punya `delay` sendiri
> (dikirim H-1), antrean tidak menumpuk sekaligus. Tetap, ini angka yang perlu
> Anda pantau.

**Alur handle()** (baris 62-113):
```php
public function handle(WhatsAppSender $whatsAppSender, PengingatService $pengingatService): void
{
    // (1) Ambil & kunci pengingat; kalau sudah dihapus -> selesai diam-diam
    try {
        $pengingat = $pengingatService->prepareForDelivery($this->pengingatLogId);
    } catch (ModelNotFoundException) {
        return;
    }
    if ($pengingat === null) {
        return;   // statusnya bukan 'Menunggu' lagi -> jangan kirim ulang
    }

    // (2) Kirim
    try {
        $result = $whatsAppSender->sendText($pengingat->whatsapp, $pengingat->pesan, $pengingat->getKey());
    } catch (WhatsAppRecipientNotRegisteredException) {
        $pengingat->markAsFailed('Nomor tujuan tidak terdaftar di WhatsApp.');
        return;
    } catch (WhatsAppTransportUnavailableException $exception) {
        $pengingat->markAsFailed($exception->getMessage());
        return;
    } catch (WhatsAppSendOutcomeUnknownException $exception) {
        $pengingat->markSendOutcomeUnknown();
        Log::warning('Hasil pengiriman pengingat WhatsApp belum dapat dipastikan.', [...]);
        return;
    }

    // (3) Simpan hasil
    $pengingat = $pengingat->recordSendResult($result);

    // (4) Kalau statusnya belum pasti, jadwalkan pengecekan ulang
    if ($pengingat->status_kirim !== PengingatStatus::Terkirim
        && $pengingat->status_kirim !== PengingatStatus::Gagal) {
        ReconcileWhatsAppDeliveryJob::dispatch($pengingat->getKey())
            ->delay(now()->addSeconds(5))
            ->afterCommit();
    }
}
```

**Yang bagus dari kode ini:** ada **tiga jenis kegagalan** yang dibedakan:

| Jenis error | Artinya | Tindakan |
| --- | --- | --- |
| `WhatsAppRecipientNotRegistered` | Nomornya bukan WhatsApp | Tandai Gagal, **jangan** coba lagi |
| `WhatsAppTransportUnavailable` | GOWA menolak permintaan | Tandai Gagal, **jangan** coba lagi |
| `WhatsAppSendOutcomeUnknown` | Koneksi putus, **tidak tahu** pesan terkirim atau tidak | Jangan tandai apa-apa, catat di log |

Kasus ketiga adalah yang paling sulit di sistem apa pun: **kita tidak tahu
apakah pesannya sampai.** Menandainya "Gagal" berisiko mengirim dobel;
menandainya "Terkirim" berisiko petugas tidak diingatkan. Sistem ini memilih
**tidak berbohong** — biarkan statusnya menggantung dan catat peringatan.

**Kalau semua percobaan habis** (baris 115-126):
```php
public function failed(?Throwable $exception): void
{
    PengingatLog::markDeliveryFailedById(
        $this->pengingatLogId,
        'Pengiriman gagal setelah seluruh percobaan transport selesai.',
    );
    Log::warning('Pengiriman pengingat WhatsApp gagal.', [...]);
}
```

### Adapter GOWA: `app/Services/GoWhatsAppSender.php`

Ini "colokan" ke layanan WhatsApp. Dipisah supaya kalau nanti pindah dari GOWA
ke layanan lain, cukup ganti file ini.

**Kontraknya** ada di `app/Contracts/WhatsAppSender.php` — sebuah *interface*
(daftar fungsi yang wajib ada). Ada tiga implementasi:
- `GoWhatsAppSender` — yang asli
- `UnavailableWhatsAppSender` — dipakai kalau GOWA belum dikonfigurasi
- (fake) — dipakai saat testing

Pemilihan otomatis ada di `AppServiceProvider.php:27-42`:
```php
$this->app->bind(WhatsAppSender::class, function (Application $app): WhatsAppSender {
    $url = config('services.gowa.url');
    $deviceId = config('services.gowa.device_id');

    if (! is_string($url) || blank($url) || ! is_string($deviceId) || blank($deviceId)) {
        return $app->make(UnavailableWhatsAppSender::class);
    }

    return $app->make(GoWhatsAppSender::class);
});
```
Baca: *"Kalau GOWA_URL atau GOWA_DEVICE_ID kosong di file .env, pakai versi
'tidak tersedia' yang selalu melapor gagal dengan sopan. Kalau lengkap, pakai
yang asli."* Berkat ini, aplikasi tetap jalan walau WhatsApp belum disiapkan.

**Format nomor tujuan** (`GoWhatsAppSender.php:172-181`):
```php
private function recipientId(string $whatsapp): string
{
    $normalized = $this->numberNormalizer->normalize($whatsapp);

    if ($normalized === null) {
        throw new InvalidArgumentException('Nomor WhatsApp tidak valid.');
    }

    return $normalized.self::RecipientSuffix;   // '@s.whatsapp.net'
}
```
`WhatsAppNumberNormalizer` mengubah `0812-3456-7890` menjadi `6281234567890`,
lalu ditambahi `@s.whatsapp.net` sesuai format yang diminta WhatsApp.

### Status pengiriman dan artinya

| Status | Artinya untuk operator |
| --- | --- |
| `Menunggu` | Masih di antrean, belum dikirim |
| `Diterima Server` | Server WhatsApp sudah menerima, HP tujuan belum |
| `Terkirim` | Sudah sampai di HP petugas |
| `Gagal` | Tidak berhasil; **bisa di-resend** |

Kode `ack` dari WhatsApp: `-1` = gagal, `1` = diterima server, `>= 2` = terkirim.

### Kirim ulang (resend)

Halaman `/pengingat` → `PengingatView.tsx`. Backend:
`app/Services/PengingatService.php:19-84`.

Aturannya ketat:
```php
if ($lockedPengingat->tanggal_piket->isAfter(today())) {
    return [... 'message' => 'Kirim ulang hanya tersedia pada atau setelah tanggal tugas dimulai.'];
}

if ($lockedPengingat->status_kirim !== PengingatStatus::Gagal) {
    $message = match ($lockedPengingat->status_kirim) {
        PengingatStatus::Menunggu => 'Pengingat masih berada dalam antrean pengiriman.',
        PengingatStatus::DiterimaServer => 'Pengingat sudah diterima server WhatsApp dan menunggu perangkat tujuan.',
        PengingatStatus::Terkirim => 'Pengingat sudah mencapai perangkat tujuan dan tidak dikirim ulang.',
        PengingatStatus::Gagal => '',
    };
    return [... 'queued' => false, 'message' => $message];
}
```

**Hanya yang berstatus Gagal yang boleh dikirim ulang.** Ini mencegah operator
tidak sengaja membanjiri petugas dengan pesan berulang.

Dan resend dikirim **langsung**, bukan lewat antrean
(`PengingatService.php:76-81`):
```php
if ($result['queued']) {
    // A manual resend is an operator action: send it now instead of
    // depending on a background worker that may not be running.
    SendWhatsAppReminderJob::dispatchSync($result['pengingat']->getKey());
    ...
}
```
`dispatchSync` = kerjakan sekarang juga, tunggu selesai. Komentarnya jujur:
kalau worker mati, resend manual tetap harus jalan. Pragmatis dan benar.

---

## ALUR G — Konfirmasi Kesiapan H-1

### Cerita

Pak Budi menerima WA: *"Yth. Budi Santoso, mengingatkan jadwal Regu Pengamanan
Anda pada 2026-08-25. Balas: SIAP atau BERHALANGAN <alasan>."*

Pak Budi membalas: **"Siap komandan"**.

Dalam hitungan detik, di layar operator status Pak Budi berubah dari
"Menunggu" jadi **"Hadir"** — tanpa ada yang mengetik apa pun.

### Bagaimana itu bisa terjadi

**Langkah 1 — GOWA menerima balasan dan menelepon backend**

GOWA dikonfigurasi untuk mengirim webhook ke:
```
POST /api/v1/webhooks/gowa/reports
```
(`sijempol-be/routes/api.php:31`)

**Langkah 2 — Backend memverifikasi bahwa panggilan itu asli**

`app/Http/Controllers/Api/KonfirmasiWebhookController.php:19-30`:
```php
$signature = $request->header('X-HUB-SIGNATURE-256');
$secret = config('services.gowa.webhook_secret');

if (! $signature || ! $secret) {
    return response()->json(['error' => 'Unauthorized'], 401);
}

$expected = 'sha256='.hash_hmac('sha256', $request->getContent(), $secret);
if (! hash_equals($expected, $signature)) {
    return response()->json(['error' => 'Invalid signature'], 401);
}
```

**Penjelasan awam:** GOWA dan backend punya "kata sandi rahasia bersama"
(`GOWA_WEBHOOK_SECRET`). Setiap kali GOWA mengirim data, dia menghitung "sidik
jari" dari isi pesan + kata sandi itu, lalu menempelkannya di header. Backend
menghitung sidik jari yang sama; kalau cocok, berarti pengirimnya benar-benar
GOWA. Kalau tidak cocok, ditolak.

`hash_equals` dipakai (bukan `==`) supaya perbandingan memakan waktu yang sama
apa pun hasilnya — mencegah serangan yang menebak sidik jari huruf per huruf.
Ini detail keamanan yang sering dilupakan pengembang pemula; di sini sudah
benar.

**Langkah 3 — Menyaring pesan**

```php
$event = $request->input('event');
if ($event !== 'message') {
    return response()->json(['status' => 'ignored'], 200);
}
...
if (($payload['is_from_me'] ?? false) === true) {
    return response()->json(['status' => 'ignored'], 200);
}
```
Abaikan kejadian selain pesan masuk, dan abaikan pesan yang dikirim oleh kita
sendiri (kalau tidak, sistem akan membaca pesan pengingatnya sendiri).

**Langkah 4 — Membaca isi balasan**

`app/Services/KonfirmasiReplyParser.php:16-43`:
```php
public function parse(string $text): ?array
{
    $value = Str::of($text)->trim()->squish()->toString();

    // (1) Kalau pesan diawali "LAPORAN", itu bukan konfirmasi -> lewati
    if ($value === '' || preg_match('/^\s*LAPORAN\b/i', $value) === 1) {
        return null;
    }

    // (2) Kata-kata yang berarti TIDAK BISA HADIR
    if (preg_match('/berhalangan|tidak bisa|tidak dapat|izin|sakit/i', $value) === 1) {
        return [
            'status' => KonfirmasiStatus::Berhalangan,
            'alasan' => Str::limit($value, self::AlasanLimit, ''),
        ];
    }

    // (3) Kata-kata yang berarti SIAP
    if (preg_match('/(?:^|\s)(siap|hadir|sedia)(?:\s|$)/i', $value) === 1) {
        return [
            'status' => KonfirmasiStatus::Hadir,
            'alasan' => null,
        ];
    }

    // (4) Tidak dikenali -> jangan tebak-tebak
    return null;
}
```

**Penjelasan awam:**
- `preg_match` = mencari pola di dalam teks. Tanda `/i` di ujung berarti "tidak
  peduli huruf besar/kecil".
- Urutan pemeriksaan **penting**: "berhalangan" dicek **duluan**. Kalau tidak,
  kalimat "saya tidak bisa hadir" akan salah dibaca sebagai "hadir".
- `\b` dan `(?:^|\s)` memastikan kata dicari sebagai kata utuh — supaya
  "siapa" tidak salah dibaca sebagai "siap".
- Kalau tidak ada kata kunci yang cocok, fungsi mengembalikan `null` = "saya
  tidak yakin, jangan ubah apa-apa". Sikap yang benar.

**Langkah 5 — Mencocokkan pengirim dengan petugas**

`app/Services/KonfirmasiReplyService.php:41-66`:
```php
$sender = $this->numberNormalizer->normalize(data_get($message, 'from'));
if ($sender === null) {
    return false;
}

$petugas = Petugas::query()->where('whatsapp', $sender)->first();
if ($petugas === null) {
    return false;
}

$konfirmasi = KonfirmasiPetugas::query()
    ->where('petugas_id', $petugas->id)
    ->where('status_konfirmasi', KonfirmasiStatus::Menunggu)
    ->where('tanggal_piket', '>=', today())
    ->orderBy('tanggal_piket')
    ->orderBy('jenis_piket_id')
    ->first();
```

Baca: *"Normalkan nomor pengirim. Cari petugas dengan nomor itu. Lalu cari
konfirmasi miliknya yang masih 'Menunggu' untuk tanggal hari ini atau nanti,
ambil yang tanggalnya paling dekat."*

> **Keterbatasan yang perlu Anda tahu:** kalau seorang petugas punya **dua**
> jadwal berbeda yang sama-sama menunggu konfirmasi, balasan "SIAP" akan
> otomatis dikenakan pada yang **tanggalnya paling awal**. Petugas tidak bisa
> memilih. Untuk operasional Lapas ini biasanya cukup, tapi layak dicatat.

**Langkah 6 — Menyimpan konfirmasi**

`app/Models/KonfirmasiPetugas.php:87-110`:
```php
public function confirm(KonfirmasiStatus $status, ?string $alasan, KonfirmasiMetode $metode = KonfirmasiMetode::Manual): self
{
    $updated = self::query()
        ->whereKey($this->getKey())
        ->where('status_konfirmasi', KonfirmasiStatus::Menunggu)   // <- syarat
        ->update([
            'status_konfirmasi' => $status,
            'waktu_konfirmasi' => now(),
            'alasan' => $status === KonfirmasiStatus::Berhalangan ? $alasan : null,
            'metode' => $metode,
        ]);

    if ($updated === 0) {
        throw ValidationException::withMessages([
            'status' => 'Konfirmasi yang telah selesai tidak dapat diubah.',
        ]);
    }

    return $this->refresh()->loadForApi();
}
```

**Teknik yang dipakai di sini bernama *conditional update*.** Perhatikan
`->where('status_konfirmasi', KonfirmasiStatus::Menunggu)` di dalam perintah
update. Artinya: *"Ubah baris ini, TAPI hanya kalau statusnya masih Menunggu."*
Database mengembalikan berapa baris yang berubah. Kalau 0, berarti ada orang
lain yang sudah mengubahnya duluan → lempar error.

Ini mencegah kondisi balapan (*race condition*): operator dan balasan WhatsApp
mengubah konfirmasi yang sama pada detik yang sama. Yang duluan menang, yang
kedua dapat pesan error jelas.

Perhatikan juga: `alasan` hanya disimpan kalau statusnya Berhalangan. Kalau
"Hadir", alasan dipaksa `null`. Data bersih.

### Konfirmasi manual oleh operator

Halaman `/konfirmasi` → `KonfirmasiView.tsx`. Operator bisa mengubah status
sendiri lewat `PUT /konfirmasi/{id}`. Metodenya tercatat sebagai `Manual`,
sehingga bisa dibedakan dari yang otomatis dari WhatsApp.

Ada juga lonceng notifikasi di header:
`KonfirmasiNotificationBadge.tsx`, dipasang di `App.tsx:126`. Riwayat git
menyebut fitur ini memakai *background polling* — frontend menanyakan data
konfirmasi secara berkala.

---

## ALUR H — Presensi Scan QR (inti yang Anda tanyakan)

Ini alur yang Anda sebut sebagai inti proyek. Mari kita bedah paling detail.

### H.1 Cerita dari sisi petugas

1. Petugas datang ke pos jaga.
2. Di meja ada laptop/PC yang layarnya menampilkan kiosk SILAPONTI (layar biru
   dengan logo Kemenimipas, tulisan "Scan kehadiran petugas").
3. Petugas menempelkan kartu ber-QR ke kamera **atau** menempelkannya ke
   scanner USB.
4. Layar menampilkan NIP yang terbaca dan pesan **"Presensi berhasil dicatat."**
   dengan tanda centang hijau, lengkap dengan jam.
5. Selesai. Petugas tidak login, tidak mengetik apa pun.

### H.2 Apa yang sebenarnya ada di dalam QR

**Hanya NIP.** Bukan token, bukan URL, bukan data terenkripsi. Cukup deretan
angka NIP petugas.

Buktinya di `sijempol-scan/src/App.tsx:19-20`:
```typescript
const handleScanSuccess = async (decodedText: string) => {
  const nip = decodedText.trim();
```
Teks yang dibaca dari QR langsung dianggap sebagai NIP (setelah dibuang spasi
di depan/belakang).

Dan di backend, `KehadiranPiket.php:127`:
```php
$petugas = Petugas::query()->where('nip', $nip)->firstOrFail();
```

> **INI TEMUAN PENTING UNTUK ANDA.** Tidak ada satu pun kode di ketiga proyek
> yang **membuat/mencetak** QR code. Saya sudah mencari `qr`, `QR`, `barcode`
> di seluruh frontend dan backend — hasilnya nihil kecuali komponen **pembaca**.
> Artinya:
>
> - Kartu QR petugas harus dibuat **di luar aplikasi** (misalnya lewat
>   generator QR online, atau Excel + add-in barcode, lalu dicetak).
> - Belum ada tombol "Cetak Kartu Petugas" di aplikasi.
> - Ini kandidat kuat untuk fitur tambahan yang sangat berguna.

### H.3 Konsekuensi keamanan dari desain ini

Karena isi QR hanya NIP, dan endpoint scan tidak butuh login, maka:

- Siapa pun yang tahu NIP seseorang **bisa mengabsenkan orang itu** — cukup
  membuat QR sendiri, atau bahkan mengetik NIP di keyboard kiosk lalu Enter.
- Perlindungan yang ada saat ini hanya: (a) rate limit 60 permintaan per menit
  per IP, (b) CORS yang membatasi asal permintaan browser.

Apakah ini bug? **Tidak persis** — ini *trade-off* yang disengaja. Bukti
keputusan sadar ada di komentar `routes/api.php:30`:
```php
// Scanner kiosk submits the NIP encoded in a QR/barcode. It is deliberately
// separate from the operator's authenticated manual attendance endpoints.
Route::post('/kehadiran/scan', [KehadiranController::class, 'scan'])->middleware('throttle:scan');
```
*"Sengaja dipisah dari endpoint presensi manual yang butuh login."*

Yang mengurangi risikonya di dunia nyata: kiosk berada di dalam area Lapas yang
terkontrol, dan setiap scan tercatat lengkap dengan waktu sehingga bisa diaudit.
Tapi Anda tetap perlu tahu ini. Saya bahas mitigasinya di bagian 29.

### H.4 Langkah demi langkah di sisi kiosk

`sijempol-scan/src/App.tsx:19-55` — mari kita baca seluruh fungsinya.

```typescript
const handleScanSuccess = async (decodedText: string) => {
  const nip = decodedText.trim();
  if (!nip) {
    return;                                    // (1) kosong -> berhenti
  }

  const newItem: ScanResult = {                // (2) buat entri riwayat
    id: crypto.randomUUID(),
    text: nip,
    timestamp: new Date(),
    status: 'pending',
    message: 'Memproses presensi...',
  };

  const latestScan = latestScanRef.current;    // (3) cegah scan ganda
  if (latestScan?.text === nip && newItem.timestamp.getTime() - latestScan.timestamp < 3000) {
    return;
  }
  latestScanRef.current = { text: nip, timestamp: newItem.timestamp.getTime() };
  setScannedItems(previous => [newItem, ...previous]);   // (4) tampilkan langsung

  try {
    const response = await fetch(`${API_BASE_URL}/kehadiran/scan`, {   // (5) kirim
      method: 'POST',
      headers: { 'Content-Type': 'application/json', Accept: 'application/json' },
      body: JSON.stringify({ nip }),
    });
    const payload = await response.json().catch(() => null);
    const message = payload?.message
      || (response.ok ? 'Presensi berhasil dicatat.' : 'Presensi tidak dapat diproses.');

    setScannedItems(previous => previous.map(item => item.id === newItem.id   // (6) perbarui
      ? { ...item, status: response.ok ? 'success' : 'error', message }
      : item));
  } catch {
    setScannedItems(previous => previous.map(item => item.id === newItem.id   // (7) offline
      ? { ...item, status: 'error', message: 'Tidak dapat terhubung ke server SILAPONTI - KAMTIB.' }
      : item));
  }
};
```

**Penjelasan tiap bagian:**

**(1)** Kalau QR terbaca tapi isinya kosong, langsung berhenti.

**(2)** Membuat "kartu riwayat" dengan status `pending` dan pesan "Memproses
presensi...". `crypto.randomUUID()` membuat ID acak agar entri ini bisa
ditemukan lagi nanti untuk diperbarui.

**(3)** Pencegah scan ganda dalam 3 detik (sudah dibahas di bagian 10.3).

**(4)** `setScannedItems(previous => [newItem, ...previous])` — entri baru
diletakkan **di depan** daftar (`...previous` menyalin sisanya di belakang),
sehingga scan terbaru selalu di atas. Yang penting: **tampil duluan, sebelum
server menjawab.** Petugas langsung melihat responsnya — ini disebut *optimistic
UI*, membuat aplikasi terasa cepat.

**(5)** Mengirim ke `/api/v1/kehadiran/scan`. Perhatikan: **tidak ada
`Authorization` header.** Kiosk memang tidak login.

**(6)** Setelah jawaban datang, entri yang tadi dicari berdasarkan `id`-nya,
lalu statusnya diubah jadi `success` atau `error` beserta pesan **dari server**.
`{ ...item, status: ..., message }` artinya "salin semua isi item lama, lalu
timpa dua field ini". Ini gaya penulisan React yang benar (tidak mengubah data
lama, tapi membuat salinan baru).

**(7)** `catch` menangani kondisi jaringan mati — pesannya beda dan spesifik.

### H.5 Langkah demi langkah di sisi backend

**Rute:** `sijempol-be/routes/api.php:32`
```php
Route::post('/kehadiran/scan', [KehadiranController::class, 'scan'])->middleware('throttle:scan');
```
Perhatikan rute ini berada **di luar** blok `Route::middleware(['auth:sanctum', ...])`.
Itulah sebabnya tidak perlu login.

**Validasi:** `app/Http/Requests/Kehadiran/ScanKehadiranRequest.php` memastikan
field `nip` ada dan berupa teks.

**Controller:** `app/Http/Controllers/Api/KehadiranController.php:47-61`
```php
public function scan(ScanKehadiranRequest $request): JsonResponse
{
    $kehadiran = KehadiranPiket::recordScanByNip($request->validated('nip'));

    if ($kehadiran === []) {
        return ApiResponse::error('Tidak ada jadwal piket untuk NIP ini pada hari ini.', status: 422);
    }

    return ApiResponse::success(
        KehadiranResource::collection($kehadiran)->resolve($request),
        'Presensi dari scan berhasil dicatat.',
    );
}
```
Controllernya tipis — hanya memanggil model dan membentuk jawaban. Bagus.

**Otak sebenarnya:** `app/Models/KehadiranPiket.php:119-157`

```php
/**
 * Records presence for every assignment the scanned officer has today.
 * Re-scanning is idempotent and preserves an existing manual correction.
 */
public static function recordScanByNip(string $nip): array
{
    return DB::transaction(function () use ($nip): array {
        // (1) Cari petugas berdasarkan NIP
        $petugas = Petugas::query()->where('nip', $nip)->firstOrFail();

        // (2) Tentukan "hari ini" menurut zona waktu Pontianak
        $today = now(config('app.timezone'))->toDateString();

        // (3) Ambil SEMUA tugas petugas ini hari ini
        $assignments = JadwalAssignment::query()
            ->where('petugas_id', $petugas->id)
            ->whereDate('tanggal', $today)
            ->get();

        // (4) Tidak ada tugas -> kembalikan array kosong
        if ($assignments->isEmpty()) {
            return [];
        }

        // (5) Untuk setiap tugas, catat kehadiran
        return $assignments->map(function (JadwalAssignment $assignment) use ($petugas, $today): self {
            // (5a) Sudah ada catatan? pakai yang lama, JANGAN timpa
            $kehadiran = self::query()
                ->where('petugas_id', $petugas->id)
                ->where('jenis_piket_id', $assignment->jenis_piket_id)
                ->whereDate('tanggal_piket', $today)
                ->first();

            // (5b) Belum ada -> buat baru
            if ($kehadiran === null) {
                $kehadiran = self::query()->create([
                    'petugas_id' => $petugas->id,
                    'jenis_piket_id' => $assignment->jenis_piket_id,
                    'tanggal_piket' => $today,
                    'status_kehadiran' => KehadiranStatus::Hadir,
                    'waktu_presensi' => now(config('app.timezone')),
                    'keterangan' => 'Presensi otomatis melalui SIJEMPOL Scan.',
                ]);
            }

            return $kehadiran->loadForApi();
        })->all();
    });
}
```

**Empat hal cerdas dari fungsi ini:**

**Pertama — satu scan bisa mencatat banyak presensi.**
Kalau Pak Budi hari ini punya dua tugas (misalnya "Regu Pengamanan" dan
"Perwira Piket"), satu kali scan mencatat **keduanya**. Petugas tidak perlu
scan dua kali. Ini terlihat dari `$assignments->map(...)` yang berjalan untuk
setiap tugas.

**Kedua — scan ulang tidak merusak data (idempotent).**
Bagian (5a) mencari catatan yang sudah ada **sebelum** membuat baru. Kalau
sudah ada, dipakai yang lama. Jadi:
- Petugas scan jam 07:00 → tercatat Hadir jam 07:00
- Petugas iseng scan lagi jam 07:05 → **tetap** jam 07:00, tidak berubah

Ini yang dimaksud komentar *"Re-scanning is idempotent"*.

**Ketiga — koreksi manual dilindungi.**
Komentar aslinya: *"preserves an existing manual correction"*. Skenarionya:
1. Petugas scan → tercatat "Hadir"
2. Operator sadar ternyata dia telat, mengoreksi jadi "Izin" dengan keterangan
3. Petugas scan lagi → **koreksi operator tidak tertimpa**, tetap "Izin"

Kalau kode ini memakai `updateOrCreate` (yang lebih pendek dan sekilas lebih
"pintar"), koreksi operator akan hilang. Pilihan di sini lebih benar.

**Keempat — semua di dalam transaksi.**
`DB::transaction(...)` membungkus semuanya. Kalau petugas punya 3 tugas dan
pencatatan tugas ketiga gagal, ketiganya batal — tidak ada presensi setengah
jadi.

**Dan satu perlindungan lagi di lapisan database:**
Migration `2026_08_09_000012_create_kehadiran_piket_table.php`:
```php
$table->unique(['petugas_id', 'jenis_piket_id', 'tanggal_piket']);
```
Bahkan kalau semua logika PHP di atas gagal (misalnya karena dua permintaan
tiba pada milidetik yang sama), database **menolak** baris kedua. Ini lapisan
terakhir yang tidak bisa dibobol.

Jadi ada **tiga lapis** pencegah presensi ganda:
1. Frontend: dedup 3 detik
2. Backend: cek-dulu-sebelum-buat
3. Database: unique constraint

Ini pola *defense in depth* yang diterapkan dengan benar.

### H.6 Skenario dan pesan yang muncul

| Situasi | Yang terjadi | Pesan di layar kiosk |
| --- | --- | --- |
| NIP terdaftar, ada tugas hari ini | Presensi dibuat, status Hadir | "Presensi dari scan berhasil dicatat." (hijau) |
| NIP terdaftar, **tidak ada tugas** hari ini | Tidak ada yang disimpan | "Tidak ada jadwal piket untuk NIP ini pada hari ini." (merah) |
| NIP **tidak ada** di database | `firstOrFail()` melempar error | "Data tidak ditemukan." (dari `bootstrap/app.php`, HTTP 404) |
| Scan ulang dalam 3 detik | Diabaikan di frontend | Tidak muncul apa-apa |
| Scan ulang setelah 3 detik | Backend memakai catatan lama | "Presensi dari scan berhasil dicatat." (data tidak berubah) |
| Server mati / jaringan putus | `catch` di frontend | "Tidak dapat terhubung ke server SILAPONTI - KAMTIB." |
| Lebih dari 60 scan/menit dari 1 IP | Rate limiter menolak | "Terlalu banyak percobaan. Silakan coba lagi nanti." (HTTP 429) |

### H.7 Catatan pemasangan kiosk

**Masalah kamera dan HTTPS.** Browser modern **melarang** akses kamera kalau
halaman dibuka lewat HTTP biasa — kecuali alamatnya `localhost`. Jadi:

| Cara pasang | Kamera jalan? |
| --- | --- |
| Buka `http://localhost:3005` di PC yang sama | Ya |
| Buka `http://192.168.1.50:3005` dari PC lain | **Tidak** |
| Buka `https://silaponti.lapas.go.id` (ada sertifikat) | Ya |

Karena itu proyek scan memasang `@vitejs/plugin-basic-ssl` — untuk menyediakan
HTTPS lokal saat pengembangan. Untuk produksi, Anda perlu sertifikat SSL asli
atau menjalankan kiosk di komputer yang sama dengan servernya.

**Solusi tanpa HTTPS:** pakai **scanner USB**. Alat itu tidak butuh izin kamera
sama sekali karena berperilaku sebagai keyboard. Ini kemungkinan besar cara
yang dipakai di lapangan, dan kode sudah mendukungnya penuh (lihat bagian 10.2).

**Konfigurasi alamat backend** (`sijempol-scan/vite.config.ts:14-21`):
```typescript
server: {
  host: true,
  port: 3005,
  strictPort: true,
  proxy: {
    '/api': {
      target: env.SIJEMPOL_API_PROXY_TARGET || 'http://127.0.0.1:8000',
      changeOrigin: true,
    },
  },
}
```
Saat pengembangan, semua permintaan ke `/api` diteruskan ke backend. Karena
alamat di browser tetap `/api/v1` (satu asal), **tidak terjadi masalah CORS**.

Untuk produksi, `VITE_API_BASE_URL` di-set saat build (lihat
`sijempol-scan/docker-compose.yml`), dan backend harus mengizinkan asal kiosk
lewat `SCAN_FRONTEND_URL` di `.env` backend
(`sijempol-be/config/cors.php:22-25`):
```php
'allowed_origins' => array_values(array_filter([
    env('FRONTEND_URL', 'http://localhost:3000'),
    env('SCAN_FRONTEND_URL', 'http://localhost:5173'),
])),
```

> **Perhatikan ketidakcocokan kecil:** `.env.example` backend menulis
> `SCAN_FRONTEND_URL=http://localhost:5173`, tapi kiosk sebenarnya berjalan di
> **port 3005**. Kalau nanti Anda menjalankan kiosk dari komputer berbeda dan
> muncul error CORS, inilah penyebabnya. Perbaiki jadi
> `SCAN_FRONTEND_URL=http://localhost:3005` (atau alamat produksinya).

---

## ALUR I — Presensi Manual dan Koreksi

Halaman `/kehadiran` → `sijempol-fe/src/features/kehadiran/components/KehadiranView.tsx`

### Kenapa perlu manual?

Beberapa kasus yang tidak bisa ditangani scan:
- Petugas sakit / izin → tidak datang, tidak bisa scan
- Petugas alpa → juga tidak scan
- Kartu QR hilang atau kiosk rusak
- Koreksi jam presensi

### Formulir input manual

`KehadiranView.tsx:14-20`:
```typescript
const [selectedPetugasId, setSelectedPetugasId] = useState('');
const [selectedJenisPiketId, setSelectedJenisPiketId] = useState('');
const [selectedDate, setSelectedDate] = useState(() => new Date().toLocaleDateString('en-CA'));
const [selectedStatus, setSelectedStatus] = useState<'Hadir' | 'Sakit' | 'Izin' | 'Alpa'>('Hadir');
const [waktuPresensi, setWaktuPresensi] = useState(() => new Date().toLocaleTimeString('en-GB', { hour12: false }));
const [keterangan, setKeterangan] = useState('');
```

Dua trik format tanggal yang perlu Anda kenali:
- `toLocaleDateString('en-CA')` menghasilkan `2026-08-24` — format yang diminta
  backend. Trik yang cerdas: locale Kanada kebetulan memakai format ISO.
- `toLocaleTimeString('en-GB', { hour12: false })` menghasilkan `14:30:00` —
  format 24 jam.

### Filter petugas yang terjadwal

`KehadiranView.tsx:23-27`:
```typescript
const scheduledPetugasIds = new Set(
  (jadwal[selectedJenisPiketId] || [])
    .filter(assignment => assignment.tanggal === selectedDate)
    .map(assignment => assignment.petugasId),
);
```
Setelah operator memilih jenis piket dan tanggal, dropdown petugas hanya
menampilkan orang yang **memang terjadwal** hari itu. `Set` dipakai karena
pengecekan "ada tidak di dalam daftar" jauh lebih cepat daripada array biasa.

### Mengirim ke backend

`KehadiranView.tsx:41-49`:
```typescript
await onTakeAttendance({
  petugasId: pet.id,
  jenisPiketId: piket.id,
  tanggalPiket: selectedDate,
  statusKehadiran: selectedStatus,
  waktuPresensi: selectedStatus === 'Hadir' ? waktuPresensi.padEnd(8, ':00') : undefined,
  keterangan: keterangan || 'Presensi Manual Operator'
});
```
Perhatikan `selectedStatus === 'Hadir' ? waktuPresensi : undefined` — jam
presensi hanya dikirim kalau statusnya Hadir. Orang yang Sakit/Izin/Alpa tidak
punya jam presensi. Backend juga menegakkan aturan yang sama
(`KehadiranPiket.php:211-226`):
```php
private static function presensiTimestamp(string $tanggalPiket, KehadiranStatus $status, ?string $waktuPresensi): ?Carbon
{
    if ($status !== KehadiranStatus::Hadir || $waktuPresensi === null) {
        return null;
    }

    return Carbon::createFromFormat('Y-m-d H:i:s', "{$tanggalPiket} {$waktuPresensi}", config('app.timezone'));
}
```
Aturan yang sama ditegakkan di dua tempat — frontend untuk kenyamanan, backend
untuk kebenaran.

Dan `keterangan` otomatis diisi "Presensi Manual Operator" kalau kosong,
sehingga bisa dibedakan dari "Presensi otomatis melalui SIJEMPOL Scan."

### Koreksi

`PUT /kehadiran/{id}` → `KehadiranPiket::correctFromPayload()`
(`KehadiranPiket.php:166-181`). Aturan validasi dan hak akses sama persis
dengan pencatatan awal.

---

## ALUR J — Laporan Pengawalan + Cetak PDF

Halaman: `/laporan/pengawalan`

### Alur halaman (enam mode dalam satu komponen)

`App.tsx:170-175` — perhatikan ada enam rute untuk satu komponen:
```typescript
<Route path={APP_PATHS.laporanPengawalan} element={<LaporanPengawalanView mode="list" />} />
<Route path={`${APP_PATHS.laporanPengawalan}/baru`} element={<LaporanPengawalanView mode="create" />} />
<Route path={`${APP_PATHS.laporanPengawalan}/baru/preview`} element={<LaporanPengawalanView mode="create-preview" />} />
<Route path={`${APP_PATHS.laporanPengawalan}/:laporanId`} element={<LaporanPengawalanView mode="detail" />} />
<Route path={`${APP_PATHS.laporanPengawalan}/:laporanId/edit`} element={<LaporanPengawalanView mode="edit" />} />
<Route path={`${APP_PATHS.laporanPengawalan}/:laporanId/edit/preview`} element={<LaporanPengawalanView mode="edit-preview" />} />
```
Satu komponen, enam tampilan, dibedakan lewat properti `mode`. Adanya mode
**preview** artinya operator bisa melihat dulu bentuk suratnya sebelum
menyimpan.

### Isi laporan

Data yang dikumpulkan (dari migration `2026_08_09_000013` + migration lanjutan):

**Bagian surat:**
- Nomor surat perintah (**unik** — tidak boleh dobel)
- Nomor berita acara, nomor surat bantuan
- Tanggal surat, perihal
- Dasar pengawalan (daftar poin)
- Menimbang (daftar poin, migration `2026_08_11_000025`)
- Tujuan pengawalan (migration `2026_08_11_000027`)

**Identitas WBP:**
- Nama, nomor register, umur, jenis kelamin
- Perkara, pidana
- Tanggal setengah masa pidana, tanggal ekspirasi
- Alamat

**Kategori pengawalan** (`app/Enums/KategoriPengawalan.php`):
| Nilai |
| --- |
| Pengawalan Peradilan |
| Pengawalan Kepentingan Medis |
| Pengawalan Pemindahan/Mutasi WBP |
| Pengawalan Asimilasi/Izin Kerja Luar |
| Pengawalan Izin Luar Biasa |

**Petugas:** `petugas_pendamping_json` dan `petugas_pemberi_perintah_json`.

**Status** (`app/Enums/LaporanPengawalanStatus.php`): Draft, Menunggu
verifikasi, Disetujui, Selesai, Dibatalkan.

### Perlindungan nomor surat duplikat

Migration `2026_08_09_000013_create_laporan_pengawalan_table.php` punya
pemeriksaan yang tidak biasa:
```php
if (Schema::hasTable('laporan_pengawalan')) {
    $hasDuplicateNumber = DB::table('laporan_pengawalan')
        ->select('no_surat_perintah')
        ->groupBy('no_surat_perintah')
        ->havingRaw('COUNT(*) > 1')
        ->exists();

    if ($hasDuplicateNumber) {
        throw new RuntimeException(
            'Nomor surat perintah duplikat harus diperbaiki sebelum migration laporan pengawalan dijalankan.',
        );
    }
}
```
Kalau ada data lama dengan nomor surat kembar, migration **berhenti dengan
pesan jelas** alih-alih gagal misterius saat memasang unique constraint. Detail
kecil yang menunjukkan pengembang sebelumnya cukup teliti.

### Template Pengawalan

Menu Pengaturan → Template Pengawalan (`/template-pengawalan`, **Admin saja**).
Tabel `template_pengawalan` (migration `2026_08_11_000028`) menyimpan teks
standar "Dasar" dan "Menimbang" supaya operator tidak perlu mengetik ulang
setiap kali.

Cara pakainya terlihat di `LaporanPengawalan.php` lewat method
`resolvedDasar()` dan `resolvedMenimbang()` — kalau laporan punya isian
sendiri, itu yang dipakai; kalau tidak, ambil dari template.

### Cetak PDF

`GET /laporan-pengawalan/{id}/pdf` → `LaporanPengawalanController::downloadPdf`

Prosesnya:
1. `app/Services/LaporanPengawalanDocumentService.php` merangkai semua data
   menjadi satu array siap cetak.
2. Array itu diberikan ke template Blade
   `resources/views/pdf/laporan-pengawalan.blade.php`.
3. Paket `dompdf` mengubah HTML hasilnya jadi file PDF.
4. PDF dikirim ke browser sebagai unduhan.

**Kop surat** dibuat di `LaporanPengawalanDocumentService.php:28-37`:
```php
'header' => [
    'logo' => $this->logoDataUri(),
    'institution' => 'KEMENTERIAN IMIGRASI DAN PEMASYARAKATAN REPUBLIK INDONESIA',
    'directorate' => 'DIREKTORAT JENDERAL PEMASYARAKATAN',
    'regionalOffice' => 'KANTOR WILAYAH KALIMANTAN BARAT',
    'office' => 'LEMBAGA PEMASYARAKATAN KELAS IIA PONTIANAK',
    'address' => 'Jalan Adisucipto, Km.06, Desa Sungai Raya, Kecamatan Sungai Raya',
    'contact' => 'Laman: lapaspontianak.kemenmipas.go.id  Surel: lapas2a_pontianak@yahoo.co.id',
],
```

> **Ini di-hardcode** (ditulis langsung di kode). Kalau instansi pindah alamat
> atau ganti email, harus mengubah kode dan deploy ulang. Untuk satu instansi
> ini masih wajar; kalau nanti dipakai Lapas lain, perlu dipindah ke database.

`logoDataUri()` mengubah file `resources/images/laporan-pengawalan-logo.png`
menjadi teks base64 yang ditempel langsung di HTML — supaya dompdf tidak perlu
mengunduh gambar dari internet.

**Pemotongan teks** — perhatikan pemakaian `$this->text($nilai, 240)`:
```php
'dasar' => array_map(
    fn (string $item): string => $this->text($item, 240),
    $laporanPengawalan->resolvedDasar(),
),
```
Setiap teks dibatasi panjangnya (240, 160, 300, 420 karakter tergantung
posisinya di surat) supaya tata letak PDF tidak berantakan kalau operator
mengetik terlalu panjang. Praktis.

### Cetak PDF Jadwal

Ada juga `GET /jadwal/{jenisPiket}/pdf` →
`app/Services/JadwalPdfService.php` (341 baris) → template
`resources/views/pdf/jadwal-piket.blade.php`.

Ini mencetak **matriks jadwal** dalam bentuk tabel lebar berkop surat, dengan
kode shift huruf besar per sel (`JadwalPdfService.php:41-44`):
```php
->map(fn (Collection $sameDay): string => $sameDay
    ->map(fn ($assignment): string => $assignment->shift ? mb_strtoupper($assignment->shift, 'UTF-8') : 'X')
    ->unique()
    ->join('/'))
```
Kalau satu orang punya dua shift di hari yang sama, kodenya digabung dengan
garis miring, misal `P/M`. Kalau tidak ada kode shift, dicetak `X`.

---

## ALUR K — Laporan Penggeledahan + Upload Foto

Halaman: `/laporan/penggeledahan`. Formulirnya adalah komponen **terbesar
kedua** di frontend: `LaporanPenggeledahanForm.tsx` (1.037 baris).

### Isi laporan

- Nomor laporan (**unik**)
- Tanggal, waktu mulai & selesai
- **Area penggeledahan** (`area_penggeledahan_json`) — struktur bertingkat:
  daftar blok, masing-masing berisi daftar kamar
- Blok & kamar (kolom lama, kini jadi cermin dari area pertama — lihat
  migration `2026_08_10_000026_repair_laporan_penggeledahan_area_schema.php`)
- Lokasi spesifik
- Dasar pelaksanaan
- Petugas penggeledah (JSON)
- **Barang sitaan** (JSON) atau centang "tidak ada barang"
- Dipimpin oleh, anggota
- **Dokumentasi foto** (JSON berisi daftar path)
- Status: Draft / Menunggu verifikasi / Disetujui / Selesai / Dibatalkan

### Alur upload foto — urutannya penting

Ini pola yang perlu Anda pahami:

```
1. Operator mengisi formulir dan klik Simpan
       |
       v
2. POST /laporan-penggeledahan          -> laporan dibuat, dapat ID
       |
       v
3. UNTUK SETIAP FOTO:
   POST /laporan-penggeledahan/{ID}/foto  -> upload satu per satu (multipart)
       |
       v
4. Path foto disimpan ke dokumentasi_foto_json
```

**Kenapa laporan dibuat dulu, baru foto?** Karena file harus "menempel" pada
sesuatu. Kalau foto diupload duluan, sistem perlu menyimpan file yatim piatu
yang belum jelas milik laporan mana — dan kalau operator batal menyimpan, file
itu jadi sampah.

Ini juga dinyatakan di `architecture.md`: *"alur foto membuat laporan lebih
dahulu sebelum upload multipart."*

### Aturan upload

Dari `UploadLaporanPenggeledahanFotoRequest.php` dan `task-list.md`:

| Aturan | Nilai |
| --- | --- |
| Tipe file | Hanya gambar yang tervalidasi |
| Ukuran maksimal | **5 MB per file** |
| Lokasi simpan | Disk `public`, direktori penggeledahan |
| Nama file | **Di-generate sistem**, bukan nama dari komputer operator |
| Yang disimpan di DB | **Path saja**, bukan isi file (bukan base64) |

Poin "nama file di-generate" itu penting untuk keamanan: kalau nama file dari
client dipakai apa adanya, penyerang bisa mengirim nama seperti
`../../config/database.php` untuk menimpa file sistem.

Poin "bukan base64" penting untuk performa: menyimpan gambar sebagai teks di
database membuat database membengkak dan lambat.

### Pembersihan file

`task-list.md` mencatat: *"Hapus/ganti file secara aman ketika laporan
diperbarui atau dihapus"* — sudah dicentang selesai. Logikanya ada di
`app/Services/LaporanPenggeledahanService.php` (126 baris).

### Yang BELUM ada di modul ini

> **Tidak ada endpoint cetak PDF untuk Laporan Penggeledahan.**
>
> Bandingkan daftar rute:
> - `GET /laporan-pengawalan/{id}/pdf` — **ADA**
> - `GET /jadwal/{jenisPiket}/pdf` — **ADA**
> - `GET /laporan-penggeledahan/{id}/pdf` — **TIDAK ADA**
>
> Ini kemungkinan besar salah satu bagian dari "format laporan yang belum
> ditambahkan" yang disebut teman Anda. Saya bahas lengkap di bagian 28.

---

## ALUR L — Dashboard dan Mode TV

Halaman: `/dashboard`. Satu-satunya endpoint: `GET /dashboard/summary`.

### Komponen penyusun dashboard

| File | Menampilkan |
| --- | --- |
| `HeroBanner.tsx` | Banner atas, tanggal, sambutan |
| `KpiStrip.tsx` | Deretan angka penting (total petugas, jadwal aktif, dst) |
| `TodayScheduleTile.tsx` | Siapa piket hari ini |
| `DisciplineTile.tsx` | Indikator kedisiplinan |
| `NotificationsTile.tsx` | Notifikasi terbaru |
| `WeeklyRecapTile.tsx` | Rekap mingguan |
| `SecondaryTabsPanel.tsx` | Panel tab tambahan (304 baris) |
| `SystemFooter.tsx` | Info sistem di bawah |

### Backend: `app/Services/DashboardService.php` (504 baris)

Ini file backend **terbesar**. Tugasnya menghitung semua ringkasan dalam
**satu permintaan** — bukan 10 permintaan terpisah.

**Menentukan rentang waktu** (baris 33-39):
```php
$referenceDate = Carbon::createFromFormat('Y-m-d', $tanggal)->startOfDay();
[$periodStart, $periodEnd] = $this->periodRange($referenceDate, $periode);
$weekStart = $referenceDate->copy()->startOfWeek();
$weekEnd = $referenceDate->copy()->endOfWeek();
$monthStart = $referenceDate->copy()->startOfMonth();
$monthEnd = $referenceDate->copy()->endOfMonth();
```
Dari satu tanggal acuan, dihitung awal/akhir minggu dan bulan.

**Menghitung petugas dalam SATU query** (baris 41-49):
```php
$petugas = Petugas::query()
    ->toBase()
    ->selectRaw('count(*) as total')
    ->selectRaw(
        'sum(case when status = ? then 1 else 0 end) as aktif',
        [PetugasStatus::Aktif->value],
    )
    ->first();
```
**Ini teknik penting.** Cara naif adalah: ambil semua petugas ke PHP, lalu
hitung. Kalau ada 500 petugas, itu 500 baris data dipindahkan sia-sia. Cara di
sini menyuruh **database** yang menghitung, dan hanya dua angka yang dikirim
balik: total dan jumlah yang aktif.

`sum(case when status = 'Aktif' then 1 else 0 end)` artinya: "untuk tiap baris,
kalau statusnya Aktif tambahkan 1, kalau tidak tambahkan 0; jumlahkan semua".

`toBase()` mematikan lapisan Eloquent (yang membuat objek PHP untuk tiap baris)
karena kita hanya perlu angka. Lebih hemat memori.

**Agregasi status yang dipakai berulang** (baris 51-95):
```php
$konfirmasi = $this->statusAggregate(
    KonfirmasiPetugas::query()->whereBetween('tanggal_piket', [$periodStart, $periodEnd]),
    'status_konfirmasi',
    [KonfirmasiStatus::Menunggu->value, KonfirmasiStatus::Hadir->value, KonfirmasiStatus::Berhalangan->value],
);
```
Fungsi `statusAggregate` dipakai untuk konfirmasi, kehadiran, pengingat, dan
laporan — pola yang sama, ditulis sekali. Ini praktik yang baik.

**Menghindari masalah N+1** (baris 97-107):
```php
$assignmentsOnDate = JadwalAssignment::query()
    ->select(['id', 'jenis_piket_id', 'petugas_id', 'tanggal', 'shift'])
    ->with([
        'petugas:id,nip,nama,jabatan,unit_kerja,whatsapp,status',
        'jenisPiket:id,nama,status,tipe_regu',
    ])
    ->whereDate('tanggal', $referenceDate)
    ->orderBy('jenis_piket_id')
    ->orderBy('petugas_id')
    ->get();
```
`->with([...])` disebut **eager loading**. Tanpa ini, kalau ada 50 assignment,
sistem akan bertanya ke database 1 + 50 + 50 = 101 kali (masalah "N+1"). Dengan
`with`, hanya 3 kali.

Perhatikan juga `'petugas:id,nip,nama,...'` — hanya kolom yang dibutuhkan yang
diambil, bukan seluruh baris.

Bahkan ada pengaman otomatis di `AppServiceProvider.php:52`:
```php
Model::preventLazyLoading(! $this->app->isProduction());
```
Artinya: saat pengembangan, kalau ada kode yang lupa `with()`, aplikasi
**langsung error**. Jadi bug performa ketahuan sejak awal, bukan setelah
produksi lambat. Ini disiplin engineering yang bagus.

**Fitur khusus P2U** (baris 108-122):
```php
$activeP2uAssignments = JadwalAssignment::query()
    ->select([...])
    ->with([...])
    ->where(function (Builder $query) use ($now): void {
        $query
            ->whereDate('tanggal', $now->copy()->subDay())
            ->orWhereDate('tanggal', $now);
    })
    ->whereHas('jenisPiket', fn (Builder $query): Builder => $query->where('tipe_regu', 'p2u'))
    ->orderBy('petugas_id')
    ->get();
```
Kenapa **kemarin dan hari ini**? Karena shift malam melewati tengah malam.
Petugas P2U yang mulai jam 19:00 tanggal 24 masih bertugas jam 02:00 tanggal 25.
Sistem harus memeriksa keduanya untuk tahu siapa yang **sedang** bertugas.
Detail domain yang dipikirkan dengan baik.

**Komandan regu pengamanan** (baris 124-127):
```php
$commanderPetugasIds = Regu::query()
    ->where('tipe', 'pengamanan')
    ->whereNotNull('commander_petugas_id')
    ->pluck('commander_petugas_id', 'jenis_piket_id');
```

### Mode TV

Skenarionya: layar besar di ruang kontrol yang menampilkan dashboard 24 jam,
login dengan akun khusus role `TV`.

Perlakuan khusus di frontend:

`App.tsx:118` — sidebar disembunyikan:
```typescript
{user.role !== 'TV' && <Sidebar userRole={user.role} ... />}
```

`App.tsx:129` — padding lebih rapat (supaya muat lebih banyak di layar besar):
```typescript
className={`... ${user.role === 'TV' ? 'p-2 md:p-3 lg:p-4' : 'p-4 md:p-6 lg:p-8'}`}
```

`App.tsx:64-67` — kalau akun TV mencoba membuka halaman lain, dilempar balik:
```typescript
function RequireInteractiveRole() {
  const { user } = useAuth();
  return user?.role === 'TV' ? <Navigate to={APP_PATHS.dashboard} replace /> : <Outlet />;
}
```

Dan di backend, `EnsureTvDashboardOnly` menolak semua endpoint lain dengan 403.
**Dua lapis lagi** — frontend untuk kenyamanan, backend untuk keamanan.

---

## ALUR M — Laporan Rekap Bulanan

Halaman: `/laporan/rekap` → `LaporanRekapView.tsx` (hanya 37 baris).

### Apa yang ditampilkan

Tabel dengan kolom: Petugas, NIP, Jabatan, Unit Kerja, Frekuensi Piket,
Persentase Presensi, Status Penugasan. Ada pemilih bulan dan tombol
"Cetak Rekap Bulanan".

### Bagaimana angkanya dihitung

`sijempol-fe/src/features/laporan-rekap/hooks/useLaporanRekap.ts:34-44`:
```typescript
const rows = useMemo<RekapPetugasRow[]>(() => source.petugas.map(petugas => {
  const shifts = Object.values(source.jadwal).reduce(
    (total, assignments) => total + assignments.filter(
      assignment => assignment.petugasId === petugas.id && assignment.tanggal.startsWith(selectedMonth),
    ).length,
    0,
  );
  return { ...petugas, shifts, presensiRate: shifts > 0 ? '100%' : 'N/A' };
}), [selectedMonth, source]);
```

Baca perlahan baris terakhir:
```typescript
presensiRate: shifts > 0 ? '100%' : 'N/A'
```

**Ini bukan perhitungan.** Kalau petugas punya minimal satu jadwal di bulan itu,
kolom "Persentase Presensi" **selalu menampilkan 100%** — tanpa melihat data
kehadiran sama sekali. Data tabel `kehadiran_piket` tidak dibaca di sini.

Sama halnya kolom "Status Penugasan" di `LaporanRekapView.tsx:31`, yang selalu
menulis teks tetap:
```tsx
<span className="rounded bg-slate-100 px-2 py-0.5 text-[9px] font-semibold text-slate-700">Selesai Evaluasi</span>
```

### Tombol cetak

`LaporanRekapView.tsx:16`:
```tsx
<button onClick={() => alert('Fitur cetak PDF/Excel massal sedang memproses berkas laporan rekapitulasi. Silakan periksa file unduhan sebentar lagi.')} ...>
  <Download className="h-3.5 w-3.5" /><span>Cetak Rekap Bulanan</span>
</button>
```

**Tombol ini tidak mencetak apa pun.** Dia hanya menampilkan kotak pesan yang
berbunyi seolah-olah file sedang diproses. Tidak ada file yang dibuat, tidak ada
permintaan ke server.

### Bagaimana data diambil

`laporan-rekap/api/__request.ts:10-19`:
```typescript
async load(): Promise<LaporanRekapSource> {
  const [petugas, jenisPiket] = await Promise.all([petugasApi.all(), jenisPiketApi.all()]);
  const scheduleEntries = await Promise.all(
    jenisPiket.map(async item => [item.id, await jadwalApi.get(item.id)] as const),
  );
  return { petugas, jadwal: Object.fromEntries(scheduleEntries.map(([id, detail]) => [id, detail.assignments])) };
}
```
Halaman ini mengambil **semua petugas** + **semua jenis piket** + **semua
jadwal untuk setiap jenis piket**, lalu menghitung di browser. Kalau ada 20
jenis piket, itu 20+ permintaan sekaligus setiap kali halaman dibuka.

**Tidak ada endpoint backend `/laporan/rekap`.** Backend tidak menyediakan
agregasi ini.

### Kesimpulan modul ini

Modul Laporan Rekap adalah **kerangka/mockup**, bukan fitur yang berfungsi.
Inilah bagian yang paling jelas menjelaskan pernyataan "baru 95%, yang belum
adalah format laporan". Detail lengkap ada di bagian 28.

---

# BAGIAN III — OPERASIONAL, STATUS, DAN RENCANA

---

## 25. Hak Akses per Role

### Tabel lengkap

| Fitur | Admin | Operator | TV |
| --- | :---: | :---: | :---: |
| Login & lihat profil | Ya | Ya | Ya |
| Dashboard | Ya | Ya | **Ya (hanya ini)** |
| Lihat/tambah/ubah Petugas | Ya | Ya | Tidak |
| **Hapus Petugas** | Ya | **Tidak** | Tidak |
| Lihat/tambah/ubah Jenis Piket | Ya | Ya | Tidak |
| **Hapus Jenis Piket** | Ya | **Tidak** | Tidak |
| **Ubah jam notifikasi** | Ya | **Tidak** | Tidak |
| Lihat/tambah/ubah Regu & anggota | Ya | Ya | Tidak |
| **Hapus Regu** | Ya | **Tidak** | Tidak |
| Jadwal: lihat, simpan draft, aktivasi, nonaktifkan, hapus, cetak PDF | Ya | Ya | Tidak |
| Pengingat: lihat, ubah status, kirim ulang | Ya | Ya | Tidak |
| Konfirmasi: lihat, ubah | Ya | Ya | Tidak |
| Kehadiran: lihat, catat, koreksi | Ya | Ya | Tidak |
| Laporan Pengawalan: lihat, tambah, ubah, cetak PDF | Ya | Ya | Tidak |
| **Hapus Laporan Pengawalan** | Ya | **Tidak** | Tidak |
| Laporan Penggeledahan: lihat, tambah, ubah, upload foto | Ya | Ya | Tidak |
| **Hapus Laporan Penggeledahan** | Ya | **Tidak** | Tidak |
| **Manajemen Pengguna** (buat/ubah/hapus akun) | Ya | **Tidak** | Tidak |
| **Template Pengawalan** | Ya | **Tidak** | Tidak |
| Panduan | Ya | Ya | Tidak |

### Pola yang konsisten

Perhatikan polanya: **Operator bisa membuat dan mengubah, tapi tidak bisa
menghapus.** Ini keputusan desain yang tepat untuk instansi pemerintah — data
operasional adalah dokumen resmi, penghapusan harus melalui pejabat berwenang.

Di kode, aturan ini ditulis sebagai `->middleware('admin')` di `routes/api.php`.
Contoh:
```php
Route::delete('/petugas/{petugas}', [PetugasController::class, 'destroy'])->middleware('admin');
```

### Perlindungan Admin terakhir

`app/Services/UserService.php` (133 baris) berisi *guard* khusus: sistem
**menolak** menghapus atau menonaktifkan akun Admin kalau itu satu-satunya
Admin yang tersisa. Tanpa ini, satu klik salah bisa mengunci semua orang di
luar sistem selamanya.

Ada juga akun sistem tersembunyi bernama `__system__` (disebut di
`architecture.md`) yang dipakai sebagai pemilik data arsip ketika user aslinya
dihapus, dan tidak ditampilkan di daftar pengguna.

---

## 26. Cara Menjalankan di Komputer Anda

### 26.1 Yang harus terpasang lebih dulu

| Kebutuhan | Versi | Untuk |
| --- | --- | --- |
| **PHP** | 8.3 atau lebih baru | Backend |
| **Composer** | terbaru | Manajer paket PHP |
| **PostgreSQL** | 14+ | Database |
| **Node.js** | 20 atau lebih baru | Frontend & Scan |
| **Git** | terbaru | Mengambil pembaruan kode |
| *(opsional)* GOWA | - | WhatsApp |

### 26.2 Menjalankan Backend

```bash
cd D:/Ajis/SIjempol/sijempol-be
```

**Langkah 1 — Install semua paket PHP**
```bash
composer install
```

**Langkah 2 — Buat file konfigurasi**
```bash
cp .env.example .env
```
Lalu buka `.env` dengan editor teks dan sesuaikan minimal ini:
```
DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=sijempol_be
DB_USERNAME=postgres
DB_PASSWORD=passwordAnda

FRONTEND_URL=http://localhost:3000
SCAN_FRONTEND_URL=http://localhost:3005
```

**Langkah 3 — Buat kunci enkripsi aplikasi**
```bash
php artisan key:generate
```

**Langkah 4 — Buat database kosong di PostgreSQL**
Lewat pgAdmin atau perintah: buat database bernama `sijempol_be`.

**Langkah 5 — Buat semua tabel**
```bash
php artisan migrate
```

**Langkah 6 — Isi data demo (akun login + contoh data)**
```bash
php artisan db:seed
```
Ini membuat tiga akun (dari `database/seeders/DemoDataSeeder.php`):

| Username | Password | Role |
| --- | --- | --- |
| `admin` | isi dari `SIJEMPOL_DEMO_USER_PASSWORD` (default: `password`) | Admin |
| `operator` | sama | Operator |
| `tv` | sama | TV |

> **WAJIB:** ganti password ketiga akun ini sebelum dipakai di lingkungan nyata.

**Langkah 7 — Buat link untuk foto yang diupload**
```bash
php artisan storage:link
```
Tanpa ini, foto penggeledahan tidak bisa dibuka lewat browser.

**Langkah 8 — Jalankan server**
```bash
php artisan serve
```
Backend hidup di `http://127.0.0.1:8000`.

**Langkah 9 — Jalankan worker antrean (jendela terminal TERPISAH)**
```bash
php artisan queue:work
```
> **INI SERING TERLUPA.** Tanpa worker, pengingat WhatsApp **tidak akan pernah
> dikirim** — job hanya menumpuk di tabel `jobs` dan tidak ada yang
> mengerjakannya. Kalau ada keluhan "WA tidak masuk", periksa ini dulu.

**Alternatif praktis:** jalankan semuanya sekaligus dengan
```bash
composer run serve
```
Ini menjalankan `php artisan serve` dan `php artisan queue:work` bersamaan
(lihat `composer.json`, bagian `scripts.serve`).

### 26.3 Menjalankan Frontend Admin

```bash
cd D:/Ajis/SIjempol/sijempol-fe
npm install
cp .env.example .env
npm run dev
```
Buka `http://localhost:3000` dan login dengan `admin`.

Isi `.env` frontend cukup:
```
VITE_API_BASE_URL="http://localhost:8000/api/v1"
```

### 26.4 Menjalankan Kiosk Scan

```bash
cd D:/Ajis/SIjempol/sijempol-scan
npm install
cp .env.example .env
npm run dev
```
Buka `http://localhost:3005`, lalu izinkan akses kamera saat browser meminta.

Isi `.env` scan:
```
VITE_API_BASE_URL=/api/v1
SIJEMPOL_API_PROXY_TARGET=http://127.0.0.1:8000
```
Dengan konfigurasi ini, Vite meneruskan `/api` ke backend sehingga tidak kena
CORS.

### 26.5 Menjalankan test

Backend:
```bash
php artisan test
```
Menurut `architecture.md`, ada **190 test** yang mencakup auth, master data,
jadwal, pengingat, konfirmasi, kehadiran, laporan, dashboard, dan queue.

Frontend:
```bash
npm run test
```
Menjalankan 16 file test di `src/tests/`.

Cek tipe TypeScript frontend:
```bash
npm run lint
```

Perapi kode PHP:
```bash
vendor/bin/pint
```

### 26.6 Urutan menyalakan yang benar

```
1. PostgreSQL           (paling dulu)
2. GOWA                 (kalau mau pakai WhatsApp)
3. Backend              php artisan serve
4. Worker antrean       php artisan queue:work
5. Frontend admin       npm run dev
6. Kiosk scan           npm run dev
```

---

## 27. Cara Deploy ke Server (Docker)

Ketiga proyek sudah punya `Dockerfile` dan `docker-compose.yml`.

### 27.1 Backend

`sijempol-be/docker-compose.yml` mendefinisikan **empat layanan**:

| Layanan | Isi | Tugas |
| --- | --- | --- |
| `storage-init` | image aplikasi | Sekali jalan: menyiapkan folder & izin untuk file upload |
| `app` | PHP-FPM | Menjalankan kode Laravel |
| `web` | Caddy | Web server, membuka port 8000 |
| `queue` | image aplikasi | **Worker antrean** `php artisan queue:work` |

Perhatikan bahwa `queue` sudah menjadi layanan sendiri — jadi di produksi
worker otomatis hidup. Bagus.

Perintah worker di produksi:
```yaml
command:
  - php
  - artisan
  - queue:work
  - database
  - --queue=default
  - --sleep=3
  - --tries=3
  - --timeout=60
  - --max-time=3600
```
`--max-time=3600` artinya worker berhenti sendiri setiap 1 jam lalu
dihidupkan lagi oleh Docker (`restart: unless-stopped`). Ini trik standar untuk
mencegah kebocoran memori pada proses PHP yang jalan lama.

**Yang TIDAK ada di compose ini: PostgreSQL.** Database harus disediakan
terpisah (di server yang sama atau layanan terkelola) dan alamatnya diisi di
`.env`.

### 27.2 Perintah deploy

`sijempol-be/Makefile`:
```makefile
deploy:
	git pull
	docker compose up -d --build
	docker compose exec app php artisan optimize:clear
	docker compose exec app php artisan optimize

migrate:
	docker compose exec app php artisan migrate --force

release:
	docker compose exec app php artisan sijempol:release-check
```

Cara pakai di server:
```bash
make deploy
```
Dan kalau ada perubahan struktur database:
```bash
make migrate
```

Perhatikan `migrate` **sengaja dipisah** dari `deploy` — dengan komentar
*"Jalankan terpisah jika ada perubahan skema database"*. Ini disiplin yang
benar: perubahan database harus disadari, bukan berjalan diam-diam.

### 27.3 Perintah operasional khusus

Backend punya beberapa perintah buatan sendiri di `app/Console/Commands/`:

| Perintah | File | Fungsi |
| --- | --- | --- |
| `sijempol:release-check` | `ReleaseCheckCommand.php` | Memastikan kode lolos style check & test sebelum rilis |
| `sijempol:operations-status` | `OperationsStatusCommand.php` | Menampilkan status operasional (antrean, job gagal) |
| `sijempol:archive-schedules` | `ArchiveSchedulesCommand.php` | Mengarsipkan jadwal lama |
| (kirim tes WA) | `SendGoWhatsAppTestMessage.php` | Mengirim satu pesan uji ke nomor tertentu |

Perintah terakhir sangat berguna untuk memastikan GOWA sudah tersambung sebelum
mengaktifkan jadwal sungguhan.

Ada juga ambang peringatan di `.env`:
```
OPERATIONS_PENDING_JOBS_ALERT=100
```
Kalau job menunggu lebih dari 100, `sijempol:operations-status` akan memberi
peringatan.

### 27.4 Frontend & Scan

Keduanya memakai pola build dua tahap: Node membangun file statis, lalu Nginx
menyajikannya.

`sijempol-scan/Dockerfile`:
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
ARG VITE_API_BASE_URL
ENV VITE_API_BASE_URL=${VITE_API_BASE_URL}
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
```

**Poin penting yang sering membingungkan:** `VITE_API_BASE_URL` adalah
**build argument**, bukan variabel runtime. Artinya alamat backend
"dipanggang" ke dalam file JavaScript saat build. Kalau Anda mengubah alamat
backend, Anda **harus build ulang** — mengganti `.env` saja tidak cukup.

### 27.5 Checklist sebelum go-live

Ini diambil dari `task-list.md` Fase 7 (**semuanya masih belum dicentang**):

- [ ] Audit query list/agregat, eager loading, pagination, dan index
- [ ] Verifikasi seluruh endpoint memerlukan auth dan authorization yang tepat
- [ ] Verifikasi error API tidak membocorkan stack trace, secret, token
- [ ] Jalankan `composer audit` dan tindak lanjuti temuan
- [ ] Verifikasi CORS, upload limit, public storage link, dan permission
- [ ] Siapkan proses production untuk queue worker dan layanan Go-WhatsApp
- [ ] Siapkan monitoring log, `failed_jobs`, dan prosedur resend
- [ ] Siapkan backup serta restore PostgreSQL dan storage foto
- [ ] Jalankan pengujian end-to-end lengkap
- [ ] Perbarui `architecture.md` bila implementasi final berbeda

Tambahan dari saya:

- [ ] Ganti `APP_DEBUG=true` menjadi `APP_DEBUG=false`
- [ ] Ganti `APP_ENV=local` menjadi `APP_ENV=production`
- [ ] Ganti password akun demo `admin`, `operator`, `tv`
- [ ] Isi `GOWA_WEBHOOK_SECRET` dengan string acak panjang
- [ ] Perbaiki `SCAN_FRONTEND_URL` ke port/alamat kiosk yang benar
- [ ] Pastikan `php artisan storage:link` sudah dijalankan
- [ ] Uji `sijempol:operations-status` masuk ke monitoring

---

## 28. Verifikasi "95% Selesai" — Apa yang Benar-Benar Kurang

Teman Anda bilang proyek ini 95% selesai dan yang belum adalah **format
laporan**. Saya sudah memeriksa kodenya. Berikut hasil verifikasi saya.

### 28.1 Penilaian keseluruhan: klaim itu masuk akal

Modul yang **benar-benar jadi dan terlihat matang**:

| Modul | Status | Bukti |
| --- | --- | --- |
| Auth & sesi | Selesai | Login, /me, logout, rate limit, guard role |
| Manajemen Pengguna | Selesai | CRUD + guard Admin terakhir |
| Petugas | Selesai | CRUD, filter, import massal, proteksi hapus |
| Jenis Piket | Selesai | CRUD + auto-kategori regu |
| Regu | Selesai | CRUD + sync anggota atomik |
| **Jadwal Piket** | Selesai (sangat matang) | Matriks, drag, isi cepat, validasi, draft/aktivasi/nonaktif, snapshot regu, PDF |
| **Pengingat WhatsApp** | Selesai | Queue, retry, backoff, rate limit, ACK, rekonsiliasi, resend |
| **Konfirmasi** | Selesai | Webhook + parser + guard race condition |
| **Kehadiran (scan & manual)** | Selesai | Idempotent, 3 lapis proteksi |
| Laporan Pengawalan | Selesai | CRUD + template + PDF berkop |
| Laporan Penggeledahan | Selesai **kecuali cetak** | CRUD + area bertingkat + upload foto |
| Dashboard | Selesai | Agregasi satu query, mode TV |
| **Laporan Rekap** | **BELUM — masih mockup** | Lihat di bawah |

### 28.2 Temuan 1 — Modul Laporan Rekap belum berfungsi

Ini **temuan utama** dan paling cocok dengan keterangan teman Anda.

**Bukti A — Persentase presensi tidak dihitung**
`sijempol-fe/src/features/laporan-rekap/hooks/useLaporanRekap.ts:43`
```typescript
return { ...petugas, shifts, presensiRate: shifts > 0 ? '100%' : 'N/A' };
```
Selalu "100%". Tabel `kehadiran_piket` tidak pernah dibaca.

**Bukti B — Tombol cetak tidak mencetak**
`sijempol-fe/src/features/laporan-rekap/components/LaporanRekapView.tsx:16`
```tsx
<button onClick={() => alert('Fitur cetak PDF/Excel massal sedang memproses berkas laporan rekapitulasi. Silakan periksa file unduhan sebentar lagi.')} ...>
```
Hanya menampilkan kotak pesan.

**Bukti C — Status penugasan teks tetap**
`LaporanRekapView.tsx:31` selalu menulis "Selesai Evaluasi" untuk semua baris.

**Bukti D — Tidak ada endpoint backend**
Di `routes/api.php` tidak ada `/laporan/rekap` atau sejenisnya. Frontend
menghitung sendiri dengan menarik semua data.

**Bukti E — Perhitungan tidak memperhitungkan shift libur**
`useLaporanRekap.ts:36-39` menghitung **semua** assignment, termasuk yang
shift-nya `l` (libur), `i` (istirahat), `cd` (cadangan), `lp` (lepas piket).
Jadi "Frekuensi Piket" saat ini sebenarnya "jumlah sel terisi", bukan jumlah
piket sungguhan.

### 28.3 Temuan 2 — Laporan Penggeledahan tidak bisa dicetak

Bandingkan endpoint yang ada:

| Modul | Endpoint PDF | Service | Template Blade |
| --- | --- | --- | --- |
| Jadwal Piket | `GET /jadwal/{id}/pdf` ADA | `JadwalPdfService.php` | `pdf/jadwal-piket.blade.php` |
| Laporan Pengawalan | `GET /laporan-pengawalan/{id}/pdf` ADA | `LaporanPengawalanDocumentService.php` | `pdf/laporan-pengawalan.blade.php` |
| **Laporan Penggeledahan** | **TIDAK ADA** | **TIDAK ADA** | **TIDAK ADA** |

Isi folder `resources/views/pdf/`:
```
jadwal-piket.blade.php
laporan-pengawalan.blade.php
partials/jadwal-officer-row.blade.php
```
Hanya dua. Tidak ada `laporan-penggeledahan.blade.php`.

Padahal modul penggeledahan sudah punya semua datanya — termasuk foto — dan
laporan seperti ini di Lapas biasanya **harus** dicetak dan ditandatangani.

### 28.4 Temuan 3 — Tidak ada pembuat/pencetak kartu QR

Sudah saya sebut di bagian H.2. Saya mencari kata `qr`, `QR`, `barcode` di
seluruh `sijempol-fe/src` dan `sijempol-be/app` — nihil, kecuali komentar di
`routes/api.php:30` dan komponen **pembaca** di proyek scan.

Artinya, alur "petugas dapat kartu QR" saat ini **terputus di ujung awalnya**.
Seseorang harus membuat kartu itu secara manual di luar sistem.

### 28.5 Temuan 4 — Fase 7 (Hardening) belum dikerjakan

`task-list.md` menunjukkan Fase 1-6 semuanya `[x]`, tapi seluruh Fase 7
"Hardening dan Kesiapan Rilis" masih `[ ]` (11 butir). Ini berarti sistem
**belum pernah diaudit untuk produksi**.

### 28.6 Temuan 5 — Butir "Pasca-MVP" yang mungkin diminta instansi

`task-list.md` bagian terakhir mendaftar hal-hal yang sengaja ditunda:
- Dashboard analitik lanjutan
- **Export PDF dari backend** (sebagian sudah dikerjakan belakangan)
- Approval berlapis (misal laporan harus disetujui Kepala dulu)
- Audit trail detail untuk seluruh tabel
- Dukungan multi-instansi
- Notifikasi realtime

Menariknya, enum status laporan **sudah** punya `Menunggu verifikasi` dan
`Disetujui`, tapi alur persetujuannya sendiri belum ada. Jadi statusnya bisa
diubah manual, tapi tidak ada mekanisme "kirim ke atasan untuk disetujui".

### 28.7 Kesimpulan: daftar pekerjaan "5% terakhir"

Kalau saya rangkum apa yang perlu dikerjakan untuk sampai 100%:

| No | Pekerjaan | Estimasi bobot |
| --- | --- | --- |
| 1 | **Endpoint + halaman Rekap Bulanan yang benar** (hitung kehadiran asli, filter shift non-piket, agregasi di backend) | Besar |
| 2 | **Cetak Rekap ke PDF dan/atau Excel** | Sedang |
| 3 | **Cetak PDF Laporan Penggeledahan** (service + template Blade + endpoint) | Sedang |
| 4 | **Generator & pencetak kartu QR petugas** | Sedang |
| 5 | Menyelesaikan checklist Fase 7 (audit keamanan, backup, monitoring) | Sedang |
| 6 | Perbaikan kecil: `SCAN_FRONTEND_URL`, format tanggal pesan WA, teks "SIJEMPOL Scan", hapus paket `@google/genai` | Kecil |

Butir 1, 2, dan 3 persis sesuai deskripsi *"yang belum ditambahkan adalah
format laporan"*. Jadi keterangan teman Anda **akurat**.

---

## 29. Temuan Teknis / Risiko yang Perlu Anda Tahu

Ini bukan daftar keluhan — sebagian besar adalah *trade-off* yang wajar. Tapi
sebagai pemilik baru, Anda perlu tahu semuanya.

### 29.1 Endpoint scan tidak butuh autentikasi

**Fakta:** siapa pun yang bisa menjangkau server bisa mengirim
`POST /api/v1/kehadiran/scan` dengan NIP orang lain dan mengabsenkannya.

**Kenapa begitu:** kiosk berada di pos jaga, tidak boleh menyimpan token admin.

**Mitigasi yang sudah ada:** rate limit 60/menit per IP, CORS, semua scan
tercatat waktunya.

**Saran perbaikan (pilih salah satu):**
1. Beri kiosk **API key khusus** (satu token berumur panjang dengan izin hanya
   endpoint scan). Ini paling mudah dan paling efektif.
2. Batasi endpoint scan hanya bisa diakses dari **IP jaringan internal Lapas**
   (di level Caddy/nginx, bukan aplikasi).
3. Isi QR dengan **kode bertanda tangan** (NIP + signature) alih-alih NIP polos,
   supaya orang tidak bisa membuat QR palsu.

Saran nomor 1 dan 2 bisa dikerjakan tanpa mengubah kartu QR yang sudah dicetak.

### 29.2 Worker antrean adalah titik kegagalan tunggal

Kalau `php artisan queue:work` mati (crash, server restart, lupa dijalankan),
**semua pengingat WhatsApp berhenti** tanpa pesan error apa pun ke pengguna.
Job hanya menumpuk di tabel `jobs`.

**Sudah dimitigasi sebagian:** di Docker, worker adalah layanan dengan
`restart: unless-stopped`, dan ada perintah `sijempol:operations-status`.

**Yang perlu Anda lakukan:** pasang monitoring yang memeriksa jumlah baris di
tabel `jobs` dan `failed_jobs` setiap beberapa menit, lalu memberi peringatan
kalau melewati ambang `OPERATIONS_PENDING_JOBS_ALERT`.

### 29.3 Kecepatan kirim WhatsApp sangat terbatas

Dengan `WHATSAPP_MESSAGE_INTERVAL_SECONDS=20`, sistem hanya bisa mengirim
**3 pesan per menit** atau **180 pesan per jam**.

Ini disengaja (WhatsApp memblokir pengirim massal), tapi Anda perlu
merencanakan: kalau mengaktifkan jadwal besar, pengingat H-1 harus dijadwalkan
cukup jauh sebelumnya. Untungnya, sistem sudah memakai `delay()` per pesan,
jadi antrean terdistribusi sepanjang bulan, bukan menumpuk sekaligus.

### 29.4 Ketergantungan penuh pada GOWA

GOWA adalah layanan pihak ketiga yang memakai WhatsApp secara tidak resmi
(bukan WhatsApp Business API). Risikonya:
- Nomor bisa diblokir WhatsApp
- Sesi bisa terputus dan perlu pairing ulang (scan QR WhatsApp Web)
- Update WhatsApp bisa merusak kompatibilitas

**Yang sudah bagus:** kode sudah memisahkan GOWA di balik `WhatsAppSender`
interface, jadi kalau nanti pindah ke WhatsApp Business API resmi atau SMS,
cukup membuat satu kelas baru tanpa mengubah logika jadwal.

### 29.5 Balasan WhatsApp ambigu bila petugas punya banyak jadwal

Sudah dibahas di ALUR G. `KonfirmasiReplyService.php:53-62` mengambil
konfirmasi dengan tanggal paling awal. Kalau petugas punya dua tugas di hari
yang sama untuk jenis piket berbeda, urutannya ditentukan oleh
`orderBy('jenis_piket_id')` — praktis acak dari sudut pandang pengguna.

**Saran:** cantumkan kode singkat di pesan pengingat (misal "Balas: SIAP A1")
dan ajari parser membacanya.

### 29.6 Kop surat PDF di-hardcode

`LaporanPengawalanDocumentService.php:28-37` dan `JadwalPdfService.php:56-64`
menulis nama instansi, alamat, dan email langsung di kode.

**Dampak:** perubahan alamat = ubah kode + deploy ulang.
**Saran:** pindahkan ke tabel pengaturan atau minimal ke file config.

### 29.7 Halaman Rekap menarik terlalu banyak data

`laporan-rekap/api/__request.ts:10-19` memanggil `jadwalApi.get()` untuk
**setiap** jenis piket secara paralel. Kalau ada 20 jenis piket dan
masing-masing punya 1.200 assignment, browser menarik 24.000 baris hanya untuk
menampilkan tabel rekap. Ini akan terasa lambat seiring bertambahnya data.

**Saran:** pindahkan perhitungan ke backend (yang juga menyelesaikan temuan
28.2 sekaligus).

### 29.8 Beberapa paket terpasang tapi tidak dipakai

| Proyek | Paket | Status |
| --- | --- | --- |
| BE | `livewire/livewire`, `livewire/flux` | Tidak dipakai (aplikasi ini API-only) |
| FE | `@google/genai` | Tidak dipakai sama sekali |
| FE | `express` | Hanya untuk mode produksi opsional |

Menghapusnya mempercepat build dan mengurangi permukaan serangan keamanan.
Tapi lakukan hati-hati dan uji setelahnya.

### 29.9 Rebranding belum tuntas

Teks "SIJEMPOL Scan" masih tersimpan ke database untuk setiap presensi hasil
scan (`KehadiranPiket.php:152`). Kalau instansi resmi memakai "SILAPONTI",
ini akan muncul di laporan.

### 29.10 `APP_DEBUG=true` di `.env.example`

Ini normal untuk pengembangan, tapi **berbahaya di produksi** — mode debug
menampilkan detail error termasuk struktur database dan bisa membocorkan
konfigurasi. Pastikan `.env` produksi memakai `APP_DEBUG=false`.

### 29.11 Ketidakcocokan port kiosk

`.env.example` backend: `SCAN_FRONTEND_URL=http://localhost:5173`
`vite.config.ts` kiosk: `port: 3005`

Kalau kiosk dijalankan dari komputer lain (tanpa proxy Vite), CORS akan menolak.
Perbaikan satu baris.

### 29.12 Hal-hal yang justru SANGAT BAIK

Supaya seimbang, ini yang menurut saya dikerjakan dengan sangat baik oleh
pengembang sebelumnya:

1. **Pemakaian transaksi database** di semua operasi kritis (aktivasi jadwal,
   scan, sync regu) — tidak akan ada data setengah jadi.
2. **`lockForUpdate()`** untuk mencegah dua operator merusak jadwal bersamaan.
3. **`afterCommit()`** pada dispatch job — menghindari bug klasik race
   condition antara transaksi dan worker.
4. **Idempotensi berlapis** pada presensi scan (3 lapis).
5. **Conditional update** pada konfirmasi untuk mencegah race condition.
6. **Verifikasi HMAC** pada webhook dengan `hash_equals`.
7. **`preventLazyLoading`** saat pengembangan — bug performa ketahuan dini.
8. **Pemisahan `WhatsAppSender` sebagai interface** — mudah diganti dan diuji.
9. **190 test backend + 16 file test frontend.**
10. **Dokumentasi internal yang rapi** (`architecture.md`, `task-list.md`,
    `BACKEND_README.md`, `AGENTS.md`, `ARCHITECTURE.md` di frontend).
11. **Penanganan error terpusat** yang tidak membocorkan detail teknis.
12. **Komentar kode yang menjelaskan KENAPA**, bukan sekadar APA. Contoh
    terbaik ada di `JadwalService.php:165-167` dan `KehadiranPiket.php:119-121`.

Secara keseluruhan, ini adalah proyek yang **ditulis dengan disiplin di atas
rata-rata** untuk skala aplikasi internal instansi. Anda menerima warisan yang
baik.

---

## 30. Rencana Kerja yang Saya Sarankan

### Minggu 1 — Pahami dan pastikan bisa jalan

1. Pasang PostgreSQL, PHP, Node di komputer Anda.
2. Jalankan ketiga proyek mengikuti bagian 26 sampai bisa login.
3. Jalankan `php artisan db:seed` untuk data demo.
4. **Klik semua menu satu per satu** sambil membuka dokumen ini di sebelahnya.
5. Jalankan `php artisan test` — pastikan semua lulus. Kalau ada yang gagal,
   catat; itu petunjuk pertama tentang kondisi kode.
6. Baca `sijempol-be/task-list.md` dan `sijempol-be/architecture.md`.

### Minggu 2 — Latihan membaca kode

Ikuti satu alur dari ujung ke ujung sambil membuka file-nya:
1. Alur scan QR (bagian ALUR H) — paling pendek, paling jelas.
2. Alur login (ALUR A).
3. Alur aktivasi jadwal (ALUR E) — paling rumit, tapi paling berharga.

Cara membaca yang efektif: buka file, cari nama fungsi yang saya sebut, baca
komentarnya dulu, baru kodenya.

### Minggu 3 — Perbaikan kecil (untuk membangun kepercayaan diri)

Kerjakan yang risikonya rendah dulu:
1. Perbaiki `SCAN_FRONTEND_URL` di `.env.example` jadi port 3005.
2. Ganti teks "Presensi otomatis melalui SIJEMPOL Scan." jadi "SILAPONTI Scan".
3. Perbaiki format tanggal di pesan WhatsApp jadi "Senin, 25 Agustus 2026".
4. Hapus paket `@google/genai` dari frontend, jalankan `npm run lint` untuk
   memastikan tidak ada yang rusak.

Setiap perubahan: jalankan test, lalu commit dengan pesan yang jelas.

### Minggu 4-6 — Menyelesaikan "5%"

Urutan yang saya sarankan (dari yang paling bernilai):

**1. Cetak PDF Laporan Penggeledahan** *(paling mudah, karena ada contohnya)*
- Tiru pola `LaporanPengawalanDocumentService.php`
- Buat `LaporanPenggeledahanDocumentService.php`
- Buat `resources/views/pdf/laporan-penggeledahan.blade.php` (tiru struktur
  `laporan-pengawalan.blade.php`)
- Tambah route `GET /laporan-penggeledahan/{id}/pdf`
- Tambah tombol unduh di `LaporanPenggeledahanDetail.tsx` (tiru pola
  `downloadPdf` di `jadwal-piket/api/__request.ts:101-133`)
- Tambah test

**2. Rekap Bulanan yang benar** *(paling bernilai untuk instansi)*
- Buat `app/Services/RekapService.php` yang menghitung di database:
  - jumlah piket per petugas per bulan (kecualikan shift `l`, `i`, `cd`, `lp`)
  - jumlah Hadir / Sakit / Izin / Alpa dari `kehadiran_piket`
  - persentase = Hadir / total piket terjadwal
- Buat `GET /laporan/rekap?bulan=2026-08`
- Ganti `useLaporanRekap.ts` supaya memanggil endpoint itu, bukan menghitung
  sendiri
- Tambah test dengan data campuran

**3. Cetak Rekap ke PDF/Excel**
- PDF: tiru pola yang sama seperti nomor 1
- Excel: perlu paket tambahan (`maatwebsite/excel` atau tulis CSV sederhana)
- Ganti `alert(...)` di `LaporanRekapView.tsx:16` dengan pemanggilan asli

**4. Generator kartu QR petugas**
- Backend: tambah paket QR (`simplesoftwareio/simple-qrcode` atau
  `endroid/qr-code`)
- Buat endpoint `GET /petugas/{id}/kartu` yang menghasilkan PDF kartu berisi
  nama, NIP, foto (kalau ada), dan QR berisi NIP
- Tambah juga versi massal: `GET /petugas/kartu?ids=...` untuk cetak sekaligus
- Frontend: tombol "Cetak Kartu" di `PetugasView.tsx`

### Minggu 7-8 — Hardening (Fase 7)

Kerjakan checklist di bagian 27.5. Prioritas tertinggi:
1. Amankan endpoint scan (bagian 29.1)
2. Siapkan backup otomatis PostgreSQL + folder `storage/app/public`
3. Pasang monitoring `jobs` dan `failed_jobs`
4. `APP_DEBUG=false`, ganti semua password demo
5. Uji end-to-end lengkap dengan data nyata (tapi di lingkungan uji)

### Aturan kerja yang saya sarankan

1. **Jangan pernah edit langsung di server produksi.** Selalu edit di lokal,
   test, commit, push, lalu `make deploy`.
2. **Selalu jalankan test sebelum commit.** `php artisan test` dan
   `npm run lint`.
3. **Satu commit = satu perubahan bermakna.** Pesan commit ditulis jelas.
4. **Backup database sebelum menjalankan migration baru.**
5. **Baca komentar di kode.** Pengembang sebelumnya menulis komentar yang
   menjelaskan alasan — itu emas.
6. **Kalau ragu, cari test-nya.** Test adalah dokumentasi paling jujur tentang
   apa yang seharusnya terjadi.

---

## 31. Lampiran: Daftar Lengkap Endpoint API

Semua di bawah prefix `/api/v1`. Sumber: `sijempol-be/routes/api.php`.

### Publik (tanpa login)

| Method | Path | Fungsi | Pembatas |
| --- | --- | --- | --- |
| POST | `/auth/login` | Login | 5x/menit |
| POST | `/kehadiran/scan` | **Presensi dari kiosk scan** | 60x/menit per IP |
| POST | `/webhooks/gowa/reports` | Terima pesan masuk dari GOWA | 120x/menit + verifikasi HMAC |

### Butuh login (`auth:sanctum` + `active` + `tv-dashboard-only`)

**Auth**
| Method | Path | Fungsi |
| --- | --- | --- |
| GET | `/auth/me` | Siapa saya |
| POST | `/auth/logout` | Cabut token |

**Dashboard**
| Method | Path | Fungsi |
| --- | --- | --- |
| GET | `/dashboard/summary` | Ringkasan lengkap |

**Manajemen Pengguna (Admin saja)**
| Method | Path |
| --- | --- |
| GET | `/users` |
| POST | `/users` |
| PUT | `/users/{user}` |
| DELETE | `/users/{user}` |

**Petugas**
| Method | Path | Catatan |
| --- | --- | --- |
| GET | `/petugas` | Pagination, search, filter status & unit kerja |
| POST | `/petugas` | |
| GET | `/petugas/{petugas}` | |
| PUT | `/petugas/{petugas}` | |
| DELETE | `/petugas/{petugas}` | **Admin saja** |

**Jenis Piket**
| Method | Path | Catatan |
| --- | --- | --- |
| GET | `/jenis-piket` | |
| POST | `/jenis-piket` | |
| PUT | `/jenis-piket/{jenisPiket}` | |
| PUT | `/jenis-piket/{jenisPiket}/notification-time` | **Admin saja** |
| DELETE | `/jenis-piket/{jenisPiket}` | **Admin saja** |

**Regu**
| Method | Path | Catatan |
| --- | --- | --- |
| GET | `/regu` | |
| POST | `/regu` | |
| PUT | `/regu/{regu}` | |
| PUT | `/regu/{regu}/anggota` | Ganti seluruh anggota, atomik |
| DELETE | `/regu/{regu}` | **Admin saja** |
| GET | `/kategori-regu` | Daftar kategori |

**Jadwal**
| Method | Path | Catatan |
| --- | --- | --- |
| GET | `/jadwal?jenisPiketId=...` | Ambil matriks |
| GET | `/jadwal/{jenisPiket}/pdf` | **Cetak matriks jadwal** |
| PUT | `/jadwal/{jenisPiket}` | Simpan draft |
| POST | `/jadwal/{jenisPiket}/activate` | **Aktivasi** |
| POST | `/jadwal/{jenisPiket}/deactivate` | Kembalikan ke draft |
| DELETE | `/jadwal/{jenisPiket}` | Reset |

**Pengingat**
| Method | Path | Catatan |
| --- | --- | --- |
| GET | `/pengingat` | |
| PUT | `/pengingat/{pengingat}/status` | Ubah status manual |
| POST | `/pengingat/{pengingat}/resend` | Hanya untuk yang Gagal |

**Konfirmasi**
| Method | Path |
| --- | --- |
| GET | `/konfirmasi` |
| PUT | `/konfirmasi/{konfirmasi}` |

**Kehadiran**
| Method | Path | Catatan |
| --- | --- | --- |
| GET | `/kehadiran` | |
| POST | `/kehadiran` | Presensi manual |
| PUT | `/kehadiran/{kehadiran}` | Koreksi |

**Laporan Pengawalan**
| Method | Path | Catatan |
| --- | --- | --- |
| GET | `/laporan-pengawalan` | |
| POST | `/laporan-pengawalan` | |
| GET | `/laporan-pengawalan/{id}` | |
| GET | `/laporan-pengawalan/{id}/pdf` | **Cetak surat** |
| PUT | `/laporan-pengawalan/{id}` | |
| DELETE | `/laporan-pengawalan/{id}` | **Admin saja** |
| GET | `/template-pengawalan` | Baca template |
| PUT | `/template-pengawalan` | **Admin saja** |

**Laporan Penggeledahan**
| Method | Path | Catatan |
| --- | --- | --- |
| GET | `/laporan-penggeledahan` | |
| POST | `/laporan-penggeledahan` | |
| GET | `/laporan-penggeledahan/{id}` | |
| PUT | `/laporan-penggeledahan/{id}` | |
| POST | `/laporan-penggeledahan/{id}/foto` | Upload foto, maks 5 MB |
| DELETE | `/laporan-penggeledahan/{id}` | **Admin saja** |
| ~~GET~~ | ~~`/laporan-penggeledahan/{id}/pdf`~~ | **BELUM ADA** |

**Lain-lain**
| Method | Path | Fungsi |
| --- | --- | --- |
| GET | `/up` | Health check (dipakai Docker) |

---

## 32. Lampiran: Peta File Penting

Kalau Anda hanya punya waktu membaca 15 file, baca ini:

### Backend — 8 file terpenting

| # | File | Baris | Kenapa penting |
| --- | --- | --- | --- |
| 1 | `routes/api.php` | 106 | **Peta seluruh sistem.** Semua yang bisa dilakukan aplikasi ada di sini |
| 2 | `app/Services/JadwalService.php` | 380 | Jantung sistem: draft, aktivasi, pengingat, konfirmasi |
| 3 | `app/Models/KehadiranPiket.php` | 224 | Logika scan QR (`recordScanByNip`) |
| 4 | `app/Jobs/SendWhatsAppReminderJob.php` | 127 | Pengiriman WA dengan retry & rate limit |
| 5 | `app/Services/DashboardService.php` | 504 | Semua agregasi dashboard |
| 6 | `app/Services/GoWhatsAppSender.php` | 185 | Sambungan ke GOWA |
| 7 | `bootstrap/app.php` | 111 | Middleware & penanganan error terpusat |
| 8 | `app/Providers/AppServiceProvider.php` | 116 | Rate limit & pemilihan WhatsAppSender |

### Frontend — 5 file terpenting

| # | File | Baris | Kenapa penting |
| --- | --- | --- | --- |
| 9 | `src/app/App.tsx` | 194 | Peta halaman + semua penjaga role |
| 10 | `src/shared/api/__request.ts` | 121 | Jembatan ke backend, token, error |
| 11 | `src/features/auth/hooks/useAuth.ts` | 90 | Login, restore sesi, logout |
| 12 | `src/features/jadwal-piket/components/JadwalPiketView.tsx` | 1.424 | Fitur terbesar: matriks jadwal |
| 13 | `src/features/ARCHITECTURE.md` | - | Aturan penataan kode frontend |

### Scan — 2 file

| # | File | Baris | Kenapa penting |
| --- | --- | --- | --- |
| 14 | `src/App.tsx` | 182 | Seluruh logika kiosk |
| 15 | `src/components/QRScanner.tsx` | 144 | Kontrol kamera |

### Dokumen yang sudah ada di repositori

| File | Isi |
| --- | --- |
| `sijempol-be/BACKEND_README.md` | Rancangan produk awal, 823 baris. Sangat detail |
| `sijempol-be/architecture.md` | Ringkasan arsitektur target, 314 baris |
| `sijempol-be/task-list.md` | **Checklist progres, 267 baris. Baca ini untuk tahu apa yang sudah/belum** |
| `sijempol-be/AGENTS.md` | Aturan kerja untuk AI coding agent, 326 baris |
| `sijempol-be/GOWA.md` | Catatan integrasi GOWA, 40 baris |
| `sijempol-fe/src/features/ARCHITECTURE.md` | Aturan batas antar-fitur frontend |
| `sijempol-scan/README.md` | Cara menjalankan kiosk |

---

## PENUTUP

Yang Anda warisi adalah sistem yang **arsitekturnya rapi, disiplinnya tinggi,
dan sebagian besar sudah selesai**. Fondasinya kuat: transaksi database dipakai
dengan benar, race condition sudah dipikirkan, integrasi eksternal dipisah
dengan baik, dan ada ratusan test yang menjaga supaya perubahan Anda tidak
merusak yang sudah jalan.

Pekerjaan yang tersisa terkonsentrasi di **satu area: pelaporan** — persis
seperti yang dikatakan teman Anda. Tiga hal konkret:
1. Rekap bulanan yang benar-benar menghitung (bukan "100%" hardcoded)
2. Cetak rekap ke PDF/Excel
3. Cetak PDF laporan penggeledahan

Ditambah satu hal yang mungkin belum disadari siapa pun: **belum ada cara
membuat kartu QR petugas dari dalam aplikasi**.

Mulailah dari menjalankan ketiganya di komputer Anda, klik semua menunya, lalu
baca ALUR H (scan QR) sambil membuka file-nya. Setelah alur itu terasa masuk
akal, alur lain akan mengikuti.

Selamat bekerja.

---

*Dokumen ini dibuat dengan membaca langsung seluruh kode ketiga repositori.
Setiap klaim di dalamnya merujuk ke file dan baris yang bisa Anda verifikasi
sendiri. Kalau ada bagian yang berubah setelah dokumen ini ditulis, kode selalu
menjadi sumber kebenaran.*
